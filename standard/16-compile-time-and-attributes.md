# 16. Compile-time evaluation, conditional compilation and attributes

## 16.1 Constant expressions

**16.1.1** A *constant expression* is an expression that must be evaluated at
compile time. The contexts that require one are: the initialiser of a `const`
or `static`; an array length `[N]T` and the count of `[e; N]`; a const generic
argument; an enum tag value; a `@recursion_limit`, `@stack_limit`,
`@stack_bound`, `@align` or `@bitpacked` argument; and the `stack` clause of
`asm` (which must be a literal). A constant expression is evaluated with the
target's sizes, alignments, byte order and pointer width.

```tally
const SECTORS: usize = 2048 * 1024 / 512;
let buf: [SECTORS]u8 = ...;
const LE: bool = core.target.ENDIAN == .Little;
```
*From: D40.*

**16.1.2** A constant expression may contain any expression, including calls,
loops and local bindings, subject to these limits:

1. every function called (directly or indirectly) must have an effect set
   within `alloc | panic`; a call to any other function is an error;
2. evaluation must not reach a foreign call, an `asm` expression, a volatile
   access, an access to a `static var` or to a static containing an
   `UnsafeCell`, or a conversion between pointers and integers (`addr`,
   `expose`, `from_exposed`, `with_addr`, transmutes between pointer and
   integer types); reaching one is an error;
3. other unsafe operations (raw pointer reads and writes, pointer arithmetic,
   union reads) are allowed, but evaluation must detect undefined behaviours
   1, 3, 11 and 12 of 13.2.1 and report them as errors;
4. a panic during evaluation is an error whose diagnostic includes the panic
   message.

```tally
const TABLE: [4]u32 = make_table();              // fine: make_table may at most panic
fn make_table() -> [4]u32 { var t: [4]u32 = [0; 4]; for i in 0..4 { t[i] = 1 << i; } t }
const NOW: u64 = clock.now().as_secs();          // error: now has effect sys
const BAD: u8 = [1, 2][5];                      // error: evaluation panicked: index out of bounds
```
*From: D40, D3.*

**16.1.3** Allocation during evaluation uses a compile-time heap in place of
the global allocator. The final value of a `const` must not contain a pointer
into the compile-time heap (an error otherwise). The final value of a
`static` may: every heap block reachable from it is placed in the program's
read-only static data, and the static's value refers to that data. Heap blocks
not reachable from the final value are discarded without running drops.

```tally
fn make_keywords() -> Vec[&str] ! AllocError may alloc { ... }
static KEYWORDS: Vec[&str] = make_keywords().unwrap();   // fine: the buffer becomes static data
const K2: Vec[&str] = make_keywords().unwrap();          // error: const holds a heap pointer
```
*From: D40.*

**16.1.4** Evaluation is deterministic: its result depends only on the source,
the target and the configuration (features, profile). An implementation must
support evaluations of at least 10 000 000 primitive steps and 64 MiB of
compile-time heap per constant; it may document larger limits. Exceeding its
limits is an error.

```tally
const SLOW: u64 = count_primes(1_000_000_000);   // may exceed the step limit: error
```
*From: D40, D2.*

**16.1.5** Stack bounds (11.5.2) are link-time constants, not constant
expressions; using them in a constant expression is an error.

```tally
const S: usize = stack.bound_required(&f);   // error: not available at compile time
```
*From: D33, D40.*

## 16.2 Conditional compilation

**16.2.1** The attribute `@cfg(p)` may be placed on an item, import, struct
field, enum variant, trait or impl member, match arm, or statement (including
a block expression statement). If the predicate `p` is false for the current
build, the element is removed before name resolution and is not checked
further (it must still parse). If it is true, the attribute has no effect.

```tally
@cfg(os("linux")) fn page_size() -> usize { 4096 }
@cfg(not(os("linux"))) fn page_size() -> usize { sys_page_size() }
fn f() {
    @cfg(feature("trace")) { trace("entering f"); }
}
```
*From: D42.*

**16.2.2** A predicate is one of:

| Predicate | True when |
|---|---|
| `true`, `false` | always, never |
| `os("x")` | the target operating system is `x` (21.6.2: `linux`, `macos`, `windows`, `freebsd`, `none`, …) |
| `family("x")` | the target belongs to family `x` (`unix`, `windows`) |
| `arch("x")` | the target architecture is `x` (`x86_64`, `aarch64`, `riscv64`, `riscv32`, `arm`, `wasm32`, …) |
| `env("x")` | the target environment/C library is `x` (`gnu`, `musl`, `msvc`, `none`) |
| `endian("little")`, `endian("big")` | the target byte order |
| `pointer_width(n)` | `usize` has `n` bits |
| `feature("f")` | feature `f` of the current package is enabled (18.5) |
| `test` | the package is being built by `tally test` (21.3.4) |
| `profile("p")` | the build profile is `p` |
| `debug_assertions` | the profile's `debug-assertions` setting is true |
| `all(p, ...)`, `any(p, ...)`, `not(p)` | conjunction, disjunction, negation |

An unknown predicate name is an error. A string value not known to the
implementation for that predicate is a warning, and the predicate is false.

```tally
@cfg(all(family("unix"), not(os("macos")))) import std.os.linux;
@cfg(debug_assertions) assert(self.invariant_holds());
@cfg(os("linx")) fn f() {}      // warning: unknown os "linx"
```
*From: D42.*

**16.2.3** The module `core.target` defines constants describing the target:
`OS`, `FAMILY`, `ARCH`, `ENV` (as `&str`), `ENDIAN` (an enum `Endian`),
`POINTER_WIDTH` (`u32`). Code that branches on these constants with an
ordinary `if` is type-checked on every target.

