# Lesson 1 — Ownership & Borrowing

> Where your GC went, and why the compiler is going to yell at you for a week.

---

## Objectives

By the end of this lesson you should be able to:

- State the ownership rule in one sentence.
- Predict whether a given assignment is a **move** or a **copy**.
- Use `&T` and `&mut T` correctly in function signatures.
- Explain why an "obviously fine" piece of Java-flavored code fails to compile in Rust.
- Connect all of the above to a concrete class of bugs (use-after-free, iterator invalidation, data races).

---

## 1. The one rule

> **Every value has exactly one owner. When the owner goes out of scope, the value is dropped — memory freed, files closed, locks released.**

Everything else in Rust — borrowing, lifetimes, `&`, `&mut`, `Box`, `Arc`, `Rc` — is machinery for bending that rule safely.

There is no garbage collector. There is no `free()`. Drops are deterministic and happen at a compile-time-known point.

---

## 2. Move semantics

In Java, `String b = a;` aliases: both `a` and `b` refer to the same object; GC frees it later.

In Rust:

```rust
fn main() {
    let a = String::from("hello");
    let b = a;                // MOVE — ownership transfers a → b
    println!("{}", a);        // compile error: borrow of moved value: `a`
}
```

Error:

```
error[E0382]: borrow of moved value: `a`
```

### Why move instead of copy?

A `String` is `(ptr, len, capacity)` — 24 bytes on the stack plus a heap buffer. Copying the struct byte-for-byte would give two owners of the *same* heap buffer → double-free. Deep-copying the buffer would be a silent, expensive surprise. Rust picks the third option: **transfer ownership.** No aliasing, no double-free, no surprise allocation.

To deep-copy explicitly, call `a.clone()`. To share without transferring ownership, **borrow** (§4).

---

## 3. Copy vs Move

Types that are trivially bit-copyable and own no external resources implement the `Copy` trait. Assignment and function-argument passing **copy** them; the original stays usable.

```rust
fn main() {
    let x: i32 = 5;
    let y = x;
    println!("{} {}", x, y);   // fine, both valid
}
```

| Copy (bit-copied)                               | Move (ownership transfers)                  |
|-------------------------------------------------|---------------------------------------------|
| `i32`, `u64`, `f64`, `bool`, `char`             | `String`                                    |
| `&T` (shared references)                        | `Vec<T>`, `HashMap<K, V>`                   |
| Fixed arrays `[T; N]` where `T: Copy`           | `Box<T>`, `Rc<T>`, `Arc<T>`                 |
| Tuples of `Copy` types                          | `File`, `TcpStream`, anything holding an OS resource |

**Mental shift from Java:** in Java *everything non-primitive* is a reference. In Rust *everything is a value by default*, and the compiler tracks ownership of it. References (`&T`, `&mut T`) are a separate, explicit thing.

---

## 4. Borrowing

Moving everywhere is unusable. Borrowing lets a function read or modify a value without taking ownership.

```rust
fn length(s: &String) -> usize {       // shared borrow
    s.len()
}

fn push_world(s: &mut String) {        // exclusive borrow
    s.push_str(", world");
}

fn main() {
    let mut a = String::from("hello");
    let n = length(&a);                // pass &a
    push_world(&mut a);                // pass &mut a
    println!("{} ({} → {})", a, n, a.len());
}
```

Key points:

- `&T` — **shared** reference, read-only, non-owning.
- `&mut T` — **exclusive** reference, read-write, non-owning.
- Neither drops the value; only the owner does.
- `mut` at the `let` site is required before you can take `&mut`.

---

## 5. The two borrow rules (memorize)

At any point in time, for any given value, you may have **either**:

- **any number of shared references** `&T`, **or**
- **exactly one exclusive reference** `&mut T`.

Never both. Never two `&mut`. This one rule is what:

- Prevents data races at compile time.
- Prevents iterator invalidation at compile time.
- Lets the compiler safely auto-vectorize (SIMD) and alias-analyze your hot loops.
- Makes `unsafe { get_unchecked(i) }` realistic — the surrounding safe code proves no one else is touching the buffer.

