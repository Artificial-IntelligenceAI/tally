# 20. The standard library

`std` is available on hosted targets (any operating system other than
`none`). It contains everything in `core` and the interfaces to the operating
system. Declarations follow the conventions of chapter 19. Every function
listed here that performs a system call has the `sys` effect; every waiting
primitive (20.1.3) has `block`.

## 20.1 Organisation

**20.1.1** For every module `core.m`, `std.m` is the same module (a
re-export). In addition `std` has the modules `std.io`, `std.fs`, `std.net`,
`std.process`, `std.time` (extending `core.time`), `std.thread`, `std.fiber`,
`std.sync`, `std.signal`, `std.alloc` (extending `core.alloc`) and `std.os`.

```tally
import std.collections.HashMap;     // the same item as core.collections.HashMap
import std.net.TcpListener;
```
*From: D46.*

**20.1.2** Paths in `std` APIs are `&str` (UTF-8). On Unix targets, file names
that are not valid UTF-8 can be handled with the byte-path functions of
`std.os.unix.fs` (20.10.2); on Windows, `std` converts between UTF-8 and the
system's UTF-16 and reports unpaired surrogates as `io.ErrorKind.InvalidData`.

```tally
let f = fs.File.open("/etc/hosts")?;
```
*From: D46.*

**20.1.3** The *waiting primitives* are the functions listed below as
"waiting". Each is a primitive source of `block` (10.7.1) and is fiber-aware
(15.4.2): on a fiber it suspends only the fiber. A waiting primitive that is
interrupted by a signal retries automatically (it never reports `EINTR`).
The waiting primitives are: reading from or writing to a `File`, `Stdin`,
`Stdout`, `Stderr`, `TcpStream`, `UdpSocket`, `ChildStdin`, `ChildStdout` or
`ChildStderr` (when the operation must wait); `TcpListener.accept`;
`TcpStream.connect`; `net.resolve`; `Child.wait`; `Command.output`;
`thread.sleep`; `JoinHandle.join`; `Fiber.join`; `fiber.yield_now`;
`Mutex.lock`, `RwLock.read`, `RwLock.write`, `Condvar.wait`,
`Condvar.wait_timeout`, `OnceLock.get_or_init` (while another context
initialises), `Barrier.wait`; `Sender.send` and `Receiver.recv`;
`signal.wait`; and `File.sync_all`.

```tally
let line = io.stdin().read_line(&mut buf)?;      // waiting: suspends the fiber if on one
```
*From: D3, D34.*

## 20.2 Input and output

**20.2.1** `std.io.Error` holds an `ErrorKind` and, when the error came from
the operating system, its error code:

```tally
@derive(Copy, Eq, Debug) pub struct Error { ... }
impl Error {
    pub fn kind(&self) -> ErrorKind;
    pub fn os_code(&self) -> ?i32;
    pub fn new(kind: ErrorKind) -> Error;
    pub fn from_os(code: i32) -> Error;               // kind derived from the code
}
@non_exhaustive @derive(Copy, Eq, Hash, Debug)
pub enum ErrorKind {
    NotFound, PermissionDenied, AlreadyExists, NotADirectory, IsADirectory, DirectoryNotEmpty,
    WouldBlock, BrokenPipe, ConnectionRefused, ConnectionReset, ConnectionAborted, NotConnected,
    AddrInUse, AddrNotAvailable, TimedOut, InvalidInput, InvalidData, UnexpectedEof, WriteZero,
    StorageFull, Unsupported, OutOfMemory, Other,
}
```

