# 9. References and borrow checking

This chapter defines when references may be created and used. The checking
is done separately for each function body (including the bodies of closures
in it, which are checked together with their enclosing function), using only
the signatures of the functions it calls. The model follows the non-lexical
lifetimes design of Rust (RFC 2094), adapted to signatures without lifetime
names.

## 9.1 The guarantees

**9.1.1** In a well-formed program without unsafe code, every use of a
reference is to a live, valid value, and:

1. while an exclusive reference (`&mut`) to a place can still be used, the
   place is accessed only through that reference or references derived from
   it;
2. while a shared reference (`&`) to a place can still be used, the place is
   not modified, moved or dropped, and no exclusive reference to it is
   created, except for the contents of an `UnsafeCell` (9.8.1);
3. a place is not moved, dropped, or reassigned while a reference into it can
   still be used.

The rest of this chapter defines an analysis that rejects every program that
could break these rules, and accepts the programs described.

```tally
var v = vec_of(1, 2, 3)?;
let first = &v[0];
v.push(4)?;              // error: v is mutably borrowed while `first` is still used
print(*first);
```
*From: D24, D2.*

## 9.2 The control-flow graph

**9.2.1** For checking, a function body is lowered to a *control-flow graph*
(CFG) whose nodes, called *points*, each perform one primitive action. The
primitive actions are: assign a value (computed from operands) to a place;
create a borrow of a place; copy or move from a place; call a function; drop a
place; begin or end the storage of a local (`StorageLive`, `StorageDead`); and
branch. The order of points follows the evaluation order of 5.1.2, the
desugarings of chapter 5 (`for`, `?`, `orelse`, `defer`), and the drop order
of 8.5.2 and 8.4.2. Every binding and every temporary is a *local*.
`StorageDead(x)` occurs where `x`'s scope ends (for temporaries, at the end of
their full expression, 8.4.2).

```tally
let a = 1;              // StorageLive(a); a = const 1
let r = &a;             // StorageLive(r); r = &a        (creates a loan of a)
print(*r);              // tmp = copy *r; call print(tmp)
                        // StorageDead(r); StorageDead(a)
```
*From: D24, D64.*

**9.2.2** A *place* is a local followed by zero or more *projections*: field
`.f`, tuple field `.0`, dereference `*`, and index `[_]`. All index
projections of the same base are considered to be the same place for
borrowing purposes, whatever the index values; subslicing `[a..b]` is an
index projection. Two places *overlap* if one is a prefix of the other, or if
they differ first at index projections; places that differ first at two
different fields of a struct or tuple are *disjoint*. Places that differ
first at fields of a union overlap.

```tally
let a = &mut p.x;
let b = &mut p.y;        // fine: p.x and p.y are disjoint
let c = &mut v[0];
let d = &mut v[1];       // error: v[_] and v[_] overlap
let (l, r) = v.split_at_mut(1);   // the library method gives two disjoint slices
```
*From: D24.*

## 9.3 Loans and conflicts

**9.3.1** Each borrow expression creates a *loan*: a triple of the borrowed
place, the kind (*shared* or *exclusive*), and a region (9.4.1), at the point
where the borrow is created. Each reborrow through a reference, such as
`&*r` or `&mut (*r).f`, also creates a loan of its place (which starts with
`*r`).

```tally
let r = &mut s;            // loan L1: (s, exclusive, ρ1)
let q = &(*r).name;        // loan L2: ((*r).name, shared, ρ2)
```
*From: D24.*

**9.3.2** A loan L is *in scope* at a point P if there is a path in the CFG
from L's creation point to P such that every point on the path after the
creation point belongs to L's region, and no point on the path assigns to a
local `x` such that L's place begins with `*x` (assigning a new reference to
`x` ends loans made through the old one).

```tally
var r = &a;
let s = &*r;               // loan of *r
r = &b;                    // assignment to r: the loan of *r is no longer in scope
```
*From: D24.*

**9.3.3** Every access to a place at a point P is checked against every loan
in scope at P whose place overlaps the accessed place. Accesses are
classified as *read* (copy, shared borrow, inspecting an enum tag in a match,
comparing), *write* (assignment, exclusive borrow), *move*, *drop*, and
*storage-dead*. The access is an error if:

* the loan is exclusive (any access conflicts), or
* the loan is shared and the access is anything other than a read.

A storage-dead or drop of a local `x` conflicts with every loan whose place
begins with `x` and does not pass through a dereference of a shared or
exclusive reference stored in `x` (the referent is not owned by `x`). A
storage-dead of `x` does not conflict with loans of `*x` when `x` is a
reference. A drop of a `Box` does conflict with loans of its contents.

```tally
let r = &mut v;
v.len();                   // error: read of v while exclusive loan in scope
r.push(1)?;                // (use of r keeps the loan in scope above)
fn bad() -> &u32 {
    let x = 5;
    &x                     // error: x's storage ends while the returned loan is in scope
}
```
*From: D24.*

**9.3.4** An exclusive borrow created implicitly as the receiver of a method
call (5.6.3), or as the left operand of a compound assignment, is
*two-phase*: from its creation until the call (or operation) begins, its loan
is treated as shared; it becomes exclusive when the call begins. This permits
the receiver's place to be read while the arguments are evaluated.

```tally
v.push(v.len())?;          // fine: &mut v is reserved, v.len() reads v, then push activates
v.push({ v.clear(); 1 })?; // error: clear writes v while the reservation is in scope
```
*From: D24.*

## 9.4 Regions

**9.4.1** A *region* is a set of CFG points, together with zero or more
*end markers* `end(ρ)` for universal regions ρ (9.4.3). Every reference type
`&T` / `&mut T` that appears in the type of a local carries a region; every
view struct or view enum type carries one region (9.5.1); every closure type
that captures by reference carries one region per such capture. Regions in the
types of locals are *region variables* to be inferred.

```tally
let r: &u32 = &x;          // r: &ρ1 u32, the loan's region ρ2, with constraint ρ2 ⊇ ρ1
```
*From: D24.*

**9.4.2** The regions are the smallest sets satisfying these constraints:

1. *Liveness.* If a local `x` is live at point P (some path from P reaches a
   use of `x` without an intervening assignment to `x`), then every region in
   the type of `x` contains P. A drop of `x` is a use of the regions its drop
   glue uses (9.6.1), and only of those.
2. *Subtyping.* Wherever a value of type `T1` is stored into a place of type
   `T2` (assignment, initialisation, argument passing, return, field
   initialisation), `T1` must be a subtype of `T2` (9.4.5), which generates
   constraints `ρa ⊇ ρb` (written `ρa : ρb`, "ρa outlives ρb").
3. *Loans.* A borrow producing a reference of type `&ρ T` from a loan with
   region ρL requires ρL : ρ.
4. *Reborrows.* For a loan of a place containing dereferences, consider
   its dereferences from the last (rightmost) to the first. For each
   dereference of a reference of region ρd, require ρd : ρL; then stop if
   that reference is shared (`&`), or continue with the preceding
   dereferences if it is exclusive (`&mut`). Dereferences of `Box` are passed
   over. (A loan through a shared reference needs only that reference to stay
   valid; a loan through an exclusive reference needs the whole chain.)
5. *Calls.* As given in 9.4.4.
6. *Universal regions* contain every point of the body and their own end
   marker, and the end markers of every universal region they are known to
   outlive (9.4.3).

A constraint `ρa : ρb` is satisfied when ρa ⊇ ρb (points and end markers).

```tally
let x = 1;
let a = &x;          // loan region ρL ⊇ region of a
let b = a;           // region of a ⊇ region of b (subtyping)
print(*b);           // b live here ⇒ point in b's region ⇒ in a's ⇒ in ρL
```
*From: D24.*

**9.4.3** The *universal regions* of a function are: `static`; one region
for each elided region in each parameter type (each `&`, each view type
occurrence, each region inside a generic argument, written or implied, in
the order they appear); in a method of an `impl` whose implemented type
contains regions, those regions (*impl regions*); and one region `ret` for all
elided regions in the return type. The *known relations* are: `static` outlives
every region; every region outlives itself; for a parameter type `&ρ1 T` where
`T` contains region ρ2, ρ2 outlives ρ1 (well-formedness); for each `from`
source, its regions outlive `ret` (9.4.4). After solving, if a universal region
ρa contains `end(ρb)` and "ρa outlives ρb" is not a known relation, the
program has an error.

