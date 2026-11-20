# 7. Traits and generics

## 7.1 Traits

**7.1.1** A trait declares a set of *members* that implementing types must
provide: methods, associated types, associated effects and associated
constants. A method member may have a default body, which implementations
inherit unless they override it. An associated type, effect or constant may
have a default.

```tally
pub trait Shape {
    const SIDES: u32;
    type Unit = f64;
    effect Measure = none;
    fn area(&self) -> Self.Unit may Self.Measure;
    fn describe(&self) -> &str { "a shape" }      // default body
}
```
*From: D27, D30.*

**7.1.2** A trait method's first parameter may be `self`, `&self` or
`&mut self` (a *method*), or the method may have no `self` parameter (an
*associated function*). In a trait, `Self` denotes the implementing type.

```tally
pub trait Default { fn default() -> Self; }        // associated function
pub trait Clone { effect Clone = none; fn clone(&self) -> Self ! AllocError may Self.Clone; }
```
*From: D27.*

**7.1.3** `trait A: B + C { ... }` declares `B` and `C` as *supertraits* of
`A`: a type may implement `A` only if it implements `B` and `C`, and a bound
`T: A` implies `T: B` and `T: C`. Supertrait relations must not form a cycle.

```tally
pub trait Ord: Eq { fn cmp(&self, other: &Self) -> Ordering; }
fn sort[T: Ord](xs: &mut []T) { ... if a == b { ... } }   // == available through Eq
```
*From: D27.*

**7.1.4** A trait declared `unsafe trait` may be implemented only by an
`unsafe impl`. The trait's documentation states the guarantees an implementer
promises; violating them may cause undefined behaviour in safe code that
relies on them. An `unsafe impl` of a trait that is not unsafe is an error.

```tally
pub unsafe trait Send {}
unsafe impl Send for MyHandle {}     // implementer vouches for thread safety
```
*From: D26, D36.*

## 7.2 Generic parameters

**7.2.1** Functions, structs, enums, unions, traits, type aliases and `impl`
items may declare generic parameters in `[ ]`. A *type parameter* `T` ranges
over sized types (7.2.5) and may have bounds `T: Tr1 + Tr2`. A *const
parameter* `const N: K` ranges over compile-time values of type `K`, which
must be an integer type, `bool` or `char`. An *effect parameter* `effect E`
ranges over effect sets (10.2). Parameters may have defaults, which must
follow all parameters without defaults.

```tally
pub struct Array[T, const N: usize] { items: [N]T }
pub fn run[effect E](f: fn() may E) may E { f() }
pub struct Map[K: Hash + Eq, V, S: BuildHasher = SipBuild] { ... }
```
*From: D27, D30, D13.*

**7.2.2** Within the scope of a type parameter `T`, the only operations on
values of type `T` are those provided by its bounds (including supertraits),
by `where` clauses, and by the operations valid on every sized type (moving,
taking references, dropping). Using any other operation is an error at the
generic definition, independent of how it is instantiated.

```tally
fn dup[T](x: T) -> (T, T) { (x, x) }               // error: T is not Copy
fn dup2[T: Clone](x: T) -> (T, T) ! AllocError may T { (x.clone()?, x) }
```
*From: D27, D2.*

**7.2.3** A `where` clause adds bounds on any type built from the generic
parameters, including associated types. `T.Item` denotes the associated type
`Item` of `T` from the unique bound of `T` that declares `Item`; if more than
one bound declares it, the reference is an error and must be written in a
`where`-bound form `T: Iterator[Item = U]` with a fresh parameter `U`.

```tally
fn print_all[I: Iterator](it: I) may I | I.Item | sys | block | panic where I.Item: Display {
    for x in it { io.println(x); }
}
```
*From: D27.*

**7.2.4** A bound may *bind* associated types and effects of the trait:
`I: Iterator[Item = u8]`, `W: Write[Io = sys | block]`. A bound may also be
followed by an effect clause, `T: Iterator may none`, which requires every
associated effect of that trait for `T` (and the drop effects of `T`) to be
within the given set (10.5.3).

