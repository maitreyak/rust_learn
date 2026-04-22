# Lesson 3 — Traits, Generics, and Lifetimes

> Java interfaces + C++ templates + a type-level proof system, all fused. The lesson where Rust stops looking like Java and starts looking like itself.

---

## Objectives

- Write generic functions and types with trait bounds.
- Define and implement traits, including default methods and associated types.
- Pick between generics (static dispatch) and `dyn Trait` (dynamic dispatch) with a real reason.
- Use key stdlib traits fluently: `Copy`, `Clone`, `Debug`, `Display`, `PartialEq`, `Eq`, `Hash`, `PartialOrd`, `Ord`, `Default`, `From`/`Into`, `Iterator`, `IntoIterator`, `AsRef`, `Deref`.
- Read and write lifetime annotations; understand elision rules.
- Explain `'static` precisely.
- Know when to use `impl Trait` in argument and return positions.

---

## 1. Generics

Type parameters in angle brackets. Essentially identical in shape to Java generics or C++ templates.

```rust
fn max_of<T: PartialOrd>(a: T, b: T) -> T {
    if a >= b { a } else { b }
}

fn main() {
    println!("{}", max_of(3, 7));         // i32
    println!("{}", max_of(3.2, 2.9));     // f64
    println!("{}", max_of("bar", "baz")); // &str
}
```

The bound `T: PartialOrd` says "T must implement the `PartialOrd` trait." Without it, `a >= b` wouldn't compile — unlike Java, Rust does **not** let you call `equals` or `compareTo` on a bare `T`. Every capability you need must appear as a bound.

**Monomorphization.** The compiler generates a separate specialized copy of `max_of` for each concrete `T` you call it with — `max_of::<i32>`, `max_of::<f64>`, etc. This is exactly like C++ templates. Pros: zero runtime overhead, aggressive inlining. Cons: binary bloat if overused.

**Multiple bounds:**

```rust
fn print_and_compare<T: PartialOrd + std::fmt::Debug>(a: T, b: T) {
    println!("{:?} vs {:?} -> {}", a, b, a > b);
}
```

**`where` clauses** — same thing, more readable when bounds get messy:

```rust
fn median<T>(xs: &mut [T]) -> T
where
    T: Copy + PartialOrd,
{
    xs.sort_by(|a, b| a.partial_cmp(b).unwrap());
    xs[xs.len() / 2]
}
```

---

## 2. Traits

A trait is a named set of methods that types can implement. Java's closest analogue is `interface`, but traits are more powerful in three ways:

- Traits can provide **default method implementations**.
- Traits can have **associated types**.
- Traits can be implemented for types you don't own (subject to the orphan rule).

```rust
trait Summary {
    // Required methods (no body).
    fn title(&self) -> &str;

    // Default method — implementors may override but don't have to.
    fn summary(&self) -> String {
        format!("[{}]", self.title())
    }
}

struct Article { title: String, body: String }

impl Summary for Article {
    fn title(&self) -> &str { &self.title }
    // summary() uses the default implementation.
}

fn main() {
    let a = Article { title: "rust 2026".into(), body: "...".into() };
    println!("{}", a.summary());  // [rust 2026]
}
```

### Orphan rule

You may `impl Trait for Type` only if **either the trait or the type is local to your crate**. This prevents two unrelated crates from both implementing `Display for Vec<T>` and having the linker pick one randomly. Workaround when you really need it: the **newtype pattern** — wrap the foreign type in your own struct.

### Associated types

An associated type is a type parameter that belongs to *one* impl — the impl picks what it is. The canonical example is `Iterator`:

```rust
trait Iterator {
    type Item;                        // associated type
    fn next(&mut self) -> Option<Self::Item>;
}
```

For each type that implements `Iterator`, `Item` is pinned to one specific type (`i32`, `&str`, whatever). Contrast with generic parameters on traits (`trait Iterator<T>`), which would let the same type impl `Iterator<i32>` AND `Iterator<i64>` — almost never what you want. Associated types are the right tool for "one-to-one" relationships.

---

## 3. Generics vs `dyn Trait` — static vs dynamic dispatch

You have two ways to abstract over "any T that implements this trait":

```rust
// Static dispatch: monomorphized, one copy per T. Zero overhead, more code.
fn render_static<T: Summary>(x: &T) {
    println!("{}", x.summary());
}

// Dynamic dispatch: one copy, vtable lookup at call site. Indirection, less code.
fn render_dyn(x: &dyn Summary) {
    println!("{}", x.summary());
}
```

