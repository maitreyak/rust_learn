# Lesson 2 — `&mut`, Slices, and Strings

> The concrete primitives you'll use every day — and the ones your database is going to be built out of.

---

## Objectives

By the end of this lesson you should be able to:

- Pass mutable state between functions with `&mut T` and know the single-writer rule inside out.
- Explain what a slice (`&[T]`, `&str`) actually is at the machine level.
- Pick the right function signature between `&Vec<T>` / `&[T]` / `Vec<T>` (and the `String` trio).
- Slice, sub-slice, chunk, and window a contiguous buffer without allocating.
- See why slices are the natural unit of work in a columnar analytical database.

---

## 1. `&mut` — the second kind of reference

```rust
fn increment_all(xs: &mut Vec<i32>) {
    for x in xs.iter_mut() {
        *x += 1;
    }
}

fn main() {
    let mut v = vec![1, 2, 3];
    increment_all(&mut v);
    println!("{:?}", v);   // [2, 3, 4]
}
```

Three things to notice:

1. **Mutability is opt-in at every layer.** `let mut v`, `&mut v` at the call site, and `&mut Vec<i32>` in the signature. If any of the three is missing, the compiler refuses.
2. **`*x` dereferences.** `x` here is `&mut i32`. `*x += 1` writes through the reference. In most cases the `.` operator auto-dereferences for you (method calls, field access), but write access through a `&mut T` to a primitive needs the explicit `*`.
3. **Exclusivity.** While `&mut v` is alive, nothing else — not the owner, not another reference — may read or write `v`. This one rule is how Rust eliminates data races at compile time.

The borrow rules, restated:

- Any number of `&T` **or** exactly one `&mut T` — never both, never two `&mut`.
- Rule applies per-value across the whole program, including across threads.

---

## 2. Slices — `&[T]`

A slice is **a pair `(pointer, length)`** — a non-owning, bounds-checked view into a contiguous run of `T`s. On a 64-bit machine it's 16 bytes regardless of the length.

```rust
fn sum(xs: &[i32]) -> i32 {
    let mut total = 0;
    for x in xs {
        total += x;
    }
    total
}

fn main() {
    let v = vec![1, 2, 3, 4, 5];
    let arr = [10, 20, 30];

    println!("{}", sum(&v));         // Vec coerces to &[i32]
    println!("{}", sum(&arr));       // array coerces
    println!("{}", sum(&v[1..4]));   // sub-slice: elements 1, 2, 3
}
```

### Construction

| Expression          | Produces                         |
|---------------------|----------------------------------|
| `&v`                | `&Vec<T>` → coerces to `&[T]`    |
| `&v[..]`            | `&[T]` explicitly (full range)   |
| `&v[a..b]`          | `&[T]` over half-open `[a, b)`   |
| `&v[a..]` / `&[..b]`| Slice from `a` / up to `b`       |
| `&mut v[..]`        | `&mut [T]` — mutable slice       |

Ranges are half-open: `v[1..4]` gives elements at indices 1, 2, 3. Out-of-range indices panic at runtime (unlike `&v[..]`, which is always safe).

### Why prefer `&[T]` over `&Vec<T>` in signatures

- `&[T]` accepts `Vec<T>`, `[T; N]`, `Box<[T]>`, sub-slices, mmap'd regions, or raw allocations from FFI — anything contiguous.
- `&Vec<T>` only accepts a `Vec`. You've over-specified.

**Rule of thumb:** take `&[T]`, return `Vec<T>`. Take the cheapest input; return the thing you own.

### Common slice methods you'll use constantly

```rust
xs.len()                      // usize
xs.is_empty()
xs.first()                    // Option<&T>
xs.last()                     // Option<&T>
xs.get(i)                     // Option<&T> — non-panicking index
xs.iter()                     // iterator of &T
xs.iter_mut()                 // iterator of &mut T (needs &mut [T])
xs.chunks(n)                  // iterator of &[T] of length ≤ n
xs.chunks_exact(n)            // iterator of &[T] of length exactly n; remainder via .remainder()
xs.windows(n)                 // iterator of overlapping &[T] of length n
xs.split_at(i)                // (&[T], &[T]) — no alloc
xs.split_first()              // Option<(&T, &[T])>
xs.binary_search(&needle)     // Result<usize, usize>
xs.sort()                     // needs &mut [T]
```

