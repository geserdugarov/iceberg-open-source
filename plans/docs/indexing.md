# Iceberg Indexing Mechanisms

This document describes every form of "index" Apache Iceberg ships today: structures and statistics that prune data, accelerate scans, or accelerate row identification. It traces how each one is produced during writes, consumed during reads, maintained during compaction, and where there is headroom to improve performance.

**Related docs:** [architecture_overview.md](architecture_overview.md) | [read_path_callstack.md](read_path_callstack.md) | [write_path_callstack.md](write_path_callstack.md) | [compaction_callstack.md](compaction_callstack.md) | [delete_mechanisms.md](delete_mechanisms.md) | [update_and_merge_mechanisms.md](update_and_merge_mechanisms.md)

---

## 1. What Counts as an "Index" in Iceberg

Iceberg does not expose a `CREATE INDEX` DDL. Instead, it relies on a stack of statistics, layout-based clustering, and side-car files that engines consume as predicate pushdown progresses from the catalog down to individual Parquet/ORC pages. Conceptually, every layer is an index:

```
┌────────────────────────────────────────────────────────────────────────┐
│            ICEBERG PRUNING PIPELINE (top-down predicate flow)          │
│                                                                        │
│  Snapshot                                                              │
│    │  (1) Manifest list partition summaries                            │
│    ▼                                                                   │
│  Manifest files                                                        │
│    │  (2) ManifestEvaluator — prune whole manifests                    │
│    ▼                                                                   │
│  Manifest entries                                                      │
│    │  (3) InclusiveMetricsEvaluator — column lower/upper/nulls/NaN     │
│    ▼                                                                   │
│  DataFile / DeleteFile selected                                        │
│    │  (4) Parquet row-group filters: stats, dictionary, bloom          │
│    ▼                                                                   │
│  Surviving row groups                                                  │
│    │  (5) Parquet column index / offset index (page-level skipping):   │
│    │      applied only on the ReadSupport fallback path; NOT applied   │
│    │      on the custom `ReadConf` reader path used by Spark/Flink     │
│    │      engine integrations. See §2.5.                               │
│    ▼                                                                   │
│  Surviving pages                                                       │
│    │  (6) ResidualEvaluator — per-row filter                           │
│    ▼                                                                   │
│  Rows handed to engine                                                 │
│                                                                        │
│  Plus, in parallel:                                                    │
│    • Sort order / Z-order — write-time layout that boosts (3)–(5)      │
│    • Bucket transforms — partition-level hash index for equality       │
│    • Puffin side-cars: deletion vectors (positional delete index)      │
│                         theta sketches (NDV for CBO)                   │
│    • Statistics / partition-statistics files — table-wide stats        │
│    • Variant shredding — promotes JSON fields into stat-able columns   │
│    • Planning-time `DeleteFileIndex` + runtime `PositionDeleteIndex`   │
│      / `BitmapPositionDeleteIndex` — pair each data file with only the │
│      deletes that can apply to it (§2.10)                              │
└────────────────────────────────────────────────────────────────────────┘
```

The rest of this document walks each layer, then summarizes how they interact during the write, read, and compaction paths, and finally lists improvement opportunities.

---

## 2. Mechanisms

### 2.1 Partition Spec & Manifest-List Partition Summaries

Partitioning is the coarsest index. A `PartitionSpec` is a list of `PartitionField`s, each applying a `Transform` (`identity`, `bucket(N)`, `truncate(N)`, `year/month/day/hour`) to a source column. The transformed value is the partition key written into each data file's `partition` struct.

For every manifest file referenced from a snapshot's manifest list, Iceberg stores a per-partition-field `PartitionFieldSummary` with `lower_bound`, `upper_bound`, `contains_null`, and `contains_nan`. This is the first thing scan planning evaluates.

- **Classes:** `api/.../PartitionSpec.java`, `api/.../transforms/*.java`, `api/.../ManifestFile.java#PartitionFieldSummary`.
- **Write:** Computed by `ManifestWriter` as each entry is appended; flushed into the manifest list (`snap-*.avro`).
- **Read:** `ManifestEvaluator` rejects manifests whose summaries cannot intersect the query predicate.
- **Compaction:** Rewriting changes partition assignments only if the spec evolves; otherwise summaries are regenerated for the new manifests.

### 2.2 Manifest Evaluation

```
api/src/main/java/org/apache/iceberg/expressions/
  ManifestEvaluator.java
  InclusiveMetricsEvaluator.java
  StrictMetricsEvaluator.java
```

- `ManifestEvaluator`: takes a query expression and a `ManifestFile`; returns `true` if any entry could match given the partition-field summaries. Wired in `ManifestGroup.planFiles()` (see [read_path_callstack.md](read_path_callstack.md) §3, Phase 3).
- `InclusiveMetricsEvaluator`: applied per-`DataFile`/`DeleteFile` using per-column metrics; returns `true` if at least one row might match (the "inclusive" semantics).
- `StrictMetricsEvaluator`: complementary "must match all rows" semantics; used when planning DELETE/UPDATE rewrites and when promoting filters into residuals.