A `&dyn Summary` is a **fat pointer** — two words: `(data_ptr, vtable_ptr)`. Same layout as `&[T]` but the second word is a vtable instead of a length. Method calls through `dyn` go through the vtable — indirect call, no inlining.

**When to pick which:**

- **Generics when** the hot loop calls the method many times (inlining matters) or the set of types is known statically.
- **`dyn` when** you need heterogeneous collections (`Vec<Box<dyn Operator>>`) or to avoid monomorphization bloat in cold code.

For a DB: physical operators in a query plan are usually `Box<dyn Operator>` — you build the plan with a tree of heterogeneous node types — but the inner per-tuple kernels are generic to enable SIMD and inlining.

---

## 4. Key stdlib traits tour

The traits you'll see daily. Know what each marks.

| Trait               | Purpose                                                   | Derive? |
|---------------------|-----------------------------------------------------------|---------|
| `Copy`              | Bit-copy instead of move on assignment                    | Yes     |
| `Clone`             | Explicit deep copy via `.clone()`                         | Yes     |
| `Debug`             | `{:?}` formatting (engineer-facing)                       | Yes     |
| `Display`           | `{}` formatting (user-facing) — **no derive**             | No      |
| `PartialEq` / `Eq`  | `==` / `!=`. `Eq` adds reflexivity (no `NaN`)            | Yes     |
| `PartialOrd` / `Ord`| `<`, `>`, `.cmp()`. `Ord` is total                       | Yes     |
| `Hash`              | Hashable for `HashMap`, `HashSet`                         | Yes     |
| `Default`           | `T::default()` — a "zero" value                           | Yes     |
| `From<U>` / `Into<U>` | Infallible conversion. Impl `From`, get `Into` free    | No      |
| `TryFrom` / `TryInto` | Fallible conversion                                     | No      |
| `AsRef<T>`          | Cheap view conversion (`&String → &str`, etc.)            | No      |
| `Deref`             | `*x` and auto-deref (think "smart pointer → inner")       | No      |
| `Iterator`          | Produces a sequence                                       | No      |
| `IntoIterator`      | Can be converted into an iterator (what `for` loops use)  | No      |
| `Drop`              | Destructor — called on scope exit                         | No      |
| `Send` / `Sync`     | Auto-traits: safe to move/share across threads (Lesson 8) | Auto    |

**Deriving:**

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Row { id: u64, name: String }
```

One line, five traits, no manual boilerplate. You'll see `#[derive(...)]` on most structs.

### `From` / `Into` — the conversion idiom

```rust
impl From<&str> for Row {
    fn from(s: &str) -> Row {
        let mut parts = s.splitn(2, ',');
        let id: u64 = parts.next().unwrap().parse().unwrap();
        let name = parts.next().unwrap().to_string();
        Row { id, name }
    }
}

fn main() {
    let r: Row = "42,alice".into();       // uses Into, which comes free from From
    let r2 = Row::from("7,bob");
}
```

Whenever you write a constructor-like `fn`, consider whether `From` fits — it chains nicely with `?` for error propagation in Lesson 4.

---

## 5. Lifetimes — why they exist

Every reference in Rust has a lifetime: the compile-time interval during which it's valid. Most of the time they're inferred (elided). Sometimes you have to write them.

### The problem lifetimes solve

```rust
fn longest(a: &str, b: &str) -> &str {   // won't compile — teaser from Lesson 1
    if a.len() >= b.len() { a } else { b }
}
```

Error: "missing lifetime specifier." The compiler's question: the returned `&str` is valid for how long? As long as `a`? As long as `b`? As long as both? You have to tell it.

### The annotation

```rust
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}
```

Read: "there exists some lifetime `'a` such that both inputs are valid for `'a` and the output is valid for `'a`." The compiler picks the **intersection** of the two input lifetimes — the shortest interval where both are valid — and guarantees the output reference doesn't outlive that interval.

Lifetimes are **not run-time values**. They're type-level labels the borrow checker uses to prove references don't dangle. Monomorphization erases them entirely; they're zero runtime cost.

### Elision rules

Most function signatures don't need explicit lifetimes because the compiler applies three elision rules:

1. Each input reference gets its own lifetime: `fn f(a: &A, b: &B)` → `fn f<'a, 'b>(a: &'a A, b: &'b B)`.
2. If there's exactly one input lifetime, it's assigned to all output references.
3. If one of the inputs is `&self` or `&mut self`, its lifetime is assigned to all output references.

