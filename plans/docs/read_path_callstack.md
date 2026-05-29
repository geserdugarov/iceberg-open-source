# Iceberg Read Path — Call Stack

This document traces the complete read path from a Spark SQL query to Parquet file reading.

---

## 1. High-Level Flow

```
┌────────────────────────────────────────────────────────────┐
│  SPARK SQL:  SELECT * FROM catalog.db.table WHERE x > 10   │
└───────────────────────────────┬────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │  TABLE RESOLUTION     │
                    │                       │
                    │  Either entry point:  │
                    │                       │
                    │  (a) Catalog SQL      │
                    │      SparkCatalog     │
                    │        .loadTable()   │
                    │                       │
                    │  (b) format("iceberg")│
                    │      IcebergSource    │
                    │        .getTable()    │
                    │        → catalog      │
                    │          .loadTable() │
                    │                       │
                    │  Both return          │
                    │  SparkTable           │
                    └───────────┬───────────┘
                                │ .newScanBuilder()
                    ┌───────────▼───────────┐
                    │  SCAN BUILDING        │   DRIVER
                    │                       │
                    │  SparkScanBuilder     │◄── pushdown: predicates,
                    │    .build()           │    columns, aggregates,
                    │       │               │    limit
                    │       ▼               │
                    │  SparkBatchQueryScan  │
                    │  (wraps Iceberg       │
                    │   BatchScan built     │
                    │   via                 │
                    │   table.newBatchScan()│
                    │   or SparkDistributed-│
                    │   DataScan)           │
                    └───────────┬───────────┘
                                │ SparkScan.toBatch()
                    ┌───────────▼───────────┐
                    │  ICEBERG SCAN PLANNING│   DRIVER
                    │                       │
                    │  SparkPartitioning-   │
                    │  AwareScan            │
                    │    .taskGroups()      │
                    │       │ (lazy)        │
                    │       ▼               │
                    │  BatchScan.planFiles()│
                    │   default:            │
                    │   BatchScanAdapter(   │
                    │    DataTableScan      │
                    │    .doPlanFiles());   │
                    │   distributed-plan:   │
                    │   SparkDistributed-   │
                    │   DataScan extends    │
                    │   BaseDistributedData-│
                    │   Scan.doPlanFiles()  │
                    │       │               │
                    │       ▼               │
                    │  ManifestGroup        │
                    │    .planFiles()       │
                    │       │               │
                    │       ▼               │
                    │  CloseableIterable    │
                    │  of FileScanTask      │
                    │  (or other            │
                    │   PartitionScanTask)  │
                    │       │               │
                    │       ▼               │
                    │  TableScanUtil        │
                    │    .planTaskGroups()  │
                    │       │               │
                    │       ▼               │
                    │  List<ScanTaskGroup   │
                    │       <T extends      │
                    │    PartitionScanTask>>│
                    │       │               │
                    │       ▼               │
                    │  SparkBatch           │
                    │  .planInputPartitions()│
                    └───────────┬───────────┘
                                │ serialized as SparkInputPartition[]
                                │
         ┌──────────────────────┴─────────────────────────┐
         │                                                │
         ▼                                                ▼
┌────────────────────┐                     ┌────────────────────┐
│  EXECUTOR 1        │                     │  EXECUTOR N        │
│                    │                     │                    │
│  SparkRowReader-   │                     │  SparkRowReader-   │
│  Factory           │                     │  Factory           │
│    .createReader() │                     │    .createReader() │
│       │            │                     │       │            │
│       ▼            │                     │       ▼            │
│  RowDataReader     │                     │  RowDataReader     │
│    .next()         │                     │    .next()         │
│       │            │                     │       │            │
│       ▼            │                     │       ▼            │
│  FormatModel-      │                     │  FormatModel-      │
│  Registry          │                     │  Registry          │
│    .readBuilder()  │                     │    .readBuilder()  │
│       │            │                     │       │            │
│       ▼            │                     │       ▼            │
│  ParquetReader     │                     │  ParquetReader     │
│    .iterator()     │                     │    .iterator()     │
│       │            │                     │       │            │
│       ▼            │                     │       ▼            │
│  InternalRow       │                     │  InternalRow       │
└────────────────────┘                     └────────────────────┘
```

