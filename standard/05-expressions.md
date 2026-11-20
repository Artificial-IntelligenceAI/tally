# 5. Expressions and statements

## 5.1 Places, values and evaluation order

**5.1.1** Every expression is either a *place expression*, which denotes a
memory location, or a *value expression*. Place expressions are: a path to a
local binding, a parameter or a static; `self`; a field access or tuple field
access whose operand is a place; an indexing expression (5.6.4); and a
dereference `*e`. All others are value expressions. When a place expression is
used where a value is needed, the value is copied out if its type is `Copy`
(7.6.1) and moved out otherwise (8.2).

```tally
let p = Point { x: 1.0, y: 2.0 };
let a = p.x;                // place p.x read; f64 is Copy, so copied
let s = String.from("hi")?;
let t = s;                  // place s moved into t
```
*From: D22.*

**5.1.2** The operands of an expression are evaluated left to right, each
exactly once, before the operation itself. This applies to binary operators
(except `&&`, `||` and `orelse`, which may skip their right operand), function
arguments (after the callee expression; for a method call the receiver is
evaluated first), array and tuple elements, struct literal fields in the
order written, and index expressions (base before index). In an assignment
`p = v` or `p op= v`, `v` is evaluated first, then the operands of the place
`p`, then the store happens.

```tally
f(a(), b());            // f, then a(), then b(), then the call
v[i()] = g();           // g(), then v and i(), then the store
xs.push(next())?;       // xs (receiver), then next(), then push
```
*From: D64, D2.*

## 5.2 Literals, paths and grouping

**5.2.1** A literal evaluates to its value with the type given by chapter 4.
A path to a constant evaluates to a copy of the constant's value; to a
function, to the function item; to a unit struct or unit variant, to that
value; to a tuple struct or tuple variant, to its constructor function item.
A parenthesised expression has the value and kind (place or value) of its
contents.

```tally
let k = LIMIT;                     // constant
let f = Meters;                    // constructor: fn(f64) -> Meters
let m = (p).x;                     // parentheses keep p a place
```
*From: D2.*

**5.2.2** `.Name` (an *inferred variant*) denotes the variant `Name` of the
enum that is the expected type of the expression. If the expected type is not
known, or is not an enum, or has no variant `Name`, the program has an error.
For `?T` and `T ! E` the expected type's outermost constructor suffices.

```tally
let s: Shape = .Circle(1.0);
fn get() -> ?u8 { .None }
let x = .None;          // error: expected type unknown
```
*From: D19, D63.*

## 5.3 Arithmetic and bitwise operators

**5.3.1** The binary arithmetic operators `+ - * / %` and their wrapping
(`+% -% *%`) and saturating (`+| -| *|`) forms require both operands to have
the same integer type, or (for `+ - * / %` only) the same float type. The
result has that type. For integers:

* `+`, `-`, `*` compute the mathematical result and panic if it is not
  representable;
* `/` divides and truncates toward zero; `%` gives the remainder, with the
  sign of the dividend, so that `(a / b) * b + a % b == a`; both panic if
  `b == 0`, and for signed types both panic if `a` is the minimum value and
  `b == -1`;
* `+%`, `-%`, `*%` compute the result modulo 2^N (wrapping);
* `+|`, `-|`, `*|` clamp the mathematical result to the type's range.

```tally
let a: u8 = 250;
let b = a + 10;          // panics: 260 not representable
let c = a +% 10;         // 4
let d = a +| 10;         // 255
let e = -7 / 2;          // -3
let f = -7 % 2;          // -1
```
*From: D15.*

**5.3.2** Unary `-` on a signed integer negates it and panics if the operand
is the minimum value. Unary `-` on an unsigned integer type is an error,
except applied directly to a literal whose type is then inferred as signed.
Unary `-` on a float negates it (flips the sign bit).

```tally
let x: i8 = -128;
let y = -x;              // panics
let u: u32 = 5;
let z = -u;              // error: negation of unsigned type
```
*From: D15.*

**5.3.3** `&`, `|`, `^` require two operands of the same integer type, or two
`bool` operands, and compute bitwise AND, OR, XOR. On `bool` they evaluate
both operands. Unary `~` is bitwise NOT on integers. Unary `!` is logical NOT
on `bool`; `!` on an integer and `~` on a `bool` are errors.

