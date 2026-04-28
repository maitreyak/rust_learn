# Lesson 11 — In-Memory Formats and Storage

> What a column actually looks like in memory. How nulls are represented. Why dictionary encoding wins. How a buffer pool keeps you from running out of RAM. The ground truth your kernels operate on.

---

## Objectives

- Read the Arrow columnar layout: validity bitmap + values buffer + (optional) offsets buffer.
- Encode nulls with a validity bitmap and operate on it bitwise.
- Apply dictionary encoding, run-length encoding, and bit-packing — and know when each wins.
- Design a buffer pool for working sets larger than RAM.
- Use `mmap` and understand when it helps (and when it hurts).
- Allocate buffers aligned for SIMD.

---

## 1. Row-major vs columnar — the 30-second case

A table of `(id: i64, name: utf8, region: utf8, price: i64)` over 1M rows.

**Row-major** (PostgreSQL, MySQL, OLTP engines):

```
[id|name|region|price][id|name|region|price][id|name|region|price]...
```

Reading row 5: one cache line, all four fields. Reading just `price` for all 1M rows: scattered loads, every row pulls in name/region/id we don't need. Bad cache utilization for analytics.

**Columnar** (Parquet, Arrow, ClickHouse, DuckDB, OLAP engines):

```
id:     [.....1M values.....]
name:   [.....1M values.....]
region: [.....1M values.....]
price:  [.....1M values.....]
```

Reading `price` for all 1M rows: one contiguous scan. SIMD-friendly. Cache-friendly. CPU prefetchers love it. The price you pay: reading "the whole row" requires gathering from 4 separate locations — slow for OLTP, irrelevant for OLAP because OLAP queries don't read whole rows.

For an analytical engine, the entire architecture optimizes for "scan a few columns out of a wide table fast."

---

## 2. Arrow primitive array layout

A `PrimitiveArray<T>` of N rows is two `Arc<[u8]>` buffers:

```
validity:  [b0 b1 b2 b3 ...]       ⌈N/8⌉ bytes — one bit per row, 1=valid, 0=null
values:    [v0 v1 v2 v3 ...]        N × sizeof(T) bytes
```

For `Int64Array` of `[10, NULL, 30, 40, NULL]`:

```
validity:  0b00010110           // bits 1, 2, 3 set; 0 and 4 cleared
values:    [10, ?, 30, 40, ?]   // ? = undefined (don't read)
```

Note bit-ordering: bit `i` corresponds to row `i`, so bit `0` is the least significant bit of byte `0`. This is **little-endian** bit ordering — Arrow's spec.

### Memory cost

For 1M `i64`s with sparse nulls:

- Values: 8 MB
- Validity: 128 KB

Validity overhead: 1.5%. Negligible. Always reserve 1 bit per row for validity, even if the column is declared `NOT NULL` — the kernel code stays uniform.

### Why a bitmap, not `[T; null sentinel]`?

A separate validity bitmap means:

- The values buffer is fully typed and SIMD-friendly. No "is this `i64::MIN` or a null?" check per element.
- Combining filters is a bitwise AND of bitmaps — 64 rows per CPU op.
- Null-counting is `popcount` over the bitmap — extremely fast.

---

## 3. Variable-length arrays

For `Utf8Array` (strings), `BinaryArray` (bytes), or any variable-width type, a third buffer is added: **offsets**.

```
validity:  [...]                              // bitmap as before
offsets:   [0, 5, 8, 8, 13, 18]              // i32 (or i64), length N+1
values:    "hello" "ab" "" "world" "byte!"   // contiguous bytes
```

For row `i`:

- Length: `offsets[i+1] - offsets[i]`.
- Bytes: `values[offsets[i]..offsets[i+1]]`.
- Empty strings have `length = 0` (e.g., row 2 above).
- Null strings have validity bit `0` AND length usually 0 (the offsets buffer must still be valid; null rows just point at empty ranges).

`N+1` offsets, not N: lets you compute every row's length without a special case for the last one.

### Why offsets, not lengths?

With offsets:

- Random access: `O(1)` to find any row's bytes.
- Slicing: `&values[offsets[i]..offsets[j]]` is a single contiguous slice for rows `[i..j)`.

With lengths, finding row `i` would require summing the first `i` lengths — `O(i)`.

### `LargeUtf8Array` — `i64` offsets

`Utf8Array` uses `i32` offsets, capping the values buffer at 2 GiB. `LargeUtf8Array` uses `i64` offsets when you need bigger buffers. Same shape; different offset type.

---

## 4. Dictionary encoding

For low-cardinality columns (e.g., country codes, status enums), dictionary encoding is a massive win.

Layout:

- **Dictionary** — a separate Array of unique values, length `K` (the cardinality).
- **Indices** — a primitive array of `u8`/`u16`/`u32`/`i32` per row, indexing into the dictionary.

For 1M rows of `region` with values from `{US, EU, JP, IN}`:

- Plain `Utf8Array`:
  - validity: 128 KB
  - offsets: 4 MB (i32 × 1M)
  - values: ~3 MB (avg 3 bytes per region × 1M)
  - **Total: ~7 MB**
- `DictionaryArray<Int32Type, Utf8>`:
  - dictionary: 4 strings ≈ 32 bytes
  - indices: 4 MB (i32 × 1M)
  - validity: 128 KB
  - **Total: ~4 MB** (smaller than just the offsets of the plain version)

If your indices fit in `u8` (cardinality ≤ 256), drop to 1 MB total. For 1M rows of `country_code` with ~250 distinct values, that's a ~7× reduction.

### Dictionary-aware execution

Beyond memory savings, dictionary encoding accelerates execution:

- **Group-by:** group on indices (1–4 byte integer hash) instead of strings.
- **Equality filter:** `region = 'US'` becomes "look up `'US'` in the dictionary, get index 0, scan indices for `== 0`." 64 comparisons per `_mm256_cmpeq_epi32`.
- **Hash join:** hash on indices instead of variable-length keys. Faster, more cache-friendly hash table.

Production engines lean hard on dict encoding. Parquet writes it automatically when it pays. Arrow's compute kernels are dictionary-aware for the common cases.

---

## 5. Run-Length Encoding (RLE)

For columns with long runs of the same value:

```
unencoded:  [US, US, US, EU, EU, JP, JP, JP, JP, US]
RLE:        [(US,3), (EU,2), (JP,4), (US,1)]
```

Wins for:

- **Sorted columns.** A `region` column sorted by region has the longest possible runs.
- **Boolean masks with locality** (alternating selectivity → not great; long stretches of all-true/all-false → great).
- **Time-series data with stable values.**

Loses for:

- **Random data with no runs** — RLE is *bigger* than the original.
- **Random access** — finding row `i` requires walking the runs (or a separate index).

Arrow has `RunArray<RunEnds, Values>` for native RLE columns. Most kernels still go through a "decompress to plain array" step internally, but compression-aware kernels are common.

---

## 6. Bit-packing

For low-magnitude integers, store fewer than 64 bits per value.

```
unencoded:  [3, 5, 7, 2, 6]      // each i64 = 8 bytes = 40 bytes total
12-bit pack:                     // pack 5 values into 5 × 12 = 60 bits = 7.5 bytes
```

Parquet does this aggressively at the storage layer (DELTA_BINARY_PACKED, BIT_PACKED). Arrow's in-memory format doesn't bit-pack primitive arrays — they stay 64 bits — but Arrow's spec defines `RunArray` and various dictionary forms that achieve similar size wins.

