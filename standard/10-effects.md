# 10. Effects

This chapter defines the effect half of cost accounting (D3). An *effect* is a
compile-time marker for something a function may do besides computing its
result. Effects have no run-time representation and never change what a
program does; they only decide which programs are accepted.

## 10.1 Overview

**10.1.1** Every function, closure, function pointer type, `dyn` type and
trait method has an *effect set*. The effect set of a function is an upper
bound on the effects that any execution of the function, including
everything it calls and everything it drops, can perform (10.7.1). The
compiler computes the effects of every body and rejects any function whose
body may perform an effect outside its declared set.

```tally
fn on_irq() may none {
    COUNTER.fetch_add(1, .Relaxed);    // effect-free
    log("irq");                        // error: log has effects `sys | block`, not within `none`
}
```
*From: D3, D28.*

## 10.2 Effects and effect sets

**10.2.1** There are four built-in effects. They are not items: their names
`alloc`, `block`, `panic` and `sys` are recognised only as effect terms
(1.3.3), cannot be shadowed there, and do not conflict with items of the same
name (such as the prelude function `panic`, 19.18.1).

| Effect | A function with it may... |
|---|---|
| `alloc` | allocate memory from, or return memory to, the program's global allocator (19.13.5), by any route |
| `block` | wait for an event outside the current execution context (15.1.1): I/O readiness, a lock, a condition, a timer, another thread or fiber, a child process; this includes sleeping. Busy-waiting is not `block`. |
| `panic` | panic (12.4) |
| `sys` | interact with anything outside the program's own memory: make a system call, access memory-mapped hardware through volatile operations, execute inline assembly declared `sys`, or call foreign code declared `sys` |

The effects of the global allocator's own implementation (which may lock or
make system calls) are not separately visible; they are part of what `alloc`
means.

```tally
pub fn now() -> Instant may sys { ... }               // clock_gettime: a system call
pub fn sleep(d: Duration) may sys | block { ... }
pub fn sum(xs: &[]u64) -> u64 may panic { ... }        // overflow may panic
pub fn len(xs: &[]u64) -> usize { xs.len() }           // no effects
```
*From: D28.*

**10.2.2** `effect name;` declares a user effect. Its meaning is whatever its
package documents; the compiler treats it as an opaque marker. A user effect
enters a function's effect set only by appearing in a `may` clause (of the
function, or of something it calls), since declared sets may be larger than
inferred ones (10.4.3). User effects follow the visibility rules of items
(3.2.2) and are named by path (`db.query`). A user effect may not be named
`none` or `any`, nor the name of a built-in effect.

```tally
// package `store`
pub effect disk_write;
pub fn commit(t: &mut Txn) ! Error may disk_write | sys | block { ... }
// a dependent:
fn save(t: &mut store.Txn) ! store.Error may store.disk_write | sys | block { store.commit(t) }
```
*From: D28.*

**10.2.3** An *effect expression* is a union of *effect terms*, written
`t1 | t2 | ...`. The terms are:

* `none`, the empty set, and `any`, the set of all effects (built-in and
  user-declared, in every package);
* the name of an effect (built-in or user-declared);
* the name of an effect parameter `E` in scope;
* the name of a type parameter `T` in scope, or `Self` inside a trait or
  impl, standing for the *type effects* of that type (10.2.4);
* `T.Name` or `Self.Name`, standing for the associated effect `Name` of
  that type, from the unique bound of `T` that declares it;
* `T.Name` or `Self.Name` where `Name` is an associated *type*, standing for
  the type effects (10.2.4) of that associated type with respect to the
  bounds declared on it (for an unbounded associated type, its drop effects).
  Library signatures use this for items that a method drops, as in
  `count(self) -> usize may Self | Self.Item | panic`.

```tally
fn f[T: Iterator + Clone, effect E](t: T, g: fn() may E) may T.Next | E | panic { ... }
```
*From: D30, D31.*

**10.2.4** The *type effects* of a type `X` with respect to a set of bounds
`B` (the bounds that a type parameter was declared with, where `X` was
substituted for it) are the union of: for each trait in `B` and each of its
supertraits, every associated effect of `X`'s impl of that trait; and the
drop effects of `X` (10.3.3). For a type parameter `T` that has not been
substituted, the type effects are the symbolic term `T`.

```tally
fn g[T: Clone](x: &T) -> T may T { x.clone() }
// g[String]: T = String.Clone ∪ drop(String) = alloc ∪ alloc = alloc
// g[u32]: none
```
*From: D30.*

