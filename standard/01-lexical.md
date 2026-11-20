# 1. Lexical structure

This chapter defines how source text is divided into tokens.

## 1.1 Source text

**1.1.1** A source file is a sequence of bytes that must be valid UTF-8. A
file that is not valid UTF-8 is an error. If the file begins with the byte
order mark U+FEFF, the mark is ignored. A U+FEFF anywhere else outside a
string or character literal or comment is an error.

```text
EF BB BF 66 6E ...   (BOM, then "fn ...")   -> accepted, BOM ignored
```
*From: D8.*

**1.1.2** A line ends at U+000A (LF) or at the pair U+000D U+000A (CR LF).
A U+000D not immediately followed by U+000A is an error. Line numbers used in
diagnostics, `@cfg` and `core.source.line()` (19.17.1) count line ends,
starting from 1.

```text
"let x = 1;\r\nlet y = 2;\n"   -> two lines
"let x = 1;\rlet y = 2;"       -> error: lone carriage return
```
*From: D8, D2.*

**1.1.3** A source file's name, without the `.tly` extension, must be a valid
identifier that is not a keyword (1.3.2). Its path within the package
determines its module path (3.1.2).

```text
src/net/tcp_stream.tly    -> valid
src/net/tcp-stream.tly    -> error: "tcp-stream" is not an identifier
src/fn.tly                -> error: "fn" is a keyword
```
*From: D18.*

## 1.2 Whitespace and comments

**1.2.1** Whitespace is U+0020 (space), U+0009 (tab), and line ends.
Whitespace separates tokens and is otherwise ignored. No other Unicode space
characters are whitespace; outside literals and comments they are errors.

```tally
let   x =	1;   // spaces and tabs are interchangeable here
```
*From: D8, D10.*

**1.2.2** A comment starts with `//` and continues to the end of the line. A
comment that starts with exactly `///` (not `////`) is an *outer doc
comment*. A comment that starts with `//!` is an *inner doc comment*. All
other comments are ignored. There are no block comments.

```tally
// ordinary comment, ignored
/// Documents the item that follows.
//// four slashes: ordinary comment
//! Documents the enclosing module; only allowed before the first import or item.
```
*From: D9.*

**1.2.3** An outer doc comment produces a `DOC` token whose text is the rest
of the line after `///`, with one leading space removed if present.
Consecutive `DOC` tokens with nothing but whitespace between them are joined
into one `DOC` token, their texts separated by LF. A `DOC` token is treated as
the attribute `@doc("text")` (16.3.1) applied to the following item, field,
variant, or trait or impl member. A `DOC` token anywhere else is an error.
Inner doc comments produce an `INNER_DOC` token, joined the same way, which
may appear only at the start of a file and documents the module.

```tally
/// Returns the larger of `a` and `b`.
/// If they are equal, returns `a`.
pub fn max(a: i64, b: i64) -> i64 { if b > a { b } else { a } }
// equivalent to:  @doc("Returns the larger of `a` and `b`.\nIf they are equal, returns `a`.")
```
*From: D9, D11.*

## 1.3 Identifiers and keywords

**1.3.1** An identifier is a sequence of ASCII letters (`A`–`Z`, `a`–`z`),
ASCII digits (`0`–`9`) and underscores (`_`) that does not begin with a digit
and is not a single underscore. A single underscore is the token `_`.
Identifiers are case-sensitive. Identifiers beginning with `__` (two
underscores) are reserved for the implementation; declaring one is an error.

```tally
let count_2 = 0;     // identifier
let _unused = 1;     // identifier
let _ = f();         // the token _, not an identifier
let __x = 1;         // error: reserved identifier
```
*From: D8.*

**1.3.2** The following identifiers are *reserved keywords* and cannot be
used as names:

```text
as      asm     break   const   continue  defer   dyn     else
enum    extern  false   fn      for       if      impl    import
in      let     loop    match   move      mut     orelse  pub
return  self    Self    static  struct    trait   true    type
unsafe  var     where   while
```

