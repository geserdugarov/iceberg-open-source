# Iceberg Write Path — Call Stack

This document traces the complete write path from a Spark SQL INSERT to Parquet file writing and metadata commit.

---

## 1. High-Level Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│  SPARK SQL:  INSERT INTO catalog.db.table VALUES (...)               │
│              CREATE TABLE ... AS SELECT ...                          │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │  TABLE RESOLUTION     │   DRIVER
                    │                       │
                    │  SparkTable           │
                    │    .newWriteBuilder() │
                    │       │               │
                    │       ▼               │
                    │  SparkWriteBuilder    │
                    │    .build()           │
                    │       │               │
                    │       ▼               │
                    │  SparkWrite           │
                    │    .toBatch()         │
                    │       │               │
                    │       ▼               │
                    │  BatchAppend (or      │
                    │  DynamicOverwrite/    │
                    │  OverwriteByFilter)   │
                    └───────────┬───────────┘
                                │
                                │ createBatchWriterFactory()
                                │ → broadcast SerializableTableWithSize
                                │   ship WriterFactory to executors
                                │
         ┌──────────────────────┴─────────────────────────┐
         │                                                │
         ▼                                                ▼
┌────────────────────┐                     ┌────────────────────┐
│  EXECUTOR 1        │                     │  EXECUTOR N        │
│                    │                     │                    │
│  WriterFactory     │                     │  WriterFactory     │
│    .createWriter() │                     │    .createWriter() │
│       │            │                     │       │            │
│       ▼            │                     │       ▼            │
│  DataWriter        │                     │  DataWriter        │
│  (Unpartitioned    │                     │  (Partitioned      │
│   or Partitioned)  │                     │   or Unpartitioned)│
│       │            │                     │       │            │
│       ▼            │                     │       ▼            │
│  RollingDataWriter │                     │  FanoutDataWriter  │
│  / ClusteredWriter │                     │  / ClusteredWriter │
│       │            │                     │       │            │
│       ▼            │                     │       ▼            │
│  Parquet.writeData │                     │  Parquet.writeData │
│       │            │                     │       │            │
│       ▼            │                     │       ▼            │
│  ParquetWriter     │                     │  ParquetWriter     │
│    .add(row)       │                     │    .add(row)       │
│       │            │                     │       │            │
│       ▼            │                     │       ▼            │
│  .parquet files    │                     │  .parquet files    │
│                    │                     │                    │
│  commit() →        │                     │  commit() →        │
│  TaskCommit(files) │                     │  TaskCommit(files) │
└────────┬───────────┘                     └────────┬───────────┘
         │                                          │
         └────────────────┬─────────────────────────┘
                          │ WriterCommitMessage[]
                          ▼
              ┌───────────────────────┐
              │  METADATA COMMIT      │   DRIVER
              │                       │
              │  BatchAppend.commit() │
              │       │               │
              │       ▼               │
              │  AppendFiles          │
              │  (MergeAppend, via    │
              │   table.newAppend())  │
              │    .appendFile(f)     │
              │    .commit()          │
              │       │               │
              │       ▼               │
              │  SnapshotProducer     │
              │    write manifests    │
              │    write manifest list│
              │    commit metadata    │
              │       │               │
              │       ▼               │
              │  NEW SNAPSHOT         │
              └───────────────────────┘
```

---

## 2. Detailed Call Stack

### Phase 1: Write Configuration (Driver)

```
SparkTable.newWriteBuilder(LogicalWriteInfo info)                [spark/source/SparkTable]
  └─► new SparkWriteBuilder(spark, table, branch, info)

SparkWriteBuilder                                                [spark/source/SparkWriteBuilder]
  implements: WriteBuilder, SupportsDynamicOverwrite, SupportsOverwriteV2
  │
  ├─► overwriteDynamicPartitions()                               dynamic partition overwrite
  ├─► overwrite(filters)                                         filter-based overwrite
  │
  └─► build()
        └─► new SparkWrite(spark, table, writeConf, info, ...)
              └─► toBatch()
                    ├─► BatchAppend (INSERT INTO)
                    ├─► DynamicOverwrite (INSERT OVERWRITE dynamic)
                    └─► OverwriteByFilter (INSERT OVERWRITE static)