`chunks_exact(n)` is the one you'll reach for when you vectorize kernels to process, say, 2048 rows at a time.

### The important subtlety: iteration modes

```rust
let v = vec![1, 2, 3];

for x in &v       { /* x: &i32     */ }   // iter()     — shared borrow
for x in &mut v   { /* x: &mut i32 */ }   // iter_mut() — exclusive borrow
for x in v        { /* x: i32      */ }   // into_iter() — consumes v
```

The third form *moves* `v`; you can't use it again afterward. This is deliberate: sometimes you want to pull each element out by value (e.g., transferring ownership into another container).

---

## 3. `String` and `&str`

Same pattern as `Vec<T>` / `&[T]`, but for UTF-8 text.

- `String` — owned, growable, heap-allocated UTF-8. `(ptr, len, capacity)` on the stack + heap buffer.
- `&str` — borrowed view into UTF-8 bytes. `(ptr, len)` on the stack, no ownership.

Sources of `&str`:

- String literals: `"hello"` has type `&'static str`, pointing into the binary's read-only data section.
- Slicing a `String`: `&s[..5]`.
- Parsed/read input, function returns, etc.

```rust
fn greet(name: &str) -> String {
    let mut out = String::from("hello, ");
    out.push_str(name);
    out
}

fn main() {
    let s = String::from("world");
    println!("{}", greet(&s));    // &String coerces to &str
    println!("{}", greet("rust")); // string literal already &str
}
```

**Signature rule of thumb:** take `&str`, return `String`.

### The char-boundary gotcha

Because `&str` is UTF-8 bytes — not Unicode code points — byte indexing can panic if you land in the middle of a multi-byte character:

```rust
let s = "héllo";    // 'é' is two bytes (0xC3 0xA9)
let bad  = &s[0..2];   // panic: byte index 2 is not a char boundary
let good = &s[0..3];   // "hé"
```

For character-level work, use `s.chars()` (iterator of `char`) or the `unicode-segmentation` crate for grapheme clusters. For text processing at DB scale, you'll generally stay at the byte level and treat UTF-8 as opaque bytes except at display boundaries.

---

## 4. Deref coercion — why `&String` works where `&str` is expected

When you pass `&my_string` to a function expecting `&str`, Rust automatically calls `String::deref()` which returns `&str`. This is the **Deref coercion** mechanism, and it's why the world is pleasant:

| Owned      | Deref target     | You write              |
|------------|------------------|------------------------|
| `String`   | `&str`           | `&my_string`           |
| `Vec<T>`   | `&[T]`           | `&my_vec`              |
| `Box<T>`   | `&T`             | `&*my_box` or just `&my_box` in many contexts |
| `Rc<T>`    | `&T`             | same                   |
| `Arc<T>`   | `&T`             | same                   |

You'll see this everywhere. Just know the rule: owning types auto-deref to their non-owning view type.

---

## 5. `Vec<T>` — the growable buffer

Think of `Vec<T>` as Java's `ArrayList<T>` but value-typed and genuinely contiguous (no boxing for `Vec<i64>`).

```rust
let mut v: Vec<i64> = Vec::new();
v.push(1);
v.push(2);
v.push(3);

let v2 = vec![10, 20, 30];                  // macro
let v3 = Vec::with_capacity(1024);          // pre-allocate, no reallocation up to 1024 pushes
let v4: Vec<i64> = (0..1000).collect();     // from an iterator
```

### Capacity model

- `len()` — how many elements you have.
- `capacity()` — how many you could hold without reallocating.
- `push` grows capacity geometrically (typically ×2) when exceeded — amortized O(1).
- `reserve(n)` grows capacity to at least `len() + n` immediately.
- `shrink_to_fit()` drops excess capacity.