`Error` implements `Display` (a description of the kind, plus the operating
system's message for the code if any) and `core.error.Error`. It never
allocates.

```tally
match fs.remove_file(p) { .Err(e) if e.kind() == .NotFound => {}, r => r? }
```
*From: D20.*

**20.2.2** The `Read` and `Write` traits:

```tally
pub trait Read {
    effect Io;
    fn read(&mut self, buf: &mut []u8) -> usize ! Error may Self.Io;      // 0 means end of input
    fn read_exact(&mut self, buf: &mut []u8) ! Error may Self.Io { ... }   // UnexpectedEof if input ends
    fn read_to_end[A: Allocator](&mut self, v: &mut Vec[u8, A]) -> usize ! Error may Self.Io | A { ... }
    fn read_to_string[A: Allocator](&mut self, s: &mut String[A]) -> usize ! Error may Self.Io | A { ... }
}
pub trait Write {
    effect Io;
    fn write(&mut self, buf: &[]u8) -> usize ! Error may Self.Io;
    fn flush(&mut self) ! Error may Self.Io;
    fn write_all(&mut self, buf: &[]u8) ! Error may Self.Io { ... }        // WriteZero if write returns 0
}
```

`read_to_end` and `read_to_string` report allocation failure as an `Error`
with kind `OutOfMemory`. `Vec[u8, A]` implements `Write` (effect `A`); `&[]u8`
implements `Read` (effect `none`). Every `Write` type with `Io` within
`E` can be used as a `fmt.Sink` through `io.sink(&mut w)`.

```tally
var data: Vec[u8] = Vec.new();
_ = file.read_to_end(&mut data)?;
```
*From: D30, D22.*

**20.2.3** `BufReader[R: Read]` and `BufWriter[W: Write]` add an 8192-byte
buffer: `BufReader.new(r) -> BufReader[R] ! AllocError may alloc`,
`read_line[A](&mut self, s: &mut String[A]) -> usize ! Error` (appends up to
and including the next `\n`; 0 at end of input), `lines(self) -> Lines[R]`
(an iterator of `String ! Error`, line ends removed); `BufWriter.new(w)`,
which flushes on `flush` and when dropped (ignoring errors; its drop effect
is `W.Io`).

```tally
var r = io.BufReader.new(fs.File.open(path)?)?;
for line in r.lines() { let l = line?; ... }
```
*From: D46.*

**20.2.4** `stdin()`, `stdout()` and `stderr()` return handles to the
process's standard streams, which implement `Read` or `Write` with
`Io = sys | block`. Standard output is buffered: its buffer (at least 4096
bytes) is flushed when a line end is written, when it is full, on `flush`,
and at normal process exit (17.1.3). Standard error is unbuffered. Each
stream has a lock so that one call writes its data contiguously.
`Stdin.read_line` reads a line as in 20.2.3.

```tally
var out = io.stdout();
out.write_all(b"ready\n")?;
```
*From: D46.*

**20.2.5** `print[X: Display](..args: X) may sys | block | panic` writes the
arguments' `Display` output to standard output; `println` also writes `\n`;
`eprint` and `eprintln` write to standard error. They ignore write errors
(including a closed pipe). To detect errors, write to `stdout()` directly.

```tally
io.println("listening on ", addr);
```
*From: D47.*

## 20.3 Files

**20.3.1** `std.fs`:

```tally
pub struct File { ... }              // owns a file descriptor/handle; Drop closes it (effect sys)
impl File {
    pub fn open(path: &str) -> File ! io.Error may sys | block;                // read-only
    pub fn create(path: &str) -> File ! io.Error may sys | block;              // write, create, truncate
    pub fn open_with(path: &str, o: &OpenOptions) -> File ! io.Error may sys | block;
    pub fn metadata(&self) -> Metadata ! io.Error may sys | block;
    pub fn set_len(&mut self, n: u64) ! io.Error may sys | block;
    pub fn sync_all(&mut self) ! io.Error may sys | block;                     // waiting
    pub fn seek(&mut self, to: SeekFrom) -> u64 ! io.Error may sys;
}
impl io.Read for File { effect Io = sys | block; ... }
impl io.Write for File { effect Io = sys | block; ... }
@derive(Copy, Default) pub struct OpenOptions { pub read: bool, pub write: bool, pub append: bool,
                                               pub create: bool, pub create_new: bool, pub truncate: bool,
                                               pub mode: u32 }      // mode: Unix permission bits, default 0o666
pub enum SeekFrom { Start(u64), End(i64), Current(i64) }
pub struct Metadata { ... }
impl Metadata { pub fn len(&self) -> u64; pub fn is_file(&self) -> bool; pub fn is_dir(&self) -> bool;
                pub fn is_symlink(&self) -> bool; pub fn modified(&self) -> time.SystemTime; }
pub fn read(path: &str) -> Vec[u8] ! io.Error may sys | block | alloc;
pub fn read_to_string(path: &str) -> String ! io.Error may sys | block | alloc;   // InvalidData if not UTF-8
pub fn write(path: &str, data: &[]u8) ! io.Error may sys | block;
pub fn metadata(path: &str) -> Metadata ! io.Error may sys | block;              // follows symlinks
pub fn remove_file(path: &str) ! io.Error may sys | block;
pub fn rename(from: &str, to: &str) ! io.Error may sys | block;
pub fn create_dir(path: &str) ! io.Error may sys | block;
pub fn create_dir_all(path: &str) ! io.Error may sys | block | alloc;
pub fn remove_dir(path: &str) ! io.Error may sys | block;
pub fn read_dir(path: &str) -> ReadDir ! io.Error may sys | block;
```

`ReadDir` is an iterator (`Next = sys | block | alloc`) of `DirEntry ! io.Error`;
`DirEntry` has `name(&self) -> &str from self` and `metadata()`. Entries `.`
and `..` are not returned; the order is unspecified.

```tally
for e in fs.read_dir("logs")? { let e = e?; if e.name().ends_with(".log") { ... } }
```
*From: D46.*

## 20.4 Networking

**20.4.1** `std.net`:

```tally
@derive(Copy, Eq, Ord, Hash, Debug) pub enum IpAddr { V4([4]u8), V6([16]u8) }
@derive(Copy, Eq, Ord, Hash, Debug) pub struct SocketAddr { pub ip: IpAddr, pub port: u16 }
impl FromStr for IpAddr { ... }        // "127.0.0.1", "::1"
impl FromStr for SocketAddr { ... }    // "127.0.0.1:80", "[::1]:80"
pub fn resolve(host: &str, port: u16) -> Vec[SocketAddr] ! io.Error may sys | block | alloc;   // waiting
pub struct TcpListener { ... }
impl TcpListener {
    pub fn bind(a: SocketAddr) -> TcpListener ! io.Error may sys;     // SO_REUSEADDR set on Unix
    pub fn accept(&self) -> (TcpStream, SocketAddr) ! io.Error may sys | block;   // waiting
    pub fn local_addr(&self) -> SocketAddr ! io.Error may sys;
}
pub struct TcpStream { ... }
impl TcpStream {
    pub fn connect(a: SocketAddr) -> TcpStream ! io.Error may sys | block;          // waiting
    pub fn peer_addr(&self) -> SocketAddr ! io.Error may sys;
    pub fn shutdown(&self, how: Shutdown) ! io.Error may sys;
    pub fn set_nodelay(&self, on: bool) ! io.Error may sys;
    pub fn set_timeout(&self, read: ?Duration, write: ?Duration) ! io.Error may sys;   // TimedOut errors
}
impl io.Read for TcpStream { effect Io = sys | block; ... }
impl io.Write for TcpStream { effect Io = sys | block; ... }
pub struct UdpSocket { ... }
impl UdpSocket {
    pub fn bind(a: SocketAddr) -> UdpSocket ! io.Error may sys;
    pub fn send_to(&self, b: &[]u8, to: SocketAddr) -> usize ! io.Error may sys | block;
    pub fn recv_from(&self, b: &mut []u8) -> (usize, SocketAddr) ! io.Error may sys | block;
}
pub enum Shutdown { Read, Write, Both }
```

Sockets are closed when dropped (effect `sys`).

```tally
let l = TcpListener.bind("0.0.0.0:8080".parse()?)?;
loop { let (c, peer) = l.accept()?; handle(c, peer); }
```
*From: D46.*

## 20.5 Processes and environment

**20.5.1** `std.process`:

```tally
pub fn args() -> Args may sys;          // Iterator[Item = &str] (region static); invalid UTF-8 replaced by U+FFFD
pub fn env(name: &str) -> ?String ! AllocError may sys | alloc;
pub unsafe fn set_env(name: &str, value: &str) ! io.Error may sys | alloc;   // unsafe: races with other threads reading the environment
pub fn exit(code: u8) -> never may sys;  // 17.1.4
pub fn id() -> u32 may sys;
pub struct Command { ... }
impl Command {
    pub fn new(program: &str) -> Command ! AllocError may alloc;
    pub fn arg(&mut self, a: &str) -> &mut Command ! AllocError from self may alloc;
    pub fn env(&mut self, k: &str, v: &str) -> &mut Command ! AllocError from self may alloc;
    pub fn current_dir(&mut self, d: &str) -> &mut Command ! AllocError from self may alloc;
    pub fn stdin(&mut self, s: Stdio) -> &mut Command from self;    // also stdout, stderr
    pub fn spawn(&mut self) -> Child ! io.Error may sys | alloc;
    pub fn output(&mut self) -> Output ! io.Error may sys | block | alloc;   // waiting
}
pub enum Stdio { Inherit, Null, Piped }
pub struct Child { pub stdin: ?ChildStdin, pub stdout: ?ChildStdout, pub stderr: ?ChildStderr, ... }
impl Child {
    pub fn wait(&mut self) -> ExitStatus ! io.Error may sys | block;     // waiting
    pub fn kill(&mut self) ! io.Error may sys;
    pub fn id(&self) -> u32;
}
pub struct Output { pub status: ExitStatus, pub stdout: Vec[u8], pub stderr: Vec[u8] }
@derive(Copy, Eq, Debug) pub struct ExitStatus { ... }
impl ExitStatus { pub fn success(&self) -> bool; pub fn code(&self) -> ?i32; pub fn signal(&self) -> ?i32; }
```

```tally
let out = Command.new("git")?.arg("rev-parse")?.arg("HEAD")?.output()?;
```
*From: D46, D61.*

## 20.6 Time

**20.6.1** `std.time` adds:

```tally
@derive(Copy, Eq, Ord, Hash, Debug) pub struct Instant { ... }    // monotonic clock
impl Instant {
    pub fn now() -> Instant may sys;
    pub fn duration_since(&self, earlier: Instant) -> Duration;    // ZERO if earlier is later
    pub fn elapsed(&self) -> Duration may sys;
    pub fn checked_add(&self, d: Duration) -> ?Instant;
}
@derive(Copy, Eq, Ord, Hash, Debug) pub struct SystemTime { ... }  // wall clock
impl SystemTime {
    pub const UNIX_EPOCH: SystemTime;
    pub fn now() -> SystemTime may sys;
    pub fn duration_since(&self, earlier: SystemTime) -> Duration ! Duration;   // .Err(how far before)
}
```

```tally
let t0 = Instant.now();
work();
io.println("took ", t0.elapsed().as_millis(), " ms");
```
*From: D46.*

## 20.7 Threads

**20.7.1** `std.thread`:

```tally
pub fn spawn[T: Send, F: FnOnce() -> T + Send](f: F) -> JoinHandle[T] ! SpawnError may alloc | sys | F;
pub fn scope[R, F: FnOnce(&Scope) -> R](f: F) -> R may F | block | sys;
impl Scope { pub fn spawn[T: Send, F: FnOnce() -> T + Send](&self, f: F) -> ScopedJoinHandle[T] ! SpawnError from self may alloc | sys | F; }
pub struct Builder { pub name: ?&str, pub stack_size: ?usize }
impl Builder { pub fn spawn[T: Send, F: FnOnce() -> T + Send](self, f: F) -> JoinHandle[T] ! SpawnError may alloc | sys | F; }
impl[T] JoinHandle[T] { pub fn join(self) -> T may sys | block; }            // waiting
impl[T] ScopedJoinHandle[T] { pub fn join(self) -> T may sys | block; }
pub fn sleep(d: Duration) may sys | block;                                    // waiting
pub fn current_id() -> u64 may sys;
pub fn available_parallelism() -> usize may sys;
@derive(Copy, Eq, Debug) pub enum SpawnError { from Alloc(AllocError), Os(i32) }
```

The new thread's stack is `stack.bound(f)` plus `CONTEXT_OVERHEAD`, rounded up
to the page size, if the bound is known and `Builder.stack_size` is not
given; otherwise `stack_size`, or 2 MiB by default, with a guard page
(11.6.2). The thread starts with a signal stack (20.9.2) and fresh
context-local statics. `scope` returns after every thread spawned on its
`Scope` has finished; such threads may borrow from outside the scope.
Dropping a `JoinHandle` detaches the thread. Because panics end the process,
`join` has no error case.

```tally
var results: [4]u64 = [0; 4];
thread.scope(|s| {
    for (i, r) in results.iter_mut().enumerate() { _ = s.spawn(move || *r = crunch(i)); }
});
```
*From: D34, D33.*

## 20.8 Fibers

**20.8.1** `std.fiber`:

```tally
pub fn run[R, F: FnOnce(&Scope) -> R](f: F) -> R ! SpawnError may F | alloc | sys | block;
impl Scope {
    pub fn spawn[T, F: FnOnce() -> T](&self, f: F) -> Fiber[T] ! SpawnError from self may alloc | sys | F;
    pub fn spawn_sized[T, F: FnOnce() -> T](&self, size: usize, f: F) -> Fiber[T] ! SpawnError from self may alloc | sys | F;
}
impl[T] Fiber[T] { pub fn join(self) -> T may sys | block; }     // waiting
pub fn yield_now() may sys | block;                              // waiting
pub fn in_fiber() -> bool may sys;
```

`run` creates a scheduler on the current thread, runs `f` as the first fiber,
and returns `f`'s result after every fiber spawned on its `Scope` (and,
recursively, their scopes) has finished. Fibers may borrow anything that
outlives the call to `run`. A `Scope` may be passed to fibers by reference so
they can spawn more fibers.

