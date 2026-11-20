# 22. Open questions

This chapter lists what is still undecided. Each question has an *interim
answer*, which is normative: implementations of Tally 1 must follow it, and
programs may rely on it. A later revision may change an interim answer only in
the way the entry says it might, and a change that breaks programs requires a
new major language version (18.7.1). Each entry also says what evidence would
settle the question.

## 22.1 Language

**22.1.1** *Can view types hold more than one region?* Interim answer: no;
every view type has exactly one region (9.5.1). Might change to: optional named
region parameters on view declarations, in addition to the single default
region, if experience shows that common library designs (for example a
parser holding both input and an arena) are blocked. A change would be
backward compatible.

Settled by: whether parser and arena-using libraries written against Tally 1 need workarounds (owned copies, indices) often enough to matter.

```tally
view struct Parser { src: &str, arena: &Bump }   // interim: both share one region
```
*From: D24.*

**22.1.2** *Should the aliasing rules be refined?* Interim answer: the rules
of 13.4.4. Might change to: a more permissive model in the style of Tree
Borrows, which would only turn some programs with undefined behaviour into
programs without it, which is compatible.

Settled by: a formal model of the rules, and evidence from checked execution of real unsafe code that the stricter rule rejects useful programs.

```tally
// interim: any access through a pointer not derived from a live &mut is UB
```
*From: D36.*

**22.1.3** *Should there be immovable types for intrusive data structures?*
Interim answer: no; values may always be moved by copying their bytes, and
intrusive structures use raw pointers and `unsafe` with documented
invariants. Might change to: a `@pinned` attribute forbidding moves after a
value's address is first taken, if kernel and runtime libraries need it often.

Settled by: how much unsafe code kernel and runtime libraries need for intrusive lists, measured once such libraries exist.

```tally
struct ListNode { next: ?*mut ListNode, prev: ?*mut ListNode }   // interim: manual invariants
```
*From: D23, D36.*

**22.1.4** *Should there be compile-time reflection?* Interim answer: no;
derives are limited to 16.3.3 and other generated code uses generators
(18.6). Might change to: read-only reflection over the fields of a type, in
constant expressions, if generator-based serialisation proves too awkward.

Settled by: experience writing serialisation libraries with generators.

```tally
@derive(Debug) struct S { a: u32 }   // interim: only the built-in derives
```
*From: D41.*

**22.1.5** *Should operators be overloadable for numeric library types?*
Interim answer: no (D17). Might change to: a closed set of arithmetic traits
whose implementations must be effect-free, if numeric libraries (vectors,
fixed-point, big integers) prove too verbose.

Settled by: experience with numeric libraries written without it.

```tally
let c = a.add(b);    // interim: method calls for user numeric types
```
*From: D17.*

**22.1.6** *Should there be variadic generics?* Interim answer: rest
parameters (5.7.5) and built-in tuple impls up to arity 16 (7.9.2) only.
Might change to: a way for user traits to be implemented for all tuple
arities.

Settled by: whether libraries need traits implemented for all tuple arities beyond the built-in ones.

```tally
println(a, b, c);     // interim: works through built-in tuple impls of Display
```
*From: D47.*

**22.1.7** *Should effects be more precise for generic code?* Interim answer:
the type term `T` covers all associated effects of `T`'s bounds and its drop
effects (10.2.4); `T.Name` gives precision for one associated effect. Might
change to: a separate term for drop effects (`drop(T)`), which would be
backward compatible.

Settled by: how often generic library signatures over-approximate effects because of drop effects.

```tally
fn f[T: Iterator](t: T) may T { ... }   // interim: covers T.Next and T's drop
```
*From: D30.*

**22.1.8** *Should there be effects for time (worst-case execution time) or
user-defined handlers (capabilities supplied by callers)?* Interim answer:
no. Effects are markers only (D3). Might change to: handler-style user effects
whose handler is passed as an implicit parameter, if a sound design is found
that avoids the allocator problem recorded in D32.

Settled by: a design for caller-supplied capabilities that keeps the effect guarantee sound when values outlive the handler that created them.

```tally
pub effect metrics;   // interim: a marker, with no handler
```
*From: D3, D32.*

**22.1.9** *Should identifiers allow Unicode?* Interim answer: ASCII only
(D8). Might change to: UAX #31 identifiers with a confusables check, which
would be backward compatible.

Settled by: demand from users writing in other languages, and a confusables check that tools can implement cheaply.

```tally
let größe = 1;    // interim: error
```
*From: D8.*

**22.1.10** *Should there be specialisation?* Interim answer: no (D27).

Might change to: nothing planned; settled by evidence that coherence without specialisation blocks important library designs.

```tally
impl[T] Describe for T { ... }
impl Describe for u8 { ... }    // interim: error (overlap)
```
*From: D27.*

## 22.2 Concurrency

**22.2.1** *How are fibers cancelled?* Interim answer: there is no
cancellation; a fiber ends only by returning. Programs cancel cooperatively
with flags or channels and use timeouts on waiting primitives. Might change to:
a `Fiber.cancel()` that makes the target's next waiting primitive return a
`Canceled` error, which would add an error kind (compatible, since
`io.ErrorKind` is `@non_exhaustive`) but would change the error sets of
`Mutex.lock` and `Receiver.recv` (incompatible, so it would require new
functions).

Settled by: experience with servers built on fibers, and whether cooperative cancellation is too error-prone in practice.

