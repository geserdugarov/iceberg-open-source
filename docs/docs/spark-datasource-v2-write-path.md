---
title: "Spark DataSource V2 Write Path"
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

# Spark DataSource V2 Write Path

This document describes the internal architecture of Iceberg's Spark DataSource V2 write path. It traces how data flows from a Spark write operation through file writing on executors to the atomic metadata commit on the driver.

## Overview

The write path follows this high-level flow:

```
INSERT INTO / INSERT OVERWRITE / MERGE INTO / UPDATE / DELETE
  → SparkTable.newWriteBuilder() (write builder creation)
    → SparkWriteBuilder (mode selection)
      → SparkWrite (distribution/ordering requirements)
        → BatchWrite (commit coordinator)
          → WriterFactory → DataWriter (file writing on executors)
            → TaskCommit messages (file metadata)
              → SnapshotUpdate.commit() (atomic metadata commit on driver)
```

For row-level operations (DELETE, UPDATE, MERGE), an alternative path through `SparkRowLevelOperationBuilder` selects between Copy-on-Write and Merge-on-Read strategies.

## Entry Point

### SparkTable Write Capabilities

`SparkTable` (`o.a.i.spark.source.SparkTable`) declares the following write-related capabilities:

| Capability | Description |
|------------|-------------|
| `BATCH_WRITE` | Standard batch append |
| `STREAMING_WRITE` | Spark Structured Streaming writes |
| `OVERWRITE_BY_FILTER` | INSERT OVERWRITE with WHERE clause |
| `OVERWRITE_DYNAMIC` | Dynamic partition overwrite (INSERT OVERWRITE PARTITION) |
| `AUTOMATIC_SCHEMA_EVOLUTION` | Automatic column addition during writes |
| `ACCEPT_ANY_SCHEMA` | Accept any write schema (when `spark.write.accept-any-schema` is enabled) |

### newWriteBuilder

`SparkTable.newWriteBuilder(LogicalWriteInfo)` is the entry point for all write operations. It returns:

- `SparkWriteBuilder` — for regular data tables
- `SparkPositionDeletesRewriteBuilder` — for rewriting position deletes on `PositionDeletesTable`

The method validates that the table is not being accessed at a specific snapshot (writes to historical snapshots are not allowed).

### newRowLevelOperationBuilder

`SparkTable.newRowLevelOperationBuilder(RowLevelOperationInfo)` is the entry point for row-level operations (DELETE, UPDATE, MERGE). It returns a `SparkRowLevelOperationBuilder` which selects between Copy-on-Write and Merge-on-Read strategies based on table properties.

## Write Mode Selection (SparkWriteBuilder)

`SparkWriteBuilder` (`o.a.i.spark.source.SparkWriteBuilder`) implements `WriteBuilder`, `SupportsDynamicOverwrite`, and `SupportsOverwrite`. It determines which write mode to use based on how the builder is configured:

### Configuration Methods

| Method | Trigger | Mode |
|--------|---------|------|
| (default) | `INSERT INTO` | Batch Append |
| `overwriteDynamicPartitions()` | `INSERT OVERWRITE` (dynamic) | Dynamic Overwrite |
| `overwrite(Filter[])` | `INSERT OVERWRITE` with filter | Overwrite by Filter |
| `overwriteFiles(Scan, Command, IsolationLevel)` | Row-level CoW operations | Copy-on-Write |

### Schema Validation

Before building the write, `SparkWriteBuilder` validates and optionally merges the write schema:

- **Schema merging** — when `merge-schema` is enabled, new columns in the write schema are added to the table
- **Nullability checks** — validates that non-nullable columns are not receiving null values (when `check-nullability` is enabled)
- **Ordering checks** — validates column ordering constraints (when `check-ordering` is enabled)

## SparkWrite — Central Coordinator

`SparkWrite` (`o.a.i.spark.source.SparkWrite`) extends `BaseSparkWrite` and implements `Write` and `RequiresDistributionAndOrdering`. It is the central coordinator for all write operations.

### Distribution and Ordering Requirements

`SparkWrite` implements `RequiresDistributionAndOrdering` to tell Spark how to distribute and sort data before writing:

- `requiredDistribution()` — returns the distribution requirement (NONE, HASH by partition, or RANGE)
- `requiredOrdering()` — returns the sort order requirement
- `advisoryPartitionSizeInBytes()` — returns the suggested partition size for Spark's adaptive query execution