```tally
fn serve_all(addr: SocketAddr) ! AppError may sys | block | alloc | panic {
    fiber.run(|s| -> () ! AppError {
        let l = TcpListener.bind(addr)?;
        loop {
            let (conn, _) = l.accept()?;          // suspends this fiber, not the thread
            _ = s.spawn(move || serve(conn))?;    // AppError has `from Spawn(SpawnError)`
        }
    })?
}
```
*From: D34.*

**20.8.2** `Scope.spawn(f)` allocates a stack of exactly
`stack.bound_required(&f) + CONTEXT_OVERHEAD` bytes (rounded up to the
target's stack alignment) with no guard page, so linking fails if `f` is
unbounded. `spawn_sized(size, f)` instead allocates `size` bytes plus a guard
page and is for unbounded or foreign-calling code; overflow then panics
(11.6.2). The fiber's stack is freed when it finishes.

```tally
_ = s.spawn(move || handle(conn))?;          // stack = bound(handle closure) + overhead, e.g. 3.2 KiB
_ = s.spawn_sized(1 << 20, move || legacy(conn))?;   // calls unbounded C code
```
*From: D33, D34.*

**20.8.3** The scheduler runs fibers from a FIFO run queue. A fiber runs
until it reaches a suspension point (15.1.2); when no fiber is runnable, the
scheduler waits (blocking its thread) for any registered I/O readiness, timer
or wake-up, using the operating system's readiness mechanism (implementation-
defined: for example `epoll`, `kqueue`, I/O completion ports or `io_uring`). A
fiber woken by another thread (through a channel or lock) is placed on its own
scheduler's run queue. Fibers never move to another thread.

