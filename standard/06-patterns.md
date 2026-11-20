# 6. Patterns

## 6.1 Where patterns appear

**6.1.1** Patterns appear in `let`/`var` statements, `let ... else`, `if let`,
`while let`, `for`, and `match` arms. A pattern is *irrefutable* if it matches
every value of its type, and *refutable* otherwise. `let`, `var` and `for`
require irrefutable patterns; `let ... else`, `if let`, `while let` and `match`
accept refutable ones. Function and closure parameters are names, not
patterns.

```tally
let (a, b) = pair;                     // irrefutable
let .Some(x) = opt;                    // error: refutable pattern in let; use let-else
let .Some(x) = opt else { return; };   // fine
```
*From: D60.*

**6.1.2** Matching a pattern against a value either succeeds, producing
bindings, or fails. Matching is tried from left to right in the pattern, has
no effects, and evaluates no user code.

```tally
match pt { Point { x: 0, y } => on_axis(y), _ => other() }   // no calls during matching
```
*From: D60, D2.*

## 6.2 Pattern forms

**6.2.1** An identifier pattern `x` matches any value and binds it to the new
name `x`; `var x` binds a mutable binding. A bare identifier in a pattern is
*always* a new binding, even if a constant, unit struct or unit variant of
that name is in scope; to match against those, use a path with a `.` (6.2.4)
or a guard.

```tally
const ZERO: u32 = 0;
match n {
    ZERO => a(),          // a binding named ZERO that matches everything (and shadows)
    _ => b(),             // error: unreachable arm (6.4.2)
}
match n { k if k == ZERO => a(), _ => b() }    // intended
```
*From: D60, D2.*

**6.2.2** `_` matches any value and binds nothing. Matching against `_` does
not move or borrow the value.

```tally
let _ = take_lock();     // the guard is a temporary, dropped at end of statement
let (x, _) = pair;
```
*From: D60, D25.*

**6.2.3** A literal pattern matches a value equal to the literal (by `==`).
Integer, character, byte, boolean, string and byte string literals are
allowed; a `-` may precede integer literals. Float literals are not allowed in
patterns. A string literal pattern matches `&str` (or `str` behind a
reference, through binding modes) with equal contents; a byte string literal
matches `&[N]u8` or `&[]u8`.

```tally
match cmd {
    "quit" | "exit" => stop(),
    "" => {},
    other => run(other),
}
match x { 1.0 => ..., _ => ... }     // error: float literal pattern
```
*From: D60, D17.*

**6.2.4** A path pattern contains at least one `.`. It names a unit variant
(`.None`, `Op.Ret`), a constant (`limits.MAX`, `Self.EMPTY`), or, followed by
`( ... )` or `{ ... }`, a tuple-like or struct-like variant or a tuple or
named struct. A constant pattern matches values equal to the constant; the
constant's type must be comparable (5.5.1).

```tally
match t {
    .Eof => done(),
    Token.Num(n) => push(n),
    .Ident { name, .. } => lookup(name),
}
match mode { config.DEFAULT_MODE => ..., _ => ... }
```
*From: D60.*

**6.2.5** A range pattern `a..=b` matches values `v` with `a <= v <= b`;
`a..b` matches `a <= v < b`. Bounds are literals or constant paths of the same
integer or `char` type. An empty range pattern (`b < a` for `..=`, `b <= a`
for `..`) is an error.

```tally
match c {
    'a'..='z' | 'A'..='Z' => letter(),
    '0'..='9' => digit(),
    _ => other(),
}
match n { 10..=1 => ..., _ => ... }     // error: empty range pattern
```
*From: D60.*

**6.2.6** A tuple pattern `(p1, ..., pn)` matches a tuple of the same arity
whose elements match. A single `..` element stands for any number of elements
not otherwise matched; it binds nothing. Tuple-struct and tuple-variant
patterns `P(p1, ...)` work the same way on their fields.

```tally
let (first, .., last) = (1, 2, 3, 4);
match ev { .Key(code, ..) => press(code), _ => {} }
```
*From: D60.*

**6.2.7** A struct pattern `S { f: p, g, .. }` matches a struct or
struct-like variant of type `S` whose named fields match. `g` alone means
`g: g`; `var g` means `g: var g`. Every field must be mentioned unless `..`
ends the list. A field that is not visible at the pattern's location cannot
be mentioned; a struct with such a field can only be matched with `..`.

```tally
let Point { x, y: var y2 } = p;
match shape { .Rect { w, h } if w == h => square(w), .Rect { .. } => rect(), _ => {} }
```
*From: D60, D18.*

