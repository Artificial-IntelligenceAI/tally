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

---

## D14. Integer types, including arbitrary widths; literals default to `i64`

**Chosen.** Tally has signed and unsigned integers of every width from 1 to
128 bits (`u1` … `u128`, `i1` … `i128`), plus `usize` and `isize`, which have
the width of a pointer. There is no `int` type whose size varies by target.
Integer literals have no type of their own: they take the type their context
requires, and a literal whose type is not determined by anything defaults to
`i64`.

**Alternatives weighed.** Only the power-of-two widths (C, Rust, Go); a
platform-sized default `int` (C, Go); a 32-bit default (Rust's `i32`); type
suffixes on literals (`5u8`, Rust).

**Why.** Odd widths describe hardware registers, protocol fields and packed
formats exactly (`u3` for a 3-bit field), which is control (D2.4); Zig has
shown they are practical, and LLVM supports them directly. A target-sized
default would give the same source different overflow behaviour on different
targets, which violates predictability (D2.1). `i64` is chosen over `i32`
because overflow is checked (D15) and a 64-bit default makes accidental
overflow on counts and sizes rare on every target; on 32-bit targets the cost
is acceptable because hot code names its types. Suffixes are unnecessary
because context almost always fixes the type, and `as` handles the rest.

---

## D15. Arithmetic is checked in every build; wrapping and saturating are separate operators

**Chosen.** `+`, `-`, `*`, `/`, `%`, unary `-`, and `<<`/`>>` with an
out-of-range shift amount panic when the mathematical result is not
representable (or on division by zero). The panic happens in every build
profile. Wrapping arithmetic is written `+%`, `-%`, `*%`; saturating
arithmetic is written `+|`, `-|`, `*|`. Checked arithmetic returning an
optional is a library method (`a.checked_add(b)`).

**Alternatives weighed.** Rust's model (panic in debug builds, wrap in release
builds); C's model (signed overflow is undefined behaviour, unsigned wraps);
always wrapping (Go, Java); Zig's model (undefined in release-fast).

**Why.** A program whose meaning changes with the optimisation level is not
predictable (D2.1): Rust's release builds silently wrap where debug builds
panic, so tests can pass on behaviour that production does not have. Undefined
overflow is unsound (D2.2). Always wrapping hides bugs. Checking always has a
small run-time cost that compilers remove when they can prove the range; where
it matters, the programmer writes `+%` and the intent is visible in the source.
The cost is that most arithmetic carries the `panic` effect (D3), which is
honest: such code *can* crash.

---

## D16. Conversions: `as` is checked, `as%` truncates, widening is implicit

**Chosen.** `x as T` converts between numeric types and panics if the value is
not representable in `T` (for floats converted to integers: if it is NaN or
out of range after truncation toward zero). `x as% T` converts between
integer types by keeping the low bits (two's complement truncation or
extension). A conversion that can never lose information (a narrower integer
to a wider integer of compatible signedness, `f32` to `f64`) happens
implicitly at *coercion sites* (assignment, argument passing, return, struct
fields, array elements), but never between the operands of a binary operator.

**Alternatives weighed.** Rust's `as`, which truncates and saturates silently;
Zig's family of builtins (`@intCast`, `@truncate`, `@floatFromInt` …);
C's implicit conversions including narrowing; no implicit widening at all
(Rust).

**Why.** A conversion that silently changes a value is a classic source of
bugs in systems code (lengths truncated to 16 bits, negative values becoming
huge sizes). Making the common spelling checked follows D15, and making the
bit-level truncation a distinct, visible operator keeps it available for
hashing and protocol code. Implicit widening removes noise that carries no
information, and limiting it to coercion sites keeps binary operators simple:
`a + b` requires both operands to have the same type, so a reader never has to
work out which operand was widened.

---

## D17. No operator overloading; `==` is structural

**Chosen.** Operators have fixed meanings on built-in types and cannot be
defined for user types. `==` and `!=` are defined for every type whose values
are made of comparable parts (integers, floats, `bool`, `char`, raw pointers
by address, references by the value they refer to, arrays, slices, `str`,
tuples, structs and enums whose fields are all comparable and that declare
`@derive(Eq)`), and compare structurally. Ordering operators work only on
numbers, `char` and `bool`. Other comparisons and arithmetic on user types are
ordinary method calls (`a.cmp(b)`, `v.add(w)`).

**Alternatives weighed.** Full operator overloading through traits (Rust,
C++); overloading only for arithmetic traits; no `==` on aggregates (Zig).

**Why.** An operator that may run arbitrary user code (allocate, lock, panic)
breaks the rule that costs are visible at the call (D2.1, D2.3). With effects
this could be tracked, but a reader skimming `a + b` would still not expect
a function call. The loss is real for numeric code (vectors, matrices, big
integers), which becomes more verbose; Tally accepts that because numeric
library code is a minority of systems code. Structural equality covers the
most common need (comparing keys, tags and records) without user code and
with a cost that is obvious from the type.

---

## D18. Files are modules; one namespace; two visibility levels

**Chosen.** Every source file is a module, and its path in the package's
source directory is its module path; there are no module declarations.
Each module has a single namespace for types, functions, constants, modules
and all other items. An item is visible either throughout its package
(the default) or to every package (`pub`). Other modules' items are reached
only through `import`.

**Alternatives weighed.** Rust's explicit module tree with `mod`
declarations, separate type and value namespaces, and five visibility levels;
Go's directory-as-package with capitalisation-based export; Zig's
files-as-structs.

**Why.** Deriving modules from files removes a class of "where is this
defined" questions (D2.3) and keeps tools simple (D2.5). A single namespace
means a name in a scope denotes exactly one thing. Two visibility levels match
the unit that matters for cost accounting: effects are inferred inside a
package and declared at its boundary (D29), so "package" and "public" are the
boundaries that need distinguishing. Finer levels are rarely worth their
complexity; a large package that needs internal walls should be split.
Explicit `pub` was chosen over capitalisation so that renaming for style
never changes visibility.

---

## D19. Optional and result types are ordinary enums with short syntax

**Chosen.** `?T` is the type `Option[T]` with variants `None` and `Some(T)`.
`T ! E` is `Result[T, E]` with variants `Ok(T)` and `Err(E)`. They are
declared in `core` as normal enums and get pattern matching, methods and
layout like any enum. Three conversions happen implicitly at coercion sites:
`T` to `?T` (as `.Some`), `T` to `T ! E` (as `.Ok`), and `never` to anything.
The absent value is written `.None`; there is no null.

