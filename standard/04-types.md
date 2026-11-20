# 4. Types

## 4.1 Overview

**4.1.1** Every value and every expression has exactly one type, determined at
compile time. Types are either *primitive* (integers, `bool`, `char`, floats,
`()`, `never`), *constructed* from other types (tuples, arrays, slices,
references, raw pointers, function pointers, trait objects, optional and
result types), or *nominal* (declared structs, enums and unions, and the
anonymous types of functions and closures).

```tally
let a: u8 = 1;                  // primitive
let b: (u8, &str) = (1, "x");   // constructed
let c: Point = Point { x: 0.0, y: 0.0 };   // nominal
```
*From: D2.*

**4.1.2** A type is *sized* if all its values have the same size, known at
compile time. Every type is sized except `[]T`, `str` and `dyn Trait`. An
unsized type may appear only as the target of `&`, `&mut`, `*`, `*mut`, as
the type argument of `Box` and the other library types that accept
`?Sized` arguments (7.2.5), or as the argument of a generic parameter
declared `?Sized`. Local variables, parameters, fields, array elements and
return values must have sized types.

```tally
let s: &str = "hi";          // fine: str behind a reference
fn bad(x: str) {}            // error: parameter of unsized type
let b: Box[[]u8] = ...;      // fine: Box accepts ?Sized
```
*From: D59.*

## 4.2 Integer types

**4.2.1** For every N from 1 to 128, `uN` is the unsigned integer type of N
bits, with values 0 to 2^N − 1, and `iN` is the signed integer type of N
bits, with values −2^(N−1) to 2^(N−1) − 1. `usize` and `isize` are the
unsigned and signed integer types with the width of a data pointer on the
target, which must be 16, 32 or 64 bits. `usize` and `isize` are distinct
types from the fixed-width types of the same width.

```tally
let flags: u3 = 0b101;
let delta: i12 = -2048;
let n: usize = v.len();
let m: u64 = n;          // implicit widening: usize is at most 64 bits on every target
```
*From: D14.*

**4.2.2** The *storage size* of an integer type of N bits is the smallest of
1, 2, 4, 8 and 16 bytes that holds N bits. Its alignment equals its storage
size, except that on targets whose C ABI aligns 128-bit integers to 8 bytes,
the alignment of `u65`–`u128` and `i65`–`i128` is still 16. When stored in
memory, the value occupies the low N bits of the storage in the target's byte
order, and the remaining high bits hold the zero extension (unsigned) or sign
extension (signed) of the value. A bit pattern whose high bits differ from
that extension is not a valid value (13.4.2).

```tally
// mem.size_of.[u3]() == 1, mem.size_of.[u24]() == 4, mem.align_of.[u128]() == 16
// The u3 value 5 is stored as the byte 0b0000_0101.
// The i3 value -1 is stored as the byte 0b1111_1111.
```
*From: D14, D37.*

**4.2.3** Integers are represented in two's complement. The arithmetic and
conversion operators on integers are defined in 5.3 and 5.4.

```tally
let x: i8 = -1;
let bits = x as% u8;      // 255: same two's-complement bit pattern
```
*From: D14, D16.*

**4.2.4** An integer literal has a type determined by type inference (4.13).
If inference leaves it undetermined, its type is `i64`. If it is determined to
be an integer type, its value must be representable in that type, or the
program has an error. If `-` is applied directly to an integer literal, the
range check uses the negated value, so `-128` is valid for `i8`.

```tally
let a = 7;               // i64
let b: u8 = 255;         // fine
let c: u8 = 256;         // error: 256 not representable in u8
let d: i8 = -128;        // fine
let e = [1, 2, 3];       // [3]i64
```
*From: D14.*

**4.2.5** An integer literal may also be determined to have a floating-point
type. It then denotes the float value equal to the integer, which must be
exactly representable, or the program has an error.

```tally
let half: f64 = 1 / 2;          // both literals are f64: 0.5
let big: f32 = 16777217;        // error: not exactly representable in f32
```
*From: D14, D16.*