The following are *reserved for future use* and cannot be used as names:
`async`, `await`, `macro`, `try`, `yield`.

```tally
let match = 1;   // error: `match` is a keyword
let yield = 2;   // error: `yield` is reserved
```
*From: D10, D2.*

**1.3.3** The following are *contextual keywords*: they are ordinary
identifiers except in the grammar positions listed, where the grammar
(chapter 2) spells them as terminals.

| Word | Is a keyword when it appears |
|---|---|
| `any`, `none` | as an effect term (10.2.3) |
| `effect` | at the start of an item or trait/impl member, or in a generic parameter list, followed by an identifier |
| `from` | after a function's return type, or before an enum variant name |
| `may` | where an effect clause may begin (2.4.2) |
| `stack` | inside an `asm` expression head |
| `test` | at the start of an item, followed by a string literal |
| `union` | at the start of an item, followed by an identifier |
| `view` | at the start of an item, followed by `struct` or `enum` |

A contextual keyword used as a name elsewhere is an ordinary identifier, with
two exceptions: an effect may not be named `any` or `none`, and no item may
be named `may`.

```tally
fn from(view: u32, any: u32) -> u32 { view + any }   // all ordinary identifiers here
pub fn parse(src: &[]u8) -> Token from src may panic { ... }   // `from`, `may` are keywords
```
*From: D3, D28, D31.*

**1.3.4** The names of the primitive types (4.2.1, 4.3.1, 4.4.1), `never`
(4.5.3), and the names in the prelude (19.1.2) are *predeclared
identifiers*. They are not keywords. Declaring an item whose name is a
primitive type name or `never` is an error. Other predeclared names may be
shadowed (3.4.4).

```tally
struct u32 {}          // error: `u32` names a primitive type
fn Vec() {}            // allowed: shadows the prelude's Vec within this module
```
*From: D14, D2.*

## 1.4 Literals

**1.4.1** An *integer literal* is one of:

```text
DEC_LIT = "0" | [1-9] { ["_"] [0-9] } ;
HEX_LIT = "0x" HEXDIG { ["_"] HEXDIG } ;      (* HEXDIG = [0-9a-fA-F] *)
OCT_LIT = "0o" [0-7]  { ["_"] [0-7] } ;
BIN_LIT = "0b" [01]   { ["_"] [01] } ;
```

Underscores may appear only between two digits. A decimal literal with a
leading zero (`012`) is an error. Integer literals have no sign and no type
suffix; `-1` is the negation operator applied to `1`, and the literal's type
comes from context (4.2.4). The value of an integer literal may be arbitrarily
large; whether it fits is checked when its type is known.

```tally
let a = 1_000_000;
let b: u8 = 0xff;
let c: u32 = 0b1010_0101;
let d = 012;          // error: leading zero
let e = 1__000;       // error: underscore not between two digits
```
*From: D14.*

**1.4.2** A *float literal* is:

```text
FLOAT_LIT = DIGITS "." DIGITS [ EXP ] | DIGITS EXP ;
DIGITS    = [0-9] { ["_"] [0-9] } ;
EXP       = ("e" | "E") ["+" | "-"] DIGITS ;
```

A float literal needs digits on both sides of the point: `1.` and `.5` are not
float literals. `1.e5` is the integer `1` followed by the field access `.e5`.
Its type comes from context (4.4.3).

```tally
let x = 1.5;
let y = 2e-3;
let z: f32 = 6.02e23;
let w = .5;     // error: expected expression
```
*From: D14.*