**Alternatives weighed.** Separate built-in optional and error-union types with
their own rules (Zig); no implicit wrapping (Rust); nullable pointers (C, Go).

**Why.** Using ordinary enums means no special cases in pattern matching,
generics or layout, and readers already know how enums behave (D2.3).
Wrapping a successful or present value is the overwhelmingly common case and
carries no information, so it is implicit; failure and absence are always
written out, so the unusual path is the visible one. Implicit wrapping never
runs user code and has a fixed cost, so it does not weaken D2.1.

---

## D20. Errors are values; `?` converts only through declared `from` variants

**Chosen.** A function that can fail returns `T ! E`. The postfix `?`
operator returns early with the error if there is one. If the operand's error
type differs from the function's, `?` converts it only when the function's
error type is an enum with exactly one variant marked `from` whose payload is
the operand's error type; any other mismatch is a compile error. There are no
exceptions and no other implicit error conversion.

**Alternatives weighed.** Exceptions (C++, Java), which make every call a
possible exit; Rust's `?` with arbitrary `From::from` conversions, which can run
any code; Zig's global error sets, which merge automatically but cannot carry
data; Go's multiple return values, which can be ignored silently.

**Why.** Errors that are ordinary values are visible in signatures and at
every call site that can fail (`?` marks it), which serves D2.1 and D2.3.
Restricting `?`'s conversion to a declared enum variant gives the convenience
of automatic wrapping (the common case is "this error is one of the causes of
my error") while guaranteeing that `?` never runs user code, never allocates,
and can be understood by reading the error enum. Payload-carrying errors are
kept because systems code needs to report which file, which line, which errno.

---

## D21. Panics stop the program; there is no unwinding

**Chosen.** A panic (from a bug such as an out-of-range index, an overflow,
or an explicit `panic` call) calls the program's panic handler, which must not
return; by default it prints a message and aborts the process. No
destructors or deferred expressions run. There is no way to catch a panic.

**Alternatives weighed.** Unwinding with optional catching (C++ exceptions,
Rust's default `panic = unwind` with `catch_unwind`); per-thread panics that
leave the rest of the process running (Rust).

**Why.** Unwinding adds hidden control-flow edges to every call, landing pads
to every function, and a requirement that every data structure be left in a
consistent state at every possible panic point, which is rarely achieved in
practice ("exception safety"). Those edges make code size and behaviour less
predictable (D2.1). Systems programs that must survive a bug survive it at a
process or partition boundary (a supervisor restarts the process), which is
more robust than catching. The loss is real: a server cannot contain one
request's bug inside one thread, and test harnesses must run tests in separate
processes (D49).

---

## D22. Allocation failure is an error value

**Chosen.** Every operation that allocates memory returns an error
(`AllocError`) when the allocator cannot satisfy it, rather than aborting.
`Vec.push`, `Box.new`, `String.push_str` and similar return `T ! AllocError`.

**Alternatives weighed.** Abort on allocation failure (Rust's default, C++'s
default `new`); a separate fallible API alongside an infallible default (the
direction Rust took for the Linux kernel).

**Why.** Kernels, embedded systems and databases must handle running out of
memory; an infallible default makes the wrong behaviour the easy one, and two
parallel APIs double the library surface. With `?` (D20), handling is one
character per call. Application programs pay a small syntactic cost for
behaviour that is correct everywhere. Zig showed this is workable.

---

## D23. Ownership and moves; no partial moves

**Chosen.** Every value has exactly one owner (a binding, a field, an element,
or a temporary). Assignment, argument passing and returning move the value,
leaving the source unusable, unless the type is `Copy`. Copying of other
values is explicit through `Clone`. A value is destroyed (dropped) exactly
once, when its owner goes out of scope without having been moved from. A
single field cannot be moved out of a struct; a struct is moved as a whole
(or destructured as a whole by a pattern, or a field is taken with
`mem.replace`).

**Alternatives weighed.** Garbage collection; reference counting by default
(Swift); manual allocation and freeing (C, Zig); Rust's model including
partial moves; C++ copy-by-default with move semantics opt-in.

**Why.** Ownership gives deterministic destruction with no run-time cost and
is the foundation of memory safety without a collector, which is soundness
(D2.2) without giving up predictability (D2.1). Move-by-default with explicit
`clone` makes every expensive copy visible. Partial moves (Rust) let a struct
be half-moved, which forces readers and the compiler to track per-field
states and conditional drop flags per field; banning them removes that with
little loss, because destructuring and `mem.replace` cover the uses.

---

## D24. References are checked without lifetime annotations

**Chosen.** Tally has shared (`&T`) and exclusive (`&mut T`) references,
checked by a borrow checker in the style of Rust's non-lexical lifetimes:
while an exclusive reference to a place is live, nothing else may access the
place; while a shared reference is live, nothing may modify it. Unlike Rust,
**lifetimes are never written**. Instead:

* inside function bodies, regions are inferred;
* a function whose return type contains references says which parameters it
  may borrow from with a `from` clause (`-> &str from a, b`); `from *p`
  means "from what `p` refers to" rather than from `p` itself; without a
  clause, the result may borrow from every reference parameter;
* a struct or enum holding references is declared `view`, and has exactly one
  region: all references inside it share one lifetime.

**Alternatives weighed.** Rust's explicit lifetime parameters; Hylo/Val's
second-class references (references can never be stored, only passed down),
which need no annotations but forbid storing a slice in a struct, so parsers,
iterators and string views become awkward; Vale's generational references,
which check at run time; garbage collection.

**Why.** Rust demonstrated that compile-time borrow checking gives memory
safety and data-race freedom with no run-time cost, which D2.2 requires.
Its main usability cost is lifetime syntax, and most of that syntax encodes
two facts: which inputs a returned reference comes from, and that a struct
holds borrowed data. `from` clauses and `view` types state exactly those two
facts in words. The single-region rule loses structs that hold references with
two independent lifetimes, and functions whose lifetimes relate in ways a
`from` clause cannot express (Rust's `'a: 'b` bounds, higher-ranked
lifetimes); experience with Rust suggests these are rare in application code
but do occur in advanced library code, which must then use owned data,
indices, or unsafe code. That loss is accepted for a much simpler model.

---

## D25. Drop order and temporaries

**Chosen.** Bindings are dropped at the end of their block in reverse order of
declaration; struct fields are dropped in declaration order; array and slice
elements from first to last. A temporary created while evaluating an
expression is dropped at the end of the enclosing *full expression*
(statement, `let` initialiser, condition, block tail), except that the
temporaries of a `match`, `if let` or `while let` scrutinee live until the end
of that construct. Temporaries are never extended to the lifetime of a
binding: `let r = &make();` is an error.

**Alternatives weighed.** Rust's rules, including temporary lifetime extension
for `let x = &temp;` and edition-dependent tail-expression rules; C++'s rules
(temporaries die at the end of the full expression, with extension when bound
to a const reference).

**Why.** Destruction order must be easy to predict (D2.1) because destructors
release locks, close files and free memory. One rule ("end of the
statement") plus one exception that every reader needs (a `match` needs its
scrutinee for its whole body) is easier to hold in the head than Rust's set
of extension rules. The cost is an occasional extra `let`.

---

## D26. Interior mutability and thread-safety markers

**Chosen.** Shared references normally forbid mutation. Types built on the
primitive `UnsafeCell[T]` (such as `Cell`, `Atomic`, `Mutex`) may be mutated
through shared references under their own rules. Two marker traits, `Send`
(may be moved to another thread) and `Sync` (may be shared between threads by
reference), are implemented automatically by the compiler for types whose
parts implement them, and can be asserted with `unsafe impl` for types that
manage their own synchronisation.

**Alternatives weighed.** No interior mutability (pure functional
approaches); C-style `volatile`/unchecked sharing; Rust's design (which this
is); per-object locks (Java).

**Why.** Without some form of interior mutability, shared caches, counters,
locks and lazy initialisation cannot be written safely. Rust's markers are the
only known design that makes data races a compile error without a run-time
cost, which D2.2 requires. Auto-implementation keeps them out of most code.

---

## D27. Traits: monomorphised generics, strict coherence, no specialisation

**Chosen.** Polymorphism is through traits, as in Rust: traits declare
methods, associated types, associated constants and associated *effects*
(D30); generic functions and types are compiled separately for each set of
type arguments (monomorphisation); `dyn Trait` gives dynamic dispatch through
a vtable when chosen explicitly. An implementation of a trait for a type is
allowed only in the package that declares the trait or the type's outermost
constructor (the orphan rule), and two implementations of the same trait whose
headers could apply to the same type are an error, regardless of their bounds.
There is no specialisation, no inheritance, and no higher-kinded types.

**Alternatives weighed.** Go-style structural interfaces (no explicit impls);
C++ templates checked only at instantiation; Zig's `comptime` duck typing;
Rust's coherence rules (more permissive: `#[fundamental]` types, negative
reasoning about bounds, upcoming specialisation); type classes with
dictionary passing (Haskell), which hides allocation and indirection.

