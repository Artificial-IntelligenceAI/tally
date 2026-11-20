# 19. The core library

`core` is the part of the library that every Tally program can use,
including freestanding programs. It never makes system calls of its own: no
function in `core` has the `sys` or `block` effect except through a type or
callable argument, the volatile operations of 19.4.3, `core.context` and
`core.stack.remaining` (19.16), and `core.process.abort` (19.18.2). It includes allocating collections, which use the program's global
allocator (19.13.5) or an allocator given as a type argument.

Declarations below are written in Tally. A body written `{ ... }`, and a method
declaration ending in `;` inside an `impl` (written that way for brevity), is
supplied by the implementation with the behaviour described. `impl I` and
`impl F` stand for one impl for each integer or float type. Every listed item is
`pub`. Effects shown are exact: an implementation must not add effects, and
must not declare fewer where that would change what programs are accepted.
Generic parameters named `A` are always bounded by `Allocator`.

## 19.1 Organisation and prelude

**19.1.1** `core` consists of the modules `core.alloc`, `core.atomic`,
`core.cell`, `core.cmp`, `core.collections`, `core.context`,
`core.default`, `core.endian`, `core.error`, `core.ffi`, `core.fmt`,
`core.hash`, `core.hint`, `core.iter`, `core.marker`, `core.math`, `core.mem`,
`core.num`, `core.ops`, `core.option`, `core.panic`, `core.process`,
`core.ptr`, `core.result`, `core.slice`, `core.source`, `core.stack`,
`core.text`, `core.chars`, `core.target`, `core.time`. (The modules for
string and character functions are named `text` and `chars` so that they do
not shadow the primitive types `str` and `char`.) The root module `core`
re-exports nothing.

```tally
import core.mem;
import core.collections.{HashMap, HashSet};
```
*From: D46.*

**19.1.2** The *prelude* is a set of names that every module can use without
importing, as if declared in the predeclared scope (3.4.1): the types
`Option`, `Result`, `Box`, `Vec`, `String`, `AllocError`, `Global`; the traits
`Copy`, `Clone`, `Drop`, `Send`, `Sync`, `Sized`, `Eq`, `Ord`, `Hash`,
`Default`, `Debug`, `Display`, `Iterator`, `IntoIterator`, `FromIterator`,
`Fn`, `FnMut`, `FnOnce`, `Deref`, `DerefMut`, `Allocator`; the enum `Ordering`;
and the functions `panic`, `assert`, `assert_eq`, `unreachable` and `todo`.

```tally
fn f() -> Vec[u8] ! AllocError may alloc { var v = Vec.new(); v.push(1)?; v }   // no imports needed
```
*From: D46.*

## 19.2 Fundamental traits

**19.2.1** `core.marker` declares `Copy` (7.6.1), `Send`, `Sync` (15.2),
`Sized` (7.6.5), `Static` (7.6.7), and the zero-sized marker structs `NotSend` and `NotSync`
(15.2.2), each with a constant `NotSend.VALUE`/`NotSync.VALUE` used to build
them. `core.ops` declares `Drop` (7.6.3), `Deref`, `DerefMut` (7.6.4), `Fn`,
`FnMut`, `FnOnce` (7.7). `core.default` declares:

```tally
pub trait Default { fn default() -> Self; }
```

`Default` is implemented by integers and floats (0), `bool` (`false`), `char`
(`'\0'`), `()`, `&str` (`""`), `?T` (`.None`), tuples and arrays of `Default`
types, and by `Vec`, `String`, `HashMap`, `HashSet` (empty, using the default
allocator).

```tally
let n: u32 = Default.default();      // 0
let v: Vec[u8] = Vec.default();      // empty
```
*From: D27, D46.*

**19.2.2** `core.marker` also declares `Clone` (7.6.2):

```tally
pub trait Clone { effect Clone = none; fn clone(&self) -> Self ! AllocError may Self.Clone; }
```

`Clone` is implemented with effect `none` by each primitive type, by shared
references, raw and function pointers, by every type that derives `Copy`
(16.3.3), by arrays and tuples of such types (7.9.2), and by
`Rc` and `Arc` (which copy a pointer and increment a count); with effect
`T | A` by `Vec[T, A]`, `Box[T, A]` and `HashMap`/`HashSet` (which allocate
through `A` and clone elements); by `String[A]` with effect `A`.

```tally
let a = String.from("hi")?;
let b = a.clone()?;        // allocates through Global: effect alloc
```
*From: D22, D23.*

## 19.3 Option, Result and Error

**19.3.1** `core.option` declares `@derive(Copy, Eq, Ord, Hash, Debug) pub
enum Option[T] { None, Some(T) }` (derived impls require the same of `T`;
`None` orders before `Some`) and these methods:

```tally
impl[T] Option[T] {
    pub fn is_some(&self) -> bool;
    pub fn is_none(&self) -> bool;
    pub fn unwrap(self) -> T may panic;                       // panics with "called unwrap on None"
    pub fn expect(self, msg: &str) -> T may panic;            // panics with msg
    pub fn unwrap_or(self, default: T) -> T may T;            // drops default if unused
    pub fn unwrap_or_else[F: FnOnce() -> T](self, f: F) -> T may F;
    pub fn map[U, F: FnOnce(T) -> U](self, f: F) -> ?U may F;
    pub fn and_then[U, F: FnOnce(T) -> ?U](self, f: F) -> ?U may F;
    pub fn filter[P: FnOnce(&T) -> bool](self, p: P) -> ?T may P | T;
    pub fn ok_or[E](self, e: E) -> T ! E may E;
    pub fn as_ref(&self) -> ?&T from self;
    pub fn as_mut(&mut self) -> ?&mut T from self;
    pub fn take(&mut self) -> ?T;
    pub fn replace(&mut self, v: T) -> ?T;
    pub fn insert(&mut self, v: T) -> &mut T from self may T;
}
impl[T: Copy] Option[&T] { pub fn copied(self) -> ?T; }
```

```tally
let port = args.get(1).map(|s| s.parse.[u16]()).unwrap_or(.Ok(80))?;
```
*From: D19.*

**19.3.2** `core.result` declares `@derive(Copy, Eq, Ord, Hash, Debug) pub
enum Result[T, E] { Ok(T), Err(E) }` and these methods:

```tally
impl[T, E] Result[T, E] {
    pub fn is_ok(&self) -> bool;
    pub fn is_err(&self) -> bool;
    pub fn ok(self) -> ?T may E;
    pub fn err(self) -> ?E may T;
    pub fn unwrap(self) -> T may panic | E where E: Debug;    // message includes E's Debug output
    pub fn expect(self, msg: &str) -> T may panic | E where E: Debug;
    pub fn unwrap_err(self) -> E may panic | T where T: Debug;
    pub fn unwrap_or(self, default: T) -> T may T | E;
    pub fn map[U, F: FnOnce(T) -> U](self, f: F) -> U ! E may F;
    pub fn map_err[G, F: FnOnce(E) -> G](self, f: F) -> T ! G may F;
    pub fn and_then[U, F: FnOnce(T) -> U ! E](self, f: F) -> U ! E may F;
    pub fn as_ref(&self) -> (&T) ! (&E) from self;
}
```

```tally
let n = parse(s).map_err(|e| AppError.Parse(e))?;
```
*From: D19, D20.*

**19.3.3** `core.error` declares:

```tally
pub trait Error: Display + Debug {
    fn source(&self) -> ?&dyn Error from self { .None }
}
```

`AllocError`, the library's parse errors and `std.io.Error` implement it.

```tally
fn report(e: &dyn Error) may sys | block | panic {
    io.eprintln("error: ", e);
    var cur = e.source();
    while let .Some(s) = cur { io.eprintln("  caused by: ", s); cur = s.source(); }
}
```
*From: D20.*

## 19.4 Memory and pointers

**19.4.1** `core.mem` declares:

```tally
pub fn size_of[T]() -> usize;                 // 13.3; usable in constant expressions
pub fn align_of[T]() -> usize;
pub fn size_of_val[T: ?Sized](v: &T) -> usize;
pub fn align_of_val[T: ?Sized](v: &T) -> usize;
pub fn swap[T](a: &mut T, b: &mut T);
pub fn replace[T](dest: &mut T, v: T) -> T;
pub fn take[T: Default](dest: &mut T) -> T;
pub fn drop[T](v: T) may T;                   // drops v now
pub fn forget[T](v: T);                       // never drops v (8.5.3)
pub unsafe fn transmute[T, U](v: T) -> U;     // reinterprets bits; error unless size_of[T] == size_of[U]
pub unsafe fn zeroed[T]() -> T;               // all-zero bits; UB if that is not a valid T
```

```tally
let old = mem.replace(&mut self.buf, Vec.new());
mem.swap(&mut a, &mut b);
```
*From: D23, D36.*

**19.4.2** Raw pointer methods, for `*T` and `*mut T` with sized `T` (the
`*mut T` forms that write are only on `*mut T`):

```tally
impl[T] *T {
    pub unsafe fn add(self, n: usize) -> *T;          // UB rule 12 (13.2.1); panics never
    pub unsafe fn sub(self, n: usize) -> *T;
    pub unsafe fn offset(self, n: isize) -> *T;
    pub unsafe fn offset_from(self, origin: *T) -> isize;   // both in one object
    pub unsafe fn read(self) -> T;                    // copies bits out, even if T is not Copy
    pub unsafe fn read_unaligned(self) -> T;
    pub unsafe fn read_volatile(self) -> T may sys;   // 13.4.5
    pub unsafe fn copy_to(self, dest: *mut T, n: usize);              // memmove
    pub unsafe fn copy_to_nonoverlapping(self, dest: *mut T, n: usize);   // memcpy
    pub unsafe fn as_ref(self) -> &T;                 // region inferred at use (9.7.2)
    pub fn cast[U](self) -> *U;
    pub fn cast_mut(self) -> *mut T;
    pub fn addr(self) -> usize;                       // 13.4.1
    pub fn expose(self) -> usize;
    pub fn with_addr(self, a: usize) -> *T may panic; // panics if a is 0
    pub fn is_aligned(self) -> bool;
}
impl[T] *mut T {
    // all of the above with *mut results, plus:
    pub unsafe fn write(self, v: T);                  // does not drop the old value
    pub unsafe fn write_unaligned(self, v: T);
    pub unsafe fn write_volatile(self, v: T) may sys;
    pub unsafe fn write_bytes(self, byte: u8, n: usize);
    pub unsafe fn as_mut(self) -> &mut T;
    pub unsafe fn drop_in_place(self) may T;
}
```

and in `core.ptr`: `pub fn dangling[T]() -> *mut T` (non-null, aligned, not
dereferenceable), `pub unsafe fn from_exposed[T](a: usize) -> *mut T may panic`
(13.4.1; panics if `a` is 0), `pub fn eq[T: ?Sized](a: *T, b: *T) -> bool` (address and
metadata equality).

```tally
let p: *mut u32 = buf.as_mut_ptr();
unsafe { p.add(3).write(7); }
```
*From: D36.*

**19.4.3** Volatile operations (`read_volatile`, `write_volatile`) are the
primitive sources of `sys` (10.7.1); see the chapter introduction for the
other `core` functions with `sys`.

```tally
unsafe { (GPIO_OUT as *mut u32).write_volatile(1 << 5); }   // GPIO_OUT: a *mut u32 from from_exposed
```
*From: D3, D36.*

**19.4.4** `core.mem.Uninit[T]` is storage for a `T` that may not be
initialised. It has the size and alignment of `T`, no drop glue, and is
`Copy` if `T` is:

```tally
pub struct Uninit[T] { ... }
impl[T] Uninit[T] {
    pub fn new() -> Self;                        // uninitialised
    pub fn of(v: T) -> Self;                      // initialised with v
    pub fn as_ptr(&self) -> *T;
    pub fn as_mut_ptr(&mut self) -> *mut T;
    pub fn write(&mut self, v: T) -> &mut T from self;   // does not drop previous contents
    pub unsafe fn assume_init(self) -> T;        // UB if not initialised
    pub unsafe fn assume_init_ref(&self) -> &T from self;
}
```

```tally
var slot: Uninit[Big] = Uninit.new();
init_big(slot.as_mut_ptr());
let big = unsafe { slot.assume_init() };
```
*From: D36.*

## 19.5 Slices

**19.5.1** Methods of `[]T` (callable on arrays and `Vec` through auto-deref
and unsizing):

```tally
impl[T] []T {
    pub fn len(&self) -> usize;
    pub fn is_empty(&self) -> bool;
    pub fn get(&self, i: usize) -> ?&T from self;
    pub fn get_mut(&mut self, i: usize) -> ?&mut T from self;
    pub fn first(&self) -> ?&T from self;
    pub fn last(&self) -> ?&T from self;
    pub fn iter(&self) -> slice.Iter[T] from self;
    pub fn iter_mut(&mut self) -> slice.IterMut[T] from self;
    pub fn split_at(&self, mid: usize) -> (&[]T, &[]T) from self may panic;      // panics if mid > len
    pub fn split_at_mut(&mut self, mid: usize) -> (&mut []T, &mut []T) from self may panic;
    pub fn chunks(&self, n: usize) -> slice.Chunks[T] from self may panic;    // panics if n == 0
    pub fn windows(&self, n: usize) -> slice.Windows[T] from self may panic;  // panics if n == 0
    pub fn swap(&mut self, a: usize, b: usize) may panic;
    pub fn reverse(&mut self);
    pub fn rotate_left(&mut self, k: usize) may panic;                        // panics if k > len
    pub fn as_ptr(&self) -> *T;
    pub fn as_mut_ptr(&mut self) -> *mut T;
    pub unsafe fn get_unchecked(&self, i: usize) -> &T from self;
}
impl[T: Eq] []T {
    pub fn contains(&self, x: &T) -> bool;
    pub fn starts_with(&self, p: &[]T) -> bool;
    pub fn ends_with(&self, p: &[]T) -> bool;
}
impl[T: Ord] []T {
    pub fn sort(&mut self) may T;                          // unstable, in place, O(n log n) worst case, no allocation
    pub fn binary_search(&self, x: &T) -> usize ! usize may T;   // .Ok(index) or .Err(insertion point)
}
impl[T: Copy] []T {
    pub fn copy_from_slice(&mut self, src: &[]T) may panic;   // panics if lengths differ
    pub fn fill(&mut self, v: T);
}
```

`slice.Iter[T]` and `slice.IterMut[T]` are view structs implementing
`Iterator` with `Item = &T` / `&mut T`, effect `none`. `slice.from_raw[T](p:
*T, n: usize) -> &[]T` and `slice.from_raw_mut` are `unsafe` (the region of
the result is inferred at the use).

```tally
let (head, tail) = data.split_at(4);
var xs = [3, 1, 2];
xs.sort();
```
*From: D24, D46.*

**19.5.2** `sort` must not allocate and must run in O(n log n) comparisons in
the worst case; the order of equal elements afterwards is unspecified. Its
effects are those of `T`'s `Ord` impl (`T`, which includes `T.Cmp`). If a
comparison panics, the program stops as for any panic.

```tally
names.sort();                      // ordering by Ord for &str: byte-wise
```
*From: D2, D22.*

## 19.6 Strings and characters

**19.6.1** Methods of `str`:

```tally
impl str {
    pub fn len(&self) -> usize;                                   // bytes
    pub fn is_empty(&self) -> bool;
    pub fn as_bytes(&self) -> &[]u8 from self;
    pub fn as_ptr(&self) -> *u8;
    pub fn chars(&self) -> text.Chars from self;                   // Iterator[Item = char]
    pub fn char_indices(&self) -> text.CharIndices from self;      // Item = (usize, char)
    pub fn bytes(&self) -> slice.Iter[u8] from self;
    pub fn lines(&self) -> text.Lines from self;                   // splits on \n, strips a trailing \r
    pub fn split(&self, sep: char) -> text.Split from self;        // Item = &str
    pub fn split_str(&self, sep: &str) -> text.SplitStr from self, sep may panic;   // panics if sep is empty
    pub fn trim(&self) -> &str from self;                         // Unicode White_Space at both ends
    pub fn trim_start(&self) -> &str from self;
    pub fn trim_end(&self) -> &str from self;
    pub fn starts_with(&self, p: &str) -> bool;
    pub fn ends_with(&self, p: &str) -> bool;
    pub fn contains(&self, p: &str) -> bool;
    pub fn find(&self, p: &str) -> ?usize;                        // byte index of first match
    pub fn is_char_boundary(&self, i: usize) -> bool;
    pub fn eq_ignore_ascii_case(&self, o: &str) -> bool;
    pub fn parse[T: FromStr](&self) -> T ! T.Err may T;
}
pub fn from_utf8(b: &[]u8) -> &str ! Utf8Error from b;            // in module core.text
pub unsafe fn from_utf8_unchecked(b: &[]u8) -> &str from b;
pub trait FromStr { type Err; effect Parse = none; fn from_str(s: &str) -> Self ! Self.Err may Self.Parse; }
```

All integer types implement `FromStr` (optional leading `-` for signed types,
or `+`; decimal digits only; error `num.ParseIntError` with kinds `Empty`,
`InvalidDigit`, `Overflow`), as do `f32`/`f64` (decimal and exponent forms,
`inf`, `-inf`, `NaN`; correctly rounded; error `num.ParseFloatError`) and
`bool` (`true`/`false`).

```tally
for word in line.trim().split(' ') { ... }
let n = "42".parse.[u32]()?;
```
*From: D46, D57.*

**19.6.2** Methods of `char` (with the function `from_digit` in `core.chars`):

```tally
impl char {
    pub const MAX: char = '\u{10FFFF}';
    pub fn len_utf8(self) -> usize;                       // 1 to 4
    pub fn encode_utf8(self, buf: &mut [4]u8) -> &str from buf;
    pub fn is_ascii(self) -> bool;
    pub fn is_ascii_digit(self) -> bool;
    pub fn is_ascii_alphabetic(self) -> bool;
    pub fn is_alphabetic(self) -> bool;                   // Unicode Alphabetic property
    pub fn is_numeric(self) -> bool;                      // Unicode general categories Nd, Nl, No
    pub fn is_whitespace(self) -> bool;                   // Unicode White_Space
    pub fn to_digit(self, radix: u32) -> ?u32 may panic;  // panics if radix not in 2..=36
    pub fn to_ascii_uppercase(self) -> char;
    pub fn to_ascii_lowercase(self) -> char;
}
pub fn from_digit(d: u32, radix: u32) -> ?char may panic;
```

The Unicode properties are those of Unicode version 16.0; an implementation
may use a later version and must document which.

```tally
let d = c.to_digit(16)?;
```
*From: D57.*

## 19.7 Numbers

**19.7.1** Every integer type `I` (with `N` bits) has these associated
constants and methods in `core.num` (signed-only and unsigned-only items
noted):

```tally
impl I {
    pub const MIN: I; pub const MAX: I; pub const BITS: u32 = N;
    pub fn checked_add(self, o: I) -> ?I;  // also checked_sub, checked_mul, checked_div, checked_rem,
                                           // checked_neg, checked_shl(u32), checked_shr(u32), checked_pow(u32)
    pub fn wrapping_add(self, o: I) -> I;  // also wrapping_sub/mul/neg/shl/shr/pow (shift amount masked to N)
    pub fn saturating_add(self, o: I) -> I;   // also saturating_sub, saturating_mul, saturating_pow
    pub fn overflowing_add(self, o: I) -> (I, bool);   // also _sub, _mul
    pub fn pow(self, e: u32) -> I may panic;
    pub fn div_euclid(self, o: I) -> I may panic;
    pub fn rem_euclid(self, o: I) -> I may panic;
    pub fn count_ones(self) -> u32; pub fn count_zeros(self) -> u32;
    pub fn leading_zeros(self) -> u32; pub fn trailing_zeros(self) -> u32;
    pub fn rotate_left(self, n: u32) -> I; pub fn rotate_right(self, n: u32) -> I;   // n taken mod N
    pub fn swap_bytes(self) -> I;          // only for N a multiple of 8
    pub fn reverse_bits(self) -> I;
    pub fn to_le(self) -> I; pub fn to_be(self) -> I; pub fn from_le(x: I) -> I; pub fn from_be(x: I) -> I;
    pub fn to_le_bytes(self) -> [N/8]u8;   // only for N a multiple of 8; also to_be_bytes, to_ne_bytes
    pub fn from_le_bytes(b: [N/8]u8) -> I; // also from_be_bytes, from_ne_bytes
    // signed only:
    pub fn abs(self) -> I may panic;       // panics on MIN
    pub fn unsigned_abs(self) -> U;        // U: the unsigned type of the same width
    pub fn signum(self) -> I;
    // unsigned only:
    pub fn is_power_of_two(self) -> bool;
    pub fn next_power_of_two(self) -> I may panic;   // panics if the result exceeds MAX
}
```

```tally
let h = x.wrapping_mul(0x9E37_79B9).rotate_left(5);
let bytes = len.to_be_bytes();
```
*From: D14, D15.*

**19.7.2** `core.num.NonZero[I]`, for integer types `I`, holds a non-zero
value of `I`, with the same size as `I`, and is pointer-like for 13.3.6:

```tally
pub struct NonZero[I] { ... }
impl[I] NonZero[I] { pub fn new(v: I) -> ?Self; pub unsafe fn new_unchecked(v: I) -> Self; pub fn get(self) -> I; }
```

```tally
let id: ?NonZero[u32] = NonZero.new(raw);   // same size as u32
```
*From: D37.*

**19.7.3** For `F` in `f32`, `f64`, in `core.num`, with constants in
`core.math` (`PI`, `TAU`, `E`, `SQRT_2`, `LN_2`, `LN_10`, as `f64` and with
`F32` suffixed names for `f32`):

```tally
impl F {
    pub const NAN: F; pub const INFINITY: F; pub const NEG_INFINITY: F;
    pub const MIN: F; pub const MAX: F; pub const MIN_POSITIVE: F; pub const EPSILON: F;
    pub fn abs(self) -> F;  pub fn copysign(self, s: F) -> F;
    pub fn floor(self) -> F; pub fn ceil(self) -> F; pub fn trunc(self) -> F;
    pub fn round(self) -> F;                 // halfway cases away from zero
    pub fn round_ties_even(self) -> F;
    pub fn sqrt(self) -> F; pub fn mul_add(self, a: F, b: F) -> F;
    pub fn exp(self) -> F; pub fn exp2(self) -> F; pub fn ln(self) -> F; pub fn log2(self) -> F;
    pub fn log10(self) -> F; pub fn powf(self, e: F) -> F; pub fn powi(self, e: i32) -> F;
    pub fn sin(self) -> F; pub fn cos(self) -> F; pub fn tan(self) -> F;
    pub fn asin(self) -> F; pub fn acos(self) -> F; pub fn atan(self) -> F; pub fn atan2(self, x: F) -> F;
    pub fn sinh(self) -> F; pub fn cosh(self) -> F; pub fn tanh(self) -> F; pub fn hypot(self, o: F) -> F;
    pub fn min(self, o: F) -> F; pub fn max(self, o: F) -> F;    // IEEE 754-2019 minimumNumber/maximumNumber
    pub fn is_nan(self) -> bool; pub fn is_finite(self) -> bool; pub fn is_infinite(self) -> bool;
    pub fn is_sign_negative(self) -> bool;
    pub fn to_bits(self) -> UN; pub fn from_bits(b: UN) -> F;     // UN: u32 or u64
    pub fn total_cmp(&self, o: &F) -> Ordering;                   // IEEE 754 totalOrder
}
```