## 4.3 Booleans and characters

**4.3.1** `bool` has the values `true` and `false`. It occupies one byte with
value 1 for `true` and 0 for `false`; other byte values are not valid values
of `bool`.

```tally
let ok: bool = n > 0;
// mem.size_of.[bool]() == 1
```
*From: D2, D37.*

**4.3.2** `char` holds one Unicode scalar value: an integer in 0x0–0xD7FF or
0xE000–0x10FFFF. It occupies four bytes, aligned to four, holding the value as
a `u32`. Other `u32` values are not valid values of `char`.

```tally
let c: char = 'λ';
let n = c as u32;              // 955
let d = 0xD800 as char;        // panics: not a scalar value (5.4.3)
```
*From: D57.*

## 4.4 Floating-point types

**4.4.1** `f32` and `f64` are the IEEE 754-2019 binary32 and binary64 formats.
Their size and alignment are 4 and 8 bytes.

```tally
let x: f32 = 1.5;
let y: f64 = x;          // implicit widening (4.11.1, rule 6)
```
*From: D2.*

**4.4.2** Float arithmetic follows IEEE 754-2019 with rounding to nearest,
ties to even, and default exception handling: it never panics. The
implementation must not reassociate operations, contract `a * b + c` into a
fused multiply-add, or treat NaN or infinity specially except as IEEE 754
requires. When an operation produces a NaN, its sign and payload are
unspecified. `%` on floats computes the remainder with the sign of the
dividend, as C's `fmod`.

```tally
let inf = 1.0 / 0.0;          // +infinity, no panic
let r = -7.5 % 2.0;           // -1.5
let fma = a.mul_add(b, c);    // explicit fused multiply-add
```
*From: D2, D15.*

**4.4.3** A float literal has a type determined by inference. If inference
leaves it undetermined, its type is `f64`. A float literal cannot have an
integer type. The literal is rounded to the nearest representable value,
ties to even.

```tally
let a = 0.1;           // f64
let b: f32 = 0.1;      // nearest f32 to 0.1
let c: u32 = 1.0;      // error: float literal cannot have an integer type
```
*From: D14.*

## 4.5 Unit, tuples and never

**4.5.1** `()` is the unit type. Its only value is written `()`. It has size 0
and alignment 1. A function with no return clause returns `()`.

```tally
fn tick() {}             // returns ()
let u: () = tick();
```
*From: D10.*

**4.5.2** A tuple type `(T1, T2, ..., Tn)` with n ≥ 1 holds one value of each
element type. A one-element tuple is written `(T,)`. Elements are accessed
with `.0`, `.1`, and so on. Tuples are laid out as a struct with fields in
element order (13.3.2).

```tally
let pair: (u8, bool) = (1, true);
let single: (u32,) = (5,);
let first = pair.0;
```
*From: D12, D37.*

**4.5.3** `never` is the type of expressions that never produce a value:
`return`, `break`, `continue`, a call to a function returning `never`, and
`loop` without a reachable `break`. It has no values, size 0, and alignment 1.
An expression of type `never` coerces to any type (4.11.1).

```tally
fn fail(msg: &str) -> never may panic { panic(msg) }
let n: u32 = if ok { 1 } else { fail("no") };   // never coerces to u32
```
*From: D19, D21.*

## 4.6 Optional and result types

**4.6.1** `?T` is the type `core.option.Option[T]`, an enum with variants
`None` and `Some(T)`. The two spellings denote the same type.

```tally
let a: ?u32 = .Some(3);
let b: Option[u32] = .None;
let c: ?u32 = 3;          // implicit wrapping (4.11.1)
```
*From: D19.*

**4.6.2** `T ! E` is the type `core.result.Result[T, E]`, an enum with
variants `Ok(T)` and `Err(E)`. The two spellings denote the same type. A
value of a result type must be used (5.9.2).

