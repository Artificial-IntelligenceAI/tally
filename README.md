# Tally

Tally is a systems programming language for software in which *what the code
does to the machine* is part of whether it is correct: kernels and drivers,
firmware, real-time audio and control, runtimes and allocators, databases,
network stacks and servers. It compiles ahead of time to native code, has no
garbage collector, is memory-safe and data-race-free outside explicit `unsafe`
blocks, and calls and is called by C at no cost.

This repository holds the **Tally 1 standard**, a specification precise enough
to implement the language, its core and standard libraries, and its toolchain
from. No compiler exists yet; the plan for building one is decision D7.

```tally
import std.io;
import std.fiber;
import std.net.{TcpListener, SocketAddr};

/// Runs on the audio thread: the compiler proves this cannot allocate,
/// wait, crash or make a system call, through any library it calls.
fn mix(out: &mut []f32, voices: &[]Voice) may none { ... }

/// The `may` clause is part of the public contract and is checked.
pub fn serve(addr: SocketAddr) ! io.Error may sys | block | alloc | panic {
    let l = TcpListener.bind(addr)?;
    fiber.run(|s| -> () ! io.Error {
        loop {
            let (conn, _) = l.accept()?;           // suspends only this fiber
            _ = s.spawn(move || handle(conn));     // stack sized exactly, at link time
        }
    })?
}
```

## Why this design

Most expensive bugs in systems code are not wrong answers. They are an
allocation in an interrupt handler, a lock inside an audio callback, a stack
overflow on a small kernel stack, a crash in a path that had to survive, a
dependency that quietly started doing network I/O. They have one cause: the
code did something to the machine that its author could not see. So Tally
ranks its values (D2) with **predictability first**: a reader and the
compiler can tell from source and signatures whether code allocates, waits,
can crash, touches the outside world, how much stack it needs, and how its
data is laid out. Memory safety comes second, because it is necessary but not
sufficient; then local reasoning, control, and toolability.

Everything else follows from that ranking: checked arithmetic with the same
meaning in every build mode, fully specified data layout, errors as values
with no hidden conversions, no operator overloading or macros, no code before
`main`, deterministic destruction, correctly rounded maths, reproducible
builds, and a borrow checker in the style of Rust's but without lifetime
annotations (a `from` clause says where a returned reference comes from, and
`view` types are the ones that hold references).

## The major feature: cost accounting

Every Tally function has a **cost**, and the toolchain checks it.

* **Effects.** `alloc`, `block`, `panic`, `sys`, plus effects libraries declare,
  written in every public signature with `may`, inferred inside a package,
  propagated through generics, closures, trait objects and destructors. A
  function declared `may none` provably does none of them, through every call
  in every dependency.
* **Stack bounds.** At link time the toolchain computes each function's
  worst-case stack use from the real machine code over the whole call graph,
  including function pointers, trait objects and foreign code. Recursion is
  bounded by `@recursion_limit`. Programs assert bounds (`@stack_limit(4096)`)
  and read them (`stack.bound(f)`).
* **Concurrency without `async`.** Because entry functions' stack bounds are
  known, the standard library gives each fiber exactly the stack it needs. Code
  that blocks just blocks: on a thread it waits, on a fiber it suspends, with
  no function colouring, no futures and no `Pin`.

**Why adopt Tally over C, C++, Rust, Zig or Go?** Because "this must not
allocate / block / crash / overflow" becomes a compile or link error instead
of a code-review convention, which removes the hardest class of systems bugs;
because dependency upgrades that add costs show up as breaking API changes
(`tally api diff`) and `tally audit` shows which dependencies can touch the
operating system; because one style of blocking code serves threads and
lightweight tasks alike; and because this only works if *every* library is
annotated from the start, which an existing ecosystem cannot retrofit. The
honest costs (longer signatures, `panic` on most arithmetic, conservative and
target-specific stack bounds, fibers that hold their peak stack, a new
ecosystem) are recorded in D3 and throughout `DECISIONS.md`.

## How the standard is organised

`standard/` contains numbered chapters. Each rule is numbered
(`chapter.section.rule`), shows a short example, and names the decisions it
follows from; diagnostics cite rule numbers.

| Chapter | Contents |
|---|---|
| [0](standard/00-introduction.md) | Scope, conformance, notation, overview |
| [1](standard/01-lexical.md)–[2](standard/02-grammar.md) | Lexical structure; complete grammar |
| [3](standard/03-modules-and-names.md)–[7](standard/07-traits-and-generics.md) | Modules and names; types; expressions; patterns; traits and generics |
| [8](standard/08-ownership.md)–[9](standard/09-borrowing.md) | Ownership and drops; references and borrow checking |
| [10](standard/10-effects.md)–[11](standard/11-stack-bounds.md) | Cost accounting: effects; stack bounds |
| [12](standard/12-errors-and-panics.md) | Errors and panics |
| [13](standard/13-unsafe-and-layout.md)–[14](standard/14-ffi-and-assembly.md) | Unsafe code, layout, undefined behaviour; FFI and inline assembly |
| [15](standard/15-concurrency.md) | Execution contexts, Send/Sync, memory model, fibers |
| [16](standard/16-compile-time-and-attributes.md)–[17](standard/17-program-execution.md) | Compile-time evaluation, `@cfg`, attributes; program execution |
| [18](standard/18-packages.md) | Packages, manifest, version selection, lock file |
| [19](standard/19-core-library.md)–[20](standard/20-std-library.md) | The `core` and `std` libraries |
| [21](standard/21-toolchain.md) | The `tally` command |
| [22](standard/22-open-questions.md) | Open questions, with the interim answers implementations must use |

[`DECISIONS.md`](DECISIONS.md) is the numbered decision log: what was chosen,
what else was weighed, and why. It starts with the language's purpose (D1),
values (D2) and major feature (D3).

## Licences

Code (the future compiler, libraries, tools, and code examples in the
documentation) is under Apache-2.0 WITH LLVM-exception
([`LICENSE-CODE`](LICENSE-CODE)); the standard and other documentation are
under CC BY 4.0 ([`LICENSE-DOCS`](LICENSE-DOCS)). The reasons are in D5.
