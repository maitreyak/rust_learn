# Lesson 10 — The Rust DB Ecosystem

> What already exists, what you'd use, what you'd bend, what you'd skip. The map of crates that turns "I want to build a DuckDB-like thing" into a tractable project.

---

## Objectives

- Know the role of each major crate: `arrow-rs`, `parquet-rs`, `datafusion`, `sqlparser-rs`, `object_store`.
- Build a `RecordBatch` programmatically and read its bytes.
- Open a Parquet file and stream batches out.
- Run a SQL query through DataFusion in ~50 lines.
- Parse SQL with `sqlparser-rs` and walk the AST.
- Decide between three paths for your project: compose, extend DataFusion, or from scratch.

---

## 1. The ecosystem map

```
                              ┌──────────────┐
                              │ datafusion   │  query engine
                              └──────┬───────┘
                ┌────────────────────┼─────────────────────┐
                ▼                    ▼                     ▼
        ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
        │ sqlparser-rs │     │ parquet-rs   │     │ object_store │
        │ (SQL parser) │     │ (file format)│     │ (S3 / local) │
        └──────────────┘     └──────┬───────┘     └──────────────┘
                                    │
                                    ▼
                            ┌──────────────┐
                            │  arrow-rs    │  in-memory columnar format
                            └──────────────┘
```

Apache Arrow is the foundation: a cross-language in-memory columnar format. Once your data is in Arrow, the rest of the stack (file I/O, query execution, serialization) just operates on `RecordBatch`es. arrow-rs is to a Rust DB engine what `numpy.ndarray` is to a Python data tool — the lingua franca everyone speaks.

---

## 2. arrow-rs — the foundation

Pin to the latest major version (`50` or higher as of 2026).

```toml
[dependencies]
arrow = "50"
```

### The four core types

- **`DataType`** — describes a column's logical type (`Int64`, `Utf8`, `List(Box<DataType>)`, `Struct(...)`, `Decimal128(p, s)`, …).
- **`Field`** — `(name, DataType, nullable)`.
- **`Schema`** — `Vec<Field>` plus optional metadata.
- **`RecordBatch`** — a columnar table-shaped batch: a `SchemaRef` and `Vec<ArrayRef>`, where `ArrayRef` is `Arc<dyn Array>`.

### Building a RecordBatch programmatically

```rust
use arrow::array::{ArrayRef, Int64Array, StringArray};
use arrow::datatypes::{DataType, Field, Schema};
use arrow::record_batch::RecordBatch;
use std::sync::Arc;

fn make_batch() -> arrow::error::Result<RecordBatch> {
    let schema = Arc::new(Schema::new(vec![
        Field::new("id",    DataType::Int64, false),
        Field::new("name",  DataType::Utf8,  true),
        Field::new("price", DataType::Int64, false),
    ]));

    let id    = Int64Array::from(vec![1, 2, 3, 4, 5]);
    let name  = StringArray::from(vec![Some("alice"), Some("bob"), None, Some("dave"), Some("eve")]);
    let price = Int64Array::from(vec![100, 200, 300, 400, 500]);

    RecordBatch::try_new(schema, vec![
        Arc::new(id) as ArrayRef,
        Arc::new(name),
        Arc::new(price),
    ])
}
```

### Inspecting an array

The `Array` trait gives you the polymorphic interface; downcast to the concrete type to get the typed slice:

```rust
use arrow::array::AsArray;
use arrow::datatypes::Int64Type;

let batch = make_batch()?;
let prices = batch.column(2).as_primitive::<Int64Type>();
let slice: &[i64] = prices.values();          // your slice from Lesson 2!
let total: i64 = slice.iter().sum();
println!("{}", total);                         // 1500
```

`as_primitive::<Int64Type>()` returns a `&PrimitiveArray<Int64Type>`. `.values()` returns `&[i64]` — exactly the slice abstraction you've spent two lessons on. Every arrow numeric kernel ultimately operates on slices like this.

### Memory layout, briefly

A primitive `Int64Array` of N rows is backed by:

- A **values buffer** — `Arc<[u8]>` containing N `i64`s laid out contiguously.
- An optional **validity bitmap** — one bit per row, `1 = valid`, `0 = null`.

Both buffers are reference-counted. Two arrays can share the same values buffer with different validity bitmaps — useful for filter results.

We'll dig into this in Lesson 11.