---

## 2. Detailed Call Stack

### Phase 1: Table Resolution (Driver)

Two independent entry points both end at `SparkTable`:

```
(a) Catalog SQL — SELECT/INSERT against a Spark catalog identifier
    Spark SQL Parser
      └─► SparkCatalog.loadTable(ident)                          [spark/SparkCatalog]
            └─► Iceberg Catalog.loadTable(TableIdentifier)       [api/Catalog]
                  └─► returns Table (BaseTable)                  [core/BaseTable]
                        └─► returns SparkTable                   [spark/source/SparkTable]

(b) DataSourceV2 — spark.read.format("iceberg").load(...)
    IcebergSource.getTable(StructType schema,
                           Transform[] partitioning,
                           Map<String,String> options)           [spark/source/IcebergSource]
      │  Resolves a Spark catalog from the `path` option, then
      │  delegates to that catalog's loadTable(Identifier). Two
      │  sub-cases inside SparkCatalog.load(Identifier, ...):
      │
      ├─► identifier path (catalog.db.table):
      │     icebergCatalog.loadTable(buildIdentifier(ident))
      │       └─► Iceberg Catalog.loadTable(TableIdentifier)     [api/Catalog]
      │             └─► returns SparkTable
      │
      └─► location path (PathIdentifier — e.g. "s3://bucket/path"):
            SparkCatalog.loadPath(PathIdentifier, ...)
              └─► tables.load(location[#metadataTable])          [HadoopTables / similar]
                    └─► returns SparkTable
                  (this path goes through tables.load, NOT
                   icebergCatalog.loadTable)
```

### Phase 2: Scan Building (Driver)

```
SparkTable.newScanBuilder(options)                               [spark/source/SparkTable]
  └─► new SparkScanBuilder(spark, table, schema, snapshot, branch, timeTravel, options)
        │
        │  Spark optimizer calls pushdown methods. These live on two layers:
        │
        │  (a) Inherited from BaseSparkScanBuilder:
        ├─► pushPredicates(Predicate[])                          predicate pushdown
        │      → SparkV2Filters.convert(predicate) → Iceberg Expression
        │      → returns post-scan predicates Spark must still apply
        ├─► pruneColumns(StructType)                             column pruning
        ├─► pushLimit(int)                                       limit pushdown
        │
        │  (b) Implemented directly on SparkScanBuilder:
        ├─► pushAggregation(Aggregation)                         aggregate pushdown
        │     → may short-circuit to SparkLocalScan when stats answer the query
        │
        └─► build()                                              [SparkScanBuilder]
              └─► buildBatchScan()
                    └─► new SparkBatchQueryScan(
                          spark, table, schema, snapshot, branch,
                          buildIcebergBatchScan(projection, ...),
                          readConf, projection, filters, scanReportSupplier)
                    │
                    │  buildIcebergBatchScan() creates an Iceberg BatchScan:
                    └─► table.newBatchScan()                     [api/Table]
                          (default returns BatchScanAdapter(DataTableScan));
                          or new SparkDistributedDataScan(spark, table, readConf)
                          when distributed planning is enabled.
                          .caseSensitive(...)
                          .filter(combinedFilter)
                          .project(projection)
                          .metricsReporter(reporter)
                          [.useSnapshot(snapshotId)]
                          [.includeColumnStats()]
```

### Phase 3: Partition Planning (Driver)