```tally
fn pick(a: &str, b: &str) -> &str from a { b }
// error: the returned value borrows from `b`, which is not named in `from`
fn name() -> &str { "fixed" }       // fine: string literals have region static
```
*From: D24.*

**9.4.4** The `from` clause of a function relates `ret` to its sources. A
source `p` denotes the *top-level regions* of parameter `p`'s type: for a
reference type `&ρ T` or `&ρ mut T`, the region ρ; for any other type, every
region occurring in it. A source `*p` denotes the top-level regions of the
referent type of `p` (which must be a reference type), `**p` of its referent,
and so on; `self` and `*self` work the same for the receiver. Each region
denoted by a source outlives `ret`. If the return type contains elided
regions and there is no `from` clause, the sources are all parameters whose
types contain regions. If the return type contains no elided regions, a
`from` clause is an error.

At a call site, each of the callee's universal regions other than `static`
is replaced by a fresh region variable; the argument types must be subtypes
of the instantiated parameter types; and for each source region σ of the
callee's `from` clause, the corresponding instantiated σ must outlive the
instantiated `ret`, which is the region of the call's result.

```tally
fn first_word(s: &str) -> &str { ... }              // implicit `from s`
fn longer(a: &str, b: &str) -> &str from a, b { ... }
view struct Lexer { src: &str, pos: usize }
impl Lexer {
    // the token borrows from the source text, not from the &mut borrow of the lexer:
    fn next_token(&mut self) -> ?&str from *self { ... }
}
var lx = Lexer { src: text, pos: 0 };
let t1 = lx.next_token();
let t2 = lx.next_token();   // fine: t1 borrows text (region of *self), not lx
```
*From: D24.*

**9.4.5** `T1` is a subtype of `T2` if they are the same type after equating
regions according to variance: a region or type argument in a *covariant*
position requires (T1's region) : (T2's region) or recursively T1 part <: T2
part; in an *invariant* position it requires both directions (equality).
Positions are: the region of `&ρ T` covariant and `T` covariant; for
`&ρ mut T`, ρ covariant and `T` invariant; `*T` covariant; `*mut T`
invariant; elements of arrays, slices, tuples, `?T`, `T ! E` covariant;
function pointer parameter and return types invariant; `dyn` bindings
invariant; `UnsafeCell[T]` invariant. For a nominal type, each region and
type parameter's variance is computed from its fields: covariant if every
occurrence in the fields is in a covariant position, otherwise invariant
(computed as a greatest fixed point over mutually recursive types).

```tally
fn shorten(r: &mut &str, tmp: &str) {
    *r = tmp;      // error: &mut &ρ str is invariant in ρ; tmp's region would have to equal r's referent's
}
```
*From: D24.*

## 9.5 View types

**9.5.1** Each view struct or view enum declaration has exactly one region
parameter, called its *own region*. Every reference type written in its
fields carries that region, and every view type written in its fields is
instantiated with that region. A generic argument of a view type keeps its
own regions.

```tally
view struct Pair { a: &str, b: &str }       // both a and b have Pair's region
let p = Pair { a: &s1, b: &s2 };            // p's region is within both borrows
view struct Outer { inner: Pair, n: &u32 }  // inner instantiated with Outer's region
```
*From: D24.*

**9.5.2** Because all references in a view type share one region, building a
view value from references of different lifetimes gives it the shortest of
them, and a view struct cannot hold one reference that must outlive another
held in it. Code that needs this uses owned data, indices into a separately
borrowed container, or two separate view values.

```tally
view struct Ctx { input: &str, scratch: &mut Vec[u8] }
// both fields share one region: `input` can be used only as long as `scratch` is borrowed
```
*From: D24.*

**9.5.3** Inside an `impl` for a view type, the implemented type's own region
is an impl region (9.4.3). An associated type defined in such an impl may
mention references; those carry the impl region. A method can return data
of the impl region by naming `*self` (or `self` for a by-value receiver) in
its `from` clause, or by returning an associated type that carries the impl
region.