```tally
let m = flags & 0x0f;
let both = a() & b();    // both called, even if a() is false
let inv = ~mask;
let no = !ready;
```
*From: D17, D2.*

**5.3.4** `a << n` and `a >> n` require `a` to have an integer type and `n` to
have any integer type; the result has `a`'s type. If `n` is negative or not
less than the bit width of `a`'s type, the operation panics. `<<` discards the
bits shifted out and does not check for overflow. `>>` is an arithmetic shift
for signed types and a logical shift for unsigned types.

```tally
let x: u8 = 0b1100_0001;
let y = x << 1;          // 0b1000_0010: high bit discarded, no panic
let z = x >> 8;          // panics: shift amount 8 not less than width 8
let s: i8 = -8;
let t = s >> 1;          // -4
```
*From: D15.*

**5.3.5** An arithmetic, shift or conversion operation contributes the
`panic` effect (10.3.2) to its function unless it cannot panic for any
operand values, as determined by these rules only:
the wrapping and saturating operators, `&`, `|`, `^`, `~`, `!`, and float
arithmetic never panic; `/` and `%` by a constant operand that is neither 0
nor -1 never panic; `<<` and `>>` by a constant amount within range never
panic; `as` between types where every source value is representable (4.11.2)
never panics. No other reasoning (about ranges of variables, or what an
optimiser could prove) is used.

```tally
fn halve(x: u32) -> u32 { x / 2 }                 // effect-free
fn avg(a: u32, b: u32) -> u32 may panic { (a + b) / 2 }   // + may panic
fn mask(x: u32) -> u32 { (x << 4) & 0xff0 }       // constant shift: effect-free
```
*From: D3, D15.*

## 5.4 Conversions with `as` and `as%`

**5.4.1** `e as T` and `e as% T` convert the value of `e` to type `T`. The
permitted pairs are those in 5.4.2–5.4.5; any other pair is an error.
Conversions on pointers and references are library operations, not `as`
(19.4).

```tally
let n = 3.9 as u8;         // 3
let p = &x as *u8;         // error: use (&x).as_ptr() or a coercion (4.11.1)
```
*From: D16.*

**5.4.2** Between integer types: `as` yields the same mathematical value and
panics if it is not representable in `T`; `as%` keeps the low N bits of the
two's-complement representation of the value, where N is the width of `T`
(truncating, or sign- or zero-extending according to the source type).

```tally
let a: i32 = -1;
let b = a as u8;           // panics
let c = a as% u8;          // 255
let d = 300 as% u8;        // 44
let e = (200 as u8) as% i8;  // -56
```
*From: D16.*

**5.4.3** Involving floats (`as` only; `as%` is an error):
integer to float rounds to the nearest representable value, ties to even,
and may produce an infinity; float to integer truncates toward zero and panics
if the value is NaN or the truncated value is not representable; `f64` to
`f32` rounds to nearest, ties to even (out-of-range values become infinities);
`f32` to `f64` is exact.

```tally
let i = 1e10 as u32;       // panics: out of range
let j = -0.9 as i32;       // 0
let k = 2.5 as u8;         // 2
let f = u128.MAX as f32;   // +infinity
```
*From: D16.*

**5.4.4** `bool` converts with `as` to any integer type (`false` is 0, `true`
is 1). `char` converts with `as` to any integer type (panicking if its scalar
value is not representable) and with `as%` to any integer type (truncating).
Integer types convert to `char` with `as`, which panics if the value is not a
Unicode scalar value; `u8 as char` never panics.

```tally
let one = true as u8;          // 1
let code = 'A' as u32;         // 65
let low = '€' as% u8;          // 0xAC
let c = 0x41 as char;          // 'A'
```
*From: D16, D57.*

**5.4.5** An enum with a declared tag type whose variants are all unit
variants converts with `as` to any integer type, yielding its tag value
(panicking if not representable in the target) and with `as%` truncating
it. Such an enum also has the associated function `from_tag(t: T) -> ?Self`,
where `T` is its tag type, which returns the variant with that tag value.

```tally
enum Op: u8 { Nop = 0x90, Ret = 0xC3 }
let b = Op.Ret as u32;           // 195
let o = Op.from_tag(0x90);       // .Some(Op.Nop)
let none = Op.from_tag(0);       // .None
```
*From: D16, D37.*

## 5.5 Comparison and logical operators