### Compute kernels

`arrow::compute` ships dozens of vectorized kernels. The ones you'll meet first:

```rust
use arrow::compute::{filter, sum, take, sort};

let total = sum(prices);                                // Option<Int64Array>
let kept  = filter(prices, &mask)?;                     // mask: BooleanArray
let perm  = sort(prices, None)?;                        // sorted permutation
```

These are written by Apache Arrow contributors; they're SIMD-friendly and battle-tested. You're going to write your own kernels for learning, but in production you'd compose these where possible.

---

## 3. parquet-rs — the file format

```toml
[dependencies]
parquet = { version = "50", features = ["arrow"] }
```

### Reading a Parquet file

```rust
use parquet::arrow::arrow_reader::ParquetRecordBatchReaderBuilder;
use std::fs::File;

fn read_parquet(path: &str) -> anyhow::Result<()> {
    let file   = File::open(path)?;
    let reader = ParquetRecordBatchReaderBuilder::try_new(file)?
        .with_batch_size(8192)
        .build()?;

    for batch in reader {
        let batch = batch?;
        println!("{} rows × {} cols", batch.num_rows(), batch.num_columns());
    }
    Ok(())
}
```

The reader is an `Iterator<Item = Result<RecordBatch>>`. Each batch is up to `batch_size` rows — same chunked-execution shape from Lesson 2 (§6).

### Writing

```rust
use parquet::arrow::ArrowWriter;
use parquet::file::properties::WriterProperties;
use parquet::basic::Compression;
use std::fs::File;

let props = WriterProperties::builder()
    .set_compression(Compression::ZSTD(Default::default()))
    .build();

let file   = File::create("out.parquet")?;
let mut writer = ArrowWriter::try_new(file, batch.schema(), Some(props))?;
writer.write(&batch)?;
writer.close()?;
```

Compression options: `SNAPPY` (fast, decent ratio), `ZSTD` (slower, best ratio), `LZ4` (fastest decode), `UNCOMPRESSED` (lol).

### Predicate pushdown

The killer feature: skip row groups based on per-column min/max statistics stored in the file footer.

```rust
use parquet::arrow::arrow_reader::{ArrowPredicate, RowFilter};

// Read only row groups where price > 100 has a chance of matching.
// Your predicate is consulted with statistics; row groups proven empty are skipped.
```

This is what makes `SELECT ... WHERE x > k LIMIT 10` over a 100GB Parquet dataset fast: 99% of row groups are never read from disk. Your engine should use this from day one.

---

## 4. object_store — local files, S3, GCS, Azure

```toml
[dependencies]
object_store = { version = "0.10", features = ["aws"] }
```

A unified `ObjectStore` trait abstracts local filesystem, S3, GCS, and Azure Blob Storage:

```rust
use object_store::{ObjectStore, path::Path};
use object_store::aws::AmazonS3Builder;

let s3 = AmazonS3Builder::from_env()
    .with_bucket_name("my-bucket")
    .build()?;

let bytes = s3.get(&Path::from("data/orders.parquet")).await?
    .bytes().await?;
```

Pair with `parquet-rs` to read S3-resident Parquet directly, no local download:

```rust
use parquet::arrow::async_reader::ParquetObjectReader;

let meta = s3.head(&Path::from("data/orders.parquet")).await?;
let reader = ParquetObjectReader::new(s3.into(), meta);
let stream = ParquetRecordBatchStreamBuilder::new(reader).await?.build()?;
```

For an analytical engine that wants to query lakes (S3, GCS), `object_store` is the right abstraction.

---

## 5. DataFusion — DuckDB-shaped, in Rust

```toml
[dependencies]
datafusion = "37"
tokio = { version = "1", features = ["full"] }
```

DataFusion is the closest thing to "DuckDB in Rust" that already exists. It's an embedded SQL engine built on Arrow and Parquet, with a query planner, optimizer, and parallel async executor. Its design heritage is closer to Spark / Calcite than DuckDB internally, but the user-facing shape is similar.

### Hello world (50 lines)

```rust
use datafusion::prelude::*;

#[tokio::main]
async fn main() -> datafusion::error::Result<()> {
    let ctx = SessionContext::new();

    ctx.register_parquet("orders", "orders.parquet", ParquetReadOptions::default()).await?;

    let df = ctx.sql(r#"
        SELECT region, SUM(price) AS total
        FROM orders
        WHERE amount > 100
        GROUP BY region
        ORDER BY total DESC
        LIMIT 10
    "#).await?;

    df.show().await?;
    Ok(())
}
```