**10.2.5** The *normal form* of an effect set is a set of concrete effects
(built-in and user) plus a set of symbolic terms (effect parameters, type
parameters, associated effects of type parameters), or `any`. `any` absorbs
everything. An effect set E1 is *within* E2 (E1 ⊆ E2) if E2 is `any`, or every
concrete effect of E1 is in E2 and every symbolic term of E1 is *covered* by
E2. A term is covered if it occurs in E2; or it is `T.Name` and `T` occurs in
E2; or a bound in scope limits it (10.5.3) to a set within E2.

```tally
fn h[T: Iterator may sys](t: T) may sys | panic { for x in t { ... } }
// the loop contributes T.Next and T's drop effects; the bound `may sys` limits both to sys
```
*From: D30.*

## 10.3 The effects of a body

**10.3.1** The *body effects* of a function or closure are the union of the
contributions of every call, operation and drop in its body (not counting
the bodies of closures it contains, which have their own effects, 10.3.5),
as given in 10.3.2–10.3.7. An expression or statement that is never executed
because it follows an expression of type `never` still contributes.

```tally
fn f() -> u32 may panic {
    return 1;
    risky();      // still contributes risky's effects; and this is unreachable code
}
```
*From: D3, D2.*

**10.3.2** A *call* contributes the callee's effect set, instantiated:

* calling a function item contributes its declared effect set (or its
  inferred set, 10.4.2), with its generic type parameters, effect parameters
  and `Self` replaced by the instance's arguments, type terms expanded by
  10.2.4;
* calling a trait method through a bound on a type parameter `T` contributes
  the trait method's declared set with `Self` replaced by `T`;
* calling a function pointer contributes the effect set in its type;
* calling a method on a `dyn` value contributes the binding (or `may` bound)
  for the method's associated effects (7.8.2);
* calling a value of generic type `F` through `Fn`/`FnMut`/`FnOnce`
  contributes `F.Call`;
* calling a closure directly contributes the closure's effect set;
* `for` loops contribute the effects of `into_iter`, `next` and dropping the
  iterator (5.10.6).

```tally
fn apply[F: Fn(u8) -> u8](f: F) -> u8 may F { f(1) }   // contributes F.Call ⊆ F
fn run(cb: fn() may sys) may sys { cb(); }
```
*From: D3, D30.*

