---
title: "Spark DataSource V2 Read Path"
---
<!--
 - Licensed to the Apache Software Foundation (ASF) under one or more
 - contributor license agreements.  See the NOTICE file distributed with
 - this work for additional information regarding copyright ownership.
 - The ASF licenses this file to You under the Apache License, Version 2.0
 - (the "License"); you may not use this file except in compliance with
 - the License.  You may obtain a copy of the License at
 -
 -   http://www.apache.org/licenses/LICENSE-2.0
 -
 - Unless required by applicable law or agreed to in writing, software
 - distributed under the License is distributed on an "AS IS" BASIS,
 - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 - See the License for the specific language governing permissions and
 - limitations under the License.
 -->

# Spark DataSource V2 Read Path

This document describes the internal architecture of Iceberg's Spark DataSource V2 read path. It traces how a query against an Iceberg table flows from Spark's catalog resolution all the way to reading individual data files on executors.

## Overview

The read path follows this high-level flow:

```
SQL Query / DataFrame API
  → SparkCatalog / IcebergSource (table resolution)
    → SparkTable.newScanBuilder() (scan builder creation)
      → SparkScanBuilder (pushdown optimization)
        → SparkBatchQueryScan (scan planning)
          → SparkBatch → SparkInputPartition[] (task distribution)
            → SparkRowReaderFactory / SparkColumnarReaderFactory (reader creation)
              → RowDataReader / BatchDataReader (file reading on executors)
```

## Entry Point and Table Resolution

### IcebergSource

`IcebergSource` (`o.a.i.spark.source.IcebergSource`) is the DataSource V2 entry point registered under the `iceberg` format name via `META-INF/services`. It implements:

- `TableProvider` — enables `spark.read.format("iceberg").load("path")` usage
- `SupportsCatalogOptions` — allows extracting catalog/namespace/table identifiers from options

When Spark resolves a table reference, `IcebergSource` provides the bridge between Spark's data source API and Iceberg's catalog system.

### SparkCatalog and SparkSessionCatalog

For catalog-based table access (e.g., `spark.sql("SELECT * FROM catalog.db.table")`):

- `SparkCatalog` implements Spark's `TableCatalog` and `SupportsNamespaces` — the primary catalog for Iceberg tables
- `SparkSessionCatalog` wraps Spark's built-in session catalog to add Iceberg table support alongside non-Iceberg tables

Both return `SparkTable` instances when loading a table.

### SparkTable

`SparkTable` (`o.a.i.spark.source.SparkTable`) wraps an Iceberg `Table` and exposes it to Spark's query planning infrastructure. It implements:

| Interface | Purpose |
|-----------|---------|
| `SupportsRead` | Provides `newScanBuilder()` for read operations |
| `SupportsWrite` | Provides `newWriteBuilder()` for write operations |
| `SupportsDeleteV2` | Enables metadata-only DELETE WHERE operations |
| `SupportsRowLevelOperations` | Enables row-level DELETE, UPDATE, MERGE |
| `SupportsMetadataColumns` | Exposes metadata columns (_spec_id, _partition, _file, _pos, _deleted, etc.) |

Key read-related capabilities declared:

- `BATCH_READ` — standard batch scan
- `MICRO_BATCH_READ` — streaming micro-batch reads

The `newScanBuilder()` method creates either a `SparkStagedScanBuilder` (for pre-planned scan task sets, used by actions like rewrite) or a `SparkScanBuilder` (for standard queries). When the table was loaded with a specific branch, the scan operates against that branch's snapshot.

#### Metadata Columns

`SparkTable.metadataColumns()` exposes the following columns that queries can reference:

- `_spec_id` — partition spec ID
- `_partition` — partition struct
- `_file` — data file path
- `_pos` — row position within the file
- `_deleted` — whether the row is marked as deleted
- `_row_id` and `_last_updated_sequence_number` (when row lineage is supported)

## Query Optimization (SparkScanBuilder)

`SparkScanBuilder` (`o.a.i.spark.source.SparkScanBuilder`) is where Spark's optimizer interacts with Iceberg to push down query operations. It implements multiple pushdown interfaces:

### Predicate Pushdown (SupportsPushDownV2Filters)

