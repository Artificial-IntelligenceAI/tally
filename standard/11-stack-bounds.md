# 11. Stack bounds

This chapter defines the stack half of cost accounting (D3, D33). Every
function in a linked program has a *stack bound*: either a number of bytes
that no execution of the function, including everything it calls, can exceed
on its execution context's stack, or `unbounded`. Bounds are computed by the
toolchain when it links a program, from the generated code.

## 11.1 Frames

**11.1.1** The *frame size* of a function instance in the generated code is
the largest amount, in bytes, by which the instance's own instructions move
the stack pointer below its value at the call instruction that entered the
instance, including the return address or link register save, saved
registers, local storage, spill slots, outgoing argument areas, alignment
padding (including the worst case of any dynamic realignment of the stack
pointer for over-aligned locals), and any red zone it uses, but excluding
what its callees use. Code
inlined into an instance is part of that instance's frame. Frame sizes are
implementation-defined, and must be the same in every build with the same
compiler version, target, profile and source.

```text
fn leaf(x: u64) -> u64 { x * 3 }       // frame size might be 8 on x86_64 (return address only)
fn mid() -> u64 { let a: [256]u8 = [0; 256]; leaf(a.len() as u64) }   // frame ≥ 256 + 8
```
*From: D33, D7.*

**11.1.2** A Tally program never adjusts its stack by an amount determined at
run time: there are no variable-length arrays or `alloca`. An implementation
must not generate code whose stack usage depends on run-time values. The
context switch `core.context.switch` (19.16.2) is a call-graph node whose
frame size the implementation declares, like a foreign function with
`@stack_bound`; every function that can suspend a fiber reaches it, so its
use of the fiber's stack is part of their bounds (15.4.3).

```tally
fn f(n: usize) { let a: [n]u8 = ...; }    // error: array length must be a constant (4.7.1)
```
*From: D33, D2.*

## 11.2 The call graph

**11.2.1** The *call graph* of a program has a node for every function
instance in the linked program, including drop glue, instances from `core`
and `std`, runtime routines and compiler helper routines (memory copy,
wide division, stack probing), and a node for every foreign function the
program calls. It has these edges:

1. a *direct edge* from `f` to `g` for every call instruction in `f`'s
   generated code whose target is `g`, and for every jump to the entry of
   `g` (a tail call);
2. an *indirect edge* from `f` to every *address-taken* function `g` whose
   type is compatible with a function pointer type through which `f` makes a
   call; a function is address-taken if it is coerced to a function pointer
   anywhere in the program (including capture-free closures), or is exported
   (14.3.1); compatible means equal parameter and return types after erasing
   regions and effects, and equal calling convention, where a safe function
   is compatible with both safe and `unsafe` pointer types;
3. a *dynamic edge* from `f` to the implementation of method `m` for every
   type `U` that the program, in any instance, coerces to `dyn Tr` or to
   `dyn Sub` for a trait `Sub` with supertrait `Tr` (with any bindings), for
   every call of `m` through such a `dyn` value in `f`, and to `U`'s drop glue
   for every drop of an owner of such a `dyn` value in `f`. (There is no
   coercion from one `dyn` type to another.)

```tally
fn a() {} fn b() { big_stack(); }
let table: [2]fn() = [a, b];          // a and b are address-taken
table[i]();                           // indirect edges to a, b, and every other address-taken fn()
```
*From: D33.*

**11.2.2** A direct call to a foreign function is bounded only if the
function's declaration has the attribute `@stack_bound(N)` (14.1.5): the
callee then has frame size N and no outgoing edges. Otherwise the callee is
`unbounded`. Every indirect call through a function pointer type with an
`extern` ABI (`"C"`, `"system"` or any other) is `unbounded`, because its
targets may be foreign.

```tally
extern "C" {
    @stack_bound(512) fn write(fd: i32, p: *u8, n: usize) -> isize may sys | block;
    fn qsort(base: *mut u8, n: usize, sz: usize, cmp: extern "C" fn(*u8, *u8) -> i32) may sys;
}
// write: bounded by 512; qsort: unbounded
```
*From: D33, D38.*

**11.2.3** An `asm` expression adds to its function's frame size the number
of bytes given in its `stack` clause (14.4.4), default 0, and contributes no
edges. Assembly that calls other code must account for that code's stack in
its `stack` clause.

