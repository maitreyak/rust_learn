# Lesson 4 — Error Handling

> No exceptions. No `throws`. No `try/catch`. Errors are values; the type system threads them for you.

---

## Objectives

- Use `Option<T>` and `Result<T, E>` idiomatically.
- Propagate errors with `?` and understand what it actually does.
- Define custom error types — by hand, and with `thiserror`.
- Use `anyhow` for application-level error juggling.
- Know when to `panic!`, when to return `Result`, and when `None` is the right shape.

---

## 1. The two shapes Rust uses to say "maybe not"

```rust
enum Option<T>    { Some(T), None }
enum Result<T, E> { Ok(T),   Err(E) }
```

Everything else — `?`, `.unwrap()`, `.map()`, `.and_then()`, error conversion traits — is machinery layered on top of these two plain enums.

- **`Option<T>`** — "might or might not have a `T`." Use when absence is expected and uninformative: "not found in the map," "end of the iterator," "no first element." The caller doesn't need to know *why* it's missing.
- **`Result<T, E>`** — "produced a `T`, or an error `E` describing what went wrong." Use whenever the caller deserves to know the reason.

Rust has no `null`. Dereferencing a non-`Option` reference that "might be missing" is not a category of bug that exists. The type system shunts you into `Option` whenever absence is possible.

---

## 2. `Option<T>` in practice

```rust
fn find_user(id: u64, users: &[(u64, String)]) -> Option<&str> {
    for (uid, name) in users {
        if *uid == id { return Some(name); }
    }
    None
}

fn main() {
    let db = vec![(1, "alice".into()), (2, "bob".into())];
    match find_user(2, &db) {
        Some(name) => println!("found: {}", name),
        None       => println!("not found"),
    }
}
```

### Everyday methods

| Method                           | Returns        | Use when…                              |
|----------------------------------|----------------|-----------------------------------------|
| `.unwrap()`                      | `T`            | tests / prototypes; panics if `None`   |
| `.expect("msg")`                 | `T`            | like `unwrap` with a custom panic msg  |
| `.unwrap_or(default)`            | `T`            | substitute a default                   |
| `.unwrap_or_else(\|\| compute())`| `T`            | substitute a *computed* default        |
| `.unwrap_or_default()`           | `T`            | use `T::default()`                     |
| `.map(\|x\| f(x))`               | `Option<U>`    | transform the inner if present         |
| `.and_then(\|x\| opt2)`          | `Option<U>`    | chain another Option-returning op      |
| `.or(other_option)`              | `Option<T>`    | fall back to another Option            |
| `.ok_or(err)` / `.ok_or_else`    | `Result<T, E>` | lift `None` → `Err`                    |
| `.is_some()` / `.is_none()`      | `bool`         | test without unwrapping                 |
| `.as_ref()`                      | `Option<&T>`   | borrow through an `Option<T>`           |
| `.take()`                        | `Option<T>`    | swap in `None`, yield the old value    |

### `if let` / `while let`

```rust
if let Some(name) = find_user(2, &db) {
    println!("found: {}", name);
}
```

"If this matches `Some(name)`, bind `name` and run the block." Same for `while let`: loop while the pattern keeps matching. It's the tool you reach for when a `match` would be overkill.

### Avoid `.unwrap()` in real code

`.unwrap()` panics on `None`. Outside of tests and throwaway scripts, prefer:

- `?` (next section) if you're in a function that returns `Result`/`Option`.
- `.ok_or(...)` to turn `None` into a meaningful `Err`.
- `.expect("reason why this can't be None")` if you've proven it can't be `None` and want the proof in code.

---

## 3. `Result<T, E>` in practice

```rust
use std::num::ParseIntError;

fn parse_id(s: &str) -> Result<u64, ParseIntError> {
    s.parse::<u64>()
}

fn main() {
    match parse_id("42") {
        Ok(id)  => println!("id = {}", id),
        Err(e)  => println!("bad id: {}", e),
    }
}
```

Every I/O, parsing, or network call in the stdlib returns `Result<T, E>`. There is no exception path — if something goes wrong, it appears in the return type.

### Everyday methods (mirrors `Option`, plus a few)

| Method                          | Returns                           | Use when…                            |
|---------------------------------|-----------------------------------|---------------------------------------|
| `.unwrap()` / `.expect`         | `T`                               | tests / proofs                        |
| `.unwrap_or` / `.unwrap_or_else`| `T`                               | default                               |
| `.map(f)` / `.map_err(f)`       | `Result<U, E>` / `Result<T, F>`   | transform one side                    |
| `.and_then(f)`                  | `Result<U, E>`                    | chain fallible ops                    |
| `.or_else(f)`                   | `Result<T, F>`                    | fall back or convert error           |
| `.ok()`                         | `Option<T>`                       | throw away the error                  |
| `.err()`                        | `Option<E>`                       | keep only the error                   |
| `?` (see §4)                    | `T`, with early return on `Err`   | propagate — the one you'll use most  |

---

## 4. The `?` operator