**Why.** Trait-checked generics give errors at the definition rather than at
each use (local reasoning, D2.3) and monomorphisation gives predictable,
dispatch-free code (D2.1); dynamic dispatch is available but always visible as
`dyn`. Strict coherence (header-overlap ignoring bounds) is less expressive
than Rust's, but it means a reader can find every implementation of a trait
for a type in at most two packages and adding an implementation in a
dependency can never break a dependent's existing impls in surprising ways.
Specialisation was rejected because it makes which code runs depend on
distant impls, against D2.3.

---

## D28. Four built-in effects, plus effects declared by libraries

**Chosen.** The built-in effects are `alloc` (uses the global heap), `block`
(may wait for something outside the current thread of execution), `panic`
(may stop the program) and `sys` (talks to anything outside the program's own
memory: system calls, memory-mapped hardware, foreign code, assembly).
Packages may declare further effects with `effect name;`; such an effect has
no built-in meaning and enters a function's cost only through declarations,
from which it then propagates like any other.

**Alternatives weighed.** A larger fixed set (separate effects for I/O,
network, file system, time, randomness, locking, spinning, unbounded loops);
a single "impure" bit; only user-defined effects.

**Why.** Each built-in effect corresponds to a requirement that real systems
code states and that is currently enforced by hand: no allocation (real-time
and interrupt code), no waiting (interrupt handlers, executors, signal
handlers, latency-critical paths), no crashing (fault handlers, destructors,
safety-critical paths), no outside interaction (pure computation, plugin
sandboxes, auditing dependencies). Finer splits of `sys` (file vs network)
were rejected because the boundary between them is the operating system's
choice, not the program's, and libraries that want such distinctions can
declare their own effects. Termination and time bounds were rejected because
they cannot be checked without proofs.

---

## D29. Effects are written in public signatures and inferred elsewhere

**Chosen.** A public function, every trait method declaration, every
function pointer type and every `dyn` type has exactly the effects its `may`
clause lists; no clause means no effects. Package-visible functions and
closures without a clause have their effects inferred from their bodies;
with a clause, the clause is checked. Foreign function declarations must
have a clause.

**Alternatives weighed.** Inference everywhere with tooling to show results
(rejected in D3 because API effects could change silently); mandatory clauses
everywhere (too verbose inside a package, where the author controls every
caller); "no clause means any effect" for public items (would make the
feature opt-in again).

**Why.** The package is the unit at which code is versioned, reviewed and
published, so it is where costs must be contracts (D2.1). Inside a package,
the author sees and changes all callers together, so inference costs nothing
in predictability and removes most annotations. Making "no clause" mean
"none" for public items means the shortest signature is the strongest
promise, and adding a cost always requires writing it down.

---

## D30. Effect polymorphism through type terms, associated effects and erased effect parameters

**Chosen.** Generic code refers to the effects of its type parameters: in a
`may` clause, a type parameter `T` stands for all effects associated with
`T`'s bounds plus the effects of dropping a `T`, and `T.Name` stands for one
associated effect. Traits declare *associated effects* (`effect Next;`) that
each impl binds (`effect Next = sys | block;`), the effect counterpart of
associated types. Functions, structs and traits may also take explicit
`effect E` parameters. Effects have no run-time representation, so effect
parameters never cause additional monomorphised copies, and methods generic
only over effects remain callable through `dyn`.

