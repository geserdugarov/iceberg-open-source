# Iceberg UPDATE and MERGE INTO (Upsert) Mechanisms

This document provides a comprehensive description of how Apache Iceberg implements row-level UPDATE and MERGE INTO (upsert) operations in Spark, covering both Copy-on-Write and Merge-on-Read strategies.

**Related docs:** [architecture_overview.md](architecture_overview.md) | [read_path_callstack.md](read_path_callstack.md) | [write_path_callstack.md](write_path_callstack.md) | [delete_mechanisms.md](delete_mechanisms.md) | [compaction_callstack.md](compaction_callstack.md)

---

## 1. Overview

### UPDATE

Modifies existing rows in a single Iceberg table based on a WHERE condition:

```sql
UPDATE catalog.db.table
SET value = 'new_value', updated_at = current_timestamp()
WHERE id = 42
```

### MERGE INTO (Upsert)

Joins a source dataset against a target Iceberg table and applies conditional INSERT, UPDATE, and DELETE in a single atomic transaction:

```sql
MERGE INTO target t
USING source s
ON t.id = s.id
WHEN MATCHED AND s.op = 'DELETE' THEN DELETE
WHEN MATCHED THEN UPDATE SET t.value = s.value, t.updated_at = s.updated_at
WHEN NOT MATCHED THEN INSERT (id, value, updated_at) VALUES (s.id, s.value, s.updated_at)
WHEN NOT MATCHED BY SOURCE AND t.updated_at < '2024-01-01' THEN DELETE
```

### Execution Strategies

Both operations use the same two strategies as DELETE (see [delete_mechanisms.md](delete_mechanisms.md)):

- **Copy-on-Write (CoW):** Read affected data files, apply modifications, write new files with updated rows. Old files replaced in one atomic commit.
- **Merge-on-Read (MoR):** Write position delete files (or DVs) for old rows + new data files for updated/inserted rows. Deletes applied at read time.

### UPDATE vs MERGE Comparison

```
┌────────────────────┬──────────────────────────┬──────────────────────────────┐
│                    │  UPDATE                  │  MERGE INTO                  │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ SQL                │ Single table + WHERE     │ Source-target JOIN +         │
│                    │                          │ WHEN MATCHED / NOT MATCHED   │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Logical Plan       │ UpdateTable              │ MergeIntoTable               │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Can INSERT?        │ No                       │ Yes (WHEN NOT MATCHED)       │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Can DELETE?        │ No                       │ Yes (WHEN MATCHED THEN       │
│                    │                          │ DELETE)                      │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ CoW metadata cols  │ _file + _pos             │ _file only                   │
│                    │ (+ row lineage cols)     │ (+ row lineage cols)         │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ CoW distribution   │ File-aware when          │ Standard append              │
│                    │ unpartitioned;           │ distribution (partition      │
│                    │ partition spec when      │ clustering / sort order)     │
│                    │ partitioned              │                              │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ MoR behavior       │ DELETE old + INSERT new  │ DELETE old + INSERT new      │
│                    │ (identical)              │ + INSERT-only for unmatched  │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Config property    │ write.update.mode        │ write.merge.mode             │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Isolation property │ write.update.isolation-  │ write.merge.isolation-       │
│                    │ level                    │ level                        │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Commit API (CoW)   │ OverwriteFiles           │ OverwriteFiles               │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Commit API (MoR)   │ RowDelta                 │ RowDelta                     │
└────────────────────┴──────────────────────────┴──────────────────────────────┘
```

---

## 2. High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│  SQL:  UPDATE table SET col=val WHERE cond                              │
│        MERGE INTO target USING source ON cond WHEN MATCHED ...          │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                    ┌──────────▼────────────┐
                    │  SPARK SQL PARSER     │
                    │                       │
                    │  UPDATE → UpdateTable │
                    │  MERGE  → MergeInto   │
                    │          Table        │
                    └──────────┬────────────┘
                               │
                    ┌──────────▼────────────┐
                    │  SPARK ANALYZER       │
                    │                       │
                    │  Iceberg row lineage  │
                    │  resolution rules     │
                    │  (v3.5 extensions     │
                    │  only — not yet in    │
                    │  v4.0 / v4.1):        │
                    │  RewriteUpdateTable   │
                    │  ForRowLineage        │
                    │  RewriteMergeInto     │
                    │  TableForRowLineage   │
                    └──────────┬────────────┘
                               │
                    ┌──────────▼────────────┐
                    │  SparkTable           │
                    │  .newRowLevelOp-      │
                    │   erationBuilder()    │
                    └──────────┬────────────┘
                               │
                    ┌──────────▼────────────┐
                    │  SparkRowLevel-       │
                    │  OperationBuilder     │
                    │                       │
                    │  Read mode + isolation│
                    │  from table props:    │
                    │  write.update.mode    │
                    │  write.merge.mode     │
                    │  write.{op}.isolation │
                    │  -level               │
                    └──────┬───────┬────────┘
                           │       │
            ┌──────────────┘       └───────────────────┐
            │ COPY_ON_WRITE                            │ MERGE_ON_READ
            │                                          │
            ▼                                          ▼
┌───────────────────────┐               ┌──────────────────────────┐
│ SparkCopyOnWrite-     │               │ SparkPositionDelta-      │
│ Operation             │               │ Operation                │
│                       │               │                          │
│ SCAN:                 │               │ SCAN:                    │
│  SparkCopyOnWriteScan │               │  SparkBatchQueryScan     │
│  (reads affected      │               │  (reads rows with        │
│   files entirely)     │               │   _file + _pos metadata) │
│                       │               │                          │
│ WRITE:                │               │ WRITE:                   │
│  Rewrite files with   │               │  Position deletes for    │
│  modified rows        │               │  old rows + new data     │
│                       │               │  files for updated rows  │
│                       │               │                          │
│ COMMIT:               │               │ COMMIT:                  │
│  OverwriteFiles       │               │  RowDelta                │
│  (delete old files    │               │  .addDeletes(deleteFile) │
│   + add new files)    │               │  .addRows(dataFile)      │
└───────────────────────┘               └──────────────────────────┘
```

---

## 3. Copy-on-Write UPDATE

### 3.1 Complete Call Stack

```
UPDATE catalog.db.table SET value = 'X' WHERE id = 42
       │
       ▼
Spark SQL Parser → UpdateTable logical plan               [Spark Core]
       │
       ▼