These requirements ensure that data arrives at writers in the correct order for efficient file writing. For example, hash distribution by partition key ensures each writer handles a single partition, enabling clustered writes without needing a fanout buffer.

### Distribution Modes

The distribution mode is configured via `write.distribution-mode` (table property) or `spark.sql.iceberg.distribution-mode` (session):

| Mode | Behavior |
|------|----------|
| `none` | No distribution requirement; data arrives in arbitrary order |
| `hash` | Data is hash-distributed by partition key |
| `range` | Data is range-distributed by partition key and sort order |

### BatchWrite Selection

`SparkWrite.toBatch()` returns the appropriate `BatchWrite` implementation:

| BatchWrite Class | Condition | Iceberg Operation |
|-----------------|-----------|-------------------|
| `RewriteFiles` | `rewrittenFileSetId` is set | File compaction via `FileRewriteCoordinator` |
| `OverwriteByFilter` | Overwrite filter expression present | `table.newOverwrite()` with `overwriteByRowFilter()` |
| `DynamicOverwrite` | Dynamic partition overwrite enabled | `table.newReplacePartitions()` |
| `CopyOnWriteOperation` | Copy-on-write scan provided | `table.newOverwrite()` replacing specific files |
| `BatchAppend` | Default | `table.newAppend()` |

## File Writing Infrastructure

### WriterFactory

Each `BatchWrite` creates a `WriterFactory` that runs on executors. The factory:

1. Deserializes the broadcast `Table` reference
2. Creates a `SparkFileWriterFactory` with the appropriate format configuration
3. Returns either an `UnpartitionedDataWriter` or `PartitionedDataWriter`

### Writer Types

| Writer | Use Case | Underlying Writer |
|--------|----------|-------------------|
| `UnpartitionedDataWriter` | Tables without partitioning | `RollingDataWriter` — rolls to a new file when target size is reached |
| `PartitionedDataWriter` (fanout) | Partitioned tables, unsorted data | `FanoutDataWriter` — maintains open writers for multiple partitions simultaneously |
| `PartitionedDataWriter` (clustered) | Partitioned tables, sorted data | `ClusteredDataWriter` — assumes data arrives sorted by partition, one partition at a time |

The choice between fanout and clustered writing is controlled by `write.spark.fanout.enabled` (table property). Fanout writing uses more memory but handles arbitrarily ordered input; clustered writing is more memory-efficient but requires pre-sorted input (enforced by `RequiresDistributionAndOrdering`).

### SparkFileWriterFactory

`SparkFileWriterFactory` (`o.a.i.spark.source.SparkFileWriterFactory`) extends `BaseFileWriterFactory<InternalRow>` and creates format-specific file writers:

| Format | Data Writer | Delete Writer |
|--------|------------|---------------|
| Parquet | `SparkParquetWriters` | `SparkParquetWriters` with delete schema |
| ORC | `SparkOrcWriter` | `SparkOrcWriter` with delete schema |
| Avro | `SparkAvroWriter` | `SparkAvroWriter` with delete schema |

The factory configures:

- File format and compression
- Schema mapping between Iceberg and Spark types
- Write properties (Parquet page size, dictionary encoding, etc.)
- Rolling file target sizes

### Target File Sizes

| Configuration | Table Property | Default |
|---------------|---------------|---------|
| Data file size | `write.target-file-size-bytes` | 128 MB |
| Delete file size | `write.delete.target-file-size-bytes` | 32 MB |

### TaskCommit Messages

When a writer task completes, it produces a `TaskCommit` message containing:

- Array of `DataFile` objects describing the files written
- Output metrics (bytes written, records written) reported to Spark's metrics system

These messages are collected by the driver for the commit phase.

## Commit Flow

### Batch Commit

Each `BatchWrite` implementation's `commit(WriterCommitMessage[])` method:

1. Extracts `DataFile` objects from all `TaskCommit` messages
2. Creates the appropriate `SnapshotUpdate` operation
3. Sets snapshot properties (Spark app ID, WAP ID, commit metadata)
4. Targets the correct branch (if branch-targeted writing is enabled)
5. Calls `commit()` for an atomic metadata update

### Branch-Targeted Writes

When Write-Audit-Publish (WAP) is enabled or a branch is specified:

- Writes target a specific branch rather than `main`
- The WAP branch is determined from session configuration (`spark.wap.branch`)
- A WAP ID (`spark.wap.id`) can be set for auditing purposes

### Snapshot Properties

All commits set the following properties:

- `spark.app.id` — the Spark application ID for traceability
- WAP ID (when WAP is enabled)
- Custom commit metadata (via `CommitMetadata.commitProperties()`)

