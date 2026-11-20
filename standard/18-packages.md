# 18. Packages

## 18.1 Packages and targets

**18.1.1** A *package* is a directory containing a manifest file named
`tally.toml` and a source directory `src/`. A package is the unit of
versioning, distribution, effect inference (10.4.4) and visibility (3.2.2). A
package's name, given in its manifest, must be an identifier that is not a
keyword, not `core`, `std`, `bin` or `this`, and not beginning with
`vendor_`.

```text
httpd/
  tally.toml
  tally.lock
  src/lib.tly
  src/router.tly
  src/bin/httpd.tly
```
*From: D43.*

**18.1.2** A package has a *library target* if `src/lib.tly` exists, and one
*binary target* for each file `src/bin/NAME.tly`. It must have at least one
target.

```text
src/lib.tly           -> library target
src/bin/httpd.tly     -> binary "httpd"
src/bin/bench.tly     -> binary "bench"
```
*From: D43, D18.*

## 18.2 The manifest

**18.2.1** `tally.toml` is a TOML 1.0.0 document. Unknown top-level tables
and unknown keys are errors, except keys beginning with `x-`, which are
ignored and reserved for tools.

```toml
[package]
name = "httpd"
version = "1.4.2"
tally = "1"
license = "Apache-2.0 WITH LLVM-exception"
description = "A small HTTP server"
repository = "https://example.org/httpd.git"
x-ci = "nightly"
```
*From: D43.*

**18.2.2** The `[package]` table has these keys: `name` (string, required);
`version` (string, required, a semantic version `MAJOR.MINOR.PATCH` with
optional `-prerelease`, as in Semantic Versioning 2.0.0, without build
metadata); `tally` (string, required, the major version of this standard the
package is written against, `"1"` for this standard, 18.7.1); `license`
(string, an SPDX licence expression); `description`, `repository`,
`homepage` (strings); `authors` (array of strings).

```toml
[package]
name = "json"
version = "0.9.0"
tally = "1"
```
*From: D43, D62.*

**18.2.3** The `[dependencies]` table maps a *dependency key* (the name the
dependent uses in `import`, 3.3.1) to a table describing the dependency:

* `git = "URL"` and `version = "X.Y.Z"`: the package in that git repository,
  at the selected version (18.4); optionally `dir = "path"` if the package is
  in a subdirectory of the repository;
* `path = "relative/dir"`: the package in that local directory (18.4.6).

Either form may add `features = ["f", ...]` and `default-features = false`
(18.5), and `optional = true`, which makes the dependency available only when
a feature names it. The key must be an identifier and must not equal the
package's own name. The dependency package's own name need not equal the key.

```toml
[dependencies]
json = { git = "https://example.org/json.git", version = "0.9.1" }
tls = { git = "https://example.org/libs.git", dir = "tls", version = "2.1.0", optional = true }
util = { path = "../util" }
```
*From: D43.*

**18.2.4** `[dev-dependencies]` has the same form as `[dependencies]`; its
packages are available only to `test` items, `@cfg(test)` code and
`tally test` builds.

```toml
[dev-dependencies]
proptest = { git = "https://example.org/proptest.git", version = "1.0.0" }
```
*From: D43, D49.*

**18.2.5** `[features]` maps feature names to arrays of what they enable:
other feature names of this package, `"dep:KEY"` (enables optional dependency
`KEY`), or `"KEY/FEATURE"` (enables a feature of a dependency). The feature
named `default` is enabled unless the dependent disables default features.

```toml
[features]
default = ["std-io"]
std-io = []
https = ["dep:tls", "tls/aws-lc"]
```
*From: D43, D42.*

**18.2.6** `[native]` describes foreign code to compile and link: `sources`
(array of glob patterns relative to the package directory, of C files `.c` and
assembly files `.s`/`.S`), `include` (array of directories), `cflags` (array
of strings passed to the C compiler), `link` (array of library names to link,
such as `"ssl"`), `link-search` (array of directories), and
`linker-script` (path, binary targets on freestanding platforms). The C
compiler used is the one configured for the target (21.6.4).

```toml
[native]
sources = ["c/*.c", "asm/start.S"]
include = ["c/include"]
link = ["z"]
```
*From: D45, D38.*