Every one of these functions must return the correctly rounded result (the
exact mathematical result rounded to nearest, ties to even), for every
argument, on every target, except `powi`, which must equal `powf` of the
converted exponent. None of them has effects.

```tally
let d = (dx * dx + dy * dy).sqrt();
let a = y.atan2(x);       // identical bits on every target
```
*From: D65, D2.*

## 19.8 Ranges

**19.8.1** `core.ops` declares the range structs (5.11.1), all with public
fields and `@derive(Copy, Eq, Hash, Debug)`:
`Range[T] { start: T, end: T }`, `RangeInclusive[T] { start: T, end: T }`,
`RangeFrom[T] { start: T }`, `RangeTo[T] { end: T }`,
`RangeToInclusive[T] { end: T }`, `RangeFull`. `Range` and `RangeInclusive`
have `contains(&self, x: &T) -> bool` and `is_empty(&self) -> bool` for
`T: Ord`.

```tally
let r = 1..10;
if r.contains(&n) { ... }
```
*From: D46.*

**19.8.2** `Range[T]`, `RangeInclusive[T]` and `RangeFrom[T]` implement
`Iterator` with `Item = T` and effect `Next = none` for every integer type
and `char`. A range iterator yields `start`, then each next value, up to (for
`..`) or including (for `..=`) `end`; for `char`, it skips the surrogate
range. `RangeFrom` stops after yielding `MAX` (it never overflows). Iterating
`a..=I.MAX` yields `MAX` and stops.

```tally
for i in 0..3 { ... }                 // 0, 1, 2
for b in (250 as u8)..=255 { ... }     // 250 ... 255, then stops without overflow
```
*From: D15, D46.*

## 19.9 Comparison

**19.9.1** `core.cmp` declares `Eq` (7.6.6) and:

```tally
@derive(Copy, Eq, Hash, Debug)
pub enum Ordering: i8 { Less = -1, Equal = 0, Greater = 1 }
impl Ordering {
    pub fn reverse(self) -> Ordering;
    pub fn then(self, o: Ordering) -> Ordering;   // self unless Equal
    pub fn is_lt(self) -> bool;  pub fn is_le(self) -> bool;
    pub fn is_gt(self) -> bool;  pub fn is_ge(self) -> bool;
}
```

```tally
let o = a.len.cmp(&b.len).then(a.name.cmp(&b.name));
```
*From: D17.*

**19.9.2** `Ord` is a total order consistent with `==`:

```tally
pub trait Ord: Eq {
    effect Cmp = none;
    fn cmp(&self, o: &Self) -> Ordering may Self.Cmp;
    fn max(self, o: Self) -> Self may Self { ... }    // o if self < o, else self
    fn min(self, o: Self) -> Self may Self { ... }    // self if self <= o, else o
    fn clamp(self, lo: Self, hi: Self) -> Self may Self | panic { ... }   // panics if lo > hi
}
pub fn max[T: Ord](a: T, b: T) -> T may T;
pub fn min[T: Ord](a: T, b: T) -> T may T;
```

`Ord` is implemented with effect `none` by integers, `char`, `bool`
(`false < true`), `()`, `str` and `String`; and with `Cmp` equal to the union
of the elements' `Cmp` effects by slices and arrays of `Ord` types
(lexicographic, shorter prefix first), tuples, references to `Ord` types (by
referent) and `?T` (`None` first). Floats do not implement `Ord`.

```tally
let longest = max(a.len(), b.len());
```
*From: D17, D27.*

## 19.10 Iterators

**19.10.1** `core.iter` declares:

```tally
pub trait Iterator {
    type Item;
    effect Next;
    fn next(&mut self) -> ?Self.Item may Self.Next;
    // provided methods; each has an implicit `where Self: Sized` (7.8.1):
    fn count(self) -> usize may Self | Self.Item | panic { ... }
    fn last(self) -> ?Self.Item may Self | Self.Item { ... }
    fn nth(&mut self, n: usize) -> ?Self.Item may Self | Self.Item { ... }
    fn fold[B, F: FnMut(B, Self.Item) -> B](self, init: B, f: F) -> B may Self | F { ... }
    fn for_each[F: FnMut(Self.Item)](self, f: F) may Self | F { ... }
    fn any[F: FnMut(Self.Item) -> bool](&mut self, f: F) -> bool may Self | F { ... }
    fn all[F: FnMut(Self.Item) -> bool](&mut self, f: F) -> bool may Self | F { ... }
    fn find[P: FnMut(&Self.Item) -> bool](&mut self, p: P) -> ?Self.Item may Self | Self.Item | P { ... }
    fn position[P: FnMut(Self.Item) -> bool](&mut self, p: P) -> ?usize may Self | P | panic { ... }
    fn map[U, F: FnMut(Self.Item) -> U](self, f: F) -> Map[Self, F] { ... }
    fn filter[P: FnMut(&Self.Item) -> bool](self, p: P) -> Filter[Self, P] { ... }
    fn enumerate(self) -> Enumerate[Self] { ... }
    fn zip[J: Iterator](self, other: J) -> Zip[Self, J] { ... }
    fn chain[J: Iterator[Item = Self.Item]](self, other: J) -> Chain[Self, J] { ... }
    fn take(self, n: usize) -> Take[Self] { ... }
    fn skip(self, n: usize) -> Skip[Self] { ... }
    fn peekable(self) -> Peekable[Self] { ... }
    fn collect[C: FromIterator[Self.Item]](self) -> C ! AllocError may Self | C { ... }
}
pub trait IntoIterator {
    type Item; type IntoIter: Iterator[Item = Self.Item]; effect Into = none;
    fn into_iter(self) -> Self.IntoIter may Self.Into;
}
pub trait FromIterator[T] {
    effect Build;
    fn from_iter[I: Iterator[Item = T]](it: I) -> Self ! AllocError may I | Self.Build;
}
```

There is no blanket implementation of `IntoIterator` for iterators (it would
overlap the others under 7.4.2); `for` accepts either (5.10.6). The adapter
structs (`Map`, `Filter`, `Enumerate`, `Zip`, `Chain`, `Take`, `Skip`,
`Peekable`) implement `Iterator` with `effect Next` equal to the union of
their parts' type effects (including `Self.Item` for `Filter` and `Skip`,
which drop items); `Enumerate` panics if the count overflows `usize`, so its
`Next` includes `panic`. `Peekable` adds `peek(&mut self) -> ?&Self.Item from
self`. `Vec`, `String`, `HashMap` and `HashSet` implement `FromIterator` with
`Build` equal to their allocator's effects.

```tally
let evens: Vec[u32] = (0..100).filter(|n| n % 2 == 0).collect()?;
let total = prices.iter().fold(0, |acc, p| acc + *p);
```
*From: D27, D30.*

**19.10.2** `IntoIterator` is implemented by `[N]T` (by value, `Item = T`),
`&[N]T` and `&[]T` (`Item = &T`), `&mut [N]T` and `&mut []T`
(`Item = &mut T`), `Vec[T, A]` (by value), `&Vec[T, A]` and `&mut Vec[T, A]`,
`&HashMap[K, V]` (`Item = (&K, &V)`), `&HashSet[T]` (`Item = &T`), and `?T`
(zero or one item).

```tally
for x in &v { ... }        // x: &T
for x in v { ... }         // x: T, v consumed
```
*From: D27.*

## 19.11 Cells and atomics

**19.11.1** `core.cell`:

```tally
pub struct UnsafeCell[T: ?Sized] { ... }               // 9.8.1; not Sync
impl[T] UnsafeCell[T] {
    pub fn new(v: T) -> Self;
    pub fn get(&self) -> *mut T;
    pub fn get_mut(&mut self) -> &mut T from self;
    pub fn into_inner(self) -> T;
}
pub struct Cell[T] { ... }                             // Send if T: Send; not Sync
impl[T] Cell[T] {
    pub fn new(v: T) -> Self;
    pub fn replace(&self, v: T) -> T;
    pub fn set(&self, v: T) may T;                     // drops the old value
    pub fn take(&self) -> T where T: Default;
    pub fn into_inner(self) -> T;
}
impl[T: Copy] Cell[T] { pub fn get(&self) -> T; }
pub struct RefCell[T] { ... }                          // run-time checked borrowing; not Sync
impl[T] RefCell[T] {
    pub fn new(v: T) -> Self;
    pub fn borrow(&self) -> cell.Ref[T] from self may panic;        // panics if mutably borrowed
    pub fn borrow_mut(&self) -> cell.RefMut[T] from self may panic; // panics if borrowed at all
    pub fn try_borrow(&self) -> ?cell.Ref[T] from self;
    pub fn try_borrow_mut(&self) -> ?cell.RefMut[T] from self;
}
pub struct OnceCell[T] { ... }                         // set at most once; not Sync
impl[T] OnceCell[T] {
    pub fn new() -> Self; pub fn get(&self) -> ?&T from self;
    pub fn set(&self, v: T) -> () ! T;                  // .Err(v) if already set
    pub fn get_or_init[F: FnOnce() -> T](&self, f: F) -> &T from self may F | panic;  // panics if f re-enters
}
```

`Ref[T]` and `RefMut[T]` are view structs that implement `Deref` (and
`DerefMut`) to `T` and release the borrow when dropped (effect `none`).

```tally
let cache = RefCell.new(Vec.new());
cache.borrow_mut().push(1)?;
```
*From: D26.*

**19.11.2** `core.atomic.Atomic[T]` exists for `T` among `bool`, the integer
types of 8, 16, 32 and 64 bits, `usize`, `isize`, `*mut U` and, on targets that
support it, 128-bit integers; instantiating it with another `T`, or with a
width the target cannot perform atomically, is an error. It has the size of
`T` and alignment equal to its size. It is `Send` and `Sync`.

```tally
pub enum Ordering { Relaxed, Release, Acquire, AcqRel, SeqCst }   // atomic.Ordering
impl[T] Atomic[T] {
    pub fn new(v: T) -> Self;
    pub fn load(&self, o: Ordering) -> T may panic;          // panics if o is Release or AcqRel
    pub fn store(&self, v: T, o: Ordering) may panic;        // panics if o is Acquire or AcqRel
    pub fn swap(&self, v: T, o: Ordering) -> T;
    pub fn compare_exchange(&self, cur: T, new: T, ok: Ordering, fail: Ordering) -> T ! T may panic;
    pub fn compare_exchange_weak(&self, cur: T, new: T, ok: Ordering, fail: Ordering) -> T ! T may panic;
    pub fn get_mut(&mut self) -> &mut T from self;
    pub fn into_inner(self) -> T;
}
impl[I] Atomic[I] {                                     // for integer I only; wrapping; return the previous value
    pub fn fetch_add(&self, v: I, o: Ordering) -> I;  pub fn fetch_sub(&self, v: I, o: Ordering) -> I;
    pub fn fetch_and(&self, v: I, o: Ordering) -> I;  pub fn fetch_or(&self, v: I, o: Ordering) -> I;
    pub fn fetch_xor(&self, v: I, o: Ordering) -> I;  pub fn fetch_max(&self, v: I, o: Ordering) -> I;
    pub fn fetch_min(&self, v: I, o: Ordering) -> I;
}
impl Atomic[bool] { pub fn fetch_and(&self, v: bool, o: Ordering) -> bool; pub fn fetch_or(&self, v: bool, o: Ordering) -> bool; }
pub fn fence(o: Ordering) may panic;           // panics if o is Relaxed
pub fn compiler_fence(o: Ordering) may panic;  // panics if o is Relaxed
```

`compare_exchange` panics if `fail` is `Release` or `AcqRel`. When the
ordering arguments are constants, the compiler checks them and reports the
invalid cases as errors, and the call then has no `panic` effect.

```tally
static NEXT_ID: Atomic[u64] = Atomic.new(1);
fn next_id() -> u64 { NEXT_ID.fetch_add(1, .Relaxed) }     // constant ordering: effect-free
```
*From: D35.*

## 19.12 Hashing

**19.12.1** `core.hash`:

```tally
pub trait Hasher { fn write(&mut self, bytes: &[]u8); fn finish(&self) -> u64; }
pub trait Hash { fn hash[H: Hasher](&self, h: &mut H) may panic; }
pub trait BuildHasher { type Hasher: Hasher; fn build(&self) -> Self.Hasher; }
```

Integers hash by writing the little-endian bytes of their storage (4.2.2),
holding the zero- or sign-extended value (`bool` as one byte, `char` as
`u32`); `str` writes its bytes followed by the byte `0xFF`; slices and
arrays write their length as `u64` and then each element; tuples write each
element; `?T` writes the tag as `u64` then the payload; references hash their
referent. Floats do not implement `Hash`.

```tally
@derive(Hash, Eq) struct Key { id: u32, name: String }
```
*From: D48.*

**19.12.2** `core.hash.SipHasher13` implements SipHash-1-3 with 128-bit key
`(k0, k1)` as specified by Aumasson and Bernstein, producing the 64-bit
output. `core.hash.RandomSip` is a zero-sized `BuildHasher` that builds
`SipHasher13` with the *process hash key*. The process hash key is
`(0, 0)` until set; `std` sets it from the operating system's random source at
start-up (17.1.2); freestanding programs may set it once, before any hashing,
with `unsafe fn set_process_key(k0: u64, k1: u64)`.

```tally
var h = SipHasher13.new_with_keys(1, 2);
"abc".hash(&mut h);
let x = h.finish();
```
*From: D48.*

## 19.13 Allocation

**19.13.1** `core.alloc.Layout` describes a memory block:

```tally
@derive(Copy, Eq, Debug) pub struct Layout { ... }
impl Layout {
    pub fn new(size: usize, align: usize) -> ?Layout;   // None unless align is a power of two and size rounded up to align <= isize.MAX
    pub fn of[T]() -> Layout;
    pub fn array[T](n: usize) -> ?Layout;               // None on overflow
    pub fn size(&self) -> usize; pub fn align(&self) -> usize;
}
```

```tally
let l = Layout.array.[u64](n) orelse return .Err(AllocError);
```
*From: D32.*

**19.13.2** `AllocError` is `@derive(Copy, Eq, Debug) pub struct
AllocError;` and implements `Display` (writing `memory allocation failed`) and
`Error`.

```tally
fn grow(v: &mut Vec[u8]) ! AllocError may alloc { v.reserve(4096) }
```
*From: D22.*

**19.13.3** The `Allocator` trait:

```tally
pub unsafe trait Allocator {
    effect Use;
    fn allocate(&self, l: Layout) -> *mut u8 ! AllocError may Self.Use;
    unsafe fn deallocate(&self, p: *mut u8, l: Layout) may Self.Use;
    unsafe fn grow(&self, p: *mut u8, old: Layout, new: Layout) -> *mut u8 ! AllocError may Self.Use { ... }
    unsafe fn shrink(&self, p: *mut u8, old: Layout, new: Layout) -> *mut u8 ! AllocError may Self.Use { ... }
}
```

An implementer promises: `allocate` returns a block of at least `l.size()`
bytes aligned to `l.align()`, not overlapping any other live block, valid
until passed to `deallocate`, `grow` or `shrink`; zero-size requests return a
non-null aligned pointer that must not be accessed; the allocator's blocks
stay valid as long as any reference to the allocator (or the allocator value)
can be used. `&A` implements `Allocator` whenever `A` does, with the same
`Use` (this is the only implementation of `Allocator` for a reference type,
so it overlaps nothing).