**6.2.8** A slice pattern `[p1, ..., pn]` matches an array `[N]T` or a slice
`[]T`. Without a `..` element it matches only values of length exactly `n`;
with one `..` element it matches values of length at least `n − 1`. `..name`
binds the middle part: for an array, as an array of the remaining length; for
a slice, as a sub-slice (which requires a reference binding mode, 6.3.1). For
an array, a pattern whose length cannot match is an error.

```tally
fn head(xs: &[]u32) -> ?u32 {
    match xs { [] => .None, [h, ..] => *h }
}
match bytes { [0x7f, b'E', b'L', b'F', ..rest] => elf(rest), _ => other() }
```
*From: D60.*

**6.2.9** A reference pattern `&p` or `&mut p` matches a value of type `&T`
or `&mut T` respectively and matches `p` against the referenced value. It is
allowed only when the current default binding mode is *move* (6.3.1).
Binding the referenced value by move through `&p` is allowed only for `Copy`
types.

```tally
for &x in &[1, 2, 3] { total += x; }    // x: i64, copied
```
*From: D60, D22.*

**6.2.10** An or-pattern `p | q` matches if `p` or `q` matches, trying them in
order. Each alternative must bind exactly the same names, with the same types,
mutability and binding modes.

```tally
match c { 'y' | 'Y' => yes(), _ => no() }
match e { .A(x) | .B(x) => use(x), .C => {} }      // x bound in both
match e { .A(x) | .C => ... }                     // error: x not bound in every alternative
```
*From: D60.*

## 6.3 Binding modes

**6.3.1** Each position in a pattern is matched with a *default binding
mode*, which is *move*, *ref* or *ref mut*. Matching starts in mode *move*.
When a pattern that is not an identifier, `_`, or reference pattern is matched
against a value of type `&T` (or `&mut T`), the value is dereferenced
automatically and the mode becomes *ref* (or *ref mut*, unless it was already
*ref*). An identifier pattern in mode *move* binds the value itself (moving or
copying it); in mode *ref* it binds a `&` reference to it; in mode *ref mut* a
`&mut` reference. `var x` in mode *ref* or *ref mut* is an error.

```tally
let opt: &?String = &maybe_name;
match opt {
    .Some(s) => print(s),       // s: &String (mode ref)
    .None => {},
}
let pair: &mut (u32, u32) = &mut p;
let (a, b) = pair;              // a, b: &mut u32
*a += 1;
```
*From: D60.*

**6.3.2** If any binding in move mode binds a value whose type is not `Copy`,
the whole scrutinee is moved by the match (it is a use by value of the
scrutinee place, 8.2.1), even if other parts are not bound. A match against a
place whose arms bind nothing by move does not move it.

```tally
let o: ?String = get();
match o { .Some(s) => keep(s), .None => {} }   // moves o
let n = o;                                      // error: o was moved
match o2 { .Some(_) => {}, .None => {} }         // does not move o2
```
*From: D23, D60.*

## 6.4 Exhaustiveness and reachability

**6.4.1** The arms of a `match` must be *exhaustive*: every value of the
scrutinee's type must be matched by at least one arm without a guard. For
this purpose the values of a type are: all values of integer types and
`char` in their range; both `bool` values; for floats, `str` and slices of
non-fixed length, infinitely many values, so only an irrefutable arm or an
arm per length class can cover them; every variant of an enum with all
values of its fields; every combination of field values of tuples, structs
and arrays; every slice length with every element value; for references, the
values of the referenced type. A type with no values (`never`, an enum
without variants) needs no arms. Implementations decide exhaustiveness with
the usefulness algorithm of Maranget (2007), or any algorithm with the same
results.

```tally
match b { true => 1, false => 0 }             // exhaustive
match n { 0..=127 => lo(), 128..=255 => hi() } // exhaustive for u8
match n { 0..=127 => lo() }                   // error: 128..=255 not covered (for u8)
match xs { [] => a(), [_] => b(), [_, _, ..] => c() }   // exhaustive for &[]T
```
*From: D60, D2.*

**6.4.2** An arm whose pattern can match no value not already matched by
earlier arms without guards is *unreachable*, and is an error. Likewise each
alternative of an or-pattern must be able to match some value not matched by
earlier alternatives or arms.

```tally
match x { _ => a(), 0 => b() }        // error: arm `0` is unreachable
match c { 'a' | 'a' => ... , _ => ... }   // error: second alternative unreachable
```
*From: D60, D2.*

**6.4.3** An enum declared with `@non_exhaustive` (16.4.4) is treated, outside
the package that declares it, as if it had additional unknown variants: a
`match` on it in another package must have an arm that matches every value
(for example `_`). Inside its own package it is matched normally.

```tally
// package net:
@non_exhaustive pub enum Proto { Tcp, Udp }
// another package:
match p { .Tcp => t(), .Udp => u() }            // error: add an arm for future variants
match p { .Tcp => t(), .Udp => u(), _ => other() }
```
*From: D60, D44.*