```tally
// two fibers each printing in a loop alternate at each println (a waiting primitive)
```
*From: D34.*

## 20.9 Synchronisation and signals

**20.9.1** `std.sync` (all fiber-aware; none poisons, since panics end the
process):

```tally
pub struct Mutex[T] { ... }                            // Send + Sync if T: Send
impl[T] Mutex[T] {
    pub fn new(v: T) -> Self;                          // usable in static initialisers
    pub fn lock(&self) -> MutexGuard[T] from self may sys | block;    // waiting
    pub fn try_lock(&self) -> ?MutexGuard[T] from self may sys;
    pub fn get_mut(&mut self) -> &mut T from self;
    pub fn into_inner(self) -> T;
}
pub struct RwLock[T] { ... }                            // read() -> ReadGuard, write() -> WriteGuard; waiting
pub struct Condvar { ... }
impl Condvar {
    pub fn new() -> Self;
    pub fn wait[T](&self, g: MutexGuard[T]) -> MutexGuard[T] may sys | block;         // may wake spuriously
    pub fn wait_timeout[T](&self, g: MutexGuard[T], d: Duration) -> (MutexGuard[T], bool) may sys | block;
    pub fn notify_one(&self) may sys; pub fn notify_all(&self) may sys;
}
pub struct OnceLock[T] { ... }                          // like OnceCell, Sync
pub struct Barrier { ... }                              // new(n), wait(): waiting
pub fn channel[T: Send](capacity: usize) -> (Sender[T], Receiver[T]) ! AllocError may alloc | panic;  // panics if capacity is 0
impl[T] Sender[T] {
    pub fn send(&self, v: T) ! SendError[T] may sys | block;      // waits while full; error returns v if all receivers are gone
    pub fn try_send(&self, v: T) ! TrySendError[T] may sys;
}
impl[T] Receiver[T] {
    pub fn recv(&self) -> T ! RecvError may sys | block;          // error once empty and all senders gone
    pub fn try_recv(&self) -> T ! TryRecvError may sys;
}
```

