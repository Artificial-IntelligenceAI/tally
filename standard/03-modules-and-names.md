# 3. Modules, items and name resolution

## 3.1 Modules

**3.1.1** A *module* is the contents of one source file. A module contains
imports followed by items. Modules have no other form: there is no inline
module declaration.

```tally
// file: src/geometry.tly  -> module `shapes.geometry` in package `shapes`
import core.math;
pub struct Point { pub x: f64, pub y: f64 }
pub fn dist(a: Point, b: Point) -> f64 may panic { math.sqrt(sq(a.x - b.x) + sq(a.y - b.y)) }
fn sq(v: f64) -> f64 { v * v }
```
*From: D18.*

**3.1.2** Every module has a *module path*, a sequence of identifiers joined by
`.`. For a file in a package's source root (18.3.1), the path is the package's
name followed by the file's directory components and its name without `.tly`.
The file `src/lib.tly` is the package's *root module* and its path is the
package name alone. A directory and a file may share a name; `src/net.tly`
(module `p.net`) and `src/net/tcp.tly` (module `p.net.tcp`) are distinct
modules, and neither contains the other.

```text
package "shapes":
  src/lib.tly            -> shapes
  src/geometry.tly       -> shapes.geometry
  src/render/svg.tly     -> shapes.render.svg
  src/render.tly         -> shapes.render       (unrelated to shapes.render.svg)
```
*From: D18.*

**3.1.3** A module path names a module only; the modules of a package do not
form a tree in which a parent sees its children. To use another module,
including one in the same package, a module must import it (3.3).

```tally
// in shapes.render.svg:
import shapes.geometry;            // required, even within the same package
fn draw(p: geometry.Point) { ... }
```
*From: D18, D2.*

## 3.2 Items and visibility

**3.2.1** An *item* is a declaration at module level: a function, struct,
enum, union, trait, implementation, type alias, constant, static, effect,
extern block or test. Items may be declared in any order and may refer to
items declared later in the same module.

```tally
fn a() -> u32 { b() + 1 }   // b is declared below: allowed
fn b() -> u32 { 41 }
```
*From: D18.*

**3.2.2** An item without `pub` is *package-visible*: it can be named from any
module of the same package. An item with `pub` is *public*: it can be named
from any package that depends on this one. `impl` items take no `pub`; their
members do. Trait members, enum variants, and the fields of a union take the
visibility of their trait, enum or union. Struct fields are package-visible
unless marked `pub`.

```tally
pub struct Conn {
    pub id: u64,         // public field
    fd: i32,             // package-visible field
}
impl Conn {
    pub fn close(self) may sys { ... }   // public method
    fn raw(&self) -> i32 { self.fd }     // package-visible method
}
```
*From: D18.*

**3.2.3** A public item's interface must not mention a package-visible item.
The interface of a function is its parameter types, return type, error type,
generic bounds, and effect clause; of a struct, its `pub` fields' types; of an
enum, every variant's types; of a trait, every member's signature; of a type
alias, its definition; of a constant or static, its type; of a public member
of an `impl`, the implemented trait and the implementing type as well. A
violation is an error.

```tally
struct Secret;
pub fn leak() -> Secret { Secret }       // error: public function returns package-visible type
pub effect io_ring;
effect internal;
pub fn submit() may internal { ... }     // error: public signature names package-visible effect
```
*From: D18, D29.*

**3.2.4** Two items in one module with the same name are an error, except
that any number of `impl` items and `test` items may appear (they have no
name). An item named like an import in the same module is an error.

```tally
import std.io;
fn io() {}         // error: `io` is already imported
fn run() {}
struct run;        // error: `run` is already declared
```
*From: D18.*

## 3.3 Imports

**3.3.1** The first identifier of an import path must be one of: `core`,
`std`, the name of the current package, or the local name of a dependency
declared in the package manifest (18.4.1). Otherwise the import is an
error. The remaining identifiers name a module, then optionally an item
within that module.

```tally
import core.mem;              // core library module
import std.net.TcpStream;     // item TcpStream of module std.net
import httpd.router;          // module of the current package `httpd`
import json;                  // root module of dependency `json`
import ghost.thing;           // error: `ghost` is not core, std, this package or a dependency
```
*From: D18, D43.*

**3.3.2** `import P;` binds the last identifier of `P` in the importing
module's scope to the module or item that `P` names. `import P as N;` binds
`N` instead. `import P.{a, b as c, self};` is equivalent to separate imports
of `P.a`, `P.b as c`, and `P` (for `self`). There are no wildcard imports.

```tally
import std.collections.{HashMap, HashSet as Set, self};
// binds HashMap, Set, and `collections` (the module itself)
let m: HashMap[String, u32] = HashMap.new();
let s: collections.BTreeMap[u8, u8] = ...;
```
*From: D18, D2.*

**3.3.3** An import may name an item only if that item is visible from the
importing module (3.2.2). Importing a module is always allowed for modules of
the current package, and for modules of other packages if the module contains
at least one public item.

```tally
// package b depends on package a; a.internal has no pub items
import a.internal;       // error: module a.internal has no public items
import a.util.helper;    // error if helper is not pub
```
*From: D18.*

