# Iceberg Delete Mechanisms — Position Deletes, Equality Deletes, and Deletion Vectors

This document provides a comprehensive description of how Apache Iceberg handles row-level deletes across format versions V2 and V3, covering Position Deletes, Equality Deletes, and Deletion Vectors (DVs).

**Related docs:** [architecture_overview.md](architecture_overview.md) | [read_path_callstack.md](read_path_callstack.md) | [write_path_callstack.md](write_path_callstack.md) | [compaction_callstack.md](compaction_callstack.md)

---

## 1. Overview — The Iceberg Delete Model

Iceberg data files are **immutable** — once written, they are never modified in place. To support SQL operations that logically remove or update rows (DELETE, UPDATE, MERGE INTO), Iceberg uses two strategies:

- **Copy-on-Write (CoW):** Rewrite entire data files with the affected rows removed. No delete files are produced — the old data file is replaced by a new one in the next snapshot.
- **Merge-on-Read (MoR):** Write lightweight **delete files** that record which rows are logically deleted. The deletes are applied at read time by filtering out deleted rows.

The MoR approach is faster for writes (avoids full file rewrites) but adds read overhead. Iceberg supports three types of delete files:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ICEBERG DELETE FILE TYPES                               │
│                                                                            │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌────────────────────┐  │
│  │  Position Deletes   │  │  Equality Deletes   │  │ Deletion Vectors   │  │
│  │  (row-based, V2     │  │  (V2+)              │  │ (V3+)              │  │
│  │   only — V3+ uses   │  │                     │  │                    │  │
│  │   DVs instead)      │  │                     │  │                    │  │
│  │                     │  │                     │  │                    │  │
│  │  Parquet/Avro/ORC   │  │  Parquet/Avro/ORC   │  │  Puffin file with  │  │
│  │  file with          │  │  file with rows in  │  │  RoaringBitmap     │  │
│  │  file_path + pos    │  │  equalityDeleteRow- │  │  per data file     │  │
│  │  columns            │  │  Schema             │  │                    │  │
│  │                     │  │                     │  │                    │  │
│  │  Scope: per file    │  │  Scope: partition   │  │  Scope: per file   │  │
│  │  Apply: O(1) bitmap │  │  Apply: hash lookup │  │  Apply: O(1) bitmap│  │
│  └─────────────────────┘  └─────────────────────┘  └────────────────────┘  │
│                                                                            │
│  Format Version:  V1 = no deletes (append-only)                            │
│                   V2 = row-based position deletes + equality deletes       │
│                   V3+ = adds deletion vectors (Puffin); position deletes   │
│                         MUST be DVs (MergingSnapshotProducer rejects       │
│                         row-based position deletes in V3/V4)               │
└────────────────────────────────────────────────────────────────────────────┘
```

### Comparison Table

| Property                    | Position Deletes                      | Equality Deletes                                     | Deletion Vectors                                            |
|-----------------------------|---------------------------------------|------------------------------------------------------|-------------------------------------------------------------|
| **Format version**          | V2 only (row-based position deletes are rejected in V3/V4 by `MergingSnapshotProducer.validateDeleteFileForVersion`; use DVs instead) | V2+ | V3+ (Puffin DVs are the V3/V4 carrier for positional deletes) |
| **File format**             | Parquet / Avro / ORC (configurable via `write.delete.format.default`, falls back to data file format) | Parquet / Avro / ORC (same configuration) | Puffin (binary blob, V3+) |
| **Content type**            | `POSITION_DELETES`                    | `EQUALITY_DELETES`                                   | `POSITION_DELETES`                                          |
| **Scope**                   | Single data file (by `file_path`)     | All files in partition matching equality fields      | Single data file (by `referencedDataFile`)                  |
| **Columns stored**          | `file_path` (string) + `pos` (long)   | Configured `equalityDeleteRowSchema` (must contain equality fields; may be a projection of the table schema, may carry extra cols for metrics/sort) | Serialized RoaringBitmap of positions |
| **Read-time cost**          | Low — bitmap lookup O(1) per row      | High — hash set lookup per row, loaded for all files | Lowest — compact bitmap, direct offset access               |
| **Write-time cost**         | Low — record (path, pos) pairs        | Low — record matching rows                           | Low — set bits in bitmap                                    |
| **Storage overhead**        | Moderate — one record per deleted row | Moderate–high — one schema-shaped record per deleted row | Low — compressed bitmap                                  |
| **Metadata fields**         | `recordCount`, `fileSizeInBytes`      | `equalityFieldIds[]`, metrics                        | `referencedDataFile`, `contentOffset`, `contentSizeInBytes` |
| **Multiple per data file?** | Yes (across multiple delete files)    | N/A (partition-scoped)                               | No (at most one DV per data file)                           |
| **Practical usage**         | Primary V2 mechanism                  | Rarely used in practice                              | Preferred V3 mechanism                                      |

---

## 2. Copy-on-Write vs Merge-on-Read

### Copy-on-Write (CoW) — Default

```
BEFORE:                                 AFTER:
┌────────────────────┐                  ┌────────────────────┐
│  data-file-001     │                  │  data-file-001     │  (deleted from snapshot)
│  row A  ◄── DELETE │   ══════════►    │                    │
│  row B             │                  └────────────────────┘
│  row C             │                  ┌────────────────────┐
└────────────────────┘                  │  data-file-002     │  (new file, rewritten)
                                        │  row B             │
                                        │  row C             │
                                        └────────────────────┘

Snapshot: OverwriteFiles (deleted files + replacement files committed
together via table.newOverwrite() in SparkWrite.CopyOnWriteOperation;
the standalone RewriteFiles API is used by maintenance, not row-level
CoW). No delete files produced.
```

- **Write cost:** High — must read + rewrite entire data file(s) containing affected rows
- **Read cost:** Zero — no delete files to apply at read time
- **Best for:** Read-heavy workloads, infrequent updates

### Merge-on-Read (MoR)

```
BEFORE:                                 AFTER:
┌────────────────────┐                  ┌────────────────────┐
│  data-file-001     │                  │  data-file-001     │  (unchanged)
│  row A  ◄── DELETE │   ══════════►    │  row A             │
│  row B             │                  │  row B             │
│  row C             │                  │  row C             │
└────────────────────┘                  └────────────────────┘
                                        ┌────────────────────┐
                                        │  delete-file-001   │  (new delete file)
                                        │  (data-file-001,   │
                                        │   pos=0)  → row A  │
                                        └────────────────────┘

Snapshot: RowDelta (add delete file referencing data-file-001)
Data file untouched. Delete applied at read time.
```

- **Write cost:** Low — write only a small delete file
- **Read cost:** Moderate — must load and apply delete file during scan
- **Best for:** Write-heavy workloads, frequent updates, near-real-time ingestion

### Configuration

| Property                       | Default         | Values                           | Scope                     |
|--------------------------------|-----------------|----------------------------------|---------------------------|
| `write.delete.mode`            | `copy-on-write` | `copy-on-write`, `merge-on-read` | DELETE statements         |
| `write.update.mode`            | `copy-on-write` | `copy-on-write`, `merge-on-read` | UPDATE statements         |
| `write.merge.mode`             | `copy-on-write` | `copy-on-write`, `merge-on-read` | MERGE INTO statements     |
| `write.delete.isolation-level` | `serializable`  | `serializable`, `snapshot`       | DELETE conflict detection |
| `write.update.isolation-level` | `serializable`  | `serializable`, `snapshot`       | UPDATE conflict detection |
| `write.merge.isolation-level`  | `serializable`  | `serializable`, `snapshot`       | MERGE conflict detection  |

These are **table properties** set via `ALTER TABLE ... SET TBLPROPERTIES(...)`.

### Mode Selection Flow (Spark)

```
SQL: DELETE FROM / UPDATE / MERGE INTO
         │
         ▼