`?` is a single-character alternative to a 5-line match. Inside a function returning `Result<T, E>`:

```rust
fn expr() -> Result<T, E>;

let x = expr()?;
// desugars to:
let x = match expr() {
    Ok(v)  => v,
    Err(e) => return Err(e.into()),
};
```

Three things happen:

1. On `Ok(v)`, unwrap and bind to `x`.
2. On `Err(e)`, call `.into()` to convert it to the function's declared error type, then `return`.
3. `?` also works on `Option<T>` inside a function returning `Option<U>` — `None` early-returns as `None`.

### Chaining

```rust
fn load_and_sum(path: &str) -> Result<i64, Box<dyn std::error::Error>> {
    let text = std::fs::read_to_string(path)?;              // io::Error → Box<dyn Error>
    let numbers: Vec<i64> = text
        .lines()
        .map(|l| l.trim().parse::<i64>())
        .collect::<Result<_, _>>()?;                        // ParseIntError → Box<dyn Error>
    Ok(numbers.iter().sum())
}
```

Three different potential error sources, one linear happy path. Java equivalent would need `try/catch` or `throws IOException, NumberFormatException`.

### The `.into()` / `From` contract

`?` calls `.into()` to convert the error. That works because of a blanket impl: `impl<T, U: From<T>> Into<U> for T`. So if you define `impl From<io::Error> for MyError`, you get `io::Error → MyError` conversion automatically, and `?` picks it up. This is why error hierarchies are so ergonomic.

---

## 5. Custom error types — by hand

For library code — crates others depend on — roll a proper error enum:

```rust
#[derive(Debug)]
pub enum DbError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
    BadSchema { line: usize, message: String },
    OutOfRange(u64),
}

impl std::fmt::Display for DbError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            DbError::Io(e)                => write!(f, "io error: {}", e),
            DbError::Parse(e)             => write!(f, "parse error: {}", e),
            DbError::BadSchema { line, message } => {
                write!(f, "bad schema at line {}: {}", line, message)
            }
            DbError::OutOfRange(n)        => write!(f, "out of range: {}", n),
        }
    }
}

impl std::error::Error for DbError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            DbError::Io(e)    => Some(e),
            DbError::Parse(e) => Some(e),
            _                 => None,
        }
    }
}

impl From<std::io::Error> for DbError {
    fn from(e: std::io::Error) -> Self { DbError::Io(e) }
}
impl From<std::num::ParseIntError> for DbError {
    fn from(e: std::num::ParseIntError) -> Self { DbError::Parse(e) }
}
```

That's a lot of boilerplate for four variants. `thiserror` collapses it.

---

## 6. `thiserror` — proc-macro derive for library errors