```tally
unsafe { asm may sys stack 16 ("push rax; pop rax") }
```
*From: D33, D39.*

## 11.3 Computing bounds

**11.3.1** Compute the strongly connected components of the call graph. A
component is *recursive* if it has more than one node or a node with an edge
to itself. A recursive component is *limited* if every cycle in it passes
through at least one function with `@recursion_limit` (11.4.1); otherwise it
is *unlimited*.

```tally
fn eval(e: &Expr) -> i64 may panic { match e { .Add(a, b) => eval(a) + eval(b), .Lit(n) => *n } }
// {eval} is recursive and unlimited: eval is unbounded
```
*From: D33.*

**11.3.2** Bounds are computed in reverse topological order of the
components (callees first):

* A node is `unbounded` if it is an undeclared foreign function, belongs to
  an unlimited recursive component, or has an edge to an `unbounded` node.
* For a node `f` in a non-recursive component:
  `bound(f) = frame(f) + max(0, max over edges f→g of bound(g))`.
* For every node `f` in a limited recursive component `S`, with
  `L = Σ N_h` over the functions `h ∈ S` with `@recursion_limit(N_h)`, and
  `F = Σ frame(g)` over all `g ∈ S`:
  `bound(f) = (L + 1) × F + max(0, max over edges h→k with h ∈ S, k ∉ S of bound(k))`.

```tally
@recursion_limit(64)
fn eval(e: &Expr) -> i64 may panic { ... eval(a) + eval(b) ... }
// S = {eval}, L = 64, F = frame(eval):  bound(eval) = 65 × frame(eval) + max bound of its other callees
```
*From: D33.*

**11.3.3** The bound formula for limited components is sound because along
any chain of activations inside the component, between two consecutive
activations of limited functions no function repeats (otherwise a cycle would
avoid every limited function), so each such segment uses at most `F` bytes,
and there are at most `L + 1` segments. Implementations may compute a smaller
bound by any method that is sound for every execution permitted by this
standard, but must document it; a program's accepted behaviour (11.6) must not
depend on which sound method is used, except through the values reported.

```text
segments:  [g1 g2 eval] [g3 eval] [g2 eval] ... at most L+1 segments, each ≤ F bytes
```
*From: D33.*

## 11.4 Recursion limits

**11.4.1** `@recursion_limit(N)` on a function `f`, where N is a positive
constant, limits to N the number of activations of `f` that may be in
progress at once on one execution context's stack (15.1.1). Entering `f`
when N activations are already in progress on the current execution context
panics before `f`'s body begins. A function with `@recursion_limit` must
include `panic` in its effect set. Activations entered through any route
(direct, indirect, `dyn`) count. Each instance of a generic function has its
own counter, with the same N. A function with `@recursion_limit` is never
inlined, so that it remains a node of the call graph.

```tally
@recursion_limit(32)
fn parse_value(p: &mut Parser) -> Value ! ParseError may panic | alloc {
    match p.peek() { b'[' => parse_array(p), _ => parse_scalar(p) }
}
fn parse_array(p: &mut Parser) -> Value ! ParseError may panic | alloc { ... parse_value(p)? ... }
// a 33-deep nested array panics instead of overflowing the stack
```
*From: D33, D21.*

**11.4.2** The count of activations is kept per execution context; how it is
stored is implementation-defined (a hidden parameter threaded through the
component, or context-local storage). The cost is one increment, one
comparison and one decrement per activation of `f`.

```tally
// conceptually:
// on entry to f:  if ctx.depth_f == N { panic("recursion limit") }; ctx.depth_f += 1
// on exit:        ctx.depth_f -= 1
```
*From: D33, D2.*

## 11.5 Using bounds in programs

**11.5.1** `@stack_limit(N)` on a function asserts that its bound (the value
of 11.3.2, without `CALL_OVERHEAD`) is not `unbounded` and is at most N
bytes. The assertion is checked when a program containing the function is
linked, for that program's target and profile, for every instance of a
generic function, and also for functions that nothing in the program calls;
a violation is an error. The diagnostic must give the computed bound (or the
reason it is unbounded) and a deepest path through the call graph.