SparkTable.newRowLevelOperationBuilder()                     [spark/source/SparkTable]
         │
         ▼
SparkRowLevelOperationBuilder                                [spark/source]
  │
  ├─► Read table property for command type:
  │     DELETE → write.delete.mode
  │     UPDATE → write.update.mode
  │     MERGE  → write.merge.mode
  │
  ├─► if mode == COPY_ON_WRITE:
  │     └─► return SparkCopyOnWriteOperation
  │           └─► Rewrites affected data files entirely
  │               (no delete files produced)
  │
  └─► if mode == MERGE_ON_READ:
        └─► return SparkPositionDeltaOperation
              └─► Writes position delete files or DVs
                  (data files left untouched)
```

---

## 3. Position Deletes (V2)

Position deletes identify rows to remove by their **physical location**: the data file path and the row's ordinal position (0-based) within that file.

### File Format

```
Position Delete File (row-based):
┌───────────────────────────────────────────────┐
│  Format: Parquet / Avro / ORC                 │  ◄── from write.delete.format.default
│                                               │      (falls back to data file format)
│  Schema:                                      │
│    file_path: string (required)               │  ◄── MetadataColumns.DELETE_FILE_PATH
│    pos:       long   (required)               │  ◄── MetadataColumns.DELETE_FILE_POS
│                                               │
│  Row 0: ("s3://bucket/data-001.parquet", 42)  │
│  Row 1: ("s3://bucket/data-001.parquet", 108) │
│  Row 2: ("s3://bucket/data-003.parquet", 7)   │
│  ...                                          │
│                                               │
│  Sorted by: (file_path ASC, pos ASC)          │  ◄── required for efficient merge
└───────────────────────────────────────────────┘
```

Note: the optional `row` column (`DELETE_FILE_ROW_FIELD_*`) and the
`PositionDelete.set(path, pos, row)` / `PositionDelete.row()` overloads
are deprecated as of Iceberg 1.11.0 and will be removed in 1.12.0.
The schema and data structure still accept the field for reading
existing files and for code paths that have not migrated (e.g. legacy
position-delete table reads and rewrites). New MoR row-level writes
must not populate row data; CDC/changelog use cases are served by
deletion vectors and changelog scans.

### Write Path

```
SQL: DELETE FROM table WHERE id = 42  (mode = merge-on-read)
        │
        ▼
SparkPositionDeltaOperation                                  [spark/source]
  │
  └─► SparkPositionDeltaWrite                                [spark/source/SparkPositionDeltaWrite]
        │
        └─► PositionDeltaBatchWrite.createBatchWriterFactory()
              │
              └─► Broadcast WriterFactory to executors

EXECUTOR:
  │
  ├─► Scan phase identifies affected rows:
  │     file_path = "data-001.parquet", pos = 42
  │     file_path = "data-001.parquet", pos = 108
  │
  ├─► BaseDeltaWriter variants:
  │     ├─► DeleteOnlyDeltaWriter      (DELETE only — no new data rows)
  │     ├─► UnpartitionedDeltaWriter   (UPDATE/MERGE unpartitioned)
  │     └─► PartitionedDeltaWriter     (UPDATE/MERGE partitioned)
  │
  └─► Delete writer selection (V2 tables):
        │  Granularity comes from SparkWriteConf.deleteGranularity():
        │    Spark default = DeleteGranularity.FILE
        │    (TableProperties.DELETE_GRANULARITY_DEFAULT = PARTITION
        │     applies only when consumers read the property directly;
        │     SparkWriteConf overrides with .defaultValue(FILE))
        │
        ├─► if input ordered by (file, pos):
        │     ClusteredPositionDeleteWriter                  [core/io]
        │       │
        │       ├─► granularity = FILE (Spark default):
        │       │     wraps FileScopedPositionDeleteWriter   [core/deletes]
        │       │       └─► delegates to RollingPositionDeleteWriter
        │       │           per referenced data file — typically one
        │       │           delete file per data file, but the rolling
        │       │           writer may roll over to additional files
        │       │           when target-file-size is exceeded
        │       │
        │       ├─► granularity = PARTITION:
        │       │     wraps RollingPositionDeleteWriter      [core/io]
        │       │       └─► PositionDeleteWriter             [core/deletes]
        │       │       └─► may also roll over to multiple delete files
        │       │           per (spec, partition) when target-file-size
        │       │           is exceeded
        │       │
        │       └─► tracks referencedDataFiles (CharSequenceSet)
        │
        └─► if input unordered:
              FanoutPositionOnlyDeleteWriter                 [core/io]
                └─► SortingPositionOnlyDeleteWriter          [core/deletes]
                      └─► sorts by (path, pos), then writes via
                          RollingPositionDeleteWriter
                      └─► optional loadPreviousDeletes hook for
                          merging with prior file-scoped deletes

PositionDeleteWriter                                         [core/deletes/PositionDeleteWriter]
  │
  ├─► for each PositionDelete<R>:
  │     appender.add(positionDelete)                         write (path, pos) to the
  │                                                          configured delete file
  │                                                          format (Parquet, Avro,
  │                                                          or ORC)
  │     referencedDataFiles.add(positionDelete.path())       track which data files
  │
  └─► close():
        └─► DeleteFile result:
              ├─ content = POSITION_DELETES
              ├─ path = delete file location
              ├─ format = configured delete FileFormat
              │           (Parquet / Avro / ORC)
              ├─ recordCount = number of deleted positions
              ├─ fileSizeInBytes
              └─ partition

RollingPositionDeleteWriter                                  [core/io]
  └─► Splits into multiple files when target file size exceeded
```

### Data Structure

```java
// core/src/main/java/org/apache/iceberg/deletes/PositionDelete.java
public class PositionDelete<R> implements StructLike {
    private CharSequence path;     // Data file path containing the deleted row
    private long pos;              // 0-based ordinal position within the file
    private R row;                 // Deprecated since 1.11.0, removed in 1.12.0
}
```

The canonical setter is `set(CharSequence path, long pos)`. The
`set(path, pos, row)` and `row()` accessors are deprecated; new code
must not rely on row data being carried in position delete records.

### Read Path — How Position Deletes Are Applied

```
Planning (Driver):
  ManifestGroup.planFiles()
    └─► DeleteFileIndex.forDataFile(seqNum, dataFile)
          │
          ├─► Find position deletes / DVs scoped to this data file's path
          ├─► Sequence number filtering (per kind):
          │     • Position deletes / DVs: delete.seqNum >= data.seqNum
          │     • Equality deletes:       delete.seqNum >  data.seqNum
          │       (via applySequenceNumber = dataSequenceNumber - 1)
          └─► Return DeleteFile[] associated with this data file