**10.3.3** A *drop* (8.5.4) of a value of type `X` contributes the *drop
effects* of `X`: the union, over every `Drop` impl in `X`'s drop glue (8.5.1),
of that impl's `Drop` associated effect, instantiated. For a type parameter
`T`, dropping a `T` contributes the term `T`. Drop points include scope ends,
assignments over initialised places, `_ = e`, the end of a temporary, moving
a value into a function that drops it (the callee's effects cover that), and
the drop of an iterator after `for`.

```tally
fn clear[T](v: &mut Vec[T]) may T | alloc { v.truncate(0); }   // drops elements: T
fn tmp() may alloc { let v: Vec[u8] = Vec.new(); }             // dropping v: Vec[u8, Global] drop = alloc
```
*From: D3, D25.*

**10.3.4** An operation that can panic (5.3.5, 5.4, 5.6.4, 5.6.5, 12.4.2)
contributes `panic`. An `asm` expression contributes its declared effects
(14.4.3). Dereferencing a raw pointer, reading or writing a `static var`, and
accessing a union field contribute nothing. The built-in operations not
mentioned in this chapter (arithmetic that cannot panic, comparisons, field
access, borrowing, moves, `Deref` calls, which must be effect-free) contribute
nothing.

```tally
fn peek(p: *u32) -> u32 { unsafe { *p } }       // no effects
fn mmio(p: *u32) -> u32 may sys { unsafe { p.read_volatile() } }   // volatile access: sys
```
*From: D3, D15.*

**10.3.5** A closure expression contributes nothing when it is evaluated;
the closure's own effect set is the effect set of its `Call` associated
effect, which is its declared `may` clause, or else its body effects. Dropping
a closure value contributes the drop effects of its by-value captures.

```tally
let f = || alloc_thing();     // creating f: no effect
f();                          // calling f: f's effects
```
*From: D54, D3.*

**10.3.6** The body effects of `unsafe may E { B }` are `E`, regardless of
the effects of `B` (10.6.1).

```tally
let x = unsafe may none { fast_path_that_never_allocates(v) };
```
*From: D3, D36.*

**10.3.7** Code excluded by `@cfg` (16.2) contributes nothing. Effects are
computed after `@cfg` evaluation for the current target and configuration, so
a function may have different effects on different targets; a public
function's declared effects must cover its body on every target it is built
for.

```tally
pub fn yield_now() may sys {
    @cfg(os("linux")) { unsafe { sched_yield(); } }
    @cfg(os("none")) { spin_hint(); }
}
```
*From: D3, D42.*

## 10.4 Declared and inferred effect sets

**10.4.1** A function's *declared* effect set is the set given by its `may`
clause. These must have a `may` clause or else have the declared set `none`:
public functions and methods (`pub fn` items, `pub` members of inherent
impls), trait method declarations, function pointer types, and `dyn` types.
Methods in trait impls, package-visible functions, and closures may omit the
clause, and then have an *inferred* effect set (10.4.2). Foreign function
declarations must have a `may` clause (an omitted clause is an error).

```tally
pub fn area(r: f64) -> f64 { 3.14159 * r * r }       // declared: none (fine)
pub fn grow(v: &mut Vec[u8]) ! AllocError { v.push(0) }   // error: push has effect alloc
fn helper(v: &mut Vec[u8]) ! AllocError { v.push(0) }     // inferred: alloc
```
*From: D29.*

**10.4.2** If a function has a declared effect set and its body effects are
not within it (10.2.5), the program has an error. The diagnostic must name
at least one effect or term that is not covered and one operation, call or
drop in the body that contributes it.

```tally
fn quiet() may none { alloc_something(); }
// error: `quiet` may not have effect `alloc`
//        `alloc_something()` (line 1) has effect `alloc`
```
*From: D3, D29.*

**10.4.3** A declared set may be larger than the body effects; the declared
set is what callers see. This lets a public function reserve the right to
add an effect later without breaking callers.

```tally
pub fn lookup(k: u32) -> ?u32 may block { TABLE.get(k) }   // reserves `block` for a future cache lock
```
*From: D29, D44.*

**10.4.4** The inferred effect sets of all functions in a package without
declared sets are the smallest sets satisfying: each such function's set
equals its body effects, where calls to other inferred functions use their
inferred sets. They are computed by starting from empty sets and
re-evaluating every body until nothing changes. Symbolic terms in an inferred
generic function's set refer to its own type and effect parameters; calls
between generic functions substitute as in 10.3.2. The sets of closures are
inferred together with their enclosing function.

```tally
fn a(n: u32) -> u32 { if n == 0 { 0 } else { b(n) } }
fn b(n: u32) -> u32 { a(n - 1) }      // inferred: a = b = {panic} (from `n - 1`)
```
*From: D29.*

**10.4.5** An associated effect of a trait impl may be omitted only if the
trait declares a default for it (which is then used) or the impl is not
visible outside its package (the trait or the implemented type is
package-visible); in the latter case it is inferred as the smallest set that
makes every method of the impl satisfy the trait's declarations.

```tally
struct Lines { ... }              // package-visible
impl Iterator for Lines {         // `effect Next` inferred from `next`'s body
    type Item = String;
    fn next(&mut self) -> ?String { ... }
}
```
*From: D29, D30.*

## 10.5 Effects in traits and generics

**10.5.1** A trait method's `may` clause is the upper bound for every impl of
that method: an impl method's effect set (declared or inferred), with the
trait's `Self.Name` terms replaced by the impl's bindings, must be within the
trait's clause. An impl method may declare a smaller set; callers that call it
on the concrete type (not through a bound or `dyn`) see the smaller set.

```tally
pub trait Store { effect Io; fn get(&self, k: u32) -> ?u32 may Self.Io | panic; }
impl Store for MemStore {
    effect Io = none;
    fn get(&self, k: u32) -> ?u32 { self.map.get(&k).copied() }   // none: within none | panic
}
```
*From: D30.*

**10.5.2** A trait declares an associated effect with `effect Name;` or
`effect Name = default;`. An impl binds it with `effect Name = expr;`, where
`expr` may use the impl's generic parameters as terms. A bound or `dyn` type
may bind it: `T: Iterator[Next = E]` requires `T`'s binding of `Next` to be
within `E`.

```tally
impl[T: Clone] Clone for Vec[T, Global] { effect Clone = T | alloc; ... }
fn quiet_iter[I: Iterator[Next = none]](it: I) may I { ... }   // next is effect-free; drop may not be
```
*From: D30.*

**10.5.3** A bound `T: Tr may E` requires every associated effect of `T`'s
impl of `Tr` and its supertraits, and the drop effects of `T`, to be within
`E`. Inside the scope of the bound, the terms `T.Name` (for those effects) and
`T`'s drop effects are known to be within `E` when checking (10.2.5). A bound
`T: Tr[Name = E1]` makes `T.Name` within `E1`.

```tally
fn drain[T: Iterator may none](var it: T) -> u32 may panic {
    var n: u32 = 0;
    while let .Some(_) = it.next() { n += 1; }     // next: within none
    n                                              // drop of `it`: within none
}
```
*From: D30.*

**10.5.4** Effect parameters (`effect E`) range over effect sets. They may be
used in `may` clauses and in effect positions of types (function pointer
types, `dyn` bindings, generic arguments to other effect parameters). They are
inferred at uses like type parameters (4.13.4), or given explicitly. A method
whose only generic parameters are effect parameters is still object-safe
(7.8.1). Effect parameters are erased: they never create separate instances
(7.10.1).

```tally
pub struct Formatter[effect E] { sink: &mut dyn Sink may E, ... }
pub trait Display { fn fmt[effect E](&self, f: &mut Formatter[E]) ! FmtError may E | panic; }
let d: &dyn Display = &x;     // fine: fmt is generic only over an effect
```
*From: D30, D47.*

**10.5.5** When a generic function is instantiated, its effect set is
obtained by substituting: each type term `T` by the type effects of the
argument type with respect to `T`'s declared bounds (10.2.4); each `T.Name`
by the argument's impl binding; each effect parameter by its argument. The
result is used as the call's contribution (10.3.2). If a generic body is
correct, every instance is correct: no instance needs re-checking.

```tally
pub fn push[T, A: Allocator](v: &mut Vec[T, A], x: T) ! AllocError may A { ... }
push(&mut global_vec, 1)?;   // may alloc
push(&mut arena_vec, 1)?;    // may none, if the arena's allocator is effect-free
```
*From: D30, D32.*

## 10.6 Overriding effects in unsafe code

**10.6.1** `unsafe may E { B }` declares that executing `B` performs no
effect outside `E`. The compiler uses `E` as the block's effects (10.3.6)
without checking `B` against it. Its purpose is to state facts the checker
cannot see, such as that a `push` will not allocate because capacity was
reserved earlier. If `B` does perform an effect outside `E`, the behaviour of
the program is still defined (the effect happens), but the guarantee of
10.7.1 does not hold for executions that reach that block.

```tally
fn record(log: &mut Vec[Event], e: Event) may none {
    if log.len() < log.capacity() {
        // capacity was reserved in setup; push cannot allocate here
        unsafe may none { _ = log.push(e); }
    }
}
```
*From: D3, D36.*

## 10.7 The guarantee

**10.7.1** Let `f` be a function with effect set `S`, in a well-formed
program. Then no execution of `f` performs a built-in effect outside `S`,
where performing an effect means reaching one of its *primitive sources*:

* for `alloc`, a call to the global allocator (19.13.5);
* for `block`, a call to one of the waiting primitives of `core` and `std`
  that are declared `may block` and documented as primitive sources (20.1.3),
  or to a foreign function or `asm` declared `may block`;
* for `panic`, the start of panic handling (12.4.1);
* for `sys`, a volatile memory access, an `asm` declared `may sys`, or a call
  to a foreign function declared `may sys`.

The guarantee holds provided that every `unsafe may` block in the execution
is truthful, every foreign declaration's `may` clause is truthful, and the
program has no undefined behaviour.

```tally
fn audio_callback(buf: &mut []f32) may none { mix(buf); }
// guaranteed: no allocation, no waiting, no panic, no system call during mix(buf),
// assuming no lying unsafe blocks or foreign declarations.
```
*From: D3.*

## 10.8 Effects of entry points

**10.8.1** The program entry point `main` (17.1.1) may have any effects. The
entry function of a thread, a fiber, an interrupt handler or a signal handler
is a closure or function whose effect set is limited by the API or attribute
that installs it (chapters 15 and 17); exceeding the limit is an error at the
installation site.

```tally
@interrupt("TIMER0") fn tick() may none { ... }        // interrupt handlers: no alloc, no block (17.4.2)
thread.spawn(move || serve(conn))?;                    // threads: any built-in effects
```
*From: D3, D34.*
