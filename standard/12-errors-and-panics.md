# 12. Errors and panics

Tally separates two kinds of failure. An *error* is an expected outcome that
the caller should handle (a file is missing, input is malformed, memory ran
out); it is a value of a result type. A *panic* is the detection of a bug (an
index out of range, an overflow, a broken invariant); it stops the program.

## 12.1 Error values

**12.1.1** Any type may be used as the error type `E` of `T ! E`. By
convention, error types are enums or structs that implement `Display` and
`Debug`, and `core.error.Error` (19.3.1). Error values are ordinary values:
they are moved, matched and dropped like any other.

```tally
@derive(Debug, Eq)
pub enum ConfigError {
    from Io(io.Error),
    from Alloc(AllocError),
    Missing { line: u32 },
}
```
*From: D20.*

**12.1.2** A result must be used (5.9.2). Ignoring an error requires writing
`_ = e;`, which makes the decision visible.

```tally
file.flush();              // error: unused value of type () ! io.Error
_ = file.flush();          // explicitly ignored
```
*From: D20, D52.*

## 12.2 The `?` operator

**12.2.1** `e?` requires `e` to have type `T ! E1` or `?T`, and the innermost
enclosing function or closure to have a return type `U ! E2` or `?U`
respectively (a result operand in an optional-returning function, or the
reverse, is an error). If `e` is `.Ok(v)` or `.Some(v)`, `e?` evaluates to
`v`. Otherwise `e?` returns from the function: with `.None`, or with
`.Err(c)`, where `c` is the error converted by 12.2.2. The return runs the
same drops and deferred expressions as a `return` would.

```tally
fn read_config(path: &str) -> Config ! ConfigError may sys | block | alloc | panic {
    let text = fs.read_to_string(path)?;   // io.Error converted to ConfigError.Io
    parse(&text)
}
fn first_char(s: &str) -> ?char { let c = s.chars().next()?; c.to_lowercase() }
```
*From: D20, D19.*

**12.2.2** The error conversion of `?` from `E1` to `E2` is: the identity, if
`E1` and `E2` are the same type; otherwise, if `E2` is an enum that has a
variant declared `from V(X)` with `X` the same type as `E1`, the value
`E2.V(c)`; otherwise the program has an error. An enum may have at most one
`from` variant for each payload type, and a `from` variant must have exactly
one unnamed field. The conversion never runs user code and has no effects.

```tally
enum AppError { from Io(io.Error), from Parse(ParseError), Usage }
fn run() ! AppError may sys | block {
    let f = File.open("a")?;          // io.Error -> AppError.Io
    let n = parse_num("x")?;          // ParseError -> AppError.Parse
    let m = other()?;                 // error if other's error type is not io.Error, ParseError or AppError
}
enum Bad { from A(u8), from B(u8) }   // error: two `from` variants with payload u8
```
*From: D20.*

**12.2.3** `?` is not allowed in a `defer` expression (5.9.5), in a constant
initialiser, or in a function whose return type is neither a result nor an
optional type.

```tally
fn f() { let x = g()?; }      // error: f does not return a result or optional
```
*From: D20, D56.*

## 12.3 Error types in the library

**12.3.1** `core.alloc.AllocError` is the error of every allocating
operation (D22). `std.io.Error` is the error of I/O operations; it carries an
`ErrorKind` and the operating system's error code (20.2.1). Library errors are
`@non_exhaustive` enums so that new causes can be added in minor versions.

```tally
match File.open(p) {
    .Ok(f) => use_file(f),
    .Err(e) if e.kind() == .NotFound => create(p),
    .Err(e) => return .Err(e),
}
```
*From: D20, D22, D44.*

## 12.4 Panics

**12.4.1** When a panic occurs, the program builds a `core.panic.PanicInfo`
holding the panic message and the source location of the operation that
panicked, and calls the program's *panic handler* with a reference to it. The
panic handler is a function of type `fn(&PanicInfo) -> never may sys | block`;
it must not return. While it runs, no destructors run, no deferred expressions
run, and other threads are not stopped; if a second panic occurs on any thread
while a panic handler is running, the program aborts immediately
(`core.process.abort`, 19.16.1) without calling the handler again. The panic
handler of `std` (used unless the program defines one) writes
`panic: <message>` and ` at <file>:<line>:<column>` to the standard error
stream, without allocating and without flushing any buffered output of the
program, and then aborts the process.

```tally
fn main() {
    let v = [1, 2, 3];
    let i = input_index();
    print(v[i]);
}
// with i == 7, standard error shows:
// panic: index out of bounds: the index is 7 but the length is 3
//  at src/main.tly:4:11
```
*From: D21.*

**12.4.2** The operations that panic are: the arithmetic, shift and
conversion failures of 5.3 and 5.4; out-of-range indexing and slicing (5.6.4,
5.6.5); exceeding a recursion limit (11.4.1); stack overflow in an execution
context without an exact bound (11.6.2); calls to `core.panic.panic`, and to
library functions documented to panic (for example `unwrap` on `.None`,
`unreachable`, failed `assert`), all of which declare `may panic`.

```tally
let x: ?u32 = .None;
let y = x.unwrap();          // panics: called unwrap on None
assert(y > 0, "y must be positive");
unreachable();
```
*From: D15, D21.*

**12.4.3** `panic(..args)` builds the message by writing each argument's
`Display` output, in order, into a buffer of `core.panic.MESSAGE_CAPACITY`
bytes (at least 256) held in the panicking context; output beyond the
capacity is discarded at a character boundary and the message ends with `…`.
Building the message does not allocate. If an argument's `Display` panics or
fails, the remaining arguments are skipped.

```tally
panic("invalid state ", state, " for request ", id);
```
*From: D21, D47.*

**12.4.4** A program may define its own panic handler by placing
`@panic_handler` on a function with the type given in 12.4.1. A program has
exactly one panic handler: defining more than one is a link error, and a
program that does not link `std` and does not define one is a link error.

```tally
@panic_handler
fn on_panic(info: &PanicInfo) -> never may sys {
    uart.write_str("PANIC: ");
    uart.write_str(info.message());
    loop { cpu.halt(); }
}
```
*From: D21.*

**12.4.5** Panics are not a substitute for errors. A public library function
must not panic for a condition the caller cannot prevent by checking its
inputs; such conditions are reported as errors. This is a requirement on
libraries that claim conformance with the `core` and `std` specifications of
chapters 19 and 20, and a convention for all other code.

```tally
pub fn parse_port(s: &str) -> u16 ! ParseError { ... }     // invalid input: an error
pub fn get(v: &[]u8, i: usize) -> u8 may panic { v[i] }   // caller can check i < len first
```
*From: D20, D21.*