Execution (Per-Task on Executors):
  DeleteFilter.filter(records)                               [data/DeleteFilter]
    │
    ├─► 1. Load position deletes:
    │     deleteLoader.loadPositionDeletes(posDeleteFiles, filePath)
    │       │
    │       ├─► Read position delete files (Parquet/Avro/ORC,
    │       │   per write.delete.format.default)
    │       ├─► Filter records where file_path == current data file
    │       └─► Build BitmapPositionDeleteIndex:
    │             RoaringPositionBitmap.set(pos)              for each deleted position
    │
    ├─► 2. Apply position delete filter:
    │     applyPosDeletes(records)
    │       │
    │       ├─► for each record:
    │       │     long pos = record.getAs(ROW_POSITION)
    │       │     if positionIndex.isDeleted(pos):            O(1) bitmap lookup
    │       │       skip record (deleted)
    │       │     else:
    │       │       emit record (not deleted)
    │       │
    │       └─► return filtered iterator
    │
    └─► 3. Apply equality deletes (if any — see section 4)
```

### Bitmap Index Implementation

```
BitmapPositionDeleteIndex                                    [core/deletes]
  │
  └─► Uses RoaringPositionBitmap:
        │
        ├─► 64-bit position split:
        │     upper 32 bits → key (selects which RoaringBitmap)
        │     lower 32 bits → position within that bitmap
        │
        ├─► Array of RoaringBitmap (one per key)
        │     supports billions of positions efficiently
        │
        ├─► Operations:
        │     set(long pos)              mark position as deleted
        │     setRange(start, end)       mark range as deleted
        │     contains(long pos)         check if position is deleted → O(1)
        │
        └─► Serialization:
              4-byte length + 4-byte magic + Roaring portable format + 4-byte CRC-32
```

---

## 4. Equality Deletes (V2)

Equality deletes identify rows to remove by **matching column values**. Instead of specifying a physical position, they contain the values of "equality columns" — any data row matching those column values is logically deleted.

### File Format

```
Equality Delete File (row-based):
┌─────────────────────────────────────────┐
│  Format: Parquet / Avro / ORC           │  ◄── from write.delete.format.default
│                                         │      (falls back to data file format)
│  Schema:                                │
│    equalityDeleteRowSchema supplied to  │
│    the writer factory. MUST include the │
│    equality fields; MAY include extra   │
│    columns (e.g. for metrics/sorting).  │
│    Often a projection of the table      │
│    schema, not necessarily the full     │
│    schema.                              │
│                                         │
│  Metadata:                              │
│    equalityFieldIds = [3, 7]            │  ◄── field IDs used for matching
│                                         │
│  Row 0: {id=42, name="Alice"}           │  ◄── delete all rows where
│  Row 1: {id=99, name="Bob"}             │      id=42 AND name="Alice", etc.
│  ...                                    │
│                                         │
│  Non-equality columns present in the    │
│  schema are written but ignored by the  │
│  equality match (still useful for       │
│  metrics, sorting, and bounds).         │
└─────────────────────────────────────────┘
```

### Key Characteristics

- **Partition-scoped:** An equality delete file in partition P applies to ALL data files in partition P whose data sequence number is less than the delete's sequence number
- **No file_path column:** Unlike position deletes, equality deletes do not reference a specific data file
- **Broad impact:** A single equality delete file can affect many data files — every file in the partition must be checked
- **Stored columns:** The delete file contains values for every column in the configured `equalityDeleteRowSchema`. That schema must at least cover the equality fields, but it is typically a projection of the table schema rather than every table column

### Write Path

```
EqualityDeleteWriter                                         [core/deletes/EqualityDeleteWriter]
  │
  ├─► Constructor takes:
  │     int[] equalityFieldIds         which columns determine equality
  │     FileAppender<T> appender       writes rows in the configured
  │                                     delete file format (Parquet,
  │                                     Avro, or ORC)
  │     Schema eqDeleteRowSchema       schema of the rows being written
  │
  ├─► write(T row):
  │     appender.add(row)              writes one record per call
  │                                    in eqDeleteRowSchema (may be a
  │                                    projection of the table schema)
  │
  └─► close():
        └─► DeleteFile result:
              ├─ content = EQUALITY_DELETES
              ├─ equalityFieldIds = [3, 7, ...]
              ├─ format = configured delete FileFormat
              │           (Parquet / Avro / ORC)
              ├─ recordCount = number of delete rows
              ├─ metrics (column stats, bounds)
              └─ sortOrderId (if sorted)
```

### Read Path — How Equality Deletes Are Applied

```
Planning (Driver):
  DeleteFileIndex.forDataFile(seqNum, dataFile)
    │
    ├─► Find equality deletes in same partition
    ├─► Filter by sequence number
    └─► Group by equalityFieldIds

Execution (Per-Task on Executors):
  DeleteFilter.applyEqDeletes(records)                       [data/DeleteFilter]
    │
    ├─► For each group of equality deletes (same equalityFieldIds):
    │     │
    │     ├─► Project delete schema to equality columns only
    │     │
    │     ├─► Load all delete rows into StructLikeSet:
    │     │     deleteSet = deleteLoader.loadEqualityDeletes(deleteFiles, deleteSchema)
    │     │       └─► Read all row-based delete files
    │     │           (Parquet/Avro/ORC)
    │     │           Project to equality columns
    │     │           Add each row to hash set
    │     │
    │     └─► Create predicate:
    │           isDeleted = record →
    │             deleteSet.contains(
    │               projectRow.wrap(asStructLike(record)))     hash lookup
    │
    ├─► Combine predicates with OR:
    │     isEqDeleted = pred1.or(pred2).or(...)
    │
    └─► Filter records:
          for each record:
            if isEqDeleted(record): skip
            else: emit
```

### Schema Expansion

Equality deletes require the equality field columns to be present in the read projection, even if the user's query didn't select them:

```
User query:   SELECT name FROM table WHERE age > 30
Table has equality delete with equalityFieldIds = [1]  (field 1 = "id")

Actual read projection:
  ┌──────┬──────┐
  │ name │  id  │  ◄── "id" added for equality delete evaluation
  └──────┴──────┘       (stripped from output after filtering)
