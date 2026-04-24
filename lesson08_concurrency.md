# Lesson 8 — Concurrency

> Threads, channels, data parallelism. The `Send`/`Sync` traits. How Rust turns "data race" into a compile error.

---

## Objectives

- Spawn threads with `std::thread` and share data via `Arc`, `Mutex`, and channels.
- Read and apply the `Send` and `Sync` traits.
- Use scoped threads for safe borrowing.
- Use `rayon` for data parallelism.
- Reason about atomic operations and memory ordering.
- Know when async is worth reaching for — and when it isn't.

---

## 1. Threads — `std::thread`

```rust
use std::thread;

fn main() {
    let handles: Vec<_> = (0..4).map(|i| {
        thread::spawn(move || {
            println!("hi from thread {}", i);
        })
    }).collect();

    for h in handles { h.join().unwrap(); }
}
```

- `thread::spawn` takes a closure and runs it on a fresh OS thread.
- The closure's type must be `FnOnce + Send + 'static` — self-contained (no borrows of the spawning stack).
- `.join()` blocks until the thread finishes; returns `Result<T, Box<dyn Any + Send>>` — `Err` if the thread panicked.

### Why `move`?

Without `move`, the closure would try to borrow captured variables. Those borrows can't outlive the spawning stack frame. `move` transfers ownership into the closure, satisfying the `'static` requirement.

```rust
let v = vec![1, 2, 3];
thread::spawn(move || println!("{:?}", v));   // ownership of v moves in
```

---

## 2. `Send` and `Sync` — the two auto-traits

These two traits are how Rust enforces thread safety at compile time.

- **`Send`** — "safe to transfer ownership to another thread."
- **`Sync`** — "safe to access from multiple threads via shared reference." A type `T` is `Sync` iff `&T` is `Send`.

Both are **auto-traits**: the compiler automatically implements them for types whose fields are all `Send`/`Sync`. You almost never implement them by hand; you just observe whether a type has them.

### Who has them

| Type                    | `Send`? | `Sync`? |
|-------------------------|---------|---------|
| `i32`, `bool`, `f64`    | yes     | yes     |
| `String`, `Vec<T>` where `T: Send/Sync` | yes | yes   |
| `Box<T>` where `T: Send/Sync` | yes / yes |       |
| `&T` where `T: Sync`    | yes     | yes     |
| `&mut T` where `T: Send`| yes     | yes     |
| `Rc<T>`                 | **no**  | **no**  |
| `Arc<T>` where `T: Send + Sync` | yes | yes   |
| `RefCell<T>` where `T: Send` | yes | **no**  |
| `Mutex<T>` where `T: Send` | yes  | yes     |
| raw `*const T`, `*mut T`| **no**  | **no**  |

Practical consequences:

- `thread::spawn(move || ...)` requires the closure to be `Send`. Anything you move in must be `Send`.
- Sharing state across threads usually means `Arc<T>` where `T: Send + Sync`.
- `Arc<Mutex<Vec<i64>>>` is `Send + Sync`. `Arc<RefCell<Vec<i64>>>` is **not** — the compiler refuses to share it across threads.

---

## 3. Channels — `std::sync::mpsc`

Message passing: "Do not communicate by sharing memory; share memory by communicating."

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<i64>();

    thread::spawn(move || {
        for i in 1..=5 { tx.send(i).unwrap(); }
        // tx drops here → channel closes
    });

    for received in rx {                 // iterates until channel closes
        println!("got {}", received);
    }
}
```

- `mpsc` = multi-producer, single-consumer.
- `tx.send(x)` — non-blocking for unbounded channels; returns `Err` if the receiver has been dropped.
- `rx.recv()` — blocks until a value arrives; returns `Err` when the last sender drops.
- Clone `tx` for additional producers: `let tx2 = tx.clone();`.

For production work, prefer `crossbeam_channel` — faster, bounded channels, and supports `select!`.

---

## 4. Scoped threads (`std::thread::scope`)

The `'static` requirement on `thread::spawn` is annoying when you just want a thread to borrow local data. Scoped threads solve this:

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3, 4, 5];

    thread::scope(|s| {
        s.spawn(|| {
            let sum: i64 = v.iter().sum();
            println!("sum = {}", sum);
        });
        s.spawn(|| {
            let max = v.iter().max().unwrap();
            println!("max = {}", max);
        });
    });  // scope() blocks until all spawned threads finish

    println!("{:?}", v);   // still owned here
}
```

`scope` guarantees every spawned thread is joined before `scope` returns, so borrows of `v` are safe — they can't outlive `main`. Use scoped threads whenever you can; they're strictly more ergonomic than `thread::spawn` for local computations.

---

## 5. Data parallelism with `rayon`

`rayon` gives you parallel iterators. For most "map over a big collection" problems, it's the right tool.

```toml
[dependencies]
rayon = "1"
```

```rust
use rayon::prelude::*;