RewriteUpdateTableForRowLineage (resolution rule)         [spark/v3.5/spark-extensions only]
  └─► When the table supports row lineage, inject extra
      assignments into the UPDATE action:
        _row_id = _row_id                       (preserve original ID)
        _last_updated_sequence_number = null    (mark for next-seq backfill)
       │
       ▼
SparkTable.newRowLevelOperationBuilder(info)              [spark/source/SparkTable]
  └─► new SparkRowLevelOperationBuilder(spark, table, branch, info)
        │
        ├─► mode = properties.getOrDefault(UPDATE_MODE, UPDATE_MODE_DEFAULT)
        │     → default: "copy-on-write"
        ├─► isolationLevel = properties.getOrDefault(
        │       UPDATE_ISOLATION_LEVEL, UPDATE_ISOLATION_LEVEL_DEFAULT)
        │     → default: "serializable"
        │
        └─► return new SparkCopyOnWriteOperation(           [COPY_ON_WRITE path]
              spark, table, branch, info, isolationLevel)

SparkCopyOnWriteOperation                                [spark/source]
  │
  ├─► requiredMetadataAttributes() (UPDATE):
  │     _file (FILE_PATH)                      file identification
  │     _pos  (ROW_POSITION)                   row-level targeting
  │     _row_id, _last_updated_sequence_number (when row lineage supported)
  │
  ├─► SCAN PHASE:
  │     newScanBuilder()
  │       └─► anonymous SparkScanBuilder whose build()
  │           calls super.buildCopyOnWriteScan()
  │             └─► SparkCopyOnWriteScan
  │                   │
  │                   ├─► Captures scan snapshot ID
  │                   │     (baseline for conflict detection)
  │                   │
  │                   └─► Supports runtime filtering:
  │                         Spark pushes In(_file, [list])
  │                         to narrow scan to only affected files
  │
  ├─► SPARK EXECUTION:
  │     1. Scan affected files with metadata columns
  │     2. Apply WHERE filter → identify rows to update
  │     3. Apply SET expressions → compute new column values
  │     4. Output ALL rows from affected files:
  │        - Updated rows (with new values)
  │        - Unchanged rows (passed through as-is)
  │        (entire file must be rewritten, not just changed rows)
  │
  ├─► WRITE PHASE:
  │     newWriteBuilder()
  │       └─► SparkWriteBuilder.overwriteFiles(scan, command, isolationLevel)
  │             └─► SparkWrite.CopyOnWriteOperation
  │
  │     Distribution (SparkWriteUtil.copyOnWriteRequirements,
  │                   delegating to copyOnWriteDeleteUpdateDistribution
  │                   for UPDATE and DELETE):
  │       HASH mode:
  │         partitioned table   → cluster by table.spec() transforms
  │                              (NOT file-aware: rows from the same file
  │                               may land in different tasks; this groups
  │                               by partition for write locality)
  │         unpartitioned table → cluster by _file (FILE_CLUSTERING)
  │                              (file-aware: groups same-file rows)
  │       RANGE mode:
  │         partitioned / sorted table → order by SortOrderUtil.buildSortOrder
  │         otherwise                  → order by (_file, _pos)
  │       → on the unpartitioned path, groups rows from the same file
  │         together for efficient rewrite; on the partitioned path the
  │         grouping is partition-level only
  │
  │     Writers:
  │       Same writer infrastructure as INSERT
  │       (clustered / fanout data writers via SparkFileWriterFactory)
  │       → produce new Parquet data files
  │
  └─► COMMIT PHASE:
        SparkWrite.CopyOnWriteOperation.commit()         [spark/source/SparkWrite]
          │
          ├─► Collect overwrittenFiles + danglingDVs from scan tasks
          │     (original data files that were read and rewritten,
          │      plus any DVs that referenced only those files)
          │
          ├─► OverwriteFiles overwrite = table.newOverwrite()
          │
          ├─► overwrite.deleteFiles(overwrittenFiles, danglingDVs)
          │
          ├─► for each new DataFile:
          │     overwrite.addFile(newFile)
          │
          ├─► Validation (depends on isolation level; skipped only when
          │   the optimizer replaced the scan with an empty relation):
          │     │
          │     ├─► SERIALIZABLE  (commitWithSerializableIsolation):
          │     │     overwrite.validateFromSnapshot(scanSnapshotId)
          │     │     overwrite.conflictDetectionFilter(filter)
          │     │     overwrite.validateNoConflictingData()
          │     │     overwrite.validateNoConflictingDeletes()
          │     │
          │     └─► SNAPSHOT       (commitWithSnapshotIsolation):
          │           overwrite.validateFromSnapshot(scanSnapshotId)
          │           overwrite.conflictDetectionFilter(filter)
          │           overwrite.validateNoConflictingDeletes()
          │
          └─► commitOperation(overwrite, msg)
                └─► SnapshotProducer.commit()
                      atomic CAS on metadata.json
```

### 3.2 CoW UPDATE Data Flow Example

```
BEFORE:                                 DURING (Spark execution):
┌────────────────────┐                  ┌────────────────────────────────────┐
│ data-file-001      │                  │ Scan data-file-001:                │
│                    │                  │   Row 0: {id=41, val="A"} → pass   │
│ Row 0: id=41 "A"   │                  │   Row 1: {id=42, val="B"} → MATCH  │
│ Row 1: id=42 "B"  ←── UPDATE          │          SET val="X"               │
│ Row 2: id=43 "C"   │                  │   Row 2: {id=43, val="C"} → pass   │
│                    │                  │                                    │
└────────────────────┘                  │ Output ALL rows (modified + not):  │
                                        │   {id=41, val="A"} (unchanged)     │
                                        │   {id=42, val="X"} (updated)       │
                                        │   {id=43, val="C"} (unchanged)     │
                                        └────────────────────────────────────┘

AFTER:
┌────────────────────┐
│ data-file-001      │  ← DELETED (old file removed)
└────────────────────┘
┌────────────────────┐
│ data-file-002      │  ← NEW (rewritten with all rows)
│                    │
│ Row 0: id=41 "A"   │
│ Row 1: id=42 "X"   │  ← updated value
│ Row 2: id=43 "C"   │
└────────────────────┘

Commit: OverwriteFiles
  .deleteFile(data-file-001)
  .addFile(data-file-002)
  .commit()
```

---

## 4. Copy-on-Write MERGE INTO

### 4.1 Complete Call Stack

```
MERGE INTO target t USING source s ON t.id = s.id
  WHEN MATCHED THEN UPDATE SET t.value = s.value
  WHEN NOT MATCHED THEN INSERT (id, value) VALUES (s.id, s.value)
       │
       ▼