```tally
view struct Iter[T] { rest: &[]T }
impl[T] Iterator for Iter[T] {
    type Item = &T;                  // carries Iter's own region
    effect Next = none;
    fn next(&mut self) -> ?&T from *self {
        match self.rest { [] => .None, [h, ..t] => { self.rest = t; h } }
    }
}
```
*From: D24, D27.*

**9.5.4** A generic type instantiated with arguments containing regions
(`Vec[&str]`, `HashMap[&str, u32]`) contains those regions. It is not a view
declaration, but values of it are subject to the same liveness and subtyping
rules through its type arguments.

```tally
var words: Vec[&str] = Vec.new();
for w in text.split(' ') { words.push(w)?; }    // words borrows text
drop(text);                                     // error: text is borrowed by words, used below
print(words.len());
```
*From: D24.*

**9.5.5** A closure that captures bindings by reference creates, at the
closure expression, a loan of each captured place (shared or exclusive,
5.8.2), and the closure's type carries the region of each loan. While the
closure value is live, those loans are in scope.

```tally
var count = 0;
var inc = || count += 1;      // exclusive loan of count
print(count);                 // error: count is exclusively borrowed by inc, used below
inc();
```
*From: D24, D54.*

## 9.6 Drops and borrows

**9.6.1** The drop glue of type `T` *uses* region ρ if `T`'s glue includes a
`Drop.drop` call for a type `U` in which ρ occurs, other than solely within
type arguments for which `U`'s `Drop` impl is marked `@drop_ignores(T)`
(16.3.1). Dropping a value whose type has no `Drop` impl in its glue uses no
regions.

```tally
var v: Vec[&u32] = Vec.new();
let x = 5;
v.push(&x)?;
// at scope end: x dropped before v (reverse declaration order). Vec's Drop impl is
// marked @drop_ignores(T), and &u32 has no drop glue, so dropping v does not use
// the region of &x: accepted.
```
*From: D24, D25.*

**9.6.2** `@drop_ignores(T)` may be placed only on an `unsafe impl` of `Drop`
whose implemented type has type parameter `T`. It is the implementer's
promise that `drop` does not read or write any value of type `T` except to
drop it, and does not otherwise use references contained in `T`. Breaking the
promise is undefined behaviour.

```tally
@drop_ignores(T)
unsafe impl[T, A: Allocator] Drop for Vec[T, A] {
    effect Drop = T | A;
    fn drop(&mut self) may T | A { ... drop elements, free buffer ... }
}
```
*From: D24, D36.*

## 9.7 Static and raw data

**9.7.1** String literals, byte string literals, references to statics, and
references to constant values promoted to static storage have region
`static`. A reference to a constant expression `&C` where `C` is a constant
of a type with no interior mutability and no drop glue is *promoted*: the
value is placed in static storage and the reference has region `static`.

```tally
fn tbl() -> &[4]u8 { &[1, 2, 4, 8] }        // promoted to static storage
static NAMES: [2]&str = ["a", "b"];
```
*From: D24, D40.*

**9.7.2** Raw pointers carry no region and are not checked. Converting a raw
pointer to a reference (`unsafe { &*p }`) produces a reference whose region is
inferred like any other; the programmer is responsible for its validity
(13.4.3).

```tally
let r: &u32 = unsafe { &*ptr };    // unchecked: the pointee must stay valid while r is used
```
*From: D36.*

## 9.8 Interior mutability

**9.8.1** `core.cell.UnsafeCell[T]` is the only type through which a value
reached by a shared reference may be modified. Its method
`get(&self) -> *mut T` gives a raw pointer to its contents; writes through
that pointer while shared references to the cell exist are allowed and are
not undefined behaviour, provided no `&T` or `&mut T` reference to the
*contents* overlaps the write (13.4.4). Types built on `UnsafeCell` (such as
`Cell`, `Atomic`, `Mutex`) provide safe interfaces. A type containing an
`UnsafeCell` is not `Sync` unless it implements `Sync` explicitly.

```tally
let c = Cell.new(1);
let r1 = &c;
let r2 = &c;
r1.set(2);              // mutation through a shared reference
print(r2.get());        // 2
```
*From: D26.*