fn main() {
    let v: Vec<i64> = (0..10_000_000).collect();
    let sum: i64 = v.par_iter().sum();        // parallel sum
    println!("{}", sum);

    let squared: Vec<i64> = v.par_iter().map(|x| x * x).collect();
}
```

- `.par_iter()` / `.into_par_iter()` — parallel versions of iteration.
- Same adapters as regular iterators: `map`, `filter`, `fold`, `sum`, etc.
- Work-stealing thread pool under the hood; no manual thread management.
- Closure must be `Fn + Send + Sync` (parallelism implies concurrent invocation).

### Filtered sum, in parallel

```rust
let total: i64 = v.par_iter()
    .filter(|&&x| x % 3 == 0)
    .sum();
```

Compared to explicit `Arc<Mutex<...>>` + `spawn` + `join`, this is ~10 lines less code and usually faster (better work distribution).

---

## 6. Atomics

For very fine-grained shared state — counters, flags, lock-free structures — use the atomic types in `std::sync::atomic`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicU64::new(0));

    let mut handles = Vec::new();
    for _ in 0..8 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1_000_000 {
                counter.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }
    for h in handles { h.join().unwrap(); }

    println!("{}", counter.load(Ordering::Relaxed));   // 8_000_000
}
```

### Memory orderings (the short version)

Every atomic operation takes an `Ordering`. In increasing strictness:

- **`Relaxed`** — no synchronization; just atomic. Use for counters, stats, anything where only the *final* total matters.
- **`Acquire`** / **`Release`** — establishes a happens-before relationship. Use for handoffs: writer `Release`-stores a flag; reader `Acquire`-loads the flag and then reads the data the writer prepared.
- **`AcqRel`** — both, for read-modify-write operations that act as a handoff.
- **`SeqCst`** — strictest; sequentially consistent across all threads. Default if you're unsure, but often overkill.

**Rule of thumb:** use `Relaxed` for counters; `Acquire`/`Release` for flag-based handoffs; `SeqCst` only if you can justify why you need it. For anything more complex, step up to a higher-level abstraction (`Mutex`, `RwLock`, `crossbeam` primitives).

Real advice: atomics are a footgun. Reach for `Mutex<T>` first; drop to atomics only when profiling shows the `Mutex` is a bottleneck.

---

## 7. Async (brief)

`async`/`await` and `tokio` build an entirely different concurrency model: many logical tasks multiplexed onto a small thread pool, with suspension points at I/O operations.

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    loop {
        let (socket, _) = listener.accept().await?;
        tokio::spawn(async move {
            // handle connection
        });
    }
}
```

Async is useful when:

- You have tens of thousands of concurrent I/O operations (network servers, scrapers).
- The workload is mostly waiting, not computing.

Async is **not** the right tool for:

- CPU-bound analytical queries — use `rayon` or plain threads.
- Low task counts (say, <1000 concurrent).
- Anywhere the added complexity (`Pin`, `Future`, colored functions, runtime dependencies) isn't buying you something concrete.

For an analytical DB engine, async is usually confined to the networking / client-facing edge — the query execution core is synchronous and thread-pooled. DuckDB is almost entirely sync. ClickHouse uses async only for its server loop.

---

## 8. DB relevance

### Parallel scan / morsel-driven execution

A modern analytical engine splits a table into small chunks ("morsels" — typically 100K rows) and processes them with a work-stealing thread pool. Each morsel is a unit of work; an operator pipeline runs the same sequence of kernels on each morsel.

```rust
use rayon::prelude::*;