```tally
fn checksum[I: Iterator[Item = u8] may none](bytes: I) -> u32 {
    var s: u32 = 0;
    for b in bytes { s = s +% (b as u32); }
    s
}
```
*From: D27, D30.*

**7.2.5** Every type parameter has an implicit bound `Sized` unless the bound
`?Sized` is written. A `?Sized` parameter `T` may be used only behind pointer
types (`&T`, `&mut T`, `*T`, `*mut T`) and as an argument to other `?Sized`
parameters. In `core`, `Box`, `Rc`, `Arc` and the `Deref` target accept
`?Sized` arguments.

```tally
pub fn show[T: ?Sized + Display](x: &T) may panic { ... }
show("text");          // T = str
show(&5);              // T = i64
```
*From: D59.*

## 7.3 Implementations

**7.3.1** An *inherent impl* `impl[P] Type { ... }` adds associated functions,
methods and constants to `Type`. It must be in the package that declares
`Type`; inherent impls for primitive types, arrays, slices, `str`, tuples,
references and pointers are allowed only in `core`. Two inherent items with
the same name for overlapping instantiations of the same type are an error.

```tally
impl Point {
    pub const ORIGIN: Point = Point { x: 0.0, y: 0.0 };
    pub fn norm(&self) -> f64 may panic { (self.x * self.x + self.y * self.y).sqrt() }
}
impl u32 { fn twice(self) -> u32 { self * 2 } }     // error outside core
```
*From: D27, D18.*

**7.3.2** A *trait impl* `impl[P] Trait[args] for Type { ... }` declares that
`Type` implements `Trait`. It must define every member of the trait that has
no default (except associated effects that 10.4.5 lets it omit), and may
override members that have one; it must not define
anything the trait does not declare. Each method's signature, after
substituting `Self` and the trait's generic arguments, must have the same
parameter and return types as the trait's declaration, a `from` clause naming
at most the sources the trait's clause names (where the trait's return type is
an associated type that the impl binds to a type with elided regions, any
sources 9.5.3 allows), and an effect set within the
trait's declared effects (10.5.1). Trait impl members are as visible as the
trait; they take no `pub`.

```tally
impl Shape for Square {
    const SIDES: u32 = 4;
    effect Measure = panic;
    fn area(&self) -> f64 may panic { self.side * self.side }
}
```
*From: D27, D30.*

**7.3.3** The generic parameters of an impl must all appear in the implemented
type or the trait's arguments. An impl applies to every instantiation of its
parameters that satisfies its bounds.

```tally
impl[T: Clone] Clone for Pair[T] {
    effect Clone = T;
    fn clone(&self) -> Self ! AllocError may T { Pair(self.0.clone()?, self.1.clone()?) }
}
impl[T, U] Marker for Wrap[T] {}      // error: U is not constrained
```
*From: D27.*

## 7.4 Coherence

**7.4.1** (Orphan rule.) An impl of trait `Tr` for type `X` is allowed only
in the package that declares `Tr`, or in the package that declares the
outermost nominal type constructor of `X`. References, pointers, arrays,
slices, tuples, function types, `dyn` types, `Option`, `Result`, primitive
types and type parameters have no declaring package other than `core`.

```tally
// in package `geo`:
impl Display for Point { ... }        // fine: Point is declared in geo
impl Display for Vec[Point] { ... }   // error: neither Display nor Vec is declared in geo
impl[T] MyTrait for T { ... }         // fine if MyTrait is declared in geo
```
*From: D27.*

**7.4.2** (No overlap.) Two impls of the same trait overlap if their headers
(implemented type and trait arguments) can be made equal by some choice of
their generic parameters, ignoring bounds and `where` clauses. Overlapping
impls in one program are an error, reported when compiling the package that
contains the later impl in dependency order (or either impl, if in the same
package).

```tally
impl[T] Describe for T { ... }        // in the package declaring Describe
impl Describe for u8 { ... }          // error: overlaps the blanket impl
impl[T: Copy] Fast for Vec[T] {}
impl[T: Clone] Fast for Vec[T] {}     // error: overlap (bounds are ignored)
```
*From: D27.*