```tally
// interim: stop.store(true, .Release); then the fiber notices and returns
```
*From: D34.*

**22.2.2** *Should fibers migrate between threads (work stealing)?* Interim
answer: no; fibers stay on their scheduler's thread (D34). Might change to: an
opt-in scheduler that migrates fibers whose entry closures are `Send`, if
thread-per-core designs prove insufficient for common servers.

Settled by: measurements of load imbalance in thread-per-core servers.

```tally
fiber.run(&env, |s| ...);   // interim: single-threaded scheduler per call
```
*From: D34.*

**22.2.3** *How should stack bounds work across dynamically loaded code?*
Interim answer: calls into code not linked into the program are unbounded
(11.2.2, 11.6.4); Tally shared libraries carry no bound metadata. Might change
to: a metadata section recording exported functions' bounds and indirect-call
signatures, checked when the library is loaded.

Settled by: whether plugin systems in Tally need bounded calls into loaded code.

```tally
let f: extern "C" fn() = unsafe { mem.transmute(dlsym(h, c"plugin".as_ptr())) };   // interim: unbounded
```
*From: D33.*

**22.2.4** *Are thread-local and context-local statics dropped when their
context ends?* Interim answer: no; like all statics they are never dropped
(8.5.3), so their types should not own resources that must be released.
Might change to: running drop glue at context exit, with the drop effects
charged to the context's entry function.

Settled by: whether programs need context-local values that own resources.

```tally
@thread_local static BUF: Cell[u32] = Cell.new(0);   // interim: never dropped
```
*From: D25, D34.*

## 22.3 Library and tools

**22.3.1** *Should there be a central package registry?* Interim answer: no;
dependencies come from git sources and paths only (18.2.3). Might change to: a
registry protocol that maps names to sources and versions to content hashes,
added as a new dependency form (compatible).

Settled by: ecosystem growth making discovery and name squatting practical problems.

```toml
json = { git = "https://example.org/json.git", version = "0.9.1" }   # interim: the only remote form
```
*From: D43.*

**22.3.2** *Should the toolchain import C headers?* Interim answer: no;
foreign declarations are written by hand (or produced by a generator). Might
change to: a `tally bind` command that writes extern blocks from headers, with
effects defaulting to `sys | block | panic` and no stack bounds, for the
programmer to tighten.

Settled by: how much binding code real projects write by hand.

```tally
extern "C" { fn inflate(s: *mut ZStream, flush: c_int) -> c_int may none; }   // interim: by hand
```
*From: D38.*

**22.3.3** *Which further collections belong in `core`?* Interim answer:
only those in 19.14. Might change to: adding `VecDeque`, `BTreeMap`,
`BTreeSet` and `BinaryHeap` (compatible).

Settled by: which collections third-party packages converge on.

```tally
// interim: use a third-party package for ordered maps
```
*From: D46.*

**22.3.4** *Should `core` provide SIMD vector types?* Interim answer: no;
vectorisation is left to the optimiser, `asm`, or foreign code. Might change
to: fixed-width vector types `Simd[T, N]` with element-wise methods and a
specified fallback on targets without SIMD.

Settled by: how often vectorisation by the optimiser falls short in practice.

```tally
// interim: for i in 0..n { out[i] = a[i] * b[i]; }  — the optimiser may vectorise
```
*From: D2, D37.*

**22.3.5** *Should there be `f16`, `bf16` and `f128`?* Interim answer: no.
Might change to: adding them as primitive types with the same rules as `f32`
and `f64`, which is compatible because 1.3.4 already reserves the names.

Settled by: demand from machine-learning and graphics code.

```tally
let h: f16 = 1.0;   // interim: error, unknown type f16
```
*From: D14, D65.*

**22.3.6** *Should panics record backtraces?* Interim answer: no; a panic
reports its message and the location from `@track_caller` (19.18.4). Might
change to: an optional backtrace in `PanicInfo`, collected without
allocation, in profiles that keep frame information.

Settled by: whether a backtrace can be collected without allocation and without frame pointers on all targets.

```text
panic: index out of bounds: the index is 7 but the length is 3
 at src/main.tly:4:11            (interim: no backtrace)
```
*From: D21.*

**22.3.7** *Should the toolchain have benchmark and fuzzing commands?*
Interim answer: no; benchmarks are binary targets and fuzzing uses
implementation-defined options. Might change to: `tally bench` and
`tally fuzz` with specified output formats.

Settled by: whether ecosystem conventions for benchmarks and fuzzers emerge that are worth standardising.

```text
tally run --bin bench --profile release     (interim)
```
*From: D50.*

**22.3.8** *Is there a stable ABI for calls between separately compiled
Tally code?* Interim answer: no; Tally-to-Tally calls across separately
linked binaries must use `extern "C"`. Within one program all Tally code is
compiled by one implementation. Might change to: a stable Tally ABI for a
subset of types.

Settled by: demand for plugins written in Tally and loaded by Tally programs.

```tally
@unsafe(export("plugin_init")) pub extern "C" fn init(api: *HostApi) -> c_int may sys { ... }   // interim
```
*From: D38, D7.*

**22.3.9** *Should signals have a Windows equivalent?* Interim answer:
`std.signal` exists only on `family("unix")` targets; Windows console events
are reached through `std.os.windows`.

Might change to: a portable subset of signals (interrupt and termination) on Windows; settled by demand from portable command-line tools.

```tally
@cfg(family("unix")) signal.set_handler(.Int, on_int)?;
```
*From: D46.*