For DB work you almost always use `with_capacity` — you usually know the row count or chunk size up front, and realloc-mid-loop is a pointless cache flush.

---

## 6. Why slices are the columnar-DB primitive

A columnar analytical engine is, at its core, a **pipeline of operators over slices of primitive arrays**.

Imagine this query:

```sql
SELECT SUM(price) FROM orders WHERE region = 'US';
```

Column layout on disk (or in memory):

```
region: [US, EU, US, JP, US, EU, ...]       // &[RegionId]
price:  [100, 200, 50, 400, 300, 25, ...]   // &[i64]
```

Vectorized execution (simplified):

```
for each chunk of 2048 rows:
    sel: &[bool] = mask where region[i] == US
    chunk_sum  = sum_where(price[chunk], sel)
    total     += chunk_sum
```

Every arrow in that flow is a slice. Every operator is a function from `&[T]` (plus maybe a `&[bool]` or `&[u16]` selection vector) to a result. Rust's slice type is a near-perfect fit because:

- **Aliasing-free reads let the compiler SIMD-vectorize.** When the hot loop reads `&[i64]`, the borrow checker has already proven no one else is writing — so LLVM can widen loads to AVX2/NEON freely.
- **Bounds checks elide when provable.** A `for x in xs` iterator doesn't bounds-check; a `for i in 0..xs.len()` with `xs[i]` usually doesn't either (the compiler proves the index is in range).
- **Sub-columns are free.** `&prices[1024..2048]` is a pointer arithmetic operation, nothing more. Row-group streaming becomes trivial.
- **FFI-clean.** `&[T]` is the same thing C calls `(T*, size_t)`. Interop with mmap'd Parquet files, or C++ kernels, is a cast.

This is why arrow-rs and datafusion model every column as essentially `&[T]` (plus a validity bitmap). You'll do the same.

---

## 7. Updated Java → Rust translation

Extension to Lesson 1's table:

| Java                                         | Rust                               |
|----------------------------------------------|------------------------------------|
| `ArrayList<T>`                                | `Vec<T>`                           |
| `T[]` (primitive array)                       | `[T; N]` (fixed) or `Vec<T>` (dynamic) |
| `List.subList(a, b)`                          | `&v[a..b]` (zero-copy)             |
| `String`                                     | `String` (owned) or `&str` (view)  |
| `StringBuilder`                               | `String` — it *is* the builder     |
| `String.substring(a, b)` (copies)             | `&s[a..b]` (zero-copy, but UTF-8 boundaries matter) |
| `final` field                                 | default; `mut` is the opt-in       |
| `synchronized` / volatile                     | covered by `&mut` + `Mutex<T>` later |

---

## 8. Exercises

### 8.1 `sum_where` — your first vectorized kernel

Create `lesson02_slices` (or just a fresh file) and write:

```rust
fn sum_where(values: &[i64], mask: &[bool]) -> i64
```

It returns the sum of `values[i]` where `mask[i] == true`. Panic on length mismatch with `assert_eq!`.

Test:

```rust
fn main() {
    let prices = [100, 200, 300, 400, 500];
    let is_us  = [true, false, true, false, true];
    println!("{}", sum_where(&prices, &is_us));  // expect 900
}
```

Sub-questions:

1. Write the straightforward version (for-loop with index).
2. Rewrite using `iter().zip()` + `filter_map` + `sum`. Which do you find clearer?
3. What happens if you pass `&prices[..3]` and `&is_us[..5]`? Predict before running.

### 8.2 Selection vectors

Selection vectors are the other common vectorized-DB pattern (in place of a boolean mask). Write:

```rust
fn selection_vector(mask: &[bool]) -> Vec<usize>
```

that returns the indices where `mask[i] == true`. Example:

```rust
selection_vector(&[true, false, true, false, true])
// → vec![0, 2, 4]
```

Then write:

```rust
fn gather(values: &[i64], sel: &[usize]) -> Vec<i64>
```