Spark SQL Parser → MergeIntoTable logical plan            [Spark Core]
  │  matchedActions: [UpdateAction(SET t.value = s.value)]
  │  notMatchedActions: [InsertAction(s.id, s.value)]
  │  notMatchedBySourceActions: []
       │
       ▼
RewriteMergeIntoTableForRowLineage (resolution rule)      [spark/v3.5/spark-extensions only]
  └─► When the table supports row lineage, inject lineage assignments
      into matchedActions (UPDATE) and notMatchedBySourceActions (UPDATE):
        _row_id = _row_id
        _last_updated_sequence_number = null
       │
       ▼
SparkRowLevelOperationBuilder                             [spark/source]
  └─► mode = properties.getOrDefault(MERGE_MODE, MERGE_MODE_DEFAULT)
        → default: "copy-on-write"
      isolationLevel = properties.getOrDefault(
          MERGE_ISOLATION_LEVEL, MERGE_ISOLATION_LEVEL_DEFAULT)
        → default: "serializable"
        → return SparkCopyOnWriteOperation(
              spark, table, branch, info, isolationLevel)
       │
       ▼
SparkCopyOnWriteOperation                                [spark/source]
  │
  ├─► requiredMetadataAttributes() (MERGE):
  │     _file only (_pos is added only for DELETE / UPDATE)
  │     _row_id, _last_updated_sequence_number (when row lineage supported)
  │
  ├─► SCAN PHASE:
  │     SparkScanBuilder.buildCopyOnWriteScan() → SparkCopyOnWriteScan
  │       Runtime filtering: In(_file, [affected files])
  │
  ├─► SPARK EXECUTION (JOIN + CLAUSE EVALUATION):
  │     │
  │     ├─► 1. Join target ⋈ source ON t.id = s.id
  │     │
  │     ├─► 2. For each result row:
  │     │     │
  │     │     ├─► Target-Source match (WHEN MATCHED):
  │     │     │     Evaluate conditions in clause order (first match wins):
  │     │     │       WHEN MATCHED AND cond1 THEN UPDATE SET ...
  │     │     │       WHEN MATCHED AND cond2 THEN DELETE
  │     │     │       WHEN MATCHED THEN UPDATE SET ...  (catch-all)
  │     │     │
  │     │     ├─► Source-only row (WHEN NOT MATCHED):
  │     │     │     New row to INSERT into target
  │     │     │
  │     │     └─► Target-only row (WHEN NOT MATCHED BY SOURCE):
  │     │           Apply UPDATE or DELETE to target row
  │     │
  │     └─► 3. Output: ALL rows from affected files
  │           (matched-updated + matched-deleted-excluded
  │            + unmatched-target-passthrough + new inserts)
  │
  ├─► WRITE PHASE:
  │     Distribution mode (SparkWriteConf.copyOnWriteMergeDistributionMode):
  │       1. If write.merge.distribution-mode is set, parse it and run
  │          through adjustWriteDistributionMode (downgrades range/hash
  │          to none for unpartitioned/unsorted tables).
  │       2. Else if the table is partitioned → return HASH
  │          (NOT range, even when the table has a sort order —
  │           this is the key difference from the generic
  │           write.distribution-mode logic, which would pick RANGE
  │           for any sorted table).
  │       3. Else (unpartitioned) → return distributionMode(), i.e.
  │          generic write.distribution-mode / defaultWriteDistributionMode.
  │
  │     Distribution shape (SparkWriteUtil.copyOnWriteRequirements):
  │       For MERGE the command is neither DELETE nor UPDATE, so the
  │       requirements builder falls through to writeRequirements()
  │       (same shape used by INSERT/APPEND, but driven by the MERGE-
  │       specific mode above):
  │         HASH  mode: cluster by table.spec() transforms (no file column)
  │         RANGE mode: order by SortOrderUtil.buildSortOrder
  │         NONE  mode: unspecified distribution
  │       → NOT file-aware (the join shuffles rows across files anyway)
  │
  │     Writers: clustered / fanout data writers → new Parquet data files
  │
  └─► COMMIT PHASE:
        Same as CoW UPDATE:
          SparkWrite.CopyOnWriteOperation.commit()
            OverwriteFiles
              .deleteFiles(overwrittenFiles, danglingDVs)
              .addFile(newFiles...)
              + isolation-level validation (see CoW UPDATE above)
              .commit()
```

### 4.2 CoW MERGE Data Flow Example

```
Target:                    Source:
┌──────────────────┐       ┌──────────────────┐
│ data-file-001    │       │ source DataFrame │
│ id=1, val="A"    │       │ id=1, val="A2"   │  ← match → UPDATE
│ id=2, val="B"    │       │ id=3, val="C"    │  ← no match → INSERT
│                  │       │                  │
└──────────────────┘       └──────────────────┘

After MERGE (CoW):
┌──────────────────┐       (data-file-001 deleted)
│ data-file-002    │
│ id=1, val="A2"   │       ← updated via WHEN MATCHED
│ id=2, val="B"    │       ← unchanged (passthrough)
│ id=3, val="C"    │       ← inserted via WHEN NOT MATCHED
└──────────────────┘

Commit: OverwriteFiles
  .deleteFile(data-file-001)
  .addFile(data-file-002)
```

---

## 5. Merge-on-Read UPDATE

### 5.1 Complete Call Stack

```
UPDATE catalog.db.table SET value = 'X' WHERE id = 42
       │                                           (write.update.mode = merge-on-read)
       ▼
SparkRowLevelOperationBuilder                             [spark/source]
  └─► mode = "merge-on-read"
        → return SparkPositionDeltaOperation(
              spark, table, branch, info, isolationLevel)