**Alternatives weighed.** Row-polymorphic effect types with full inference
(Koka), which are more expressive but produce signatures most systems
programmers would find hard to read; checking effects only after
monomorphisation (C++-template style), which reports errors at use sites
rather than definitions; no polymorphism (every generic function is `may
any`), which would make the feature useless for libraries.

**Why.** Systems libraries are full of generic containers and callbacks, and
their costs genuinely depend on their arguments: `Vec[T, A].push` allocates
only if `A` does. Type terms express "whatever my parameters cost" in one
word, which keeps common signatures short (`may T | panic`). Associated
effects reuse a concept Rust programmers already know from associated types.
Erasure keeps the feature free at run time (D2.1) and avoids code-size
growth. The coarse meaning of `T` (all of its bounds' effects, plus drop)
loses precision in rare cases, where `T.Name` is available.

---

## D31. Effect syntax: `may e1 | e2`

**Chosen.** Effects are written after the keyword `may`, joined with `|`:
`may alloc | panic`. `may none` is the empty set and `may any` the set of all
effects. The same form is used in function declarations, function types,
trait bounds, `dyn` types and `unsafe may` blocks. `may` is a contextual
keyword.

**Alternatives weighed.** Comma-separated lists (`may alloc, panic`), which are
ambiguous inside generic argument lists and bounds; `+`, which is ambiguous
with trait bounds; braces (`{alloc, panic}`), which collide with blocks;
`effects(...)`, `uses ...`, `throws`-like keywords.

**Why.** `|` cannot appear in a type, so an effect clause ends unambiguously
wherever it is embedded (`Box[dyn Write may sys | block]`, `T: Iterator may
none + Clone`), which keeps the grammar context-free (D2.5). Reading "may
alloc or panic" is accurate: the function may do any subset. `may` reads as
English and states permission, not obligation.

---

## D32. Allocators are type parameters; `alloc` means the global heap

**Chosen.** The `alloc` effect means "uses the program's global allocator".
Allocating library types take their allocator as a type parameter with a
default (`Vec[T, A: Allocator = Global]`), store it, and use it for every
operation; the effects of an allocating operation are those of the
allocator's impl (`Global`'s are `alloc`, an arena's may be `none`). The
global allocator is chosen once per program.

**Alternatives weighed.** An implicit allocator in a dynamically scoped
context, installed with something like `with alloc = arena { ... }` (Odin's
context, effect handlers); Zig's explicit allocator arguments on every call;
a single global allocator only (Rust stable, C).

**Why.** The context approach was the first design tried, because it makes
every library allocator-agnostic for free. It fails on soundness of the
effect guarantee: a collection created under one allocator and grown or
dropped under another would allocate from the wrong one, so code that the
checker proved allocation-free could still call `malloc`. Storing the
allocator in the value's type keeps the effect exact (the effect checker knows
which allocator every operation uses), keeps memory safety (a collection
borrowing an arena is a view type that cannot outlive it), and costs nothing
at run time for zero-sized allocators like `Global`. The loss is that
libraries must be generic over `A` to be allocator-agnostic, which is the
same situation as Rust's allocator API.

---

## D33. Stack bounds are computed at link time over the whole program

**Chosen.** After code generation, the toolchain computes for every function
in the linked program its worst-case stack usage from the actual frame sizes
and the call graph of the generated code. Indirect calls are bounded by every
function of matching type whose address the program takes; `dyn` calls by
every implementation the program converts to that trait object; foreign
functions by their declared `@stack_bound`. A cycle in the call graph is
bounded only if it passes through a function with `@recursion_limit(N)`,
which the program enforces at run time by panicking on the (N+1)th nested
activation. Programs can assert bounds (`@stack_limit`) and read them
(`stack.bound(f)`), as link-time constants.

**Alternatives weighed.** Bounds in source signatures (not portable across
targets or optimisation levels); compile-time estimates before optimisation
(unsound after inlining and register allocation); external analysis tools
only (not part of the language, so the program cannot rely on the result,
and the standard library cannot use it to size stacks); forbidding recursion
(too restrictive for parsers and tree code).

**Why.** Only the generated code knows the real frame sizes, and only the
linker sees the whole program, which is required to bound indirect calls
soundly. Making the result a link-time constant lets the program use it
(exact fiber stacks, D34) and lets the build fail when an interrupt handler
grows past its stack. Runtime-enforced recursion limits turn the one
unboundable construct into a bounded one at the cost of a counter and a
possible panic. The cost of this decision is that the stack half of a cost
error appears only at link time, that bounds are conservative where function
pointers are used widely, and that dynamically loaded code is unbounded
unless it carries bound metadata.

---

## D34. Concurrency: threads and exactly-sized fibers, no `async`

**Chosen.** Tally has OS threads and *fibers* (stackful tasks scheduled by a
library executor on one OS thread). Blocking operations in `std` are
fiber-aware: when called inside a fiber they suspend the fiber instead of the
thread, so the same function works unchanged in both. A fiber's stack is
allocated with exactly the link-time bound of its entry function (D33). There
is no `async`/`await`, no future type in the language and no `Pin`. Fibers
are scheduled within *scopes*: a fiber may borrow from the code that created
its scope, and the scope waits for all its fibers before it returns
(structured concurrency). Fibers do not migrate between threads.

**Alternatives weighed.** Rust-style `async`/`await` with compiler-generated
state machines; Go-style goroutines with growable stacks and a work-stealing
scheduler; threads only; callback-based event loops; Zig's evolving model of
passing an `Io` interface (from which the idea of the same code running
blocking or evented comes).

**Why.** `async`/`await` splits every library in two (function colouring),
adds `Pin` and self-referential types to the language, makes trait methods
returning futures difficult, and makes stack traces and debugging worse. It
exists mainly because stackful tasks need big stacks, which waste memory.
Exact stack bounds remove that reason: a fiber gets the stack it needs and no
more, without guard pages or growth. Go's growable stacks need the ability to
move stacks, which needs a garbage collector's knowledge of pointers; Tally
has raw pointers, so stacks cannot move. Not migrating fibers between threads
avoids having to prove that everything held across a suspension point is
`Send`, at the cost of no automatic load balancing across cores (the
thread-per-core model of Seastar and Glommio). The memory cost relative to
`async` is recorded in D3.

---

## D35. The memory model is C++20's