```
SparkScan.toBatch()                                              [spark/source/SparkScan]
  │  (inherited by SparkBatchQueryScan)
  │
  ├─► taskGroups()                                               [SparkPartitioningAwareScan]
  │     │  lazy — first call materializes file scan tasks and groups them
  │     │
  │     ├─► tasks()                                              [SparkPartitioningAwareScan]
  │     │     └─► scan.planFiles()                               (Iceberg BatchScan)
  │     │           │  scan is the BatchScan built in Phase 2:
  │     │           │   • default          → BatchScanAdapter(DataTableScan)
  │     │           │                        → DataTableScan.doPlanFiles()
  │     │           │   • distributed plan → SparkDistributedDataScan
  │     │           │                        extends BaseDistributedDataScan
  │     │           │                        → BaseDistributedDataScan.doPlanFiles()
  │     │           │
  │     │           └─► (default path shown below — distributed plan
  │     │                fans the same ManifestGroup-style work out to
  │     │                Spark executors)
  │     │                 └─► ManifestGroup.planFiles()          [core/ManifestGroup]
  │     │                       │
  │     │                       ├─► for each ManifestFile in snapshot:
  │     │                       │     ├─► ManifestEvaluator.eval(manifest)
  │     │                       │     │     [api/expressions/ManifestEvaluator]
  │     │                       │     │     (skip manifest if partition
  │     │                       │     │      summaries don't match filter)
  │     │                       │     │
  │     │                       │     └─► ManifestFiles.read(manifest, io, specsById)
  │     │                       │           .filterRows(...)
  │     │                       │           .filterPartitions(...)
  │     │                       │           .caseSensitive(...)
  │     │                       │           .select(columns)
  │     │                       │           .scanMetrics(scanMetrics)
  │     │                       │           returns a ManifestReader
  │     │                       │           whose liveEntries()/entries()
  │     │                       │           drives the loop:
  │     │                       │           └─► for each ManifestEntry:
  │     │                       │                 ├─► InclusiveMetricsEvaluator
  │     │                       │                 │   [api/expressions]
  │     │                       │                 │     .eval(dataFile)  (column stats)
  │     │                       │                 │
  │     │                       │                 ├─► DeleteFileIndex
  │     │                       │                 │     .forEntry(entry)
  │     │                       │                 │     (passes the ManifestEntry
  │     │                       │                 │      so its data-sequence
  │     │                       │                 │      number is respected
  │     │                       │                 │      when selecting deletes)
  │     │                       │                 │
  │     │                       │                 └─► yield FileScanTask
  │     │                       │                       ├─ file: DataFile
  │     │                       │                       ├─ deletes: List<DeleteFile>
  │     │                       │                       ├─ residual: Expression
  │     │                       │                       └─ spec: PartitionSpec
  │     │                       │
  │     │                       └─► CloseableIterable<FileScanTask>
  │     │
  │     └─► TableScanUtil.planTaskGroups(tasks, splitSize,
  │                                       splitLookback, openFileCost)
  │           bin-pack tasks (T extends PartitionScanTask) into a
  │           List<ScanTaskGroup<T>>. For SparkBatchQueryScan the type
  │           parameter is fixed by its parent chain
  │             SparkRuntimeFilterableScan
  │               extends SparkPartitioningAwareScan<PartitionScanTask>
  │           so T is PartitionScanTask at the type-system level;
  │           regular reads carry FileScanTask instances inside that
  │           group and SparkRowReaderFactory dispatches on the runtime
  │           task type (FileScanTask vs PositionDeletesScanTask, both
  │           of which extend PartitionScanTask via ContentScanTask).
  │           SparkBatch finally holds the result as
  │           List<? extends ScanTaskGroup<?>>.
  │
  │           Note: changelog reads do NOT go through this path.
  │           SparkChangelogScan implements Scan directly (it is not a
  │           SparkPartitioningAwareScan) and its taskGroups() calls
  │           scan.planTasks() on an IncrementalChangelogScan to obtain
  │           List<ScanTaskGroup<ChangelogScanTask>> before constructing
  │           SparkBatch — ChangelogScanTask only extends ScanTask, not
  │           PartitionScanTask, so TableScanUtil.planTaskGroups is not
  │           used here.
  │
  └─► new SparkBatch(sparkContext, table, fileIO, readConf,
                     groupingKeyType, taskGroups, projection, hashCode)

SparkBatch.planInputPartitions()                                 [spark/source/SparkBatch]
  │  taskGroups are already materialized; this step only wraps them
  │  for serialization to executors.
  │
  ├─► broadcast SerializableTableWithSize.copyOf(table)
  ├─► broadcast SerializableFileIOWithSize.wrap(fileIO)
  ├─► computePreferredLocations() (data or executor cache locality, if enabled)
  │
  └─► for each ScanTaskGroup:
        wrap as SparkInputPartition                              [spark/source]
          ├─ taskGroup: ScanTaskGroup<?>
          ├─ tableBroadcast / fileIOBroadcast
          ├─ projectionString (SchemaParser.toJson(projection))
          ├─ groupingKeyType
          ├─ caseSensitive / cacheDeleteFilesOnExecutors
          └─ preferredLocations

SparkBatch.createReaderFactory()                                 [spark/source/SparkBatch]
  │
  ├─► if Parquet vectorization is enabled and every projected field is a
  │       primitive type or a metadata column, and every task reads Parquet only:
  │     return new SparkColumnarReaderFactory(parquetBatchReadConf)
  │
  ├─► else if ORC vectorization is enabled and every task reads ORC only
  │       (with no delete files):
  │     return new SparkColumnarReaderFactory(orcBatchReadConf)
  │
  └─► else:
        return new SparkRowReaderFactory()
```