The tradeoff: bit-unpacking costs cycles. If your hot path is memory-bandwidth bound (Lesson 2's plain-sum experiment), reducing bytes per value might pay. If it's CPU-bound, the unpack overhead can dominate.

For your engine: don't bit-pack in-memory until you've measured. Do honor bit-packing on disk (Parquet) — the I/O savings are real.

---

## 7. Validity bitmap mechanics

Manually:

```rust
fn is_valid(bitmap: &[u8], row: usize) -> bool {
    bitmap[row / 8] & (1u8 << (row % 8)) != 0
}

fn set_valid(bitmap: &mut [u8], row: usize) {
    bitmap[row / 8] |= 1u8 << (row % 8);
}
```

Arrow ships `BooleanBuffer` and `NullBuffer` with optimized implementations.

### Bitwise operations on bitmaps

```rust
fn and_bitmaps(a: &[u64], b: &[u64], out: &mut [u64]) {
    for i in 0..a.len() {
        out[i] = a[i] & b[i];          // 64 rows per AND
    }
}
```

LLVM auto-vectorizes this to AVX2 / NEON: 256/128 bits per instruction. Combining N predicates becomes O(N) cheap bitmap ANDs, regardless of row count.

### Population count (popcount)

Counting set bits is a single CPU instruction on every modern architecture (`popcnt` on x86, `cnt` on ARM). Use it for null-counting, selectivity estimation, and converting bitmaps to row counts:

```rust
fn count_valid(bitmap: &[u64]) -> u64 {
    bitmap.iter().map(|w| w.count_ones() as u64).sum()
}
```

---

## 8. Buffer pool — when working sets exceed RAM

You can't always fit everything in memory. For tables larger than RAM, you use a **buffer pool**: a fixed-size cache of disk pages.

### Pages

A page is a unit of disk I/O — typically 4 KB to 256 KB depending on the engine. Bigger pages amortize I/O cost; smaller pages waste less on partial reads. DuckDB uses 256 KB; PostgreSQL uses 8 KB; Arrow's IPC format streams batches that act as logical pages.

### Page table + frames

```rust
use std::sync::Mutex;
use std::collections::HashMap;
use std::sync::Arc;

pub struct BufferPool {
    pages: Mutex<HashMap<PageId, Arc<Page>>>,
    capacity: usize,                        // max pages in memory
    eviction: Mutex<EvictionState>,
}

#[derive(Clone)]
pub struct Page {
    pub id: PageId,
    pub data: Arc<[u8]>,
    pub dirty: std::sync::atomic::AtomicBool,
}
```

`Arc<Page>` lets multiple readers share a page without locking. Cloning is one atomic increment. Evicting a page just means dropping its entry from the `HashMap` — the `Arc` lives until the last reader releases it (so a page being scanned by an operator is naturally pinned by the `Arc` reference).

### Eviction

Common policies:

- **LRU** — evict the least-recently-used. Easy. Pessimal under sequential scans (everything is "recent"; eviction thrashes).
- **LRU-K** — evict the page with the oldest K-th-most-recent access. Bandit-resistant.
- **Clock** / **Clock-Pro** — circular buffer with a "use" bit. Cheap. PostgreSQL uses Clock.
- **ARC** — Adaptive Replacement Cache. Self-tuning. IBM uses it.

For a learning project: implement LRU first; upgrade if you measure thrashing.

### Sharded mutex

A single `Mutex<HashMap<...>>` becomes a contention point with many threads. The fix: shard the mutex.

```rust
const SHARDS: usize = 64;

pub struct ShardedBufferPool {
    shards: [Mutex<HashMap<PageId, Arc<Page>>>; SHARDS],
}

fn shard_for(id: PageId) -> usize {
    (id as usize) % SHARDS
}
```

64 shards → most accesses go to different shards → near-zero contention. Arrow / DataFusion / DuckDB / RocksDB all do this.

---

## 9. mmap — let the OS do the buffering

`mmap` maps a file's bytes directly into your process's address space. Reads become memory accesses; the OS handles paging.

```rust
use memmap2::MmapOptions;
use std::fs::File;

let file = File::open("orders.parquet")?;
let mmap = unsafe { MmapOptions::new().map(&file)? };
let bytes: &[u8] = &mmap[..];
```

`memmap2::Mmap` derefs to `&[u8]` — your slice abstraction from Lesson 2 works on memory-mapped storage just as well as it does on a `Vec<u8>`.

### When mmap is great

- **Local files** — no network, no fault costs to speak of.
- **Read-heavy workloads** — the OS page cache is excellent.
- **Random access** — no seek or buffer-management code in your engine.
- **Engineering simplicity** — drop a `&[u8]` into your Parquet reader, you're done.

### When mmap is terrible

- **Networked or cloud storage** — page faults are now over the network. Latency spikes are catastrophic. Don't mmap S3.
- **Tight memory budgets** — you can't easily limit the page cache; the OS may evict your other workloads.
- **Pages bigger than memory** — you'll thrash.
- **Writes** — write-mmap is a minefield (durability semantics, partial flushes).

DuckDB's stance: mmap for local data, fall back to explicit reads otherwise. `arrow-rs` works either way — feed it a `Vec<u8>` or a `&[u8]` from mmap; same code path.

### `madvise`

OS-level hint to the kernel about access pattern. Through `mmap.advise(...)`:

- `Sequential` — prefetch ahead aggressively.
- `Random` — don't prefetch.
- `WillNeed` — pre-fault pages.
- `DontNeed` — release pages from cache.

Use `Sequential` when streaming a Parquet file's row groups.

---

## 10. Memory alignment for SIMD

For SIMD loads to be optimal, buffers should be aligned to the vector width:

- AVX2 (256-bit): align to 32 bytes.
- AVX-512 (512-bit): align to 64 bytes.
- NEON (128-bit): align to 16 bytes.

Misaligned loads "work" on modern CPUs (they're not faults), but slower than aligned ones. AVX-512 misaligned loads are sometimes much slower.

### Aligned allocation

`std::alloc::alloc` with a custom `Layout`:

```rust
use std::alloc::{alloc, dealloc, Layout};

unsafe fn alloc_aligned<T>(n: usize, align: usize) -> *mut T {
    let layout = Layout::from_size_align(n * std::mem::size_of::<T>(), align).unwrap();
    alloc(layout) as *mut T
}
```

Or via `Vec` in a wrapper that over-allocates and offsets the pointer.

`bytemuck` and Arrow's `MutableBuffer` handle this for you. Manual alignment matters when you build custom buffer types.

### Cache line padding

False sharing (two threads writing to different fields of one cache line cause invalidation traffic) is solved by padding to 64 bytes:

```rust
#[repr(align(64))]
pub struct CacheLineAligned<T>(pub T);
```

Useful for per-thread accumulators in a parallel reduction.

---

## 11. Putting it together: a column type

Sketch of an in-memory `Int64Column` for your engine:

```rust
use std::sync::Arc;

#[derive(Clone)]
pub struct Int64Column {
    /// Length in rows.
    len: usize,

    /// Reference-counted, immutable values buffer.
    /// Length = len × 8 bytes, aligned to at least 8.
    values: Arc<[i64]>,

    /// Optional validity bitmap. None = all rows valid.
    /// Length = ⌈len / 8⌉ bytes when present.
    nulls: Option<Arc<[u8]>>,
}

impl Int64Column {
    pub fn len(&self) -> usize { self.len }
    pub fn values(&self) -> &[i64] { &self.values[..self.len] }

    pub fn is_valid(&self, row: usize) -> bool {
        match &self.nulls {
            None => true,
            Some(b) => b[row / 8] & (1u8 << (row % 8)) != 0,
        }
    }

    /// Zero-copy slice of [start..end).
    pub fn slice(&self, start: usize, end: usize) -> Self {
        // Sub-slicing the values buffer requires aligned offsets;
        // for full generality you keep an offset field. Sketched here without.
        assert!(end <= self.len);
        Int64Column {
            len: end - start,
            values: self.values.clone(),       // shared
            nulls: self.nulls.clone(),
        }
    }
}
```

That's the basic shape. arrow-rs gives you a fully-optimized version (`Int64Array`); your engine can either use that directly or wrap it for ergonomics.

---

## 12. Exercises

### 12.1 Hand-build a validity bitmap

Given `rows = [10, None, 30, 40, None, 60, 70, None]`, build the validity bitmap as a `Vec<u8>`. Verify with `is_valid` from §7. Then count non-null rows using `count_ones()` and compare with `rows.iter().filter(|r| r.is_some()).count()`.

### 12.2 Bitwise filter combination

Two bitmaps `a` and `b` (each 1M bits) representing two filter results. Compute `a AND b` (intersection) and `a OR b` (union) using `Vec<u64>` representations. Time both. Compare against a naive `Vec<bool>`-based version. Expected speedup: 8–16×.

### 12.3 Dict encode

Take a `Vec<&str>` of region values (1M rows, 4 distinct values). Build:

- A dictionary (`Vec<&str>`) of unique values.
- An indices array (`Vec<u8>`).

Verify reconstruction matches the input. Compute the size of (a) the original `Vec<String>`, (b) the dict + indices. Confirm the savings.

### 12.4 RLE encode/decode

Implement:

```rust
fn rle_encode<T: PartialEq + Copy>(xs: &[T]) -> Vec<(T, u32)>;
fn rle_decode<T: Copy>(runs: &[(T, u32)]) -> Vec<T>;
```

Test with a sorted column (long runs) and an unsorted one (short runs). Compare encoded sizes.

### 12.5 LRU buffer pool

Implement a buffer pool with:

- 4-page capacity.
- `LinkedHashMap` (or a custom doubly-linked list) for LRU order.
- `get(page_id)` returns `Arc<Page>`, evicting if needed.

Stress test: 100K random page accesses across 1000 pages. Track hit rate. Compare against a naive HashMap with random eviction.

### 12.6 mmap a file

Write a 1 GB file of random bytes. mmap it. Touch every 4 KB page to force the OS to fault it in; time this. Then re-read the same pages; time again. The first run is bound by disk I/O, the second by memory bandwidth. Compute both throughputs.

### 12.7 Aligned allocation

Allocate a 1M `i64` buffer aligned to 64 bytes (cache-line aligned). Verify alignment with `assert_eq!(ptr as usize % 64, 0)`. Compute a sum kernel over it and compare with a default-aligned `Vec<i64>` (which is 8-byte aligned). On AVX-2 hardware, expect minor differences; on AVX-512, expect more pronounced ones.

---

## 13. Self-check

1. What three buffers does an Arrow `Utf8Array` have, and what does each contain?
2. Why is a separate validity bitmap (rather than a sentinel value) the right way to encode nulls?
3. When does dictionary encoding pay off, and what's the cardinality threshold rule of thumb?
4. Why does RLE win on sorted columns and lose on random ones?
5. What does sharded-mutex sharding solve in a buffer pool?
6. When is mmap the wrong choice for storage I/O?
7. Why does SIMD alignment matter, and which alignment do you target for AVX2?

Next: **Lesson 12 — Kernel design and vectorized execution.** Where everything in Lessons 2, 8, 9, and 11 finally combines into a working query engine.