That's why `fn first_word(s: &str) -> &str` needs no explicit `'a` — rule 2 applies. `longest` takes two references, so rule 2 doesn't apply, and you have to write it.

---

## 6. Struct lifetimes

If a struct holds a reference, you must say so:

```rust
struct ColumnRef<'a> {
    name: &'a str,
    data: &'a [i64],
}

impl<'a> ColumnRef<'a> {
    fn len(&self) -> usize { self.data.len() }
}
```

A `ColumnRef<'a>` is valid only as long as both of its borrowed fields are. You can't have a `ColumnRef` outlive the data it points into. This is exactly the constraint you'd want for a zero-copy view into a columnar buffer.

### Multiple lifetimes

```rust
struct BiRef<'short, 'long: 'short> {
    brief: &'short str,
    lasting: &'long str,
}
```

The `'long: 'short` bound reads "'long outlives 'short." This shows up when you have one reference that must be guaranteed to live at least as long as another.

---

## 7. `'static` — precisely

`'static` is the lifetime that covers the entire program. Two very different things get this label:

- **String literals** and other compile-time data: `"hello"` has type `&'static str` — it points into the binary's read-only section, which exists for the whole program.
- **Owned heap data with no non-`'static` references inside it**: a `String` contains no references, so it satisfies `T: 'static`. Same for `Vec<i64>`, `Arc<Row>`, etc.

The second case surprises Java devs. When a bound says `T: 'static`, it doesn't mean "T is statically allocated." It means "T doesn't borrow anything that could expire." A `String` fits; `&str` (with a non-static lifetime) doesn't.

This matters for `std::thread::spawn`, which requires its closure to be `'static` — meaning the closure can't capture any references whose scope is the current function.

---

## 8. `impl Trait`

Syntactic sugar for "some type implementing this trait, but I don't want to name it."

**Argument position** — equivalent to a generic bound, shorter:

```rust
fn print_all(xs: impl IntoIterator<Item = i64>) {
    for x in xs { println!("{}", x); }
}

// Desugars to:
fn print_all<I: IntoIterator<Item = i64>>(xs: I) { ... }
```

**Return position** — hides the concrete type. The real type (usually a messy iterator adapter chain) stays opaque:

```rust
fn evens_up_to(n: i64) -> impl Iterator<Item = i64> {
    (0..n).filter(|x| x % 2 == 0)
}
```

Without `impl Trait`, the return type would be something like `Filter<Range<i64>, {closure}>` — a type you can't spell. `impl Iterator<Item = i64>` papers over that.

**Restriction:** `impl Trait` in return position hides exactly *one* concrete type. If your function returns different concrete types on different branches, use `Box<dyn Iterator<Item = i64>>` instead.

---

## 9. DB relevance — generic kernels

The point of a vectorized DB's kernel layer is to write an operation once and have it compiled specifically for every primitive type. In Rust, that looks like:

```rust
use std::ops::Add;

fn sum_column<T>(xs: &[T]) -> T
where
    T: Copy + Add<Output = T> + Default,
{
    let mut total = T::default();
    for &x in xs {
        total = total + x;
    }
    total
}

fn main() {
    let a: Vec<i32> = (0..1000).collect();
    let b: Vec<f64> = (0..1000).map(|x| x as f64).collect();
    println!("{}", sum_column(&a));     // monomorphized to sum_column::<i32>
    println!("{}", sum_column(&b));     // monomorphized to sum_column::<f64>
}
```

The compiler produces two separate, fully-inlined, SIMD-vectorizable functions. No boxing, no dyn dispatch. This is how `arrow-rs` and `datafusion` express kernels internally: generic over numeric types, specialized at compile time, competitive with hand-written C++.

Contrast with Java: `SumColumn<T extends Number>` would box each element to `Integer` or `Double` and iterate via virtual method calls on `Number.doubleValue()`. Panama foreign memory + JIT specialization can get close, but it's a multi-year effort to catch up to what Rust does by default.

---

## 10. Java → Rust translation (extended)