**5.5.1** A type is *comparable*, and implements the marker trait `Eq`
(7.6.6), if it is: an integer, float, `bool`, `char`,
`()` or `never`; a raw pointer or function pointer (compared by address); a
reference `&T` or `&mut T` to a comparable `T` (compared by the referenced
values); an array or slice of a comparable type, or `str`; a tuple of
comparable types; or a struct or enum declared with `@derive(Eq)` (16.3.3)
whose field types are all comparable for the given generic arguments. Unions,
closures, function items and `dyn` types are never comparable. `Option[T]` and
`Result[T, E]` are declared with `@derive(Eq)`.

```tally
@derive(Eq) struct Key { id: u32, name: String }   // String derives Eq
let same = k1 == k2;
let ok = "abc" == "abc";              // &str: compares contents
let p = &a as *u8 == &b as *u8;       // error: `as` cannot convert pointers; see 19.4
struct Handle(u32);
let h = Handle(1) == Handle(1);       // error: Handle does not derive Eq
```
*From: D17.*

**5.5.2** `==` and `!=` require two operands of the same comparable type and
yield `bool`. Equality is structural: arrays and slices are equal if they have
the same length and equal elements in order; `str` values if they have the
same bytes; tuples and structs if all fields are equal; enums if they have the
same variant and equal fields. Floats compare by IEEE 754 rules (NaN is not
equal to itself; `-0.0 == 0.0`). Comparison stops at the first unequal part.
`==` never has effects.

```tally
let a = [1.0, f64.NAN];
let b = a;
let e = a == b;        // false: NaN != NaN
let z = -0.0 == 0.0;   // true
```
*From: D17, D2.*

**5.5.3** `<`, `<=`, `>`, `>=` require two operands of the same integer,
float, `char` or `bool` type, or the same raw pointer type, and yield `bool`.
Integers and `char` compare by value, `bool` with `false < true`, floats by
IEEE 754 (every comparison involving NaN is false), raw pointers by address.
Ordering other types uses methods of the `Ord` trait (19.9.2).

```tally
let lt = 'a' < 'b';
let q = s1.cmp(&s2) == .Less;     // strings via Ord
```
*From: D17.*

**5.5.4** `&&` and `||` require `bool` operands. `a && b` evaluates `b` only if
`a` is `true`; `a || b` evaluates `b` only if `a` is `false`.

```tally
if i < v.len() && v[i] == 0 { ... }   // v[i] not evaluated when out of range
```
*From: D2.*

## 5.6 Field access, method calls, indexing, dereference

**5.6.1** `*e` dereferences `e`. If `e` has type `&T` or `&mut T`, `*e` is a
place of type `T`. If `e` has type `Box[T]`, `*e` is the place of the boxed
value. If `e` has a raw pointer type, `*e` is a place of type `T` and is an
unsafe operation. If `e`'s type `U` implements `Deref` (7.6.4), `*e` means
`*Deref.deref(&e)` as a place (or `*DerefMut.deref_mut(&mut e)` when used
mutably). Otherwise it is an error.

```tally
let r = &5;
let v = *r;
var b = Box.new(3)?;
*b += 1;
let s: String = ...;
let first = (*s).len();      // through Deref to str
```
*From: D58, D36.*

**5.6.2** `e.f` accesses the field `f`. Let `T` be the type of `e`. If `T` is
a struct (or union) with a field `f` visible here, the result is that field,
a place if `e` is a place. Otherwise, if `T` is a reference, a `Box`, or
implements `Deref`, the access is retried on `*e`; this *auto-deref* repeats
until a field is found or no deref applies, in which case the program has an
error. Raw pointers are never auto-dereferenced. `e.0`, `e.1`, … access
tuple and tuple-struct fields the same way.

```tally
let p: &&Point = &&pt;
let x = p.x;              // two auto-derefs
let raw: *Point = ...;
let y = raw.x;            // error: raw pointers are not auto-dereferenced
```
*From: D58.*