That's a working SQL engine over Parquet with parallel execution, predicate pushdown, and a query optimizer. Roughly 50 lines.

### Internals worth knowing

- **`LogicalPlan`** — an enum with variants for each relational operator: `Filter`, `Projection`, `Aggregate`, `Join`, `TableScan`, `Limit`, `Sort`, `Window`, `Subquery`, …
- **`ExecutionPlan`** — a trait. Each physical operator (e.g., `HashAggregateExec`, `FilterExec`, `ParquetExec`) implements it.
- **`SendableRecordBatchStream`** — `Pin<Box<dyn Stream<Item = Result<RecordBatch>> + Send>>`. Operators produce streams of batches.
- **`TableProvider`** — trait you implement for a custom data source.
- **`ScalarUDF` / `AggregateUDF`** — register user-defined functions.

### Extending it

You can plug in:

- A custom `TableProvider` to expose any data source (a custom file format, an internal API, a rolling buffer of in-memory data).
- A `ScalarUDF` for a function the SQL engine should know about (e.g., `geohash(lat, lon)`).
- An `OptimizerRule` for plan rewrites (e.g., predicate pushdown into your custom source).
- A `PhysicalPlanner` extension for new operator types.

This is what production DataFusion users do: ship Arrow + DataFusion + their own glue, instead of writing a query engine from scratch.

---

## 6. sqlparser-rs — independent SQL parser

```toml
[dependencies]
sqlparser = "0.45"
```

```rust
use sqlparser::parser::Parser;
use sqlparser::dialect::PostgreSqlDialect;
use sqlparser::ast::Statement;

let dialect = PostgreSqlDialect {};
let sql = "SELECT a, b FROM t WHERE c = 1 ORDER BY a LIMIT 10";

let statements: Vec<Statement> = Parser::parse_sql(&dialect, sql)?;
println!("{:#?}", statements[0]);
```

You get a fully-typed Rust AST you can pattern-match on. Supports many dialects — PostgreSQL, MySQL, SQLite, Snowflake, Redshift, Hive, generic. DataFusion uses sqlparser internally; if you build your own engine, you'll use sqlparser too — there's no compelling reason to write your own SQL parser.

The work isn't parsing SQL — it's lowering the AST into a logical plan, then a physical plan.

---

## 7. Other crates worth knowing

| Crate                      | What for                                                             |
|----------------------------|----------------------------------------------------------------------|
| `comfy-table`               | Pretty-printing tables in a CLI                                      |
| `arrow-flight`              | gRPC service for transferring Arrow data                              |
| `iceberg-rs`, `delta-rs`    | Lakehouse table formats (Iceberg, Delta Lake) on top of Parquet      |
| `bytemuck`                  | Safe POD byte casts (Lesson 9)                                       |
| `crossbeam-channel`         | Faster channels than `std::sync::mpsc`                               |
| `parking_lot`               | Faster `Mutex` / `RwLock` than `std::sync`                           |
| `mimalloc`, `jemallocator`  | Allocator replacements; ~5–15% faster on alloc-heavy workloads       |
| `ahash`, `fxhash`           | Fast non-cryptographic hashers for `HashMap`                          |
| `num-traits`                | Generic numeric traits (`Num`, `Float`, `PrimInt`)                   |
| `serde`, `serde_json`       | Serialization                                                         |
| `tracing`                   | Structured logging / spans                                            |
| `criterion`                 | Benchmarking (Lesson 6)                                               |
| `proptest`                  | Property-based testing                                                |

You won't need all of these. Start with `arrow`, `parquet`, `tokio`, `anyhow`, `thiserror`, and `tracing`. Add others as you hit specific problems.

---

## 8. Build vs. extend vs. compose — the project decision

Three paths for your DuckDB-like project:

### Path A — Compose primitives, build your own engine

Use `arrow` for in-memory format, `parquet` for files, optionally `sqlparser` for parsing. Write your own logical/physical planner, executor, and operators.

- **Pros:** maximum learning. You touch every layer of an analytical engine. You own the architecture.
- **Cons:** SQL coverage will be a fraction of DataFusion's. Years to match production engines.
- **When to choose:** for learning, for a specialized engine where DataFusion's design is a poor fit (e.g., embedded, real-time, GPU).

