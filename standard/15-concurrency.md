# 15. Concurrency

## 15.1 Execution contexts

**15.1.1** An *execution context* is a stack together with the sequence of
function activations running on it. The execution contexts are: the main
thread; each thread created by `std.thread` (20.7); each fiber created by
`std.fiber` (20.8); and, on freestanding targets, each interrupt context
(17.4). Every activation runs in exactly one execution context, and recursion
limits (11.4.1) and context-local statics (15.5.2) are per execution context.

```tally
fn main() {                                  // main thread
    thread.scope(|s| {
        s.spawn(|| work());                  // a second thread
        fiber.run(|fs| { fs.spawn(|| io_task()); });   // fibers on the current thread
    });
}
```
*From: D34.*

**15.1.2** Threads may run in parallel. Fibers belonging to one scheduler run
on one thread, one at a time; control passes from one fiber to another only at
a *suspension point*: a call to a waiting primitive (20.1.3) or to
`fiber.yield_now()`. Code between suspension points in a fiber is never
interleaved with other fibers of the same scheduler.

```tally
fs.spawn(|| { counter.set(counter.get() + 1); });   // no other fiber of this scheduler runs in between
```
*From: D34.*

**15.1.3** A function that spawns a thread or fiber running a callable `F`
includes `F`'s effects in its own effect set: the spawned work is charged to
the spawner. This keeps every effect a program can cause visible in the
effect set of some caller, up to `main`.

```tally
pub fn start_logger(q: Queue) ! SpawnError may alloc | sys | block | disk_write {
    _ = thread.spawn(move || drain_to_disk(q))?;     // drain_to_disk may disk_write | sys | block
}
```
*From: D3, D34.*

## 15.2 Send and Sync

**15.2.1** `Send` and `Sync` (in `core.marker`) are `unsafe` marker traits.
A type is `Send` if a value of it may be moved to another thread, and `Sync`
if a shared reference to it may be used from several threads at once
(equivalently, `&T` is `Send`). The compiler implements both automatically:
a struct, enum, tuple, array or closure type is `Send` (`Sync`) if all its
fields, elements or captures are. Primitive types are `Send` and `Sync`.
`&T` is `Send` and `Sync` if `T` is `Sync`; `&mut T` is `Send` if `T` is
`Send` and `Sync` if `T` is `Sync`. Raw pointers, `UnsafeCell[T]` (for
`Sync`), `core.marker.NotSend` and `core.marker.NotSync` are not.

```tally
struct Job { id: u64, data: Vec[u8] }       // Send and Sync automatically
struct Local { p: *mut u8 }                 // neither: contains a raw pointer
```
*From: D26.*

**15.2.2** A type may be declared `Send` or `Sync` despite its parts with
`unsafe impl Send for X {}`, which promises that moving (sharing) it between
threads cannot cause a data race. A type opts out of the automatic impls by
containing a field of type `NotSend` or `NotSync` (zero-sized).

```tally
pub struct Spin[T] { locked: Atomic[bool], value: UnsafeCell[T] }
unsafe impl[T: Send] Sync for Spin[T] {}
pub struct Guard { _not_send: NotSend, ... }     // e.g. a lock guard tied to its thread
```
*From: D26, D36.*

**15.2.3** Values moved into a thread must be `Send`; the closure passed to
`thread.spawn` must be `Send`; statics must be `Sync` (3.5.2). Fibers do not
require `Send`, because they do not leave their thread (D34), but a callable
passed to a scheduler on another thread does.

```tally
let rc = Rc.new(5)?;
thread.spawn(move || print(*rc));      // error: Rc[i64] is not Send
fiber.run(|fs| { fs.spawn(|| print(*rc)); });   // fine: same thread
```
*From: D26, D34.*

## 15.3 The memory model

**15.3.1** Tally's memory model for concurrent access is the one of C++20
(ISO/IEC 14882:2020, [intro.multithread] and [atomics.order]). The atomic
types of `core.atomic` (19.11) correspond to `std::atomic`, and the
`Ordering` values `Relaxed`, `Acquire`, `Release`, `AcqRel`, `SeqCst` to
`memory_order_relaxed`, `_acquire`, `_release`, `_acq_rel` and `_seq_cst`.
`atomic.fence(o)` corresponds to `std::atomic_thread_fence(o)` and
`atomic.compiler_fence(o)` to `std::atomic_signal_fence(o)`. Thread creation
and joining, and every synchronisation primitive in `std.sync`, establish the
happens-before relations their documentation states.