**5.6.3** A method call `e.m(args)` is resolved as follows. Let `T0` be the
type of `e`, and `T1`, `T2`, … the types obtained by repeatedly applying the
auto-deref of 5.6.2. For each `Ti` in order, and for each receiver form in the
order *by value* (`Ti`), *by shared reference* (`&Ti`), *by exclusive
reference* (`&mut Ti`), look for a method named `m` whose `self` parameter has
that type, first among inherent methods of `Ti`'s type, then among methods of
traits in scope (3.4.3) that `Ti`'s type implements, or that the bounds of a
type parameter `Ti` provide. The first form that finds exactly one method is
used: the receiver is dereferenced `i` times, then borrowed as the form
requires, and passed as the first argument. If one step finds more than one
trait method, the call is an error and must be written `Trait.m(e, args)`. If
no step finds a method, the call is an error. A field and a method may have
the same name; `e.m(...)` always looks for a method and `(e.m)(...)` calls a
field holding a callable.

```tally
var v: Vec[u8] = Vec.new();
v.push(1)?;            // Vec.push(&mut v, 1): autoref &mut
let n = v.len();       // Vec.len(&v)
let s = v.as_slice();  // found on Vec
let k = (&v).len();    // T0 = &Vec[u8]: deref once, then &
let f = (cfg.callback)(3);   // call a field
```
*From: D58, D27.*

**5.6.4** `a[i]` indexes `a`. After auto-deref (5.6.2), the base must be an
array `[N]T` or a slice `[]T`; the index must have type `usize`; the result is
a place of type `T`. If `i >= length`, the operation panics. If the base is an
array, the length is known, and the index is a constant expression, an
out-of-range index is a compile-time error. Indexing has the `panic` effect
except in that constant, in-range case.

```tally
let a = [10, 20, 30];
let x = a[1];
let y = a[3];          // error: index 3 out of bounds for [3]i64
let z = v[i];          // may panic
```
*From: D15, D2.*

**5.6.5** `a[r]` where `r` is a range expression (5.11) of `usize` *slices*
`a`. The base after auto-deref must be `[N]T`, `[]T` or `str`. The result is
a place of type `[]T` (or `str`) covering the elements from the range's start
(default 0) up to its end (default the length; inclusive ranges add one). It
panics if start > end, if end > length, or, for `str`, if either bound is not
on a UTF-8 character boundary. The result is normally borrowed immediately:
`&a[1..3]`.

```tally
let a = [1, 2, 3, 4];
let mid: &[]i64 = &a[1..3];      // [2, 3]
let all = &a[..];
let s = "héllo";
let bad = &s[0..2];              // panics: byte 2 is inside 'é'
```
*From: D15, D57.*

## 5.7 Calls

**5.7.1** `f(args)` calls `f`. The callee may be a function item, a function
pointer, a constructor (tuple struct or tuple variant), or a value whose type
implements `Fn`, `FnMut` or `FnOnce` (7.7.2), in which case the call uses the
first of `Fn.call(&f, args)`, `FnMut.call_mut(&mut f, args)`,
`FnOnce.call_once(f, args)` whose trait the type implements. The number of
arguments must equal the number of parameters, except as allowed by 5.7.5.

```tally
let g = |x: u32| x + 1;
let y = g(4);
let p = Point3(1.0, 2.0, 3.0);     // tuple-struct constructor
```
*From: D27, D54.*

**5.7.2** Each argument is evaluated and then moved or copied into its
parameter. Parameters are local bindings of the callee; a parameter declared
`var` is mutable. Calling an `unsafe fn` is an unsafe operation. Calling an
`extern` function is an unsafe operation.

```tally
fn bump(var n: u32) -> u32 may panic { n += 1; n }
let s = String.from("a")?;
consume(s);           // s moved into the parameter
let t = s;            // error: use of moved value (8.2.2)
```
*From: D22, D36.*

**5.7.3** Tally has no named arguments, default argument values or function
overloading. Within one scope a name refers to one function.

```tally
fn open(path: &str) {}
fn open(path: &str, mode: u32) {}     // error: `open` already declared
```
*From: D2, D18.*