```tally
unsafe impl Allocator for Bump { effect Use = none; ... }   // and so &Bump implements it too
```
*From: D32, D36.*

**19.13.4** `Global` is a zero-sized `@derive(Copy, Default)` struct
implementing `Allocator` with `effect Use = alloc` by forwarding to the
program's global allocator. `core.alloc.Bump` is an arena allocator over a
borrowed buffer:

```tally
pub view struct Bump { ... }
impl Bump {
    pub fn new(buf: &mut []u8) -> Bump from buf;
    pub fn used(&self) -> usize;
    pub fn reset(&mut self);                 // requires no outstanding borrows of the Bump
}
unsafe impl Allocator for Bump { effect Use = none; ... }    // deallocate frees only the most recent block
```

Because collections using `&Bump` borrow it, `reset` cannot be called while
any of them is alive (chapter 9).

```tally
var mem: [65536]u8 = [0; 65536];
var arena = Bump.new(&mut mem);
{
    var v: Vec[u32, &Bump] = Vec.new_in(&arena);
    v.push(1)?;          // effect none
}
arena.reset();
```
*From: D32, D24.*

**19.13.5** The *global allocator* is the `static` marked `@global_allocator`,
whose type must implement `GlobalAlloc`:

```tally
pub unsafe trait GlobalAlloc {
    effect Use;
    fn alloc(&self, l: Layout) -> ?*mut u8 may Self.Use;
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) may Self.Use;
    unsafe fn realloc(&self, p: *mut u8, old: Layout, new_size: usize) -> ?*mut u8 may Self.Use { ... }
}
```

`Use` must be within `sys | block | panic`. A program has at most one
`@global_allocator`. If none is defined, `std` provides `std.alloc.System`
(20.10.1); a program that does not link `std` and contains any function with
the `alloc` effect must define one, or linking fails. Calls to the global
allocator's methods are the primitive sources of `alloc` (10.7.1).

```tally
@global_allocator static HEAP: TlsfAlloc = TlsfAlloc.new_static();
```
*From: D32, D28.*

## 19.14 Collections and owning pointers

**19.14.1** `Box[T: ?Sized, A = Global]` (in `core.alloc`) owns one `T` on the
heap:

```tally
impl[T] Box[T, Global] { pub fn new(v: T) -> Self ! AllocError may alloc | T; }
impl[T, A: Allocator] Box[T, A] {
    pub fn new_in(v: T, a: A) -> Self ! AllocError may A | T;
    pub fn into_inner(b: Self) -> T may A;
    pub fn leak(b: Self) -> &mut T;               // never freed; region static unless T or A holds borrows
    pub fn into_raw(b: Self) -> *mut T;
    pub unsafe fn from_raw_in(p: *mut T, a: A) -> Self;
}
```

`Box` is dereferenced with built-in `*` (5.6.1); dropping it drops the value
and frees the block (effect `T | A`). It implements `Clone` (effect `T | A`,
when `T: Clone` and `A: Clone`), `Eq`, `Ord`, `Hash`, `Debug`, `Display` by
forwarding.

```tally
let node = Box.new(Node { value: 1, next: .None })?;
```
*From: D22, D32.*

**19.14.2** `Vec[T, A = Global]` (in `core.alloc`, prelude) is a growable
array:

```tally
impl[T] Vec[T, Global] { pub fn new() -> Self; pub fn with_capacity(n: usize) -> Self ! AllocError may alloc; }
impl[T, A: Allocator] Vec[T, A] {
    pub fn new_in(a: A) -> Self;
    pub fn with_capacity_in(n: usize, a: A) -> Self ! AllocError may A;
    pub fn len(&self) -> usize; pub fn capacity(&self) -> usize; pub fn is_empty(&self) -> bool;
    pub fn push(&mut self, v: T) ! AllocError may A | T;          // on failure v is dropped
    pub fn push_within_capacity(&mut self, v: T) -> () ! T;       // .Err(v) if full; never allocates
    pub fn pop(&mut self) -> ?T;
    pub fn insert(&mut self, i: usize, v: T) ! AllocError may A | T | panic;   // panics if i > len
    pub fn remove(&mut self, i: usize) -> T may panic;
    pub fn swap_remove(&mut self, i: usize) -> T may panic;
    pub fn truncate(&mut self, n: usize) may T;
    pub fn clear(&mut self) may T;
    pub fn reserve(&mut self, extra: usize) ! AllocError may A;   // capacity overflow is an AllocError
    pub fn shrink_to_fit(&mut self) ! AllocError may A;
    pub fn extend_from_slice(&mut self, s: &[]T) ! AllocError may A | T where T: Clone;
    pub fn as_slice(&self) -> &[]T from self;
    pub fn as_mut_slice(&mut self) -> &mut []T from self;
    pub fn allocator(&self) -> &A from self;
}
```

`Vec` implements `Deref`/`DerefMut` to `[]T`, `Drop` (with
`@drop_ignores(T)`, effect `T | A`), `Clone`, `Eq`, `Ord`, `Hash`, `Debug`,
`Default` (for `A: Default`), `IntoIterator` and `FromIterator`. Growth
multiplies the capacity by 2 (at least 4 elements); `push` takes amortised
constant time. `Vec` is `Send`/`Sync` when `T` and `A` are.

```tally
var v: Vec[u32] = Vec.with_capacity(16)?;
for i in 0..16 { v.push_within_capacity(i) orelse unreachable(); }   // cannot allocate
```
*From: D22, D32.*

**19.14.3** `String[A = Global]` (prelude) is a `Vec[u8, A]` that always holds
valid UTF-8:

```tally
impl String[Global] {
    pub fn new() -> Self;
    pub fn from(s: &str) -> Self ! AllocError may alloc;
}
impl[A: Allocator] String[A] {
    pub fn new_in(a: A) -> Self;
    pub fn from_in(s: &str, a: A) -> Self ! AllocError may A;
    pub fn from_utf8(b: Vec[u8, A]) -> Self ! FromUtf8Error[A] may A;   // the error gives the bytes back
    pub fn push(&mut self, c: char) ! AllocError may A;
    pub fn push_str(&mut self, s: &str) ! AllocError may A;
    pub fn pop(&mut self) -> ?char;
    pub fn truncate(&mut self, n: usize) may panic;                    // panics if not a char boundary
    pub fn clear(&mut self);
    pub fn as_str(&self) -> &str from self;
    pub fn into_bytes(self) -> Vec[u8, A];
    pub fn len(&self) -> usize; pub fn capacity(&self) -> usize;
    pub fn reserve(&mut self, extra: usize) ! AllocError may A;
}
```

`String` derefs to `str` and implements `Clone`, `Eq`, `Ord`, `Hash`,
`Debug`, `Display`, `Default`, `fmt.Sink` (19.15.1, effect `A`) and
`FromIterator[char]`.

```tally
var s = String.new();
s.push_str("x = ")?;
fmt.write(&mut s, 42)?;
```
*From: D22, D57.*

**19.14.4** `core.collections.HashMap[K, V, S = RandomSip, A = Global]`
requires `K: Hash + Eq` and `S: BuildHasher` for its operations:

```tally
impl[K, V] HashMap[K, V, RandomSip, Global] { pub fn new() -> Self; }
impl[K: Hash + Eq, V, S: BuildHasher, A: Allocator] HashMap[K, V, S, A] {
    pub fn with_hasher_in(s: S, a: A) -> Self;
    pub fn len(&self) -> usize; pub fn is_empty(&self) -> bool;
    pub fn get(&self, k: &K) -> ?&V from self may panic;
    pub fn get_mut(&mut self, k: &K) -> ?&mut V from self may panic;
    pub fn contains_key(&self, k: &K) -> bool may panic;
    pub fn insert(&mut self, k: K, v: V) -> ?V ! AllocError may A | K | V | panic;
    pub fn remove(&mut self, k: &K) -> ?V may K | panic;
    pub fn get_or_insert_with[F: FnOnce() -> V](&mut self, k: K, f: F) -> &mut V ! AllocError from self may A | K | V | F | panic;
    pub fn clear(&mut self) may K | V;
    pub fn reserve(&mut self, extra: usize) ! AllocError may A | panic;
    pub fn iter(&self) -> collections.HashMapIter[K, V] from self;      // Item = (&K, &V)
    pub fn keys(&self) -> collections.Keys[K, V] from self;
    pub fn values(&self) -> collections.Values[K, V] from self;
}
```

