# Lesson 6 — Modules, Cargo, and Workspaces

> Project structure for a multi-crate engine. How Rust's module system fits together, what Cargo does, and how workspaces let you split a DB into logical components.

---

## Objectives

- Organize code with `mod`, `use`, and visibility modifiers.
- Know when a file becomes a module and when it needs its own folder.
- Read and write `Cargo.toml` — dependencies, features, profiles.
- Structure a multi-crate workspace.
- Write unit tests, integration tests, and doc tests.
- Set up Criterion benchmarks.
- Plan a plausible directory layout for a DuckDB-like engine.

---

## 1. The module system in one screen

A crate is a compilation unit. A module is a namespace inside a crate. Modules form a tree rooted at `lib.rs` (library crates) or `main.rs` (binary crates).

### Declaring a module

```rust
// in src/lib.rs
mod storage;              // expects src/storage.rs OR src/storage/mod.rs
mod catalog;              // expects src/catalog.rs OR src/catalog/mod.rs

pub use storage::BufferPool;
```

- `mod foo;` tells the compiler: "look for `foo.rs` next to me, or `foo/mod.rs`, and include it as a child module."
- Modern convention: use `foo.rs` for a leaf module, or `foo.rs` + `foo/` directory for a module with submodules (no `mod.rs` required since Rust 2018).

### Using items from modules

```rust
// Absolute path (from crate root):
use crate::storage::BufferPool;

// From parent:
use super::catalog::Schema;

// External crate (declared in Cargo.toml):
use arrow::array::Int64Array;

// Rename:
use crate::storage::Error as StorageError;

// Glob (sparingly — only for preludes):
use my_engine::prelude::*;
```

---

## 2. Visibility

By default, everything is private to its defining module. Opt into visibility explicitly:

| Modifier          | Visible from                              | Typical use                               |
|-------------------|-------------------------------------------|--------------------------------------------|
| (nothing)         | within this module only                   | implementation details                     |
| `pub`             | anywhere                                  | public API                                 |
| `pub(crate)`      | anywhere in this crate                    | shared internals of a crate                |
| `pub(super)`      | the parent module                          | shared between sibling modules             |
| `pub(in path)`    | within a specific ancestor module          | rare, fine-grained control                 |

Pattern for crates: expose a narrow `pub` surface in `lib.rs`, keep the rest `pub(crate)`. This is how you enforce module boundaries in a growing codebase.

### Re-exports

`pub use` re-exports a symbol so callers don't have to know its internal path:

```rust
// lib.rs
mod storage;
pub use storage::{BufferPool, Page};   // now `my_engine::BufferPool` works
```

Re-exports let you reorganize internals without breaking callers.

---

## 3. File layout conventions

Given this tree:

```
src/
├── lib.rs
├── storage.rs
├── storage/
│   ├── buffer_pool.rs
│   └── page.rs
├── catalog.rs
└── catalog/
    └── schema.rs
```

- `src/lib.rs` declares `mod storage; mod catalog;`
- `src/storage.rs` declares `mod buffer_pool; mod page;` and re-exports their items
- `src/storage/buffer_pool.rs` and `src/storage/page.rs` are submodules

A file is a module. A directory with the same name holds its submodules. No `mod.rs` file is required in Rust 2018+.

---

## 4. Crates: binary vs library

A **binary crate** has `src/main.rs` with `fn main()`. Compiles to an executable.

A **library crate** has `src/lib.rs`. Compiles to an rlib, dylib, cdylib, or staticlib (controlled by `[lib] crate-type` in Cargo.toml).

A single package can have both:

```
myengine/
├── Cargo.toml
├── src/
│   ├── lib.rs         # the library crate (my_engine::...)
│   └── main.rs        # the binary crate (depends on my_engine)
```

The binary's `main.rs` can `use my_engine::...` — the package name becomes the library's root.

---

## 5. `Cargo.toml` anatomy