fn parallel_sum_where(
    columns: &[Int64Column],
    predicate: impl Fn(&Int64Column) -> Vec<bool> + Sync,
) -> i64 {
    columns.par_iter()
        .map(|col| {
            let mask = predicate(col);
            sum_where(&col.values, &mask)   // from Lesson 2
        })
        .sum()
}
```

`rayon` handles the thread pool, work stealing, and reduction. The only state shared across threads is the input (`Arc<[i64]>`-style) and the output accumulator (atomic add, or per-thread result reduced at the end).

### Why this works in Rust specifically

In C++, "share an immutable buffer across threads and have each one write to its own output array" requires discipline and review to avoid bugs. In Rust, the type system enforces it: if a closure mutates shared state unsafely, `par_iter` won't accept it (the closure isn't `Sync`). You get data-race-freedom as a compile-time guarantee.

### The shared-mutable anti-pattern

Do NOT do this:

```rust
let total = Arc::new(Mutex::new(0_i64));
(0..1_000_000).into_par_iter().for_each(|x| {
    let mut t = total.lock().unwrap();   // 1M lock acquisitions!
    *t += x;
});
```

Every element takes the mutex. You've serialized all work through one lock. Instead:

```rust
let total: i64 = (0..1_000_000).into_par_iter().sum();   // par_iter handles the reduction
```

Rayon's `.sum()` does per-thread accumulation and a final reduction — O(threads) lock operations instead of O(N).

---

## 9. Exercises

### 9.1 Spawn and join

Start 4 threads, each computing the sum of a disjoint slice of a 10M-element `Vec<i64>`. Use `Arc<[i64]>` to share. Combine their results. Then do the same with `rayon::par_iter().chunks(...)` and compare timings.

### 9.2 Scoped threads

Repeat 9.1 using `thread::scope`. Note that you don't need `Arc` anymore — the scoped threads can borrow directly.

### 9.3 mpsc pipeline

Build a pipeline: thread A produces 1M integers, thread B doubles them, thread C sums them. Connect with two `mpsc` channels. Confirm the final sum matches `(0..1_000_000).map(|x| x*2).sum()`.

### 9.4 Counter race

Write the `AtomicU64` counter example (§6) and replace `AtomicU64` with a plain `i64` (cast the type through some unsafe). Observe: the final value is wrong. Explain why.

### 9.5 `Mutex` bottleneck

Two implementations: (a) 8 threads, each counts its own local `i64` then at the end does one `Mutex::lock` to merge into a global; (b) 8 threads, each does `counter.lock().unwrap() += 1` per iteration. Measure both on 1M iterations. Which is faster and by how much?

### 9.6 Parallel filter_sum

Use rayon to implement this on a 50M-row column:

```rust
fn parallel_sum_where(values: &[i64], mask: &[bool]) -> i64
```

Compare against your single-threaded version from Lesson 2. How much speedup do you see, and does it match your core count?

### 9.7 `Send` / `Sync` predict

For each type, say whether it's `Send`, `Sync`, both, or neither. Don't look it up:

- `i32`
- `Rc<i32>`
- `Arc<i32>`
- `Mutex<Rc<i32>>`
- `*const i32`
- `Cell<i32>`
- `&mut Cell<i32>`

Then check with a compile probe: `fn assert_send<T: Send>() {}` then `assert_send::<Cell<i32>>();` etc.

---

## 10. Self-check

1. What's the difference between `Send` and `Sync`? Is every `Send` type also `Sync`?
2. Why does `thread::spawn` require the closure to be `'static`, and why does `thread::scope` not?
3. Why does `Rc<T>` not implement `Send`, and what does the compiler actually check to enforce it?
4. When would you reach for channels over shared `Arc<Mutex<T>>`?
5. What's the cost of `fetch_add(1, Ordering::Relaxed)` vs taking a `Mutex`?
6. Describe a case where async is the right tool, and a case where it isn't.
7. In an analytical query engine, what's the natural unit of parallelism — rows, batches, columns, or something else? Why?

Next: **Lesson 9 — Unsafe Rust.** The part your DB will eventually need.