(The `panic` effect comes from `Hash`.) `HashSet[T, S = RandomSip, A =
Global]`:

```tally
impl[T] HashSet[T, RandomSip, Global] { pub fn new() -> Self; }
impl[T: Hash + Eq, S: BuildHasher, A: Allocator] HashSet[T, S, A] {
    pub fn with_hasher_in(s: S, a: A) -> Self;
    pub fn len(&self) -> usize; pub fn is_empty(&self) -> bool;
    pub fn contains(&self, x: &T) -> bool may panic;
    pub fn insert(&mut self, x: T) -> bool ! AllocError may A | T | panic;   // true if newly added
    pub fn remove(&mut self, x: &T) -> bool may T | panic;
    pub fn clear(&mut self) may T;
    pub fn iter(&self) -> collections.HashSetIter[T] from self;            // Item = &T
}
```

```tally
var counts: HashMap[String, u32] = HashMap.new();
for w in words { *counts.get_or_insert_with(String.from(w)?, || 0)? += 1; }
```
*From: D22, D48.*

**19.14.5** The iteration order of `HashMap` and `HashSet` is unspecified; it
may differ between runs (because of the process hash key) and between
implementations, but is the same for two iterations of one unmodified map in
one run.

```tally
for (k, v) in &counts { ... }    // order unspecified
```
*From: D48, D2.*

**19.14.6** `Rc[T: ?Sized, A = Global]` and `Arc[T: ?Sized, A = Global]` are
reference-counted shared pointers: `new(v: T) -> Self ! AllocError may alloc |
T` (and `new_in`), `clone` (effect `none`, never fails: it increments the
count; `Arc` atomically), `strong_count(&Self) -> usize`, `get_mut(&mut Self)
-> ?&mut T from self` (if the count is 1), `ptr_eq(&Self, &Self) -> bool`.
Both deref to `T`. Dropping the last reference drops the value and frees the
block (effect `T | A`). `Rc` is neither `Send` nor `Sync`; `Arc[T]` is both
when `T` is.

```tally
let shared = Arc.new(Config.load()?)?;
let s2 = shared.clone()?;      // cannot fail; ? is required by the Clone signature
```
*From: D26, D22.*

## 19.15 Formatting

**19.15.1** `core.fmt` declares:

```tally
@derive(Copy, Eq, Debug) pub struct FmtError;
pub trait Sink { effect Write; fn write_str(&mut self, s: &str) ! FmtError may Self.Write; }
pub view struct Formatter[effect E] { ... }    // wraps &mut dyn Sink may E
impl[effect E] Formatter[E] {
    pub fn new(sink: &mut dyn Sink may E) -> Formatter[E] from sink;
    pub fn write_str(&mut self, s: &str) ! FmtError may E;
    pub fn write_char(&mut self, c: char) ! FmtError may E;
}
pub trait Display {
    fn fmt[effect E](&self, f: &mut Formatter[E]) ! FmtError may E | panic;
    fn to_string(&self) -> String ! AllocError may alloc | panic { ... }
}
pub trait Debug { fn fmt[effect E](&self, f: &mut Formatter[E]) ! FmtError may E | panic; }
pub fn write[S: Sink, X: Display](s: &mut S, ..args: X) ! FmtError may S.Write | panic | X;
pub fn format[X: Display](..args: X) -> String ! AllocError may alloc | panic | X;
```

`String`, `Vec[u8]` and `core.fmt.FixedBuf[N]` (a `[N]u8` buffer that reports
`FmtError` when full; effect `none`) implement `Sink`.

```tally
var buf: fmt.FixedBuf[64] = fmt.FixedBuf.new();
fmt.write(&mut buf, "t=", t, "ms")?;          // effect-free formatting into a fixed buffer
let s = fmt.format("id-", id)?;               // allocating
```
*From: D47, D30.*

**19.15.2** `Display` output of the built-in types: integers in decimal with
a leading `-` if negative; `bool` as `true`/`false`; `char` and `str` as
themselves; floats as the shortest decimal string that converts back to the
same value, in positional notation with at least one digit after the point
(`1.0`, `0.001`) when the decimal exponent is between −5 and 16 inclusive, and
otherwise in scientific notation `d.ddde±x` with at least one digit after the
point (`1.5e+20`, `1.0e-7`); `NaN`, `inf`, `-inf`; `-0.0` for negative zero.
`Debug` output is the same as `Display` except that `str` and `char` are
quoted and escaped (1.4.4), and as derived (16.3.3) for library types; `Vec`
and slices as `[a, b, c]`; `?T` as `.None` or `.Some(x)`; `T ! E` as `.Ok(x)` or
`.Err(e)`; tuples as `(a, b)`.

```tally
fmt.format(0.1 + 0.2)?        // "0.30000000000000004"
fmt.format(1e21)?             // "1.0e+21"
```
*From: D47, D2.*

**19.15.3** `core.fmt` provides wrapper types implementing `Display` for
formatting options, each built by a function and each a `view` (it borrows
or holds its argument):

```tally
pub fn hex[I](x: I) -> Hex[I];            // I: an integer type; lowercase digits, no prefix; negative as `-` then magnitude
pub fn hex_upper[I](x: I) -> HexUpper[I];
pub fn bin[I](x: I) -> Bin[I];
pub fn oct[I](x: I) -> Oct[I];
pub fn pad_left[X: Display](x: X, width: usize, fill: char) -> PadLeft[X];    // fill before the output
pub fn pad_right[X: Display](x: X, width: usize, fill: char) -> PadRight[X];  // fill after the output
pub fn zero_pad[X: Display](x: X, width: usize) -> ZeroPad[X];   // like pad_left with '0', but a leading '-' stays first
pub fn fixed[F](x: F, digits: u32) -> Fixed[F];   // F: f32 or f64; exactly `digits` digits after the point, correctly rounded, ties to even
pub fn sci[F](x: F, digits: u32) -> Sci[F];       // d.ddd e±x with `digits` digits after the point
pub fn debug[X: Debug](x: X) -> DebugOf[X];       // writes Debug output
```

Widths count Unicode scalar values of the inner output; output already at
least `width` long is not padded or truncated. Each wrapper's `Display`
effects are those of its inner value's `Display` plus `panic`.

```tally
io.println("addr=0x", fmt.zero_pad(fmt.hex(addr), 16), " t=", fmt.fixed(secs, 3));
```
*From: D47.*

## 19.16 Stack bounds and execution contexts

**19.16.1** `core.stack`:

```tally
pub fn bound[F: ?Sized](f: &F) -> ?usize;            // 11.5.2; F: a function item, closure or fn pointer type
pub fn bound_required[F: ?Sized](f: &F) -> usize;    // link error if unbounded
pub const CALL_OVERHEAD: usize;      // target-defined: frame of the runtime entry trampoline (11.5.2)
pub const CONTEXT_OVERHEAD: usize;   // link-time: control block, context-local statics, CALL_OVERHEAD,
                                     // and INTERRUPT_RESERVE on targets that need it
pub const INTERRUPT_RESERVE: usize;  // link-time (11.6.3); 0 on targets whose interrupts use their own stack
pub fn remaining() -> usize may sys;                 // bytes left on the current context's stack, rounded down to 16
```

`CONTEXT_OVERHEAD` includes the space for context-local statics (15.5.2), so
it is a link-time constant, like the bounds.

```tally
let size = stack.bound_required(&worker) + stack.CONTEXT_OVERHEAD;
```
*From: D33.*