### Phase 4: Data Reading (Executors)

```
SparkRowReaderFactory.createReader(inputPartition)               [spark/source]
  │  (SparkColumnarReaderFactory follows the same shape for batches)
  │
  ├─► if partition.allTasksOfType(FileScanTask.class):
  │     return new RowDataReader(partition)
  ├─► else if partition.allTasksOfType(ChangelogScanTask.class):
  │     return new ChangelogRowReader(partition)
  └─► else if partition.allTasksOfType(PositionDeletesScanTask.class):
        return new PositionDeletesRowReader(partition)

RowDataReader extends BaseRowReader<FileScanTask>                [spark/source/RowDataReader]
  extends BaseReader<InternalRow, FileScanTask>                  [spark/source/BaseReader]
  implements PartitionReader<InternalRow>

BaseReader.next()                                                [spark/source/BaseReader]
  │
  ├─► if currentIterator.hasNext() → return true
  │
  └─► while tasks.hasNext():
        ├─► currentTask = tasks.next()                           (next FileScanTask)
        ├─► currentIterator = open(currentTask)                  [RowDataReader.open(FileScanTask)]
        │
        └─► open(FileScanTask task)                              [RowDataReader]
              │
              ├─► Build SparkDeleteFilter for this file:
              │     deleteFilter = new SparkDeleteFilter(
              │       filePath, task.deletes(), counter(), true)
              │     requiredSchema = deleteFilter.requiredSchema()
              │     idToConstant   = constantsMap(task, requiredSchema)
              │
              ├─► InputFileBlockHolder.set(filePath, start, length)
              │     (so Spark's input_file_name() reports the right file)
              │
              ├─► open(task, requiredSchema, idToConstant)       [RowDataReader (protected)]
              │     │
              │     ├─► if task.isDataTask():
              │     │     newDataIterable(task.asDataTask(), schema)
              │     │       (used for metadata tables; in-memory rows)
              │     │
              │     └─► else (data file path):
              │           newIterable(inputFile, format,         [BaseRowReader]
              │                       start, length, residual,
              │                       projection, idToConstant)
              │             │
              │             │  Format-agnostic dispatch via the
              │             │  FormatModel registry (no inline
              │             │  Parquet/Avro/ORC switch):
              │             │
              │             └─► FormatModelRegistry              [core/formats]
              │                   .readBuilder(format,
              │                                InternalRow.class,
              │                                inputFile)
              │                   .project(projection)
              │                   .idToConstant(idToConstant)
              │                   .reuseContainers()
              │                   .split(start, length)
              │                   .caseSensitive(...)
              │                   .filter(residual)
              │                   .withNameMapping(...)
              │                   .build()
              │                     └─► returns CloseableIterable<InternalRow>
              │
              └─► deleteFilter.filter(iter).iterator()           [SparkDeleteFilter — inner class of BaseReader]
                    ├─ apply position deletes / deletion vectors
                    └─ apply equality deletes
```

The per-format reader function (e.g. `SparkParquetReaders.buildReader(...)`,
`VectorizedSparkParquetReaders.buildReader(...)`, `SparkOrcReader`, ...) is no
longer wired inline in `RowDataReader`. It is registered once per JVM in
`SparkFormatModels.register()` and looked up by `(format, InternalRow.class)` /
`(format, ColumnarBatch.class)` from `FormatModelRegistry`. The registry itself
lives in `core/src/main/java/org/apache/iceberg/formats/` and is bootstrapped
statically: it reflectively invokes the `register()` method on each known
`*FormatModels` class (generic, Arrow, Flink, Spark) the first time it loads.