## 7.5 Using traits

**7.5.1** Trait methods are called with method syntax (5.6.3) or in qualified
form `Trait.method(args)`, where the receiver is the first argument. For a
type parameter `T`, `T.name` refers to the associated function, constant or
type `name` supplied by `T`'s bounds. Calls through trait bounds are
resolved statically for each instantiation (7.10.1).

```tally
fn zero[T: Default]() -> T { T.default() }
let s = Display.to_string(&x)?;       // qualified call
```
*From: D27, D58.*

**7.5.2** A default method body is checked once, in the trait, against the
trait's declarations, with `Self` as an abstract type bounded by the trait.
Each impl that does not override it gets a copy specialised to the impl's
type.

```tally
pub trait Greet {
    fn name(&self) -> &str;
    fn hello(&self) may sys | block | panic { io.println("hello, ", self.name()); }
}
```
*From: D27.*

## 7.6 Built-in traits

**7.6.1** `Copy` (a marker trait in `core.marker`) means values of the type
are duplicated bit for bit when used by value, instead of moved. A type may
implement `Copy` only with `@derive(Copy)` and only if all its fields are
`Copy` and it has no `Drop` impl. Integers, floats, `bool`, `char`, `()`,
`never`, shared references, raw pointers, function pointers and function item
types are `Copy`; arrays and tuples are `Copy` if their elements are; `&mut T`
is never `Copy`.

```tally
@derive(Copy, Clone) struct Rgb { r: u8, g: u8, b: u8 }
let c = Rgb { r: 1, g: 2, b: 3 };
let d = c;             // copy
let e = c;             // still fine
```
*From: D23.*

**7.6.2** `Clone` declares `fn clone(&self) -> Self ! AllocError may
Self.Clone` with associated effect `Clone` (default `none`). Cloning returns a
result because cloning a value that owns memory allocates, and allocation can
fail (D22). Every `Copy` type implements `Clone` by copying, never failing.
`@derive(Clone)` generates an impl that clones each field with `?`, with
`Clone` effect equal to the union of the fields' clone effects.

```tally
@derive(Clone) struct Doc { title: String, pages: Vec[Page] }
let d2 = doc.clone()?;   // String.clone and Vec.clone allocate: Doc.Clone = alloc
```
*From: D23, D30.*

**7.6.3** `Drop` declares `fn drop(&mut self) may Self.Drop` with associated
effect `Drop` (default `none`). It may be implemented only for a struct, enum
or union declared in the same package, and the impl must have exactly the
same generic parameters and bounds as the type declaration. A type with a
`Drop` impl cannot be `Copy`. `Drop.drop` cannot be called directly; it is
called by the drop glue (8.5.1). Values cannot be moved out of a type with a
`Drop` impl by destructuring.

```tally
pub struct Fd(i32);
impl Drop for Fd {
    effect Drop = sys;
    fn drop(&mut self) may sys { unsafe { c_close(self.0); } }
}
fd.drop();                 // error: Drop.drop cannot be called; use mem.drop(fd)
```
*From: D23, D25.*

**7.6.4** `Deref` declares `type Target: ?Sized;` and
`fn deref(&self) -> &Self.Target from self`; `DerefMut: Deref` declares
`fn deref_mut(&mut self) -> &mut Self.Target from self`. Implementations of
both must have no effects (`may none`); this is checked like any effect
clause. They are used by `*` (5.6.1), auto-deref (5.6.2, 5.6.3) and deref
coercion (4.11.1). `Box` uses built-in dereference rather than `Deref`.

```tally
impl Deref for String { type Target = str; fn deref(&self) -> &str from self { ... } }
impl Deref for Logged { type Target = u32; fn deref(&self) -> &u32 from self {
    io.println("read");          // error: Deref impls must be effect-free
    &self.v
} }
```
*From: D58, D3.*

**7.6.5** `Sized` (marker) is implemented by every sized type (4.1.2) and
cannot be implemented manually.

```tally
fn size[T](x: &T) -> usize { mem.size_of.[T]() }     // T: Sized implicitly
```
*From: D59.*