### Conflict Detection

For overwrites, Iceberg validates that no conflicting changes occurred between scan and commit:

| Isolation Level | Validation |
|-----------------|-----------|
| `SERIALIZABLE` | Validates no conflicting data files or delete files were added |
| `SNAPSHOT` | Validates no conflicting delete files were added |

The validation uses the filter expression from the scan to scope conflict checks to the affected partitions only.

## Row-Level Operations

### SparkRowLevelOperationBuilder

`SparkRowLevelOperationBuilder` (`o.a.i.spark.source.SparkRowLevelOperationBuilder`) selects the row-level operation strategy based on table properties:

| Command | Table Property | Default |
|---------|---------------|---------|
| DELETE | `write.delete.mode` | `copy-on-write` |
| UPDATE | `write.update.mode` | `copy-on-write` |
| MERGE | `write.merge.mode` | `copy-on-write` |

Each command also has a configurable isolation level:

| Command | Table Property | Default |
|---------|---------------|---------|
| DELETE | `write.delete.isolation-level` | `serializable` |
| UPDATE | `write.update.isolation-level` | `serializable` |
| MERGE | `write.merge.isolation-level` | `serializable` |

### Copy-on-Write (SparkCopyOnWriteOperation)

`SparkCopyOnWriteOperation` (`o.a.i.spark.source.SparkCopyOnWriteOperation`) implements `RowLevelOperation` for the Copy-on-Write strategy:

1. **Scan phase** — `SparkCopyOnWriteScan` reads existing files matching the operation's filter, with `ignoreResiduals()` to get complete file matches
2. **Rewrite phase** — the files are rewritten with modifications applied
3. **Commit phase** — `CopyOnWriteOperation` BatchWrite uses `table.newOverwrite()` to atomically replace old files with new files

Required metadata attributes: `_file` (always), `_pos` (for DELETE/UPDATE), plus `_row_id` and `_last_updated_sequence_number` when row lineage is supported.

### Merge-on-Read (SparkPositionDeltaWrite)

`SparkPositionDeltaWrite` (`o.a.i.spark.source.SparkPositionDeltaWrite`) implements `DeltaWrite` for the Merge-on-Read strategy. Instead of rewriting entire files, it writes:

- **Position delete files** — identifying (file_path, position) pairs of deleted rows
- **Deletion vectors (DVs)** — bitmap-encoded position deletes in Puffin format (format V3+)
- **New data files** — for inserted or updated rows

#### Delta Writer Types

| Writer | Command | Behavior |
|--------|---------|----------|
| `DeleteOnlyDeltaWriter` | DELETE | Writes position deletes only |
| `UnpartitionedDeltaWriter` | UPDATE, MERGE (unpartitioned) | Writes position deletes + new data files |
| `PartitionedDeltaWriter` | UPDATE, MERGE (partitioned) | Writes position deletes + new data files with partition keys |

All delta writers extract metadata from Spark's row ID columns (`_file`, `_pos`) and metadata columns (`_spec_id`, `_partition`) to identify which rows to delete.

Updates are represented as DELETE + INSERT pairs (`representUpdateAsDeleteAndInsert()` returns true), not as in-place updates.

#### DeltaTaskCommit

Delta writers produce `DeltaTaskCommit` messages containing:

- `DataFile[]` — newly written data files
- `DeleteFile[]` — newly written position delete files or DVs
- `DeleteFile[]` — rewritten (consolidated) delete files
- `CharSequence[]` — referenced data files (for validation)

#### Commit and Validation

`PositionDeltaBatchWrite.commit()` uses `RowDelta` to atomically:

1. Add new data files
2. Add new delete files
3. Validate that referenced data files still exist
4. Validate no conflicting deletes were added (for UPDATE/MERGE)
5. For SERIALIZABLE isolation: validate no conflicting data was added

#### Deletion Vectors (Format V3+)

For tables at format version 3 or higher, the delete file format can be set to `PUFFIN` to use deletion vectors (DVs). DVs are more efficient than traditional position delete files because they use bitmap encoding and can be applied without sorting.

The `Context` inner class tracks DV configuration:

- `useDVs()` — true when delete file format is PUFFIN
- `deleteGranularity()` — FILE or PARTITION level granularity

## Streaming Writes

### StreamingWrite Classes

`SparkWrite` provides two streaming write implementations:

| Class | Trigger | Iceberg Operation |
|-------|---------|-------------------|
| `StreamingAppend` | Default streaming writes | `table.newFastAppend()` |
| `StreamingOverwrite` | Complete output mode | `table.newOverwrite()` with `alwaysTrue()` filter |

### Epoch-Based Exactly-Once Semantics

Streaming writes use epoch-based idempotent commits:

1. Each micro-batch is identified by a `(queryId, epochId)` pair
2. Before committing, the writer checks if the epoch was already committed (via snapshot properties)
3. If already committed, the commit is skipped (idempotent)
4. Snapshot properties store `queryId` and `epochId` for deduplication

This ensures exactly-once semantics even if a micro-batch is retried after a transient failure.

### Rate Limiting

Streaming reads (which feed streaming writes) support rate limiting via:

- `streaming-max-files-per-micro-batch` — limits files per batch
- `streaming-max-rows-per-micro-batch` — limits rows per batch

## Configuration Reference (SparkWriteConf)

`SparkWriteConf` (`o.a.i.spark.SparkWriteConf`) centralizes all write configuration. Configuration precedence (highest to lowest): write options > session configuration > table properties.

### File Format and Size

| Configuration | Table Property | Default |
|---------------|---------------|---------|
| Data file format | `write.format.default` | `parquet` |
| Delete file format | `write.delete.format` | (data format) |
| Data file target size | `write.target-file-size-bytes` | 128 MB |
| Delete file target size | `write.delete.target-file-size-bytes` | 32 MB |

### Distribution and Ordering

| Configuration | Table Property | Session Property | Default |
|---------------|---------------|-----------------|---------|
| Distribution mode | `write.distribution-mode` | `spark.sql.iceberg.distribution-mode` | `hash` |
| Fanout writer | `write.spark.fanout.enabled` | — | `false` |

### Row-Level Operation Modes

| Configuration | Table Property | Default |
|---------------|---------------|---------|
| Delete mode | `write.delete.mode` | `copy-on-write` |
| Update mode | `write.update.mode` | `copy-on-write` |
| Merge mode | `write.merge.mode` | `copy-on-write` |
| Delete isolation level | `write.delete.isolation-level` | `serializable` |
| Update isolation level | `write.update.isolation-level` | `serializable` |
| Merge isolation level | `write.merge.isolation-level` | `serializable` |

### Write-Audit-Publish

| Configuration | Table Property | Session Property |
|---------------|---------------|-----------------|
| WAP enabled | `write.wap.enabled` | — |
| WAP ID | — | `spark.wap.id` |
| WAP branch | — | `spark.wap.branch` |

### Schema Evolution

| Configuration | Write Option | Default |
|---------------|-------------|---------|
| Merge schema | `merge-schema` | `false` |
| Check nullability | `check-nullability` | `true` |
| Check ordering | `check-ordering` | `true` |
| Accept any schema | `spark.write.accept-any-schema` (table property) | `false` |

### Additional Configuration

| Configuration | Write Option / Property | Default |
|---------------|------------------------|---------|
| Output spec ID | `output-spec-id` | (current spec) |
| Branch | `branch` | `main` |
| Rewritten file set ID | `rewritten-file-set-id` | — |
| Delete granularity | `write.delete.granularity` | `partition` |

## Error Handling

### Task-Level Abort

When a writer task fails, `DataWriter.abort()` is called:

1. The writer's `close()` method is invoked to flush any buffered data
2. All files written by the task are deleted via the table's `FileIO`

### Driver-Level Abort

When the overall write operation fails, `BatchWrite.abort(WriterCommitMessage[])` is called:

1. All `TaskCommit` messages are collected
2. Each referenced file is deleted via the table's `FileIO`
3. The `CleanableFailure` interface ensures cleanup even on unexpected failures

This two-level abort strategy ensures that partial writes do not leave orphan files in the table's storage.

## Note on DataSource V1

Iceberg does **not** implement Spark's DataSource V1 write API. The entire Spark integration uses exclusively DataSource V2 for both reads and writes. For Hadoop-based systems requiring OutputFormat compatibility, the `mr` module provides separate MapReduce-level integration.

## Further Reading

- [Spark DataSource V2 Read Path](spark-datasource-v2-read-path.md) — the read-side counterpart to this document
- [Spark Integration Architecture](spark-integration-architecture.md) — module structure and build system
- [Writes](spark-writes.md) — user-facing write documentation
- [Configuration](spark-configuration.md) — catalog and session configuration
- [Structured Streaming](spark-structured-streaming.md) — streaming read/write user guide