**3.3.4** `pub import P;` additionally makes the bound name a public item of
the importing module, called a *re-export*: other packages may name it through
this module. Re-exporting a package-visible item is an error.

```tally
// file src/lib.tly of package `net2`
pub import net2.tcp.Stream;      // users may now write net2.Stream
```
*From: D18.*

**3.3.5** Imports among the modules of a package may be cyclic. Import
resolution does not depend on the order of modules or imports.

```tally
// a.tly                    // b.tly
import p.b;                 import p.a;
pub fn f() { b.g(); }       pub fn g() {}
                            pub fn h() { a.f(); }
```
*From: D18.*

## 3.4 Scopes and name resolution

**3.4.1** Names are resolved in nested scopes, innermost first:

1. block scopes, holding local bindings introduced by `let`, `var`, patterns
   and closure parameters, each visible from the statement after its
   introduction to the end of its block;
2. the function scope, holding parameters;
3. generic parameter scopes of the function, and of the enclosing `impl` or
   trait, plus `Self` inside an `impl` or trait;
4. the module scope, holding the module's items and imports;
5. the predeclared scope (1.3.4).

A name not found in any scope is an error. Each scope has one namespace
(D18): a type, a value and a module may not share a name in one scope.

```tally
const N: usize = 4;
fn f[T](x: T) -> usize {
    let n = N;          // module scope
    let N = 2;          // local shadows the constant from here on
    n + N
}
```
*From: D18, D55.*

**3.4.2** In an expression, a chain `a.b.c...` is resolved left to right. If
`a` resolves to a local binding, parameter or `self`, every `.name` is a field
access or method call (5.6). If `a` resolves to a module, `.b` names an item
of that module visible from here; if that item is itself a module, resolution
continues the same way. If a prefix resolves to a type, the next `.name` names
an associated item of the type (3.4.3) and further components are field
accesses or method calls on the result. A chain in a type position is resolved
the same way except that it must end at a type, trait or effect.

```tally
import std;
let f = std.fs.File.open("x");   // module std, module fs, type File, assoc fn open
let len = f.path.len();          // local f: field path, method len
```
*From: D18, D13.*

**3.4.3** The *associated items* of a type are found in this order: enum
variants (if the type is an enum); items of inherent `impl`s of the type
(7.3.1); items of traits implemented by the type whose trait is in scope
(imported, or declared in the current module, or in the prelude). If the name
is found at an earlier step, later steps are not searched. If more than one
trait supplies the name at the last step, the reference is an error and must
be written in qualified form `Trait.name`.

```tally
let v = Vec.[u8].new();          // inherent associated function
let d = Point.default();         // from trait Default, if Point implements it
let x = Clone.clone(&p);         // qualified form: always unambiguous
```
*From: D27, D2.*

**3.4.4** A local binding may shadow any name from an outer scope, including
an earlier binding in the same block. A shadowed name cannot be reached by its
unqualified name for the rest of the shadowing binding's scope.

```tally
let s = read_line()?;
let s = s.trim();        // shadows the earlier `s`
```
*From: D55.*

**3.4.5** Inside a type, trait or `impl`, `Self` names the type being defined
or implemented (with its generic parameters as declared). Inside a trait,
`Self` is the implementing type. Associated types and effects of `Self` are
written `Self.Name`.

```tally
impl[T] Stack[T] {
    pub fn new() -> Self { Self { items: Vec.new() } }
}
trait Shape { fn scaled(&self, k: f64) -> Self; }
```
*From: D27.*

## 3.5 Constants and statics

**3.5.1** `const NAME: T = e;` declares a constant. `e` is evaluated at compile
time (chapter 16). Each use of the constant is a copy of its value; a constant
has no address of its own. A constant's type must be `Copy` or the use must be
the whole initialiser of a `static` (to build static data such as tables).

```tally
const PAGE: usize = 4096;
const MASKS: [4]u8 = [0x1, 0x3, 0x7, 0xf];
let m = MASKS[2];            // copies the array
```
*From: D40, D2.*

**3.5.2** `static NAME: T = e;` declares a value with a single fixed address
for the whole run of the program, initialised before the program starts with
the value of `e` computed at compile time. There is no initialisation code
run at start-up for statics. The type of a non-`var` static must implement
`Sync` (15.2.2). `&NAME` yields a `&T` that may be used anywhere.

```tally
static GREETING: &str = "hi";
static COUNT: Atomic[u64] = Atomic.new(0);     // Atomic[u64] is Sync
fn bump() { COUNT.fetch_add(1, .Relaxed); }
```
*From: D2, D26, D40.*

**3.5.3** `static var NAME: T = e;` declares a mutable static. Any read,
write, or reference to a `static var` is an unsafe operation (13.1.2). Data
races on a `static var` are undefined behaviour.

```tally
static var BOOT_TICKS: u64 = 0;
fn record(t: u64) { unsafe { BOOT_TICKS = t; } }
fn bad() -> u64 { BOOT_TICKS }       // error: use of static var outside unsafe
```
*From: D36, D2.*

**3.5.4** An extern static (14.1.4) is declared in an `extern` block, has no
initialiser, and refers to a symbol defined elsewhere. All accesses to it are
unsafe.

```tally
extern "C" { static var errno: i32; }
fn last_error() -> i32 { unsafe { errno } }
```
*From: D38.*