```

### Performance Implications

- **Memory:** All equality delete rows loaded into memory as `StructLikeSet`
- **CPU:** Hash set membership check per data row, per equality delete group
- **I/O:** Must read all equality delete files for the partition
- **Blast radius:** Every data file in the partition must be scanned against the delete set
- **Practical usage:** Rarely used in production. Position deletes (or DVs) are strongly preferred because they are file-scoped and cheaper to apply

---

## 5. Deletion Vectors (V3)

Deletion Vectors (DVs) are the V3 evolution of position deletes. They use a compact **RoaringBitmap** serialized into a **Puffin file** instead of row-by-row records in a row-based delete file (Parquet/Avro/ORC). In V3 and V4 they are also the *only* allowed encoding for positional deletes — row-based position delete files are rejected at commit time.

### File Format

```
Puffin File (deletion-vectors.puffin):
┌──────────────────────────────────────────────────────────────────┐
│  Magic: 0x50 0x46 0x41 0x31  ("PFA1")                            │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ Blob 0:  DV for data-file-001.parquet                   │     │
│  │                                                         │     │
│  │  Type: "deletion-vector-v1" (DV_V1)                     │     │
│  │  Metadata:                                              │     │
│  │    referenced-data-file: "data-file-001.parquet"        │     │
│  │    cardinality: 3                                       │     │
│  │  Payload: serialized RoaringBitmap                      │     │
│  │    [positions: 42, 108, 255]                            │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ Blob 1:  DV for data-file-003.parquet                   │     │
│  │                                                         │     │
│  │  Type: "deletion-vector-v1" (DV_V1)                     │     │
│  │  Metadata:                                              │     │
│  │    referenced-data-file: "data-file-003.parquet"        │     │
│  │    cardinality: 1                                       │     │
│  │  Payload: serialized RoaringBitmap                      │     │
│  │    [positions: 7]                                       │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  Footer:                                                         │
│    Blob metadata (JSON): types, offsets, lengths                 │
│    Footer length (4 bytes)                                       │
│    Flags (4 bytes)                                               │
│    Magic: 0x50 0x46 0x41 0x31                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Key Differences from Position Deletes

| Aspect         | Position Deletes (V2)                           | Deletion Vectors (V3)                                       |
|----------------|-------------------------------------------------|-------------------------------------------------------------|
| File format    | Row-based (Parquet/Avro/ORC, configurable)      | Puffin (binary blob)                                        |
| Storage        | One record per deleted row                      | Compressed bitmap — orders of magnitude smaller             |
| Access pattern | Sequential scan of delete file                  | Direct offset/length access to blob                         |
| Per data file  | Multiple delete files possible                  | At most ONE DV per data file                                |
| Metadata       | Just `recordCount`                              | `referencedDataFile`, `contentOffset`, `contentSizeInBytes` |
| Mergeability   | Must read all delete files and union            | Can load bitmap, set new bits, rewrite                      |
| Loading cost   | Read delete file (Parquet/Avro/ORC), filter by file_path, build bitmap | Read blob at offset, deserialize bitmap directly  |

### Write Path

```
SQL: DELETE FROM table WHERE id = 42  (mode = merge-on-read, V3 table)
        │
        ▼
SparkPositionDeltaWrite.newDeleteWriter()                    [spark/source]
  │
  ├─► context.useDVs() == true   (V3 table)
  │
  └─► new PartitioningDVWriter(fileFactory, loadPreviousDeletes)

PartitioningDVWriter                                         [core/io]
  │
  └─► delegates to BaseDVFileWriter                          [core/deletes/BaseDVFileWriter]

BaseDVFileWriter                                             [core/deletes/BaseDVFileWriter]
  │
  ├─► Constructor:
  │     BaseDVFileWriter(
  │       OutputFileFactory fileFactory,
  │       Function<String, PositionDeleteIndex> loadPreviousDeletes)
  │
  ├─► ACCUMULATION PHASE:
  │     │
  │     ├─► delete(String path, long pos, PartitionSpec spec, StructLike partition)
  │     │     │
  │     │     └─► Map<String, Deletes> deletesByPath:
  │     │           key = data file path
  │     │           value = Deletes {
  │     │             path: data file path,
  │     │             positions: BitmapPositionDeleteIndex,  ◄── RoaringBitmap
  │     │             spec: partition spec,
  │     │             partition: partition values
  │     │           }
  │     │
  │     └─► positions.delete(pos)                            set bit in bitmap
  │
  ├─► MERGE WITH PREVIOUS (on close):
  │     │
  │     ├─► For each accumulated path:
  │     │     previous = loadPreviousDeletes.apply(path)     Function call
  │     │     if previous != null:
  │     │       positions.merge(previous)                    union of bitmaps
  │     │
  │     └─► For each previous DeleteFile reachable via the index:
  │           if ContentFileUtil.isFileScoped(previousDeleteFile):
  │             rewrittenDeleteFiles.add(previousDeleteFile)  ◄── must be replaced
  │                                                              in the new commit
  │
  └─► CLOSE (write Puffin file):
        │
        ├─► PuffinWriter puffinWriter = Puffin.write(outputFile)
        │
        ├─► For each data file in dvs:
        │     │
        │     ├─► Serialize RoaringPositionBitmap to bytes:
        │     │     bitmap.runLengthOptimize()                compact before serialization
        │     │     bytes = bitmap.serialize()
        │     │
        │     ├─► Write blob to Puffin:
        │     │     puffinWriter.add(
        │     │       type = StandardBlobTypes.DV_V1,
        │     │       fieldIds = [ROW_POSITION],
        │     │       metadata = {
        │     │         "referenced-data-file": path,
        │     │         "cardinality": bitmap.cardinality()
        │     │       },
        │     │       payload = bytes)
        │     │
        │     └─► Record blob position:
        │           contentOffset = blob start in file
        │           contentSizeInBytes = blob length
        │
        ├─► puffinWriter.close()
        │
        └─► Create one DeleteFile per data file:
              FileMetadata.deleteFileBuilder(spec)
                .ofPositionDeletes()                         ◄── still POSITION_DELETES content type
                .withFormat(FileFormat.PUFFIN)
                .withPath(puffinFilePath)
                .withFileSizeInBytes(puffinFileSize)
                .withReferencedDataFile(dataFilePath)        ◄── single data file reference
                .withContentOffset(blobMetadata.offset())    ◄── direct access to blob
                .withContentSizeInBytes(blobMetadata.length())
                .withRecordCount(positions.cardinality())
                .build()

DeleteWriteResult                                            [core/io]
  ├─ deleteFiles            new DV DeleteFiles
  ├─ referencedDataFiles    CharSequenceSet of data file paths
  └─ rewrittenDeleteFiles   prior file-scoped/DV deletes that the new
                            commit will remove (see RowDelta.removeDeletes)
```

### Read Path

```
Planning (Driver):
  DeleteFileIndex.forDataFile(seqNum, dataFile)
    │
    ├─► Look for DV referencing this data file:
    │     dv = pathToDV.get(dataFile.path())
    │
    └─► DV takes priority:
          if dv != null && no other deletes:
            return [dv]
          if dv != null && has equality deletes:
            return concat(equalityDeletes, [dv])

Execution (Executors):
  BaseDeleteLoader.loadPositionDeletes(deleteFiles, filePath)
    │
    ├─► if ContentFileUtil.containsSingleDV(deleteFiles):
    │     │
    │     ├─► validateDV(dv, filePath)
    │     │     checks contentOffset, contentSizeInBytes,
    │     │     and that filePath == dv.referencedDataFile()
    │     │
    │     ├─► readDV(dv):
    │     │     IOUtil.readFully(inputFile,
    │     │                      dv.contentOffset(),
    │     │                      buf,
    │     │                      0,
    │     │                      dv.contentSizeInBytes().intValue())
    │     │
    │     └─► PositionDeleteIndex.deserialize(bytes, dv)
    │           → BitmapPositionDeleteIndex (RoaringPositionBitmap)
    │
    └─► else (legacy V2 position delete files):
          getOrReadPosDeletes(deleteFiles, filePath)
            └─► caches per-file PositionDeleteIndex when beneficial

    Same interface as position deletes:
      positionIndex.isDeleted(pos) → true/false              O(1) per row
```