| Java                                              | Rust                                    |
|---------------------------------------------------|-----------------------------------------|
| `interface Comparable<T>`                          | `trait Ord` / `trait PartialOrd`        |
| `implements Comparable<Foo>`                       | `impl Ord for Foo`                      |
| Default interface method (Java 8+)                 | Default trait method                    |
| `class Foo<T extends Bar>`                         | `struct Foo<T: Bar> { ... }`            |
| Type erasure at runtime                            | Monomorphization — concrete at compile time |
| `List<?>` (wildcard)                               | `&dyn Trait` / `impl Trait`             |
| `ArrayList<Integer>` (boxed)                       | `Vec<i32>` (unboxed, contiguous)        |
| `equals`, `hashCode`                               | `PartialEq`, `Hash` (derived)           |
| `toString`                                         | `Display` (for users) / `Debug` (for devs) |
| `implements Iterator<T>`                           | `impl Iterator for Foo { type Item = T; ... }` |
| `final class`                                      | — (no inheritance to prevent; compose with traits) |
| `@FunctionalInterface`                             | closure types `Fn` / `FnMut` / `FnOnce` |

---

## 11. Exercises

### 11.1 Resolve the teaser

Write `longest` from Lesson 1 §10.6 correctly, with explicit lifetimes. Confirm it compiles. Then write this variant and explain the compile error in one sentence:

```rust
fn longest_first_word<'a>(s: &'a str, other: &str) -> &'a str {
    let first_of_other = other.split_whitespace().next().unwrap();
    if s.len() >= first_of_other.len() { s } else { first_of_other }
}
```

### 11.2 Generic sum_where

Generalize your Lesson 2 `sum_where` to any numeric type:

```rust
fn sum_where<T>(values: &[T], mask: &[bool]) -> T
where
    T: Copy + std::ops::Add<Output = T> + Default,
{
    // your code
}
```

Call it with `i32`, `f64`, and `i64`. Confirm each monomorphized version compiles and runs.

### 11.3 Define a trait

Define:

```rust
trait Column {
    type Value: Copy;
    fn len(&self) -> usize;
    fn get(&self, i: usize) -> Self::Value;
}
```

Implement it for `Vec<i64>`, `Vec<f64>`, and `&[i32]`. Write a generic function `fn sum_any<C: Column>(col: &C) -> C::Value where C::Value: std::ops::Add<Output = C::Value> + Default`. Call it on one of each.

### 11.4 Trait objects

Convert:

```rust
enum Shape { Circle(f64), Square(f64) }
fn area(s: &Shape) -> f64 { ... }
```

to use a `trait Shape { fn area(&self) -> f64; }` with `impl` blocks for `Circle` and `Square` structs. Then build a `Vec<Box<dyn Shape>>` containing both, and sum their areas. Note where you use `dyn` vs. where you could have used a generic.

### 11.5 Default and `From`

Derive `Default` for this:

```rust
#[derive(Default, Debug)]
struct Config {
    batch_size: usize,
    parallelism: usize,
    enable_cache: bool,
}
```

Then implement `From<(usize, usize)>` so `Config::from((2048, 8))` gives you a `Config` with the given fields and default `enable_cache`. Use `..Default::default()` in the impl body.

### 11.6 Lifetime elision — predict

For each signature, state whether lifetimes are elided, and if so, what the "unelided" signature looks like:

```rust
fn f1(s: &str) -> &str
fn f2(a: &str, b: &str) -> &str
fn f3(&self, other: &str) -> &str
fn f4(s: &str) -> (&str, &str)
```

Which ones will the compiler accept without explicit `'a`?

### 11.7 `impl Trait` discipline

Write:

```rust
fn chunks_selective(xs: &[i64], stride: usize) -> impl Iterator<Item = &[i64]>
```

that returns every second chunk of size `stride`. Try it with `for group in chunks_selective(&v, 256)`. Then — why can't you write this the same way if the function sometimes returns `chunks_exact` and sometimes returns `windows`, even though both are iterators of `&[i64]`?

---

## 12. Self-check

1. When does the compiler require you to write a trait bound, and what is it protecting against?
2. What's the runtime representation of `&dyn Trait`?
3. What does the compiler do differently for `fn f<T: Display>(x: T)` vs. `fn f(x: &dyn Display)`?
4. Why does `longest(a: &str, b: &str) -> &str` need an explicit lifetime but `first_word(s: &str) -> &str` doesn't?
5. What does `T: 'static` mean? (Hint: not what it sounds like.)
6. Why does the orphan rule exist, and when does it bite you?
7. When would you reach for `Box<dyn Trait>` over a generic parameter, and vice versa?

Next: **Lesson 4 — Error handling.** No more `unwrap()` everywhere.
