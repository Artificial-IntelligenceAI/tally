# 8. Ownership, moves and drops

## 8.1 Owners

**8.1.1** Every value that is not `Copy` has exactly one *owner* at any time.
An owner is a local binding or parameter, a temporary (8.4), a field of an
owned struct, enum or tuple, an element of an owned array, the heap cell of
a `Box`, or the storage managed by a library type built on raw pointers (such
as the buffer of a `Vec`). A value is dropped (8.5) when its owner ends
without the value having been moved out.

```tally
fn f() may alloc {
    let a = String.from("x") orelse return;  // a owns the string
    let b = a;                               // ownership moves to b
}                                            // b's string is dropped here; a owns nothing
```
*From: D23.*

## 8.2 Moves

**8.2.1** Using a place by value (5.1.1) *moves* the value out of it if the
place's type is not `Copy`. A value may be moved out only of a whole local
binding, parameter or temporary. Moving out of a field, an element, a
dereference (`*r`, `*b` for a `Box`, `*p` for a raw pointer) or a `static` is
an error; use `mem.replace`, `mem.take`, `Box.into_inner` or pattern
destructuring of the whole owner instead.

```tally
let p = Pair { a: String.new(), b: String.new() };
let x = p.a;                          // error: cannot move out of field p.a
let Pair { a, b } = p;                // fine: p moved as a whole, destructured
var v: Vec[String] = ...;
let s = v[0];                         // error: cannot move out of an element
let s = mem.replace(&mut v[0], String.new());   // fine
```
*From: D23.*

**8.2.2** After a binding is moved from, it is *uninitialised*. Using an
uninitialised binding in any way (reading, borrowing, moving, calling a
method on it) is an error. A `var` binding becomes initialised again when it
is assigned (8.3.2). A `let` binding cannot be reassigned.

```tally
var s = String.from("a")?;
consume(s);
print(s.len());              // error: use of moved value `s`
s = String.from("b")?;       // re-initialised
print(s.len());              // fine
```
*From: D23.*

**8.2.3** Destructuring a value whose type has a `Drop` impl by a pattern that
moves any of its fields is an error.

```tally
impl Drop for Guard { ... }
let Guard { lock } = g;       // error: Guard has a Drop impl
```
*From: D23, D25.*

**8.2.4** A move may happen on some control-flow paths and not others. At a
point where a binding is initialised on some incoming paths and moved on
others, it is *maybe-moved*: using it is an error, and if it is dropped at
the end of its scope, the implementation uses a hidden flag, set when the
binding is initialised and cleared when it is moved, to drop it only if
initialised. This flag (a *drop flag*) is the only run-time state the
language adds for ownership.

```tally
let s = String.from("x")?;
if cond { consume(s); }
// s is maybe-moved here
print(s.len());              // error
// at scope end, s is dropped only if `cond` was false (drop flag)
```
*From: D23, D2.*

**8.2.5** A binding declared outside a loop and moved inside the loop body is
an error unless every path from the move back to the loop head assigns the
binding again.

```tally
var buf = Vec.new();
for line in lines {
    buf.push(line)?;
    send(buf);               // error: buf moved in a previous iteration
}
for line in lines {
    send(buf);
    buf = Vec.new();         // fine: re-initialised before next iteration
}
```
*From: D23.*

## 8.3 Initialisation

**8.3.1** A binding is *definitely initialised* at a point if on every path
from its declaration to that point it has been assigned and not moved since.
Every use of a binding must be at a point where it is definitely initialised.
A `let` binding declared without an initialiser may be assigned only at points
where it is definitely *un*initialised (it is assigned at most once on every
path).

```tally
let x: u32;
if a { x = 1; }
print(x);              // error: x possibly uninitialised
let y: u32;
y = 1;
y = 2;                 // error: `let` binding assigned twice
```
*From: D23, D2.*

**8.3.2** Assigning to an uninitialised `var` binding initialises it without
dropping anything. Assigning to an initialised one drops the old value first
(5.9.4). Assigning to a field or element requires its owner to be initialised:
values cannot be built field by field.

```tally
var p: Point;
p.x = 1.0;             // error: p is uninitialised
p = Point { x: 1.0, y: 2.0 };
```
*From: D23.*