```tally
fn swap_to_le(x: u32) -> u32 {
    if core.target.ENDIAN == .Big { x.swap_bytes() } else { x }
}
```
*From: D42, D2.*

## 16.3 Attributes

**16.3.1** The standard attributes are listed below with the elements they
apply to. An attribute on an element it does not apply to, an unknown
attribute whose name does not begin with `vendor_`, a repeated attribute
that is not repeatable (only `@doc`, `@cfg`, `@allow` and `@derive` are
repeatable), or an argument of the wrong form is an error.

| Attribute | Applies to | Meaning |
|---|---|---|
| `@align(N)` | struct, field | 13.3.3 |
| `@allow(lint, ...)` | item, statement | suppress the named warnings (16.3.5) |
| `@bitpacked(uN)` | struct | 13.3.4 |
| `@cfg(p)` | see 16.2.1 | conditional compilation |
| `@cold` | function | hint that calls are rare |
| `@context_local` | static | 15.5.2 |
| `@default` | unit enum variant | the variant `@derive(Default)` uses |
| `@deprecated(since = "v", note = "...")` | item | uses produce a warning |
| `@derive(Tr, ...)` | struct, enum | 16.3.3 |
| `@doc("...")` | item, field, variant, member, module (via `//!`) | documentation (1.2.3) |
| `@drop_ignores(T)` | `unsafe impl` of `Drop` | 9.6.2 |
| `@entry` | function | freestanding entry point (17.3.1) |
| `@export` / `@export("sym")` | public foreign-ABI function | 14.3.1 |
| `@global_allocator` | static | the program's global allocator (19.13.5) |
| `@inline(always)`, `@inline(never)`, `@inline` | function | inlining request; `always` and `never` must be obeyed |
| `@interrupt("NAME")` | function | 17.4.1 |
| `@link_name("sym")` | foreign function or static | 14.1.2 |
| `@non_exhaustive` | public struct or enum | 6.4.3, 16.3.4 |
| `@packed` | struct | 13.3.3 |
| `@panic_handler` | function | 12.4.4 |
| `@recursion_limit(N)` | function | 11.4.1 |
| `@section("name")` | function, static | place in the named object-file section |
| `@stack_bound(N)` | foreign function | 14.1.5 |
| `@stack_limit(N)` | function | 11.5.1 |
| `@thread_local` | static | 15.5.1 |
| `@track_caller` | function | panics and `source.location()` inside it report its caller's location (19.18.4) |
| `@used` | static | keep the symbol even if unreferenced |

```tally
@inline(never) @cold fn report_failure(code: u32) may sys | block { ... }
@section(".isr_vector") @used static VECTORS: [16]?extern "C" fn() = [...];
```
*From: D11, D2.*

**16.3.2** Attributes whose names begin with `vendor_` are reserved for
implementation extensions. An implementation ignores (and may warn about)
`vendor_` attributes it does not recognise. A vendor attribute must not change
the meaning of a well-formed program in a way this standard defines; it may
only affect diagnostics, optimisation, or implementation-defined behaviour.

```tally
@vendor_acme_hot fn fast_path() {}
```
*From: D11, D2.*

**16.3.3** `@derive(...)` accepts `Copy`, `Clone`, `Eq`, `Ord`, `Hash`,
`Default` and `Debug`. For a generic type, each derived impl requires the
same trait for every type parameter. The generated behaviour is:

* `Copy`: values are copied bitwise; all fields must be `Copy`; deriving
  `Copy` also derives `Clone` (by copying) whether or not `Clone` is listed.
* `Clone`: clones each field in declaration order (for enums, the active
  variant's fields), returning the first error if one fails; the `Clone`
  effect is the union of the fields' `Clone` effects.
* `Eq`: makes the type comparable (5.5.1).
* `Ord` (requires `Eq`): compares field by field in declaration order, the
  first unequal field deciding; for enums, compares tag values first (in
  declaration order when no tag values are given), then fields.
* `Hash`: feeds each field to the hasher in declaration order; for enums, the
  tag value (as `u64`) first.
* `Default`: for a struct, every field's `Default`; for an enum, the unit
  variant marked `@default` (required).
* `Debug`: writes the type name, then `{ f1: v1, f2: v2 }` for named fields,
  `(v1, v2)` for positional fields, or nothing for unit structs and variants
  (enum values write the variant name instead of the type name); field values
  use `Debug`; strings and characters are written quoted with escapes as in
  1.4.4.

```tally
@derive(Clone, Eq, Ord, Hash, Debug)
struct Version { major: u32, minor: u32 }
// Debug output: Version { major: 1, minor: 4 }
@derive(Default) enum Level { Low, @default Mid, High }
```
*From: D41, D17.*

**16.3.4** `@non_exhaustive` on a public enum is described in 6.4.3. On a
public struct, it prevents other packages from building the struct with a
struct literal or destructuring it without `..`, so that fields can be added
in a compatible version.

```tally
@non_exhaustive pub struct Options { pub verbose: bool }
let o = Options { verbose: true };        // error in another package: use a constructor
```
*From: D44.*

**16.3.5** `@allow(name, ...)` suppresses the named warnings within the
element it is attached to. The standard warnings that every implementation
must recognise by name (and may emit) are: `unused` (unused bindings, imports,
items, results of `_ = ...` are not warned), `unreachable_code`, `deprecated`,
`unknown_cfg`, and `effect_reserved` (a public function's declared effects
are larger than its body's, 10.4.3). Unknown names are a warning.

```tally
@allow(effect_reserved)
pub fn lookup(k: u32) -> ?u32 may block { TABLE.get(k) }
```
*From: D2, D29.*
