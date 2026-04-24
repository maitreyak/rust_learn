# Lesson 5 — Iterators, Closures, and Collections

> The functional-flavored spine of idiomatic Rust. Zero-cost abstraction territory — chains that read like Kotlin or Scala and compile to tight for-loops.

---

## Objectives

- Write and use closures; understand `Fn`, `FnMut`, `FnOnce`.
- Know the three closure capture modes and when `move` is required.
- Read the `Iterator` trait and implement it for a custom type.
- Use adapters (`map`, `filter`, `flat_map`, `zip`, etc.) fluently.
- Consume iterators with `collect`, `fold`, `reduce`, `sum`, etc.
- Pick the right collection: `Vec`, `VecDeque`, `HashMap`, `BTreeMap`, `HashSet`, `BTreeSet`, `BinaryHeap`.
- Explain why iterators in Rust are lazy and what that buys you.

---

## 1. Closures

```rust
let add = |a, b| a + b;
println!("{}", add(2, 3));   // 5
```

A closure is an anonymous function value. Types are inferred from first use. Explicit form:

```rust
let add = |a: i32, b: i32| -> i32 { a + b };
```

### Capture modes

Closures automatically capture variables from the enclosing scope in whichever way is minimally needed:

```rust
let v = vec![1, 2, 3];
let print_len = || println!("{}", v.len());   // captures &v (shared borrow)
print_len();
```

```rust
let mut v = vec![1, 2, 3];
let mut push = |x| v.push(x);                 // captures &mut v
push(4);
push(5);
```

```rust
let v = vec![1, 2, 3];
let take_v = move || { println!("{:?}", v); }; // captures v by value (move)
take_v();
// println!("{:?}", v);  // ERROR — v moved into closure
```

### The three closure traits

A closure implements one or more of:

- **`FnOnce`** — can be called at least once; may consume captured values.
- **`FnMut`** — can be called multiple times; may mutate captured values.
- **`Fn`** — can be called multiple times; may only read captured values.

Every closure implements `FnOnce`. Closures that don't consume captures also implement `FnMut`. Closures that don't mutate anything also implement `Fn`.

Function signatures take whichever they need:

```rust
fn run_once<F: FnOnce()>(f: F) { f(); }
fn run_many<F: FnMut()>(mut f: F) { f(); f(); }
fn run_ro  <F: Fn()>(f: F)     { f(); f(); }
```

`Iterator::map` takes `FnMut` — it calls the closure once per element and allows the closure to mutate its environment. `rayon::par_iter().map` takes `Fn + Sync` — it may call the closure from many threads simultaneously.

### When is `move` required?

Whenever the closure outlives the scope that created it. The canonical case is `std::thread::spawn`:

```rust
let v = vec![1, 2, 3];
std::thread::spawn(move || println!("{:?}", v)).join().unwrap();
```

Without `move`, the closure would borrow `v` — but the thread might outlive `v`'s scope. `move` takes ownership so the closure is self-contained.

---

## 2. The `Iterator` trait

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // ...plus ~80 default methods built on `next`.
}
```

That's the entire required interface. Everything else — `map`, `filter`, `fold`, `collect`, `zip`, `chain` — is a default method built on top of `next()`.

### Implementing it yourself

```rust
struct Fibs { a: u64, b: u64 }

impl Iterator for Fibs {
    type Item = u64;
    fn next(&mut self) -> Option<u64> {
        let out = self.a;
        let next = self.a + self.b;
        self.a = self.b;
        self.b = next;
        Some(out)
    }
}