Guards are view structs that deref to `T` and release the lock when dropped
(effect `sys`). `Sender` and `Receiver` implement `Clone` (effect `none`):
channels are multi-producer, multi-consumer, and FIFO per sender.

```tally
static COUNTS: Mutex[[16]u64] = Mutex.new([0; 16]);
COUNTS.lock()[slot] += 1;
```
*From: D26, D34.*

**20.9.2** `std.signal` (Unix targets):

```tally
@non_exhaustive @derive(Copy, Eq, Debug) pub enum Signal { Int, Term, Hup, Usr1, Usr2, Chld, Pipe, Alrm, Winch }
pub fn set_handler(s: Signal, h: fn(Signal) may sys | panic) ! io.Error may sys;
pub fn wait(set: &[]Signal) -> Signal ! io.Error may sys | block;   // waiting; blocks the signals in set first
```

A handler set with `set_handler` runs asynchronously on the interrupted
thread's signal stack (a separate stack of at least
`max(stack.bound of every installed handler) + CONTEXT_OVERHEAD`, set up by
`std` for every thread), and its type limits it to effects within
`sys | panic`, so it cannot allocate or wait, which makes it
async-signal-safe as far as `std` is concerned.

```tally
static STOP: Atomic[bool] = Atomic.new(false);
fn on_term(_s: Signal) may sys { STOP.store(true, .Release); }
signal.set_handler(.Term, on_term)?;
```
*From: D3, D33.*

## 20.10 Allocation and platform modules

**20.10.1** `std.alloc.System` is a zero-sized type implementing
`GlobalAlloc` with `Use = sys | block`, using the platform's allocator or an
implementation-provided one (implementation-defined, documented). It is the
global allocator of every program that links `std` and does not declare its
own.

```tally
@global_allocator static A: std.alloc.System = std.alloc.System;   // the default, made explicit
```
*From: D32.*

**20.10.2** `std.os.unix`, `std.os.linux`, `std.os.macos` and
`std.os.windows` are reserved for platform-specific interfaces, available only
with the matching `@cfg`. Their contents are implementation-defined, except
that `std.os.unix` must provide `trait AsRawFd { fn as_raw_fd(&self) -> i32; }`
implemented by `File`, `TcpStream`, `TcpListener`, `UdpSocket` and the
standard streams, `unsafe fn File.from_raw_fd(fd: i32) -> File`, and
`std.os.unix.fs` with byte-path versions of the `std.fs` functions
(`open_bytes(path: &[]u8)`, …).

```tally
@cfg(family("unix")) import std.os.unix.AsRawFd;
let fd = sock.as_raw_fd();
```
*From: D46, D38.*