**7.6.6** `Eq` (marker, in `core.cmp`) is implemented by exactly the
*comparable* types of 5.5.1: built in for primitive and constructed types
whose parts are `Eq`, and by `@derive(Eq)` for structs and enums. It cannot
be implemented by a hand-written impl. Within generic code, `==` and `!=` are
available on `T` exactly when `T: Eq`. `Hash`, `Ord`, `Default`, `Debug` and
`Display` are ordinary traits in `core` (chapter 19) and may be derived where
16.3.3 says.

```tally
fn contains[T: Eq](xs: &[]T, x: &T) -> bool {
    for y in xs { if y == x { return true; } }
    false
}
impl Eq for Thing {}          // error: Eq can only be derived
```
*From: D17, D27.*

**7.6.7** `Static` (marker, in `core.marker`) is implemented by every type in
which every region that occurs is `static`: types that hold no references,
and types holding only references to static data. It cannot be implemented by
hand. A bound `T: Static` lets generic code store a `T` beyond the current
call (in a static, another thread, a fiber that outlives its creator), and
makes every region of `T` known to outlive every region (9.4.3). A `dyn` type
is `Static` exactly when its region is `static` (9.5.6).

```tally
pub fn spawn[T: Send + Static, F: FnOnce() -> T + Send + Static](f: F) -> JoinHandle[T] ! SpawnError may alloc | sys | F;
let name = String.from("w")?;
thread.spawn(|| print(&name));        // error: the closure borrows `name`, so it is not Static
thread.spawn(move || print(&name));   // fine: name is moved into the closure
```
*From: D24, D66.*

## 7.7 Function traits

**7.7.1** `FnOnce[Args]`, `FnMut[Args]: FnOnce[Args]` and
`Fn[Args]: FnMut[Args]` (in `core.ops`) are implemented by closures (5.8.3),
function items and function pointers. `Args` is a tuple type of the
parameter types. Each has the associated type `Output` and the associated
effect `Call`, and the methods `call_once(self, args: Args)`,
`call_mut(&mut self, args: Args)` and `call(&self, args: Args)`, each returning
`Self.Output` and having effect `Self.Call`. These methods cannot be named
directly; a value is called with `f(args)` (5.7.1).

```tally
fn twice[F: FnMut()](var f: F) may F { f(); f(); }
```
*From: D54, D30.*

**7.7.2** The bound `Fn(A, B) -> R may E` is shorthand for
`Fn[(A, B), Output = R, Call = E]`, and likewise for `FnMut` and `FnOnce`;
one parameter gives `Args = (A,)` and none gives `()`. A missing `-> R` means
`Output = ()`. A missing effect clause leaves `Call`
unbound (generic), so the effects of calling `F` are written `F` (10.2.3).
The same shorthand is allowed after `dyn`, where a missing effect clause means
`may none`.

```tally
fn map_all[T, U, F: Fn(&T) -> U](xs: &[]T, f: F) -> Vec[U] ! AllocError may F | alloc { ... }
let cb: Box[dyn FnMut(u32) may sys] = Box.new(move |n| log(n))?;
```
*From: D54, D31.*

## 7.8 Trait objects

**7.8.1** A trait is *object-safe* if: every method has a receiver `&self` or
`&mut self` (methods and associated functions with `where Self: Sized` are
exempt and unavailable on the object; the provided methods of `Iterator` in
19.10.1 that take `self` by value or have type parameters carry this
clause); `FnOnce.call_once` is the exception: `Box[dyn FnOnce(...)]` is
callable, which moves the boxed closure out and frees the box; no method has type parameters or const
parameters (effect parameters are allowed, 10.5.4); `Self` does not appear in
any method's parameters or return type except as the receiver; the trait has
no associated constants; and its supertraits are object-safe. Only
object-safe traits may appear after `dyn`.

```tally
pub trait Write { effect Io; fn write(&mut self, b: &[]u8) -> usize ! Error may Self.Io; }
let w: &mut dyn Write may sys | block = &mut file;  // fine
let c: &dyn Clone = &x;                              // error: Clone returns Self
```
*From: D27.*

