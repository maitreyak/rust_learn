# Lesson 12 — Kernel Design and Vectorized Execution

> Where everything from Lessons 2, 8, 9, and 11 combines. The actual hot loop of an analytical database. Predicate evaluation, hash join, hash aggregation, morsel-driven parallelism.

---

## Objectives

- Compare the Volcano (row-at-a-time) and vectorized (batch-at-a-time) execution models.
- Choose between bitmap and selection-vector representations of "rows that pass."
- Write a vectorized predicate kernel that produces a bitmap output.
- Implement a hash join with a build phase and a probe phase.
- Implement hash aggregation with two-phase partial-final reduction.
- Apply morsel-driven parallelism via `rayon` or a custom work-stealing pool.
- Reason about late materialization and when it wins.

---

## 1. The two execution models

### Volcano / iterator model

Each operator implements `Iterator<Item = Row>`. A query plan is a tree of operators; the root pulls rows; pulls cascade down the tree.

```rust
// Pseudocode
trait VolcanoOp {
    fn next(&mut self) -> Option<Row>;
}
```

- **Pros:** simple, streaming, no intermediate materialization.
- **Cons:** one virtual call per operator per row. With a 5-deep plan and 100M rows, that's 500M virtual calls — pure overhead.

PostgreSQL, MySQL, SQLite use this model (with assorted optimizations).

### Vectorized model

Each operator processes a **batch** of rows (typically 1024–8192) per call.

```rust
trait VectorizedOp {
    fn next_batch(&mut self) -> Option<RecordBatch>;
}
```

- **Pros:** virtual calls amortize over thousands of rows; inner loops over slices are SIMD-friendly; cache lines fully utilized.
- **Cons:** higher memory in flight; harder to fit a strict "one row at a time" abstraction.

DuckDB, ClickHouse, MonetDB/X100, DataFusion, Apache Arrow's compute kernels — all vectorized. **2–10× faster** than Volcano on analytical workloads.

The seminal paper: *MonetDB/X100: Hyper-Pipelining Query Execution* (Boncz et al., 2005). The follow-up batch-size study: *Vectorization vs. Compilation in Query Execution* (Sompolski et al., 2011).

You will write vectorized operators.

---

## 2. Selection: bitmap vs. selection vector

Two ways to represent "the subset of rows that survived a filter."

### Bitmap

```rust
type Bitmap = Vec<u64>;     // 1 bit per row, 64 rows per word
```

- 1 bit per row.
- 64 rows → 1 CPU register operation (`AND`, `OR`, `popcount`).
- **Densely packed** — small.
- Best for: **high selectivity** (many rows surviving), and combining multiple predicates with bitwise ops.

### Selection vector

```rust
type SelectionVector = Vec<u16>;    // indices of selected rows; u16 fits 65k-row batches
```

- 16 (or 32) bits per *selected* row.
- **Sparse** — small only if selectivity is low.
- Best for: **low selectivity** (few rows surviving) or after the filter has been applied (subsequent operators can index directly without scanning the bitmap).

### Crossover

Empirical rule: **selectivity > ~10–20% → bitmap**, **< ~10% → selection vector**. DuckDB uses both adaptively, switching based on selectivity estimates and downstream operator preferences. Most academic vectorized systems pick one and stick with it.

### Bitmap → selection vector

You'll need this conversion when downstream code wants explicit indices:

```rust
fn bitmap_to_selection(bitmap: &[u64], total_rows: usize) -> Vec<u16> {
    let mut sel = Vec::with_capacity(total_rows);
    for (word_idx, &word) in bitmap.iter().enumerate() {
        let mut w = word;
        while w != 0 {
            let bit = w.trailing_zeros() as usize;
            let row = word_idx * 64 + bit;
            if row < total_rows {
                sel.push(row as u16);
            }
            w &= w - 1;             // clear lowest set bit
        }
    }
    sel
}
```

The `w & (w - 1)` trick clears the lowest set bit; `trailing_zeros` finds it. Three CPU instructions per selected row. Modern x86 has `PEXT` (BMI2) for parallel bit extraction, which can speed this up further.

---

## 3. Vectorized predicate evaluation

The simplest vectorized kernel: a comparison producing a bitmap.