**Chosen.** Atomic operations and their orderings (`Relaxed`, `Acquire`,
`Release`, `AcqRel`, `SeqCst`) have the semantics of the C++20 memory model.
A data race, which is two unsynchronised accesses to the same memory location
from different threads where at least one is a write and not both are
atomic, is undefined behaviour; safe code cannot create one.

**Alternatives weighed.** A new memory model; Java's model (races are not
undefined but give weak guarantees); sequential consistency only.

**Why.** The C++20 model is what the hardware vendors, LLVM and every other
systems language implement, which makes interoperating with C and C++ code
exact. Inventing a different one would be a research project with a high
chance of being wrong. Sequential consistency alone would forbid the
lock-free algorithms systems code needs (D2.4).

---

## D36. Unsafe code: explicit, local, and with a defined list of undefined behaviour

**Chosen.** Operations the compiler cannot check (dereferencing raw
pointers, calling foreign or `unsafe` functions, accessing mutable statics,
reading union fields, inline assembly, implementing unsafe traits, asserting
effects) are allowed only inside `unsafe` blocks. The body of an `unsafe fn`
is not itself an unsafe block. The standard lists every kind of undefined
behaviour; anything not listed is defined. The aliasing rule for references
is stated simply in terms of a reference's period of use.

**Alternatives weighed.** No unsafe subset (a checked-only language cannot
write allocators, drivers or FFI); Rust's model, including unsafe function
bodies being unsafe blocks (changed in Rust 2024) and an aliasing model still
under research (Stacked Borrows, Tree Borrows); C's model, where undefined
behaviour is spread through the whole language.

**Why.** Systems code needs unchecked operations (D2.4), and soundness (D2.2)
requires them to be marked and few. Requiring an `unsafe` block even inside an
`unsafe fn` keeps each unchecked operation visible. A complete list of
undefined behaviour is part of predictability: an implementer knows what
optimisations are allowed, and a programmer knows what to avoid. The aliasing
rule chosen is simpler and stricter than the research models; programs that
follow it are valid under them too, and the open question of refining it is
recorded in chapter 22.

---

## D37. Data layout is fully specified

**Chosen.** Struct fields are laid out in declaration order with natural
alignment, exactly as a C compiler for the target would lay out the same
fields. Enums are a tag followed by a payload, with a tag type chosen by a
fixed rule. The only layout optimisation is that `?T` for pointer-like `T`
(references, non-null raw pointers, `Box`, function pointers, `NonZero`
integers) uses the zero value for `.None`. `@packed`, `@align(N)` and
`@bitpacked(uN)` give further control.

**Alternatives weighed.** Rust's default of unspecified layout with field
reordering and aggressive niche filling, plus opt-in `#[repr(C)]`; C's
partially implementation-defined bitfields.

**Why.** Systems programs read and write memory in fixed formats (hardware
registers, file formats, network packets, shared memory, FFI), and
predictability (D2.1) means a reader can compute a type's size and every
field's offset by hand. Reordering saves some padding, which the programmer
can recover by ordering fields themselves. Niche filling beyond the pointer
case saves space in rare cases at the cost of making layouts hard to predict.
`@bitpacked` replaces C bitfields with a definition that does not depend on
the compiler.

---

## D38. Foreign functions: the C ABI, declared effects, declared stack bounds

**Chosen.** Tally calls and is called by C through `extern "C"` declarations
and `@export`. Every foreign function declaration must state its effects,
may state its stack bound, and is unsafe to call. Only types with a C
equivalent may cross the boundary. Panics never cross it: a panic inside an
exported function aborts.