**1.4.3** A *character literal* is a single Unicode scalar value or escape
sequence between single quotes: `'a'`, `'\n'`, `'\u{1F600}'`. It has type
`char`. A *byte literal* is `b` followed by a single-quoted ASCII character
(U+0020–U+007E except `'` and `\`) or escape; it has type `u8`. In a byte
literal, `\u{...}` is an error and `\xHH` may have any value 00–FF.

```tally
let c: char = 'é';
let nl: u8 = b'\n';
let hi: u8 = b'\xff';
let bad = b'é';    // error: byte literal must be ASCII
```
*From: D57.*

**1.4.4** The escape sequences are:

| Escape | Meaning |
|---|---|
| `\n` `\r` `\t` | LF, CR, tab |
| `\\` `\'` `\"` | backslash, single quote, double quote |
| `\0` | U+0000 |
| `\xHH` | byte HH; in character and string literals HH must be 00–7F |
| `\u{H...}` | Unicode scalar value with 1–6 hex digits; surrogates (D800–DFFF) and values above 10FFFF are errors |

Any other character after a backslash is an error.

```tally
let s = "tab\there \u{263A} \x41";   // "tab<TAB>here ☺ A"
let t = "\q";                         // error: unknown escape
```
*From: D57.*

**1.4.5** A *string literal* is a sequence of characters and escapes between
double quotes. It must not contain a raw line end; use `\n` or a raw string. Its
type is `&str` (19.6) referring to static data. A *byte string literal* is `b`
followed by a string literal whose characters are all ASCII; its type is
`&[N]u8` where N is the number of bytes after escapes are applied.

```tally
let greeting: &str = "hello";
let magic = b"\x7fELF";        // type &[4]u8
let multi = "first
second";                        // error: line end in string literal
```
*From: D57, D12.*

**1.4.6** A *C string literal* is `c` immediately followed by a string
literal (or `cr` followed by a raw string literal). It denotes the literal's
bytes followed by one zero byte, has type `&CStr` (14.2.2), and must not
contain a zero byte. Its token class is `CSTRING_LIT`.

```tally
let path = c"/dev/null";      // bytes "/dev/null\0"
let bad = c"a\0b";            // error: interior zero byte
```
*From: D38.*

**1.4.7** A *raw string literal* is `r`, then zero or more `#`, then `"`, then
any text, then `"` followed by the same number of `#`. No escapes are
processed and line ends are allowed; a CR LF inside a raw string is stored as
LF. `br` introduces a raw byte string, whose content must be ASCII, and `cr` a raw C string.

```tally
let path = r"C:\temp\new";
let quoted = r#"she said "hi""#;
let text = r"line one
line two";                      // contains one LF
```
*From: D57.*

**1.4.8** `true` and `false` are the literals of type `bool`. There is no null
literal; the absent value of an optional type is written `.None` (4.6.1).

```tally
let ready = false;
let p: ?&u32 = .None;
```
*From: D19.*

## 1.5 Punctuation and tokenisation

**1.5.1** The punctuation tokens are:

```text
(   )   [   ]   {   }   ,   ;   :   .   ..  ..=  ...  @   _
=   ==  !=  <   <=  >   >=  !   ~   ?   ->  =>
+   -   *   /   %   &   |   ^   <<  >>  &&  ||
+%  -%  *%  +|  -|  *|
+=  -=  *=  /=  %=  &=  |=  ^=  <<= >>=
+%= -%= *%= +|= -|= *|=
```

and the token `as%`, formed by the keyword `as` immediately followed by `%`
with no whitespace between them.

```tally
let w = x +% 1;        // wrapping add
let s = x +| 1;        // saturating add
let b = big as% u8;    // truncating conversion; `as%` is one token
```
*From: D15, D16.*

**1.5.2** Tokenisation is greedy: at each position the lexer takes the longest
sequence of characters that forms a token. Thus `a+%b` is `a`, `+%`, `b`, and
`x..=y` is `x`, `..=`, `y`. Where the grammar needs two tokens that the lexer
has merged, the parser splits them: `&&` in a position where a prefix operator
is expected is two `&` tokens, and `||` in a position where a closure may
begin is an empty closure parameter list.

```tally
let r = &&x;          // reference to a reference to x
let f = || 42;        // closure with no parameters
let z = 1..=10;       // inclusive range
```
*From: D10, D2.*

**1.5.3** A file is tokenised in full before parsing. A character sequence
that is not whitespace, a comment, a literal, an identifier, a keyword or
a punctuation token is an error.

```tally
let x = 5 $ 3;   // error: unexpected character `$`
```
*From: D2.*
