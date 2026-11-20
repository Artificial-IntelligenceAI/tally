# 2. Grammar

This chapter gives the complete context-free grammar of Tally in the notation
of 0.3.1. Token classes are those of chapter 1: `IDENT`, `INT_LIT`,
`FLOAT_LIT`, `CHAR_LIT`, `BYTE_LIT`, `STRING_LIT` (including raw strings),
`BYTE_STRING_LIT`, `DOC`, `INNER_DOC`. Contextual keywords (1.3.3) are
written as quoted terminals in the positions where they are keywords.

**2.0.1** A source file is well-formed only if its token sequence is derived
from `File` by this grammar together with the disambiguation rules of 2.10.
Any other token sequence is an error. Semantic rules in later chapters may
reject programs the grammar accepts.

```tally
fn f( { }   // error: expected parameter or `)`
```
*From: D6, D2.*

## 2.1 Files and imports

**2.1.1** File structure:

```text
File        = [ INNER_DOC ] { Import } { Item } ;
Import      = { Attribute } [ "pub" ] "import" ImportSpec ";" ;
ImportSpec  = Path [ "as" IDENT ]
            | Path "." "{" ImportName % "," "}" ;
ImportName  = IDENT [ "as" IDENT ] | "self" [ "as" IDENT ] ;
Path        = IDENT { "." IDENT } ;
```

All imports come before all items.

```tally
//! A small HTTP server.
import std.io;
import std.net.{TcpListener, TcpStream as Stream};
pub import this_pkg.util.Buffer;
fn main() {}
import std.fs;     // error: import after an item
```
*From: D18, D10.*

## 2.2 Items

**2.2.1** Items:

```text
Item        = { Attribute } [ "pub" ] ItemKind ;
ItemKind    = FnDecl | StructDecl | EnumDecl | UnionDecl | TraitDecl
            | ImplDecl | TypeAlias | ConstDecl | StaticDecl | EffectDecl
            | ExternBlock | TestDecl ;

StructDecl  = [ "view" ] "struct" IDENT [ GenericParams ] [ WhereClause ]
              ( "{" [ FieldDecl % "," ] "}"
              | "(" [ TupleField % "," ] ")" ";"
              | ";" ) ;
FieldDecl   = { Attribute } [ "pub" ] IDENT ":" Type ;
TupleField  = { Attribute } [ "pub" ] Type ;

EnumDecl    = [ "view" ] "enum" IDENT [ GenericParams ] [ ":" Type ] [ WhereClause ]
              "{" [ Variant % "," ] "}" ;
Variant     = { Attribute } [ "from" ] IDENT
              [ "(" Type % "," ")" | "{" [ FieldDecl % "," ] "}" ]
              [ "=" Expr ] ;

UnionDecl   = "union" IDENT [ GenericParams ] [ WhereClause ]
              "{" FieldDecl % "," "}" ;

TypeAlias   = "type" IDENT [ GenericParams ] "=" Type ";" ;
ConstDecl   = "const" IDENT ":" Type "=" Expr ";" ;
StaticDecl  = "static" [ "var" ] IDENT ":" Type "=" Expr ";" ;
EffectDecl  = "effect" IDENT ";" ;
TestDecl    = "test" STRING_LIT Block ;
```

```tally
pub struct Point { pub x: f64, pub y: f64 }
pub view struct Cursor { bytes: &[]u8, pos: usize }
struct Meters(f64);
struct Marker;
enum ParseError { from Io(io.Error), Eof, Bad { line: u32 } }
enum Color: u8 { Red = 1, Green = 2, Blue = 4 }
type Bytes = Vec[u8];
const LIMIT: usize = 64;
static var TICKS: u64 = 0;
pub effect db;
test "points add" { assert(Point { x: 1.0, y: 2.0 }.x == 1.0); }
```
*From: D17, D19, D20, D24, D28, D49.*

**2.2.2** Traits and implementations:

```text
TraitDecl   = [ "unsafe" ] "trait" IDENT [ GenericParams ] [ ":" Bounds ]
              [ WhereClause ] "{" { TraitMember } "}" ;
TraitMember = { Attribute } ( FnDecl
            | "type" IDENT [ ":" Bounds ] [ "=" Type ] ";"
            | "effect" IDENT [ "=" EffectExpr ] ";"
            | "const" IDENT ":" Type [ "=" Expr ] ";" ) ;

ImplDecl    = [ "unsafe" ] "impl" [ GenericParams ] [ TraitRef "for" ] Type
              [ WhereClause ] "{" { ImplMember } "}" ;
ImplMember  = { Attribute } [ "pub" ] ( FnDecl
            | "type" IDENT "=" Type ";"
            | "effect" IDENT "=" EffectExpr ";"
            | ConstDecl ) ;
```

In `ImplDecl`, a `[` immediately after `impl` always begins `GenericParams`.

```tally
pub trait Iterator {
    type Item;
    effect Next;
    fn next(&mut self) -> ?Self.Item may Self.Next;
}
impl[T] Iterator for SliceIter[T] {
    type Item = &T;
    effect Next = none;
    fn next(&mut self) -> ?&T from *self { ... }
}
```
*From: D27, D30.*

**2.2.3** Foreign blocks:

```text
ExternBlock = "extern" STRING_LIT "{" { ExternItem } "}" ;
ExternItem  = { Attribute } [ "pub" ]
              ( "fn" IDENT "(" [ ExternParams ] ")" [ "->" Type ]
                [ EffectClause ] ";"
              | "static" [ "var" ] IDENT ":" Type ";" ) ;
ExternParams = Param { "," Param } [ "," "..." ] [ "," ] ;
```

```tally
extern "C" {
    fn write(fd: i32, buf: *u8, n: usize) -> isize may sys | block;
    fn printf(fmt: *u8, ...) -> i32 may sys | block;
    static environ: *?*u8;
}
```
*From: D38.*

## 2.3 Generic parameters and bounds

**2.3.1** Generic parameters, bounds and where clauses:

```text
GenericParams = "[" GenericParam % "," "]" ;
GenericParam  = IDENT [ ":" Bounds ] [ "=" Type ]
              | "const" IDENT ":" Type [ "=" Expr ]
              | "effect" IDENT ;
Bounds        = Bound { "+" Bound } ;
Bound         = "?" IDENT                          (* only ?Sized *)
              | TraitRef [ EffectClause ] ;
TraitRef      = TypePath
              | TypePath "(" [ Type % "," ] ")" [ "->" Type ] ;
WhereClause   = "where" WherePred % "," ;
WherePred     = Type ":" Bounds ;
```

The parenthesised form of `TraitRef` is allowed only for the traits `Fn`,
`FnMut` and `FnOnce` (7.7.2).

```tally
fn apply[F: Fn(u32) -> u32 may none](f: F, x: u32) -> u32 { f(x) }
struct Buf[T, A: Allocator = Global, const N: usize = 16] { ... }
fn show[T: ?Sized + Display](x: &T) may panic { ... }
fn sum[I](it: I) -> u64 may I | panic where I: Iterator[Item = u64] { ... }
```
*From: D13, D27, D30, D59.*

## 2.4 Functions

**2.4.1** Function declarations:

```text
FnDecl      = [ "unsafe" ] [ "extern" STRING_LIT ] "fn" IDENT [ GenericParams ]
              "(" [ ParamList ] ")" [ ReturnClause ] [ FromClause ]
              [ EffectClause ] [ WhereClause ] ( Block | ";" ) ;
ParamList   = ( SelfParam [ "," [ Param % "," ] ] ) | Param % "," ;
SelfParam   = [ "var" ] "self" | "&" "self" | "&" "mut" "self" ;
Param       = [ "var" ] ( IDENT | "_" ) ":" Type
            | ".." IDENT ":" Type ;
ReturnClause = "->" Type | "!" PrefixType ;
FromClause  = "from" FromSource % "," ;
FromSource  = { "*" } ( IDENT | "self" ) ;
```