`pushPredicates(Predicate[])` receives Spark V2 filter predicates and classifies each into three categories:

1. **Fully pushed** — filters that select entire partitions (evaluated entirely by Iceberg, not returned as post-scan filters)
2. **Partially pushed** — filters pushed to Iceberg for file pruning but still evaluated by Spark for row-level filtering
3. **Not pushed** — unsupported filters evaluated entirely by Spark

The conversion uses `SparkV2Filters.convert(Predicate)` to translate Spark predicates into Iceberg `Expression` objects. Supported operations include:

| Spark Predicate | Iceberg Expression |
|-----------------|-------------------|
| `=`, `<>`, `<`, `<=`, `>`, `>=` | `equal`, `notEqual`, `lessThan`, `lessThanOrEqual`, `greaterThan`, `greaterThanOrEqual` |
| `IS_NULL`, `IS_NOT_NULL` | `isNull`, `notNull` |
| `IN` | `in`, `notIn` |
| `STARTS_WITH` | `startsWith` |
| `AND`, `OR`, `NOT` | `and`, `or`, `not` |
| `ALWAYS_TRUE`, `ALWAYS_FALSE` | `alwaysTrue`, `alwaysFalse` |

`SparkV2Filters` also supports pushdown of Iceberg transform functions (`years`, `months`, `days`, `hours`, `bucket`, `truncate`) when used in filter predicates via `UserDefinedScalarFunc`.

After conversion, each expression is bound against the table schema to verify it can be pushed down. The method returns the post-scan filters that Spark must still evaluate.

### Column Projection (SupportsPushDownRequiredColumns)

`pruneColumns(StructType)` receives the requested schema from Spark's optimizer and prunes the Iceberg schema to include only the requested columns. Metadata columns (prefixed with `_`) are tracked separately and merged into the schema later.

The pruning also considers columns referenced in filter expressions to ensure they are available for residual predicate evaluation, even if not part of the query's SELECT list.

### Aggregate Pushdown (SupportsPushDownAggregates)

`pushAggregation(Aggregation)` attempts to evaluate simple aggregations (COUNT, MIN, MAX) entirely from file-level metadata statistics, avoiding full file scans. This is only possible when:

- The table is a `BaseTable` (not a metadata table)
- Aggregate pushdown is enabled (`spark.sql.iceberg.aggregate-push-down.enabled`)
- There are no GROUP BY expressions (global aggregates only)
- No files have row-level deletes
- Column metrics modes support the required statistics (e.g., cannot produce MIN/MAX from `counts`-only mode, or from truncated string values)

When successful, `pushAggregation` creates a `SparkLocalScan` containing the pre-computed aggregate result as a single `InternalRow`, completely bypassing file reading.

### Limit Pushdown (SupportsPushDownLimit)

`pushLimit(int)` passes the query's LIMIT value down to Iceberg's scan planning via `Scan.minRowsRequested()`. This allows Iceberg to short-circuit file planning once enough rows are identified, reducing planning time for queries that only need a small number of rows.

## Scan Planning

### Build Phase

`SparkScanBuilder.build()` creates the appropriate scan object based on the query type:

| Scan Type | Condition | Iceberg Scan | Spark Scan |
|-----------|-----------|--------------|------------|
| Batch | Default (no incremental options) | `BatchScan` | `SparkBatchQueryScan` |
| Incremental Append | `start-snapshot-id` is set | `IncrementalAppendScan` | `SparkBatchQueryScan` |
| Changelog | Accessed via `buildChangelogScan()` | `IncrementalChangelogScan` | `SparkChangelogScan` |
| Copy-on-Write | Row-level CoW operations | `BatchScan` (with `ignoreResiduals()`) | `SparkCopyOnWriteScan` |
| Merge-on-Read | Row-level MoR operations | `BatchScan` (pinned snapshot) | `SparkBatchQueryScan` |
| Local (aggregate) | Aggregate pushdown succeeded | N/A | `SparkLocalScan` |

Each Iceberg scan is configured with:

- Filter expression (pushed-down predicates)
- Schema projection (pruned columns + metadata columns)
- Case sensitivity setting
- Split planning parameters (size, lookback, open file cost)
- Time travel options (snapshot ID, timestamp, branch, tag)