```

### Phase 2: Writer Factory Creation (Driver)

```
BaseBatchWrite.createBatchWriterFactory(PhysicalWriteInfo)       [spark/source/SparkWrite]
  │   (inherited by BatchAppend / DynamicOverwrite / OverwriteByFilter)
  │
  └─► createWriterFactory()                                      [SparkWrite]
        │
        ├─► sparkContext.broadcast(
        │       SerializableTableWithSize.copyOf(table))         broadcast table to executors
        │
        └─► new WriterFactory(tableBroadcast, queryId, format,
                              outputSpecId, targetFileSize,
                              writeSchema, dsSchema,
                              useFanoutWriter, writeProperties,
                              sortOrderId)
              │
              └─► serialized & shipped to executors with each task
                  (the per-format SparkFileWriterFactory is built
                   lazily on the executor in createWriter(); see
                   Phase 3.)
```

### Phase 3: Per-Task Writing (Executors)

```
WriterFactory.createWriter(partitionId, taskId[, epochId])       [spark/source/SparkWrite]
  │
  ├─► table = tableBroadcast.value()                             resolve broadcast table
  ├─► spec  = table.specs().get(outputSpecId)
  │
  ├─► OutputFileFactory.builderFor(table, partitionId, taskId)
  │       .format(format).operationId(queryId + "-" + epochId)
  │       .build()                                               file naming / paths
  │
  ├─► SparkFileWriterFactory.builderFor(table)                   [spark/source/SparkFileWriterFactory]
  │       .dataFileFormat(format)                                  extends RegistryBasedFileWriterFactory
  │       .dataSchema(writeSchema)                                   <InternalRow, StructType>
  │       .dataSparkType(dsSchema)
  │       .writeProperties(writeProperties)
  │       .dataSortOrder(table.sortOrders().get(sortOrderId))
  │       .build()
  │
  │     (RegistryBasedFileWriterFactory resolves per-format
  │      writer builders via FormatModelRegistry, so the
  │      Spark-specific `SparkParquetWriters` / `SparkOrcWriter`
  │      are registered once in SparkFormatModels.register()
  │      rather than referenced inline from this factory.)
  │
  ├─► if spec.isUnpartitioned():
  │     └─► new UnpartitionedDataWriter(...)
  │           └─► wraps RollingDataWriter<InternalRow>
  │
  └─► else (partitioned):
        └─► new PartitionedDataWriter(...)
              │
              ├─► if useFanoutWriter:
              │     └─► wraps FanoutDataWriter<InternalRow>
              │           (keeps one writer open per partition seen)
              │
              └─► else:
                    └─► wraps ClusteredDataWriter<InternalRow>
                          (expects rows sorted by partition)

  (Version note: in spark/v3.5 these wrappers are plain
   `DataWriter<InternalRow>` and call `delegate.write(record, ...)`
   directly with no lineage decoration. In spark/v4.0 and
   spark/v4.1 they extend `DataWriterWithLineage<InternalRow>`,
   which adds `decorateWithRowLineage(meta, record)` on the
   write path shown below.)

DataWriter.write(InternalRow row)                                [spark/source/SparkWrite]
  │
  ├─► UnpartitionedDataWriter:
  │     └─► rollingWriter.write(record)                          v3.5
  │         rollingWriter.write(decorateWithRowLineage(meta,     v4.0 / v4.1
  │                                                    record))
  │           ├─► if currentWriter == null || file >= targetSize:
  │           │     closeCurrentWriter()
  │           │     openNewWriter()                              (rolls to new file)
  │           └─► currentWriter.write(row)
  │
  └─► PartitionedDataWriter:
        ├─► partitionKey.partition(internalRowWrapper.wrap(row)) compute partition key
        └─► delegate.write(record, spec, partitionKey)           v3.5
            delegate.write(decorateWithRowLineage(meta, record), v4.0 / v4.1
                           spec, partitionKey)
              ├─► route to partition-specific writer
              └─► writer.write(row)
```

### Phase 4: Parquet File Writing (Executors)

```
SparkFileWriterFactory.newDataWriter(file, spec, partition)      [data/RegistryBasedFileWriterFactory]
  │
  │  Format-agnostic dispatch via FormatModelRegistry — the
  │  per-format builder (Parquet / ORC / Avro) is selected by
  │  looking up the registered model for (format, InternalRow.class).
  │
  └─► FormatModelRegistry.dataWriteBuilder(                      [core/formats]
        format, InternalRow.class, encryptedOutputFile)
        .schema(dataSchema)
        .engineSchema(inputSchema)
        .setAll(tableProperties)
        .setAll(writerProperties)
        .metricsConfig(metricsConfig)
        .spec(spec)
        .partition(partition)
        .keyMetadata(keyMetadata)
        .sortOrder(sortOrder)
        .overwrite()
        .build()
          └─► returns DataWriter<InternalRow>
                wrapping the registered format writer
                (e.g. ParquetWriter<InternalRow> using
                 SparkParquetWriters.buildWriter as the
                 createWriterFunc — wired in
                 SparkFormatModels.register())