fn main() {
    let fibs = Fibs { a: 0, b: 1 };
    let first10: Vec<_> = fibs.take(10).collect();
    println!("{:?}", first10);
    // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
}
```

One method (`next`) and you get all 80 adapters and consumers for free.

### `IntoIterator`

`for x in v` desugars to `IntoIterator::into_iter(v)`. The three iteration modes of `Vec<T>`:

```rust
for &x in &v        { /* x: i32;   reads via iter()     */ }
for x in &mut v     { /* x: &mut i32; iter_mut()        */ }
for x in v          { /* x: i32;   into_iter(), consumes v */ }
```

---

## 3. Adapter taxonomy (lazy)

Adapters return another iterator. They don't do work until *driven* by a consumer.

| Adapter                                  | Effect                                    |
|------------------------------------------|--------------------------------------------|
| `.map(\|x\| f(x))`                       | transform each element                     |
| `.filter(\|x\| pred(x))`                 | keep elements where pred is true           |
| `.filter_map(\|x\| Option::<B>::_ )`     | filter + map in one step                   |
| `.flat_map(\|x\| iter_of_x)`             | flatten mapped iterables                   |
| `.flatten()`                             | one level of nesting off                   |
| `.zip(other)`                            | pair up with another iterator              |
| `.enumerate()`                           | yield `(usize, T)` pairs                   |
| `.chain(other)`                          | concat two iterators                       |
| `.take(n)`, `.skip(n)`                   | prefix / suffix                            |
| `.take_while(pred)`, `.skip_while(pred)` | prefix / suffix by predicate               |
| `.step_by(n)`                            | every nth element                          |
| `.rev()`                                 | reverse (needs `DoubleEndedIterator`)      |
| `.cloned()` / `.copied()`                | `Iter<T>` over `&T` → iter of `T`          |
| `.inspect(\|x\| ...)`                    | side effect on each element, pass through  |
| `.peekable()`                            | enable `peek()` without advancing          |
| `.scan(init, \|s, x\| ... )`             | stateful map (think running aggregate)     |
| `.windows(n)`, `.chunks(n)`              | (on slices) overlapping / non-overlapping  |

---

## 4. Consuming methods (drive the iterator)

| Method                              | Returns                       | Notes                                |
|-------------------------------------|-------------------------------|---------------------------------------|
| `.collect::<C>()`                   | `C`                           | into `Vec`, `String`, `HashMap`, …   |
| `.fold(init, \|acc, x\| ... )`      | `Acc`                         | generic reduce                        |
| `.reduce(\|a, b\| ... )`            | `Option<T>`                   | fold but init is the first element   |
| `.sum()`, `.product()`              | `T`                           | for numeric `T`                       |
| `.count()`                          | `usize`                       |                                       |
| `.max()`, `.min()`                  | `Option<T>`                   |                                       |
| `.max_by_key(f)`, `.min_by_key(f)`  | `Option<T>`                   | max/min by projection                 |
| `.any(pred)`, `.all(pred)`          | `bool`                        | short-circuit                         |
| `.find(pred)`                       | `Option<T>`                   | first match                           |
| `.position(pred)`                   | `Option<usize>`               | index of first match                  |
| `.for_each(\|x\| ...)`              | `()`                          | prefer `for` loops for readability   |
| `.last()`, `.nth(i)`                | `Option<T>`                   |                                       |
| `.partition(pred)`                  | `(C, C)`                      | split into two collections            |

---

## 5. Laziness

```rust
let _ = (0..10).map(|x| { println!("mapped {}", x); x * 2 });
// Prints NOTHING. map() just wraps the iterator.
```

An iterator adapter builds a *description* of a pipeline. No work runs until a consumer pulls. This enables:

- **Short-circuiting** — `.find()`, `.any()`, `.all()` stop as soon as they can answer.
- **Fusion** — chained adapters compile down to a single loop. `.filter(...).map(...).sum::<i64>()` is one loop, no intermediate allocations.
- **Streaming unbounded data** — `(0u64..).filter(is_prime).take(1000)` is fine.

### The `.collect()` hint

`.collect()` is "drive the iterator to completion and materialize." Type inference picks the collection:

```rust
let v: Vec<i32>          = (0..10).collect();
let s: String            = ['h', 'i'].iter().collect();
let m: HashMap<i32, i32> = (0..5).map(|i| (i, i*i)).collect();
let r: Result<Vec<_>, _> = texts.iter().map(|s| s.parse::<i64>()).collect();
//             ^^^ collecting Results: Ok(Vec) if all succeed, first Err if any fail
```

The `Result` collect pattern is the idiomatic way to fail-fast on a fallible batch.

---

## 6. `impl Trait` for iterators (recap)

Iterator adapter types are unnameable in practice — `Filter<Map<Chain<Take<Range<u64>>, ...>>>`. Use `impl Trait` to hide them:

```rust
fn evens_up_to(n: u64) -> impl Iterator<Item = u64> {
    (0..n).filter(|x| x % 2 == 0)
}
```

---

## 7. Collection types tour

Each has a purpose. Pick deliberately.

| Type                   | When to pick                                                      | Complexity (typical)        |
|------------------------|-------------------------------------------------------------------|------------------------------|
| `Vec<T>`               | Default dynamic array; append, iterate, index                      | push/index O(1), insert O(n) |
| `VecDeque<T>`          | Queue / deque — push_back, push_front, pop_back, pop_front         | all O(1)                     |
| `HashMap<K, V>`        | Unordered key→value, fast lookup; `K: Hash + Eq`                   | O(1) amortized               |
| `BTreeMap<K, V>`       | Ordered key→value; `K: Ord`; range queries                         | O(log n)                     |
| `HashSet<T>`           | Set membership                                                      | O(1) amortized               |
| `BTreeSet<T>`          | Ordered set                                                         | O(log n)                     |
| `BinaryHeap<T>`        | Priority queue (max-heap by default)                                | push/pop O(log n)            |
| `LinkedList<T>`        | Rarely useful (bad cache, bad API); almost always prefer `VecDeque` | O(1) splice                  |
| `Box<[T]>`             | Fixed-size heap slice — `Vec<T>` without capacity overhead          | —                            |
| `Arc<[T]>`             | Shared, immutable, reference-counted slice                           | —                            |

Rule of thumb: default to `Vec<T>`. Reach for `HashMap<K, V>` when you need lookup. Everything else is situational.

### HashMap idioms

```rust
let mut counts: HashMap<String, u32> = HashMap::new();
for word in text.split_whitespace() {
    *counts.entry(word.to_string()).or_insert(0) += 1;
}
```

The `.entry().or_insert()` pattern is the canonical "get or create" — replaces three lines of Java map.get / map.putIfAbsent boilerplate.

### Choosing `HashMap` vs `BTreeMap`

- `HashMap` is faster for point lookups but gives you no ordering.
- `BTreeMap` is O(log n) but supports `range()` queries and iterates in sorted order.

For DB work: `BTreeMap` shines for sorted-run merging and range scans; `HashMap` is the hash-join build side.

### Choosing the hasher

`HashMap`'s default hasher (`SipHash`) is DoS-resistant but slow. For internal engine use where inputs aren't adversarial, swap in a faster hasher like `ahash` or `fxhash`:

```rust
use ahash::AHashMap;
let mut m: AHashMap<&str, i64> = AHashMap::new();
```

Real DB engines use this routinely — 2–5× speedup on hash-heavy operations.

---

## 8. DB relevance

### Operator pipelines as iterator chains (the pedagogical case)

A simple query `SELECT x*2 FROM t WHERE x > 10 LIMIT 5` maps to:

```rust
table.rows()
    .filter(|r| r.x > 10)
    .map(|r|   r.x * 2)
    .take(5)
    .collect()
