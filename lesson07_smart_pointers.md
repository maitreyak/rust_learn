# Lesson 7 — Smart Pointers and Interior Mutability

> The heap, shared ownership, and mutation-through-shared-references. When the borrow checker's default single-owner rule isn't what you need.

---

## Objectives

- Use `Box<T>`, `Rc<T>`, `Arc<T>`, and `Weak<T>` correctly.
- Understand `Deref` and `Drop`.
- Use `Cell<T>`, `RefCell<T>`, `Mutex<T>`, `RwLock<T>` for interior mutability.
- Pick the right container for a given ownership pattern.
- Design shared structures (buffer pools, shared columns) without fighting the borrow checker.

---

## 1. Why "smart pointers"

The stdlib ships a handful of types that wrap a value and add behavior:

- **`Box<T>`** — owns a heap-allocated `T`. Single owner.
- **`Rc<T>`** — reference-counted shared ownership, single-threaded.
- **`Arc<T>`** — atomically reference-counted, thread-safe.
- **`RefCell<T>` / `Mutex<T>` / `RwLock<T>`** — mutate through a shared reference, with borrow checking at runtime (instead of compile time).

They're "smart pointers" in the C++ sense: structs that implement `Deref<Target = T>` so `*x` and `x.method()` reach through to the inner value transparently.

---

## 2. `Box<T>` — single-owner heap

```rust
let b: Box<i64> = Box::new(42);
println!("{}", *b);        // 42
println!("{}", b.leading_zeros());  // auto-deref: same as (*b).leading_zeros()
```

Use `Box<T>` when:

- You need a value on the heap (recursive types, large values).
- You want a trait object: `Box<dyn Trait>`.
- You're handing ownership across an API boundary where stack storage won't work.

### Recursive types

```rust
enum Tree {
    Leaf(i64),
    Branch(Box<Tree>, Box<Tree>),   // infinite size without Box
}
```

Without `Box`, the compiler can't compute the size of `Tree` — it's recursive. `Box<Tree>` is one word (a pointer), so the outer enum has a known size.

### Trait objects

```rust
trait Operator { fn execute(&mut self); }

let ops: Vec<Box<dyn Operator>> = vec![
    Box::new(Scan { /* ... */ }),
    Box::new(Filter { /* ... */ }),
    Box::new(Aggregate { /* ... */ }),
];
```

Dynamic dispatch, heterogeneous collection. The bedrock of most query plan representations.

---

## 3. `Deref` and `Drop`

### `Deref`

A type implementing `Deref<Target = T>` lets `*x` and method-auto-deref reach through to `T`:

```rust
impl<T> Deref for Box<T> {
    type Target = T;
    fn deref(&self) -> &T { &**self }
}
```

This is why `Box<String>::len()` works — `.len()` is actually `String::len`, reached through one deref. Same mechanism: `String → &str`, `Vec<T> → &[T]`, `Rc<T> → &T`, `Arc<T> → &T`.

There's also `DerefMut` for `&mut` access. **Don't overuse `Deref`** — it's fine for smart pointers, misleading elsewhere.

### `Drop`

A type implementing `Drop` gets a custom destructor called when it goes out of scope:

```rust
struct Logger { name: String }

impl Drop for Logger {
    fn drop(&mut self) {
        println!("dropping logger {}", self.name);
    }
}

fn main() {
    let _a = Logger { name: "outer".into() };
    {
        let _b = Logger { name: "inner".into() };
    }  // prints: dropping logger inner
    // prints: dropping logger outer
}
```

Uses: closing file handles, releasing OS locks, decrementing ref counts, freeing heap memory. You rarely write `Drop` impls yourself — the stdlib does it for every resource-holding type. But you will encounter it.

Deterministic. No finalizer. No GC pauses. This is the part of Rust that feels most like well-written C++ RAII.

---

## 4. `Rc<T>` — shared ownership, single-threaded

```rust
use std::rc::Rc;

let a = Rc::new(String::from("shared"));
let b = Rc::clone(&a);
let c = Rc::clone(&a);
println!("{} (strong count: {})", a, Rc::strong_count(&a));
// shared (strong count: 3)
```

- **Clones the pointer, not the value.** All three variables share the same heap allocation.
- **Reference counted.** When the last `Rc` drops, the inner value drops.
- **Immutable access.** An `Rc<T>` only gives you `&T`. Multiple `Rc`s = multiple readers. If you need mutation, combine with `RefCell<T>` → `Rc<RefCell<T>>`.
- **Not thread-safe.** The ref count is not atomic. Sharing an `Rc` across threads is a compile error (it doesn't implement `Send`).

### Cycles and `Weak<T>`

Two `Rc`s pointing at each other create a cycle that will never drop — classic ref-count leak. Break cycles with `Weak<T>` (a non-owning reference):

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    parent: RefCell<Weak<Node>>,     // weak to break the cycle
    children: RefCell<Vec<Rc<Node>>>,
}
```

Pattern: owners hold `Rc`s, back-references hold `Weak`s.

---

## 5. `Arc<T>` — shared ownership, thread-safe

Identical API to `Rc<T>`, but the ref count uses atomic operations — ~1.5× slower per clone/drop, but safe across threads.

```rust
use std::sync::Arc;