**7.8.2** `dyn Trait[bindings] may E` is an unsized type. Every associated
type of the trait (and its supertraits) must be bound in `bindings`.
Associated effects may be bound in `bindings`; every associated effect not
bound there is bounded by `E`, as are the drop effects of the concrete type.
A missing effect clause means `may none`. Calling a method through the object
has the effect bound for that method's associated effects. Dropping a `Box`
(or other owner) of a trait object has effect `E`. `dyn Trait + Send` and
`+ Sync` may be added to require those markers of the concrete type.

```tally
let it: Box[dyn Iterator[Item = Line] may sys | block] = Box.new(file.lines())?;
for line in it { ... }         // `next` has effect sys | block
let shared: Arc[dyn Logger + Send + Sync] = ...;
```
*From: D27, D30, D31.*

**7.8.3** A pointer to `U` coerces to a pointer to `dyn Trait[bindings] may
E` (4.11.1) only if `U: Trait`, the associated types of `U`'s impl equal the
bindings, each associated effect of `U`'s impl is within its binding (or
within `E` if unbound), and the drop effects of `U` are within `E`.

```tally
let f: File = ...;                                // File.Io = sys | block; drop effect sys
let w: Box[dyn Write may sys] = Box.new(f)?;      // error: block not within sys
let w: Box[dyn Write may sys | block] = Box.new(f)?;   // fine
```
*From: D27, D30.*

**7.8.4** A trait object pointer holds the data address and the address of a
*vtable* for the concrete type. The vtable layout is implementation-defined;
it contains at least the size, alignment and drop glue of the concrete type
and one entry per method. Two vtable pointers for the same type and trait
need not be equal.

```tally
let a: &dyn Display = &5;
// mem.size_of.[&dyn Display]() == 2 * mem.size_of.[usize]()
```
*From: D2, D37.*

## 7.9 Built-in implementations

**7.9.1** `core` provides implementations of its traits for the primitive and
constructed types as listed in chapter 19. Every function item type and every
function pointer type implements `Fn` (unless `unsafe`), `Copy` and `Clone`.

```tally
fn add1(x: u32) -> u32 may panic { x + 1 }
fn apply[F: Fn(u32) -> u32](f: F, x: u32) -> u32 may F { f(x) }
let three = apply(add1, 2);          // the function item implements Fn(u32) -> u32 may panic
```
*From: D27.*

**7.9.2** For every arity from 0 to 16, tuples implement `Copy`, `Clone`,
`Eq`, `Ord` (lexicographic), `Hash`, `Default`, `Debug` and `Display` when all
their element types do. A tuple's `Display` writes each element's `Display`
output in order with nothing between them; this is what rest parameters
(5.7.5) of `print`-like functions rely on. Associated effects of a tuple's
impl are the union of its elements'. User traits cannot be implemented for
all tuple arities at once.

```tally
let t = ("x = ", 4, "\n");
io.print(t);                 // prints "x = 4\n"
```
*From: D47.*

## 7.10 Instantiation

**7.10.1** Each use of a generic function or type with a distinct set of type
and const arguments produces a separate *instance*, compiled separately, in
which bounds are resolved to the concrete impls. Effect arguments do not
produce separate instances: an instance's code is the same for all effect
arguments.

```tally
fn id[T](x: T) -> T { x }
_ = id(1); _ = id(1.0);  // two instances: id[i64], id[f64]
fn run[effect E](f: fn() may E) may E { f() }
run(a); run(b);          // one instance, whatever the effects of a and b
```
*From: D27, D30.*

**7.10.2** If instantiating a program would require infinitely many instances
(polymorphic recursion such as `f[T]` calling `f[Box[T]]`), the program has an
error. An implementation may report this when instance nesting exceeds a
documented depth of at least 128.

```tally
fn grow[T](x: T, n: u32) ! AllocError may alloc | panic { if n > 0 { grow(Box.new(x)?, n - 1)?; } }
// error: requires grow[Box[T]], grow[Box[Box[T]]], ...
```
*From: D27, D2.*