ParquetWriter<InternalRow>                                       [parquet/ParquetWriter]
  │
  └─► add(InternalRow value)
        ├─► recordCount += 1
        ├─► model.write(0, value)                                ParquetValueWriter tree
        │     └─► SparkParquetWriters encode each column:
        │           StructWriter
        │             ├─► IntWriter.write(col0)
        │             ├─► StringWriter.write(col1)
        │             ├─► DecimalWriter.write(col2)
        │             └─► ...
        │
        ├─► writeStore.endRecord()                               buffer in page store
        │
        └─► checkSize()
              └─► if buffered ≈ targetRowGroupSize:
                    flushRowGroup(false)
                      ├─► writer.startBlock(recordCount)         ParquetFileWriter
                      ├─► writeStore.flush()
                      ├─► pageStore.flushToFileWriter(writer)    ColumnChunkPageWriteStore
                      └─► writer.endBlock()

  close()  → void
    ├─► flushRowGroup(true)                                      drain remaining records
    ├─► writeStore.close()
    └─► writer.end(metadata)                                     write Parquet footer
                                                                  (schema, row groups, stats)

  metrics()  → Metrics                                           called after close() by the
    └─► ParquetMetrics.metrics(schema, parquetSchema,            wrapping DataWriter while
          metricsConfig, writer.getFooter(), model.metrics())    building the DataFile
```

### Phase 5: Task Commit (Executors → Driver)

```
Unpartitioned/PartitionedDataWriter.commit()                     [spark/source/SparkWrite]
  │
  ├─► close()                                                    closes the delegate writer
  │
  ├─► DataWriteResult result = delegate.result()
  │     └─► collect DataFile[] produced by the writer:
  │           DataFile
  │             ├─ filePath
  │             ├─ fileFormat (PARQUET / ORC / AVRO)
  │             ├─ partition
  │             ├─ recordCount
  │             ├─ fileSizeInBytes
  │             ├─ columnSizes
  │             ├─ valueCounts
  │             ├─ nullValueCounts
  │             ├─ lowerBounds
  │             └─ upperBounds
  │
  ├─► TaskCommit taskCommit = new TaskCommit(result.dataFiles())
  ├─► taskCommit.reportOutputMetrics()                           Spark output metrics
  │
  └─► return taskCommit                                          WriterCommitMessage
        └─► serialized, sent back to driver
```

### Phase 6: Metadata Commit (Driver)

```
BatchAppend.commit(WriterCommitMessage[] messages)               [spark/source/SparkWrite]
  │
  ├─► collect all DataFile[] from TaskCommit messages
  │
  ├─► AppendFiles append = table.newAppend()                     [api/Table]
  │     └─► new MergeAppend(name, ops)                           [core/MergeAppend]
  │           (BaseTable.newAppend() returns MergeAppend, which
  │            extends MergingSnapshotProducer to keep the manifest
  │            count low; FastAppend is only reachable via the
  │            explicit table.newFastAppend() entry point.)
  │
  ├─► for each DataFile:
  │     append.appendFile(dataFile)
  │
  └─► commitOperation(append, description)                       [SparkWrite]
        └─► append.commit()                                      [core/MergeAppend]
              └─► SnapshotProducer.commit()                      [core/SnapshotProducer]

SnapshotProducer.commit()                                        [core/SnapshotProducer]
  │
  ├─► apply() → produce new Snapshot
  │     │
  │     ├─► write new manifest files:
  │     │     ManifestWriter.write(manifestEntries)
  │     │       └─► Avro file with DataFile entries
  │     │           (status: ADDED for new files)
  │     │
  │     ├─► write manifest list:
  │     │     ManifestListWriter.write(manifests)
  │     │       └─► snap-<snapshotId>-<attempt>.avro
  │     │
  │     └─► create new Snapshot:
  │           ├─ snapshotId (unique)
  │           ├─ parentId (previous snapshot)
  │           ├─ sequenceNumber (incremented)
  │           ├─ timestampMillis
  │           ├─ operation = "append"
  │           ├─ summary (added-data-files, added-records, etc.)
  │           └─ manifestListLocation
  │
  └─► TableOperations.commit(base, updated)
        │
        ├─► write new metadata.json:
        │     v<N+1>.metadata.json
        │       ├─ current-snapshot-id = new snapshot
        │       ├─ snapshots[] += new snapshot
        │       └─ snapshot-log[] += entry
        │
        ├─► atomic compare-and-swap on metadata pointer
        │     (implementation varies by catalog)
        │
        └─► on conflict: retry with exponential backoff
              └─► re-read base, re-apply changes, commit again