```toml
[package]
name = "myengine"
version = "0.1.0"
edition = "2024"          # language edition; 2024 is current as of rustc 1.85+
authors = ["Maitreya K <maitreyak@gmail.com>"]
description = "A minimal analytical database engine"
license = "MIT OR Apache-2.0"

[lib]
name = "my_engine"        # how other crates import it (use my_engine::...)

[dependencies]
arrow       = "50"         # or { version = "50", features = ["ipc"] }
thiserror   = "1"
anyhow      = "1"
serde       = { version = "1", features = ["derive"] }
tokio       = { version = "1", features = ["rt-multi-thread"], optional = true }

[dev-dependencies]
criterion   = "0.5"
proptest    = "1"

[build-dependencies]
# for build.rs scripts (e.g., codegen, linking C libs)

[features]
default = ["simd"]
simd    = []                           # no extra deps, just a #[cfg] gate
async   = ["dep:tokio"]                 # enabling `async` pulls in tokio
jemalloc = ["dep:jemallocator"]

[profile.release]
lto            = true       # link-time optimization
codegen-units  = 1          # slower compile, better optimization
panic          = "abort"    # smaller binary, faster panics
debug          = false
strip          = true

[profile.bench]
debug = true                # preserve symbols for flamegraphs

[[bench]]
name    = "kernels"
harness = false              # Criterion uses its own harness
```

### Profiles

- **`dev`** (default for `cargo build` / `cargo run`) — debug symbols, overflow checks, no inlining. Fast compile, slow runtime. Use for development.
- **`release`** (`cargo build --release`) — full optimization, no debug checks. Use for benchmarks and production.
- **`test`** — based on `dev` but adjustable.
- **`bench`** — based on `release` but usually with `debug = true` for profiling.

### Features

Features are opt-in compile flags. In code:

```rust
#[cfg(feature = "simd")]
mod simd_kernels;

#[cfg(feature = "simd")]
pub use simd_kernels::sum_chunked;

#[cfg(not(feature = "simd"))]
pub use scalar_kernels::sum_chunked;
```

Turn on from the command line: `cargo build --features simd,async` or `cargo build --no-default-features`.

---

## 6. Workspaces — multi-crate projects

Once your code grows past one crate, split into a workspace. Each crate gets its own `Cargo.toml`; a workspace root ties them together:

```
myengine/
├── Cargo.toml              # workspace root (no [package])
├── crates/
│   ├── core/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── parser/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── planner/
│   ├── execution/
│   ├── storage/
│   └── cli/
│       ├── Cargo.toml
│       └── src/main.rs     # the binary
└── target/                  # shared build output
```

Workspace root `Cargo.toml`:

```toml
[workspace]
resolver = "2"
members  = ["crates/*"]

[workspace.dependencies]
arrow     = "50"
thiserror = "1"
anyhow    = "1"
```

Individual member `Cargo.toml`:

```toml
[package]
name    = "my_engine_execution"
version = "0.1.0"
edition = "2024"

[dependencies]
my_engine_core     = { path = "../core" }
my_engine_storage  = { path = "../storage" }
arrow              = { workspace = true }
thiserror          = { workspace = true }
```

Benefits:

- **Shared `target/`** — crates compile once, not per project.
- **Shared dependency versions** via `[workspace.dependencies]`.
- **Parallel compilation** — cargo builds independent crates in parallel.
- **Clear module boundaries** — inter-crate calls must go through `pub` APIs, enforcing discipline.

Build everything: `cargo build --release`. Build one crate: `cargo build -p my_engine_execution`.

---

## 7. Tests

Three flavors, all run by `cargo test`.

### Unit tests — in the same file

```rust
pub fn add(a: i64, b: i64) -> i64 { a + b }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn basic() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    #[should_panic]
    fn overflow_panics_in_debug() {
        let _ = i64::MAX + 1;
    }
}
```