### Write Decision Logic

```
SparkPositionDeltaWrite.newDeleteWriter()                    [spark/source]
  │
  ├─► if context.useDVs():                                   V3+ table
  │     └─► PartitioningDVWriter(files, previousDeleteLoader)
  │           └─► BaseDVFileWriter → Puffin file
  │
  ├─► elif inputOrdered && rewritableDeletes == null:        V2, ordered,
  │     │                                                    nothing to merge
  │     └─► ClusteredPositionDeleteWriter(
  │           writers, files, io, targetFileSize, granularity)
  │
  └─► else:                                                  V2, unordered
        │                                                    OR V2, ordered
        │                                                    with rewritable
        │                                                    previous deletes
        └─► FanoutPositionOnlyDeleteWriter(
              writers, files, io, targetFileSize,
              granularity, previousDeleteLoader)
              │
              ├─► granularity = FILE (Spark default):
              │     deletes routed per referenced data file via
              │     RollingPositionDeleteWriter; usually one delete
              │     file per data file, but rolling may produce
              │     additional files when target-file-size is exceeded
              │
              └─► granularity = PARTITION:
                    deletes routed per (spec, partition) via
                    RollingPositionDeleteWriter; usually one delete
                    file per partition, but rolling may produce
                    additional files when target-file-size is exceeded

Granularity is controlled by `write.delete.granularity`.
SparkWriteConf.deleteGranularity() defaults to FILE in all current
versioned Spark modules (v3.5/v4.0/v4.1), overriding the table-level
default of PARTITION exposed by TableProperties. Only DVs are
inherently file-scoped.
```

---

## 6. Delete Application During Reads — Unified Flow

### Planning Phase (Driver)

```
DataTableScan.planFiles()                                    [core/DataTableScan]
  │
  └─► ManifestGroup.planFiles()                              [core/ManifestGroup]
        │
        ├─► Build DeleteFileIndex from all delete manifests:
        │     ManifestGroup constructor:
        │       deleteIndexBuilder =
        │         DeleteFileIndex.builderFor(io, deleteManifests)  [core/DeleteFileIndex]
        │     planFiles():
        │       DeleteFileIndex deleteFiles =
        │         deleteIndexBuilder.scanMetrics(scanMetrics).build()
        │       │
        │       ├─► Read all delete manifest files
        │       ├─► Index position deletes by partition
        │       ├─► Index equality deletes by partition + field IDs
        │       ├─► Index DVs by referenced data file path
        │       └─► Store global equality deletes separately
        │
        └─► For each data manifest entry:
              │
              ├─► deleteFiles = DeleteFileIndex.forDataFile(
              │     entry.dataSequenceNumber(),
              │     entry.file())
              │       │
              │       ├─► Sequence number filtering (per delete kind):
              │       │     • Position deletes / DVs:
              │       │         delete.dataSequenceNumber >= data.dataSequenceNumber
              │       │         (same-seq deletes apply; DVs additionally
              │       │         validated via ValidationException)
              │       │     • Equality deletes:
              │       │         delete.dataSequenceNumber > data.dataSequenceNumber
              │       │         (DeleteFileIndex caches an
              │       │         applySequenceNumber = dataSequenceNumber - 1)
              │       │
              │       ├─► global   = findGlobalDeletes(seq, file)      cross-partition eq deletes
              │       ├─► eqPart   = findEqPartitionDeletes(seq, file) partition-scoped eq deletes
              │       ├─► dv       = findDV(seq, file)                 keyed by dataFile.location()
              │       │
              │       └─► DV precedence:
              │             if dv != null && global == null && eqPart == null:
              │               return new DeleteFile[]{ dv }
              │             elif dv != null:
              │               return concat(global, eqPart, new DeleteFile[]{ dv })
              │             else:
              │               posPart = findPosPartitionDeletes(seq, file)
              │               posPath = findPathDeletes(seq, file)
              │               return concat(global, eqPart, posPart, posPath)
              │
              └─► yield FileScanTask(
                    file = dataFile,
                    deletes = deleteFiles[],     ◄── associated delete files
                    residual = residualFilter,
                    spec = partitionSpec)
```

### Sequence Number Rule

```
Timeline:

  Snapshot 1 (seq=1):  data-file-A and data-file-B added
                       (same partition P)
  Snapshot 2 (seq=2):  delete-file-X added
                       • position delete / DV variant: references
                         data-file-A only
                       • equality delete variant: scoped to
                         partition P, no file_path column
  Snapshot 3 (seq=3):  data-file-C added (same partition P)

  delete-file-X.dataSequenceNumber = 2

  Position delete / DV (rule: delete.seq >= data.seq AND
  referenced data file path matches):
    Applies to data-file-A?  YES  (2 >= 1 AND path matches)
    Applies to data-file-B?  NO   (2 >= 1 but path does NOT match —
                                   position deletes/DVs are
                                   file-scoped via file_path column
                                   or referencedDataFile())
    Applies to data-file-C?  NO   (2 < 3)

  Equality delete (rule: delete.seq > data.seq AND same partition):
    Applies to data-file-A?  YES  (2 > 1)
    Applies to data-file-B?  YES  (2 > 1)
    Applies to data-file-C?  NO   (2 < 3, equivalently
                                   applySeq=1 < 3)

  Same-sequence behavior differs by kind. If data-file-C and
  delete-file-X were both committed at seq=2 in the same RowDelta:
    • a position delete / DV with seq=2 still applies to its
      referenced data file (e.g. data-file-A) at seq=1
    • an equality delete with seq=2 does NOT apply to data-file-C
      at seq=2 (applySequenceNumber = 1 < 2), so newly inserted rows
      in the same commit are not retroactively masked.
```

### Execution Phase (Executors)

```
DeleteFilter.filter(CloseableIterable<T> records)            [data/DeleteFilter]
  │
  │  DELETE APPLICATION ORDER:
  │  Position deletes first, then equality deletes
  │
  ├─► Step 1: Apply position deletes
  │     applyPosDeletes(records)
  │       │
  │       ├─► Load position index:
  │       │     PositionDeleteIndex posIndex = deletedRowPositions()
  │       │       └─► deleteLoader.loadPositionDeletes(posDeletes, filePath)
  │       │             │
  │       │             ├─► For row-based position deletes
  │       │             │   (Parquet/Avro/ORC):
  │       │             │     Read the delete file
  │       │             │     Filter: file_path == current data file path
  │       │             │     For each (file_path, pos): bitmap.set(pos)
  │       │             │
  │       │             └─► For DVs (Puffin):
  │       │                   Read blob at contentOffset
  │       │                   Deserialize RoaringPositionBitmap
  │       │
  │       └─► Filter:
  │             isDeleted = record → posIndex.isDeleted(pos(record))
  │             return filterDeleted(records, isDeleted)
  │
  ├─► Step 2: Apply equality deletes (if any)
  │     applyEqDeletes(posFiltered)
  │       │
  │       ├─► For each equality delete group (same field IDs):
  │       │     StructLikeSet deleteSet = loadEqualityDeletes(files, schema)
  │       │     isEqDeleted = record → deleteSet.contains(project(record))
  │       │
  │       └─► Filter:
  │             combinedPredicate = allPredicates.reduce(OR)
  │             return filterDeleted(posFiltered, combinedPredicate)
  │
  └─► Return: fully filtered record stream

  Two modes of operation:
  ├─► Standard mode (no _is_deleted column):
  │     Deletes.filterDeleted(records, predicate)
  │     → Actually removes rows from output
  │
  └─► Marking mode (with _is_deleted metadata column):
        Deletes.markDeleted(records, predicate, markRowDeleted)
        → Sets _is_deleted=true but keeps rows in output (for CDC/changelogs)
```