**18.2.7** Each `[[generate]]` entry declares a generator (18.6): `tool`
(either `"dep:KEY/BIN"`, binary `BIN` of dependency `KEY`, or `"bin/NAME"`, a
binary target of this package's workspace), `args` (array of strings),
`inputs` (array of globs relative to the package directory), and `module` (the
module path, beneath this package, that the generated file becomes).

```toml
[[generate]]
tool = "dep:protogen/protogen"
inputs = ["proto/*.proto"]
module = "httpd.proto.messages"
```
*From: D45, D41.*

**18.2.8** `[profile.NAME]` tables configure build profiles. The profiles
`debug` and `release` always exist; a manifest may override their settings or
define others with `inherits = "debug"` or `"release"`. Settings:
`opt-level` (`0`, `1`, `2`, `3`, `"s"`, `"z"`), `debug-info` (boolean),
`debug-assertions` (boolean), `lto` (boolean), `strip` (boolean). Defaults:
`debug` is `opt-level = 0, debug-info = true, debug-assertions = true,
lto = false, strip = false`; `release` is `opt-level = 3, debug-info = false,
debug-assertions = false, lto = false, strip = true`. Only the root package's
profiles apply to a build. Profiles never change the meaning of a program
except through `@cfg(profile(...))` and `@cfg(debug_assertions)`; in
particular overflow checking is the same in every profile (D15).

```toml
[profile.release]
lto = true
[profile.bench]
inherits = "release"
debug-info = true
```
*From: D15, D42.*

**18.2.9** `[workspace]` with `members = [paths...]` makes the directory a
*workspace*: a set of packages built together, sharing one `tally.lock` and one
output directory at the workspace root. The workspace root's `tally.toml`
may contain only `[workspace]` and `[profile.*]` tables if it is not itself a
package.

```toml
[workspace]
members = ["httpd", "util", "tools/loadgen"]
```
*From: D43.*

## 18.3 Source layout

**18.3.1** The library target's modules are the `.tly` files under `src/`,
excluding `src/bin/`. Their module paths are given by 3.1.2 with the package
name as the first segment. Files that are not `.tly` files are ignored.
Directories and files whose names begin with `.` are ignored.

```text
src/lib.tly           -> httpd
src/router.tly        -> httpd.router
src/http/parse.tly    -> httpd.http.parse
```
*From: D18.*

**18.3.2** Within a package, the first segment of an import may be the package
name or `this`, which means the same package: `import this.router;`.

```tally
import this.http.parse;      // same as: import httpd.http.parse;
```
*From: D18.*

**18.3.3** A binary target `NAME` consists of the root module
`src/bin/NAME.tly`, whose module path is `bin`, and the modules under
`src/bin/NAME/`, whose module paths begin with `bin` (`src/bin/NAME/cli.tly`
is `bin.cli`). A binary target depends on its package's library target (if
any), which it imports by the package name, and on the package's
dependencies. Items of a binary target are not visible to other targets.

```tally
// src/bin/httpd.tly
import httpd.router;      // the library target
import bin.cli;           // src/bin/httpd/cli.tly
fn main() ! cli.Error may sys | block | alloc | panic { router.serve(cli.parse()?) }
```
*From: D18, D61.*

## 18.4 Version selection

**18.4.1** A package's *identity* is the pair of its *source* (for git
dependencies, the URL after normalisation (18.4.7) together with `dir`; for
path dependencies, the absolute directory) and its *compatibility line*: the
major version if it is at least 1, otherwise `0.MINOR` if the minor version is
at least 1, otherwise `0.0.PATCH`. Packages with different identities are
different packages, even if they have the same name; a program may contain
several versions of one source if they are on different compatibility lines.

```text
json 1.4.0 and json 1.9.2   -> same identity (line 1)
json 1.9.2 and json 2.0.0   -> different identities; both may be in one program
json 0.8.3 and json 0.9.0   -> different identities
```
*From: D43.*

**18.4.2** The version of each identity in a build is chosen by *minimal
version selection*:

1. Start with the set of *requirements* given by the root package's
   dependencies (and dev-dependencies for `tally test`): pairs of an identity
   and a minimum version.
2. For each identity, the *selected* version is the greatest minimum version
   required for it.
3. Read the manifest of every selected version; add its dependencies'
   requirements (not its dev-dependencies) to the set.
4. Repeat steps 2 and 3 until the selection does not change.

A version is *available* if the source has it (18.4.3). If a selected version
is not available, the build fails; Tally never chooses a version higher than
the greatest stated minimum. Pre-release versions are selected only if some
package requires that exact pre-release version or a later pre-release of
the same `MAJOR.MINOR.PATCH`.

```text
root requires  a ≥ 1.2.0, b ≥ 1.0.0
a 1.2.0 requires c ≥ 1.1.0
b 1.0.0 requires c ≥ 1.3.0, a ≥ 1.1.0
selected: a 1.2.0, b 1.0.0, c 1.3.0
```
*From: D43.*