```

---

## 3. Write Variants

```
┌─────────────────────┬─────────────────────────────────┬─────────────────────────┐
│  Operation          │  BatchWrite class               │  Iceberg SnapshotUpdate │
├─────────────────────┼─────────────────────────────────┼─────────────────────────┤
│  INSERT INTO        │  BatchAppend                    │  AppendFiles            │
│                     │                                 │  (MergeAppend via       │
│                     │                                 │   table.newAppend();    │
│                     │                                 │   FastAppend via        │
│                     │                                 │   table.newFastAppend())│
├─────────────────────┼─────────────────────────────────┼─────────────────────────┤
│  INSERT OVERWRITE   │  DynamicOverwrite               │  ReplacePartitions      │
│  (dynamic)          │                                 │                         │
├─────────────────────┼─────────────────────────────────┼─────────────────────────┤
│  INSERT OVERWRITE   │  OverwriteByFilter              │  OverwriteFiles         │
│  (static)           │                                 │                         │
├─────────────────────┼─────────────────────────────────┼─────────────────────────┤
│  DELETE / UPDATE /  │  SparkPositionDeltaOperation /  │  RowDelta               │
│  MERGE (MoR)        │  PositionDeltaBatchWrite        │  (data + delete files)  │
│                     │  (SparkPositionDeltaWrite)      │                         │
├─────────────────────┼─────────────────────────────────┼─────────────────────────┤
│  DELETE / UPDATE /  │  SparkCopyOnWriteOperation /    │  OverwriteFiles         │
│  MERGE (CoW)        │  CopyOnWriteOperation           │  (full file rewrite)    │
│                     │  (SparkWrite inner class)       │                         │
└─────────────────────┴─────────────────────────────────┴─────────────────────────┘
```

---

## 4. Key Classes Reference

| Step          | Class                            | Module       | Key Method                                       |
|---------------|----------------------------------|--------------|--------------------------------------------------|
| Entry         | `SparkTable`                     | spark/source | `newWriteBuilder()`                              |
| Config        | `SparkWriteBuilder`              | spark/source | `build()`                                        |
| Write         | `SparkWrite`                     | spark/source | `toBatch()`                                      |
| Batch         | `BatchAppend` / `DynamicOverwrite` / `OverwriteByFilter` | spark/source | `commit()` (`createBatchWriterFactory()` inherited from `BaseBatchWrite`) |
| Factory       | `WriterFactory`                  | spark/source | `createWriter(partitionId, taskId[, epochId])`   |
| Task writer   | `UnpartitionedDataWriter`        | spark/source | `write()`, `commit()`                            |
| Task writer   | `PartitionedDataWriter`          | spark/source | `write()`, `commit()`                            |
| Rolling       | `RollingDataWriter`              | core/io      | file size rolling                                |
| Fanout        | `FanoutDataWriter`               | core/io      | multi-partition write                            |
| Clustered     | `ClusteredDataWriter`            | core/io      | partition-sorted write                           |
| File factory  | `SparkFileWriterFactory`         | spark/source | builder pattern (`builderFor(table)`)            |
| Base factory  | `RegistryBasedFileWriterFactory` | data         | format-agnostic factory (uses FormatModelRegistry)|
| Format lookup | `FormatModelRegistry`            | core/formats | `dataWriteBuilder(format, type, file)`           |
| Spark formats | `SparkFormatModels`              | spark/source | `register()` (one-time at startup)               |
| Parquet write | `ParquetWriter`                  | parquet      | `add()`, `close()`                               |
| Value encode  | `SparkParquetWriters`            | spark/data   | `buildWriter()` (registered via FormatModel)     |
| Append        | `MergeAppend` (via `newAppend()`); `FastAppend` (via `newFastAppend()`) | core | `appendFile()`, `commit()`                       |
| Snapshot      | `SnapshotProducer`               | core         | `commit()`, `apply()`                            |
| Manifest      | `ManifestWriter`                 | core         | write manifest Avro                              |
| Metadata      | `TableOperations`                | core         | `commit(base, updated)`                          |

---

## 5. File Layout After Write

```
table-location/
├── metadata/
│   ├── v1.metadata.json
│   ├── v2.metadata.json               ◄── new version after commit
│   ├── snap-123456-0-uuid.avro        ◄── manifest list
│   ├── uuid-m0.avro                   ◄── manifest (new data files)
│   └── uuid-m1.avro                   ◄── manifest (existing files, carried over)
│
└── data/
    ├── partition=A/
    │   ├── 00000-0-uuid.parquet       ◄── new data file
    │   └── 00001-0-uuid.parquet       ◄── new data file
    └── partition=B/
        └── 00000-0-uuid.parquet       ◄── new data file
```