```rust
pub fn predicate_gt(values: &[i64], threshold: i64) -> Vec<u64> {
    let n = values.len();
    let n_words = (n + 63) / 64;
    let mut bitmap = vec![0u64; n_words];

    for chunk_idx in 0..n_words {
        let mut mask: u64 = 0;
        let base = chunk_idx * 64;
        let end = (base + 64).min(n);

        for i in base..end {
            if values[i] > threshold {
                mask |= 1u64 << (i - base);
            }
        }
        bitmap[chunk_idx] = mask;
    }

    bitmap
}
```

LLVM auto-vectorizes the inner loop on `target-cpu=native`: on AVX2, it's 4 lanes of `i64` per `vpcmpgtq`, on NEON it's 2 lanes. The `mask |= ...` accumulation may stay scalar; for maximum throughput, use the explicit intrinsic `_mm256_movemask_pd` (or equivalent) to extract the comparison mask directly.

### Combining predicates

`WHERE a > 10 AND b < 100`:

```rust
let mask_a = predicate_gt(&col_a, 10);
let mask_b = predicate_lt(&col_b, 100);
let combined: Vec<u64> = mask_a.iter().zip(&mask_b).map(|(&x, &y)| x & y).collect();
```

64 rows per `AND`. Combining N predicates is O(N) cheap bitmap ops, regardless of row count. This is why bitmaps shine for multi-predicate filters.

### Branchless aggregates over a mask

From Lesson 2, the masked sum:

```rust
pub fn sum_where(values: &[i64], bitmap: &[u64]) -> i64 {
    let mut total: i64 = 0;
    for (chunk_idx, &word) in bitmap.iter().enumerate() {
        let base = chunk_idx * 64;
        let end = (base + 64).min(values.len());
        for i in base..end {
            let take = (word >> (i - base)) & 1;
            total = total.wrapping_add(values[i].wrapping_mul(take as i64));
        }
    }
    total
}
```