```

Read top-down, executes as a single fused loop, zero intermediate allocations.

### Why production engines don't actually do this

In a vectorized engine, you process **chunks of 1024–8192 rows at a time**, not individual rows. The "filter" operator doesn't check one row — it produces a selection vector or bitmap. The "map" isn't a closure — it's a dispatched kernel on a primitive column. Row-at-a-time iterator chains are too slow: one virtual call per row, per operator.

Iterators *are* still used as the **outer glue** — connecting morsels between operators, driving the work stealer, collecting final results. The inner kernels are explicit `for i in 0..n` or `chunks_exact` loops as you saw in Lesson 2.

### Rule

Use iterators where they buy you clarity at no cost (operator glue, catalog operations, result materialization, tests). Drop to explicit loops where the cost matters (hot kernels, per-tuple evaluation, SIMD targets).

---

## 9. Exercises

### 9.1 Implement `filter_map` via `flat_map`

Using only `flat_map`, write a function equivalent to `.filter_map()`:

```rust
fn my_filter_map<I, F, B>(iter: I, f: F) -> impl Iterator<Item = B>
where
    I: IntoIterator,
    F: FnMut(I::Item) -> Option<B>,
```

Hint: `Option<B>` implements `IntoIterator` — a `Some(b)` iterates once, a `None` iterates zero times. So `flat_map(f)` with `f: Item -> Option<B>` does the job.

### 9.2 Group-by with `HashMap`

Given `Vec<(String, i64)>` of (region, price) pairs, produce a `HashMap<String, i64>` of total price per region. Use `entry().or_insert(0)` in a fold or a for loop.

### 9.3 Running aggregate with `scan`

Using `.scan(...)`, write an iterator adapter that yields a running sum of its input:

```rust
let prefixes: Vec<i64> = (1..=5).into_iter()
    .scan(0, |acc, x| { *acc += x; Some(*acc) })
    .collect();