SparkPositionDeltaOperation                              [spark/source]
  │
  │  Implements RowLevelOperation + SupportsDelta:
  │    rowId() = [_file, _pos]                           row identification
  │    representUpdateAsDeleteAndInsert() = true         decompose UPDATE
  │
  ├─► requiredMetadataAttributes():
  │     _spec_id                                          partition spec routing
  │     _partition                                        partition values
  │     _row_id, _last_updated_sequence_number            (when row lineage)
  │   (_file and _pos are NOT in requiredMetadataAttributes — they are
  │    contributed separately through SupportsDelta.rowId().)
  │
  ├─► SCAN PHASE:
  │     newScanBuilder()
  │       └─► anonymous SparkScanBuilder whose build()
  │           calls super.buildMergeOnReadScan()
  │             └─► SparkBatchQueryScan
  │
  ├─► SPARK EXECUTION (DELTA DECOMPOSITION):
  │     │
  │     ├─► 1. Scan target table with _file + _pos metadata
  │     │
  │     ├─► 2. Apply WHERE filter → find rows to update
  │     │
  │     ├─► 3. Decompose each UPDATE into DELETE + INSERT:
  │     │     │
  │     │     │  For row at (data-file-001, pos=17): id=42, val="B"
  │     │     │
  │     │     ├─► DELETE marker:
  │     │     │     (_file="data-file-001", _pos=17)
  │     │     │
  │     │     └─► INSERT row:
  │     │           {id=42, val="X", ...}  (with new values from SET)
  │     │
  │     └─► 4. Output delta stream:
  │           delete markers + new data rows
  │
  ├─► WRITE PHASE:
  │     SparkPositionDeltaWrite                          [spark/source]
  │       │
  │       ├─► PositionDeltaWriteFactory.createWriter()
  │       │     │
  │       │     ├─► if command == DELETE (plain SQL DELETE only —
  │       │     │   MERGE never takes this branch, even when every
  │       │     │   clause is DELETE):
  │       │     │     DeleteOnlyDeltaWriter (no data writer)
  │       │     │
  │       │     ├─► elif unpartitioned:
  │       │     │     UnpartitionedDeltaWriter
  │       │     │
  │       │     └─► else (partitioned):
  │       │           PartitionedDeltaWriter
  │       │
  │       │  All three are inner classes of SparkPositionDeltaWrite.
  │       │  Each delta writer composes sub-writers:
  │       │    dataWriter   (BaseDeltaWriter.newDataWriter)
  │       │      → ClusteredDataWriter or FanoutDataWriter
  │       │    deleteWriter (BaseDeltaWriter.newDeleteWriter)
  │       │
  │       ├─► Delete writer selection (BaseDeltaWriter.newDeleteWriter):
  │       │     │
  │       │     ├─► if context.useDVs():                 V3+ table
  │       │     │     PartitioningDVWriter
  │       │     │       → Puffin file with RoaringBitmap blobs
  │       │     │       → also handles merging previous DVs via the
  │       │     │         PreviousDeleteLoader passed to the writer
  │       │     │
  │       │     ├─► elif inputOrdered && rewritableDeletes == null:
  │       │     │     ClusteredPositionDeleteWriter       V2 table
  │       │     │       → Parquet file with (file_path, pos)
  │       │     │
  │       │     └─► else:
  │       │           FanoutPositionOnlyDeleteWriter      non-DV path:
  │       │           → per-file Parquet delete files     input is unordered
  │       │             (also receives a                   OR rewritableDeletes
  │       │              PreviousDeleteLoader to           is non-null (merging
  │       │              merge file-scoped position        previous file-scoped
  │       │              delete files when present)        position deletes)
  │       │
  │       └─► Distribution (SparkWriteUtil.positionDeltaRequirements,
  │             delegating to positionDeltaUpdateMergeDistribution for
  │             UPDATE and MERGE):
  │             HASH mode:
  │               partitioned tbl   → cluster by (_spec_id, _partition)
  │                                  ++ table.spec() transforms
  │               unpartitioned tbl → cluster by
  │                                  (_spec_id, _partition, _file)
  │                                  ++ table.spec() transforms
  │             Local ordering (positionDeltaUpdateMergeOrdering):
  │               if fanoutEnabled AND table.sortOrder().isUnsorted():
  │                 EMPTY_ORDERING (no local ordering)
  │                 → input arrives unordered. On the NON-DV path this
  │                   makes BaseDeltaWriter.newDeleteWriter fall to
  │                   FanoutPositionOnlyDeleteWriter; when context.useDVs()
  │                   is true (V3+ tables) PartitioningDVWriter is still
  │                   chosen regardless of ordering.
  │               else:
  │                 (_spec_id, _partition, _file, _pos)
  │                 ++ table sort order
  │                 → groups delete markers by file. On the NON-DV path
  │                   this lets ClusteredPositionDeleteWriter run; on the
  │                   DV path the writer is still PartitioningDVWriter.
  │
  └─► COMMIT PHASE:
        PositionDeltaBatchWrite.commit(messages)         [spark/source/
                                                          SparkPositionDeltaWrite]
          │
          ├─► RowDelta rowDelta = table.newRowDelta()
          │
          ├─► for each DeltaTaskCommit message:
          │     │
          │     ├─► for each DataFile in message.dataFiles():
          │     │     rowDelta.addRows(dataFile)          new data with updated values
          │     │
          │     ├─► for each DeleteFile in message.deleteFiles():
          │     │     rowDelta.addDeletes(deleteFile)     position deletes / DVs
          │     │
          │     ├─► for each DeleteFile in message.rewrittenDeleteFiles():
          │     │     rowDelta.removeDeletes(deleteFile)  drop any delete file
          │     │                                         being replaced — DVs
          │     │                                         OR file-scoped position
          │     │                                         delete files that were
          │     │                                         merged into the new
          │     │                                         output
          │     │
          │     └─► referencedDataFiles += message.referencedDataFiles()
          │
          ├─► Validation (skipped only when the optimizer replaced the
          │   scan with an empty relation):
          │     rowDelta.conflictDetectionFilter(filter)           (always)
          │     rowDelta.validateDataFilesExist(referencedDataFiles)
          │     if scan.snapshotId() != null:
          │       rowDelta.validateFromSnapshot(scan.snapshotId())
          │     if command == UPDATE or command == MERGE:
          │       rowDelta.validateDeletedFiles()
          │       rowDelta.validateNoConflictingDeleteFiles()
          │     if isolationLevel == SERIALIZABLE:
          │       rowDelta.validateNoConflictingDataFiles()
          │
          └─► commitOperation(rowDelta, msg)
                └─► SnapshotProducer.commit()
```

### 5.2 MoR UPDATE Data Flow Example

```
BEFORE:                                 AFTER:
┌────────────────────┐                  ┌────────────────────┐
│ data-file-001      │                  │ data-file-001      │  (UNCHANGED)
│                    │                  │                    │
│ Row 0: id=41 "A"   │                  │ Row 0: id=41 "A"   │
│ Row 1: id=42 "B"  ◄── UPDATE          │ Row 1: id=42 "B"   │  (logically deleted)
│ Row 2: id=43 "C"   │                  │ Row 2: id=43 "C"   │
└────────────────────┘                  └────────────────────┘
                                        ┌────────────────────┐
                                        │ delete-file-001    │  (NEW: position delete)
                                        │ (data-file-001,    │
                                        │  pos=1)            │
                                        └────────────────────┘
                                        ┌────────────────────┐
                                        │ data-file-002      │  (NEW: updated row)
                                        │ id=42, val="X"     │
                                        └────────────────────┘