let data = Arc::new(vec![1_i64, 2, 3, 4, 5]);

let mut handles = Vec::new();
for _ in 0..4 {
    let data = Arc::clone(&data);
    handles.push(std::thread::spawn(move || {
        let sum: i64 = data.iter().sum();
        println!("{}", sum);
    }));
}
for h in handles { h.join().unwrap(); }
```

All four threads share the same `Vec<i64>` through `Arc`. When the last thread finishes, the `Vec` drops.

Rule:

- Single thread → `Rc<T>`.
- Multiple threads → `Arc<T>`.

### `Arc<[T]>` — the shared immutable column

```rust
let col: Arc<[i64]> = Arc::from(vec![1, 2, 3, 4, 5]);
let view1 = Arc::clone(&col);
let view2 = Arc::clone(&col);
```

This is exactly what arrow-rs uses for immutable column buffers: one allocation, many readers. Cloning is just an atomic increment. Every operator that "reads the column" gets an `Arc<[i64]>` clone and holds it for the duration of its use.

---

## 6. Interior mutability — the `Cell`/`RefCell`/`Mutex` family

The borrow checker's rule: shared `&T` = no mutation. But sometimes you genuinely need "mutate through a shared reference" — and you can prove it's safe by adding runtime (or atomic) checks.

### `Cell<T>` — copy-type interior mutability

```rust
use std::cell::Cell;

let counter = Cell::new(0);
counter.set(counter.get() + 1);
println!("{}", counter.get());   // 1
```

`Cell` works for `Copy` types (`i32`, `u64`, `bool`, etc.). You can only `get` and `set` — no borrowing of the inner value. Zero runtime overhead; it's just a type-system trick to tell the compiler "trust me."

### `RefCell<T>` — runtime-checked borrowing

```rust
use std::cell::RefCell;

let v = RefCell::new(vec![1, 2, 3]);

{
    let b1 = v.borrow();            // shared borrow
    let b2 = v.borrow();            // another shared borrow, fine
    println!("{:?} {:?}", *b1, *b2);
}                                    // both released

{
    let mut m = v.borrow_mut();     // exclusive borrow
    m.push(4);
}                                    // released
```

- `.borrow()` → `Ref<T>` (shared). `.borrow_mut()` → `RefMut<T>` (exclusive).
- The same borrow rules apply (any number of shared OR one exclusive) — but **checked at runtime**. Violation → panic.
- Use when you can't prove statically what borrows will be live, but you know the pattern is safe.

### `Mutex<T>` — thread-safe version of `RefCell`

```rust
use std::sync::Mutex;

let v = Mutex::new(vec![1, 2, 3]);

{
    let mut guard = v.lock().unwrap();   // exclusive, blocks
    guard.push(4);
}   // guard drops, releases the lock
```

Any thread can `.lock()`; the blocking lock provides the exclusive access. The `MutexGuard` implements `Deref` and `DerefMut` to the inner `T`. When it drops, the lock is released — deterministic unlock, no "forgot to call `unlock()`."

### `RwLock<T>` — multi-reader, single-writer

```rust
use std::sync::RwLock;

let v = RwLock::new(vec![1, 2, 3]);

{
    let guards: Vec<_> = (0..4).map(|_| v.read().unwrap()).collect();
    // 4 concurrent readers
}

{
    let mut w = v.write().unwrap();
    w.push(4);
}
```

Faster than `Mutex` for read-heavy workloads. Slower for write-heavy or highly-contended workloads (writers are blocked by any active reader).

---

## 7. Choosing the right container

| Pattern                                               | Type                               |
|-------------------------------------------------------|-------------------------------------|
| Single owner, stack is fine                            | `T` directly                        |
| Single owner, on the heap                              | `Box<T>`                            |
| Shared readers, single thread                          | `Rc<T>`                             |
| Shared readers, multiple threads                       | `Arc<T>`                            |
| Shared + copy-type mutation, single thread             | `Rc<Cell<T>>` / `Cell<T>`           |
| Shared + complex mutation, single thread               | `Rc<RefCell<T>>`                    |
| Shared + thread-safe mutation                          | `Arc<Mutex<T>>`                     |
| Shared + read-heavy + occasional writes (threaded)     | `Arc<RwLock<T>>`                    |
| Break a ref cycle                                      | `Weak<T>`                           |
| Heterogeneous collection behind a trait                 | `Vec<Box<dyn Trait>>`               |
| Heterogeneous shared collection                         | `Vec<Arc<dyn Trait + Send + Sync>>` |

### Cost, roughly

- `Box<T>` — one heap allocation at construction; free thereafter.
- `Rc<T>` clone — increment a non-atomic counter. ~2–3 ns.
- `Arc<T>` clone — increment an atomic counter. ~5–10 ns.
- `RefCell` borrow — increment/decrement a counter + a branch. ~2–5 ns.
- `Mutex::lock` on an uncontended lock — atomic CAS. ~20–40 ns.
- `Mutex::lock` under contention — a kernel syscall for the wait. ~thousands of ns.
- `RwLock::read` on an uncontended lock — atomic add. ~20 ns.

---

## 8. DB relevance

### Immutable shared columns

Every row that flows through a query plan holds references to the source columns. You don't want to deep-copy a 1M-element column every time it's referenced. Use `Arc<[T]>`:

```rust
#[derive(Clone)]
struct Int64Column {
    values: Arc<[i64]>,
    nulls:  Option<Arc<[u64]>>,   // bitmap, 1 bit per row
}
```

Cloning an `Int64Column` is two atomic increments. The data buffer is never copied. This is exactly how `arrow-rs` does it.

### Buffer pool

A buffer pool is a shared cache of memory pages:

```rust
pub struct BufferPool {
    pages: Mutex<HashMap<PageId, Arc<Page>>>,
    capacity: usize,
}