### Distributed Scan Planning

For large tables, Iceberg supports distributed scan planning via `SparkDistributedDataScan`. This is enabled when:

- The table supports `SupportsDistributedScanPlanning`
- Either data or delete planning mode is not `LOCAL`
- `spark.driver.maxResultSize` is at least 256 MB

The planning modes (`LOCAL` or `DISTRIBUTED`) are configured via:

- `spark.sql.iceberg.planning-mode.data` (session) / `read.data-planning-mode` (table property)
- `spark.sql.iceberg.planning-mode.delete` (session) / `read.delete-planning-mode` (table property)

### SparkBatchQueryScan

`SparkBatchQueryScan` extends `SparkPartitioningAwareScan` and represents the primary batch read scan. Key responsibilities:

- **Task planning** — delegates to the underlying Iceberg scan's `planTasks()` to produce `CombinedScanTask` objects
- **Statistics reporting** — implements `SupportsReportStatistics` to provide column-level statistics (null counts, min/max values, NaN counts) to Spark's cost-based optimizer
- **Partitioning-aware grouping** — when `preserveDataGrouping` is enabled, organizes tasks by partition to maintain data locality

### SparkPartitioningAwareScan

`SparkPartitioningAwareScan` extends `SparkScan` and adds partitioning awareness. When `preserveDataGrouping` is enabled (via `spark.sql.iceberg.preserve-data-grouping`), it:

- Reports the table's output partitioning to Spark
- Groups tasks by partition so that each Spark partition reads data from a single Iceberg partition

### SparkChangelogScan

`SparkChangelogScan` handles incremental changelog reads, producing rows annotated with change type (INSERT, DELETE, UPDATE_BEFORE, UPDATE_AFTER). It uses `IncrementalChangelogScan` from Iceberg core and supports timestamp-based or snapshot-ID-based range specification.

### Split Planning Configuration

Split planning controls how Iceberg combines file scan tasks into input splits for Spark partitions:

| Configuration | Read Option | Table Property | Default |
|---------------|-------------|----------------|---------|
| Split size | `split-size` | `read.split.target-size` | 128 MB |
| Split lookback | `lookback` | `read.split.planning-lookback` | 10 |
| Open file cost | `file-open-cost` | `read.split.open-file-cost` | 4 MB |

The configuration precedence is: read options > session configuration > table properties.

## Input Partitions and Reader Factories

### SparkBatch

`SparkBatch` (`o.a.i.spark.source.SparkBatch`) implements Spark's `Batch` interface. Its `planInputPartitions()` method:

1. Plans tasks from the Iceberg scan (producing `CombinedScanTask` objects)
2. Wraps each task into a `SparkInputPartition`
3. Broadcasts the serialized `Table` object to avoid repeated metadata fetches on executors

The reader factory selection depends on vectorization eligibility:

- **`SparkColumnarReaderFactory`** — used when vectorized reading is enabled and the data is compatible
- **`SparkRowReaderFactory`** — used as the fallback for row-at-a-time reading

### SparkInputPartition

`SparkInputPartition` wraps an array of `ScanTaskGroup` objects and carries the broadcast reference to the table. It implements `HasPartitionKey` when the scan preserves data grouping, providing partition keys for Spark's partitioning-aware operations.

### Vectorization Eligibility

Vectorized (columnar) reading is used when:

- The table property or session configuration enables it (`read.parquet.vectorization.enabled` for Parquet, `read.orc.vectorization.enabled` for ORC)
- The batch size is configured via `read.parquet.vectorization.batch-size` (default 5000) or `read.orc.vectorization.batch-size` (default 5000)
- Avro does not support vectorized reads

## File Reading

### Reader Hierarchy

The reader classes form an inheritance hierarchy:

```
BaseReader<T> (abstract)
  ├── BaseRowReader<T extends InternalRow>
  │     └── RowDataReader (row-at-a-time reading)
  └── BaseBatchReader
        └── BatchDataReader (vectorized/columnar reading)
```

### BaseReader

`BaseReader` (`o.a.i.spark.source.BaseReader`) is the abstract foundation implementing `PartitionReader`. It manages:

- Iterating over `FileScanTask` objects within a `CombinedScanTask`
- Opening the appropriate format-specific reader for each file
- Applying delete files (equality deletes, position deletes, deletion vectors)
- Residual filter evaluation for predicates not fully resolved during planning
- Metadata column population (_file, _pos, _spec_id, _partition, _deleted)

The core iteration loop in `next()`:

1. Get the next `FileScanTask` from the combined task
2. Open an `Iterable` of records for the task (format-specific)
3. Apply delete filtering
4. Apply residual predicate filter
5. Return records one at a time

### RowDataReader

`RowDataReader` (`o.a.i.spark.source.RowDataReader`) extends `BaseRowReader` and produces `InternalRow` objects. It delegates to format-specific readers:

- **Parquet** — uses `SparkParquetReaders` to create Parquet readers that produce Spark `InternalRow` directly
- **ORC** — uses `SparkOrcReader` for ORC files
- **Avro** — uses `SparkPlannedAvroReader` for Avro files

### BatchDataReader

`BatchDataReader` (`o.a.i.spark.source.BatchDataReader`) extends `BaseBatchReader` and produces `ColumnarBatch` objects for vectorized processing. It uses:

- **Parquet** — `Parquet.ReadBuilder` with vectorized batch reading support, configured with the Spark-specific batch size
- **ORC** — ORC vectorized batch reading

Vectorized reading processes data in columnar batches (default 5000 rows per batch), which is significantly more efficient for analytical queries.

### Delete Filtering

Iceberg supports multiple delete mechanisms, all handled transparently during reading:

| Delete Type | Mechanism |
|-------------|-----------|
| Position deletes | Delete files that identify specific (file_path, position) pairs to exclude |
| Equality deletes | Delete files containing column values; any data row matching those values is excluded |
| Deletion vectors (DVs) | Puffin-format files with bitmap-encoded position deletes (format V3+) |

The `BaseReader` applies deletes by wrapping the data iterator with delete-aware iterators that filter out deleted rows before returning them to Spark.

### Residual Predicate Evaluation

When a filter predicate cannot be fully evaluated during file-level pruning (i.e., the predicate narrows down to specific files but not all rows within those files match), Iceberg generates a residual expression. The reader evaluates this residual against each row, providing an additional filtering layer beyond what file-level statistics can achieve.

## Advanced Features

### Runtime V2 Filtering

Spark 3.4+ supports runtime V2 filtering for dynamic partition elimination. This allows Spark to use results from one side of a join to prune partitions on the other side at runtime, significantly reducing data read for star-schema queries.

### Statistics Reporting

`SparkBatchQueryScan` implements `SupportsReportStatistics` to feed column-level statistics to Spark's cost-based optimizer (CBO). When enabled (`spark.sql.iceberg.report-column-stats`, default true), it reports:

- Row count estimates
- Column-level null counts
- Column-level min/max values (when available from manifest statistics)
- Column-level NaN counts
- Size in bytes

These statistics help Spark choose optimal join strategies and parallelism.

### Time Travel

Iceberg supports multiple time travel mechanisms, all configured through read options:

| Option | Description |
|--------|-------------|
| `snapshot-id` | Read a specific snapshot by ID |
| `as-of-timestamp` | Read the latest snapshot as of a timestamp |
| `branch` | Read from a named branch |
| `tag` | Read from a named tag |

Time travel options are mutually exclusive with incremental scan options (`start-snapshot-id`, `end-snapshot-id`).

### Metadata Tables

Iceberg exposes internal metadata as queryable tables accessible via the `table.metadata_table` naming convention (e.g., `catalog.db.table.snapshots`). These are represented as `BaseMetadataTable` instances and follow the same read path, but produce metadata rows instead of data rows.

### Streaming Micro-Batch Reads

`SparkMicroBatchStream` (`o.a.i.spark.source.SparkMicroBatchStream`) implements Spark's `MicroBatchStream` and `SupportsTriggerAvailableNow` for Structured Streaming reads. It:

- Tracks offsets as Iceberg snapshot IDs
- Plans input partitions for each micro-batch between a start and end offset
- Supports `maxFilesPerMicroBatch` and `maxRecordsPerMicroBatch` rate limiting
- Can skip delete snapshots (`streaming-skip-delete-snapshots`) and overwrite snapshots (`streaming-skip-overwrite-snapshots`)
- Supports `stream-from-timestamp` for starting a stream from a specific point in time