---

## 7. SQL Operation -> Delete Type Matrix

```
┌───────────────┬────────────────────────┬─────────────────────────────────────────┐
│  SQL Command  │  Copy-on-Write (CoW)   │  Merge-on-Read (MoR)                    │
│               │  (write.*.mode=cow)    │  (write.*.mode=mor)                     │
├───────────────┼────────────────────────┼─────────────────────────────────────────┤
│               │                        │                                         │
│  DELETE       │  Rewrite data files    │  V2: Row-based position delete files    │
│  FROM ...     │  without deleted rows  │      (Parquet/Avro/ORC)                 │
│  WHERE ...    │                        │  V3+: Deletion vectors (Puffin); V3/V4  │
│               │                        │      forbid row-based position deletes  │
│               │  API: OverwriteFiles   │  API: RowDelta.addDeletes() and may     │
│               │                        │       removeDeletes(prior file-scoped   │
│               │                        │       /DV deletes that were merged in)  │
│               │  Snapshot op: overwrite│  Snapshot op: delete (no data files are │
│               │                        │      added; BaseRowDelta.operation()    │
│               │                        │      returns DELETE whenever delete     │
│               │                        │      files are added and no data files  │
│               │                        │      are added, regardless of removed   │
│               │                        │      delete files)                      │
│               │                        │                                         │
├───────────────┼────────────────────────┼─────────────────────────────────────────┤
│               │                        │                                         │
│  UPDATE       │  Rewrite data files    │  Position delete (or DV) for old row    │
│  ...          │  with updated values   │  + new data file with updated row       │
│  SET ...      │                        │                                         │
│               │  (delete + insert      │  (delete + insert, two file types)      │
│               │   within same file)    │                                         │
│               │                        │  API: RowDelta.addDeletes() +           │
│               │  API: OverwriteFiles   │       RowDelta.addRows()                │
│               │                        │  Snapshot op: overwrite                 │
│               │                        │                                         │
├───────────────┼────────────────────────┼─────────────────────────────────────────┤
│               │                        │                                         │
│  MERGE INTO   │  Rewrite data files    │  Position delete (or DV) for matched    │
│  ...          │  for matched rows      │  rows that are updated/deleted          │
│  WHEN MATCHED │  + new files for       │  + new data files for inserted/updated  │
│  WHEN NOT     │    inserted rows       │    rows                                 │
│  MATCHED      │                        │                                         │
│               │  API: OverwriteFiles   │  API: RowDelta                          │
│               │  (single op: removed   │  Snapshot op: append, delete, or        │
│               │   files + replacement  │    overwrite depending on what changed  │
│               │   + inserts together)  │                                         │
│               │                        │                                         │
├───────────────┼────────────────────────┼─────────────────────────────────────────┤
│               │                        │                                         │
│  INSERT INTO  │  (no deletes — append  │  (no deletes — append only)             │
│               │   only)                │                                         │
│               │                        │                                         │
│               │  API: AppendFiles      │  API: AppendFiles                       │
│               │                        │                                         │
└───────────────┴────────────────────────┴─────────────────────────────────────────┘

BaseRowDelta.operation() picks the snapshot operation from what was
added, not from what was removed:
  • APPEND    — adds data files only, no delete files added,
                no data files removed
  • DELETE    — adds delete files and no data files; the value of
                "delete files removed" does NOT affect this branch
  • OVERWRITE — every other mix (data files added, or data files
                removed together with delete additions, etc.)
```

### MoR: How UPDATE Works as Delete + Insert

```
UPDATE table SET name = 'Bob' WHERE id = 42

MoR execution flow:

1. SCAN: Find rows where id = 42
   → data-file-001.parquet, position 17:  {id=42, name="Alice"}

2. DELETE (position delete / DV):
   → delete-file (data-file-001.parquet, pos=17)

3. INSERT (new data file):
   → data-file-002.parquet, row 0:  {id=42, name="Bob"}

4. COMMIT:
   RowDelta
     .addDeletes(deleteFile)       position delete or DV
     .addRows(dataFile002)         new data file with updated row
     .commit()
```

---

## 8. RowDelta API

The `RowDelta` API is the primary interface for committing MoR changes (data files + delete files in a single atomic operation).

```
RowDelta API                                                 [api/RowDelta]
  │
  ├─► addRows(DataFile inserts)
  │     Add new data files (inserts or updated rows)
  │
  ├─► addDeletes(DeleteFile deletes)
  │     Add position delete files, equality delete files, or DVs
  │
  ├─► removeRows(DataFile file)               (default: UnsupportedOperationException)
  │     Remove a data file (for rewrite operations)
  │
  ├─► removeDeletes(DeleteFile deletes)       (default: UnsupportedOperationException)
  │     Remove old delete files (when rewriting/merging DVs)
  │
  ├─► validateFromSnapshot(long snapshotId)
  │     Set the baseline snapshot for conflict detection
  │
  ├─► caseSensitive(boolean caseSensitive)
  │     Control case sensitivity for expression binding during validation
  │
  ├─► validateDataFilesExist(Iterable<? extends CharSequence> referencedFiles)
  │     Ensure data files referenced by position deletes still exist
  │     (prevents dangling delete references)
  │
  ├─► validateDeletedFiles()
  │     Also validate that referenced data files were not removed by a
  │     concurrent delete operation (needed for read-and-reappend flows)
  │
  ├─► conflictDetectionFilter(Expression filter)
  │     Restrict conflict checks to rows matching this expression
  │
  ├─► validateNoConflictingDataFiles()
  │     Detect concurrent data modifications (serializable isolation)
  │
  ├─► validateNoConflictingDeleteFiles()
  │     Detect concurrent delete modifications
  │     (required for UPDATE and MERGE to prevent lost updates)
  │
  └─► commit()
        └─► BaseRowDelta extends MergingSnapshotProducer     [core/BaseRowDelta]
              │
              ├─► validate(base, parent):
              │     • startingSnapshotId is ancestor check
              │     • validateDataFilesExist for referenced paths
              │     • failMissingDeletePaths (if validateDeletedFiles)
              │     • validateAddedDataFiles (if validateNoConflictingDataFiles)
              │     • validateNoNewDeletesForDataFiles + validateNoNewDeleteFiles
              │       (if validateNoConflictingDeleteFiles)
              │     • validateNoConflictingFileAndPositionDeletes
              │       (blocks removing a data file that a new DV/position
              │        delete still references)
              │     • validateAddedDVs (V3 — at most one DV per data file)
              │
              ├─► Write new manifest with added/removed files
              ├─► Write manifest list
              ├─► operation() ∈ { APPEND, DELETE, OVERWRITE }
              └─► TableOperations.commit() → atomic CAS
```

