# Lesson 9 — Unsafe Rust

> The part where Rust stops holding your hand. Raw pointers, FFI, SIMD, manual memory layout. What a DB engine spends real effort on.

---

## Objectives

- Name the five things `unsafe` allows that safe Rust doesn't.
- Read and write raw pointer code without invoking undefined behavior.
- Understand the aliasing rules (Stacked Borrows / Tree Borrows).
- Use `MaybeUninit<T>` to work with uninitialized memory safely.
- Write FFI bindings with `extern "C"` and `#[repr(C)]`.
- Use SIMD via `std::arch` intrinsics and `std::simd`.
- Encapsulate unsafe internals behind safe APIs.
- Run your code under **Miri** to catch UB.

---

## 1. The five powers of `unsafe`

`unsafe` is a contract: "I've proven what I'm about to do is sound; compiler, trust me." Inside an `unsafe { ... }` block, you may:

1. **Dereference a raw pointer** (`*const T`, `*mut T`).
2. **Call an `unsafe fn`** — one whose safety depends on the caller upholding invariants.
3. **Read or write a mutable static** (`static mut X: ...`).
4. **Implement an `unsafe trait`** — one whose correctness relies on implementation invariants (e.g., `Send`, `Sync`).
5. **Access a union field** — because the compiler can't tell which variant is valid.

Outside those five, `unsafe` does nothing. It is NOT:

- A performance hint. The compiler already optimizes safe code aggressively.
- A borrow-check escape hatch for general code. Use it only for the five things above.
- A license to do anything. **Undefined behavior is still forbidden**, whether you wrote `unsafe` or not.

The safe code around an `unsafe` block must be organized such that the unsafe block's preconditions are always met. The `unsafe` keyword marks *where* you're doing something dangerous, not *that* the code is sloppy.

---

## 2. Raw pointers

```rust
let x = 42;
let p: *const i32 = &x;       // raw pointer (from reference — no unsafe needed)
let q: *mut i32  = &mut 7 as *mut i32;

unsafe {
    println!("{}", *p);        // deref — unsafe
}
```

Differences from references:

- **No lifetime.** The compiler tracks nothing about how long a raw pointer is valid.
- **No aliasing rules.** Multiple `*mut T` to the same location is permitted (but mis-using them is UB).
- **No null check.** `*const T` can be null; dereferencing null is UB.
- **May be misaligned.** References must be aligned; raw pointers don't have to be.
- **Not automatically dropped.** Owning a raw pointer doesn't free anything.

You'll see three ways to get raw pointers:

```rust
let p1: *const i32 = &x;                       // from a reference
let p2: *const i32 = x.as_ptr();               // from a slice or Vec or similar
let p3: *mut i32   = std::ptr::null_mut();     // explicit null
```

### Common operations

```rust
use std::ptr;

// Safe to obtain; unsafe to deref.
let p: *const i64 = data.as_ptr();

unsafe {
    // Equivalent to `*p`, but more explicit:
    let v: i64 = ptr::read(p);

    // Offset — same arithmetic as array indexing:
    let v2: i64 = ptr::read(p.add(1));          // p + sizeof(i64)

    // Write through a *mut:
    ptr::write(p as *mut i64, 42);

    // Bulk copy:
    ptr::copy_nonoverlapping(src, dst, count);
}
```

`std::ptr` is the namespace for low-level pointer ops. `add` and `offset` move by `T`-sized steps; `byte_add` moves by bytes.

---

## 3. Undefined behavior

Undefined behavior (UB) in Rust means the compiler is allowed to assume it doesn't happen. If it does, your program has no defined meaning — anything may result, including results that only show up in release mode, under specific codegen, on specific CPUs.

Categories of UB you must avoid even inside `unsafe`:

- **Dereferencing an invalid pointer.** Null, dangling, past-the-end, misaligned for the type.
- **Violating aliasing rules.** Having a `&mut T` and any other access (through a `&T`, `*mut T`, or another `&mut T`) to the same location overlap in time.
- **Producing an invalid value.** A `bool` with bit pattern 3, a `char` that isn't a valid Unicode scalar, a reference that's null, uninitialized memory read as any type.
- **Data races.** Two unsynchronized accesses to the same memory where at least one is a write.
- **Breaking an `unsafe fn`'s documented preconditions.**

**UB is not "crashes" or "wrong output."** UB is "the compiler was allowed to reorder, eliminate, or misinterpret your code because it proved (incorrectly, given your UB) that something couldn't happen." Symptoms are often far from the cause.

---

## 4. The aliasing model — Stacked Borrows / Tree Borrows