```tally
fn parse(s: &str) -> u32 ! ParseError { ... }
let r: Result[u32, ParseError] = parse("12");
let n = parse("12")?;      // propagate the error (12.2.1)
```
*From: D19, D20.*

## 4.7 Arrays, slices and strings

**4.7.1** `[N]T` is the type of arrays of exactly N values of type `T`, where
N is a compile-time constant expression of type `usize` (16.1). The size of
`[N]T` is N times the size of `T`; its alignment is that of `T`. N may be 0.

```tally
const W: usize = 8;
let row: [W]u16 = [0; W];
let empty: [0]u8 = [];
```
*From: D12, D37.*

**4.7.2** `[]T` is the unsized type of a contiguous sequence of zero or more
values of type `T` whose length is known only at run time. A reference or raw
pointer to `[]T` holds the address of the first element and the length.

```tally
fn sum(xs: &[]u32) -> u64 may panic {
    var s: u64 = 0;
    for x in xs { s += *x; }
    s
}
let arr = [1, 2, 3];
let t = sum(&arr);          // &[3]u32 coerces to &[]u32
```
*From: D12, D59.*

**4.7.3** `str` is the unsized type of a sequence of bytes that is valid UTF-8.
A `&str` holds an address and a length in bytes. Every safe operation that
produces a `str` maintains validity; creating an invalid `str` in unsafe code
is undefined behaviour (13.4.2).

```tally
let s: &str = "añb";
let n = s.len();           // 4 bytes
let bytes: &[]u8 = s.as_bytes();
```
*From: D57.*

## 4.8 References and pointers

**4.8.1** `&T` is a shared reference to a `T` and `&mut T` an exclusive
reference. A reference is never null, is always aligned for `T`, and always
points to a valid value of `T` for as long as it can be used. The rules that
make this true are in chapter 9. For sized `T` a reference has the size and
alignment of `usize`; for `[]T` and `str` it is two `usize` words (address,
length); for `dyn Trait` it is two pointer words (address, vtable).

```tally
let x = 5;
let r: &i64 = &x;
var y = 6;
let m: &mut i64 = &mut y;
*m += 1;
```
*From: D24.*

**4.8.2** `*T` and `*mut T` are raw pointers. A raw pointer is never null but
carries no other guarantee; dereferencing one is unsafe (13.1.2). A nullable
raw pointer is written `?*T` and has the same size as `*T`, with `.None`
represented by address 0 (13.3.6). Raw pointers to unsized types have the same
layout as references to them.

```tally
let p: *u8 = &buf[0];          // coercion; creating a raw pointer is safe
let c: ?*u8 = .None;           // a C null pointer
let v = unsafe { *p };         // dereference requires unsafe
```
*From: D36, D38.*

## 4.9 Function types

**4.9.1** Every function declaration, and every distinct instantiation of a
generic function, has its own *function item type*. It has size 0 and no
name that can be written. A function item coerces to a function pointer type
whose signature it satisfies (4.11.1).

```tally
fn double(x: u32) -> u32 may panic { x * 2 }
let f = double;                                // function item type, size 0
let g: fn(u32) -> u32 may panic = double;      // coerced to a pointer
```
*From: D2, D27.*

**4.9.2** A function pointer type `fn(P1, ..., Pn) -> R may E` holds the
address of a function. It is never null; use `?fn(...)` for a nullable one.
Its size and alignment are those of a code pointer on the target. An
`unsafe fn` pointer may point to unsafe or safe functions; a safe `fn` pointer
may not point to an unsafe function. An `extern "C"` pointer uses the C calling
convention (14.1.2). An absent effect clause on a function pointer type means
`may none`.

```tally
let cb: extern "C" fn(i32) -> i32 may sys = c_handler;
let h: ?fn() = .None;
let p: fn() = noisy;     // error if `noisy` has any effect: fn() means `may none`
```
*From: D31, D38.*

**4.9.3** Each closure expression has its own anonymous *closure type*, which
holds its captured values (5.8). Closure types implement `FnOnce`, and
possibly `FnMut` and `Fn` (7.7.2). A closure that captures nothing also
coerces to a function pointer type it satisfies.