Branchless multiply-by-mask, vectorizable. On Apple Silicon you saw this 2× faster than the branchy version (Lesson 2's experiment).

---

## 4. Hash join

The classical analytical join. Two phases:

- **Build:** scan one input (typically the smaller), build a hash table keyed on the join column, valued by row pointers.
- **Probe:** scan the other input, hash each row's key, look up in the hash table, emit match pairs.

### Skeleton

```rust
use std::collections::HashMap;

pub struct HashJoin {
    table: HashMap<i64, Vec<u32>>,    // key -> row IDs from build side
}

impl HashJoin {
    pub fn build(&mut self, keys: &[i64]) {
        for (i, &k) in keys.iter().enumerate() {
            self.table.entry(k).or_default().push(i as u32);
        }
    }

    pub fn probe(&self, keys: &[i64]) -> Vec<(u32, u32)> {
        // Returns (probe_row_id, build_row_id) pairs.
        let mut out = Vec::new();
        for (probe_row, &k) in keys.iter().enumerate() {
            if let Some(build_rows) = self.table.get(&k) {
                for &build_row in build_rows {
                    out.push((probe_row as u32, build_row));
                }
            }
        }
        out
    }
}
```

This is correct but slow. Production hash joins do five things differently.

### Production tweaks

1. **Open-addressed table, power-of-2 size.** Linear probing for cache locality. Avoid `HashMap`'s overhead and chaining.
2. **Faster hasher.** `ahash` or a custom integer hash (`x.wrapping_mul(0x9E3779B97F4A7C15)`) — `SipHash` is for adversarial inputs only.
3. **SIMD probe.** Process 4–8 probe keys in parallel, hashing and looking up simultaneously. Each probe needs `find_first_match` over a small range — perfect for SIMD.
4. **Bloom filter pre-check.** A 1-bit-per-row Bloom filter built from the build side; probe rows whose Bloom check fails are skipped without touching the hash table. Eliminates 90%+ of probe-side work for selective joins.
5. **Late materialization.** The probe returns row IDs (`(probe_row, build_row)` pairs). Actual column values are gathered only at the very end, by the projection operator.

### Spill to disk

When the build side is bigger than memory, partition by hash. Hash-partition both inputs into `2^k` buckets; join each pair of corresponding buckets independently. If a bucket still doesn't fit, recurse. This is **grace hash join**. DuckDB and Spark do it.

For your engine: skip spill in v1. Add it in v2 when you actually run out of memory.

---

## 5. Hash aggregation (`GROUP BY`)

Same shape as hash join, but the value is an accumulator instead of row IDs.

```rust
use std::collections::HashMap;

pub struct AggState {
    pub sum: i64,
    pub count: u64,
}

pub struct HashAggregator {
    table: HashMap<i64, AggState>,    // group key -> accumulator
}

impl HashAggregator {
    pub fn update(&mut self, keys: &[i64], values: &[i64]) {
        for (k, v) in keys.iter().zip(values) {
            let s = self.table.entry(*k).or_default();
            s.sum = s.sum.wrapping_add(*v);
            s.count += 1;
        }
    }
}
```

For a multi-column group key, use a tuple or a hashed composite. For string keys, hash the bytes (with `ahash`).

### Two-phase aggregation (the parallel pattern)

Per-thread partial aggregation, then merge:

1. Each thread maintains its own `HashAggregator`.
2. Threads process disjoint morsels in parallel — no contention.
3. At the end, merge per-thread tables into a single global one.

```rust
use rayon::prelude::*;

pub fn parallel_aggregate(
    morsels: &[(Vec<i64>, Vec<i64>)],   // (keys, values) per morsel
) -> HashMap<i64, AggState> {
    morsels.par_iter()
        .map(|(keys, values)| {
            let mut local = HashAggregator { table: HashMap::new() };
            local.update(keys, values);
            local.table
        })
        .reduce(HashMap::new, |mut a, b| {
            for (k, s) in b {
                let entry = a.entry(k).or_default();
                entry.sum = entry.sum.wrapping_add(s.sum);
                entry.count += s.count;
            }
            a
        })
}
```

`par_iter().map(...).reduce(...)` is the canonical rayon shape for "process in parallel, then merge." Rayon manages the thread pool.

### Production tweaks

- **Open-addressed hash table** (same as hash join).
- **Group state in a separate arena**, not embedded in the hash table — better cache behavior when accumulators are large.
- **Adaptive switching:** if cardinality is small (< thousands), keep all groups in registers; if it's huge (millions+), switch to sort-based aggregation (sort by key, then reduce adjacent entries).

ClickHouse calls this the *adaptive aggregator*. DuckDB's hash aggregation is morsel-driven and has overflow strategies. Both are good source code to read.

---

## 6. Morsel-driven parallelism

The execution model from Leis et al., *Morsel-Driven Parallelism* (SIGMOD 2014). Three ideas:

1. **Split work into morsels.** A morsel is a small chunk of input — typically 100K rows or one row group. Smaller than a whole partition, larger than a tuple.
2. **Work-stealing thread pool.** Threads pull morsels from a shared queue; idle threads steal from busy ones. No assignment in advance — adaptive to runtime imbalance.
3. **Per-thread pipeline state.** Each thread maintains its own accumulators (hash table, hash join build chunks, aggregation state). State is merged only at the end.

### Sketch

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

pub struct MorselQueue {
    next: AtomicUsize,
    total: usize,
}

impl MorselQueue {
    pub fn next_morsel(&self) -> Option<usize> {
        let i = self.next.fetch_add(1, Ordering::Relaxed);
        if i < self.total { Some(i) } else { None }
    }
}

pub fn run_pipeline<F>(queue: Arc<MorselQueue>, num_threads: usize, work: F) -> Vec<PerThreadState>
where
    F: Fn(usize, &mut PerThreadState) + Send + Sync + 'static,
{
    let work = Arc::new(work);
    let mut handles = Vec::new();
    for _ in 0..num_threads {
        let q = Arc::clone(&queue);
        let w = Arc::clone(&work);
        handles.push(std::thread::spawn(move || {
            let mut state = PerThreadState::new();
            while let Some(morsel_id) = q.next_morsel() {
                w(morsel_id, &mut state);
            }
            state
        }));
    }
    handles.into_iter().map(|h| h.join().unwrap()).collect()
}
```

`AtomicUsize::fetch_add` is the dispatch primitive: each thread atomically grabs the next morsel index. Cost is one cache-line ping per morsel — at 100K rows per morsel, this is < 1ns per row of overhead.

For a real engine, replace the simple atomic counter with a work-stealing deque (`crossbeam_deque` or a custom one).

---

## 7. Late materialization

**Eager:** every operator produces a fully-materialized output column.

**Late:** intermediate operators carry only row IDs; column values are gathered at the very end.

### Example: `SELECT name, region FROM orders WHERE price > 100 AND discount > 0.1`

**Eager pipeline:**

1. Scan: read `name`, `region`, `price`, `discount`.
2. Filter price: produce filtered `name`, `region`, `discount`.
3. Filter discount: produce filtered `name`, `region`.
4. Output.

Every intermediate stage allocates and copies the surviving rows of `name` (a string column!) and `region`.

**Late pipeline:**

1. Scan: read `price`, `discount`. (Skip `name`, `region`.)
2. Filter price: produce a bitmap.
3. Filter discount: produce a bitmap.
4. AND bitmaps: produce final selection.
5. Gather `name` and `region` at the surviving row IDs.
6. Output.

Strings are gathered exactly once, only for surviving rows. If selectivity is 1%, you've avoided ~99% of the string copies.

### When late wins

- Selectivity is low.
- Surviving columns are wide (strings, structs, large blobs).
- The query doesn't need wide columns until the end.

### When eager wins

- Selectivity is high (> 50%).
- Columns are narrow (i32, i64).
- Subsequent operators expect dense columns (e.g., a hash-join probe of a string column needs the strings).

DuckDB does both adaptively. ClickHouse leans late. The right choice is workload-dependent — instrument before deciding.

---

## 8. Putting it together — a query plan

```sql
SELECT region, SUM(price)
FROM orders
WHERE amount > 100
GROUP BY region
```

Pipeline shape:

1. **Source.** Open `orders.parquet`. Stream batches of 8192 rows.
2. **Project.** Read only `region`, `price`, `amount`. (Parquet columnar pushdown.)
3. **Filter.** `amount > 100` → bitmap.
4. **Apply filter.** Gather `region` and `price` at surviving rows. (Late materialization or eager — your call.)
5. **Hash aggregate.** Hash on `region`; accumulate `SUM(price)`.
6. **Final reduce.** Merge per-thread aggregators.
7. **Output.** Convert the final hash table to a `RecordBatch`.

### Code skeleton, vectorized + morsel-driven

```rust
use std::sync::Arc;
use rayon::prelude::*;

fn execute_query(file: &str) -> anyhow::Result<HashMap<String, i64>> {
    use parquet::arrow::arrow_reader::ParquetRecordBatchReaderBuilder;
    use std::fs::File;

    let reader = ParquetRecordBatchReaderBuilder::try_new(File::open(file)?)?
        .with_batch_size(8192)
        .with_projection(/* indices for region, price, amount */ Some(vec![0, 1, 2].into()))
        .build()?;

    // Collect batches (cheap; they're Arc-shared internally).
    let batches: Vec<_> = reader.collect::<Result<_, _>>()?;

    // Process in parallel, per-thread accumulator.
    let totals = batches.par_iter()
        .map(|batch| {
            let region = batch.column_by_name("region").unwrap();
            let price  = batch.column_by_name("price").unwrap()
                .as_primitive::<arrow::datatypes::Int64Type>().values();
            let amount = batch.column_by_name("amount").unwrap()
                .as_primitive::<arrow::datatypes::Int64Type>().values();

            let mask = predicate_gt(amount, 100);
            let mut local: HashMap<String, i64> = HashMap::new();

            // For each surviving row, accumulate. (Real impl: process bitmap word-at-a-time.)
            for i in 0..price.len() {
                let take = (mask[i / 64] >> (i % 64)) & 1;
                if take == 1 {
                    let key: String = /* extract region[i] as String */ unimplemented!();
                    *local.entry(key).or_insert(0) += price[i];
                }
            }
            local
        })
        .reduce(HashMap::new, |mut a, b| {
            for (k, v) in b {
                *a.entry(k).or_insert(0) += v;
            }
            a
        });

    Ok(totals)
}
```

That's roughly the shape of a vectorized, parallel `GROUP BY` aggregation. It's about 50 lines, and it has the bones of a real engine. To make it production-quality you'd:

- Replace `HashMap<String, i64>` with an open-addressed table on dictionary indices.
- Use a real selection vector + late materialization for `region`.
- Add a Bloom filter on `amount > 100` if the predicate is more complex.
- Spill the hash table when it exceeds a memory budget.
- Stream output batches (so the query is memory-bounded even for billion-group results).

You're now where you can start writing a real engine.

---

## 9. Exercises

### 9.1 Vectorized predicate

Implement `predicate_gt(values: &[i64], threshold: i64) -> Vec<u64>`. Verify against a naive scalar version on 10M random rows. Time both. Expected: bitmap version is 2–10× faster on `target-cpu=native`.

### 9.2 Bitmap → selection vector

Implement `bitmap_to_selection(bitmap: &[u64], total_rows: usize) -> Vec<u16>`. Test that round-tripping through `selection_to_bitmap` recovers the original bitmap. Compare the time and space of (a) bitmap representation, (b) selection vector representation, at 1%, 10%, 50%, 99% selectivity.

### 9.3 Hash join skeleton

Implement the hash join from §4. Test on two `Vec<i64>` columns of 1M rows each with 50% match rate. Compare `HashMap<i64, Vec<u32>>` vs. `ahash::AHashMap<i64, Vec<u32>>`.

### 9.4 Two-phase aggregation

Build the `parallel_aggregate` function from §5. Test on 50M rows × 100 distinct group keys. Compare against single-threaded. Expected speedup: ~`num_cores`.

### 9.5 Morsel-driven sum

Use the `MorselQueue` from §6 (or rayon, whichever you prefer) to compute the sum of a 100M-element column. Time vs. single-threaded and vs. `rayon::par_iter().sum()`. Confirm all three agree on the answer.

### 9.6 Late vs. eager materialization

Build a tiny pipeline: scan a 10M-row Parquet file with columns `(name: String, region: String, price: i64)`, filter `price > some_threshold`, output `(name, region)`. Two implementations:

- **Eager:** materialize filtered `name` and `region` after the filter.
- **Late:** carry a selection vector; gather `name` and `region` at the end.

Run at 1%, 10%, 50% selectivity. Plot the speedup. Identify the crossover.

### 9.7 Build a "toy DuckDB"

Combine everything: a workspace from Lesson 6 with crates `parser` (use `sqlparser`), `planner` (your code), `kernels` (your code, this lesson), and `cli`. Make it run:

```sql
SELECT region, SUM(price) FROM orders WHERE amount > 100 GROUP BY region
```

against a Parquet file. Output a pretty table with `comfy-table`. This is the milestone that unlocks all subsequent project work.

---

## 10. Self-check

1. Why are virtual function calls per row catastrophic in Volcano-style engines, and how does the vectorized model fix it?
2. When should you use a bitmap representation, and when a selection vector?
3. What's the role of an open-addressed hash table in production hash joins?
4. What does two-phase aggregation buy you, and why does it work even though `HashMap` is not `Send`-without-careful-thought?
5. Why is per-thread state with end-of-pipeline merging strictly faster than a shared `Mutex<HashMap>`?
6. When does late materialization win? When does it lose?
7. What's the role of a morsel queue in a parallel engine — what does it replace, and what's the cost of taking the next morsel?

---

## What's next

You've covered the canonical core Rust curriculum (Lessons 1–9) and the DB-specific track (Lessons 10–12). At this point further learning is **project-driven**:

- **Build the `toydb` workspace** from §9.7. You'll re-encounter every concept from this curriculum at least once.
- **Read other people's code.** DuckDB (C++), DataFusion (Rust), ClickHouse (C++). Each has a distinct architecture; comparing them is the fastest way to develop taste.
- **Read the papers.** Boncz et al. 2005 (vectorization), Leis et al. 2014 (morsels), Stonebraker et al. 2005 (column stores), Graefe 1994 (Volcano), Ross 2007 (selection vectors).
- **Pick one thing to do really well.** Scan + filter + sum on Parquet is enough for v1. Add one feature at a time: hash join, hash agg, sort, joins-with-spill, distributed.

The lessons are scaffolding. The engine is yours to build.