that returns `values[sel[0]], values[sel[1]], ...`. Together these let you express `sum_where(values, mask)` as `gather(values, &selection_vector(mask)).iter().sum()` — discuss with yourself why a DB engine would sometimes prefer one representation over the other.

### 8.3 Chunked processing

Using `chunks_exact` + `.remainder()`, write:

```rust
fn sum_chunked(xs: &[i64], chunk_size: usize) -> i64
```

that sums all elements by processing `chunk_size` at a time (treat the tail with `.remainder()`). This should compute the same total as a plain `.iter().sum()`, but the structure mirrors how a vectorized engine actually loops. Time both versions on a 10M-element `Vec` and compare — the structured version should be within a few percent.

Bonus: add an inner `for &x in chunk` vs. an index-based loop `for i in 0..chunk.len()` and compare assembly (`cargo rustc --release -- --emit=asm`) if you're curious.

### 8.4 Zero-copy column split

Given a single `&[i64]` representing a column of 10 000 rows, produce an iterator of 5 non-overlapping sub-slices of 2000 rows each — **without allocating any `Vec`**. Use `chunks_exact`.

```rust
fn row_groups(col: &[i64], group_size: usize) -> impl Iterator<Item = &[i64]>
```

(We haven't done `impl Trait` yet; just trust the signature for now and return `col.chunks_exact(group_size)` adapted as needed.)

### 8.5 `&mut` exclusivity drill

Which of these compile? Predict, then run.

```rust
// (a)
let mut v = vec![1, 2, 3];
let a = &mut v[0];
let b = &mut v[1];
*a += 10;
*b += 20;
println!("{:?}", v);

// (b)
let mut v = vec![1, 2, 3];
let (left, right) = v.split_at_mut(1);
left[0] += 10;
right[0] += 20;
println!("{:?}", v);
```

Both attempt to mutate two distinct elements of a `Vec` at once. One compiles, one doesn't. Explain *why* each, and explain what `split_at_mut` gives you that plain indexing can't. (This is the same pattern you'll use to parallelize kernel operations over disjoint chunks of a column.)

### 8.6 String-side counterpart

Write:

```rust
fn count_matches(haystack: &[&str], needle: &str) -> usize
```

that returns how many entries of `haystack` equal `needle`. Test:

```rust
let names = ["alice", "bob", "alice", "carol", "alice"];
println!("{}", count_matches(&names, "alice"));  // 3
```

Sub-questions:

1. Why is the input `&[&str]` and not `&[String]`? Which would be cheaper to construct from a column of string data?
2. Replace `==` with case-insensitive compare. What changes — and what does this hint about real DB string handling (hint: ICU, collations, normalization)?

### 8.7 Parsing teaser — `&str` slicing

Given `"id=42;name=alice;region=us"`, write code that splits on `;` and then on `=` to build a `Vec<(&str, &str)>`. (Hint: `.split(';')` and `.split_once('=')`.) Notice that the resulting `&str` values borrow from the original string — no allocation. Ask yourself: what lifetime constraint must that `Vec<(&str, &str)>` have relative to the source string? (We'll formalize this in Lesson 3.)

---

## 9. Self-check

You should be able to answer these without looking back:

1. What does a slice `&[T]` actually contain at the machine level?
2. Why is `&[T]` preferred over `&Vec<T>` in function signatures?
3. What's the difference between `xs.iter()`, `xs.iter_mut()`, and `xs.into_iter()` — which one moves the vector?
4. Why does slicing a `String` require you to land on a char boundary?
5. What does `v.chunks_exact(n)` give you that a manual `for i in (0..v.len()).step_by(n)` doesn't?
6. Given a column of 1M `i64`s, describe — in slice operations only — how you'd sum the elements where a parallel `&[bool]` mask is true, processing 2048 rows at a time.

If any of those are fuzzy, go back and re-read the relevant section. Otherwise: Lesson 3 is **traits, generics, and lifetimes** — the machinery for writing one `sum_where` that works over any numeric type, and the formal answer to "what does the `&str` in question 8.7 actually live for."