### Phase 5: Parquet File Reading (Executors)

```
ParquetReader<InternalRow>.iterator()                            [parquet/ParquetReader]
  │
  ├─► init() — lazily build ReadConf<InternalRow>                [parquet/ReadConf]
  │     │
  │     ├─► ParquetFileReader.open(inputFile)                    (Apache parquet-mr)
  │     │     └─► reads file footer + row group metadata
  │     │
  │     ├─► Iceberg ID handling (kept-id schema, NameMapping,
  │     │   or fallback IDs) → prune to projection schema
  │     │
  │     ├─► row group pruning (only when a residual filter exists):
  │     │     for each row group, evaluate
  │     │       ├─► ParquetMetricsRowGroupFilter      (column min/max)
  │     │       ├─► ParquetDictionaryRowGroupFilter   (dictionary lookup)
  │     │       └─► ParquetBloomRowGroupFilter        (bloom filters)
  │     │     and set shouldSkip[rowGroup] = true when none match.
  │     │
  │     └─► build the ParquetValueReader<InternalRow> tree ONCE  [spark/data/SparkParquetReaders]
  │           via readerFunc supplied by SparkFormatModels:
  │             StructReader (InternalRow)
  │               ├─► IntReader        (column 0)
  │               ├─► StringReader     (column 1)
  │               ├─► DecimalReader    (column 2)
  │               ├─► TimestampReader  (column 3)
  │               └─► ... (one reader per projected column)
  │
  └─► return new FileIterator<InternalRow>(conf)
        │
        ├─► hasNext(): valuesRead < totalValues
        │
        └─► next():
              ├─► if (valuesRead >= nextRowGroupStart) advance()
              │     │
              │     ├─► while (shouldSkip[nextRowGroup])
              │     │     reader.skipNextRowGroup(); nextRowGroup++
              │     │
              │     ├─► pages = reader.readNextRowGroup()
              │     │     (column chunks loaded into memory)
              │     │
              │     └─► model.setPageSource(pages)
              │
              └─► model.read(reuseContainers ? last : null)
                    └─► returns InternalRow
                          → back to Spark SQL execution engine
```

---

## 3. Key Classes Reference