**18.4.3** In a git source, version `X.Y.Z` is the commit named by the tag
`vX.Y.Z` (or `DIR/vX.Y.Z` for a package in subdirectory `DIR`). The manifest
at that commit must state the same version and a package name; otherwise the
build fails.

```text
git tag v1.4.2          -> httpd 1.4.2
git tag tls/v2.1.0      -> the package in directory tls, version 2.1.0
```
*From: D43.*

**18.4.4** The file `tally.lock` at the root of the package (or workspace)
records, for each selected package other than path dependencies: its identity,
version, the git commit hash, and its *content hash* (18.4.5). `tally fetch`,
`tally build` and `tally test` verify that every fetched package has the
recorded content hash, and fail if it does not. If the lock file is missing
or does not match the selection computed from the manifests, the command
writes a new lock file, unless `--locked` is given, in which case it fails.
The lock file is TOML with one `[[package]]` table per package, with keys
`name`, `source`, `version`, `commit` and `hash`, sorted by source then
version.

```toml
[[package]]
name = "json"
source = "git+https://example.org/json.git"
version = "0.9.1"
commit = "4f1c2e9a0d7b3c55e1f0a2b4c6d8e0f1a3b5c7d9"
hash = "sha256:9a0e…"
```
*From: D43.*

**18.4.5** The *content hash* of a package is `sha256:` followed by the
lowercase hexadecimal SHA-256 digest of this byte sequence: for each regular
file in the package directory, excluding the directories `.git` and `out` at
any depth and the file `tally.lock`, in increasing byte-wise order of the
file's path relative to the package directory (with `/` as separator), the
path's UTF-8 bytes, a zero byte, the file's length as an 8-byte little-endian
integer, and the file's bytes. Symbolic links are an error.

```text
"src/lib.tly" 00 <len:8 LE> <contents> "tally.toml" 00 <len:8 LE> <contents> ...
```
*From: D43.*

**18.4.6** Path dependencies are not version-selected: the package in the
directory is used as it is, its version is ignored for selection, and it is
not recorded in the lock file. A package obtained from a git source must not
have path dependencies that point outside its own repository; such a package
is an error when used as a dependency.

```toml
util = { path = "../util" }      # fine in the root or a workspace member
```
*From: D43.*

**18.4.7** A git URL is normalised by lowercasing its scheme and host,
removing a trailing `/` and a trailing `.git`, and removing a default port.
Only `https` and `ssh` URLs (and `file` URLs, for testing) are accepted.

```text
HTTPS://Example.org/json.git/  ->  https://example.org/json
```
*From: D43.*

## 18.5 Features

**18.5.1** For each selected package, the set of enabled features is the
union of the features requested for it by every package that depends on it
(plus `default` unless every dependent sets `default-features = false`), closed
under the `[features]` table's implications. A package is compiled once per
build with that set. Features must therefore only add functionality; code must
remain correct when more features are enabled than it asked for.

```toml
# a requests tls with ["ring"]; b requests tls with ["aws-lc"]
# tls is compiled once with {default, ring, aws-lc}
```
*From: D43, D42.*

## 18.6 Generators

**18.6.1** Before compiling a package, the build runs each of its generators
(18.2.7). The tool is built (for the host) and run with its `args`, with the
working directory set to an empty temporary directory containing copies of the
files matched by `inputs` (at their relative paths), and with the environment
cleared except for `TALLY_GEN_OUT`, the path of a directory where it must write
exactly one file, `module.tly`. The generator must run with no network access
and no file-system access outside those two directories; implementations
enforce this with the host's sandboxing where available and must document
where they cannot. The written file becomes the module named by `module`. A
non-zero exit status, or a missing output file, fails the build.

```text
inputs copied -> /tmp/gen-123/proto/a.proto
run protogen  -> writes $TALLY_GEN_OUT/module.tly
compiled as   -> module httpd.proto.messages
```
*From: D45, D41.*

**18.6.2** Generator outputs are cached by the hash of the tool binary, its
arguments and its inputs; a generator is rerun only if one of those changes.
Generated modules are written to `out/gen/` (21.2.2) so that they can be read.

```text
out/gen/httpd/proto/messages.tly
```
*From: D45, D2.*

## 18.7 Language versions

**18.7.1** Each package states with `tally = "N"` the major version of this
standard it is written in. This standard is version 1. A future major
version may change the language incompatibly; an implementation of a later
version must still compile packages that state an earlier version it
supports, with that version's rules, and link them with packages of other
versions. Minor revisions of a major version may only add features and fix
defects.

```toml
tally = "1"
```
*From: D62.*