**19.16.2** `core.context` provides the unsafe primitives on which threads and
fibers are built:

```tally
pub struct Context { ... }                        // an execution context control block
pub unsafe fn init(ctx: *mut Context, stack: *mut u8, size: usize, entry: fn(*mut u8) -> never may any, arg: *mut u8) may sys;
pub unsafe fn switch(from: *mut Context, to: *mut Context) may sys;   // saves registers, resumes `to`
pub fn current() -> *mut Context;
pub unsafe trait Waiter {
    fn wait(&self, what: &Wait) ! WaitError may sys | block;   // suspend the current fiber until `what` is ready
    fn wake(&self, ctx: *mut Context) may sys;         // make a suspended context runnable
}
pub fn waiter() -> ?&dyn Waiter may sys;           // the current context's waiter (15.4.2)
pub unsafe fn set_waiter(w: ?&dyn Waiter);
```

`Wait` and `WaitError` are declared as:

```tally
@non_exhaustive pub enum Wait {
    Readable(i32), Writable(i32),           // a file descriptor (Unix) or socket handle
    Deadline(Instant),                      // until a monotonic-clock instant (time.Instant)
    Address { addr: *Atomic[u32], expected: u32 },   // until woken, if *addr == expected on entry
    Child(u32),                             // a child process id has exited
    Handle(usize),                          // a platform handle (Windows) is signalled
}
@non_exhaustive @derive(Copy, Eq, Debug) pub enum WaitError { TimedOut, Os(i32) }
```

`core.time.Instant` is the monotonic clock instant type that `std.time`
re-exports (20.6.1); in `core` it has no `now`. `switch` is a call-graph node with an implementation-declared frame size
(11.1.2), so the stack it uses on the suspended context is part of the bound
of every function that reaches it.

```tally
// inside a waiting primitive (20.1.3):
match context.waiter() { .Some(w) => w.wait(&.Readable(fd))?, .None => os_poll(fd)? }
```
*From: D34.*

## 19.17 Foreign-function support

**19.17.1** `core.ffi` declares, for the target, the type aliases `c_char`,
`c_schar`, `c_uchar`, `c_short`, `c_ushort`, `c_int`, `c_uint`, `c_long`,
`c_ulong`, `c_longlong`, `c_ulonglong`, `c_float`, `c_double`, `c_size_t`,
`c_ssize_t`, matching the C ABI, and `c_void`, an enum with no variants
(used only behind pointers).

```tally
extern "C" { fn malloc(n: c_size_t) -> ?*mut c_void may sys | block; }
```
*From: D38.*

**19.17.2** `CStr` is a `@derive(Copy)` view struct referring to a
zero-terminated byte sequence without interior zeros:

```tally
impl CStr {
    pub fn as_ptr(self) -> *c_char;
    pub fn to_bytes(self) -> &[]u8 from self;              // without the terminator
    pub fn to_str(self) -> &str ! text.Utf8Error from self;
    pub unsafe fn from_ptr(p: *c_char) -> CStr;            // scans for the terminator
    pub fn from_bytes_with_nul(b: &[]u8) -> CStr ! CStrError from b;
}
```

`core.alloc.CString[A = Global]` owns such a sequence: `new(s: &str) ->
CString ! CStringError may alloc` (error if `s` contains a zero byte),
`as_cstr(&self) -> CStr from self`.

```tally
let name = CString.new(user_input)?;
unsafe { c_open(name.as_cstr().as_ptr(), 0); }
```
*From: D38.*

## 19.18 Panics, processes, hints and source locations

**19.18.1** `core.panic`:

```tally
pub fn panic[X: Display](..args: X) -> never may panic;       // 12.4.3
pub fn assert[X: Display](cond: bool, ..msg: X) may panic | X;    // panics with "assertion failed" and msg
pub fn assert_eq[T: Eq + Debug](a: T, b: T) may panic | T;    // message shows both values' Debug output
pub fn unreachable() -> never may panic;                      // "entered unreachable code"
pub fn todo() -> never may panic;                             // "not yet implemented"
pub const MESSAGE_CAPACITY: usize;                            // at least 256
pub struct PanicInfo { ... }
impl PanicInfo { pub fn message(&self) -> &str from self; pub fn location(&self) -> source.Location; }
```

```tally
assert(i < n, "index ", i, " out of range ", n);
```
*From: D21, D47.*

**19.18.2** `core.process.abort() -> never may sys` ends the process (or, on
freestanding targets, executes the target's trap instruction) immediately.

```tally
if corrupted { process.abort(); }
```
*From: D21.*

**19.18.3** `core.hint`: `spin()` (a spin-loop hint instruction, effect
`none`), `black_box[T](v: T) -> T` (prevents the optimiser from reasoning about
`v`, effect `none`), and `unsafe fn unreachable_unchecked() -> never` (undefined
behaviour if reached, 13.2.1).

```tally
while lock.load(.Relaxed) { hint.spin(); }
```
*From: D36, D2.*

**19.18.4** `core.source.Location` is `@derive(Copy, Eq, Debug) pub view
struct Location { pub file: &str, pub line: u32, pub column: u32 }` (its
`file` always refers to static data), where `file` is
the path relative to the package directory and `line` and `column` count from
1 (columns in Unicode scalar values). `core.source.location() -> Location`
returns the location of its own call, unless it is called from a function
marked `@track_caller`, in which case it returns the location of the call to
that function (recursively, through chains of `@track_caller` functions). Panic
locations (12.4.1) are determined the same way; `unwrap`, `expect`, `assert`,
`assert_eq` and indexing operations behave as if marked `@track_caller`.

```tally
@track_caller
fn must(x: ?u32) -> u32 may panic { x orelse panic("missing value at ", source.location().line) }
```
*From: D21, D2.*

## 19.19 Time, byte order and target

**19.19.1** `core.time.Duration` is `@derive(Copy, Eq, Ord, Hash, Debug,
Default)` and holds a non-negative span of time as seconds (`u64`) and
nanoseconds (`u32`, less than 1 000 000 000):

```tally
impl Duration {
    pub const ZERO: Duration;  pub const MAX: Duration;
    pub fn new(secs: u64, nanos: u32) -> Duration may panic;      // carries nanos; panics on overflow
    pub fn from_secs(s: u64) -> Duration; pub fn from_millis(ms: u64) -> Duration;
    pub fn from_micros(us: u64) -> Duration; pub fn from_nanos(ns: u64) -> Duration;
    pub fn as_secs(&self) -> u64; pub fn subsec_nanos(&self) -> u32;
    pub fn as_millis(&self) -> u128; pub fn as_nanos(&self) -> u128;
    pub fn checked_add(self, o: Duration) -> ?Duration; pub fn checked_sub(self, o: Duration) -> ?Duration;
    pub fn saturating_sub(self, o: Duration) -> Duration;
    pub fn mul(self, k: u32) -> Duration may panic;
}
```

```tally
let timeout = Duration.from_millis(250);
```
*From: D46.*

**19.19.2** `core.endian.Be[T]` and `Le[T]`, for integer types `T` whose width
is a multiple of 8, store a `T` in big- or little-endian byte order regardless
of the target, with the size of `T` and alignment 1, and `@derive(Copy, Eq,
Hash, Debug)`:

```tally
impl[T] Be[T] { pub fn new(v: T) -> Self; pub fn get(self) -> T; pub fn set(&mut self, v: T); }
```

```tally
@packed struct Ipv4Hdr { ver_ihl: u8, tos: u8, total_len: Be[u16], id: Be[u16], frag: Be[u16],
                         ttl: u8, proto: u8, csum: Be[u16], src: Be[u32], dst: Be[u32] }
let len = hdr.total_len.get();
```
*From: D37.*

**19.19.3** `core.target` declares the constants of 16.2.3 and the enum
`Endian { Little, Big }`.

```tally
const WORD: u32 = core.target.POINTER_WIDTH / 8;
```
*From: D42.*