---

## 6. Non-Lexical Lifetimes (NLL)

A borrow's lifetime ends at its **last use**, not at the enclosing `}`.

```rust
let mut v = vec![1, 2, 3];
let r = &v;
println!("{:?}", r);     // last use of r — borrow ends HERE
v.push(4);               // fine: v is freely mutable again
```

Flip the order:

```rust
let mut v = vec![1, 2, 3];
let r = &v;
v.push(4);               // ERROR: r is still live below
println!("{:?}", r);
```

The same two statements, different order → compiles vs. doesn't. The compiler does flow analysis to compute borrow extents; pre-NLL Rust (before 2018) forced you to add manual scope blocks. Don't memorize anything about this; just know the compiler is smarter than strict lexical scoping suggests.

---

## 7. Why this matters for a database

The bugs Rust eliminates at compile time are exactly the ones that plague C-family DB engines:

| Bug class                        | How it happens                                           | Rust's answer                              |
|----------------------------------|-----------------------------------------------------------|---------------------------------------------|
| Use-after-free                   | Pointer into a buffer that gets reallocated/evicted      | `&T` outliving its referent → compile error |
| Iterator invalidation            | Mutating a collection while iterating                    | `&mut` + `&` can't coexist → compile error  |
| Data race                        | Two threads, one writes, no sync                         | `&mut` is exclusive even across threads     |
| Double-free                      | Two owners free the same allocation                      | Single-owner rule → impossible in safe Rust |