**5.7.4** A function returns when control reaches the end of its body (the
body's value is returned) or a `return` expression. `return` with no operand
returns `()`. The returned value is coerced to the declared return type.

```tally
fn sign(x: i64) -> i64 {
    if x < 0 { return -1; }
    if x == 0 { 0 } else { 1 }
}
```
*From: D10.*

**5.7.5** A *rest parameter* `..name: A` must be the last parameter, and `A`
must be a type parameter of the function that appears nowhere else in the
parameter list. At a call, the arguments after the ordinary parameters (zero
or more) are evaluated left to right and collected into a tuple, which is
passed as the rest parameter; `A` is inferred as that tuple's type, or `()` if
there are none. Bounds on `A` are satisfied by the built-in tuple
implementations (7.9.2). An existing tuple cannot be passed as the rest; it is
passed as a single element.

```tally
pub fn println[A: Display](..args: A) may sys | block { ... }
println("x = ", x, ", y = ", y);   // A = (&str, i64, &str, i64)
println();                          // A = ()
```
*From: D47.*

## 5.8 Closures

**5.8.1** A closure expression creates a value of a new anonymous closure type
(4.9.3). Parameters without type annotations are inferred (4.13). The return
type is the type of the body unless annotated. `return` in a closure body
returns from the closure. A closure with a `may` clause has that declared
effect set; otherwise its effects are inferred from its body (10.4.4).

```tally
let sq = |x: f64| x * x;
let parse = |s: &str| -> u32 ! ParseError may panic { s.parse.[u32]() };
let total = items.iter().map(|i| i.price).sum();
```
*From: D54.*

**5.8.2** A closure *captures* each local binding of an enclosing function
that its body names. Without `move`, each binding is captured by shared
reference if the body only reads it, by exclusive reference if the body
mutates it or borrows it mutably, and by value if the body moves it (uses a
non-`Copy` value by value). With `move`, every captured binding is captured by
value (moved, or copied if `Copy`). A closure that captures anything by
reference mentions a reference (4.10.2) and is subject to borrow checking
(9.5.5).

```tally
var count = 0;
let name = String.from("x")?;
let inc = || count += 1;          // captures count by &mut
let show = || print(&name);       // captures name by &
let take = move || consume(name); // captures name by value
```
*From: D54, D24.*

**5.8.3** Every closure type implements `FnOnce`. It also implements `FnMut`
if its body does not move out of any value captured by value. It also
implements `Fn` if, in addition, its body does not mutate or mutably borrow
any captured binding. A closure type is `Copy` if it captures only by shared
reference or by value of `Copy` types, and `Clone` likewise.

```tally
let a = || 1;                       // Fn, FnMut, FnOnce, Copy
var n = 0;
let b = || { n += 1; n };           // FnMut, FnOnce
let s = String.from("s")?;
let c = move || s;                  // FnOnce only: moves s out
```
*From: D54, D27.*

## 5.9 Blocks and statements

**5.9.1** A block executes its statements in order and then evaluates its
tail expression, if any; the block's value is the tail's value. A block
without a tail has value `()`, unless its end is unreachable, in which case
its type is `never`. The end of a block is unreachable if one of its
statements is an expression statement or `let` initialiser of type `never`.
A block introduces a scope for the bindings declared in it.

```tally
let x = { let a = 2; a * 3 };              // 6
let y: u32 = { return 0; };               // block type never, coerces to u32
let z = { f(); };                          // ()
```
*From: D10.*

**5.9.2** An expression statement's expression must have type `()` or
`never`. A value of any other type must be used, or discarded explicitly with
`_ = e;`, which evaluates `e` and drops the result at the end of the
statement. A block-like expression statement without `;` must also have type
`()` or `never`.

```tally
compute();                 // error if compute returns a value: write `_ = compute();`
_ = file.write(b"x");      // explicit discard of a result
if ok { 1 } else { 2 }     // error as a statement: value of type i64 unused
```
*From: D52, D20.*

**5.9.3** `let p = e;` evaluates `e` and binds it by matching the irrefutable
pattern `p`. Bindings introduced by `let` are immutable; `var` (or `var`
inside a pattern, 6.2.1) makes them mutable. `let x: T;` without an
initialiser declares a binding that must be assigned before use on every
path, exactly once for `let` (8.3.1). `let p = e else { ... };` allows a
refutable `p`: if `e` does not match, the `else` block runs, and its type must
be `never`.

```tally
let a = 1;
var b = 2;
b += a;
a = 3;                         // error: `a` is not mutable
let c: u32;
if cond { c = 1; } else { c = 2; }
let .Some(v) = map.get(&k) else { return .None; };
```
*From: D52, D60.*

**5.9.4** In an assignment `p = v;` the left side must be a mutable place: a
`var` binding or a field or index of one, a place reached through `&mut` or
`Box` owned by a mutable place, `*r` for `r: &mut T`, a `static var` (inside
`unsafe`), or `*p` for a raw `*mut T` (inside `unsafe`). If the place holds a
value, that value is dropped after `v` is evaluated and before the new value
is stored. `p op= v;` evaluates `v`, then evaluates the place `p` once, then
stores `p op v` with the same checks as the operator. Assignment is a
statement, not an expression.

```tally
var s = String.from("old")?;
s = String.from("new")?;   // "old" dropped, then "new" stored
v[i] += 1;                 // place evaluated once; panics on overflow
let x = (y = 3);           // error: assignment is not an expression
```
*From: D52, D64, D25.*

**5.9.5** `defer e;` (or `defer { ... }`) registers `e` to run when control
leaves the enclosing block by any path: falling off its end, `break`,
`continue`, `return`, or `?` propagation. Deferred expressions run in reverse
order of registration, interleaved with the drops of the block's bindings in
reverse declaration order, as if each `defer` were a binding whose drop runs
`e`. The deferred expression must have type `()`, must not contain `?`, and
must not `return`, or `break` or `continue` to a target outside itself. It is
borrow-checked as if evaluated at the block's end. Deferred expressions do not
run when the program panics (12.4.1).

```tally
fn copy(src: &str) ! io.Error may sys | block {
    let fd = unsafe { c_open(src) };
    defer unsafe { c_close(fd); };
    let buf = Buf.new();     // dropped before c_close runs
    ...
}
```
*From: D56, D21.*

## 5.10 Control flow

**5.10.1** `if c { A } else { B }` evaluates `c`, which must be `bool`, then
evaluates exactly one block. Without `else`, the `if` has type `()` and `A`
must have type `()` or `never`. With `else`, the branch types are unified, or
each coerced to the expected type when it is known; an `else if` chain is
nested `if` expressions.

```tally
let kind = if n % 2 == 0 { "even" } else { "odd" };
if verbose { log("starting"); }
```
*From: D10.*

**5.10.2** `if let p = e { A } else { B }` evaluates `e` and matches it
against the pattern `p`; if it matches, `A` runs with `p`'s bindings in scope,
otherwise `B`. `while let p = e { A }` repeats while `e` matches. Temporaries
in `e` live until the end of the `if let` or of each iteration (8.4.3).

```tally
if let .Some(user) = find(id) { greet(user); } else { log("none"); }
while let .Some(job) = queue.pop() { run(job); }
```
*From: D60, D25.*

**5.10.3** `match e { arms }` evaluates `e` once (the *scrutinee*) and tests
the arms in order. An arm matches if its pattern matches and its guard, if
any, evaluates to `true`; the first matching arm's expression is evaluated
and is the value of the `match`. The arms must be exhaustive (6.4.1). Arm
types are unified, or coerced to the expected type when known. If the
scrutinee is a place, it is not moved unless a binding moves from it (6.3.2).

```tally
let msg = match code {
    0 => "ok",
    1..=99 => "warning",
    n if n < 0 => "invalid",
    _ => "error",
};
```
*From: D60.*

**5.10.4** `loop { B }` repeats `B` until a `break` leaves it. Its type is
`never` if no `break` targets it; otherwise the types of the `break` operands
targeting it are unified (a `break` without operand gives `()`).

```tally
let first_even = loop {
    let n = next();
    if n % 2 == 0 { break n; }
};
```
*From: D10, D53.*

**5.10.5** `while c { B }` evaluates `c` and, while it is `true`, runs `B`.
Its type is `()`. A `break` targeting a `while` or `for` loop must not have an
operand.

```tally
var i = 0;
while i < 10 { i += 1; }
```
*From: D10.*

**5.10.6** `for p in e { B }` evaluates `e`, converts it to an iterator with
`IntoIterator.into_iter` (19.10.1), and then behaves as:

```tally
{
    var __it = IntoIterator.into_iter(e);
    loop {
        match __it.next() {
            .Some(p) => { B }
            .None => break,
        }
    }
}
```

where `__it` is not nameable. `p` must be irrefutable. The `for` expression
has type `()`. Its effects include those of `into_iter`, `next` and the
iterator's drop (10.3).

```tally
for (i, x) in v.iter().enumerate() { print(i, ": ", x); }
for c in "abc".chars() { ... }
for k in 0..10 { ... }
```
*From: D27, D60.*

**5.10.7** A label `name:` before a block, `loop`, `while` or `for` names that
expression. `break :name` and `continue :name` target the labelled expression;
`continue` may target only loops. An unlabelled `break` or `continue` targets
the innermost enclosing `loop`, `while` or `for`; it never targets a labelled
block. `break :name v` from a labelled block makes `v` the block's value.
`break` and `continue` cannot cross a closure boundary. Labels within one
function must be distinct where they are nested.

```tally
let found = scan: {
    for row in grid {
        for x in row { if x == target { break :scan true; } }
    }
    false
};
```
*From: D53.*

## 5.11 Ranges

**5.11.1** The range expressions build values of the `core.ops` range types:
`a..b` is `Range[T]`, `a..=b` is `RangeInclusive[T]`, `a..` is
`RangeFrom[T]`, `..b` is `RangeTo[T]`, `..=b` is `RangeToInclusive[T]`, and
`..` is `RangeFull`. Both operands must have the same type `T`. Ranges of
integer types and `char` are iterators (19.10.4). A range is empty when its
start is not less than (for `..=`: greater than) its end; building such a
range never panics.

```tally
for i in 0..n { ... }            // 0, 1, ..., n-1
for c in 'a'..='z' { ... }
let r = 5..2;                     // empty, no panic
```
*From: D2.*

## 5.12 Literals of compound types

**5.12.1** A struct literal `S { f1: e1, ..., fn: en }` builds a value of
struct `S` (or of a struct-like variant). Every field must be given exactly
once, unless `..base` is present; then the fields not given are moved or
copied out of `base`, which must have type `S`. A field initialiser `f` alone
means `f: f`. Fields not visible at the literal's location cannot be given,
so a struct with a field not visible here can be built here only through
functions.

```tally
let p = Point { x: 1.0, y: 2.0 };
let x = 3.0;
let q = Point { x, ..p };        // y copied from p
let c = Conn { id: 1, fd: 3 };   // error outside the package: `fd` is not visible
```
*From: D18, D22.*

**5.12.2** An array literal `[e1, ..., en]` builds a `[n]T` where every
element has type `T`. `[e; N]` evaluates `e` once and builds a `[N]T` of N
copies; `e`'s type must be `Copy` and `N` a constant expression of type
`usize`. A tuple literal `(e1, ..., en)` builds a tuple; `(e,)` is a
one-element tuple and `(e)` is just `e`.

```tally
let a = [1, 2, 3];
let z: [256]u8 = [0; 256];
let s = [String.new(); 2];     // error: String is not Copy
```
*From: D22.*

## 5.13 `orelse`

**5.13.1** `a orelse b` requires `a` to have type `?T` or `T ! E`. If `a` is
`.Some(v)` or `.Ok(v)`, the result is `v` and `b` is not evaluated. Otherwise
`b` is evaluated and is the result; for a result type, the error value is
dropped first. `b` must have type `T` or `never`. `orelse` is
right-associative.

```tally
let port = env_port() orelse cfg_port() orelse 8080;
let n = parse(s) orelse return .None;     // b of type never
let v = map.get(&k) orelse { log("missing"); &default };
```
*From: D19, D20.*

## 5.14 References and borrows

**5.14.1** `&e` and `&mut e` create a shared or exclusive reference to the
place `e`. If `e` is a value expression, a temporary holding its value is
created and the reference refers to it (8.4). `&mut e` requires `e` to be a
mutable place (5.9.4) or a temporary. The validity of the reference is
checked by the borrow checker (chapter 9). Borrowing never has effects.

```tally
var v = [1, 2, 3];
let r = &v[0];
let m = &mut v[1];        // error while r is still used (9.3.2)
let t = &compute();       // reference to a temporary (8.4.2)
```
*From: D24.*

## 5.15 Unsafe blocks

**5.15.1** `unsafe { B }` evaluates `B` and permits the unsafe operations of
13.1.2 inside `B` (excluding the bodies of closures in `B`). It has `B`'s
value and type. `unsafe may E { B }` additionally asserts that `B`'s effects
are within `E` and gives the block the effect set `E` regardless of what
the effect checker finds in `B` (10.6.1).

```tally
let x = unsafe { *ptr };
let n = unsafe may none { v.push_unchecked(x) };   // caller vouches: no alloc happens
```
*From: D36, D3.*