Commit: RowDelta
  .addDeletes(delete-file-001)    position delete for old row
  .addRows(data-file-002)         new file with updated row
  .commit()

At READ time:
  data-file-001 is read but row at pos=1 is filtered out (deleted)
  data-file-002 is read normally (new row with updated value)
  Result: id=41 "A", id=42 "X", id=43 "C"
```

---

## 6. Merge-on-Read MERGE INTO

### 6.1 Complete Call Stack

```
MERGE INTO target t USING source s ON t.id = s.id
  WHEN MATCHED AND s.op = 'DELETE' THEN DELETE
  WHEN MATCHED THEN UPDATE SET t.value = s.value
  WHEN NOT MATCHED THEN INSERT (id, value) VALUES (s.id, s.value)
       │                                           (write.merge.mode = merge-on-read)
       ▼
SparkPositionDeltaOperation                              [spark/source]
  │
  ├─► SCAN PHASE: same as MoR UPDATE
  │
  ├─► SPARK EXECUTION (JOIN + DELTA DECOMPOSITION):
  │     │
  │     ├─► 1. Join target ⋈ source ON t.id = s.id
  │     │
  │     ├─► 2. Evaluate clauses (first match wins per row):
  │     │
  │     │   WHEN MATCHED AND s.op = 'DELETE':
  │     │     → DELETE marker: (_file, _pos)                    delete only
  │     │
  │     │   WHEN MATCHED THEN UPDATE:
  │     │     → DELETE marker: (_file, _pos) for old row        delete + insert
  │     │     → INSERT row: {t.id, s.value, ...} for new row
  │     │
  │     │   WHEN NOT MATCHED THEN INSERT:
  │     │     → INSERT row: {s.id, s.value, ...}                insert only
  │     │
  │     └─► 3. Output: mixed delta stream
  │           (delete markers + data rows)
  │
  ├─► WRITE PHASE:
  │     SparkPositionDeltaWrite
  │       │
  │       ├─► The PositionDeltaWriteFactory `command == DELETE` branch
  │       │   only fires for plain SQL DELETE; MERGE — including a
  │       │   MERGE whose only clause is DELETE — always goes through
  │       │   the unpartitioned / partitioned writer below, which
  │       │   carries a dataWriter even when no inserts are emitted.
  │       │     │
  │       │     ├─► unpartitioned table → UnpartitionedDeltaWriter
  │       │     │       dataWriter + deleteWriter
  │       │     │
  │       │     └─► partitioned table   → PartitionedDeltaWriter
  │       │           dataWriter + deleteWriter, routed per partition
  │       │
  │       └─► DeltaTaskCommit output:
  │             dataFiles[]              new inserts + update-inserts
  │             deleteFiles[]            position deletes / DVs for matched rows
  │             rewrittenDeleteFiles[]   any delete files being replaced —
  │                                       DVs or file-scoped position delete
  │                                       files that were merged into the
  │                                       new output
  │             referencedDataFiles[]    data files referenced by delete markers
  │
  └─► COMMIT PHASE:
        Identical to MoR UPDATE — RowDelta with the same validation chain
        (conflictDetectionFilter, validateDataFilesExist, validateFromSnapshot,
         validateDeletedFiles + validateNoConflictingDeleteFiles for UPDATE/MERGE,
         and validateNoConflictingDataFiles when isolation is SERIALIZABLE).
```

### 6.2 MoR MERGE Data Flow Example

```
Target:                       Source:
┌──────────────────────┐      ┌─────────────────────────┐
│ data-file-001        │      │ source DataFrame        │
│ Row 0: id=1 val="A"  │      │ id=1 op="UPD" val="A2"  │ ← MATCHED → UPDATE
│ Row 1: id=2 val="B"  │      │ id=2 op="DEL"           │ ← MATCHED → DELETE
│                      │      │ id=3          val="C"   │ ← NOT MATCHED → INSERT
└──────────────────────┘      └─────────────────────────┘

After MERGE (MoR):
┌──────────────────────┐
│ data-file-001        │  (UNCHANGED)
│ Row 0: id=1 val="A"  │  (logically deleted by position delete)
│ Row 1: id=2 val="B"  │  (logically deleted by position delete)
└──────────────────────┘
┌──────────────────────┐
│ delete-file-001      │  (NEW: position deletes)
│ (data-file-001, 0)   │  ← delete for UPDATE on id=1
│ (data-file-001, 1)   │  ← delete for DELETE on id=2
└──────────────────────┘
┌──────────────────────┐
│ data-file-002        │  (NEW: inserts + update-inserts)
│ id=1 val="A2"        │  ← re-inserted with updated value
│ id=3 val="C"         │  ← new insert
└──────────────────────┘

Commit: RowDelta
  .addDeletes(delete-file-001)
  .addRows(data-file-002)
  .commit()