```tally
let k = 3;
let add_k = |x: u32| x + k;              // captures k by shared reference
let plain: fn(u32) -> u32 may panic = |x| x + 1;   // captures nothing
```
*From: D54.*

## 4.10 Nominal types

**4.10.1** A `struct` declaration defines a new type with named fields, with
positional fields (tuple struct), or with no fields (unit struct). Two struct
types are the same type only if they come from the same declaration with the
same generic arguments. A tuple struct's name is also a function that builds
it; a unit struct's name is also its only value.

```tally
struct Meters(f64);
struct Feet(f64);
let m = Meters(3.0);
let f: Feet = m;           // error: Meters and Feet are different types
struct Unit;
let u: Unit = Unit;
```
*From: D17, D18.*

**4.10.2** A type *mentions a reference* if it is a reference type, or is a
`view` struct or `view` enum, or is a constructed type (tuple, array, slice,
optional, result) or a nominal type with a generic argument any of whose
component types or arguments mentions a reference, or is a closure type that
captures by reference. Function pointer types and `dyn` types never mention a
reference: the regions in their signatures are higher-ranked (9.5.7), and a
`dyn` type's own region is set by 9.5.6. A struct or enum whose field or variant types, as written in the
declaration, mention a reference must be declared `view`; a `view` declaration
whose fields mention no reference is an error. Generic type parameters do not
count: `Vec[T]` is not declared `view`, but `Vec[&u8]` mentions a reference
because its argument does. The borrowing rules for these types are in 9.5.

```tally
view struct Token { text: &str, line: u32 }        // must be `view`
struct Owned { text: String }                      // must not be `view`
struct Bad { r: &u8 }                              // error: field mentions a reference; declare `view struct`
view enum Arg { Flag(&str), Count(u32) }
```
*From: D24.*

**4.10.3** A struct type may not contain itself except through a pointer
type; a type with infinite size is an error.

```tally
struct Node { value: u32, next: ?Box[Node] }    // fine: indirection
struct Loop { inner: Loop }                     // error: infinite size
```
*From: D2.*

**4.10.4** An `enum` declaration defines a tagged union: each value is
exactly one of the declared variants, with that variant's fields. Variants are
unit (`A`), tuple-like (`B(u8, u8)`) or struct-like (`C { x: u8 }`). An enum
with no variants has no values and behaves like `never` except that it does
not coerce.

```tally
enum Shape { Circle(f64), Rect { w: f64, h: f64 }, Empty }
let s = Shape.Rect { w: 1.0, h: 2.0 };
let t: Shape = .Circle(0.5);     // inferred variant
enum Void {}
```
*From: D19.*

**4.10.5** An enum may declare a *tag type* with `: T`, where `T` is an integer
type. Variants may then be given explicit tag values with `= expr`, a
compile-time constant of type `T`. Variants without an explicit value take
the previous variant's value plus one, or 0 for the first. Tag values must be
distinct and representable in `T`. Explicit tag values without a declared tag
type are an error. An enum whose variants are all unit variants and that
declares a tag type converts with `as` to its tag type and to any other
integer type (5.4.5).

```tally
enum Op: u8 { Nop = 0x90, Ret = 0xC3, Int3 = 0xCC }
let byte = Op.Ret as u8;       // 0xC3
enum Bad { A = 1 }             // error: explicit value needs a tag type
```
*From: D37, D16.*

**4.10.6** A `union` declaration defines an untagged union: all fields share
the same storage. Every field type must be `Copy`. Writing a field is safe;
reading a field is an unsafe operation (13.1.2). A union is never `view`; a
union field may not mention a reference.

```tally
union Word { u: u32, f: f32, bytes: [4]u8 }
var w = Word { u: 0x3f80_0000 };
let x = unsafe { w.f };        // 1.0
```
*From: D36, D38.*

## 4.11 Coercions and subtyping