`#[cfg(test)]` means the module is compiled only when testing — zero binary bloat in release builds. Unit tests can access private items (they're children of the module).

### Integration tests — in `tests/`

```
mycrate/
├── src/lib.rs
└── tests/
    ├── roundtrip.rs
    └── query_engine.rs
```

Each file in `tests/` is a separate crate that depends on your library. Only public items are accessible. Use integration tests for end-to-end behavior.

### Doc tests

```rust
/// Adds two numbers.
///
/// ```
/// assert_eq!(my_engine::add(2, 3), 5);
/// ```
pub fn add(a: i64, b: i64) -> i64 { a + b }
```

Code in triple-backtick doc blocks is compiled and run as a test. Free correctness guarantees for your documentation examples.

### Running tests

```
cargo test                  # all tests in the current package
cargo test -p mycrate       # one crate
cargo test mytest_name      # filter by name
cargo test -- --nocapture   # show println! output
cargo test --release        # run with optimizations
```

---

## 8. Benchmarking with Criterion

Cargo has a built-in benchmark harness but it's unstable; in practice, everyone uses **Criterion**.

`Cargo.toml`:

```toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name    = "kernels"
harness = false
```

`benches/kernels.rs`:

```rust
use criterion::{criterion_group, criterion_main, Criterion, black_box};
use my_engine::sum_chunked;

fn bench_sum(c: &mut Criterion) {
    let data: Vec<i64> = (0..10_000_000).collect();

    c.bench_function("sum_chunked_2048", |b| {
        b.iter(|| sum_chunked(black_box(&data), 2048))
    });
}

criterion_group!(benches, bench_sum);
criterion_main!(benches);
```

Run: `cargo bench`. Criterion handles warmup, statistical significance, regression detection, and HTML reports (`target/criterion/`). Use it instead of hand-rolled `Instant::now()` loops when you care about the numbers.

---

## 9. A plausible DB engine layout

For reference, here's how a DuckDB-like engine could be split:

```
myengine/
├── Cargo.toml                      # workspace root
├── crates/
│   ├── common/                      # errors, types, shared utilities
│   ├── storage/                     # buffer pool, page format, mmap, WAL
│   ├── catalog/                     # schema, table metadata
│   ├── parser/                      # SQL → AST (uses sqlparser-rs)
│   ├── planner/                     # AST → logical plan → physical plan
│   ├── expr/                        # scalar expressions, constant folding
│   ├── execution/                   # physical operators, kernels
│   ├── kernels/                     # SIMD-vectorized primitive kernels
│   ├── aggregation/                 # group-by, window functions
│   ├── join/                        # hash-join, sort-merge-join
│   ├── cli/                         # bin: REPL / query runner
│   └── server/                      # optional: embed server
├── benches/                         # cross-crate benchmarks
├── examples/                        # example binaries
└── docs/
```

Each crate has a focused responsibility and a narrow public API. Internal details stay `pub(crate)`. Inter-crate calls go through well-defined interfaces — which makes the borrow checker's job easier and surface area-to-review smaller.

---

## 10. Exercises

### 10.1 Bootstrap a workspace

Create a `toydb/` workspace with three crates: `core`, `storage`, `cli`. Make `core` depend on nothing, `storage` depend on `core`, and `cli` (a binary) depend on both. Verify `cargo build` in the workspace root compiles all three.

### 10.2 Visibility

Given this code in `storage/src/lib.rs`:

```rust
struct BufferPool { pages: Vec<Page> }
struct Page(Vec<u8>);

impl BufferPool {
    fn new() -> Self { BufferPool { pages: Vec::new() } }
}
```

What's visible from the `core` crate? Adjust visibility so `BufferPool::new()` is callable from `core`, but the `pages` field stays private.

### 10.3 Feature flags

Add a `simd` feature to your `kernels` crate. Inside, write two impls of `sum_column`:

```rust
#[cfg(feature = "simd")]
pub fn sum_column(xs: &[i64]) -> i64 { /* SIMD impl */ }

#[cfg(not(feature = "simd"))]
pub fn sum_column(xs: &[i64]) -> i64 { /* scalar impl */ }
```

Verify `cargo build --features simd` and `cargo build --no-default-features` both compile. Confirm the feature actually changes which function is compiled (add a `println!` to prove it).

### 10.4 Unit + integration tests

In your `storage` crate, add:

- A unit test (in the same file) checking a private helper.
- An integration test (in `tests/`) that exercises the crate's public API end-to-end.

Confirm `cargo test` runs both.

### 10.5 Doc test

Write a doc test for `sum_column` that shows both its success case and what happens for an empty slice. Run `cargo test --doc`.

### 10.6 Criterion comparison

Add Criterion benches comparing your `sum_column` (SIMD on) against `sum_column` (SIMD off). Run `cargo bench` and open the generated HTML report. Screenshot the comparison.

### 10.7 Dependency hygiene

Take one of your crates and run `cargo tree`. Identify any unexpected transitive dependencies. Run `cargo machete` (`cargo install cargo-machete` first) to find unused dependencies in your `Cargo.toml`.

---

## 11. Self-check

1. What's the difference between `pub`, `pub(crate)`, and `pub(super)`?
2. Where can a module live on disk — how does the compiler find `mod storage;`?
3. What's the difference between a unit test, an integration test, and a doc test?
4. What does `lto = true` in `[profile.release]` do, and what does it cost?
5. Why would you split a project into a workspace instead of one big crate?
6. What does `#[cfg(feature = "simd")]` do at compile time?
7. Why does `cargo test` compile everything in debug mode by default, and when would you `cargo test --release`?

Next: **Lesson 7 — Smart pointers & interior mutability.**