| Step              | Class                          | Module                  | Key Method                                       |
|-------------------|--------------------------------|-------------------------|--------------------------------------------------|
| Entry (catalog SQL)| `SparkCatalog`                 | spark                   | `loadTable(Identifier)` → returns `SparkTable`   |
| Entry (DSv2)      | `IcebergSource`                | spark/source            | `getTable(schema, partitioning, options)` → delegates to a Spark catalog's `loadTable` |
| Table             | `SparkTable`                   | spark/source            | `newScanBuilder(options)`                        |
| Scan builder      | `SparkScanBuilder`             | spark/source            | `build()`, `pushAggregation(Aggregation)`        |
| Pushdown base     | `BaseSparkScanBuilder`         | spark/source            | `pushPredicates(Predicate[])`, `pruneColumns(StructType)`, `pushLimit(int)` |
| Spark scan        | `SparkBatchQueryScan`          | spark/source            | inherits `SparkScan.toBatch()`                   |
| Scan abstraction  | `SparkScan`                    | spark/source            | `toBatch()`                                      |
| Task grouping     | `SparkPartitioningAwareScan`   | spark/source            | `tasks()`, `taskGroups()` (call `scan.planFiles()`) |
| Physical batch    | `SparkBatch`                   | spark/source            | `planInputPartitions()`, `createReaderFactory()` |
| Partition         | `SparkInputPartition`          | spark/source            | serialized `ScanTaskGroup`                       |
| Reader factory    | `SparkRowReaderFactory`        | spark/source            | `createReader(InputPartition)`                   |
| Columnar factory  | `SparkColumnarReaderFactory`   | spark/source            | `createColumnarReader(InputPartition)`           |
| Row reader        | `RowDataReader`                | spark/source            | `open(FileScanTask)`                             |
| Base reader       | `BaseReader`                   | spark/source            | task iteration loop (`next()`)                   |
| File open         | `BaseRowReader`                | spark/source            | `newIterable()` (format-agnostic dispatch)       |
| Iceberg batch scan| `BatchScan` / `BatchScanAdapter`| api                     | `planFiles()` (wraps `DataTableScan`)            |
| File planning     | `DataTableScan`                | core                    | `doPlanFiles()` (returns `ManifestGroup.planFiles()`) |
| Distributed plan  | `SparkDistributedDataScan`     | spark (Spark planning)  | `planFiles()` distributed across Spark executors |
| Manifest grouping | `ManifestGroup`                | core                    | `planFiles()`                                    |
| Manifest read     | `ManifestReader`               | core                    | reads manifest Avro files                        |
| Partition eval    | `ManifestEvaluator`            | api/expressions         | `eval(ManifestFile)`                             |
| Stats eval        | `InclusiveMetricsEvaluator`    | api/expressions         | `eval(DataFile)`                                 |
| Task bin-packing  | `TableScanUtil`                | core/util               | `planTaskGroups(tasks, splitSize, ...)`          |
| Format lookup     | `FormatModelRegistry`          | core/formats            | `readBuilder(format, type, file)`                |
| Spark formats     | `SparkFormatModels`            | spark/source            | `register()` (one-time at startup)               |
| Parquet read      | `ParquetReader`                | parquet                 | `iterator()` → `FileIterator`                    |
| Row group prune   | `ReadConf`                     | parquet                 | initializes `shouldSkip[]` via metrics/dict/bloom row-group filters |
| Value decode      | `SparkParquetReaders`          | spark/data              | `buildReader()` (registered via `SparkFormatModels`) |
| Vectorized decode | `VectorizedSparkParquetReaders`| spark/data/vectorized   | `buildReader()` (registered via `SparkFormatModels`) |
| Delete filter     | `SparkDeleteFilter`            | spark/source            | inner class of `BaseReader`; `filter(iter)`      |

---

## 4. Filter Pushdown Pipeline

```
Spark Predicate[] (SQL WHERE clause, V2 filters)
       │
       ▼
BaseSparkScanBuilder.pushPredicates(Predicate[])
       │
       ├─► SparkV2Filters.convert(predicate) → Iceberg Expression
       ├─► Binder.bind(projection, expr, caseSensitive) — bind to schema
       ├─► classify each predicate:
       │     (1) fully evaluated by Iceberg  (selects entire partitions)
       │     (2) partially evaluated         (file pruning + residual)
       │     (3) not pushable                (Spark evaluates)
       │
       └─► return (2) ∪ (3) as post-scan predicates Spark must still apply
       │
       ▼
SparkScanBuilder.buildBatchScan() → BatchScan.filter(combinedExpression)
       │
       ▼
BatchScan.planFiles()
  (default: BatchScanAdapter(DataTableScan.doPlanFiles());
   distributed plan: SparkDistributedDataScan
     extends BaseDistributedDataScan.doPlanFiles())
       │
       ├─► ManifestEvaluator        [api/expressions]
       │     skip entire manifests based on partition field summaries
       │
       ├─► InclusiveMetricsEvaluator [api/expressions]
       │     skip data files based on column lower/upper bounds + null counts
       │
       ├─► ResidualEvaluator        [api/expressions]
       │     attach per-file residual to each FileScanTask
       │     (the part of the filter not subsumed by the partition value)
       │
       ▼
Parquet ReadConf (per file)         [parquet/ReadConf]
       │
       ├─► ParquetMetricsRowGroupFilter      (column min/max)
       ├─► ParquetDictionaryRowGroupFilter   (dictionary)
       └─► ParquetBloomRowGroupFilter        (bloom filters)
             populate shouldSkip[rowGroup] before iteration starts
       │
       ▼
Post-scan filtering (Spark)
       │
       └─► Spark applies the post-scan predicates returned by
           pushPredicates() — i.e. the partial and unpushable filters.
           The Parquet reader itself does not evaluate the residual
           row-by-row; ReadConf only consults it for row-group pruning.
```