**4.11.1** A *coercion site* is: the initialiser of a `let` or `var` with a
type annotation; the right side of an assignment; a function or method
argument, against the parameter's type; a `return` operand and a function's
tail expression, against the return type; a struct literal field, against the
field type; an array literal element, against the expected element type; a
`break` operand, against the loop's or block's type when known; a match arm or
`if` branch value, against the expression's expected type when known; a tuple
literal element, against the expected element type; a `const` or `static`
initialiser, against the declared type; a closure body, against its declared
return type. At a coercion site the actual type is first unified with the
expected type; if unification fails, the following coercions are tried, in
this order (an integer- or float-literal variable checked against `?T` or
`T ! E` is unified with `T` and wrapped by rule 7):

1. `never` to any type;
2. `&mut T` to `&T`; `&T` or `&mut T` to `*T`; `&mut T` to `*mut T`;
   `*mut T` to `*T`;
3. unsizing: `&[N]T` to `&[]T`, `&mut [N]T` to `&mut []T`, and the same for
   raw pointers and `Box`; `&U`, `&mut U`, `*U`, `*mut U` or `Box[U]` to the
   same pointer to `dyn Tr` when `U` implements `Tr` and `U`'s associated
   effects are within the `dyn` type's bindings (7.8.3);
4. deref coercion: `&U` to `&T` and `&mut U` to `&mut T` when `U` derefs to
   `T` in one or more steps through `Deref`/`DerefMut` (7.6.4);
5. function item or capture-free closure to a function pointer type whose
   signature it satisfies; function pointer `fn(..) -> R may E1` to
   `fn(..) -> R may E2` when E1 ⊆ E2;
6. integer widening (4.11.2) and `f32` to `f64`;
7. `T` to `?T` (as `.Some`) and `T` to `T ! E` (as `.Ok`).

If none applies the program has an error. Coercions do not chain except that
rule 7 may follow one of rules 1–6 (`&mut T` to `?&T`).

```tally
fn takes(s: &[]u8, o: ?&u32, w: &mut dyn Write may sys | block) { ... }
var arr: [4]u8 = [1; 4];
takes(&arr, &n, &mut file);            // unsize; &u32 to ?&u32; &mut File to &mut dyn Write
let s: &str = &string;                 // deref coercion: &String to &str
let w: u64 = small_u16;                // widening
```
*From: D16, D19, D58.*

**4.11.2** *Integer widening* converts an integer type S to an integer type
T when every value of S is representable in T on every permitted target, that
is, treating `usize` and `isize` as possibly 16, 32 or 64 bits wide (4.2.1).
Thus `uN` widens to `uM` and `iN` to `iM` when M > N, and `uN` to `iM` when
M > N; `u8`, `u16` widen to `usize` and `u8`, `i8`, `i16` to `isize`; `usize`
widens to `u64` and `u128`, and to `i128`; `isize` widens to `i64` and `i128`.
`u32` does not widen to `usize`, because the program must mean the same thing
on 16-bit targets. No other implicit numeric conversion exists. Widening never
happens between the operands of a binary operator.

```tally
let a: u8 = 200;
let b: u32 = a;            // widening at a coercion site
let c = a + b;             // error: operands have different types u8 and u32
let d = (a as u32) + b;    // fine
let i: usize = a;          // fine on every target
let j: usize = b;          // error: write `b as usize`
```
*From: D16, D14.*

**4.11.3** Subtyping exists only for regions (9.4) and effect sets: a
function pointer or `dyn` type with a smaller effect set is a subtype of the
same type with a larger effect set. Subtyping is applied implicitly wherever
types must be equal, through the variance of each type position (9.4.5).
There is no subtyping between nominal types.

```tally
let f: fn() may none = quiet;
let g: fn() may panic = f;       // fine: none ⊆ panic
let v: Vec[fn() may panic] = Vec.new();
v.push(f)?;                      // fine
```
*From: D27, D30.*

## 4.12 Type aliases and generic instantiation