```tally
static READY: Atomic[bool] = Atomic.new(false);
static var DATA: u64 = 0;
// producer:
unsafe { DATA = 42; }
READY.store(true, .Release);
// consumer:
while !READY.load(.Acquire) { hint.spin(); }
let d = unsafe { DATA };      // 42: the store happens-before this load
```
*From: D35.*

**15.3.2** Two accesses to the same memory location *conflict* if at least
one is a write. A *data race* is a pair of conflicting accesses from different
threads, not both atomic, neither of which happens before the other. A data
race is undefined behaviour (13.2.1). Programs without unsafe code cannot
contain data races, because shared mutable access requires `Sync` types,
which synchronise.

```tally
var n = 0;
thread.scope(|s| {
    s.spawn(|| n += 1);
    s.spawn(|| n += 1);       // error: n captured by &mut in two closures at once (9.5.5)
});
```
*From: D35, D26.*

**15.3.3** Fibers of the same scheduler never race with each other, because
they never run at the same time (15.1.2). A fiber and a thread, or fibers of
different schedulers, can race like threads.

```tally
fiber.run(|fs| {
    let c = Cell.new(0);
    fs.spawn(|| c.set(c.get() + 1));
    fs.spawn(|| c.set(c.get() + 1));     // fine: Cell is not Sync, but fibers share a thread
});
```
*From: D34, D35.*

## 15.4 Fibers and waiting

**15.4.1** The language's support for fibers consists of: execution contexts
(15.1.1); exact stack bounds (11.5); the unsafe primitive context switch
`core.context.switch` (19.15.3); and the rule that every waiting primitive is
*fiber-aware* (15.4.2). Schedulers are library code (20.8).

```tally
// the scheduler in std.fiber is ordinary Tally code built on core.context
```
*From: D34.*

**15.4.2** Each execution context has a *waiter*: a pointer, kept in the
context's control block, to an object implementing `core.context.Waiter`
(19.15.4), or none. A waiting primitive (20.1.3) that would block first checks
the current context's waiter. If there is none (the context is a thread), it
blocks the thread through the operating system. If there is one (the context is
a fiber), it hands the waiter what it is waiting for and suspends the fiber
until the waiter resumes it. Foreign functions and `asm` always block the
whole thread.

```tally
fn handle(c: &mut TcpStream) ! io.Error may sys | block {
    var buf: [512]u8 = [0; 512];
    let n = c.read(&mut buf)?;     // in a thread: blocking read(2); in a fiber: suspends the fiber
    c.write_all(&buf[0..n])
}
```
*From: D34.*

**15.4.3** The stack that a fiber uses while it is suspended and while a
waiting primitive switches away is part of the bound of that primitive, and
so of the fiber's entry function. The scheduler's own work happens on the
scheduler's stack, not on fiber stacks.

```tally
// bound(handle) includes read()'s path through the waiter and the switch frame
```
*From: D33, D34.*

## 15.5 Thread-local and context-local statics

**15.5.1** A `static` declared with `@thread_local` has one instance per
thread, initialised with the static's value when the thread starts. Fibers on
a thread share that thread's instance. A reference to a thread-local static is
valid only on its thread: such a reference may not be stored in a value that is
`Send`, and the static's type need not be `Sync`. Thread-local `static var`s
require `unsafe` as usual.

```tally
@thread_local static SCRATCH: Cell[u32] = Cell.new(0);
fn bump() { SCRATCH.set(SCRATCH.get() + 1); }
```
*From: D34, D26.*

**15.5.2** A `static` declared with `@context_local` has one instance per
execution context (thread or fiber), initialised with the static's value when
the context starts. The total size of context-local statics is known at link
time and is reserved in each context's control block, and is included in
`CONTEXT_OVERHEAD`. Context-local statics need not be `Sync`, and references
to them must not escape the context: they have the region of the accessing
function's body.

```tally
@context_local static REQUEST_ID: Cell[u64] = Cell.new(0);
fn log(msg: &str) may sys | block { io.eprintln("[", REQUEST_ID.get(), "] ", msg); }
```
*From: D34.*