```

---

## 7. Row Identification and Metadata Columns

### 7.1 Metadata Columns

| Column                          | Internal Name                                  | Type    | Purpose                               |
|---------------------------------|------------------------------------------------|---------|---------------------------------------|
| `_file`                         | `MetadataColumns.FILE_PATH`                    | string  | Data file path for row identification |
| `_pos`                          | `MetadataColumns.ROW_POSITION`                 | long    | 0-based row position within the file  |
| `_spec_id`                      | `MetadataColumns.SPEC_ID`                      | int     | Partition spec version for routing    |
| `_partition`                    | `MetadataColumns.PARTITION_COLUMN_NAME`        | struct  | Partition values for routing          |
| `_row_id`                       | `MetadataColumns.ROW_ID`                       | long    | Row lineage identity (optional)       |
| `_last_updated_sequence_number` | `MetadataColumns.LAST_UPDATED_SEQUENCE_NUMBER` | long    | Last modification tracking (optional) |
| `_deleted`                      | `MetadataColumns.IS_DELETED`                   | boolean | Delete marking for changelogs (column name is `_deleted`; the constant in `MetadataColumns` is `IS_DELETED`) |

### 7.2 Which Columns Are Used Where

In CoW the operation's `requiredMetadataAttributes()` lists every metadata
column the planner must project. In MoR `requiredMetadataAttributes()` only
lists `_spec_id` and `_partition` (plus row lineage columns when enabled);
`_file` and `_pos` are contributed separately through
`SupportsDelta.rowId()`.

```
┌──────────────────┬───────────────────────┬───────────────────────┐
│  Metadata Column │  CoW                  │  MoR                  │
├──────────────────┼───────────────────────┼───────────────────────┤
│  _file           │  UPDATE: yes (scan +  │  yes (rowId)          │
│                  │   distribution)       │                       │
│                  │  MERGE: yes (scan)    │                       │
├──────────────────┼───────────────────────┼───────────────────────┤
│  _pos            │  UPDATE: yes (scan +  │  yes (rowId)          │
│                  │   distribution)       │                       │
│                  │  MERGE: no            │                       │
├──────────────────┼───────────────────────┼───────────────────────┤
│  _spec_id        │  no                   │  yes (partition spec  │
│                  │                       │   routing)            │
├──────────────────┼───────────────────────┼───────────────────────┤
│  _partition      │  no                   │  yes (partition       │
│                  │                       │   routing)            │
├──────────────────┼───────────────────────┼───────────────────────┤
│  _row_id         │  optional (lineage)   │  optional (lineage)   │
├──────────────────┼───────────────────────┼───────────────────────┤
│  _last_updated_  │  optional (lineage)   │  optional (lineage)   │
│  sequence_number │                       │                       │
└──────────────────┴───────────────────────┴───────────────────────┘
```

### 7.3 Row Lineage

When the table supports row lineage (detected via
`TableUtil.supportsRowLineage(table)`, which simply checks the table is
not a metadata table and its format version is `>= MIN_FORMAT_VERSION_ROW_LINEAGE`
— currently `3`; there is no separate feature flag in this code path),
the Iceberg-injected analyzer rules add lineage projections and
assignments to UPDATE and MERGE plans:

```
RewriteUpdateTableForRowLineage:
  For the UPDATE's assignments, append:
    - _row_id = _row_id                          (preserve original ID)
    - _last_updated_sequence_number = null       (backfilled to the new
                                                  snapshot's sequence
                                                  number on write)

RewriteMergeIntoTableForRowLineage:
  Same pair of assignments injected into:
    - every UPDATE action in matchedActions
    - every UPDATE action in notMatchedBySourceActions

These rules only rewrite the analyzed UPDATE / MERGE plan; they do NOT
modify `requiredMetadataAttributes()`. The lineage metadata columns are
appended independently by SparkCopyOnWriteOperation.requiredMetadataAttributes()
and SparkPositionDeltaOperation.requiredMetadataAttributes(), each of
which calls TableUtil.supportsRowLineage(table) and, when true, adds
_row_id and _last_updated_sequence_number — so the runtime projection
happens on every Spark version that includes those operation classes,
even on builds where the analyzer-side rewrite rules are absent.
```

Both rules are registered via `injectResolutionRule` in
`IcebergSparkSessionExtensions` in the **Spark 3.5 extensions module
only** (`spark/v3.5/spark-extensions`). At the time of writing, the
`spark/v4.0/spark-extensions` and `spark/v4.1/spark-extensions` modules
do not contain or register `RewriteUpdateTableForRowLineage` /
`RewriteMergeIntoTableForRowLineage`, so the assignment-injection
behaviour described here is Spark-3.5-only on the current branch. The
runtime-side row-lineage projections in
`SparkCopyOnWriteOperation.requiredMetadataAttributes()` and
`SparkPositionDeltaOperation.requiredMetadataAttributes()` still apply
on v4.0 / v4.1 whenever `TableUtil.supportsRowLineage(table)` returns
true.

This lets downstream consumers track row identity and last-modification
order across updates.

---

## 8. MERGE INTO Multi-Clause Evaluation

### 8.1 Clause Types

```sql
MERGE INTO target t
USING source s
ON t.id = s.id

-- WHEN MATCHED: source row matched a target row (join hit)
WHEN MATCHED AND s.op = 'DELETE' THEN DELETE
WHEN MATCHED AND s.op = 'UPDATE' THEN UPDATE SET t.value = s.value
WHEN MATCHED THEN UPDATE SET t.value = s.value         -- catch-all

-- WHEN NOT MATCHED: source row has no matching target (new row)
WHEN NOT MATCHED AND s.value IS NOT NULL THEN INSERT (id, value) VALUES (s.id, s.value)

-- WHEN NOT MATCHED BY SOURCE: target row has no matching source (orphan)
WHEN NOT MATCHED BY SOURCE AND t.updated_at < '2024-01-01' THEN DELETE
WHEN NOT MATCHED BY SOURCE THEN UPDATE SET t.status = 'orphan'
```

### 8.2 Evaluation Order

```
For each row in join result:
  │
  ├─► Target-Source MATCH (both sides present):
  │     Evaluate matchedActions[] in ORDER:
  │       1. Check condition of first clause
  │          → if true: apply action (UPDATE or DELETE), STOP
  │       2. Check condition of second clause
  │          → if true: apply action, STOP
  │       3. ... (first match wins)
  │       N. If no clause matches: row passes through unchanged
  │
  ├─► Source-only row (NOT MATCHED):
  │     Evaluate notMatchedActions[] in ORDER:
  │       → first matching INSERT clause applied
  │       → if none match: row is discarded
  │
  └─► Target-only row (NOT MATCHED BY SOURCE):
        Evaluate notMatchedBySourceActions[] in ORDER:
        → first matching UPDATE or DELETE clause applied
        → if none match: row passes through unchanged
```

### 8.3 Multi-Clause Example

```sql
MERGE INTO inventory t USING updates s ON t.sku = s.sku
WHEN MATCHED AND s.qty = 0 THEN DELETE                    -- clause 1: remove zero-qty
WHEN MATCHED THEN UPDATE SET t.qty = s.qty                -- clause 2: update qty
WHEN NOT MATCHED THEN INSERT (sku, qty) VALUES (s.sku, s.qty)  -- clause 3: new SKU

