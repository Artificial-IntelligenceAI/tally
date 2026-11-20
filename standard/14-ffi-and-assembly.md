# 14. Foreign functions and inline assembly

## 14.1 Calling foreign code

**14.1.1** An `extern "ABI" { ... }` block declares functions and statics
defined outside the program's Tally code, to be resolved by the linker. The
ABI strings are `"C"`, the target's C calling convention, and `"system"`, the
convention of the target's operating-system API (on 32-bit x86 Windows,
`stdcall`; elsewhere the same as `"C"`). Other ABI strings are
implementation-defined.

```tally
extern "C" {
    fn strlen(s: *u8) -> usize;              // error: missing may clause (14.1.3)
    fn getpid() -> i32 may sys;
}
```
*From: D38.*

**14.1.2** A foreign function declaration's parameter and return types must be
FFI-safe (14.2.1). It may end its parameters with `...` to declare a C
variadic function; extra arguments must be FFI-safe and undergo the C default
argument promotions (`bool`, `u8`, `i8`, `u16` and `i16` are passed as
`c_int`, and `f32` as `f64`). The function's symbol name is its
Tally name, or the string given by `@link_name("sym")`.

```tally
extern "C" {
    fn printf(fmt: *c_char, ...) -> c_int may sys | block;
    @link_name("__errno_location") fn errno_ptr() -> *mut i32 may none;
}
unsafe { printf(c"%d items\n".as_ptr(), 3 as c_int); }
```
*From: D38.*

**14.1.3** Every foreign function declaration must have a `may` clause, which
is trusted: the compiler cannot check it, and a false clause voids the
guarantee of 10.7.1 (but is not undefined behaviour by itself). Calling a
foreign function is an unsafe operation (13.1.2).

```tally
extern "C" { fn qsort(b: *mut u8, n: usize, w: usize, f: extern "C" fn(*u8, *u8) -> i32) may sys | panic; }
```
*From: D38, D29.*

**14.1.4** A foreign static `static NAME: T;` or `static var NAME: T;` in an
extern block refers to a symbol defined elsewhere. Every access to it is an
unsafe operation. Its type must be FFI-safe.

```tally
extern "C" { static var optind: i32; }
let i = unsafe { optind };
```
*From: D38.*

**14.1.5** `@stack_bound(N)` on a foreign function declaration states that a
call to it uses at most N bytes of stack, including everything it calls. The
value is trusted; if it is false, stack bounds that depend on it are wrong and
an exactly-sized execution context calling it has undefined behaviour. Without
the attribute, the function is `unbounded` (11.2.2).

```tally
extern "C" { @stack_bound(256) fn memchr(s: *u8, c: i32, n: usize) -> ?*u8 may none; }
```
*From: D38, D33.*

## 14.2 FFI-safe types

**14.2.1** The *FFI-safe* types are: `u8`, `u16`, `u32`, `u64`, `i8`,
`i16`, `i32`, `i64`, `usize`, `isize`, `f32`, `f64`, `bool` (as C `_Bool`),
`char` (as `uint32_t`); `u128` and `i128` where the target C ABI defines
`__int128`; raw pointers and nullable raw pointers to sized types or to
`u8`/`c_void`; references to sized FFI-safe types (as non-null pointers);
`extern "C"` function pointers and nullable ones whose parameter and return
types are FFI-safe; structs, tuple structs and unions whose fields are all
FFI-safe and which are not `@bitpacked`; enums whose variants are all unit
variants and that declare a tag type of a fixed width; arrays of FFI-safe
types inside structs (not as parameters); and, as a return type only, `()`
and `never`. `core.ffi` defines aliases for C types (`c_int`, `c_long`,
`c_char`, `c_void`, …) for each target. Other types are not FFI-safe.

```tally
extern "C" {
    fn stat(path: *c_char, buf: *mut Stat) -> c_int may sys;    // Stat: a struct of FFI-safe fields
    fn bad(s: &str) may none;                                   // error: &str is not FFI-safe
}
```
*From: D38, D37.*

**14.2.2** A C string literal `c"..."` (a string literal prefixed with `c`)
has type `CStr` (19.17.2), a copyable view of static bytes: the bytes of the
literal followed by a zero byte. It must not contain `\0` or `\x00`.

```tally
let s: CStr = c"hello";
unsafe { puts(s.as_ptr()); }
```
*From: D38, D57.*

## 14.3 Exporting functions

**14.3.1** `@unsafe(export)` on a public function with an `extern "C"` (or
other foreign) ABI gives it external linkage under its own name, or the name
given by `@unsafe(export("sym"))`, so that foreign code or a linker script can
refer to it. The attribute is unsafe (13.1.2) because an exported symbol can
replace one that other code relies on (for example `malloc` or `memcpy`);
the programmer promises that it does not.
Its parameter and return types must be FFI-safe. An exported function's
effect set and stack bound are computed as usual; its address counts as taken
(11.2.1). If a panic occurs inside an exported function, the panic handler
runs as usual (12.4.1); it never returns, so no unwinding into foreign code can
occur.