```toml
# Cargo.toml
[dependencies]
thiserror = "1"
```

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DbError {
    #[error("io error: {0}")]
    Io(#[from] std::io::Error),

    #[error("parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("bad schema at line {line}: {message}")]
    BadSchema { line: usize, message: String },

    #[error("out of range: {0}")]
    OutOfRange(u64),
}
```

That replaces all the hand-written `Display`, `Error`, and `From` impls above. `#[from]` generates the `From<inner>` impl, enabling `?` conversion. This is the idiomatic library-error pattern.

---

## 7. `anyhow` — for applications

Application code (binaries, scripts, top-level bubbling) often doesn't care about fine-grained error variants — it just wants "something went wrong; here's a chain of context":

```toml
[dependencies]
anyhow = "1"
```

```rust
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let text = std::fs::read_to_string(path)
        .with_context(|| format!("reading config from {}", path))?;
    let cfg: Config = toml::from_str(&text)
        .with_context(|| format!("parsing toml config from {}", path))?;
    Ok(cfg)
}
```

`anyhow::Error` wraps `Box<dyn Error>` with extra context-chaining. `Result<T>` aliases `Result<T, anyhow::Error>`. `.context()` / `.with_context()` adds a breadcrumb each time an error propagates:

```
Error: parsing toml config from ./config.toml

Caused by:
    0: unexpected character at line 5 column 12
```

**Rule of thumb:**
- Library crates → `thiserror`. Callers can match on specific variants.
- Binary / application crates → `anyhow`. Clean diagnostics on exit, no downstream matching.

---

## 8. `panic!` vs `Result`

```rust
// Genuinely impossible — invariant violation. Panic.
fn square_root_of_positive(x: f64) -> f64 {
    if x < 0.0 { panic!("precondition violated: x = {}", x); }
    x.sqrt()
}

// Expected-to-sometimes-fail. Return Result.
fn read_u64_le(bytes: &[u8]) -> Result<u64, ParseError> {
    if bytes.len() < 8 { return Err(ParseError::TooShort); }
    Ok(u64::from_le_bytes(bytes[..8].try_into().unwrap()))
}
```

### When to panic

- **Broken invariants inside your code** — a `.unwrap()` you've proven can't fail, documented by a comment.
- **Unrecoverable environment** — OOM, corrupted internal state.
- **Tests / examples / prototypes** — panics give you a stack trace and kill the test.

### When to return `Result`

- **External input or I/O** — parse errors, file-not-found, network failures, malformed data.
- **Optional config / feature detection.**
- **Anything a user can cause.** If a user can trigger it, it's not unrecoverable.

### Panic vs abort

By default `panic!` unwinds the stack (running `Drop` impls), catchable with `catch_unwind`. Set `panic = "abort"` in `Cargo.toml` to just terminate — smaller binary, faster panics, no unwinding. Most systems code (and DB engines) prefers `abort` for release builds; unwinding through `unsafe` code is a minefield.

---

## 9. `Option` vs `Result` — which shape?

| Situation                                              | Shape                                     |
|--------------------------------------------------------|-------------------------------------------|
| "Not found" / "no next element"                         | `Option<T>` — None is uninformative       |
| "Parse failed," "I/O failed," "validation rejected it"  | `Result<T, E>` — caller needs the reason |
| "Optional config field"                                 | `Option<T>` in the struct                 |
| Internally "this might be missing, and it's fine"       | `Option<T>`                               |
| Externally "this operation can fail for real reasons"   | `Result<T, E>`                            |

Rules:

- If the caller has anything useful to do with "why it failed," return `Result`.
- If the caller only cares "it wasn't there; I'll handle that uniformly," return `Option`.
- Use `.ok_or(...)` to promote an `Option` to a `Result` when the layer above needs context.

---

## 10. DB relevance

- **Per-row errors vs. per-query errors.** Per-tuple kernels usually don't return `Result` — they're hot loops and propagating error per element is prohibitively slow. Instead, they write a null bitmap or error bitmap alongside the output column. Only the *query* boundary returns a `Result`.
- **Panics in kernel code are bugs.** A SIMD kernel that panics on bad input means a schema check failed upstream. Enforce schema/type invariants at the operator boundary.
- **Short reads, short files.** Anywhere you read from disk/mmap/socket, use `?` and let errors bubble. Do NOT silently truncate.

Standard idiom for an engine-level error enum (paraphrased from DataFusion):

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum EngineError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("parse error: {0}")]
    Parse(String),

    #[error("schema mismatch: expected {expected}, got {actual}")]
    Schema { expected: String, actual: String },

    #[error("type mismatch on column {col}: cannot compare {lhs} and {rhs}")]
    TypeMismatch { col: String, lhs: String, rhs: String },

    #[error("internal error: {0}")]
    Internal(String),
}

pub type Result<T> = std::result::Result<T, EngineError>;
```

Then every function in the engine returns `engine::Result<T>`.

---

## 11. Exercises

### 11.1 Parse or default

```rust
fn parse_or_zero(s: &str) -> i64
```

Returns the parsed number, or 0 on parse failure. Two lines, one method call.

### 11.2 Short-circuit with `?`

```rust
fn parse_and_sum(s: &str) -> Result<i64, Box<dyn std::error::Error>> {
    // "10,32" → Ok(42); "foo,32" or "10" → Err
}
```

Use `.split_once(',')`, `?`, and `str::parse`. Explain in one line why `?` works across two different error types here.

### 11.3 Custom error — by hand

Build a `CsvError` enum without `thiserror` with three variants (`Io`, `BadField { line }`, `TooFewColumns { line }`). Implement `Display`, `Error`, and `From<io::Error>`. Write a function reading a CSV file returning `Result<Vec<Vec<String>>, CsvError>`.

### 11.4 Custom error — with `thiserror`

Rewrite 11.3 using `thiserror`. Count the lines you saved.

### 11.5 `Option` → `Result`

Given `fn find_user(id: u64, users: &[(u64, String)]) -> Option<&str>`, write a wrapper returning `Result<&str, UserError>` with `UserError::NotFound { id }` when missing. Use `.ok_or(...)`.

### 11.6 Error propagation chain

A function that: reads a path from `argv[1]`, opens the file, reads contents, parses each line as `i64`, and sums. Every step can fail. Use `anyhow` and `.context()` for breadcrumbs. Run against a file with a non-numeric line and inspect the error output.

### 11.7 When to panic — design drill

For each signature, decide: panic or return Result? Defend in one line.

```rust
fn get_column_by_index(t: &Table, i: usize) -> &Column;
fn get_column_by_name (t: &Table, name: &str) -> &Column;
fn execute_query(sql: &str) -> QueryPlan;
fn u64_from_le_bytes(b: &[u8]) -> u64;
```

---

## 12. Self-check

1. Why does Rust have no `null`? What does it use instead, and what category of bug does this eliminate?
2. What does `?` actually do in terms of match arms?
3. What's the relationship between `?`, `From`, and `Into`?
4. When would you reach for `thiserror` vs. `anyhow`?
5. Give one example each of: (a) a correct use of `.unwrap()`, (b) an abuse of `.unwrap()`.
6. Why does the `Error` trait have a `source()` method?
7. In a vectorized kernel over 1M rows, why would you NOT return `Result<T, E>` per row?

Next: **Lesson 5 — Iterators, closures, and collections.**