// [1, 3, 6, 10, 15]
```

Why is `scan` shaped this way — why does the closure get `&mut Acc` instead of `Acc`? What does that let you do that `fold` couldn't?

### 9.4 Custom iterator — `ChunkedSum`

Implement `Iterator` for a type that wraps a `&[i64]` and a chunk size, yielding the sum of each chunk:

```rust
struct ChunkedSum<'a> { data: &'a [i64], chunk: usize, pos: usize }

impl<'a> Iterator for ChunkedSum<'a> {
    type Item = i64;
    fn next(&mut self) -> Option<i64> { /* … */ }
}
```

Verify: `ChunkedSum::new(&v, 4).collect::<Vec<_>>()` equals `v.chunks(4).map(|c| c.iter().sum()).collect()`.

### 9.5 `Result` in `collect`

Write:

```rust
fn parse_all(ss: &[&str]) -> Result<Vec<i64>, std::num::ParseIntError>
```

using `.iter().map(|s| s.parse()).collect::<Result<Vec<_>, _>>()`. Explain in one sentence why this short-circuits on the first error.

### 9.6 Compare `HashMap` vs `BTreeMap`

Build both with 1M random `(u64, u64)` pairs. Time: (a) 1M point lookups, (b) a range scan over the middle 10%. Report which wins each and by how much. (Use `criterion` or a quick `Instant` harness.)

### 9.7 `ahash` swap

Replace your `HashMap` in 9.2 with `ahash::AHashMap`. Measure the speedup on 10M rows. Is it consistent with the 2–5× rule of thumb?

---

## 10. Self-check

1. What's the difference between `Fn`, `FnMut`, and `FnOnce`?
2. When do you need `move` on a closure?
3. What does "iterators are lazy" mean, and why does `.filter(...).map(...).sum()` not allocate?
4. What's the relationship between `Iterator` and `IntoIterator`? Which does `for x in v` use?
5. Give a case where `BTreeMap` is the right choice over `HashMap`.
6. Why do production DB engines not use row-at-a-time iterator chains for hot paths?
7. What does `.collect::<Result<Vec<_>, _>>()` do, and when would you use it?

Next: **Lesson 6 — Modules, Cargo, workspaces.** Project structure for a multi-crate engine.