A body-less declaration (`;`) is allowed only as a trait member. A rest
parameter (`..name: T`) must be the last parameter (5.7.5).

```tally
pub fn read(&mut self, buf: &mut []u8) -> usize ! io.Error may sys | block { ... }
fn flush(&mut self) ! io.Error may sys | block { ... }      // same as -> () ! io.Error
fn longest(a: &str, b: &str) -> &str from a, b { if a.len() >= b.len() { a } else { b } }
pub fn println[A: Display](..args: A) may sys | block { ... }
```
*From: D12, D24, D31, D47.*

**2.4.2** Effect clauses:

```text
EffectClause = "may" EffectExpr ;
EffectExpr   = EffectTerm { "|" EffectTerm } ;
EffectTerm   = "none" | "any" | TypePath ;
```

An effect clause may appear after a function's return/from clauses, at the
end of a function type, after a trait in a bound, at the end of a `dyn` type,
in an `unsafe may` block, and in an `asm` head. Its extent ends at the first
token that is not `|` followed by an `EffectTerm`.

```tally
fn f() may alloc | panic { ... }
let g: fn(u8) -> u8 may panic = h;
fn k[T: Iterator may none + Clone](t: T) { ... }   // `+` ends the effect clause
let w: &mut dyn Write may sys | block = &mut file;
```
*From: D31.*

## 2.5 Types

**2.5.1** Types:

```text
Type        = PrefixType [ "!" PrefixType ] ;
PrefixType  = "?" PrefixType
            | "&" [ "mut" ] PrefixType
            | "*" [ "mut" ] PrefixType
            | "[" Expr "]" PrefixType
            | "[" "]" PrefixType
            | AtomType ;
AtomType    = TypePath
            | "(" ")"
            | "(" Type ")"
            | "(" Type "," [ Type % "," ] ")"
            | FnType
            | "dyn" TraitRef [ EffectClause ] ;
TypePath    = TypeSeg { "." TypeSeg } ;
TypeSeg     = ( IDENT | "Self" ) [ GenericArgs ] ;
FnType      = [ "unsafe" ] [ "extern" STRING_LIT ] "fn"
              "(" [ Type % "," ] ")" [ "->" Type ] [ EffectClause ] ;
GenericArgs = "[" GenericArg % "," "]" ;
GenericArg  = IDENT "=" ( Type | EffectExpr )
            | "none" | "any"
            | Type [ "|" EffectTerm { "|" EffectTerm } ]
            | ConstArg ;
ConstArg    = INT_LIT | "-" INT_LIT | CHAR_LIT | "true" | "false" | Block ;
```

`!` binds more loosely than every prefix constructor: `?u8 ! E` is
`(?u8) ! E`; write `?(u8 ! E)` for an optional result. `!` does not chain:
`A ! B ! C` is an error.

```tally
let a: [4]u8 = [0; 4];
let s: &[]u8 = &a;
let r: ?&mut Node = .None;
let p: *mut u32 = ptr;
let f: fn(&str) -> usize ! ParseError may panic = parse_len;
let it: Box[dyn Iterator[Item = u8] may sys] = ...;
let fmt: Formatter[sys | block] = ...;    // effect argument
let buf: Buf[u8, Global, 32] = ...;        // const argument
```
*From: D12, D13, D31.*

## 2.6 Blocks and statements

**2.6.1** Blocks and statements:

```text
Block       = "{" { Stmt } [ Expr ] "}" ;
Stmt        = ";"
            | { Attribute } LetStmt
            | { Attribute } DeferStmt
            | { Attribute } AssignStmt
            | { Attribute } ExprStmt ;
LetStmt     = ( "let" | "var" ) Pattern [ ":" Type ]
              [ "=" Expr [ "else" Block ] ] ";" ;
DeferStmt   = "defer" ( BlockExpr | ExprNoBlock ";" ) ;
AssignStmt  = ( Expr | "_" ) AssignOp Expr ";" ;
AssignOp    = "=" | "+=" | "-=" | "*=" | "/=" | "%=" | "&=" | "|=" | "^="
            | "<<=" | ">>=" | "+%=" | "-%=" | "*%=" | "+|=" | "-|=" | "*|=" ;
ExprStmt    = BlockExpr [ ";" ] | ExprNoBlock ";" ;
```

`BlockExpr` and `ExprNoBlock` are defined in 2.7.1. The optional final `Expr`
of a block is its *tail expression*.

```tally
{
    let a = 1;
    var b: u32;
    b = 2;
    defer log.flush();
    if a > 0 { b += 1; }        // block expression statement, no `;` needed
    let .Some(v) = lookup(a) else { return; };
    _ = compute();              // discard a value explicitly
    a + b                       // tail expression: the block's value
}
```
*From: D10, D52, D56, D60.*

## 2.7 Expressions

**2.7.1** Expressions:

```text
Expr        = JumpExpr | ClosureExpr | RangeExpr ;
JumpExpr    = "return" [ Expr ]
            | "break" [ ":" IDENT ] [ Expr ]
            | "continue" [ ":" IDENT ] ;
ClosureExpr = [ "move" ] ClosureParams [ "->" Type ] [ EffectClause ]
              ( Block | Expr ) ;
ClosureParams = "||" | "|" [ ClosureParam % "," ] "|" ;
ClosureParam  = [ "var" ] ( IDENT | "_" ) [ ":" Type ] ;

RangeExpr   = [ OrElseExpr ] ( ".." | "..=" ) [ OrElseExpr ] | OrElseExpr ;
OrElseExpr  = OrExpr [ "orelse" OrElseExpr ] ;
OrExpr      = AndExpr { "||" AndExpr } ;
AndExpr     = CmpExpr { "&&" CmpExpr } ;
CmpExpr     = BitOrExpr [ CmpOp BitOrExpr ] ;
CmpOp       = "==" | "!=" | "<" | "<=" | ">" | ">=" ;
BitOrExpr   = BitXorExpr { "|" BitXorExpr } ;
BitXorExpr  = BitAndExpr { "^" BitAndExpr } ;
BitAndExpr  = ShiftExpr { "&" ShiftExpr } ;
ShiftExpr   = AddExpr { ( "<<" | ">>" ) AddExpr } ;
AddExpr     = MulExpr { ( "+" | "-" | "+%" | "-%" | "+|" | "-|" ) MulExpr } ;
MulExpr     = CastExpr { ( "*" | "/" | "%" | "*%" | "*|" ) CastExpr } ;
CastExpr    = UnaryExpr { ( "as" | "as%" ) PrefixType } ;
UnaryExpr   = ( "-" | "!" | "~" | "*" | "&" [ "mut" ] ) UnaryExpr
            | PostfixExpr ;
PostfixExpr = Primary { Postfix } ;
Postfix     = "." IDENT
            | "." INT_LIT
            | "." "[" Type % "," "]"
            | "(" [ Arg % "," ] ")"
            | "[" Expr "]"
            | "?" ;
Arg         = Expr ;

Primary     = Literal | IDENT | "self" | "Self"
            | "." IDENT                         (* inferred enum variant *)
            | "(" Expr ")" | "(" ")" | "(" Expr "," [ Expr % "," ] ")"
            | "[" [ Expr % "," ] "]" | "[" Expr ";" Expr "]"
            | StructLit
            | BlockExpr
            | AsmExpr ;
Literal     = INT_LIT | FLOAT_LIT | CHAR_LIT | BYTE_LIT | STRING_LIT
            | BYTE_STRING_LIT | "true" | "false" ;
StructLit   = StructHead "{" [ FieldInit % "," ] [ ".." Expr ] "}" ;
StructHead  = PostfixExpr      (* restricted by 2.10.3 *) ;
FieldInit   = IDENT [ ":" Expr ] ;

BlockExpr   = [ IDENT ":" ] ( Block | LoopExpr | WhileExpr | ForExpr )
            | "unsafe" [ EffectClause ] Block
            | IfExpr | MatchExpr ;
ExprNoBlock = any Expr that is not a BlockExpr ;
IfExpr      = "if" Cond Block [ "else" ( Block | IfExpr ) ] ;
Cond        = ExprNS | "let" Pattern "=" ExprNS ;
LoopExpr    = "loop" Block ;
WhileExpr   = "while" Cond Block ;
ForExpr     = "for" Pattern "in" ExprNS Block ;
MatchExpr   = "match" ExprNS "{" [ MatchArm { MatchArm } ] "}" ;
MatchArm    = Pattern [ "if" Expr ] "=>" ( BlockExpr [ "," ] | ExprNoBlock "," ) ;
ExprNS      = Expr   (* with no StructLit outside parentheses; 2.10.3 *) ;
```

