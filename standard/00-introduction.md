# 0. Introduction

This is the standard for **Tally 1**, a systems programming language. It
defines the language, its core and standard libraries, its package format and
its toolchain precisely enough that an implementation can be written from it
alone.

## 0.1 Scope

**0.1.1** The standard defines: the source format of Tally programs
(chapters 1–2); the meaning of modules, items, types, expressions, patterns,
traits and generics (chapters 3–7); ownership and borrowing (chapters 8–9);
cost accounting, which is effects and stack bounds (chapters 10–11); errors
and panics (chapter 12); unsafe code, data layout, foreign functions and
assembly (chapters 13–14); concurrency (chapter 15); compile-time evaluation
and attributes (chapter 16); program execution (chapter 17); packages
(chapter 18); the `core` and `std` libraries (chapters 19–20); the `tally`
toolchain (chapter 21); and the questions still open (chapter 22).

```tally
// A complete Tally program.
import std.io;

pub fn main() may sys | block | panic {
    io.println("hello, world");
}
```
*From: D1, D6.*

**0.1.2** The standard does not define: the text of diagnostics, the
algorithm used for optimisation, the internal format of compiled package
interfaces, or the exact frame size of any function. Where these matter the
standard says what must be true of them, not how to achieve it.

```tally
// The standard requires a diagnostic here (rule 10.4.2), but not its wording.
fn quiet() may none { alloc_something(); }
```
*From: D6, D7.*

## 0.2 Terms used for requirements

**0.2.1** *Must* and *must not* state requirements on programs or on
implementations, according to context. *Error* means the implementation must
reject the program and emit at least one diagnostic; an implementation must
not produce an executable from a package that has an error. *Warning* means a
diagnostic that does not prevent compilation; which warnings exist is
implementation-defined unless the standard names them.

```tally
let x: u8 = 300;   // error: the literal 300 is not representable in u8 (4.2.4)
```
*From: D6.*

**0.2.2** *Implementation-defined* means the implementation chooses the
behaviour and must document it. *Unspecified* means the implementation
chooses among the possibilities the standard lists and need not document the
choice or make it consistently. *Undefined behaviour* means the standard
places no requirement on what happens; it can arise only from code inside
`unsafe` blocks or `unsafe` functions (chapter 13). The prose word *may* means
*is permitted to*; it is unrelated to the keyword `may`, which is always
written in code font.

```tally
// Unspecified: the iteration order of a HashMap (19.14.5).
// Implementation-defined: the value of mem.size_of.[usize]() on a target.
// Undefined behaviour: dereferencing a dangling raw pointer in unsafe code.
```
*From: D2, D6.*

**0.2.3** A program is *well-formed* if it has no errors. A *conforming
implementation* accepts every well-formed program within its documented
resource limits, rejects every program with an error, and gives every
well-formed program the behaviour this standard defines. A conforming
implementation may provide extensions only through attributes in the
`@vendor_*` namespace (16.3.2) and through additional toolchain commands; it
must not change the meaning of any well-formed program.

```tally
@vendor_acme_hot fn fast_path() {}   // extension attribute: allowed
```
*From: D2, D6.*

## 0.3 Notation

**0.3.1** Grammar rules use the following notation, which is a variant of
EBNF:

| Notation | Meaning |
|---|---|
| `Name = ... ;` | definition of non-terminal `Name` |
| `"text"` | the terminal token with that exact spelling |
| `UPPER` | a token class defined in chapter 1 (e.g. `IDENT`, `INT_LIT`) |
| `a b` | `a` followed by `b` |
| `a \| b` | `a` or `b` |
| `[ a ]` | zero or one `a` |
| `{ a }` | zero or more `a` |
| `( a )` | grouping |
| `a % ","` | one or more `a` separated by `","`, with an optional trailing `","` |
| `[c1-c2]` | (chapter 1 only) one character in the range c1 to c2 |
| `(* text *)` | a comment in a grammar rule |

```text
ParamList = Param % "," ;        (* accepts:  a: u8   and   a: u8, b: u8, *)
```
*From: D6.*

**0.3.2** Code examples are written in Tally unless marked otherwise. An
example illustrates the rule it follows; if an example and the rule's prose
disagree, the prose is authoritative. Comments in examples marked `error:`
show where a diagnostic is required. Examples may omit `import` declarations
and surrounding functions when they are obvious, and may show a function
declaration ending in `;` instead of a body to show only its signature.

```tally
let a = [1, 2, 3];
let b = a[5];   // error: index 5 is out of bounds for [3]i64 (5.6.4)
```
*From: D6.*

**0.3.3** References of the form *From: D12* name the entries of
`DECISIONS.md` that a rule follows from. References of the form *rule 9.3.2*
or *(9.3.2)* name another rule of this standard.

```text
*From: D24, D25.*   — this rule follows from decisions 24 and 25
(9.3.2)            — see rule 9.3.2
```
*From: D6.*

## 0.4 Overview of the language

This section is informative: it summarises the language and does not add
rules.

Tally is a compiled, statically typed language with no garbage collector and
no required runtime beyond a small start-up routine. Its design follows five
ranked values recorded in D2: predictability, soundness by default, local
reasoning, control, and toolability.

* **Values and ownership.** Every value has one owner. Assignment moves values
  unless their type is `Copy`. Destructors run deterministically at the end of
  their owner's scope (chapter 8).
* **References without lifetime annotations.** Shared (`&T`) and exclusive
  (`&mut T`) references are checked by a borrow checker. Function signatures
  never name lifetimes; a function that returns a reference says which
  parameters it borrows from with a `from` clause (chapter 9).
* **Cost accounting.** Every function has a set of effects (`alloc`, `block`,
  `panic`, `sys`, and library-defined ones) written in its public signature
  with `may`, and a worst-case stack size computed at link time
  (chapters 10–11).
* **Errors are values.** A function that can fail returns `T ! E`; the `?`
  operator propagates errors. Panics, which come from bugs such as an
  out-of-range index, stop the program without unwinding (chapter 12).
* **No hidden code.** There is no operator overloading, no implicit
  conversion that runs user code other than the effect-free `Deref`, no
  exceptions, no macros.
* **Concurrency without colouring.** Threads and fibers run the same code;
  fibers get stacks sized exactly by the link-time bound (chapter 15).
* **Packages** are fetched from their source location by URL, versioned with
  semantic versioning, selected with minimal version selection, and locked by
  content hash (chapter 18).

```tally
import std.fs;
import std.io;

/// Returns the number of lines in the file at `path`.
pub fn count_lines(path: &str) -> u64 ! io.Error may sys | block | panic {
    var file = fs.File.open(path)?;
    var buf: [4096]u8 = [0; 4096];
    var lines: u64 = 0;
    loop {
        let n = file.read(&mut buf)?;
        if n == 0 { break; }
        for b in &buf[0..n] {
            if *b == b'\n' { lines += 1; }
        }
    }
    lines
}
```
*Informative.*