**4.12.1** `type A[P...] = T;` declares `A` as another name for `T`. An alias
is fully transparent: `A[args]` is the same type as `T` with the arguments
substituted. An alias may not refer to itself directly or indirectly.

```tally
type Pair[T] = (T, T);
let p: Pair[u8] = (1, 2);
let q: (u8, u8) = p;        // same type
type R = ?R;                // error: recursive alias
```
*From: D2.*

**4.12.2** A generic type is instantiated by giving one argument per
parameter, in order. Trailing parameters with defaults may be omitted. Type
parameters take types; `const` parameters take compile-time constants of their
declared type (16.1); `effect` parameters take effect expressions (10.2.3).
Two instantiations are the same type if their arguments are pairwise the
same: equal types, equal constant values, and equal effect sets.

```tally
struct Ring[T, const N: usize = 64] { ... }
let a: Ring[u8] = ...;           // Ring[u8, 64]
let b: Ring[u8, 64] = a;         // same type
let c: Ring[u8, 32] = a;         // error: different types
```
*From: D13, D30.*

## 4.13 Type inference

**4.13.1** Types are inferred only within one function body (including the
bodies of closures in it), and within the initialiser of a constant or static.
The types of all items' signatures, struct fields, constants and statics must
be written in full; an unwritten type in those positions is an error.

```tally
fn f(x) {}                         // error: parameter type required
const K = 3;                       // error: constant type required
```
*From: D63.*

**4.13.2** Inference proceeds through a function body in source order,
assigning each expression a type that may contain *inference variables*.
There are four kinds of variable: general variables, integer-literal
variables (which may be resolved only to an integer or float type),
float-literal variables (which may be resolved only to a float type), and
const variables (which stand for the value of a `const` generic parameter and
are resolved by unifying types that contain them, such as `[N]u8` with
`[4]u8`). Each
expression is checked against an *expected type* when its context provides
one (a coercion site with a known target, an annotated binding, a parameter),
and otherwise its type is inferred and then *unified* with what the context
requires. Unification makes two types equal by resolving variables; it fails
if two different type constructors meet. At coercion sites (4.11.1), if both
the expected and the actual type have a known outermost constructor and they
differ, a coercion is attempted instead of unification.

```tally
var v = Vec.new();         // Vec[?a]
v.push(7)?;                // ?a unified with integer-literal variable
let w: u16 = v[0];         // resolves to u16
```
*From: D63, D14.*

**4.13.3** The following expressions require the outermost constructor of
their operand's type to be known (not an unresolved variable) at the point
where they are checked: field access, method call, indexing, `?`, `orelse`,
dereference, `for` iteration, and pattern matching against an enum variant
with an inferred type (`.Name`). If it is not known, the program has an error
("type must be known here"); the programmer adds an annotation.

```tally
let v = Vec.new();
let n = v.len();       // fine: Vec is known, its element type need not be
let x = parse(s)?;     // fine if parse's return type is declared
fn g[T: Default]() -> T { T.default() }
let y = g();
y.len();               // error: type of y must be known here; write `let y: String = g();`
```
*From: D63, D2.*

**4.13.4** When a generic function or type is used without explicit
arguments, a fresh general variable is created for each type parameter and
each effect parameter, and a const variable for each `const` parameter; they
are resolved by unification with the argument types and the expected type of
the call. Trait bounds on the parameters are checked after the body has
been inferred, when the variables are resolved.

```tally
fn max[T: Ord](a: T, b: T) -> T { if a.cmp(&b) == .Less { b } else { a } }
let m = max(3, 9);         // T = i64 (literal default)
let k: u8 = max(3, 9);     // T = u8
```
*From: D63, D27.*

**4.13.5** At the end of a body, every integer-literal variable still
unresolved becomes `i64`, every float-literal variable `f64`, and every
remaining general variable is an error. Literal range checks (4.2.4) and
trait-bound checks are then performed.

```tally
let a = 1 << 40;           // i64
let v = Vec.new();         // error: element type of v cannot be inferred
```
*From: D14, D63.*