The last arm of a `match` may omit its `,` after an `ExprNoBlock`. The `asm`
expression is defined in 14.4.1.

```tally
let v = x.items[i].count.[u64]();      // `.[u64]`: explicit generic argument
let y = -a as i32 * 2;                  // ((-a) as i32) * 2
let z = a & mask == 0;                  // (a & mask) == 0
let w = cfg.port orelse 8080;
let r = 0..n;
let p = Point { x: 1.0, ..origin };
let k = outer: loop { break :outer 7; };
let q = match shape { .Circle(r) => r * r, .Square(s) => s * s };
let c = move |n: u32| n + offset;
```
*From: D10, D13, D15, D16, D19, D53, D54.*

**2.7.2** The binary operators group as follows, from loosest to tightest:

| Level | Operators | Associativity |
|---|---|---|
| 1 | `..` `..=` | none |
| 2 | `orelse` | right |
| 3 | `\|\|` | left |
| 4 | `&&` | left |
| 5 | `==` `!=` `<` `<=` `>` `>=` | none |
| 6 | `\|` | left |
| 7 | `^` | left |
| 8 | `&` | left |
| 9 | `<<` `>>` | left |
| 10 | `+` `-` `+%` `-%` `+\|` `-\|` | left |
| 11 | `*` `/` `%` `*%` `*\|` | left |
| 12 | `as` `as%` | left |
| 13 | prefix `-` `!` `~` `*` `&` `&mut` | — |
| 14 | postfix `.f` `.0` `.[T]` `()` `[]` `?` | — |

"None" means the operator cannot be chained without parentheses: `a < b < c`
and `a..b..c` are errors.

```tally
let ok = 0 <= i && i < n;      // fine
let bad = 0 <= i < n;          // error: comparison operators do not chain
let m = *p.field;              // *(p.field): postfix binds tighter than prefix
```
*From: D16, D2.*

## 2.8 Patterns

**2.8.1** Patterns:

```text
Pattern     = PatternOne { "|" PatternOne } ;
PatternOne  = "_"
            | [ "var" ] IDENT
            | LitPat [ ( ".." | "..=" ) LitPat ]
            | "&" [ "mut" ] PatternOne
            | "(" [ PatElem % "," ] ")"
            | "[" [ PatElem % "," ] "]"
            | PatPath [ "(" [ PatElem % "," ] ")"
                      | "{" [ FieldPat % "," ] [ ".." ] "}" ] ;
LitPat      = Literal | "-" ( INT_LIT | FLOAT_LIT ) | PatPath ;
PatElem     = Pattern | ".." [ IDENT ] ;
FieldPat    = [ "var" ] IDENT [ ":" Pattern ] ;
PatPath     = "." IDENT | IDENT [ "." "[" Type % "," "]" ] "." IDENT { "." IDENT } ;
```

A `PatPath` always contains at least one `.`; a bare identifier in a pattern
is always a new binding (6.2.1).