```tally
@stack_limit(2048)
fn on_packet(p: &Packet) may none { route(p); }
// link error: bound of on_packet is 2312 bytes > 2048
//   on_packet (96) -> route (160) -> lookup_v6 (2056)
```
*From: D3, D33.*

**11.5.2** The library function `core.stack.bound(f) -> ?usize` (19.16.1)
returns the bound of calling `f`, where `f` is a function item, a closure, or a
function pointer (for which the bound is that of an indirect call through its
type), plus `core.stack.CALL_OVERHEAD`, the frame of the runtime's entry
trampoline that calls a context's entry function; or `.None` if unbounded.
Passing a value whose type is not callable is an error. `core.stack.bound_required(f) -> usize` is the same but
makes linking fail if the bound is `unbounded`. Their results are link-time
constants: they may be used at run time but not in compile-time evaluation
(16.1).

```tally
let job = move |_| handle_conn(c);
let need = stack.bound_required(&job);                 // e.g. 3104 on this target
let fib = scope.spawn_sized(need + stack.CONTEXT_OVERHEAD, job)?;
```
*From: D33, D34.*

**11.5.3** When a library function needs the bound of a callable it receives
as a generic parameter (for example `fiber.spawn[F: FnOnce()]`), the bound is
that of the instance `F.call_once` in the program being linked.

```tally
// in std.fiber (20.8.2):
pub fn spawn[T: Static, G: FnOnce(&Scope[E]) -> T + Static](&self, g: G) -> Fiber[T] ! SpawnError from self may alloc | sys | G {
    let size = stack.bound_required(&g) + stack.CONTEXT_OVERHEAD;   // bound of G.call_once in this program
    ...
}
```
*From: D33, D34.*

## 11.6 Execution contexts and overflow

**11.6.1** The runtime creates every execution context it provides (the main
thread is created by the operating system) with a stack of at least the bound
of its entry function plus `CONTEXT_OVERHEAD` (19.16.1), when that bound is
known and the creating API asks for exact sizing (20.7.1, 20.8.2). Code
running in such a context cannot overflow its stack.

```tally
// thread.spawn(f) with a bounded f allocates bound(f) + CONTEXT_OVERHEAD bytes, rounded to a page
```
*From: D33, D34.*

**11.6.2** For an execution context whose entry function is `unbounded`, or
whose stack size was chosen by the program or the operating system, the
implementation must detect stack overflow (for example with a guard page)
and treat it as a panic with the message `stack overflow` (12.4.1). Stack
overflow is never undefined behaviour in a program without unsafe code.

```tally
fn deep(n: u64) -> u64 may panic { if n == 0 { 0 } else { 1 + deep(n - 1) } }
// unbounded; on a thread with a 2 MiB stack, deep(10_000_000) ends in the panic "stack overflow"
```
*From: D33, D2.*

**11.6.3** Asynchronous signal handlers installed through `std` run on a
separate per-thread signal stack (20.9.2), so they do not consume the
interrupted context's stack. On targets where interrupt handlers run on the
interrupted stack (17.4), the toolchain computes at link time
`core.stack.INTERRUPT_RESERVE`: the sum, over the target's interrupt priority
levels, of the largest bound among the handlers at that level plus the size
of the exception frame the hardware pushes (`unbounded` if any handler is).
On such targets `CONTEXT_OVERHEAD` includes `INTERRUPT_RESERVE`, so exactly
sized fiber and thread stacks remain safe, and the implementation's start-up
code checks at link time that the main stack area is at least
`bound(@entry function) + INTERRUPT_RESERVE`.

```tally
@stack_limit(1024) @interrupt("UART0") fn uart() may sys { ... }
// linker script reserves STACK_SIZE; link fails if
//   STACK_SIZE < stack.bound(reset) + stack.INTERRUPT_RESERVE
```
*From: D33.*

**11.6.4** Calling, through a function pointer with the Tally calling
convention or a `dyn` value, code that is not part of the program's call graph (for example a pointer obtained from
foreign code by transmutation) is undefined behaviour, because bounds that the
program relies on may be wrong.

```tally
let f: fn() = unsafe { mem.transmute(dlsym(h, "plugin")) };   // UB to call f
let g: extern "C" fn() = unsafe { mem.transmute(dlsym(h, "plugin")) };  // fine: extern "C" calls are unbounded
```
*From: D33, D36.*