### Isolation Levels

Spark's commit logic in `SparkPositionDeltaWrite` composes the
`RowDelta` validations from three signals: whether a scan ran, the
command kind (DELETE vs UPDATE/MERGE), and the configured isolation
level.

| Validation step                       | Applied when                                              |
|---------------------------------------|-----------------------------------------------------------|
| `validateDataFilesExist(referenced)`  | Always, when a scan ran                                   |
| `validateFromSnapshot(scanSnapshotId)`| Always, when the scan has a snapshot id                   |
| `conflictDetectionFilter(scanFilter)` | Always, when a scan ran                                   |
| `validateDeletedFiles()`              | `command == UPDATE` or `command == MERGE` only            |
| `validateNoConflictingDeleteFiles()`  | `command == UPDATE` or `command == MERGE` only            |
| `validateNoConflictingDataFiles()`    | `isolationLevel == SERIALIZABLE` only                     |

Implications:

- DELETE keeps the same set of validations under both isolation
  levels except for the data-file conflict check: it is added only for
  `serializable`.
- UPDATE and MERGE always validate concurrent delete-file additions
  regardless of isolation level, because un-deleting a row that was
  read-and-rewritten would silently corrupt the result.
- If the optimizer eliminates the scan (e.g. empty relation), no
  validations are added — the commit is independent of the table state.

| Level          | Behavior                                                                                       | Use Case                                                              |
|----------------|------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| `serializable` | All of the above PLUS `validateNoConflictingDataFiles()` — rejects concurrently appended rows. | Safe default for correctness                                          |
| `snapshot`     | Drops `validateNoConflictingDataFiles()`; concurrently appended rows are allowed.              | Higher throughput when concurrent inserts are expected and acceptable |

---

## 9. Compaction and Delete Cleanup

Over time, MoR tables accumulate delete files that degrade read performance. Compaction and maintenance operations clean these up.

### Compaction Applies Deletes

```
RewriteDataFiles (compaction)
  │
  ├─► Read data files WITH their associated delete files
  │     → deletes applied during read (rows filtered out)
  │
  ├─► Write new data files (without deleted rows)
  │
  └─► Commit: RewriteFiles
        .deleteFile(old data file)        ◄── rewritten data files removed
        .deleteFile(old DV)               ◄── dangling DVs explicitly removed
                                              by the compaction layer
                                              (RewriteFileGroup.danglingDVs()
                                              filters task.deletes() by
                                              ContentFileUtil::isDV)
        .addFile(new data file)           ◄── clean file, no pending deletes
        .commit()
              │
              └─► MergingSnapshotProducer.apply (generic, runs for every
                  commit type — not specific to compaction):
                    ├─► filterManager.filterManifests(...)
                    │     rewrites DATA manifests only (drops data
                    │     file entries explicitly removed by this
                    │     commit). Does not touch delete manifests
                    ├─► deleteFilterManager.dropDeleteFilesOlderThan(
                    │       minDataSequenceNumber)
                    │     applies to every kind of delete file (DVs,
                    │     row-based position deletes, equality
                    │     deletes): drops entries whose
                    │     dataSequenceNumber < minDataSequenceNumber,
                    │     since they cannot match any live row
                    └─► deleteFilterManager.removeDanglingDeletesFor(
                          filesToBeDeleted)
                          only drops DVs whose referencedDataFile is in
                          the removed set (ManifestFilterManager.
                          isDanglingDV gates this with
                          ContentFileUtil.isDV). Row-based position
                          deletes and equality deletes that referenced
                          a removed data file are NOT pruned here

Result: rewritten data files are replaced; dangling DVs attached to
them are dropped both by the compaction layer (RewriteFileGroup.
danglingDVs()) and by the generic isDanglingDV cleanup. The generic
dropDeleteFilesOlderThan path additionally evicts any delete file
(including row-based) whose sequence number is below the minimum live
data sequence number. Row-based position deletes and equality deletes
that still apply to surviving data files — or that became dangling
without crossing the sequence-number boundary — remain in the new
snapshot until they are explicitly rewritten or cleaned up.
```

Row-based dangling cleanup is handled by explicit maintenance paths,
not by the generic commit cleanup. Opt in with `remove-dangling-deletes`
(defaults to `false`) on `RewriteDataFiles` or run
`RewritePositionDeleteFiles` to compact and prune the position delete
files themselves.

### Compaction Triggers Based on Deletes

Defined in `BinPackRewriteFilePlanner` ([core/actions]) and passed as
`options` to `rewrite_data_files`:

| Property                 | Default          | Purpose                                            |
|--------------------------|------------------|----------------------------------------------------|
| `delete-file-threshold`  | `Integer.MAX_VALUE` | Rewrite data files associated with N+ delete files (disabled by default) |
| `delete-ratio-threshold` | `0.3`            | Rewrite data files where ≥30% of rows are deleted  |
| `max-files-to-rewrite`   | (unset)          | Cap the number of files rewritten in one planning pass |

Example:
```sql
CALL catalog.system.rewrite_data_files(
  table => 'db.table',
  options => map(
    'delete-file-threshold', '3',    -- rewrite if 3+ delete files
    'delete-ratio-threshold', '0.1'  -- rewrite if 10%+ rows deleted
  )
)
```

### Delete File Lifecycle

```
Time ──────────────────────────────────────────────────────────►

1. DELETE WHERE id=42 (MoR)
   → delete-file-001 created (references data-file-A)

2. DELETE WHERE id=99 (MoR)
   → delete-file-002 created (references data-file-A)

3. Compaction runs (RewriteDataFiles):
   → Read data-file-A, apply delete-file-001 + delete-file-002
   → Write data-file-B (clean, no deleted rows)
   → Commit: remove data-file-A, add data-file-B
     • DVs attached to data-file-A are dropped both by the compaction
       layer (RewriteFileGroup.danglingDVs() filters task.deletes()
       by ContentFileUtil::isDV) and by the generic isDanglingDV path
       in deleteFilterManager.removeDanglingDeletesFor(...)
     • dropDeleteFilesOlderThan(minSeq) additionally removes any
       delete file (DV, row-based position, equality) whose sequence
       number falls below the new minimum live data sequence number
     • Row-based position deletes and equality deletes that still
       apply to surviving data files — or that became dangling
       without crossing the sequence-number boundary — remain in the
       snapshot. The generic cleanup does NOT prune them; explicit
       maintenance (`remove-dangling-deletes` or
       `RewritePositionDeleteFiles`) is required

4. (optional) Proactive dangling delete cleanup:
   → re-run RewriteDataFiles with `remove-dangling-deletes=true`, OR
   → run RewritePositionDeleteFiles to compact away dangling positions
   → Commit drops delete-file-{001,002} from the live snapshot

5. ExpireSnapshots:
   → Old snapshots referencing delete-file-{001,002} expired
   → Files now orphaned (no snapshot references them)

6. RemoveOrphanFiles (or GC):
   → delete-file-{001,002} physically deleted from storage
```

### Related Maintenance Operations