Rust's rule that "at any time, for any value, you have any number of `&T` OR exactly one `&mut T`" extends to raw pointers through a set of formal models called **Stacked Borrows** (2019) and its successor **Tree Borrows** (2023+, likely stabilizing soon).

The short version:

- An `&mut T` you create "takes exclusive control" of the referenced location for its lifetime. Any access through another pointer (even a raw one) you derived from a different parent is UB.
- An `&T` you create allows shared reads. Any write through a derived pointer is UB.
- Converting between references and raw pointers is fine; the aliasing tags travel with the derivation chain.

**Practical consequence:** if you're passing a `*mut T` around, don't also hold a live `&mut T` to the same object. Pick one tool and use it consistently for a given region of code.

**Tool:** run your code under [Miri](#11-miri) to actually check these rules.

---

## 5. `MaybeUninit<T>` — working with uninitialized memory

Sometimes you need to allocate a buffer and fill it in stages. Creating a `T` whose bytes are uninitialized is UB — so you can't write `let x: i64; ... x = 42;` in a pattern the compiler can't prove safe.

`MaybeUninit<T>` is the escape hatch:

```rust
use std::mem::MaybeUninit;

fn allocate_filled<T: Copy>(n: usize, value: T) -> Vec<T> {
    let mut v: Vec<MaybeUninit<T>> = Vec::with_capacity(n);
    unsafe {
        v.set_len(n);
        for i in 0..n {
            v[i].write(value);
        }
        std::mem::transmute::<Vec<MaybeUninit<T>>, Vec<T>>(v)
        // safe because MaybeUninit<T> has the same layout as T
        // and we've initialized all n elements
    }
}
```

Usage pattern:

1. Allocate `MaybeUninit<T>` (which is valid for any bit pattern, including all-zero).
2. Fill with `.write(value)`.
3. Once fully initialized, `assume_init()` or transmute into `T`.

Skipping step 2 for any element and then reading it — or transmuting while any slot is uninitialized — is UB.

---

## 6. `transmute` — rarely, with extreme care

`std::mem::transmute::<A, B>(x)` reinterprets bytes. Use it only when:

- `size_of::<A>() == size_of::<B>()` (compile-time checked).
- The bit pattern of `x` is valid for `B`.
- There's no safer alternative (`From`/`Into`, `as` casts, `bytemuck::cast`, `u32::to_le_bytes`).

```rust
// Fine, rare example: converting a fn pointer to a usize for hashing.
let f: fn(i32) -> i32 = |x| x + 1;
let addr: usize = unsafe { std::mem::transmute(f) };
```

For byte-level casts between POD types, the `bytemuck` crate gives you safe, derive-based alternatives:

```rust
use bytemuck::{Pod, Zeroable};

#[derive(Copy, Clone, Pod, Zeroable)]
#[repr(C)]
struct Record { id: u64, val: f64 }

let r = Record { id: 7, val: 3.14 };
let bytes: [u8; 16] = bytemuck::cast(r);            // safe
let r2: Record      = bytemuck::cast(bytes);        // safe
```

Prefer `bytemuck` for anything involving byte reinterpretation in a DB engine (row encoding, hash inputs, compressed column readers).

---

## 7. FFI — `extern "C"` and `#[repr(C)]`

Calling C or being called by C:

```rust
// Declare an external C function.
extern "C" {
    fn abs(x: i32) -> i32;
}

// Export a Rust function with C ABI.
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 { a + b }

fn main() {
    unsafe {
        println!("{}", abs(-42));
    }
}
```

### `#[repr(C)]`

By default, Rust may reorder struct fields. For FFI (and for anywhere layout matters — disk formats, wire protocols, memory maps), use `#[repr(C)]` to get C-compatible layout:

```rust
#[repr(C)]
struct Header {
    magic:   u32,
    version: u16,
    flags:   u16,
    length:  u64,
}
```

Field order, padding, and alignment match C. Combined with `bytemuck::Pod + Zeroable`, this makes safe byte-level casts possible.

### Other `#[repr]`s

- `#[repr(C)]` — C layout.
- `#[repr(transparent)]` — one field; layout and ABI identical to that field. Useful for newtype wrappers.
- `#[repr(packed)]` — no padding; fields may be misaligned. **Careful:** taking a reference to a packed field is UB if misalignment violates `&T`'s requirements.
- `#[repr(align(N))]` — force alignment to `N` bytes.
- `#[repr(u8)]` / `#[repr(u32)]` on enums — choose the discriminant type.

### Bindgen / cbindgen

For real C library integration, don't write `extern` blocks by hand:

- **`bindgen`** — reads a C header, emits Rust `extern` declarations.
- **`cbindgen`** — reads your Rust library, emits a C header for callers.

---

## 8. SIMD — `std::arch` intrinsics and `std::simd`

Two levels of SIMD access.

### `std::arch` — architecture-specific intrinsics

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

#[cfg(target_arch = "aarch64")]
use std::arch::aarch64::*;

#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn sum_i64_avx2(xs: &[i64]) -> i64 {
    let mut acc = _mm256_setzero_si256();
    let chunks = xs.chunks_exact(4);
    let rem = chunks.remainder();

    for c in chunks {
        let v = _mm256_loadu_si256(c.as_ptr() as *const __m256i);
        acc = _mm256_add_epi64(acc, v);
    }

    // Horizontal reduce ymm to scalar
    let lo = _mm256_extracti128_si256(acc, 0);
    let hi = _mm256_extracti128_si256(acc, 1);
    let sum128 = _mm_add_epi64(lo, hi);
    let hi64 = _mm_unpackhi_epi64(sum128, sum128);
    let s = _mm_add_epi64(sum128, hi64);
    let mut out: i64 = _mm_cvtsi128_si64(s);
    for &x in rem { out = out.wrapping_add(x); }
    out
}
```

- Every intrinsic is `unsafe` — the CPU feature must be available.
- `#[target_feature(enable = "avx2")]` tells rustc it may emit AVX2; callers must ensure the CPU supports it.
- Portable builds should use runtime dispatch (check CPU features, call the right variant).

