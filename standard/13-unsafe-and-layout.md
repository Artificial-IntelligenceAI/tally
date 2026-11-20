# 13. Unsafe code, layout and undefined behaviour

## 13.1 Unsafe code

**13.1.1** An *unsafe context* is the inside of an `unsafe` block, excluding
the bodies of closures written inside it. The body of an `unsafe fn` is not an
unsafe context; unsafe operations in it need their own `unsafe` blocks.
Declaring a function `unsafe fn` means that callers must uphold conditions,
stated in its documentation, that the compiler cannot check.

```tally
/// # Safety
/// `p` must point to `n` valid, initialised `u32`s.
pub unsafe fn sum_raw(p: *u32, n: usize) -> u64 may panic {
    var s: u64 = 0;
    for i in 0..n { s += unsafe { *p.add(i) } as u64; }
    s
}
```
*From: D36.*

**13.1.2** The *unsafe operations* are allowed only in an unsafe context:

1. dereferencing a raw pointer (reading, writing, or borrowing through it);
2. calling an `unsafe fn`, an `unsafe fn` pointer, or a foreign function;
3. reading, writing or borrowing a `static var` or an extern static;
4. reading a field of a union;
5. an `asm` expression;
6. an `unsafe may` block (which is itself an unsafe block).

Implementing an `unsafe trait` requires `unsafe impl` (7.1.4),
`@drop_ignores` requires `unsafe impl` (9.6.2), and the unsafe attributes
`@unsafe(export(...))` and `@unsafe(section(...))` (16.3.1) must be written in
that form; these are the only unsafe declarations.

```tally
let p: *mut u8 = buf.as_mut_ptr();
unsafe { *p = 1; }
let x = *p;                     // error: dereference of raw pointer outside unsafe
```
*From: D36.*

## 13.2 Undefined behaviour

**13.2.1** The following, and nothing else, is *undefined behaviour*. A
program whose execution performs any of them has no defined meaning, for that
entire execution. Each can occur only through unsafe operations (or through
foreign code or a truthless `unsafe impl`).

1. Accessing memory through a pointer that does not point into a live
   allocated object large enough for the access, or that is misaligned for
   the type accessed (13.4.1), or whose provenance does not permit the access.
2. A data race (15.3.2).
3. Producing an invalid value (13.4.2) of a type, whether by writing it,
   reading it, passing it or returning it.
4. Violating the aliasing rules for references (13.4.4).
5. Modifying memory that is immutable: memory reached only through `&T` and
   not inside an `UnsafeCell`, string and byte string literals, statics not
   declared `var`, and promoted constants.
6. Calling a function through a pointer of an incompatible type or calling
   convention; calling a foreign function in a way its C declaration does not
   allow; foreign code performing undefined behaviour.
7. Foreign code unwinding into, or `longjmp`-ing over, Tally frames.
8. Executing `core.hint.unreachable_unchecked()` (19.18.3).
9. Breaking the documented promise of `@drop_ignores` (9.6.2), of any
   `unsafe impl` (including `Send`, `Sync`, `Allocator`, `GlobalAlloc` and
   `Waiter`), or of an unsafe attribute (an exported symbol that replaces one
   other code relies on; a section that makes code run implicitly); a false
   `@stack_bound` on a foreign function that an exactly sized execution
   context calls (14.1.5); or calling code outside the program's call graph
   through a Tally function pointer (11.6.4).
10. Inline assembly that violates its operands, options, declared stack use,
    or the target's ABI rules for inline assembly (14.4).
11. Reading uninitialised memory as a value of any type other than through
    `core.mem.Uninit` and the raw-byte operations documented as allowing it.
12. Pointer arithmetic (`add`, `sub`, `offset`) whose result lies outside the
    object the pointer's provenance refers to, other than one byte past its
    end.
13. Violating the documented safety conditions of an `unsafe fn` in `core` or
    `std`, where the documentation states that doing so is undefined
    behaviour.

```tally
let x: u32 = 7;
let p: *u32 = &x;                     // coercion &u32 -> *u32
let q = unsafe { p.add(2) };          // UB (12): two past the end of a single u32
let b: bool = unsafe { mem.transmute(2 as u8) };   // UB (3): 2 is not a valid bool
```
*From: D36, D35.*

**13.2.2** Undefined behaviour is not a diagnostic category: the
implementation need not detect it. Implementations may provide checked build
modes that detect some cases at run time.

```text
tally test --sanitize=address      (an implementation-defined option)
```
*From: D36, D50.*

## 13.3 Layout

**13.3.1** The size and alignment of the primitive types are as given in
chapter 4: `bool` and `u1`–`u8` 1 byte; integers by storage size (4.2.2);
`char` 4; `f32` 4; `f64` 8; `()` and `never` 0 bytes with alignment 1;
`usize`, `isize`, thin references and raw pointers the target's pointer size;
function pointers the target's code pointer size. Every type's size is a
multiple of its alignment, and alignments are powers of two.

