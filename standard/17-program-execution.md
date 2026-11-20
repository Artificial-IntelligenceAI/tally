# 17. Program execution

## 17.1 Hosted programs

**17.1.1** A *binary target* (18.3.3) must have a function named `main` in its
root module, with no parameters and no generic parameters, whose return type is
one of `()`, `u8`, `() ! E` or `u8 ! E`, where `E` implements `Display`. `main`
may have any effects and need not be `pub`.

```tally
fn main() ! AppError may sys | block | alloc | panic {
    let cfg = load_config()?;
    serve(cfg)
}
```
*From: D61.*

**17.1.2** On hosted targets (operating system other than `none`), before
`main` runs, the runtime of `std` performs, in this order: setting up the main
thread's execution-context control block (including context-local statics,
15.5.2) and its signal stack (20.9.2); setting the disposition of `SIGPIPE`
to "ignore" on Unix targets, so that writing to a closed pipe returns an error
instead of killing the process; recording the command-line arguments and
environment; seeding the process-wide hash key from the operating system's
random source (19.12.2); and opening the standard streams. No user-written code
runs before `main`: statics have compile-time values (3.5.2).

```tally
fn main() {
    // runtime setup is already done; there are no user constructors
    for a in process.args() { io.println(a); }
}
```
*From: D61, D2.*

**17.1.3** When `main` returns, its bindings have already been dropped (as for
any return). Then the runtime: if `main` returned `.Err(e)`, writes `error: `,
the `Display` output of `e`, and a line end to standard error; flushes the
buffered standard output stream (20.2.4), ignoring errors; and ends the
process with exit status 0 for `()` or `.Ok(())`, the returned `u8` for `u8`
or `.Ok(n)`, and 1 for `.Err(_)`. Other threads still running are terminated
without running their drops.

```tally
fn main() -> u8 { if check() { 0 } else { 3 } }     // exit status 3 on failure
```
*From: D61.*

**17.1.4** `std.process.exit(code)` flushes standard output, ignoring
errors, and ends the process with `code` without dropping anything.
`core.process.abort()` ends the process abnormally (on POSIX targets by
`SIGABRT`) without flushing or dropping anything. A panic ends the process
through the panic handler (12.4.1).

```tally
if args.len() < 2 { io.eprintln("usage: tool FILE"); process.exit(2); }
```
*From: D61, D21.*

## 17.2 Libraries

**17.2.1** A library has no entry point. Its public items are used by
dependent packages (chapter 18). A library may also be built as a static or
shared C library (21.3.2), in which case only its exported (14.3.1) functions are
visible to foreign code, and the program that links it provides the panic
handler and global allocator unless the library is built with `std`, which
provides them.

```tally
@unsafe(export("png_decode")) pub extern "C" fn decode(p: *u8, n: usize, out: *mut Image) -> c_int may alloc | panic { ... }
```
*From: D38, D61.*

## 17.3 Freestanding programs

**17.3.1** On targets whose operating system is `none`, a binary cannot use
`std` and does not have `main`. Instead it has exactly one function marked
`@entry`, with no parameters and return type `never` (and any effects), which is called after the platform start-up
code has initialised memory. The program must define a `@panic_handler`
(12.4.4), and, if any function in it has the `alloc` effect, a
`@global_allocator` (19.13.5); otherwise linking fails.

```tally
@entry
fn reset() -> never may sys | panic {
    board.init();
    loop { poll_sensors(); cpu.wait_for_interrupt(); }
}
```
*From: D61, D21.*

**17.3.2** For each freestanding target it supports, the implementation
provides start-up code that sets the initial stack pointer to the stack area
named in the target's linker script, copies initialised static data from its
load address to its run address, zeroes zero-initialised static data, sets up
the execution-context control block for the entry context, and calls the
`@entry` function. A program may replace the start-up code by defining its own
start symbol in assembly (through a native source, 18.2.6), in which case it
must perform the same steps before calling the `@entry` function.

```text
start-up: sp = __stack_top; copy .data; zero .bss; init context block; call reset()
```
*From: D61, D45.*

## 17.4 Interrupt handlers

**17.4.1** `@interrupt("NAME")` on a function with no parameters returning
`()` makes it the handler for the interrupt or exception `NAME`, using the
target's interrupt calling convention. The valid names are defined by the
target (21.6.3). The implementation places the handler's address in the
target's vector table or exports it under the target's conventional symbol
name. Each name may have at most one handler in a program.

```tally
@interrupt("SysTick")
fn tick() may sys { TICKS.fetch_add(1, .Relaxed); }
```
*From: D61, D3.*

**17.4.2** The effect set of an interrupt handler must not contain `alloc` or
`block`; it may contain `sys`, `panic` and user effects. Every interrupt
handler runs in an execution context of its own for the purposes of recursion
limits and context-local statics (15.1.1), which the implementation keeps per
interrupt priority level.

```tally
@interrupt("UART0") fn rx() may sys | block { queue.push_blocking(read_byte()); }
// error: an interrupt handler may not have the effect `block`
```
*From: D3, D28.*

**17.4.3** An interrupt handler is entered asynchronously. Data shared between
an interrupt handler and other code must be accessed through atomic types or
through other types whose documentation states they are interrupt-safe; the
implementation treats an interrupt as a
separate thread for the purposes of the memory model (15.3).

```tally
static EVENTS: Atomic[u32] = Atomic.new(0);
@interrupt("GPIO") fn gpio() may sys { EVENTS.fetch_or(1, .Release); }
```
*From: D35, D61.*