**8.3.3** A function's parameters are initialised at entry. Statics are always
initialised. There is no way to read uninitialised memory in safe code;
`core.mem.Uninit[T]` (19.4.4) provides uninitialised storage for unsafe code.

```tally
var buf: Uninit[[4096]u8] = Uninit.new();   // no initialisation cost; unsafe to read
```
*From: D2, D36.*

## 8.4 Temporaries

**8.4.1** A *temporary* is created when a value expression is used where a
place is needed: as the operand of `&` or `&mut`, as the receiver of a method
that takes `&self` or `&mut self`, as the base of a field access or index, or
as a `match`/`if let`/`while let` scrutinee. The temporary owns the value.

```tally
let n = make_list().len();      // temporary holds the list during len()
```
*From: D25.*

**8.4.2** A temporary is dropped at the end of the *full expression* that
created it, in reverse order of creation. The full expressions are: an
expression statement; the initialiser of a `let` or `var` (the whole
statement, including `else`); the right side of an assignment (with the
whole statement); the tail expression of a block (dropped after the tail value
has been computed, before the block's bindings are dropped); an `if` or
`while` condition (dropped before the block runs); a `return`, `break` or
`defer` operand. A temporary's lifetime is never extended beyond this, so a
reference to a temporary cannot be stored in a binding.

```tally
let r = &make();          // error: temporary dropped at end of statement while r is live
let v = make();
let r = &v;               // fine
let len = { let s = name(); &s }.len();   // error: block tail borrows s, which is dropped at block end
```
*From: D25.*

**8.4.3** Temporaries created while evaluating the scrutinee of a `match` or
`if let` are dropped at the end of the `match` or `if let` expression; those of
a `while let` scrutinee at the end of each iteration (or when the loop exits).

```tally
match lock.guard().state {      // the guard lives until the end of the match
    .Ready => go(),
    .Busy => wait(),
}
```
*From: D25.*

## 8.5 Drops

**8.5.1** *Dropping* a value of type `T` runs its *drop glue*:

1. if `T` has a `Drop` impl, call `Drop.drop(&mut v)`;
2. then drop the parts of the value: struct fields in declaration order; the
   fields of an enum's active variant in declaration order; tuple elements and
   array elements from first to last; for `Box[U, A]`, the boxed `U` and then
   its memory is returned to `A`; for a closure, its by-value captures in the
   order they were captured.

References, raw pointers, function pointers, unions and `Copy` types have no
drop glue. Library types built on raw pointers (such as `Vec`) drop their
contents in their own `Drop` impl.

```tally
struct Pair { first: Noisy, second: Noisy }
// dropping a Pair drops `first` then `second`
let b = Box.new(Pair { ... })?;   // dropping b drops the Pair, then frees the box
```
*From: D25, D23.*

**8.5.2** When control leaves a block (normally or through `break`,
`continue`, `return` or `?`), the block's bindings that are initialised are
dropped in reverse order of declaration, interleaved with the block's `defer`
expressions (5.9.5). When a function returns, its parameters are dropped after
the body's bindings, in reverse order of the parameter list. Values moved out
are not dropped.

```tally
fn f(a: Noisy, b: Noisy) {
    let c = Noisy.new("c");
    defer log("d");
    let e = Noisy.new("e");
}
// drop order: e, then "d" logged, then c, then b, then a
```
*From: D25.*

**8.5.3** Each value is dropped at most once. Leaking a value (never dropping
it) is safe; `mem.forget(v)` moves `v` and does not drop it, and a value kept
alive by a reference-counting cycle is never dropped. Statics are never
dropped. When the program panics or exits through `process.exit`, no drops run
(12.4.1, 20.5.1).

```tally
let f = File.open("log")?;
mem.forget(f);           // the file descriptor is never closed: safe, but leaks
```
*From: D25, D21.*

**8.5.4** The *drop effects* of a type are the union of the effects of every
`Drop.drop` impl in its drop glue (10.3.3). Every point where a value is
dropped (scope end, assignment over an initialised place, `_ = e`, temporaries)
contributes the drop effects of its type to the enclosing function.

```tally
fn f() may sys | alloc {
    let file = File.open("x") orelse return;   // File.Drop = sys
    let v: Vec[u8] = Vec.new();                 // Vec[u8, Global].Drop = alloc
}   // both dropped here: contributes sys | alloc
```
*From: D3, D25, D28.*