impl BufferPool {
    pub fn get(&self, id: PageId) -> Arc<Page> {
        let mut pages = self.pages.lock().unwrap();
        if let Some(page) = pages.get(&id) {
            return Arc::clone(page);
        }
        // evict + load logic elided
        let page = Arc::new(load_from_disk(id));
        pages.insert(id, Arc::clone(&page));
        page
    }
}
```

`Arc<Page>` hands every caller a cheap reference. The `Mutex<HashMap<...>>` protects the page table. Real engines split the mutex into shards (one mutex per hash shard) to reduce contention.

### Query plan nodes

Physical operators in a plan form a tree where each child operator may be referenced by its parent. With single ownership you write:

```rust
pub enum Plan {
    Scan(ScanNode),
    Filter { input: Box<Plan>, pred: Expr },
    Project { input: Box<Plan>, cols: Vec<Expr> },
    Aggregate { input: Box<Plan>, groups: Vec<Expr>, aggs: Vec<AggExpr> },
}
```

Single ownership is fine here because plan nodes have a clean tree structure. If you needed to share sub-plans (e.g., common subexpression elimination), promote to `Arc<Plan>`.

---

## 9. Exercises

### 9.1 `Rc<RefCell<T>>` — the everyday combination

Build a tiny graph where each node has a `Vec<Rc<Node>>` of children. Add a method `add_child(&self, child: Rc<Node>)` that pushes onto the children list. Why do you need `RefCell` here, and what happens if you forget it?

### 9.2 Cycle and `Weak`

Extend 9.1 so each child also stores a `Weak<Node>` back-pointer to its parent. Verify (with `Rc::strong_count` and `Rc::weak_count`) that cycles are broken — when you drop the root, everything drops.

### 9.3 Buffer pool skeleton

Implement the buffer pool from §8 with a hard-coded 4-page capacity and a simple FIFO eviction policy. Write a stress test that spawns 4 threads each requesting random page IDs. Use `Arc<BufferPool>` to share the pool.

### 9.4 `Arc<[T]>` vs `Arc<Vec<T>>`

Benchmark cloning `Arc<[i64]>` vs `Arc<Vec<i64>>` for a 1M-element buffer. Are they the same cost? What's different about the in-memory layout? (Hint: `Arc<[T]>` is a thin header + the slice contents in one allocation; `Arc<Vec<T>>` is two indirections.)

### 9.5 `RefCell` panic

Write a small program that panics due to a `RefCell` borrow violation. Read the error message. When would you prefer compile-time `&mut` to runtime `RefCell::borrow_mut`?

### 9.6 `Mutex` vs `RwLock` benchmark

Build a concurrent `HashMap<u64, Vec<u8>>` behind both `Mutex` and `RwLock`. Run 8 threads doing 99% reads, 1% writes. Which wins? Now 50/50. Same answer?

### 9.7 `Drop` order

Given this code:

```rust
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }

fn main() {
    let a = Noisy("a");
    let b = Noisy("b");
    let c = Noisy("c");
}
```

Predict the output. Then check. Why is the drop order what it is?

---

## 10. Self-check

1. When do you need `Rc<T>`? When `Arc<T>`? Why not always `Arc`?
2. What's `Deref` for, and what does it let you do?
3. How does `Drop` ordering work for locally scoped bindings?
4. What's the difference between `RefCell<T>` and `Mutex<T>`?
5. Why does `Rc<T>` not implement `Send`?
6. What's the cost of an uncontended `Mutex::lock`?
7. What ownership pattern does a DB buffer pool require, and why does `Arc<Mutex<HashMap<PageId, Arc<Page>>>>` look inside-out from the outside in?

Next: **Lesson 8 — Concurrency.** Threads, channels, `rayon`, and the `Send`/`Sync` traits in depth.