```tally
// on x86_64-linux:  size_of.[(u8, u32)]() == 8, align_of.[(u8, u32)]() == 4
```
*From: D37.*

**13.3.2** A struct (or tuple, laid out as a struct with fields `0`, `1`, …)
places its fields in declaration order. Each field is placed at the smallest
offset not less than the end of the previous field that is a multiple of the
field's alignment; the first field is at offset 0. The struct's alignment is
the largest alignment of its fields (1 if none); its size is the end of the
last field rounded up to its alignment. This is the layout a C compiler for
the target gives the corresponding C struct.

```tally
struct Hdr { tag: u8, len: u32, flags: u16 }
// offsets: tag 0, len 4, flags 8; size 12, align 4
```
*From: D37.*

**13.3.3** `@align(N)` on a struct sets its alignment to the larger of N and
its natural alignment; on a field, it raises that field's alignment to at
least N. N must be a power of two. `@packed` on a struct sets every field's
alignment to 1 for layout purposes, so fields are placed at consecutive byte
offsets and the struct has alignment 1 (unless `@align` is also given). Any
borrow, explicit or implicit (a `&`/`&mut` expression, an auto-referenced
method receiver, a binding in *ref* mode, a closure capture by reference), of
a place whose path passes through a field of a `@packed` struct is an error
unless the borrowed place's type has alignment 1; such fields are read and
written by copying.

```tally
@packed struct Wire { kind: u8, len: u32 }      // size 5, len at offset 1
let l = w.len;                                  // fine: copied
let r = &w.len;                                 // error: reference to unaligned packed field
@align(64) struct CacheLine { counter: Atomic[u64] }   // size 64, align 64
```
*From: D37.*