```tally
match msg {
    .Quit => stop(),
    .Move { x, y: 0 } => horizontal(x),
    .Write(text) | .Echo(text) => print(text),
    Color.Red => red(),
    [first, ..rest] => split(first, rest),
    0..=9 => digit(),
    var other => adjust(&mut other),
}
```
*From: D60.*

## 2.9 Attributes

**2.9.1** Attributes:

```text
Attribute   = DOC | "@" IDENT [ "(" [ AttrArg % "," ] ")" ] ;
AttrArg     = [ IDENT "=" ] Expr ;
```

An attribute argument is parsed as an expression; its meaning is given by the
attribute (chapter 16). Statement attributes may only be `@cfg` (16.3.1) and
`@allow` (16.4.5).

```tally
@derive(Clone, Eq, Debug)
@cfg(all(os("linux"), not(arch("riscv64"))))
@deprecated(since = "1.4", note = "use `open_at`")
pub fn open(path: &str) -> File ! io.Error may sys | block { ... }
```
*From: D11, D41, D42.*

## 2.10 Disambiguation

**2.10.1** At the start of a statement, a `BlockExpr` is parsed as a complete
statement: the parser does not continue it as the left operand of a binary
operator, a postfix operation or a cast. To use such an expression as an
operand, wrap it in parentheses.

```tally
fn f() -> i64 {
    match x { _ => 1 } - 1      // statement `match ...`, then tail `-1`
}
let n = (match x { _ => 1 }) - 1;   // parenthesised: subtraction
```
*From: D10.*

**2.10.2** An `IDENT` followed by `:` at the start of an expression begins a
labelled `BlockExpr` and must be followed by `{`, `loop`, `while` or `for`.
Inside a struct literal's braces, `IDENT ":"` is a field initialiser instead.

```tally
search: for row in grid { for c in row { if c == 0 { break :search; } } }
let p = Point { x: 1.0, y: 2.0 };    // field initialisers, not labels
```
*From: D53.*

**2.10.3** A `StructLit` is recognised only where its `StructHead` is an
identifier, `Self`, a sequence of identifiers joined by `.` (optionally with
one `.[...]`), or `.` followed by an identifier, and is immediately followed by
`{`. In the `ExprNS` positions (the condition of `if` and `while`, the
scrutinee of `match` and `if let`, the iterable of `for`), a `{` after such a
head is never a struct literal unless the struct literal is inside
parentheses, brackets or braces.

```tally
if p == Point { x: 0.0, y: 0.0 } { ... }       // error: `{` starts the if-block
if p == (Point { x: 0.0, y: 0.0 }) { ... }     // fine
let q = .Rect { w: 2.0, h: 3.0 };              // inferred-variant struct literal
```
*From: D10, D2.*

**2.10.4** In a `Postfix`, `.` followed by `INT_LIT` is a tuple field access;
the lexer's greedy float rule (1.4.2) does not apply after `.`, so `t.0.1` is
the field `1` of the field `0` of `t`.

```tally
let pair = ((1, 2), 3);
let two = pair.0.1;
```
*From: D2.*

**2.10.5** In expressions, `a.b` is always parsed as a field access. Name
resolution (3.4.2) reinterprets a chain of field accesses whose leftmost name
denotes a module, type or enum as a path to an item. This is the only place
where a parse node's meaning depends on name resolution; its shape never does.

```tally
let f = std.fs.File.open("a.txt");   // parsed as fields, resolved as a path
let n = config.port;                 // `config` is a local: a real field access
```
*From: D13, D18.*

**2.10.6** A `JumpExpr` or `ClosureExpr` extends as far to the right as
possible. `return a + b` returns `a + b`; `|x| x + 1` has body `x + 1`.
`break` and `return` take an operand only if the next token can begin an
expression.

```tally
let f = |x| x * 2 + 1;     // body is x * 2 + 1
if done { return; }        // no operand: `;` cannot start an expression
```
*From: D10, D54.*
