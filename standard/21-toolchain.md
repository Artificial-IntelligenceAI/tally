# 21. The toolchain

A conforming implementation provides one command, `tally`, with the
subcommands of this chapter. Implementations may add subcommands and options
whose names begin with `x-`.

## 21.1 Invocation

**21.1.1** The command line is `tally COMMAND [OPTIONS] [ARGUMENTS] [-- REST]`.
Options are long-form (`--name`, `--name=value` or `--name value`). The
options accepted by every command that builds are:

| Option | Meaning |
|---|---|
| `--manifest PATH` | the package or workspace manifest (default: the nearest `tally.toml` in the current directory or its ancestors) |
| `--package NAME` | in a workspace, the member to act on (default: all members, or the one containing the current directory) |
| `--target TRIPLE` | the target (21.6.1; default: the host) |
| `--profile NAME` | the build profile (default `debug`; `release` for `tally release` checks) |
| `--features F,...`, `--all-features`, `--no-default-features` | features of the root package |
| `--jobs N` | maximum parallel jobs (default: the number of available CPUs) |
| `--locked` | fail instead of writing `tally.lock` (18.4.4) |
| `--offline` | do not access the network; fail if a needed package is not cached |
| `--quiet`, `--verbose` | less or more output |
| `--message-format human\|json` | diagnostic format (21.4) |

```text
tally build --target aarch64-linux-gnu --profile release --jobs 3
```
*From: D50.*

**21.1.2** Exit statuses: 0 on success; 1 if the requested work failed
(compilation errors, link errors, failed tests, failed checks); 2 for a usage
error (unknown command or option, bad arguments, missing manifest); 3 for an
internal error of the implementation. Diagnostics and progress go to standard
error; the requested output of query commands (`costs`, `audit`, `api`) goes to
standard output.

```text
$ tally build; echo $?
src/lib.tly:4:9: error[10.4.2]: `quiet` may not have effect `alloc`
1
```
*From: D50.*

**21.1.3** The environment variable `TALLY_HOME` names the directory for
the package cache and toolchain data (default: an implementation-defined
per-user directory). `TALLY_CC_<TRIPLE>` (with `-` replaced by `_`) names the
C compiler for that target (default `cc` for the host). No other environment
variable changes the result of a build.

```text
TALLY_HOME=/var/cache/tally tally fetch
```
*From: D43, D45.*

## 21.2 Output layout

**21.2.1** All build outputs go under `out/` in the package (or workspace)
root: `out/TRIPLE/PROFILE/bin/` (executables, named after the binary target),
`out/TRIPLE/PROFILE/lib/` (static and shared libraries),
`out/TRIPLE/PROFILE/obj/` and `out/TRIPLE/PROFILE/cache/` (intermediate,
implementation-defined), `out/doc/` (documentation), and `out/gen/` (18.6.2).
Builds never write outside `out/` except `tally.lock` and the cache in
`TALLY_HOME`.

```text
out/x86_64-linux-gnu/debug/bin/httpd
```
*From: D50, D45.*

**21.2.2** Generated modules are written to `out/gen/PKG/PATH.tly`, where
`PATH` is the module path after the package name with `.` replaced by `/`.

```text
module httpd.proto.messages  ->  out/gen/httpd/proto/messages.tly
```
*From: D45.*

**21.2.3** Given the same sources, manifest, lock file, toolchain version,
target, profile and features, the produced executables and libraries must be
identical byte for byte (reproducible builds), apart from paths of the build
directory if debug information is enabled, which the implementation must
allow to be remapped.

```text
tally build --profile release   (twice, in different directories)  ->  identical binaries
```
*From: D2, D43.*

## 21.3 Commands

**21.3.1** `tally new NAME [--lib|--bin]` creates a directory `NAME` with a
manifest (`version = "0.1.0"`, `tally = "1"`) and `src/lib.tly` (for `--lib`)
or `src/bin/NAME.tly` with an empty `main` (for `--bin`, the default).
`tally init [--lib|--bin]` does the same in the current directory, using its
name. Neither overwrites existing files.

```text
tally new blinky --bin
```
*From: D50.*

**21.3.2** `tally build [--lib] [--bin NAME]... [--emit KIND]` builds the
selected targets (default: all targets of the selected packages), fetching
dependencies as needed. `--emit` selects, for the library target,
`static-lib` or `shared-lib` (a C-callable library of the `@export`ed
functions, 17.2.1), and, for any target, additionally `asm` or `object`.
Building a binary includes linking and therefore the stack-bound phase
(chapter 11): all `@stack_limit` assertions and `bound_required` uses are
checked.

```text
tally build --lib --emit static-lib
```
*From: D33, D50.*