### `std::simd` — portable SIMD (nightly, stabilizing)

```rust
#![feature(portable_simd)]
use std::simd::Simd;
use std::simd::num::SimdInt;

fn sum_portable(xs: &[i64]) -> i64 {
    let (prefix, mid, suffix) = xs.as_simd::<4>();   // 4-lane i64 = 256 bits
    let mut acc = Simd::<i64, 4>::splat(0);
    for &v in mid { acc += v; }

    let mut total: i64 = acc.reduce_sum();
    for &x in prefix.iter().chain(suffix) { total += x; }
    total
}
```

Same code compiles on x86_64 (uses AVX2), ARM64 (uses NEON), WebAssembly (wasm-simd). Safe wrapper around the underlying intrinsics. When it stabilizes, this becomes the default choice for portable SIMD kernels.

---

## 9. The safe-API-around-unsafe-internals pattern

The ideal shape of unsafe Rust:

```rust
pub struct FastBuffer<T> {
    ptr: *mut T,
    len: usize,
    cap: usize,
}

impl<T: Copy> FastBuffer<T> {
    pub fn with_capacity(cap: usize) -> Self {
        let layout = std::alloc::Layout::array::<T>(cap).unwrap();
        let ptr = unsafe { std::alloc::alloc(layout) as *mut T };
        if ptr.is_null() { std::alloc::handle_alloc_error(layout); }
        FastBuffer { ptr, len: 0, cap }
    }

    pub fn push(&mut self, value: T) {
        assert!(self.len < self.cap);   // invariant: len <= cap
        unsafe { self.ptr.add(self.len).write(value); }
        self.len += 1;
    }

    pub fn as_slice(&self) -> &[T] {
        unsafe { std::slice::from_raw_parts(self.ptr, self.len) }
    }
}

impl<T> Drop for FastBuffer<T> {
    fn drop(&mut self) {
        let layout = std::alloc::Layout::array::<T>(self.cap).unwrap();
        unsafe { std::alloc::dealloc(self.ptr as *mut u8, layout); }
    }
}
```

- Internals use raw pointers and `unsafe`.
- Public API is safe: bounds-checked, lifetime-bounded, RAII-cleaned.
- All the unsafe blocks are small, each with a paper-trail comment explaining why they're sound.

This is how `Vec<T>`, `HashMap<K, V>`, `Arc<T>`, everything in `std::collections` is built. Your DB engine's buffer pools, columnar buffers, and lock-free structures will look the same.

---

## 10. Miri — catching UB

Miri is an interpreter that runs your code under the Rust abstract machine, catching most UB that regular execution misses.

```
rustup +nightly component add miri
cargo +nightly miri test
cargo +nightly miri run
```

Miri detects:

- Use-after-free, double-free, uninitialized reads.
- Dangling pointers, pointer-to-the-past-end dereferences.
- Aliasing violations (Stacked/Tree Borrows).
- Misaligned loads/stores.
- Data races (under `-Zmiri-track-pointer-tag`).

Miri is slow (~100× slower than native execution) — use it in CI on your `unsafe`-heavy modules, not in normal dev loops. Any DB engine's `unsafe` core should be Miri-clean.

---

## 11. DB relevance

A DuckDB-like engine has `unsafe` in specific, well-scoped places:

- **Columnar buffers.** `Vec<T>` isn't always the right shape for aligned, fixed-capacity, possibly-uninitialized storage. Custom buffer types over `*mut T` are common.
- **SIMD kernels.** Vectorized predicate evaluation, sum/min/max/count, hash probing, bit-unpacking. Every hot kernel eventually wants `std::arch`.
- **mmap.** Reading Parquet / on-disk formats via `mmap` gives you `&[u8]` slices into unmanaged memory. Safe Rust can read them; writing requires unsafe.
- **Bitpacked / compressed decoding.** Streaming decoders often work byte-by-byte through unsafe pointer arithmetic for maximum speed.
- **Arena allocators.** Short-lived allocations (per-query scratch buffers) benefit from bulk-free arenas; implementing one requires unsafe.
- **FFI.** Integrating C libraries (zstd compression, jemalloc, kernel-space AIO) goes through `extern "C"`.

In all cases, the `unsafe` is bounded: each unsafe region is small, documented, and wrapped in a safe type. The rest of the engine — most of it — is safe Rust.

---

## 12. Exercises

### 12.1 Raw pointer walk

Given `let v = vec![10_i64, 20, 30, 40];`, write code that uses `v.as_ptr()` and `ptr::read` to sum its elements. Do the same with `v.as_mut_ptr()` and `ptr::write` to double each element in place. Verify with a final `println!`.

### 12.2 `MaybeUninit` buffer

Write a function:

```rust
fn collect_into_array<T: Copy, const N: usize>(iter: impl Iterator<Item = T>) -> [T; N]
```

that pulls `N` items from the iterator and returns a `[T; N]`, initializing with `MaybeUninit`. Panic if the iterator yields fewer than `N` items.

### 12.3 Aliasing violation

Write code that (a) creates a `&mut i64` to a location, (b) creates a `*mut i64` to the same location, (c) writes through the raw pointer while the `&mut` is still live, (d) reads through the `&mut` afterward. Run under Miri. Observe the diagnostic. Fix it by ordering the operations differently.

### 12.4 FFI round-trip

Write a small C file exposing `int64_t c_sum(int64_t* data, size_t len)`. Use `cc` crate in `build.rs` to compile it. Declare the extern in Rust, call it from safe Rust, and confirm it returns the right sum.

### 12.5 Safe wrapper

Wrap your `std::arch`-based `sum_i64_avx2` in a safe function that:

- Uses `is_x86_feature_detected!("avx2")` to check CPU support.
- Falls back to a scalar implementation if AVX2 isn't available.
- Exposes a public signature with no `unsafe` visible.

### 12.6 Run Miri

Pick a small unsafe function you've written (from 12.1, 12.2, or 12.3 fixed). Run `cargo +nightly miri test`. Clean up any reports. This is the development loop you'll want for any new unsafe code you write.

### 12.7 `bytemuck` cast

Use `bytemuck` to zero-copy cast between a `#[repr(C)]` struct `Record { id: u64, val: f64 }` and `[u8; 16]`. Write tests that round-trip. Then explain: what happens if you change `#[repr(C)]` to no `repr` annotation? Why?

---

## 13. Self-check

1. Name the five things `unsafe` allows.
2. What's the difference between `*const T` and `&T`?
3. Why is dereferencing a null raw pointer UB even if the program "happens to work"?
4. When is `transmute` the right tool, and what's the safer alternative for POD types?
5. What does `#[repr(C)]` do, and why is it essential for FFI?
6. Why is most of a DB engine's code NOT `unsafe`, even though the engine uses `unsafe` extensively?
7. What is Miri, and when would you use it?

---

## Next — and after

This is the last of the **core Rust lessons** (Lessons 1–9). You now have the full mental model:

- Ownership / borrowing (L1, L7)
- Slices and strings (L2)
- Traits, generics, lifetimes (L3)
- Errors (L4)
- Iterators, closures, collections (L5)
- Modules, Cargo, workspaces (L6)
- Smart pointers, interior mutability (L7)
- Concurrency (L8)
- Unsafe (L9)

The next track is **DB-specific**:

- **Lesson 10** — the Rust DB ecosystem: `arrow-rs`, `parquet-rs`, `datafusion`, `sqlparser-rs`. What exists, what you'd use, what you'd bend.
- **Lesson 11** — in-memory formats: columnar layout, validity bitmaps, RLE, dictionary encoding, bit-packing. Buffer pool design. mmap patterns.
- **Lesson 12** — kernel design: vectorized execution, morsel-driven parallelism, SIMD predicate evaluation, hash-join internals.

These are less linear than L1–L9 — they blend into the actual project work. When you're ready, start a `toydb` workspace and we'll walk through each as you hit it for real.