-- For each matched row:
--   If s.qty = 0: clause 1 fires (DELETE) → clause 2 is NOT evaluated
--   If s.qty > 0: clause 1 skipped → clause 2 fires (UPDATE)
```

---

## 9. Isolation Levels and Conflict Detection

### 9.1 Configuration

| Property                       | Default        | Values                     |
|--------------------------------|----------------|----------------------------|
| `write.update.isolation-level` | `serializable` | `serializable`, `snapshot` |
| `write.merge.isolation-level`  | `serializable` | `serializable`, `snapshot` |

### 9.2 Validation by Strategy

**Copy-on-Write (OverwriteFiles):**

| Isolation      | Validation Steps                                                                    |
|----------------|-------------------------------------------------------------------------------------|
| `serializable` | `validateFromSnapshot(scanSnapshotId)`                                              |
|                | `conflictDetectionFilter(combinedFilter)`                                           |
|                | `validateNoConflictingData()` — fails if concurrent INSERT into affected partitions |
|                | `validateNoConflictingDeletes()` — fails if concurrent DELETE on affected files     |
| `snapshot`     | `validateFromSnapshot(scanSnapshotId)`                                              |
|                | `conflictDetectionFilter(combinedFilter)`                                           |
|                | `validateNoConflictingDeletes()` — fails if concurrent DELETE on affected files     |

**Merge-on-Read (RowDelta):**

The MoR commit always sets `conflictDetectionFilter` and calls
`validateDataFilesExist`. `validateFromSnapshot` is called whenever the
scan captured a snapshot ID. `validateDeletedFiles` and
`validateNoConflictingDeleteFiles` are added for UPDATE and MERGE (not for
DELETE). `validateNoConflictingDataFiles` is added only when the isolation
level is SERIALIZABLE.

| Isolation      | Validation Steps                                                                         |
|----------------|------------------------------------------------------------------------------------------|
| `serializable` | `conflictDetectionFilter(filter)`                                                        |
|                | `validateDataFilesExist(referencedDataFiles)`                                            |
|                | `validateFromSnapshot(scanSnapshotId)` (when the scan has a snapshot)                    |
|                | `validateDeletedFiles()` — ensures referenced data files not deleted concurrently        |
|                | `validateNoConflictingDeleteFiles()` — fails if concurrent deletes on same rows          |
|                | `validateNoConflictingDataFiles()` — fails if concurrent INSERT into affected partitions |
| `snapshot`     | `conflictDetectionFilter(filter)`                                                        |
|                | `validateDataFilesExist(referencedDataFiles)`                                            |
|                | `validateFromSnapshot(scanSnapshotId)` (when the scan has a snapshot)                    |
|                | `validateDeletedFiles()`                                                                 |
|                | `validateNoConflictingDeleteFiles()`                                                     |

### 9.3 Concurrent Operation Scenario Matrix

```
┌─────────────────────────────────────────┬──────────────┬──────────────┐
│  Concurrent Operation                   │ SERIALIZABLE │ SNAPSHOT     │
├─────────────────────────────────────────┼──────────────┼──────────────┤
│ INSERT into same partition / filter     │ FAILS        │ SUCCEEDS     │
│ INSERT into clearly other partition     │ SUCCEEDS     │ SUCCEEDS     │
│ DELETE on same data files               │ FAILS        │ FAILS        │
│ UPDATE / MERGE on same data files       │ FAILS        │ FAILS        │
│ UPDATE / MERGE on clearly other parts.  │ SUCCEEDS     │ SUCCEEDS     │
│ Compaction on affected files            │ FAILS        │ FAILS        │
└─────────────────────────────────────────┴──────────────┴──────────────┘

FAILS    = commit is rejected; the engine may retry under
           `commit.retry.*` policy (see TableProperties).
SUCCEEDS = no conflict detected.