**21.3.3** `tally run [--bin NAME] [-- ARGS...]` builds and runs the binary
(required if the package has more than one) with `ARGS`. Its exit status is
the program's, unless building failed.

```text
tally run --bin httpd -- --port 8080
```
*From: D50.*

**21.3.4** `tally test [FILTER] [--exact] [--list] [--no-run]` builds the
selected packages with `@cfg(test)` true and dev-dependencies available, and
runs every selected test. A test's *full name* is its module path, a space,
and its string literal (`httpd.router "routes static files"`); two tests in
one module with the same string are an error. A test is selected if its full
name contains `FILTER` as a substring (or equals it, with `--exact`); with no
`FILTER`, all are. `--list` prints the full names of the selected tests, one
per line, in module-path then source order, and runs nothing. Each test is run in its own process, so that a panic in one does
not affect the others, with at most `--jobs` running at once. A test body
behaves as a function body whose return type is `()` or `() ! E` for some
`E: Display` (inferred); a test *passes* if it returns `()` or `.Ok(())`, and
*fails* if it returns `.Err(e)` (the `Display` of `e` is printed) or panics,
except that a test marked `@expect_panic` passes only if it panics (and, with
`@expect_panic("text")`, only if the panic message contains `text`). For each
test one line is printed: `test NAME ... ok`, `... FAILED`, or `... panicked`,
followed by its captured output if it did not pass; then a summary line
`N passed; M failed`, where `M` counts tests reported `FAILED` or `panicked`.
Tests are reported in the order of `--list`. The exit status is 1 if any test
failed.

```tally
test "parse rejects empty" {
    assert(parse("").is_err());
}
@expect_panic("index out of bounds")
test "index past end panics" { let v = [1]; _ = v[core.hint.black_box(1)]; }
```
*From: D49, D21.*

**21.3.5** `tally check` performs every check of compilation (parsing, name
resolution, types, borrowing, effects, constant evaluation) without
generating code or linking. It reports the same errors as `tally build` except
those of the stack-bound phase.

```text
tally check --message-format json
```
*From: D50.*

**21.3.6** `tally fmt [--check] [FILES...]` rewrites the selected source files
(default: all `.tly` files of the selected packages) into canonical format
(21.5). With `--check` it writes nothing and exits with 1 if any file is not
in canonical format.

```text
tally fmt --check
```
*From: D51.*

**21.3.7** `tally doc [--open]` writes HTML documentation of the public items
of the selected packages and their dependencies to `out/doc/`, showing for
every function its full signature, including the `may` clause, and for every
type its size and alignment on the selected target when they are fixed.

```text
tally doc --target thumbv7em-none-eabihf
```
*From: D3, D50.*

**21.3.8** `tally costs [--bin NAME] [--path] [--json] [PATTERN...]` builds
the selected binary (or, for a library, the package's `test` binary) and
prints, for every function instance whose qualified name contains one of the
patterns (default: every public function of the selected packages and every
`@stack_limit` function), one line:

```text
QUALIFIED_NAME <TAB> EFFECTS <TAB> BOUND
```

where `EFFECTS` is the instance's effect set in normal form (10.2.5), written
`none` or as effect names joined by ` | ` in the order `alloc`, `block`,
`panic`, `sys`, then user effects in lexical order of their paths; and `BOUND`
is the stack bound in bytes or `unbounded`. With `--path`, each line is
followed by the deepest call path, one function per line indented by two
spaces, as `NAME (FRAME bytes)`, or the reason for `unbounded` (the recursive
cycle without a limit, or the undeclared foreign function). `--json` prints
one JSON object per function with keys `name`, `effects` (array of strings),
`bound` (number or `null`) and `path` (array of objects with keys `name` and
`frame`, or a string giving the reason for `unbounded`). A `QUALIFIED_NAME`
is the module path and item name (`httpd.net.on_packet`); for methods, the
type's path then the method (`httpd.Conn.close`, and `httpd.Conn.Drop.drop`
for trait methods); generic instances append their type and const arguments
in `.[ ]` (`core.alloc.Vec.push.[u8, Global]`); closures append
`.{closure#N}`, numbered from 1 in source order within their function.

```text
$ tally costs --path on_packet
httpd.net.on_packet	none	2312
  httpd.net.on_packet (96 bytes)
  httpd.route.route (160 bytes)
  httpd.route.lookup_v6 (2056 bytes)
```
*From: D3, D33.*

**21.3.9** `tally audit [--json]` prints, for each package in the dependency
graph, one line `NAME VERSION SOURCE` followed by the set of effects that
appear in the declared `may` clauses of its public functions and in its
foreign declarations and `asm` expressions, and the number of `unsafe`
blocks it contains. `SOURCE` is the lock-file source string (18.4.4), or the
directory for path dependencies. Packages whose effects include `sys` are
listed first; within each group, packages are sorted by name, then version.