### Executor Caching

When enabled (`spark.sql.iceberg.executor-cache.enabled`), Iceberg caches table metadata on executors to avoid redundant metadata fetches. This includes:

- Table metadata caching for locality-aware scheduling
- Delete file caching on executors (`spark.sql.iceberg.executor-cache.delete-files.enabled`)

### Parquet Reader Types

Iceberg supports multiple Parquet reader implementations, configured via `spark.sql.iceberg.parquet-reader-type`:

- Default Iceberg Parquet reader
- Native Spark Parquet reader (for compatibility)

## Configuration Reference

### Read Options (per-query)

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `snapshot-id` | Long | latest | Snapshot ID for time travel |
| `as-of-timestamp` | Long | — | Timestamp for time travel |
| `branch` | String | — | Branch name to read from |
| `tag` | String | — | Tag name to read from |
| `start-snapshot-id` | Long | — | Start snapshot for incremental scan |
| `end-snapshot-id` | Long | — | End snapshot for incremental scan |
| `start-timestamp` | Long | — | Start timestamp for changelog scan |
| `end-timestamp` | Long | — | End timestamp for changelog scan |
| `split-size` | Long | 128 MB | Target split size in bytes |
| `lookback` | Integer | 10 | Split planning lookback |
| `file-open-cost` | Long | 4 MB | Cost to open a file (used in split planning) |
| `streaming-skip-delete-snapshots` | Boolean | false | Skip delete snapshots in streaming |
| `streaming-skip-overwrite-snapshots` | Boolean | false | Skip overwrite snapshots in streaming |
| `streaming-max-files-per-micro-batch` | Integer | unlimited | Max files per streaming micro-batch |
| `streaming-max-rows-per-micro-batch` | Integer | unlimited | Max rows per streaming micro-batch |
| `stream-from-timestamp` | Long | — | Timestamp to start streaming from |

### Session Configuration

| Property | Default | Description |
|----------|---------|-------------|
| `spark.sql.iceberg.vectorization.enabled` | (table property) | Override vectorization setting |
| `spark.sql.iceberg.aggregate-push-down.enabled` | false | Enable aggregate pushdown |
| `spark.sql.iceberg.preserve-data-grouping` | false | Preserve data grouping by partition |
| `spark.sql.iceberg.report-column-stats` | true | Report column statistics to CBO |
| `spark.sql.iceberg.planning-mode.data` | (table property) | Data planning mode (LOCAL/DISTRIBUTED) |
| `spark.sql.iceberg.planning-mode.delete` | (table property) | Delete planning mode (LOCAL/DISTRIBUTED) |
| `spark.sql.iceberg.executor-cache.enabled` | false | Enable executor-side metadata caching |
| `spark.sql.iceberg.parquet-reader-type` | (default) | Parquet reader implementation |

## Version Differences (v3.5 to v4.1)

The read path has been incrementally enhanced across Spark versions:

- **Spark 3.4** — introduced V2 filter pushdown (`SupportsPushDownV2Filters`) alongside the older V1 filter interface
- **Spark 3.5** — added runtime V2 filtering support for dynamic partition elimination
- **Spark 4.0/4.1** — Scala 2.13 only, enhanced metadata column support, row lineage columns

All versions share the same fundamental architecture described in this document, with version-specific adapter classes handling API differences.

## Note on DataSource V1

Iceberg does **not** implement Spark's DataSource V1 API. The entire Spark integration uses exclusively DataSource V2. For Hadoop-based systems that require InputFormat compatibility (e.g., Hive), Iceberg provides the `mr` (MapReduce) module with `IcebergInputFormat`, which is separate from the Spark integration.

## Further Reading

- [Spark DataSource V2 Write Path](spark-datasource-v2-write-path.md) — the write-side counterpart to this document
- [Spark Integration Architecture](spark-integration-architecture.md) — module structure and build system
- [Queries](spark-queries.md) — user-facing query documentation
- [Configuration](spark-configuration.md) — catalog and session configuration
- [Structured Streaming](spark-structured-streaming.md) — streaming read/write user guide