```tally
@unsafe(export("tally_crc32"))
pub extern "C" fn crc32(p: *u8, n: usize) -> u32 { unsafe { crc(slice.from_raw(p, n)) } }
```
*From: D38, D21.*

**14.3.2** The symbol names of Tally functions that are not exported are
implementation-defined (*mangled*) and must encode the package, module path,
name and generic arguments so that distinct instances never collide in one
program.

```text
httpd.router.route[u32]   ->  e.g. _T5httpd6router5routeIu32E  (implementation-defined)
```
*From: D38, D7.*

## 14.4 Inline assembly

**14.4.1** Inline assembly is an expression:

```text
AsmExpr     = "asm" [ EffectClause ] [ "stack" INT_LIT ]
              "(" STRING_LIT { STRING_LIT } { "," AsmOperand } [ "," ] ")" ;
AsmOperand  = [ IDENT "=" ] AsmDir "(" AsmReg ")" AsmValue
            | "clobber_abi" "(" STRING_LIT ")"
            | "options" "(" [ IDENT % "," ] ")" ;
AsmDir      = "in" | "out" | "lateout" | "inout" | "inlateout" ;
AsmReg      = IDENT | STRING_LIT ;
AsmValue    = Expr | "_" | Expr "=>" ( Expr | "_" ) ;
```

The adjacent string literals are joined, with a line end between each pair,
into the *template*. An `asm`
expression is an unsafe operation. Its type is `()`, or `never` if it has the
`noreturn` option.

```tally
let lo: u32; let hi: u32;
unsafe { asm may sys ("rdtsc", out("eax") lo, out("edx") hi, options(nomem, nostack)); }
let tsc = ((hi as u64) << 32) | lo as u64;
```
*From: D39.*

**14.4.2** Operands. `R` is either a *register class* (an identifier) or an
explicit register (a string such as `"eax"`). Every target provides at least
the class `reg` (general-purpose registers able to hold a pointer-sized
integer); other classes, the register names, and which types each class
accepts are implementation-defined and documented per target. The value of
an operand must have an integer, float, raw pointer or function pointer type
that the class accepts.

* `in(R) e` evaluates `e` and places its value in a register of `R` before
  the assembly runs; the assembly must leave that register unchanged unless
  the same register is also an output.
* `out(R) p` makes the register's value after the assembly runs be stored
  into the place `p` (or discarded for `_`); the register may be allocated to
  the same register as an input only for `lateout`.
* `lateout(R) p` is `out`, except that the register may share a register with
  an input, so the assembly must read all inputs before writing it.
* `inout(R) p` places the value of `p` in the register and stores the
  register back into `p` afterwards; `inout(R) e => p` takes the input from
  `e` and stores the output into `p` (or discards it for `_`);
  `inlateout` is the same with the sharing rule of `lateout`.
* `clobber_abi("C")` marks every register that the named calling convention
  does not preserve as overwritten.

Operands are evaluated left to right before the assembly runs; outputs are
stored left to right after it. Every register the assembly writes, other than
outputs, must be listed through `clobber_abi` or as an output to `_`. In the
template, `{N}` is replaced by the register (or memory operand syntax, for
targets that use it) of the N-th operand counting from 0, `{name}` by the
operand named `name`, `{N:m}` and `{name:m}` apply the target-defined
modifier `m` (for example a sub-register name), and `{{` and `}}` stand for
literal braces.

Options: `nomem` (the assembly neither reads nor writes memory that the
program can observe); `readonly` (it does not write such memory); `pure` (it
has no side effects and its outputs depend only on its inputs, so it may be
removed or merged; requires `nomem` or `readonly`); `preserves_flags` (it does
not change condition flags); `noreturn` (control never leaves it; the
expression has type `never` and must have no outputs); `nostack` (14.4.4);
`att_syntax` (x86 only: AT&T operand order). Without `nomem`/`readonly`, the
assembly may read and write any memory reachable through pointers passed as
inputs or exposed (13.4.1). The template's own syntax is that of the target's
assembler. (These rules follow the design of Rust's `asm!`, from which the
operand syntax was taken.)

```tally
let x: u64 = 5;
var y: u64;
unsafe { asm ("lea {o}, [{i} + {i}]", i = in(reg) x, o = out(reg) y); }   // y = 10
```
*From: D39.*

**14.4.3** An `asm` expression's effect set is its `may` clause, or `sys` if
the clause is absent. It is trusted like a foreign declaration's (14.1.3).

```tally
unsafe { asm may none ("pause"); }       // a spin hint: no effects
unsafe { asm ("wfi"); }                  // default: may sys
```
*From: D39, D3.*

**14.4.4** The `stack N` clause states the maximum number of bytes below the
stack pointer's value on entry to the assembly that the assembly reaches,
either by moving the stack pointer or by accessing memory below it,
including anything it calls; it defaults to 0. It is added to the
enclosing function's frame size (11.2.3). With the `nostack` option, the
assembly must not use the stack at all (not even below the stack pointer),
and a `stack` clause is an error.

```tally
unsafe { asm may sys stack 8 ("push rbx", "cpuid", "pop rbx", inout("eax") leaf => a, out("ecx") c, out("edx") d); }
```
*From: D39, D33.*