```text
$ tally audit
tls 2.1.0 git+https://example.org/libs#dir=tls  effects: alloc | block | panic | sys  unsafe: 41
json 0.9.1 git+https://example.org/json  effects: alloc | panic  unsafe: 0
```
*From: D3, D44.*

**21.3.10** `tally api [--out FILE]` writes a description of the public
interface of the selected package (21.3.11) as JSON. `tally api diff OLD NEW`
compares two such descriptions (or two git revisions, written `git:REV`) and
prints each difference with its classification, `breaking` or `compatible`,
and finally the smallest version increment (`major`, `minor` or `patch`) that
semantic versioning requires. For a package below 1.0.0 the increments are
shifted, following the compatibility lines of 18.4.1: on `0.MINOR` (minor at
least 1), a breaking change requires a new minor version and anything else a
new patch version; on `0.0.PATCH`, every release is a new compatibility line,
so any change is allowed in any increment.

```text
$ tally api diff git:v1.4.2 git:HEAD
breaking   fn httpd.router.route: effects grew: none -> alloc
compatible fn httpd.router.route_static: added
required increment: major
```
*From: D44.*

**21.3.11** The public interface consists of every public item reachable
from the package's root module through public modules and re-exports, with:
for functions, the full signature including generic bounds, `from` and `may`
clauses; for structs and enums, their generic parameters, public fields and
all variants, whether they are `view`, `@non_exhaustive`, `Copy`, `Send` and
`Sync`, and, if every field is public, their size, alignment and field
offsets on each of the targets the package lists in the `[package]` key
`api-targets` (an array of target triples; default: the host); for traits, all members; and every impl of a public trait for a
public type. A change is *breaking* if it can make a program that compiled
against the old version fail to compile or change meaning against the new
one. In particular the following are breaking: removing or renaming a public
item; changing a type in a signature; adding an effect to a function's
declared set or to an impl's associated effect binding; adding a `from`
source; changing a trait method's `may` clause in either direction; adding a
trait member without a default; adding a variant to an enum or a field to a
struct that is not `@non_exhaustive`; making a type no longer `Copy`, `Send`
or `Sync`; changing the layout of a struct whose fields are all public;
adding a `Drop` impl to a public type; removing an impl. Adding items, impls,
defaulted trait members, and removing effects from a function's declared set,
are compatible (minor). Changes that do not alter the interface are patch
changes.

```tally
// v1.0.0
pub fn lookup(k: u32) -> ?u32 { ... }
// v1.1.0 — breaking, so this version number is rejected by `tally release`:
pub fn lookup(k: u32) -> ?u32 may alloc { ... }
```
*From: D44, D29.*

**21.3.12** `tally release VERSION` checks that the working tree is clean,
that `VERSION` is greater than the latest version tag, that the manifest's
`version` equals `VERSION`, that the package builds and its tests pass, and
that `tally api diff` between the latest version tag and the working tree
requires no larger increment than `VERSION` makes (this check is skipped when
there is no earlier version tag); then it creates the local
git tag `vVERSION` (or `DIR/vVERSION`). It never pushes.

```text
tally release 1.5.0
```
*From: D44, D43.*

**21.3.13** The remaining commands are: `tally fetch` (download every
dependency into the cache and verify hashes, 18.4.4); `tally update [KEY...]`
(raise the minimum version of the named dependencies, or all, in the manifest
to the latest version available on the same compatibility line, then
recompute the lock file); `tally clean` (remove `out/`); `tally version`
(print the implementation's name and version and the standard version it
implements, `tally 1`).

```text
tally update json
```
*From: D43, D50.*

## 21.4 Diagnostics

**21.4.1** A human-readable diagnostic begins with a line
`FILE:LINE:COLUMN: SEVERITY[RULE]: MESSAGE`, where `SEVERITY` is `error` or
`warning`, and `RULE` is the number of the rule of this standard that the
error enforces (for errors) or the warning name (16.3.5). Any further lines
are implementation-defined. With `--message-format json`, each diagnostic is
one JSON object on one line with keys `file`, `line`, `column`, `severity`,
`rule` and `message`.

```text
src/audio.tly:12:5: error[10.4.2]: `callback` may not have effect `alloc`
    12 |     buf.push(sample)?;
       |     ^^^^^^^^^^^^^^^^ `Vec.push` has effect `alloc` (Vec[f32, Global])
```
*From: D2, D6.*

## 21.5 Canonical format