**13.3.4** `@bitpacked(uN)` on a struct lays out its fields as consecutive bit
fields of a backing integer of type `uN`, starting at the least significant
bit, in declaration order. Field types must be integer types, `bool` (1 bit),
enums with a declared tag type and only unit variants (occupying the tag
type's width), or other `@bitpacked` structs (occupying their backing width).
The field widths must add up to exactly N. The struct has the size, alignment
and byte order of `uN`. Fields are read and written by shifting and masking;
any borrow (explicit or implicit) of a field is an error. A bitpacked struct converts to and from its
backing integer with `s.bits()` and `S.from_bits(x)`; `from_bits` panics if a
bit pattern is not valid for an enum field.

```tally
enum Mode: u2 { Idle = 0, Rx = 1, Tx = 2, Off = 3 }
@bitpacked(u16)
struct CtrlReg { enable: bool, mode: Mode, prescale: u5, reserved: u8 }
// bit 0: enable, bits 1-2: mode, bits 3-7: prescale, bits 8-15: reserved
var r = CtrlReg.from_bits(0);
r.mode = .Tx;
unsafe { mmio.write_volatile(r.bits()); }
```
*From: D37, D14.*

**13.3.5** An enum with at least one variant is laid out as a struct of two
fields: a *tag* and a *payload*. The tag's type is the declared tag type, or
else the smallest of `u8`, `u16`, `u32` able to hold the largest tag value.
The payload is a union of one struct per variant, each laid out by 13.3.2 from
that variant's fields. Tag values are those of 4.10.5. An enum whose variants are all unit variants is just its tag. An enum
with no variants has size 0.

```tally
enum Shape { Circle(f32), Rect(f32, f32) }
// tag: u8 at offset 0; payload at offset 4 (align of f32): size 12, align 4
enum Dir { N, E, S, W }      // size 1
```
*From: D37.*

**13.3.6** As the only exception to 13.3.5, `Option[T]` where `T` is a
reference, a non-null raw pointer, a `Box` with a zero-sized allocator, a
function pointer, or `core.num.NonZero[N]`, has the same size and alignment as
`T`; `.None` is represented by the all-zero bit pattern and `.Some(v)` by `v`.

```tally
// size_of.[?&u8]() == size_of.[&u8]()
// size_of.[?*mut Node]() == 8 on a 64-bit target, matching a C `Node*`
// size_of.[?u32]() == 8 (tag + payload)
```
*From: D37, D19.*

**13.3.7** An array `[N]T` places its elements at offsets `i × size_of(T)`; a
slice's elements are laid out the same way. A union places all fields at
offset 0; its alignment is the largest field alignment and its size the
largest field size rounded up to that alignment. A reference or raw pointer to
`[]T` or `str` is laid out as `(address: *T, length: usize)`, and one to `dyn
Tr` as `(address, vtable address)`. `Box[T, A]` is laid out as a struct of a
pointer to `T` followed by the `A` value. Closure and function item layouts are
implementation-defined (function items have size 0).

```tally
// size_of.[&[]u8]() == 2 * size_of.[usize]()
// size_of.[Box[u64]]() == size_of.[usize]()      (Global is zero-sized)
```
*From: D37.*

**13.3.8** No type may have a size greater than `isize.MAX` bytes; a type
whose size would exceed it is an error.

```tally
let big: [1 << 62][4]u8 = ...;    // error on a 64-bit target: size exceeds isize.MAX
```
*From: D37, D2.*

## 13.4 Validity, provenance and aliasing

**13.4.1** Every pointer value has an address and a *provenance*: the
allocated object (a local, a static, a heap block, or memory provided by
foreign code) it was derived from. Pointers created by borrowing, from
allocators, or from foreign code have the provenance of that object; pointers
computed from other pointers (`add`, `cast`, field projection) keep it.
`p.addr()` returns the address as `usize` without affecting provenance;
`p.expose()` returns the address and marks the object as *exposed*;
`ptr.from_exposed(a)` creates a pointer whose provenance is the exposed object
containing address `a` (undefined behaviour to use it if there is none);
`p.with_addr(a)` creates a pointer with `p`'s provenance and address `a`. A
memory access is allowed by a pointer's provenance if it lies entirely within
the object. Memory that belongs to no object the program created or received
(device registers, memory mapped by the platform before the program starts)
is treated as part of exposed objects provided by the platform, so pointers to
it are created with `ptr.from_exposed`.

```tally
let a = buf.as_ptr().expose();
let p: *u8 = unsafe { ptr.from_exposed(a + 4) };    // provenance: buf
let tagged = p.with_addr(p.addr() | 1);              // same provenance, low-bit tag
```
*From: D36.*

**13.4.2** The *valid values* of each type are: for integers and floats, any
bit pattern of the right size, with integer padding bits as in 4.2.2; for
`bool`, 0 and 1; for `char`, Unicode scalar values; for references, non-null
aligned pointers to a valid value of the referenced type (for `&[]T` and
`&str`, a length whose total size fits in `isize`, and for `str`, valid UTF-8);
for raw pointers, any non-null address; for `?*T`, any address; for function
pointers, the address of a function of compatible type; for `dyn` pointers, a
vtable produced by the implementation for that trait; for enums, a tag value
of one of its variants, with that variant's fields valid; for structs, tuples
and arrays, valid fields or elements (padding bytes may hold anything); for
unions, any bytes; `never` and empty enums have no valid values.

```tally
let c: char = unsafe { mem.transmute(0xD800 as u32) };   // UB: invalid char
let z: &u8 = unsafe { mem.transmute(0 as usize) };       // UB: null reference
```
*From: D36, D37.*

**13.4.3** Converting a raw pointer to a reference (`&*p`, `&mut *p`)
requires, for as long as the reference or anything derived from it is used,
that the pointee is a valid value, is not deallocated, and is not accessed in
a way that 13.4.4 forbids.

```tally
let r: &Node = unsafe { &*node_ptr };
free(node_ptr);
print(r.value);          // UB: r used after its pointee was deallocated
```
*From: D36, D24.*

**13.4.4** The *use period* of a reference value is from its creation to the
last use of it or of any pointer or reference *derived* from it (by
reborrowing, field or index projection, conversion to a raw pointer, or
copying). During the use period:

* of an exclusive reference `&mut T` to memory `m`: `m` must not be read or
  written through any pointer not derived from that reference;
* of a shared reference `&T` to memory `m`: the parts of `m` not inside an
  `UnsafeCell` must not be written through any pointer, and no exclusive
  reference to any part of `m` may be used.

Within these rules, accesses through raw pointers are unrestricted relative to
each other. The compiler may assume the rules hold (for example, that memory
behind a `&mut` parameter is not changed by an unknown call that does not
receive it).

```tally
var x = 1;
let p: *mut i64 = &mut x;
let r = &mut x;
unsafe { *p = 2; }          // UB: x written through p, not derived from r, during r's use period
*r += 1;
```
*From: D36, D24.*

**13.4.5** A *volatile* access (`p.read_volatile()`, `p.write_volatile(v)`)
is performed exactly as written: it is never removed, merged, duplicated or
reordered with other volatile accesses, and it accesses memory with a single
instruction where the target allows one of that size. Volatile accesses
contribute the `sys` effect (10.3.4). Volatile does not provide atomicity or
inter-thread ordering.

```tally
const UART_TX: usize = 0x4000_1000;
let reg: *mut u32 = unsafe { ptr.from_exposed(UART_TX) };
unsafe { reg.write_volatile(b'A' as u32); }
```
*From: D36, D3.*