These three evaluators share an `ExpressionVisitors`-based design and operate purely on the metrics already stored in manifests — no extra I/O.

### 2.3 Per-File Column Metrics

Each `DataFile` (and `DeleteFile`) record in a manifest carries:

| Field                 | Purpose                                            |
|-----------------------|----------------------------------------------------|
| `record_count`        | Row count for aggregate pushdown and planning      |
| `file_size_in_bytes`  | Split planning, bin-packing in compaction          |
| `column_sizes`        | Per-column on-disk size (used by stats compaction) |
| `value_counts`        | Per-column row counts (for null-fraction logic)    |
| `null_value_counts`   | Drives `IS NULL` / `IS NOT NULL` pruning           |
| `nan_value_counts`    | Drives floating-point predicate pruning            |
| `lower_bounds`        | Per-column min — drives `>=`/`>`/`=`/range filters |
| `upper_bounds`        | Per-column max — drives `<=`/`<`/`=`/range filters |

What gets stored is governed by `MetricsModes` (`Full`, `Truncate(N)`, `Counts`, `None`) and `MetricsConfig`:

- **Defaults:** `write.metadata.metrics.default = truncate(16)`.
- **Per-column overrides:** `write.metadata.metrics.column.<name> = full|counts|truncate(N)|none`.
- **Classes:** `core/.../MetricsConfig.java`, `core/.../MetricsModes.java`, `core/.../MetricsUtil.java`.

Writers collect metrics as records flow through (`org.apache.iceberg.io.DataWriter` and the format-specific writers); the resulting `Metrics` object is attached to the `DataFile` returned from each task commit (see [write_path_callstack.md](write_path_callstack.md) §3, Phase 5).

### 2.4 Parquet Row-Group Filters

```
parquet/src/main/java/org/apache/iceberg/parquet/
  ParquetMetricsRowGroupFilter.java     ← min/max/nulls in row-group metadata
  ParquetDictionaryRowGroupFilter.java  ← dictionary-encoded pages, exact equality
  ParquetBloomRowGroupFilter.java       ← Parquet bloom filter (set membership)
```

Iceberg's Parquet reader composes all three at row-group boundaries. The metrics filter is always on; the dictionary filter activates when a column is fully dictionary-encoded for the row group; the bloom filter activates whenever the row group carries a serialized bloom filter for the referenced column.

Bloom filters are opt-in at write time via column-prefixed table properties:

| Property prefix                                | Default | Meaning                                       |
|------------------------------------------------|---------|-----------------------------------------------|
| `write.parquet.bloom-filter-enabled.column.<c>`| `false` | Write a bloom filter for column `c`           |
| `write.parquet.bloom-filter-fpp.column.<c>`    | `0.01`  | Target false positive probability             |
| `write.parquet.bloom-filter-ndv.column.<c>`    | —       | Hint at distinct-value count (sizes the bloom)|
| `write.parquet.bloom-filter-max-bytes`         | `1 MiB` | Cap on per-column bloom-filter size           |
| `write.parquet.stats-enabled.column.<c>`       | —       | Disable Parquet stats for high-card columns   |

`core/src/main/java/org/apache/iceberg/TableProperties.java` exposes these as `PARQUET_BLOOM_FILTER_*` constants.

### 2.5 Parquet Page Index (Column / Offset Index) — Two Iceberg Read Paths

Parquet's column index (per-page min/max) and offset index (page byte offsets) enable sub-row-group skipping when a filter is passed to `parquet-mr`. Iceberg has **two** Parquet read paths and they behave differently:

- **Custom reader path** — used by every engine integration that supplies a `readerFunc` / `batchedReaderFunc` (Spark, Flink, the data module's generic readers, etc.). Built in `parquet/.../ReadConf.java` and driven by `ParquetReader` / `VectorizedParquetReader`. It does **not** wire a Parquet `FilterCompat` filter into `ParquetFileReader`: it computes its own `shouldSkip[]` array using `ParquetMetricsRowGroupFilter`, `ParquetDictionaryRowGroupFilter`, and `ParquetBloomRowGroupFilter`, then iterates the surviving row groups with `readNextRowGroup()`. Page-index skipping is **not** exercised on this path. This is the path traced in [read_path_callstack.md](read_path_callstack.md) §3 Phase 5 and is the dominant path in practice.

- **ReadSupport fallback path** — used when neither `readerFunc` nor `batchedReaderFunc` is set (e.g. the Avro `ReadSupport`-based call path inside `Parquet.read()`, roughly `parquet/.../Parquet.java:1526–1594`). This branch goes through a vanilla `parquet-mr` `ParquetReader.Builder` (`ParquetReadBuilder`) and explicitly opts into the full `parquet-mr` filter machinery:

  ```
  builder
      .useStatsFilter()
      .useDictionaryFilter()
      .useRecordFilter(filterRecords)
      .useBloomFilter()
      .withFilter(ParquetFilters.convert(fileSchema, filter, caseSensitive));
  ```

  Here `withFilter(...)` + `useRecordFilter(true)` engage the column-index / offset-index machinery in `parquet-mr`, so this path **does** benefit from page-index skipping (and from `parquet-mr`'s own stats/dictionary/bloom filtering).

Net effect: files Iceberg writes may carry column/offset indexes, but whether page-level skipping happens at read time depends on which Iceberg Parquet path the caller is on. Wiring `FilterCompat` (and `useRecordFilter(true)`) into the custom path so it matches the `ReadSupport` path is a tracked improvement opportunity (see §4.1).

### 2.6 ORC Row Index & Bloom Filters

ORC files contain a native row index per stripe and may include per-column bloom filters. Iceberg writes them via the ORC `WriterImpl`, configured by:

| Property                          | Default | Meaning                                         |
|-----------------------------------|---------|-------------------------------------------------|
| `write.orc.bloom.filter.columns`  | `""`    | Comma-separated column names to bloom-filter    |
| `write.orc.bloom.filter.fpp`      | `0.05`  | False positive probability                      |

(Constants `ORC_BLOOM_FILTER_COLUMNS` / `ORC_BLOOM_FILTER_FPP` in `TableProperties`.)

On the read side, Iceberg pushes the converted Iceberg expression into ORC's `SearchArgument`, and ORC's reader applies row-index and bloom-filter pruning before returning batches.

### 2.7 Sort Order & Z-Order (Data Layout Indexes)

A `SortOrder` is recorded in table metadata (`api/.../SortOrder.java`) and applied at write time and during compaction. Sorting tightens per-file lower/upper bounds, which makes the metric evaluators in §2.2–§2.3 prune far more aggressively. Iceberg supports two layouts as compaction strategies:

- **Sort:** `SparkSortFileRewriteRunner` performs range-partitioning + sort by the configured order.
- **Z-Order:** `SparkZOrderFileRewriteRunner` interleaves the bits of multiple normalized columns using `core/src/main/java/org/apache/iceberg/util/ZOrderByteUtils.java`. The Z-order curve preserves locality across all clustering columns, giving multi-dimensional pruning benefits without favoring any single column.

Z-order is purely a data-layout index — no extra side-car is written. Its benefit shows up downstream as tighter `lower_bounds`/`upper_bounds` in manifests and tighter Parquet row-group / page-level stats inside each file (which then feed §2.4's filters, and any reader that opts into Parquet's page index).

### 2.8 Bucket Transforms (Hash Index at Partition Level)

`Transforms.bucket(col, N)` (`api/.../transforms/Bucket.java`) hashes a value into `N` buckets and uses the bucket id as the partition key. This is the closest thing Iceberg has to a hash index: equality predicates on the bucketed column let the planner skip every partition whose bucket id does not match. It also enables bucketed joins in Spark when both sides share the same spec.

### 2.9 Deletion Vectors (V3+)

Deletion vectors are Iceberg's positional-delete index. A single Puffin blob with type `deletion-vector-v1` (`core/.../puffin/StandardBlobTypes.java`) stores a RoaringBitmap of deleted row positions for one data file. The matching `DeleteFile` record carries a `referencedDataFile`, `contentOffset`, and `contentSizeInBytes` so the reader can map a data file directly to its DV without scanning all delete files in the partition. See [delete_mechanisms.md](delete_mechanisms.md) §6 for the full read/write contract.

- **Write classes:** `core/.../deletes/BaseDVFileWriter.java`, `core/.../DeletionVector.java`, `core/.../DeletionVectorStruct.java`.
- **Read classes:** `data/.../BaseDeleteLoader.java` (loads bitmaps and applies them in `DeleteFilter`).

In V3+, all *new* positional deletes must be encoded as deletion vectors — writers must not add new position delete files. Pre-existing V2 position delete files remain valid after a V2→V3 upgrade and may be read until the next write that touches the affected data file merges them into a DV. Equality delete files are not replaced by DVs and remain valid in V3 (and V4) for equality-based MoR deletes. See [delete_mechanisms.md](delete_mechanisms.md) and `format/spec.md` §"Row-level Deletes" / "Deletion Vectors" for the exact rules. DVs behave as an O(1) per-row "is-deleted?" lookup.

### 2.10 Planner-Side `DeleteFileIndex` and Runtime `PositionDeleteIndex`

Two in-memory indexes sit between the persistent delete artifacts of §2.9 and the engine row stream:

**`DeleteFileIndex`** (`core/src/main/java/org/apache/iceberg/DeleteFileIndex.java`) — built by `ManifestGroup` during scan planning. It keys delete files by **data sequence number** and partitions them along several axes so `forDataFile(seq, dataFile)` returns just the deletes that can possibly match one data file in O(small) time. The applicability rule depends on the delete type (per `format/spec.md` §"Scan Planning"):

- **Equality delete files** apply to a data file only when `data_file.data_seq < equality_delete.data_seq` (strictly less than). This prevents a same-commit equality delete from removing rows added in that commit.
- **Position delete files and deletion vectors** apply when `data_file.data_seq ≤ delete.data_seq` (less than or equal). The equal case is required so position deletes can remove rows added in the same commit (positions are absolute file/offset references, so collisions are impossible).

The actual fields `DeleteFileIndex` maintains are:

- `globalDeletes` — unpartitioned equality deletes (apply across partitions).
- `eqDeletesByPartition` — partition-keyed equality deletes (`PartitionMap<EqualityDeletes>`).
- `posDeletesByPartition` — partition-keyed position delete files that span a partition.
- `posDeletesByPath` — path-keyed position delete files when `referenced_data_file` is set so they apply to exactly one data file.
- `dvByPath` — at most one deletion-vector `DeleteFile` per data file path.

The result is that every `FileScanTask` yielded by `ManifestGroup.planFiles()` carries only the delete files it actually has to consult, instead of the full per-snapshot delete set. See [read_path_callstack.md](read_path_callstack.md) §3 Phase 3 for where this fits in the planning call stack.

**`PositionDeleteIndex` / `BitmapPositionDeleteIndex`** (`core/src/main/java/org/apache/iceberg/deletes/PositionDeleteIndex.java`, `core/.../deletes/BitmapPositionDeleteIndex.java`) — the runtime "is-this-row-deleted?" lookup used inside `DeleteFilter` on the read path. For both V2 position delete files and V3 deletion vectors, deletes are materialized into a `RoaringBitmap`-backed `BitmapPositionDeleteIndex` per data file, then queried by row position as rows stream out of the format reader. Equality deletes use a separate hash-set path. The DV blob format is just a serialization of this same bitmap structure (§2.9), so the V2 position-delete path and the V3 DV path converge on the same in-memory representation at read time. `PositionDeleteIndexUtil` (`core/.../deletes/PositionDeleteIndexUtil.java`) and `BaseDeleteLoader` (`data/.../BaseDeleteLoader.java`) are the builders/loaders.

These two indexes are not persisted to the table — they are per-scan, in-memory accelerators that turn O(deletes × rows) into O(rows) for the read path.

### 2.11 Puffin Blobs & Statistics Files

`StatisticsFile` (`api/.../StatisticsFile.java`) and `GenericStatisticsFile` describe a Puffin file attached to a snapshot. Two standard blob types live in `StandardBlobTypes`:

| Blob type                       | Purpose                                              |
|---------------------------------|------------------------------------------------------|
| `apache-datasketches-theta-v1`  | Compact Theta sketch for NDV (distinct counts)       |
| `deletion-vector-v1`            | Roaring bitmap of deleted positions (§2.9)           |

Theta sketches are produced by `ComputeTableStatsSparkAction` (and its sibling procedure `ComputeTableStatsProcedure`) via `spark/.../actions/NDVSketchUtil.java`, and surface to engines through `Table.statisticsFiles()`. The Spark CBO uses them to estimate join cardinalities and filter selectivities.

### 2.12 Partition Statistics Files

`PartitionStatisticsFile` (`api/.../PartitionStatisticsFile.java`) points at a per-partition statistics side-car attached to a snapshot. Per the format spec (§Partition Statistics File), one row is written per unique partition tuple in any of the table's data file formats (for example Parquet, ORC, or Avro), sorted ascending on `partition`. The schema includes `partition`, `spec_id`, `data_record_count`, `data_file_count`, `total_data_file_size_in_bytes`, and (optionally / required from V3) position / equality delete counts, deletion-vector counts, total record count, and last-updated timestamps.

These files are explicitly **informational**: the `PartitionStatisticsFile` Javadoc states "A reader can choose to ignore statistics information. Statistics support is not required to read the table correctly," and the spec states they are not required for reading or planning and readers may ignore them. They are written by `ComputePartitionStats` (action / procedure) and registered in `metadata.json`'s `partition-statistics` list, surfaced on `Table` via `partitionStatisticsFiles()`. Default scan planning does **not** consult them — they are exposed through Iceberg's partition statistics scan APIs for callers that opt in (for example metadata-table consumers and tooling).

Note also that this is a separate surface from `Table.statisticsFiles()` (§2.11): the Spark CBO reads the snapshot-level `StatisticsFile`s (theta sketches in Puffin) returned by `statisticsFiles()`, not the per-partition side-cars described here.

(Note: the deprecated in-manifest partition-stats *read* path was removed in commit f37a04b53; the side-car file is now the canonical surface.)

### 2.13 Variant Shredding

Variant is Iceberg's semi-structured type for JSON-like payloads. Without help, predicates on a variant are opaque to every stat-based pruner above. Shredding promotes selected paths into dedicated Parquet columns next to the original variant blob. The shredding *schema* is inferred per write batch by `VariantShreddingAnalyzer` (`parquet/.../VariantShreddingAnalyzer.java`) from the buffered variant values themselves — not from query history. The analyzer:

- buffers up to `write.parquet.variant-inference-buffer-size` rows (default `100`),
- collapses the observed field tree with deterministic ordering (alphabetical at each level),
- promotes types per `TIE_BREAK_PRIORITY` (most-common wins; ties broken explicitly),
- prunes fields below `MIN_FIELD_FREQUENCY` (default `0.10`) and caps at `MAX_SHREDDED_FIELDS` (default `300`),
- limits recursion to `MAX_SHREDDING_DEPTH` (default `50`).

Knobs:

- **Toggle:** `write.parquet.shred-variants = false` (default off; constant `PARQUET_SHRED_VARIANTS`).
- **Buffer:** `write.parquet.variant-inference-buffer-size = 100` (`PARQUET_VARIANT_BUFFER_SIZE`).
- **Recent work:** Flink writer support was added in commit 10ba4eebf ("Flink: Support writing shredded variant").

Once a path is shredded it becomes an ordinary Parquet column. What stats Iceberg records for it then follows the table's `MetricsConfig` — so shredded columns participate in lower/upper bounds, null counts, row-group stats and bloom filters subject to the same metrics mode and column-level overrides as any other column. Shredding therefore does not itself store new statistics; it expands the set of fields the other indexes *can* prune on, provided `MetricsConfig` lets them.

### 2.14 Metrics Modes (Index-Tuning Knob)

`MetricsModes` controls which of §2.3's metrics are stored, and at what fidelity:

| Mode          | Stores                                                                                  | Cost           |
|---------------|-----------------------------------------------------------------------------------------|----------------|
| `Full`        | Untruncated `lower_bounds` / `upper_bounds` + `value_counts` / `null_value_counts` / `nan_value_counts` | Highest        |
| `Truncate(N)` | `lower_bounds` / `upper_bounds` truncated to `N` UTF-8 bytes + `value_counts` / `null_value_counts` / `nan_value_counts` | Default        |
| `Counts`      | `value_counts` + `null_value_counts` + `nan_value_counts` (no bounds)                   | Cheap          |
| `None`        | Nothing                                                                                 | Smallest       |

Defaults: `Truncate(16)` for the first 100 columns; explicit `None` is recommended for very high-cardinality string columns where truncated bounds add noise without enabling pruning.

---

## 3. How Indexes Are Used During Each Path

### 3.1 Write Path

```
DataWriter.write(row)
  └─► record-level metrics accumulator updates:
        column null/NaN counts, lower/upper bounds (per MetricsMode)
  └─► ParquetWriter.add(row)
        └─► dictionary, bloom filter (if enabled per column),
            row-group statistics, column/offset index buffered

ParquetWriter.close()
  └─► flush page index + bloom filters into the footer
  └─► return Metrics (input for DataFile metadata)

DataWriter.commit()
  └─► DataFile{ record_count, file_size_in_bytes,
                column_sizes, value_counts, null_value_counts,
                nan_value_counts, lower_bounds, upper_bounds }
  └─► sent to driver

SnapshotProducer.commit()
  └─► ManifestWriter writes per-entry metrics
  └─► ManifestListWriter computes partition field summaries
        (per-partition lower/upper/null/nan across the manifest)
  └─► (optional) Puffin StatisticsFile attached via .updateStatistics()
        — typically produced by a separate action, not the writer itself
```

So writes do not produce side-car indexes by default; they produce the file-/manifest-level metrics that every other layer depends on. Side-car indexes (theta sketches, partition stats) are typically built by the dedicated actions `ComputeTableStats` and `ComputePartitionStats` after a write.

### 3.2 Read Path

```
SparkScanBuilder.pushPredicates(...)
  └─► Iceberg Expression
        │
        ├─► (1) ManifestEvaluator   — manifest-list partition summaries
        ├─► (2) InclusiveMetricsEvaluator — per-file metrics (§2.3)
        ├─► (3) ResidualEvaluator   — remaining predicate carried into reader
        │
        └─► passed to format reader:
              Parquet (custom path, used by Spark/Flink engines):
                     → MetricsRowGroupFilter + DictionaryRowGroupFilter
                       + BloomRowGroupFilter
                     (no page-index skipping — see §2.5)
              Parquet (ReadSupport fallback path):
                     → parquet-mr useStatsFilter + useDictionaryFilter
                       + useBloomFilter + useRecordFilter + withFilter
                       (engages column/offset index → page-level skipping)
              ORC    → SearchArgument → row index + bloom

SparkDeleteFilter.filter(rows)
  └─► consults the DeleteFileIndex entry attached to the FileScanTask
        (§2.10) — already narrowed to deletes that can apply to this file
  └─► BaseDeleteLoader materializes V2 position deletes / V3 DVs into
        a BitmapPositionDeleteIndex (Roaring), and equality deletes into
        a hash set
  └─► applies positional bitmap / equality lookup per row
```

Each layer is conservative: a row that survives must satisfy every layer's "could match" predicate. Pruning therefore composes — the more selective the predicate, the more aggressively higher layers eliminate work.

### 3.3 Compaction Path

Compaction (`RewriteDataFilesSparkAction`; see [compaction_callstack.md](compaction_callstack.md)) rewrites *data*, which means it also rewrites *every index* derived from that data:

- **Bin-pack:** Rewrites the rows through the regular write path, so per-file `lower_bounds`/`upper_bounds`, bloom filters, and in-file indexes are all recomputed from the output rows by the writers (not copied from input files). The strategy does **not** deliberately recluster — rows keep their incoming order and `DISTRIBUTION_MODE=none` means no shuffle (see [compaction_callstack.md](compaction_callstack.md) §2 Phase 3). Net effect on pruning is therefore mostly mechanical: fewer, larger files with stats computed over the merged row sets, without the tighter per-file bounds that sort/Z-order produce.
- **Sort / Z-Order:** Re-clusters rows so per-file bounds become much tighter. This is the primary way operators improve `(2)` and `(3)` after the fact.
- **Bloom filters & in-file indexes (column index, offset index, ORC row index):** Regenerated by the new writers per current column-level properties — for example a bloom filter newly enabled on a column will appear only in files compaction has touched.
- **Deletion vectors:** When a data file is compacted, its associated DV becomes "dangling" and is dropped via `RewriteFiles.deleteFile(<DeleteFile>)` (commit manager already aggregates `danglingDVs`).
- **Statistics files:** Not rewritten by `rewrite_data_files`; they remain attached to their original snapshot. Re-running `ComputeTableStats` after a major compaction refreshes them.
- **Partition statistics files:** Same as above — managed via `compute_partition_stats`.

Sort order and Z-order are special: they are the only Iceberg "indexes" whose effectiveness is bounded by *how recently* compaction has run. Stale layout means stale pruning.

---

## 4. Improvement Opportunities

This section is forward-looking — concrete directions where pruning could become tighter, side-cars richer, or runtime cheaper.

### 4.1 Tighter & Cheaper File-Level Stats

- **Wire Parquet page-index skipping into the custom Iceberg reader.** The `ReadSupport` fallback path already engages `parquet-mr`'s page-index machinery (§2.5), but the custom `ReadConf` path used by every engine integration only applies row-group filters (stats / dictionary / bloom) and iterates surviving row groups directly. Passing the residual through `FilterCompat` (with `useRecordFilter(true)`) on the custom path would close that gap.
- **Adaptive metrics modes.** Today `Truncate(16)` is the default for every column under index 100. A profile-guided mode (sample cardinality, then choose `Full` for low-card, `Counts` for free-text, `None` for unique IDs) would shrink manifests and improve pruning on the columns that matter.
- **Histogram / quantile sketches.** Iceberg only stores min/max + counts. Adding KLL or t-digest sketches (already used by the Datasketches NDV path) per column would let CBO estimate selectivity for range predicates instead of falling back to uniform-distribution defaults.
- **Suffix indexes / range pre-aggregations** in manifest list summaries for hierarchical strings (e.g. URLs) — analogous to the Parquet truncated bounds but with a prefix tree.

### 4.2 Richer Side-Car Indexes (Puffin)

Puffin is intentionally extensible (any `(type, properties)` blob). Candidates worth standardizing:

- **Per-file bloom filter blobs** on values that change too often to live in Parquet footers but are too expensive to scan every row group for. Useful for join keys.
- **MinMax indexes for variant subfields** that are not shredded — would let evaluators prune on JSON paths cheaply.
- **Cuckoo / xor filters** as smaller / faster alternatives to Parquet bloom filters for membership tests with known false-positive budgets.
- **Sort-order proof blobs**: declare that a file is in fact sorted by `(c1, c2)`; today scan planning cannot exploit per-file sort even when it exists.

### 4.3 Layout Improvements

- **Asynchronous incremental Z-order / sort maintenance.** `rewrite_data_files` is a heavy batch operation. An incremental rewriter that targets only the *most-disordered* partitions (measured by an "order entropy" stat in the manifest) would keep pruning sharp without operator intervention.
- **Bucket transforms with adaptive bucket counts.** Today `bucket(N)` is fixed at table creation; evolving buckets is unsafe. A bucketing scheme with consistent hashing would let `N` grow with the table without rewriting every partition.
- **Skipping indexes co-located with sort order.** When a sort order is declared, per-file metrics could be augmented with the *exact* split-point set (e.g. block-level minmax for sort columns), allowing point lookups to seek directly to the relevant block without scanning the row group.

### 4.4 Cheaper Planning

- **Drive adoption of server-side REST scan planning.** The REST catalog spec already exposes `/v1/{prefix}/namespaces/{ns}/tables/{table}/plan` and `/plan/{plan-id}` (see `core/.../rest/RESTTableScan.java`, `CatalogHandlers.planTableScan`, and the OpenAPI scan-planning endpoints), so the client can ask the server to do the manifest walk and return only the surviving `FileScanTask`s. The opportunity is in adoption (most engines still plan locally with `ManifestEvaluator`) and capability (richer filter pushdown, incremental / paginated responses, server-side caches of `(manifest, predicate)` → surviving files), not in adding the endpoint itself.
- **Reuse `InclusiveMetricsEvaluator` results across refinements.** A `TableScan` produces new scans on every `filter()`/`select()`. State doesn't survive — but a session-scoped cache of `(manifest, partition)` → `surviving DataFile[]` for the same base predicate would help interactive workloads.
- **Vectorize the metrics evaluators.** Both `ManifestEvaluator` and `InclusiveMetricsEvaluator` walk a tree of `ExpressionVisitors`. Compiling each predicate to a single tight loop over the `DataFile` struct would materially cut planning CPU on large tables.

### 4.5 Index Maintenance UX

- **`ANALYZE TABLE`-style consolidation.** Today producing every index (`compute_table_stats`, `compute_partition_stats`, future bloom side-cars) requires a different procedure. A single action that brings *all* indexes up to a target snapshot would simplify operations.
- **First-class freshness reporting.** A metadata table that lists each side-car index, its source snapshot, and a freshness score would let operators schedule maintenance.
- **Auto-pruning of stale Puffin blobs.** Once a snapshot expires, the dangling sketches it referenced should be GC'd by `RemoveOrphanFiles` automatically (today this depends on each blob type being well-known to the cleaner).

### 4.6 Beyond Pruning: True Secondary Indexes

- A **format-spec-level secondary index** (e.g. value → file/position) is still an open design discussion. It would unlock point lookups at the table level — today the only "row addressing" Iceberg has is the implicit `(file_path, pos)` used by deletion vectors. Any such index must integrate with snapshot isolation: it has to be referenced from `metadata.json` and roll forward with rewrites just like manifests do.

---

## 5. Key Classes Reference

| Layer / Mechanism             | Class                                  | Module / Path                                                 |
|-------------------------------|----------------------------------------|---------------------------------------------------------------|
| Partition spec & transforms   | `PartitionSpec`, `Transforms`, `Bucket`| `api/.../`, `api/.../transforms/`                             |
| Manifest pruning              | `ManifestEvaluator`                    | `api/.../expressions/ManifestEvaluator.java`                  |
| Inclusive file pruning        | `InclusiveMetricsEvaluator`            | `api/.../expressions/InclusiveMetricsEvaluator.java`          |
| Strict file pruning           | `StrictMetricsEvaluator`               | `api/.../expressions/StrictMetricsEvaluator.java`             |
| Metrics modes                 | `MetricsModes`, `MetricsConfig`        | `core/.../MetricsModes.java`, `core/.../MetricsConfig.java`   |
| Parquet stats filter          | `ParquetMetricsRowGroupFilter`         | `parquet/.../ParquetMetricsRowGroupFilter.java`               |
| Parquet dictionary filter     | `ParquetDictionaryRowGroupFilter`      | `parquet/.../ParquetDictionaryRowGroupFilter.java`            |
| Parquet bloom filter          | `ParquetBloomRowGroupFilter`           | `parquet/.../ParquetBloomRowGroupFilter.java`                 |
| Z-order encoding              | `ZOrderByteUtils`                      | `core/.../util/ZOrderByteUtils.java`                          |
| Sort order metadata           | `SortOrder`, `SortField`               | `api/.../SortOrder.java`                                      |
| Compaction strategies         | `SparkBinPack/Sort/ZOrderFileRewriteRunner` | `spark/v*/spark/.../actions/`                            |
| Deletion vectors              | `DeletionVector`, `BaseDVFileWriter`   | `core/.../DeletionVector.java`, `core/.../deletes/`           |
| Planner delete index          | `DeleteFileIndex`                      | `core/.../DeleteFileIndex.java`                               |
| Runtime delete bitmap         | `PositionDeleteIndex`, `BitmapPositionDeleteIndex` | `core/.../deletes/PositionDeleteIndex.java`, `core/.../deletes/BitmapPositionDeleteIndex.java` |
| Delete loader                 | `BaseDeleteLoader`                     | `data/.../BaseDeleteLoader.java`                              |
| Puffin format                 | `Puffin`, `PuffinReader`, `PuffinWriter`| `core/.../puffin/`                                           |
| Standard blob types           | `StandardBlobTypes`                    | `core/.../puffin/StandardBlobTypes.java`                      |
| Snapshot statistics file      | `StatisticsFile`                       | `api/.../StatisticsFile.java`                                 |
| Partition statistics file     | `PartitionStatisticsFile`              | `api/.../PartitionStatisticsFile.java`                        |
| Theta sketch builder          | `NDVSketchUtil`                        | `spark/v*/spark/.../actions/NDVSketchUtil.java`               |
| Stats action                  | `ComputeTableStatsSparkAction`         | `spark/v*/spark/.../actions/`                                 |
| Variant shredding toggle      | `TableProperties.PARQUET_SHRED_VARIANTS`| `core/.../TableProperties.java`                              |

---

## 6. Configuration Reference

### 6.1 File-Level Metrics

| Property                                | Default        | Purpose                                          |
|-----------------------------------------|----------------|--------------------------------------------------|
| `write.metadata.metrics.default`        | `truncate(16)` | Default metrics mode for every column            |
| `write.metadata.metrics.column.<col>`   | —              | Per-column override                              |

### 6.2 Parquet

| Property                                          | Default | Purpose                                       |
|---------------------------------------------------|---------|-----------------------------------------------|
| `write.parquet.bloom-filter-enabled.column.<col>` | `false` | Enable bloom filter for column                |
| `write.parquet.bloom-filter-fpp.column.<col>`     | `0.01`  | Target false positive probability             |
| `write.parquet.bloom-filter-ndv.column.<col>`     | —       | NDV hint for sizing the filter                |
| `write.parquet.bloom-filter-max-bytes`            | `1 MiB` | Cap per-column bloom-filter size              |
| `write.parquet.stats-enabled.column.<col>`        | —       | Disable Parquet stats for a column            |
| `write.parquet.shred-variants`                    | `false` | Promote variant subfields to columns          |
| `write.parquet.variant-inference-buffer-size`     | `100`   | Sample size for variant shredding inference   |

### 6.3 ORC

| Property                          | Default | Purpose                                  |
|-----------------------------------|---------|------------------------------------------|
| `write.orc.bloom.filter.columns`  | `""`    | Comma-separated bloom-filter columns     |
| `write.orc.bloom.filter.fpp`      | `0.05`  | Target false positive probability        |

### 6.4 Statistics Side-Cars (run as actions)

| Action / Procedure              | What it produces                                |
|---------------------------------|-------------------------------------------------|
| `CALL system.compute_table_stats`    | `StatisticsFile` with theta sketches per column |
| `CALL system.compute_partition_stats`| `PartitionStatisticsFile` for the snapshot      |
| `CALL system.rewrite_data_files`     | Recomputes per-file metrics, bloom filters, page indexes; refreshes data layout (sort / Z-order) |
| `CALL system.rewrite_position_delete_files` | Compacts position deletes (consolidates DV references in V3)|

---

## 7. Summary

Iceberg's "indexing" is a layered set of statistics and side-cars:

1. **Always on:** partition summaries, per-file metrics, Parquet/ORC native row-group/stripe stats. (Parquet's column/offset page index is written into files and is consumed by Iceberg's `ReadSupport` fallback Parquet path; the custom `ReadConf` path used by Spark/Flink engine integrations skips only row groups — see §2.5. ORC's native row index is applied transparently by the ORC reader.)
2. **Opt-in at write time:** Parquet/ORC bloom filters per column, variant shredding.
3. **Computed by actions:** NDV theta sketches in Puffin (`StatisticsFile`), partition statistics files, sort/Z-order layout via `rewrite_data_files`.
4. **Produced by row-level operations:** position delete files and equality delete files (V2+); deletion vectors in Puffin (V3+, the required encoding for new positional deletes). Equality delete files remain valid in V3/V4; pre-existing V2 position delete files remain readable after upgrade but new positional deletes must be DVs.

Every layer composes during reads via successive evaluators (`ManifestEvaluator` → `InclusiveMetricsEvaluator` → format-specific row-group filters → `ResidualEvaluator`). Compaction is the operational lever that keeps the lower layers tight; statistics actions keep the side-car layers fresh. The most promising areas for improvement are richer per-column sketches (quantiles), Puffin-backed bloom/secondary indexes, lighter-weight planning (vectorized evaluators, server-side manifest pruning), wiring Parquet page-index skipping into the custom `ReadConf` path, and unified "ANALYZE-style" maintenance.