**21.5.1** `tally fmt` produces the canonical format, which is defined by
these rules; the output for a given input must be unique, and formatting must
be idempotent. Formatting changes only whitespace, line breaks, the order of
imports, and trailing commas.

1. Indentation is four spaces per block level; tabs are never produced.
2. Lines are at most 100 columns where possible.
3. Imports come first, sorted: `core` imports, then `std`, then dependency
   imports, then imports of the current package; within each group, sorted by
   path; one blank line between groups.
4. One blank line between items; never more than one consecutive blank line
   anywhere; no trailing whitespace; the file ends with one line end.
5. `{` ends the line of the construct that opens it; `}` is on its own line
   (or is followed by `else` on the same line); a block containing a single
   expression and no statements may stay on one line if it fits.
6. Binary operators, `=`, `=>`, `->`, `|` in effect clauses and `!` in result
   types are surrounded by one space; no space inside `()`, `[]`, or after
   prefix operators, or before `,`, `;`, `:`, `?`, or a `.` that follows an
   expression; one space after `,` and after a `:` that ends a name. A `.`
   beginning an inferred variant (`= .None`) and the `:` of a label reference
   (`break :outer`) are preceded by one space and followed by none.
7. When a list (parameters, arguments, generic arguments, struct fields,
   array elements, match arms' tuple patterns) does not fit on its line, it is
   written one element per line, each followed by a comma, indented one
   level, with the closing bracket on its own line; outer lists are broken
   before inner ones.
8. A function signature that does not fit puts its parameters one per line;
   if `-> T`, `from`, `may` and `where` clauses then still do not fit, each
   starts its own line, indented one level.
9. Match arms are one per line; an arm whose expression is not a block ends
   with a comma.
10. Doc comments and comments are kept on their own lines, reindented to the
    level of the following code.

```tally
pub fn copy_range(
    source_buffer: &[]u8,
    destination_buffer: &mut []u8,
    start: usize,
    end: usize,
) -> usize ! CopyError may panic {
    ...
}
```
*From: D51.*

## 21.6 Targets

**21.6.1** A target is named by a triple `ARCH-OS-ENV` (for example
`x86_64-linux-gnu`, `aarch64-macos-none`, `x86_64-windows-msvc`,
`riscv64-linux-musl`, `thumbv7em-none-eabihf`, `wasm32-wasi-none`). The
implementation documents which targets it supports; for each, the values of
the `@cfg` predicates (16.2.2) and the `core.target` constants. The `ARCH`
part of a triple may name a sub-architecture (`thumbv7em`, `armv7`, `i686`);
the implementation documents the `arch(...)` value each maps to (for
example `thumbv7em` and `armv7` to `arm`, `i686` to `x86`).

```text
tally build --target riscv64-linux-musl
```
*From: D42, D50.*

**21.6.2** The predicate values that implementations must use for these
platforms are: `os`: `linux`, `macos`, `ios`, `windows`, `freebsd`,
`netbsd`, `openbsd`, `android`, `wasi`, `none`; `family`: `unix` (all of the
above except `windows`, `wasi` and `none`), `windows`; `arch`: `x86`,
`x86_64`, `arm` (all 32-bit Arm, including Thumb), `aarch64`, `riscv32`,
`riscv64`, `wasm32`, `wasm64`, `powerpc64`, `s390x`, `loongarch64`; `env`:
`gnu`, `musl`, `msvc`, `eabi`, `eabihf`, `none`.

```tally
@cfg(any(arch("riscv32"), arch("riscv64"))) import this.arch.riscv;
```
*From: D42.*

**21.6.3** For freestanding targets, the implementation documents the valid
interrupt names for `@interrupt` (17.4.1), the linker script it uses by
default, the start-up code it provides (17.3.2), and the value of
`CALL_OVERHEAD` and `CONTEXT_OVERHEAD` (19.16.1).

```text
thumbv7em-none-eabihf: interrupts "Reset", "NMI", "HardFault", "MemManage", "BusFault",
                       "UsageFault", "SVCall", "PendSV", "SysTick", "IRQ0" ... "IRQ239"
```
*From: D61, D33.*

**21.6.4** C sources (18.2.6) are compiled with the C compiler named by
`TALLY_CC_<TRIPLE>` or, for the host, `cc`, invoked with the target's flags,
the package's `include` directories and `cflags`, and an optimisation level
derived from the profile. The produced objects are linked into every binary
that depends on the package. For stack bounds, functions defined in C
sources are foreign functions: they are bounded only if declared with
`@stack_bound` (14.1.5).

```text
cc -c -O2 -Ic/include c/zlib/inflate.c -o out/.../obj/inflate.o
```
*From: D45, D33.*