### Path B — Extend DataFusion

Take DataFusion as-is. Add custom `TableProvider`s, UDFs, optimizer rules, or physical operators. Replace pieces you outgrow.

- **Pros:** working SQL engine in days, not years. Active community. Battle-tested execution.
- **Cons:** you don't learn how the planner/executor *works* — you learn how to *use* it.
- **When to choose:** for production. Most commercial Rust DB projects are this.

### Path C — From scratch, no `arrow-rs` either

Write your own columnar format, your own Parquet reader, your own everything.

- **Pros:** the most learning. You'll understand Apache Arrow's design choices by re-deriving them.
- **Cons:** ~10× the work of Path A. You'll get bored before you build anything interesting.
- **When to choose:** rarely. Maybe a research project where Arrow's design constraints are the thing you're studying.

### Recommendation for you

**Path A.** Use `arrow-rs` as the foundation (the format work isn't where the interesting DB design lives), but build your own scan, filter, project, aggregate, and join operators. You'll learn the most about DB internals while not reinventing low-value infrastructure.

After 3–6 months on Path A, you'll have:

- A working scan + filter + project + aggregate engine over Parquet.
- A clear sense of where your engine differs from DuckDB / DataFusion (and where it doesn't).
- Enough familiarity with the ecosystem to decide what you want to do next: keep building Path A, switch to extending DataFusion (Path B) for SQL coverage, or go deeper into a specific area (vectorized execution, distributed query, GPU).

---

## 9. Exercises

### 9.1 Hand-build a RecordBatch

Type out the `make_batch` example in §2. Print the schema, the row count, and the values of column 2 as a slice. Add a fourth column `quantity: Int64` with values `[1, 2, 3, 4, 5]` and a fifth `is_active: Boolean` with one null.

### 9.2 Round-trip Parquet

Take the batch from 9.1, write it to `out.parquet` with ZSTD compression, then read it back with `ParquetRecordBatchReaderBuilder`. Verify the schema and row count match.

### 9.3 DataFusion query

Write the 50-line "hello world" example. Generate a small Parquet file (use 9.2), point DataFusion at it, run a simple aggregation. Print the result.

Then run `df.explain(false, false).await?.show().await?` and read the resulting `LogicalPlan` and `PhysicalPlan` representations. Identify each operator.

### 9.4 SQL parsing

Parse this query with sqlparser:

```sql
SELECT region, SUM(price * quantity) AS revenue
FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31'
GROUP BY region
HAVING SUM(price * quantity) > 1000
ORDER BY revenue DESC
LIMIT 10
```

Walk the resulting AST. Find: (a) the `WHERE` predicate as an expression tree, (b) the `GROUP BY` keys, (c) the `HAVING` clause, (d) the `LIMIT` value. Print each.

### 9.5 Compose your own scan + sum (no DataFusion)

Open a Parquet file with `parquet-rs`, iterate batches, use `arrow::compute::sum` per batch, accumulate. Compare runtime against a DataFusion `SELECT SUM(col) FROM ...`. They should be within ~2× of each other (DataFusion has more overhead per batch but has parallelism).

### 9.6 Decide your path

For your `toydb` project, write 200 words defending one of paths A/B/C. What goal does this serve? What will the project look like at month 1, month 3, month 6? Keep it as a file in your repo.

### 9.7 `object_store` with a local path

Initialize a `LocalFileSystem` `ObjectStore`, list files in a directory, read one as bytes. Then swap in `AmazonS3Builder` (no real S3 needed — point at MinIO or just construct the builder and observe the API). Verify the same code shape works in both cases.

---

## 10. Self-check

1. What does `arrow-rs` give you that you'd never want to write from scratch?
2. What's a `RecordBatch` made of, in terms of memory layout?
3. Why is predicate pushdown into Parquet such a big win?
4. What's the relationship between `LogicalPlan` and `ExecutionPlan` in DataFusion?
5. Name three ways you'd extend DataFusion for a custom workload.
6. When would you build your own engine instead of extending DataFusion?
7. Why does `object_store` exist as a separate crate from `parquet-rs`?

Next: **Lesson 11 — In-memory formats and storage.** Inside the columnar format. Validity bitmaps, dictionary encoding, RLE, bit-packing, buffer pools, mmap.