These outcomes are best-effort, not absolute. Iceberg's validators
(`validateNoConflictingData`, `validateNoConflictingDeletes`,
`validateNoConflictingDeleteFiles`, `validateNoConflictingDataFiles`)
flag any concurrent data / delete file that "can contain" or "can apply
to" rows matching the `conflictDetectionFilter`. When the validator
cannot prove file-level disjointness from manifest metrics and the
filter alone — for example, because the data file's lower / upper bounds
overlap the filter range, or the filter is broader than the actual rows
touched — the commit is rejected conservatively even though the two
operations may not actually overlap at the row level.
```

**Key difference:**
- `SERIALIZABLE`: Prevents phantom reads — concurrent INSERTs whose
  partition / file bounds intersect the operation's filter are rejected.
- `SNAPSHOT`: Tolerates phantom reads — only concurrent DELETEs and
  delete files whose scope intersects the operation's filter are
  rejected.

---

## 10. Configuration

| Property                            | Default                   | Values                           | Scope                                                                |
|-------------------------------------|---------------------------|----------------------------------|----------------------------------------------------------------------|
| `write.update.mode`                 | `copy-on-write`           | `copy-on-write`, `merge-on-read` | UPDATE operations                                                    |
| `write.merge.mode`                  | `copy-on-write`           | `copy-on-write`, `merge-on-read` | MERGE operations                                                     |
| `write.delete.mode`                 | `copy-on-write`           | `copy-on-write`, `merge-on-read` | DELETE operations (see [delete_mechanisms.md](delete_mechanisms.md)) |
| `write.update.isolation-level`      | `serializable`            | `serializable`, `snapshot`       | UPDATE conflict detection                                            |
| `write.merge.isolation-level`       | `serializable`            | `serializable`, `snapshot`       | MERGE conflict detection                                             |
| `write.update.distribution-mode`    | `hash`                    | `none`, `hash`, `range`          | UPDATE shuffle (both CoW and MoR; `SparkWriteConf.updateDistributionMode()`) |
| `write.merge.distribution-mode`     | `hash` (MoR); see below for CoW | `none`, `hash`, `range`    | MERGE shuffle                                                        |
| `write.delete.distribution-mode`    | `hash`                    | `none`, `hash`, `range`          | DELETE shuffle                                                       |
| `write.distribution-mode`           | derived (see below)       | `none`, `hash`, `range`          | Generic write shuffle for INSERT/APPEND and the CoW MERGE fallback   |
| `write.target-file-size-bytes`      | `536870912` (512 MB)      | long                             | Target data file size                                                |

`SparkWriteConf` reads the row-level distribution modes via dedicated methods:

- `updateDistributionMode()` — used by both CoW and MoR UPDATE; defaults to `hash`.
- `positionDeltaMergeDistributionMode()` — used by MoR MERGE; defaults to `hash`.
- `copyOnWriteMergeDistributionMode()` — used by CoW MERGE. If
  `write.merge.distribution-mode` is set, that value is parsed and run
  through `adjustWriteDistributionMode` (which downgrades `range`/`hash`
  to `none` on unpartitioned/unsorted tables). If it is unset, the
  method falls back to `HASH` for partitioned tables and to the generic
  `distributionMode()` / `defaultWriteDistributionMode()` derivation for
  unpartitioned tables.
- `deleteDistributionMode()` — used by both CoW and MoR DELETE; defaults to `hash`.

`write.distribution-mode` itself is only directly consulted on the
INSERT/APPEND path and the CoW MERGE fallback. When unset,
`SparkWriteConf.defaultWriteDistributionMode()` returns `range` for sorted
tables, `hash` for partitioned tables, and `none` otherwise;
`adjustWriteDistributionMode` further downgrades `range` to `none` for
unpartitioned/unsorted tables and `hash` to `none` for unpartitioned
tables. The dedicated UPDATE/MERGE/DELETE properties above do NOT go
through that derivation — they default to `hash` directly.

Set via table properties:
```sql
ALTER TABLE catalog.db.table SET TBLPROPERTIES (
  'write.update.mode' = 'merge-on-read',
  'write.merge.mode' = 'merge-on-read',
  'write.update.isolation-level' = 'snapshot'
);
```

---

## 11. Key Classes Reference

All Spark-side class paths below refer to the live `spark/v3.5/spark`
module; the equivalent classes also exist under `spark/v4.0/spark` and
`spark/v4.1/spark`.

The row-lineage rewrite rules (`RewriteUpdateTableForRowLineage` and
`RewriteMergeIntoTableForRowLineage`, registered via
`IcebergSparkSessionExtensions`) currently live ONLY under
`spark/v3.5/spark-extensions`. The `spark/v4.0/spark-extensions` and
`spark/v4.1/spark-extensions` modules do not contain or register these
rules, so the row-lineage assignment injection described above is a
Spark 3.5 behavior at the moment.

| Step                  | Class                                                      | Module           | Notes                                                                  |
|-----------------------|------------------------------------------------------------|------------------|------------------------------------------------------------------------|
| **Entry point**       | `SparkTable`                                               | spark/source     | `newRowLevelOperationBuilder()` returns SparkRowLevelOperationBuilder  |
| **Mode dispatch**     | `SparkRowLevelOperationBuilder`                            | spark/source     | `build()` reads mode + isolation from table properties                 |
| **CoW operation**     | `SparkCopyOnWriteOperation`                                | spark/source     | `requiredMetadataAttributes()`, `newScanBuilder`, `newWriteBuilder`    |
| **CoW scan**          | `SparkCopyOnWriteScan`                                     | spark/source     | Built via `SparkScanBuilder.buildCopyOnWriteScan()`                    |
| **CoW write builder** | `SparkWriteBuilder`                                        | spark/source     | `overwriteFiles(scan, command, isolationLevel)`                        |
| **CoW commit**        | `SparkWrite.CopyOnWriteOperation`                          | spark/source     | `commit()` → OverwriteFiles with `commitWith{Serializable,Snapshot}Isolation` |
| **MoR operation**     | `SparkPositionDeltaOperation`                              | spark/source     | `rowId()`, `representUpdateAsDeleteAndInsert()`                        |
| **MoR scan**          | `SparkBatchQueryScan`                                      | spark/source     | Built via `SparkScanBuilder.buildMergeOnReadScan()`                    |
| **MoR write builder** | `SparkPositionDeltaWriteBuilder`                           | spark/source     | Produces `SparkPositionDeltaWrite`                                     |
| **MoR write**         | `SparkPositionDeltaWrite`                                  | spark/source     | `toBatch()` → `PositionDeltaBatchWrite.commit()` with RowDelta         |
| **Delta commit msg**  | `SparkPositionDeltaWrite.DeltaTaskCommit`                  | spark/source     | `dataFiles`, `deleteFiles`, `rewrittenDeleteFiles`, `referencedDataFiles` |
| **MoR writers**       | `SparkPositionDeltaWrite.UnpartitionedDeltaWriter`         | spark/source     | Inner class; unpartitioned UPDATE/MERGE                                |
|                       | `SparkPositionDeltaWrite.PartitionedDeltaWriter`           | spark/source     | Inner class; partitioned UPDATE/MERGE                                  |
|                       | `SparkPositionDeltaWrite.DeleteOnlyDeltaWriter`            | spark/source     | Inner class; only used when command == DELETE (plain SQL DELETE) — MERGE never selects this path |
| **DV writer**         | `PartitioningDVWriter`                                     | core/io          | V3+ deletion vector writing; also merges previous DVs via the `PreviousDeleteLoader` it receives |
| **Pos delete writer** | `ClusteredPositionDeleteWriter`                            | core/io          | Ordered (non-DV) position deletes — chosen when input is ordered and no rewritable deletes |
|                       | `FanoutPositionOnlyDeleteWriter`                           | core/io          | Non-DV fallback: input is unordered OR rewritable file-scoped position delete files must be merged in (NOT used for DV merging — that stays on the `PartitioningDVWriter` path) |
| **Distribution**      | `SparkWriteUtil`                                           | spark            | `copyOnWriteRequirements()`, `positionDeltaRequirements()`             |
| **Row lineage**       | `RewriteUpdateTableForRowLineage`                          | spark/v3.5/spark-extensions only | Injects row lineage assignments into UPDATE (not present in v4.0 / v4.1 extensions yet) |
|                       | `RewriteMergeIntoTableForRowLineage`                       | spark/v3.5/spark-extensions only | Injects row lineage assignments into MERGE matched / not-matched-by-source actions (not present in v4.0 / v4.1 extensions yet) |
|                       | `IcebergSparkSessionExtensions`                            | spark-extensions | In v3.5, registers both rules via `injectResolutionRule`               |
| **CoW commit API**    | `OverwriteFiles`                                           | api              | `deleteFiles`, `addFile`, `conflictDetectionFilter`, `validateNoConflictingData`, `validateNoConflictingDeletes` |
| **MoR commit API**    | `RowDelta`                                                 | api              | `addRows`, `addDeletes`, `removeDeletes`, `conflictDetectionFilter`, `validateDataFilesExist`, `validateDeletedFiles`, `validateNoConflictingDeleteFiles`, `validateNoConflictingDataFiles` |
| **Commit base**       | `SnapshotProducer` / `MergingSnapshotProducer`             | core             | Underlies both OverwriteFiles and RowDelta commits                     |
| **Isolation**         | `IsolationLevel`                                           | core             | `SERIALIZABLE`, `SNAPSHOT`                                             |
| **Mode enum**         | `RowLevelOperationMode`                                    | core             | `COPY_ON_WRITE`, `MERGE_ON_READ`                                       |
| **Config**            | `TableProperties`                                          | core             | `UPDATE_MODE` / `MERGE_MODE` (+ `_DEFAULT`), `*_ISOLATION_LEVEL` (+ `_DEFAULT`) |
| **Row-lineage check** | `TableUtil.supportsRowLineage(table)`                      | core             | Gates row-lineage injection in both rewrite rules and operations       |
