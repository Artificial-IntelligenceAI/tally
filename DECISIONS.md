# Tally decision log

Every design decision behind Tally is recorded here, numbered in the order it
was made. Numbers are never reused or renumbered. When a decision is changed,
a new entry is added that names the entry it replaces, and the old entry is
marked *Superseded by Dn* but otherwise left as it was.

Each entry states what was chosen, which alternatives were weighed, and why
the choice won. The standard (`standard/`) cites these numbers next to every
rule, so any rule can be traced back to its reason.

---

## D1. What Tally is for

**Chosen.** Tally is a language for programs in which the machine-level
behaviour of the code is part of its correctness: operating-system kernels and
drivers, embedded firmware, real-time systems (audio, motor control,
trading), language runtimes, allocators, databases and storage engines,
network stacks and servers, game engines. In these programs it is not enough
that the code computes the right answer. It must also not allocate in an
interrupt handler, not block inside an audio callback, not overflow a 4 KiB
kernel stack, not crash in a path that must not fail, and not lay data out
differently from the hardware or the wire format.

Tally is not aimed at scripting, at application code where a garbage collector
is acceptable, or at programs whose main difficulty is expressing business
rules.

**Alternatives weighed.** (a) A general-purpose language that also does
systems work (the position of C++ and, increasingly, Rust). (b) A narrow
language for one domain, such as embedded control (the position of SPARK or
MISRA C).

**Why.** A general-purpose language has to keep every feature that any
domain wants, and the result is a large language whose costs are hard to
see. A single-domain language is too small to build an ecosystem. The set of
programs above shares one property, *the cost of the code matters as much as
its result*, and that property is specific enough to design around while
covering a large and important body of software.

---

## D2. What Tally values, in order

**Chosen.** Five values, ranked. When two conflict, the higher one wins.

1. **Predictability.** A reader, and the compiler, can tell what code does to
   the machine from its source and its signatures: whether it allocates,
   whether it can block, whether it can crash, how much stack it needs, how its
   data is laid out, and which code runs. Nothing happens that is not written
   down somewhere a reader can find without running the program. The same
   source means the same thing in every build mode.
2. **Soundness by default.** Safe code cannot cause undefined behaviour: no
   use-after-free, no data races, no out-of-bounds access, no uninitialised
   reads. Unsafe operations are possible, explicit, and lexically marked.
3. **Local reasoning.** Each piece of code can be understood from itself and
   the signatures of what it uses. The language avoids features whose meaning
   depends on distant code: implicit conversions with user code behind them,
   operator overloading, exceptions, inheritance, global type inference.
4. **Control.** The programmer can choose any data layout, any allocation
   strategy, any concurrency structure, and can call and be called by C at no
   cost.
5. **Toolability.** The grammar is context-free and simple to parse; there is
   one canonical format; packages build reproducibly; the compiler can explain
   every cost it reports.