**Alternatives weighed.** Importing C headers directly (Zig's `@cImport`);
generating bindings with an external tool (Rust's bindgen); C++ interop.

**Why.** Control (D2.4) requires zero-cost C interop. Effects and stack bounds
must be declared for foreign code because the compiler cannot see it, and
without declarations a single C call would make every caller `may any` and
unbounded. Header import is valuable but large, and it would make the
language depend on a C parser; it is left to a tool (chapter 22). C++ interop
is out of scope because it requires adopting C++'s object model.

---

## D39. Inline assembly in the style of Rust's `asm!`

**Chosen.** Inline assembly is an expression `asm may E stack N (template,
operands...)` whose operand syntax follows Rust's `asm!`: operands bound to
register classes or explicit registers, with direction (`in`, `out`,
`inout`, `lateout`), clobbers, and options. It must declare its effects
(default `sys`) and the stack it uses (default 0).

**Alternatives weighed.** GCC-style constraint strings; no inline assembly
(only separate assembly files); Zig's GCC-like syntax.

**Why.** Kernels and drivers need a few instructions inline (barriers,
special registers, system calls). Rust's operand syntax is the most readable
existing design and maps onto LLVM directly. The effect and stack clauses are
needed for cost accounting, because the compiler cannot analyse the assembly
text.

---

## D40. Compile-time evaluation is decided by effects; no macros

**Chosen.** Constants, static initialisers, array lengths and const generic
arguments are evaluated at compile time by running ordinary Tally code in an
interpreter. A function may be called at compile time exactly when its effect
set is within `alloc | panic`; no `const fn` marker exists. A panic during
evaluation is a compile error. Static initialisers may build heap data, which
becomes read-only static data. Tally has no macro system.

**Alternatives weighed.** Rust's `const fn` (a separate, growing subset of the
language marked function by function) and `macro_rules!`/procedural macros;
Zig's `comptime` (types as values, duck-typed generics); C++'s `constexpr`
plus templates; C's preprocessor.

**Why.** The effect system already says which functions interact with the
outside world, which is exactly the property that makes compile-time
evaluation impossible or non-reproducible, so a second marker would duplicate
it. Macros give great power but hurt local reasoning (D2.3) and tooling
(D2.5): code that the reader sees is not the code that runs, and every tool
must expand macros to understand a program. The needs that macros serve in
Rust are covered in Tally by built-in derives (D41), rest parameters (D47),
generators (D45) and generics. The loss is real for embedded DSLs and for
user-defined derives, which need a generator step instead.

---

## D41. Derives are built in; other code generation is a build step

**Chosen.** `@derive(...)` supports exactly the core traits `Copy`, `Clone`,
`Eq`, `Ord`, `Hash`, `Default` and `Debug`, with fully specified output. Any
other generated code (serialisation, bindings, parsers from grammars) is
produced by *generators*: ordinary programs, declared in the manifest, that
the build runs in a sandbox to write Tally source files that are then
compiled like hand-written ones.

**Alternatives weighed.** User-defined derive macros (Rust's procedural
macros); compile-time reflection (Zig's `@typeInfo`, C++26 reflection);
external code generators run by hand (Go's `go generate`).

**Why.** The core derives cover the traits almost every type wants, and
specifying their output exactly keeps them predictable. Procedural macros run
arbitrary code inside the compiler with full access to the machine, which is a
supply-chain risk and makes builds slow and non-reproducible. Generators keep
the power but make the generated code visible as files, run without network or
file system access beyond their inputs, and are cached by input hash.
Reflection was the strongest alternative; it was rejected for this version
because it ties library behaviour to the shape of types in ways that are hard
to version (chapter 22 records it as open).

---

## D42. Conditional compilation with `@cfg` predicates

**Chosen.** Items, fields, variants, match arms, members and statements can
carry `@cfg(predicate)`; elements whose predicate is false are removed before
name resolution. Predicates test the target (`os`, `arch`, `family`,
`endian`, `pointer_width`, `env`), features, the profile, `test` and
`debug_assertions`, combined with `all`, `any` and `not`. Target facts are
also available as constants in `core.target` for code that should be
type-checked on every target.

**Alternatives weighed.** A preprocessor (C); compile-time `if` over constants
only (Zig), which requires both branches to type-check, making
platform-specific APIs awkward; Rust's `#[cfg]`, which this follows.

**Why.** Systems code must call APIs that do not exist on every target, so
some removal before type checking is required; attaching it to syntax
elements (not text) keeps tools able to parse every file. Offering constants
too lets code that can be portable stay checked everywhere.

---

## D43. Packages: decentralised sources, minimal version selection, hash-locked

**Chosen.** A package is a directory with a `tally.toml` manifest (TOML) and a
`src/` directory. Dependencies are named by source location (a git URL or a
local path) and a minimum version. A package's identity is its source plus its
compatibility line (major version, or `0.minor` before 1.0), so different major
versions of a package can coexist. Versions are chosen by minimal version
selection: for each identity, the highest of the minimum versions that any
package in the graph requires. `tally.lock` records the selected versions and
a SHA-256 content hash of each, and builds verify the hashes.

**Alternatives weighed.** A central registry with SAT-style "newest
compatible" resolution (Cargo, npm); Go's module system (decentralised, MVS,
checksum database), which this mostly follows; vendoring only; header-only
distribution (C).

**Why.** Minimal version selection gives reproducible builds without a
lockfile, never picks a version nobody tested against, and needs no solver;
newest-compatible resolution makes the build change when someone else
publishes. Content hashes make a fetched dependency verifiable regardless of
where it was fetched from. A central registry was not required for the
language to function, so it is left open (chapter 22) rather than designed
in; the cost is that discovery and name squatting are not addressed yet.

---

## D44. API compatibility is checked by the toolchain, effects included

**Chosen.** `tally api` extracts a package's public interface (items,
signatures, effects, trait impls, layouts of public types with public fields)
and compares two versions, classifying each change as compatible or breaking
under semantic versioning. Adding an effect to a public function, removing a
public item, or changing a public type's layout are breaking. `tally release`
refuses to tag a version whose number does not match the changes.

**Alternatives weighed.** Relying on authors to follow semver by hand (most
ecosystems); external tools (`cargo-semver-checks`).

**Why.** Minimal version selection (D43) and the promise that effects are
contracts (D29) both depend on versions meaning what they say. Making the
check part of the toolchain means the effect half of the major feature is
enforced across package versions, not only within a build.

---

## D45. Builds are declarative; there are no build scripts

**Chosen.** The manifest describes everything the build does: Tally sources
are found by layout, C sources and libraries to link are listed, and
generators (D41) are declared with their inputs and outputs. No package can
run arbitrary code during a dependent's build except declared generators,
which run sandboxed (no network, read access only to declared inputs, write
access only to their output directory).

**Alternatives weighed.** Build scripts in the language (Rust's `build.rs`,
Zig's `build.zig`); external build systems (Make, CMake, Bazel).

**Why.** Build scripts are the largest supply-chain attack surface in modern
package ecosystems and make builds unreproducible and hard for tools to
understand. Declarative builds cover the common needs (compiling bundled C,
linking system libraries, generating code). The loss is flexibility for
unusual builds, which must use an external build system around `tally`.

---

## D46. Two library layers: `core` (including collections) and `std`

**Chosen.** The library has two layers. `core` works on every target,
including bare metal, and contains everything that does not need an operating
system: primitive methods, iterators, formatting, cells, atomics, hashing,
allocators, and the allocating collections (`Box`, `Vec`, `String`, `HashMap`,
`Rc`, `Arc`), which use the program's global allocator or an allocator type
argument. `std` adds the operating-system interfaces and re-exports `core`.
`std` paths are UTF-8 strings.

**Alternatives weighed.** Rust's three layers (`core`, `alloc`, `std`); one
library with everything conditionally compiled; a minimal library with
collections in third-party packages (C).

**Why.** Because allocators are type parameters (D32) and the global
allocator is just a declared static, collections have no reason to live
outside `core`: a freestanding program that declares a global allocator gets
`Vec` and `HashMap`, and one that uses only `Bump` never needs a global
allocator at all. Two layers are simpler to explain than three. UTF-8 paths
make the common case simple and portable; non-UTF-8 Unix paths are served by a
platform module, which is the honest place for a platform-specific concern.

---

## D47. Formatting without format strings; rest parameters

**Chosen.** Printing and formatting take any number of arguments, each
written with its `Display` implementation in order:
`io.println("x = ", x, ", y = ", y)`. Formatting options are wrapper values
(`fmt.hex(x)`, `fmt.fixed(f, 3)`). The language feature that makes this
possible is the *rest parameter* (`..args: A`), which packs trailing
arguments into a tuple whose type is a generic parameter; tuples implement
`Display` by concatenation. `Display.fmt` is generic over an effect parameter
for the output sink, so formatting into a fixed buffer is effect-free while
formatting to a file has the file's effects.

**Alternatives weighed.** Format strings checked by macros (Rust's
`format!`), which Tally cannot have without macros (D40); format strings
checked at run time (C's `printf`, unsafe or panicking); C++20 `std::format`
with compile-time checking through templates; operator chaining
(`std::cout <<`).

**Why.** Concatenating typed arguments cannot have a mismatch between a
format string and its arguments, needs no parsing at run time, and works with
the effect system. Rest parameters are a small, general feature (also used by
`assert` and `panic`) rather than a special case for printing. The cost is
that long format strings with many placeholders are less compact, and
localisation (which reorders arguments) needs a library with its own format
strings.

---

## D48. Hashing: SipHash-1-3 with a per-process random key

**Chosen.** `HashMap` and `HashSet` default to SipHash-1-3, keyed with a key
that `std` reads from the operating system's random source at start-up. The
iteration order of hash collections is unspecified. The `Hash` encodings of
built-in types are fully specified.

**Alternatives weighed.** A faster non-cryptographic hash (FxHash, wyhash) as
the default, with predictable iteration order; a fixed key.

**Why.** Systems programs are often network-facing, and a predictable hash
lets attackers make every insertion collide (HashDoS). Predictability (D2.1)
argues for deterministic order, but soundness of the program's performance
under attack wins here, and code that needs deterministic order can choose a
different hasher or sort. Specifying the byte encodings means two
implementations hashing with the same key produce the same values.

---

## D49. Tests live in the source and run in separate processes

**Chosen.** Tests are `test "name" { ... }` items written in any module,
compiled only by `tally test`, and run each in its own process. A test passes
if its body returns normally (or returns `.Ok`), fails if it returns an error
or panics, and `@expect_panic` inverts the panic case.

**Alternatives weighed.** Tests as attributed functions (Rust's `#[test]`);
separate test directories only (Go's `_test.go` files are in-package but
separate files); running all tests in one process with panic catching.

**Why.** In-source tests sit next to the code they test and can use
package-visible items. Because panics never unwind (D21), a panicking test
cannot be caught inside the process, so separate processes are the only
correct isolation; they also prevent tests from interfering through global
state. The cost is process start-up time per test.

---

## D50. One command, `tally`, with cost-reporting subcommands

**Chosen.** The whole toolchain is one command with subcommands for
building, running, testing, checking, formatting, documenting, fetching,
releasing and querying. Two subcommands exist specifically for cost
accounting: `tally costs` reports each function's effects and stack bound with
its deepest path, and `tally audit` reports the effects and unsafe code of
every dependency. Diagnostics cite the rule of the standard they enforce.

**Alternatives weighed.** A separate compiler and build tool (C, C++, Rust's
`rustc` and `cargo`); a build system written in the language (Zig).

**Why.** One entry point is easier to learn and to standardise. The cost
commands make the major feature (D3) usable day to day: a developer can ask
why a function is unbounded or which dependency introduced `sys`. Citing rule
numbers makes every error traceable to a rule and, through it, to the
decision behind it.

---

## D51. One canonical source format, defined in the standard

**Chosen.** `tally fmt` produces a single canonical format whose rules are part
of the standard, with no configuration options.

**Alternatives weighed.** A configurable formatter (clang-format, rustfmt
options); no formatter.

**Why.** A single format removes style discussion, keeps diffs small, and
makes explicit semicolons (D10) and other syntax costless to write. Putting
the rules in the standard lets independent implementations agree.

---

## D52. Assignment is a statement; values must be used

**Chosen.** Assignment and compound assignment are statements, not
expressions. An expression statement must have type `()` or `never`; any
other value must be used or discarded with `_ = expr;`.

**Alternatives weighed.** Assignment as an expression (C); warnings for unused
results (Rust's `#[must_use]`, opt-in per type); silently discarding values.

**Why.** Assignment inside expressions causes the classic `if (x = y)` bug and
complicates evaluation order. Requiring every value to be used makes ignored
errors impossible to miss (D20) without a per-type attribute, and `_ =` makes
the deliberate cases visible.

---

## D53. Labels are `name:` and are referred to as `:name`

**Chosen.** Loops and blocks may be labelled `name:`; `break :name value` and
`continue :name` refer to them. Unlabelled `break` targets the innermost loop
only.

**Alternatives weighed.** Rust's `'name:` (uses the apostrophe, which Tally
otherwise uses only for characters); `goto`; no labels.

**Why.** Labelled breaks are needed for nested loops and for early exit from
blocks that compute a value. Zig's `:name` form is unambiguous with the
optional break operand. Unlabelled `break` never targets a labelled block so
that adding a label never changes what an existing `break` does.

---

## D54. Closures capture by reference unless `move`

**Chosen.** Closures are written `|params| body`, capture each used binding
by the weakest mode their body needs (shared reference, exclusive reference,
or by value when moved), or everything by value with `move`, and implement
`Fn`, `FnMut` and `FnOnce` as their bodies allow. A closure's effects are its
body's.

**Alternatives weighed.** Explicit capture lists (C++); capture by value only
(simple but forces copies or reference-counted sharing); Rust's model (which
this is).

**Why.** Inferred capture modes are safe because the borrow checker checks
them, and they need no annotation in the common case. `move` covers threads
and fibers that outlive their creator's frame. Explicit capture lists would be
more predictable to read but rarely tell the reader anything the checker does
not already enforce.

---

## D55. Local bindings may shadow

**Chosen.** A `let` may introduce a name that shadows an earlier binding or an
outer name, including in the same block.

**Alternatives weighed.** Forbidding shadowing entirely (Zig); forbidding it
within a block only.

**Why.** Shadowing lets a value change type or form without inventing names
(`let s = s.trim();`), which is common when parsing. The risk, a reader
missing that a name was rebound, is limited because shadowing never affects
code before it, and the canonical formatter keeps `let` statements at the
start of lines.

---

## D56. `defer`, but not `errdefer`

**Chosen.** `defer e;` runs `e` when the enclosing block exits by any route
except a panic. There is no form that runs only on error.

**Alternatives weighed.** Zig's `defer` and `errdefer`; destructors only (Rust,
C++ RAII); `try`/`finally`.

**Why.** Ownership with destructors (D23) handles most cleanup, but raw
foreign resources (a file descriptor from C, a lock taken in assembly) would
otherwise each need a wrapper type. `defer` covers those in one line.
`errdefer` was rejected because it needs the notion of "exiting with an
error", which in Tally is just returning a value, and guard objects with an
explicit `commit()` express the same thing without a new rule.

---

## D57. Strings are UTF-8; `char` is a Unicode scalar value

**Chosen.** `str` is always valid UTF-8 and is indexed by byte, with slicing
that panics at a non-boundary. `char` holds one Unicode scalar value. Byte
literals and byte strings exist for binary data, and C string literals
(`c"..."`) for foreign calls.

**Alternatives weighed.** Byte strings with no encoding guarantee (C, Zig, Go
in practice); UTF-16 (Java, Windows); grapheme-based indexing (Swift).

**Why.** UTF-8 is the encoding of the network, of files and of Unix, so most
data never needs conversion; a validity guarantee lets text functions be
correct without re-validating. Byte indexing keeps costs visible (no hidden
O(n) indexing). Grapheme clusters are a library concern because they depend on
Unicode version tables.

---

## D58. Method calls auto-reference and auto-dereference; `Deref` is effect-free

**Chosen.** Method calls find methods by trying the receiver's type and its
dereferences (through references, `Box` and `Deref` impls), each by value,
then by shared reference, then by exclusive reference. Field access
auto-dereferences the same way. `Deref` and `DerefMut` impls must have no
effects. Deref coercion applies at coercion sites (`&String` to `&str`).

**Alternatives weighed.** No automatic referencing (C's `->` vs `.`, Zig);
Rust's model with arbitrary `Deref` code.

**Why.** Without auto-referencing, method calls on references and smart
pointers become noisy (`(&*b).len()`). The hidden call this introduces is the
one exception to "no hidden user code" (D17), and requiring `Deref` to be
effect-free means the hidden call cannot allocate, block, crash or touch the
outside world, which keeps D2.1.

---

## D59. Sized by default, with `?Sized` opt-out; no user-defined unsized types

**Chosen.** Slices, `str` and trait objects are the only unsized types. Type
parameters are sized unless marked `?Sized`. Programs cannot declare their own
unsized types (no struct with an unsized last field).

**Alternatives weighed.** Rust's model, including custom dynamically sized
types; no unsized types (every slice is a struct of pointer and length).

**Why.** Unsized types let `&[]T`, `&str` and `&dyn Tr` be ordinary
references, which keeps the borrowing model uniform. Custom unsized types are
rarely written and complicate layout and generic code, so they are left out.

---

## D60. Patterns: bare identifiers always bind; matches are exhaustive

**Chosen.** In patterns, a bare identifier always introduces a binding;
enum variants and constants are written with a dot (`.None`, `Color.Red`).
Matches must be exhaustive and an unreachable arm is an error. Matching a
reference auto-dereferences and binds by reference (default binding modes).

**Alternatives weighed.** Rust's rule, where a bare identifier is a binding
unless a constant or unit variant of that name is in scope (a well-known
source of bugs when an import is missing); unreachable arms as warnings.

**Why.** The meaning of a pattern should not depend on what happens to be in
scope (D2.3). Leading-dot variants keep patterns short. Unreachable arms are
almost always mistakes, and with exhaustiveness required, an error costs
nothing in correct code.

---

## D61. Program entry points

**Chosen.** Hosted binaries start at `main`, which may return `()`, an exit
code `u8`, or a result whose error is printed. No user code runs before
`main`. Freestanding programs mark one function `@entry` and must provide a
panic handler (and a global allocator if they allocate). Interrupt handlers
are functions marked `@interrupt("NAME")`, and may not have `alloc` or
`block`.

**Alternatives weighed.** C's `main(argc, argv)` with static constructors
before it (C++); Rust's `#[no_main]` and linker-driven entry points.

**Why.** No code before `main` keeps start-up predictable (D2.1): statics are
compile-time values, so initialisation order problems cannot occur.
Arguments through a library function rather than parameters keep `main`'s
signature small. Constraining interrupt handlers' effects in the language is a
direct use of the major feature.

---

## D62. Packages declare the language version they use

**Chosen.** Each manifest states `tally = "1"`, the major version of the
standard the package is written against. A future major version may change
the language; compilers must keep compiling older versions and link packages
of different versions together.

**Alternatives weighed.** No versioning (breaking changes break everyone);
Rust's editions (which this follows); separate languages.

**Why.** Mistakes in a first standard will need fixing, and the only way to
fix syntax and semantics without breaking the ecosystem is to let each
package opt in. Linking across versions keeps the ecosystem whole.

---

## D63. Type inference is local to function bodies

**Chosen.** Types are inferred only inside function bodies and constant
initialisers, with bidirectional checking and unification; signatures,
fields, constants and statics are always written in full. Expressions that
need a type to be resolved (method calls, field access, `?`) require it to be
known at that point.

**Alternatives weighed.** Whole-program Hindley–Milner inference (ML,
Haskell); inference of return types (C++ `auto`, Go closures).

**Why.** Written signatures are the contracts that local reasoning (D2.3) and
the package boundary (D29) rely on, and they keep type errors local.
Requiring types to be known in order, as Rust does, keeps inference simple to
specify and its errors understandable.

---

## D64. Evaluation order is strictly left to right

**Chosen.** Operands, arguments and fields are evaluated left to right, each
once; in an assignment the right side is evaluated before the place.

**Alternatives weighed.** Unspecified order (C, C++ before C++17), which
permits optimisations and breeds bugs; right side last in assignments (Java).

**Why.** Predictability (D2.1): a reader must be able to tell in which order
effects happen. Evaluating the assigned value before the place follows Rust:
the place is computed from the state after the value's side effects, so the
store goes where the place points at the moment of storing, and a compound
assignment `p op= v` reads and writes the same, freshly computed place.

---

## D65. Floating-point functions are correctly rounded

**Chosen.** Every floating-point function in `core` (square root, exponential,
logarithm, trigonometric and hyperbolic functions, power, hypotenuse) returns
the correctly rounded result for every argument on every target.

**Alternatives weighed.** Platform `libm` results (C, Rust, Zig), which differ
between platforms and versions; a documented error bound (such as 1 ulp);
exposing only IEEE-required operations (`sqrt`).

**Why.** Predictability (D2.1) applied to numbers: the same program computing
the same function must produce the same bits everywhere, which matters for
reproducible simulations, lock-step networking, deterministic replay and
testing. Correctly rounded implementations of all these functions for
binary32 and binary64 now exist (the CORE-MATH project) with performance close
to conventional libraries. The cost is somewhat slower code than the fastest
approximations, and implementers must use or port such implementations rather
than calling the platform's `libm`.