| Operation                    | Purpose                                                      | API                                                            |
|------------------------------|--------------------------------------------------------------|----------------------------------------------------------------|
| `RewriteDataFiles`           | Compact data files, applying pending deletes                 | `SparkActions.rewriteDataFiles()`                              |
| `RemoveDanglingDeletes`      | Remove delete files that no longer reference live data       | `rewriteDataFiles().option("remove-dangling-deletes", "true")` |
| `ExpireSnapshots`            | Remove old snapshots, enabling GC of unreferenced files      | `SparkActions.expireSnapshots()`                               |
| `RemoveOrphanFiles`          | Delete files not referenced by any snapshot                  | `SparkActions.removeOrphanFiles()`                             |
| `RewritePositionDeleteFiles` | Rewrite fragmented position delete files for better locality | `SparkActions.rewritePositionDeletes()`                        |

---

## 10. Key Classes Reference

Spark classes live under the versioned Spark module
(`spark/v3.5/`, `spark/v4.0/`, `spark/v4.1/`); "spark/source" below
refers to that subtree of any version.

| Area                        | Class                            | Module       | Key Method / Note                                                     |
|-----------------------------|----------------------------------|--------------|-----------------------------------------------------------------------|
| **Delete types**            | `DeleteFile`                     | api          | Interface for all delete files; `referencedDataFile()`, `contentOffset()`, `contentSizeInBytes()` |
|                             | `FileContent`                    | api          | Enum: `DATA`, `POSITION_DELETES`, `EQUALITY_DELETES`, `DATA_MANIFEST`, `DELETE_MANIFEST` |
|                             | `PositionDelete<R>`              | core/deletes | `set(path, pos)` is canonical; `set(path, pos, row)` and `row()` deprecated in 1.11.0 |
| **Position delete writing** | `PositionDeleteWriter`           | core/deletes | `write(PositionDelete)`, `close()`                                    |
|                             | `RollingPositionDeleteWriter`    | core/io      | Splits large delete files by size                                     |
|                             | `ClusteredPositionDeleteWriter`  | core/io      | Assumes ordered input; granularity FILE or PARTITION                  |
|                             | `FanoutPositionOnlyDeleteWriter` | core/io      | Unordered input; optional `loadPreviousDeletes`                       |
|                             | `SortingPositionOnlyDeleteWriter`| core/deletes | Sorts positions per file before flushing                              |
|                             | `FileScopedPositionDeleteWriter` | core/deletes | Routes incoming deletes per referenced data file to its rolling delegate (target-size rolling can still split into multiple files) |
|                             | `DeleteGranularity`              | core/deletes | Enum: `FILE`, `PARTITION` — controls writer fan-out                   |
| **Equality delete writing** | `EqualityDeleteWriter`           | core/deletes | `write(T row)`, `close()`                                             |
|                             | `RollingEqualityDeleteWriter`    | core/io      | Splits large equality delete files                                    |
| **Deletion vector writing** | `DVFileWriter`                   | core/deletes | Interface: `delete(path, pos, spec, partition)`, `delete(path, index, spec, partition)` |
|                             | `BaseDVFileWriter`               | core/deletes | Constructor takes `Function<String, PositionDeleteIndex>` for previous deletes |
|                             | `PartitioningDVWriter`           | core/io      | Routes `PositionDelete` records to `BaseDVFileWriter`                 |
|                             | `PuffinWriter`                   | core/puffin  | Binary blob file format; `location()`, `fileSize()`                   |
|                             | `StandardBlobTypes.DV_V1`        | core/puffin  | Blob type id: `"deletion-vector-v1"`                                  |
| **Bitmap index**            | `PositionDeleteIndex`            | core/deletes | Interface: `delete(pos)`, `merge(other)`, `isDeleted(pos)`, `deleteFiles()` |
|                             | `BitmapPositionDeleteIndex`      | core/deletes | RoaringBitmap implementation                                          |
|                             | `RoaringPositionBitmap`          | core/deletes | 64-bit positions; portable Roaring serialization                       |
| **Delete filter (reads)**   | `DeleteFilter`                   | data         | `filter()` = `applyEqDeletes(applyPosDeletes(records))`; also `deletedRowPositions()`, `eqDeletedRowFilter()`, `findEqualityDeleteRows()` |
|                             | `BaseDeleteLoader`               | data         | `loadPositionDeletes(files, filePath)`, `loadEqualityDeletes(files, schema)`; detects DVs via `ContentFileUtil.containsSingleDV` |
|                             | `DeleteFileIndex`                | core         | `forDataFile(seq, file)` / `forEntry(entry)` — DV takes precedence    |
|                             | `Deletes`                        | core/deletes | Utility: `filterDeleted()`, `markDeleted()`, `toPositionIndex(es)`    |
| **Spark delete reads**      | `PositionDeletesRowReader`       | spark/source | Reads position delete files as data                                   |
|                             | `EqualityDeleteRowReader`        | spark/source | Reads equality delete files as data                                   |
|                             | `DVIterator`                     | spark/source | Extracts positions from Puffin DV blobs                               |
| **Spark MoR write**         | `SparkPositionDeltaOperation`    | spark/source | MoR entry point                                                       |
|                             | `SparkPositionDeltaWrite`        | spark/source | MoR write orchestration                                               |
|                             | `BaseDeltaWriter`                | spark/source | Base class for MoR task writers                                       |
|                             | `DeleteOnlyDeltaWriter`          | spark/source | DELETE-only MoR writer                                                |
|                             | `UnpartitionedDeltaWriter`       | spark/source | UPDATE/MERGE unpartitioned                                            |
|                             | `PartitionedDeltaWriter`         | spark/source | UPDATE/MERGE partitioned                                              |
| **Spark CoW write**         | `SparkCopyOnWriteOperation`      | spark/source | CoW entry point                                                       |
| **Mode selection**          | `SparkRowLevelOperationBuilder`  | spark/source | CoW vs MoR decision                                                   |
|                             | `RowLevelOperationMode`          | core         | Enum: `COPY_ON_WRITE("copy-on-write")`, `MERGE_ON_READ("merge-on-read")` |
| **Commit API**              | `RowDelta`                       | api          | `addRows`, `addDeletes`, `removeRows`, `removeDeletes`, `validate*`, `caseSensitive`, `conflictDetectionFilter` |
|                             | `BaseRowDelta`                   | core         | Extends `MergingSnapshotProducer`; `operation()` → APPEND/DELETE/OVERWRITE |
| **Configuration**           | `TableProperties`                | core         | `DELETE_MODE`, `UPDATE_MODE`, `MERGE_MODE`, isolation-level keys, `DELETE_GRANULARITY` |
|                             | `BinPackRewriteFilePlanner`      | core/actions | `DELETE_FILE_THRESHOLD`, `DELETE_RATIO_THRESHOLD`, `MAX_FILES_TO_REWRITE` |
| **Metadata**                | `MetadataColumns`                | core         | `DELETE_FILE_PATH`, `DELETE_FILE_POS`, `ROW_POSITION`, `IS_DELETED`, `CONTENT_OFFSET_COLUMN_ID`, `CONTENT_SIZE_IN_BYTES_COLUMN_ID` |