You *will* drop into `unsafe` to build the guts of a DB engine (custom allocators, SIMD, mmap'd pages, raw column buffers). But the safe shell around the unsafe core is what keeps the whole system tractable. DuckDB and ClickHouse spend significant effort enforcing these invariants in code review and TSAN/ASAN runs; in Rust the compiler does it for you.

---

## 8. Java → Rust mental model, one screen

| Java                                                    | Rust                                             |
|---------------------------------------------------------|---------------------------------------------------|
| References are cheap; aliasing is the default           | Values are the default; aliasing is tracked      |
| GC decides when memory is freed                         | Scope decides; drops are deterministic           |
| `null` is a valid reference                             | No null — use `Option<T>`                        |
| `Object x = y` aliases                                  | `let x = y` moves (or copies, if `T: Copy`)      |
| `ArrayList<T>` is reference-typed                       | `Vec<T>` is value-typed, owns its buffer         |
| `ConcurrentModificationException` at runtime            | Borrow checker forbids at compile time           |
| Visibility via `public/private`                         | Visibility via `pub` on items + module tree      |
| `interface`                                              | `trait` (more powerful; we'll get there)         |
| `try/catch`                                              | `Result<T, E>` + `?` operator (we'll get there)  |

---

## 9. Exercises — completed in session

### 9.1 Predict A, B, C

```rust
// A
let mut v = vec![1, 2, 3];
let r = &v;
v.push(4);
println!("{:?}", r);

// B
let mut v = vec![1, 2, 3];
let r = &v;
println!("{:?}", r);
v.push(4);
println!("{:?}", v);

// C
let s = String::from("hi");
let t = s.clone();
println!("{} {}", s, t);
```

**Answers:**

- **A — compile error.** `r` is a live shared borrow when `v.push(4)` demands an exclusive borrow. Rule violated: can't hold `&` and `&mut` at the same time. The deeper reason: `push` may reallocate the buffer, which would turn `r` into a dangling pointer. Rust refuses.
- **B — compiles and prints `[1, 2, 3]` then `[1, 2, 3, 4]`.** NLL: `r`'s last use is the first `println!`, so the borrow ends there. The subsequent `push` sees no outstanding shared borrows.
- **C — compiles and prints `hi hi`.** `clone()` deep-copies the heap buffer; `s` and `t` are independent owners, each with their own `(ptr, len, cap)`.

### 9.2 `&v[0]` vs `v[0]`

```rust
// A — does not compile
let mut v = vec![1, 2, 3];
let first = &v[0];
v.push(4);
println!("first = {}", first);

// B — compiles
let mut v = vec![1, 2, 3];
let first: i32 = v[0];
v.push(4);
println!("first = {}", first);
```

**Answers:**

- **A** fails for the same reason as 9.1.A: `&v[0]` is a borrow into `v`'s heap buffer, held across a mutating call that may reallocate.
- **B** works because `i32: Copy`. Indexing `v[0]` internally produces a `&i32`, but the `let first: i32 = ...` binding copies the value out; the temporary reference dies on that line, and `push` sees no outstanding borrows.

---

## 10. Additional exercises

Work through these before Lesson 2. For each, **predict first**, then run. If your prediction was wrong, figure out why before reading any hint.

### 10.1 Warm-up — ownership into a function

```rust
fn take(s: String) {
    println!("took {}", s);
}

fn main() {
    let a = String::from("hello");
    take(a);
    println!("{}", a);   // ???
}
```

- Will this compile? If not, what's the minimal change to make it compile while still printing `hello` at the end?
- Give **two** different fixes. (Hint: one uses `.clone()`; the other uses `&`.)

### 10.2 Return ownership

Write a function with signature:

```rust
fn shout(s: String) -> String
```

that takes ownership of `s`, appends `"!!!"`, and returns the new string. Then call it from `main` and print the result. Do **not** use `clone()`. (Hint: mutate in place with `push_str`, then return.)

### 10.3 Borrow checker drill

Predict whether each snippet compiles. Explain *why* in one sentence each.

```rust
// (a)
let mut s = String::from("hi");
let r1 = &s;
let r2 = &s;
println!("{} {}", r1, r2);

// (b)
let mut s = String::from("hi");
let r1 = &s;
let r2 = &mut s;
println!("{} {}", r1, r2);

// (c)
let mut s = String::from("hi");
let r1 = &mut s;
let r2 = &mut s;
r1.push('!');
r2.push('?');

// (d)
let mut s = String::from("hi");
{
    let r1 = &mut s;
    r1.push('!');
}
let r2 = &mut s;
r2.push('?');
println!("{}", s);
```

### 10.4 Iterator invalidation, Rust-style

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    for x in &v {
        if *x == 3 {
            v.push(99);            // ???
        }
        println!("{}", x);
    }
}
```

- Does this compile?
- Write the equivalent loop in Java's `ArrayList` in your head — when and how would it fail there? Contrast with Rust's answer.

### 10.5 DB-flavored — aggregate over a slice

Write a function:

```rust
fn min_max(xs: &[i64]) -> Option<(i64, i64)>
```

that returns `None` if `xs` is empty, and otherwise returns `Some((min, max))`. Call it from `main` on:

```rust
let prices = [100, 200, 50, 400, 300];
let empty: [i64; 0] = [];
```

Two sub-questions:

1. Why is the return type `Option<(i64, i64)>` rather than `(i64, i64)`? What would Java typically do here, and why is the Rust version preferable?
2. Can you write this using only iterator methods (`iter().min()`, `iter().max()`), no explicit `for` loop? What's the tradeoff vs. a single-pass for-loop?

### 10.6 Lifetime teaser (don't solve — just predict the error)

```rust
fn longest(a: &str, b: &str) -> &str {
    if a.len() >= b.len() { a } else { b }
}
```

- Does this compile?
- Without looking it up: what question do you think the compiler is going to ask you?

(We'll answer this properly in Lesson 3 when we do lifetimes explicitly. For now just notice the question the compiler raises.)

---

## 11. Self-check

You should be able to answer these without looking back:

1. What does `let b = a;` do when `a: String`? What does it do when `a: i32`?
2. State the two borrow rules.
3. Why does `v.push(x)` invalidate a `&v[0]` borrow — what is the compiler actually worried about?
4. What's the difference between `String` and `&str`?
5. What's the idiomatic function signature for "takes a read-only view of a bunch of integers" — `&Vec<i32>` or `&[i32]`? Why?

If any of these feel shaky, re-read the relevant section. Otherwise: on to Lesson 2 (slices, `&str`, and building toward columnar operators).