**Alternatives weighed.** Ranking performance first (C, C++'s "zero-cost
abstractions"); ranking safety first with predictability as a side effect
(Rust); ranking simplicity first (Go, Zig); ranking expressiveness first
(C++, Haskell-influenced designs).

**Why predictability first.** The expensive bugs in systems code are mostly
not wrong answers. They are allocations in the wrong place, locks in the
wrong place, stack overflows, crashes in code that had to survive, latency
that came from a call nobody knew was slow. These all have the same cause:
the code did something to the machine that its author did not see. Memory
safety, which Rust made mainstream, removes one large class of these bugs
and is necessary, so it is second. But a memory-safe program can still
allocate in an interrupt handler. Performance is not on the list on its own
because, in systems code, performance problems are predictability problems:
you cannot fix a cost you cannot see, and once costs are visible, control
(value 4) lets you remove them. Simplicity is expressed as *local reasoning*
rather than "few features" because the property readers need is being able to
understand code without reading other code, and a small language can still
fail at that (for example through hidden dynamic dispatch).

---

## D3. The major feature: cost accounting

**Chosen.** In Tally, every function has a **cost**, and the compiler and
linker check it.

* The **effect** part of a function's cost is a set of things the function
  may do to the machine, beyond computing: `alloc` (use the global heap),
  `block` (wait for an external event), `panic` (crash the program), `sys`
  (talk to the operating system or hardware), plus effects that libraries
  declare for themselves. The compiler infers effects inside a package,
  requires them to be written in every public signature, propagates them
  through generics, closures, trait objects and destructors, and rejects any
  function whose body does more than its signature says. An interrupt handler
  written `fn on_irq() may none` cannot allocate, block, crash or make a
  system call, through any chain of calls, in any library, and this is a
  compile-time fact rather than a code-review convention.
* The **stack** part of a function's cost is its worst-case stack usage in
  bytes. The linker computes it for every function in the program from the
  real machine code, over the whole call graph including indirect calls,
  trait objects and foreign code. Recursion must carry an explicit depth
  limit to be bounded. The program can assert bounds (`@stack_limit(4096)`)
  and read them (`stack.bound(f)`) as link-time constants.

The two parts together make a third thing possible, which is the change to
how systems code is written: **fibers sized exactly to their need.** Because
the stack bound of a task's entry function is known, the standard library
spawns lightweight stackful tasks with exactly that many bytes of stack, no
guard pages and no growth. Concurrency in Tally therefore needs no
`async`/`await`, no function colouring, no pinned futures and no second
version of every I/O library; blocking calls are ordinary calls that suspend
the fiber, and the `block` effect still tells you which calls those are.

**Why it is worth adopting a new language for.**

1. *It turns the most expensive class of systems bugs into compile errors.*
   "This code must not allocate / block / crash / overflow its stack" is a
   requirement in kernels, interrupt handlers, real-time callbacks, signal
   handlers, async executors and fault-handling paths. Today it is enforced by
   convention, review, and testing that rarely hits the worst case. In Tally
   it is a type error, and the error message names the chain of calls that
   violated it.
2. *It makes dependencies honest.* Because effects appear in public
   signatures, upgrading a library that starts allocating in a hot path, or
   that starts making network calls, is a visible API change that the
   toolchain's compatibility checker flags as breaking. `tally audit` lists
   which dependencies may touch the operating system at all.
3. *It removes the async split.* Exact stack bounds let stackful concurrency
   cost close to what stackless futures cost, so one style of code works for
   both threads and lightweight tasks.
4. *It cannot be retrofitted.* The value of the feature comes from every
   library being annotated. C, C++, Rust, Zig and Go each have large
   ecosystems written without effect information; an effect system added to
   them now (such as Clang's opt-in `nonblocking`/`nonallocating` function
   attributes, added in Clang 20) can only check the code that has opted in,
   and stops at the first unannotated dependency. A new language can make it
   universal from the first package, which is the main reason to start a new
   language rather than extend an old one.

**Where the ideas come from, honestly.** Effect types come from research
languages (FX, Koka, Eff); effect inference with explicit public signatures
is close to how Java's checked exceptions were meant to work, without their
failure mode of forcing every intermediate signature to be rewritten (Tally
infers inside a package). Static stack analysis is done today by external
tools such as GNATstack, AbsInt StackAnalyzer and cargo-call-stack, and by
GCC's `-fstack-usage`. Clang's function-effect attributes check non-blocking
and non-allocating code. What is new is not any one of these but making both
parts *part of the language's meaning*, mandatory across the ecosystem, sound
across indirect calls and recursion, visible to the program as constants, and
used by the standard library to give stackful concurrency exact stacks. No
mainstream systems language does any of the last three.

**Alternatives weighed.**

* *Make effects opt-in* (Clang's approach). Rejected: the feature's value
  comes from coverage; opt-in annotations stop at the first library that
  didn't opt in.
* *Infer effects everywhere, never require them in signatures.* Rejected:
  effects would change silently when a dependency's implementation changed,
  and readers could not see them in source. Requiring them only at the
  package boundary (D29) keeps inference where it is cheap and contracts where
  they matter.
* *Put stack bounds in signatures as numbers.* Rejected: frame sizes depend
  on the target and the optimiser, so a portable signature cannot state them.
  They are computed at link time instead (D33).
* *A full algebraic-effect system with handlers and resumptions* (Koka,
  OCaml 5). Rejected: handlers need either captured continuations or heap
  allocated frames, which is the opposite of predictable cost. Tally's effects
  are markers with no run-time representation.
* *Other candidate headline features:* layout-polymorphic data (one logical
  type, many memory layouts), generational references, regions. Each would be
  useful, but none addresses as directly the costs that D2 puts first.

**What it costs (trade-offs Tally loses).**

* Public signatures are longer: `may alloc | panic` appears on many functions.
  Because integer overflow and out-of-bounds indexing panic (D15), most
  functions that do arithmetic carry `panic`.
* Stack bounds are target-specific and conservative. Indirect calls are
  bounded by the worst function of matching type anywhere in the program, so a
  single deep callback can inflate many bounds. Dynamically loaded code and
  undeclared foreign code make bounds unknown.
* Fibers use their peak stack depth for their whole lifetime, while an
  `async` future in Rust uses only the state live across suspension points.
  For a server with a million mostly idle connections that is a real memory
  cost (typically a few kilobytes per connection rather than a few hundred
  bytes).
* Generic code must name the effects of its type parameters (`may T`), which
  is a concept users of other languages have not met.
* Link-time computation means the stack part of an error is reported at link
  time, after compilation, which is slower feedback than a type error.

---

## D4. The name "Tally"

**Chosen.** The language is called **Tally**. Source files use the extension
`.tly`. The toolchain command is `tally`.

**Alternatives weighed.** *Plumb* (measuring depth; extension clashes with
Perl's `.pl`), *Fathom* (taken by an existing binary-format language),
*Ledger* and *Gauge* (well-known existing tools), *Keel* (no link to the
language's idea).

**Why.** A tally is a running count of what something costs, which is the
language's central idea (D3): the compiler keeps a tally of every function's
effects and stack. The word is short, easy to say in many languages, and a
search found no existing programming language with the name.

---

## D5. Licences

**Chosen.** Software (the eventual compiler, the standard library, the tools,
and any code in this repository) is licensed under **Apache-2.0 WITH
LLVM-exception** (`LICENSE-CODE`). Documentation, including this log and the
standard, is licensed under **CC BY 4.0** (`LICENSE-DOCS`). Code examples
inside the documentation may also be used under the software licence.

**Alternatives weighed.** MIT; dual MIT/Apache-2.0 (Rust's choice); plain
Apache-2.0; GPL-family copyleft; CC0 or CC BY-SA for the documentation.

**Why.** The standard library is compiled into every user's binary, so its
licence must not put obligations on programs written in Tally. Apache-2.0 gives
an explicit patent grant, which matters for a language that others will
implement; its weakness is that it is incompatible with GPLv2 and requires
attribution notices when binary code embeds library portions. The LLVM
exception removes both problems in a single licence, which is simpler than
Rust's dual licence and has the same effect. For the standard, CC BY 4.0 lets
anyone implement, translate, quote and republish it while keeping the source
of the text visible; share-alike was rejected because it would prevent
implementers from including parts of the standard in documentation under
other licences. Allowing the code examples under the software licence lets
people paste them into programs without attribution.

---

## D6. How the standard is written

**Chosen.** The standard is a set of numbered chapters in `standard/`. Rules
are numbered `chapter.section.rule` (for example **4.2.3**). Each rule is
normative, shows a short example, and ends with the decisions it comes from.
The words *must*, *must not*, *may* (in prose, distinct from the keyword
`may`), *error* and *implementation-defined* have fixed meanings defined in
chapter 0. The final chapter lists the questions that are still open, each
with the interim answer an implementation must use.

**Alternatives weighed.** A single long document; a reference manual without
rule numbers; a formal semantics.

**Why.** Numbered rules can be cited by diagnostics, tests and later
decisions. Examples catch ambiguities that prose hides. A full formal
semantics would be more precise but would take far longer and would be read
by far fewer implementers; algorithms that need precision (type inference,
borrow checking, effect inference, stack-bound computation) are given as
step-by-step procedures instead.

---

## D7. What Tally will be built with

**Chosen.** The first compiler, `tallyc`, is written in Rust and generates
code through LLVM. It is driven by the `tally` command. Once the language and
standard library are stable, the compiler is rewritten in Tally and the Rust
implementation is kept only to bootstrap. The stack-bound phase (chapter 11)
uses LLVM's per-function frame-size information (the `.stack_sizes`
section) and the call graph of the optimised code.

**Alternatives weighed.** Writing the bootstrap compiler in C (maximally
portable, but no memory safety and slow to write); in Zig (good
cross-compilation, but a moving target); in OCaml or Haskell (excellent for
compilers, but a smaller pool of systems contributors); a Cranelift backend
(faster compiles, fewer targets and less mature frame-size reporting);
emitting C (portable, but frame sizes and stack bounds become the C
compiler's guess and cannot be checked precisely).

**Why.** The stack part of cost accounting needs exact frame sizes from the
backend, which rules out emitting C. LLVM reports them and supports every
target Tally cares about. Rust is the most productive memory-safe language
with mature LLVM bindings, so the bootstrap compiler can itself be trusted.
Self-hosting later is the best long-term test that the language is good at the
programs it was designed for, and a compiler is such a program.

---

## D8. Source text: UTF-8, ASCII identifiers

**Chosen.** Source files are UTF-8. Identifiers and keywords use only ASCII
letters, digits and underscores. String literals, character literals and
comments may contain any Unicode text.

**Alternatives weighed.** Unicode identifiers following UAX #31 (Rust, Swift,
Go); ASCII-only source (Zig's early position).

**Why.** Unicode identifiers let non-English speakers name things in their
own language, which is a real loss. But they bring homoglyph and
normalisation problems (two different identifiers that render identically,
which matters for code review of security-sensitive systems code), make
tools more complex, and are little used in systems code, which is shared
across countries. Predictability (D2.1) and toolability (D2.5) win. Text in
strings and comments has none of these problems and stays Unicode.

---

## D9. Only line comments

**Chosen.** Comments start with `//` and run to the end of the line. `///`
starts a documentation comment for the following item and `//!` a
documentation comment for the enclosing module. There are no block comments.

**Alternatives weighed.** C-style `/* */` (non-nesting) or Rust-style nesting
block comments.

**Why.** Every line of a comment is visibly a comment, so a reader can never
mistake commented-out code for live code, and the lexer has no nesting state.
Editors comment out regions line by line. This follows Zig.

---

## D10. Explicit semicolons; blocks are expressions

**Chosen.** Statements end with `;`. A block `{ ... }` is an expression whose
value is its final expression if that expression has no trailing `;`, and `()`
otherwise. Expressions that end in a block (`if`, `match`, `loop`, `while`,
`for`, blocks) do not need a `;` when used as statements.

**Alternatives weighed.** Go-style automatic semicolon insertion at line ends;
significant indentation.

**Why.** Automatic insertion was tried in the design and failed on two
common shapes: a block whose value is its last line (the inserted semicolon
erases the distinction between `{ x }` and `{ x; }`), and method chains or
enum-variant patterns that begin a line with `.`, where any rule that joins
lines breaks match arms of the form `.None => 0`. Explicit semicolons have no
exceptions to learn, and the canonical formatter (D51) makes them cost nothing
to write. This is Rust's rule.

---

## D11. Attributes are written `@name(...)`

**Chosen.** Attributes, which attach compiler-visible information to items,
fields, statements and expressions, are written `@name` or `@name(args)`
before the thing they apply to. Documentation comments are a form of
attribute.

**Alternatives weighed.** Rust's `#[name(args)]`; C++'s `[[name]]`; keywords
for each property.

**Why.** `@` is a single character with no other use in Tally, so attributes
are short and unambiguous. A single attribute syntax, rather than a keyword
per property, keeps the keyword list small and gives vendor extensions a
namespace (`@vendor_*`) that cannot collide with future keywords.

---

## D12. Type syntax is prefix; results are written `T ! E`

**Chosen.** Type constructors are written before the type they apply to and
read left to right: `[4]u8` (array of four bytes), `[]u8` (slice of bytes),
`&mut []u8`, `?&T` (optional reference), `*T` (raw pointer). A function that
can fail returns `T ! E`, read "T or error E". A function returning nothing
but able to fail may write `! E` instead of `-> () ! E`.

**Alternatives weighed.** C's declarator syntax (types read inside out);
Rust's mixture (`[u8; 4]`, `Option<&T>`, `Result<T, E>`); postfix
constructors (`u8[4]`).

**Why.** Prefix syntax can be read aloud in order, nests without parentheses
and never needs the spiral rule. Optional and result types appear in nearly
every signature in a language with errors as values, so they get one
character each. `!` was chosen over `|` or `or` because `|` is used for effect
unions (D31) and `!` already suggests "something went wrong". This follows
Zig, except that Zig writes the error set first (`E!T`); Tally puts the
success type first because it is what the reader looks for first.

---

## D13. Generic arguments use square brackets; `.[` in expressions

**Chosen.** Generic parameters and arguments are written in square brackets:
`Vec[u8]`, `fn max[T: Ord](a: T, b: T) -> T`. In expression position,
explicit generic arguments are written after `.`: `mem.size_of.[u32]()`,
`Vec.[u8].new()`. In type position the dot is not used.

**Alternatives weighed.** Angle brackets (`Vec<u8>`), which make `<` and `>`
ambiguous with comparisons and force Rust's `::<>` turbofish and C++'s
`template` disambiguator; square brackets in expressions without a dot (Go),
which make `f[x]` mean either indexing or instantiation and require the parser
or type checker to guess, and which make type arguments like `&T` or `[4]u8`
ambiguous with expressions.

**Why.** Square brackets never conflict with comparisons. Requiring `.[` in
expressions keeps the grammar context-free: `a[i]` is always indexing, and
everything inside `.[ ]` is parsed as types. Explicit arguments are rare in
expressions because local inference (D63) usually finds them, so the extra
dot is seldom written. This is the same idea as Rust's turbofish with a
lighter spelling.
