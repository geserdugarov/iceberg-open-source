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
                    │  SparkCatalog         │
                    │    .loadTable()       │
                    │       │               │
                    │       ▼               │
                    │  IcebergSource        │
                    │    .getTable()        │
                    │       │               │
                    │       ▼               │
                    │  SparkTable           │
                    └───────────┬───────────┘
                                │ .newScanBuilder()
                    ┌───────────▼───────────┐
                    │  SCAN BUILDING        │   DRIVER
                    │                       │
                    │  SparkScanBuilder     │◄── pushdown: filters,
                    │    .build()           │    columns, aggregates
                    │       │               │
                    │       ▼               │
                    │  SparkBatchQueryScan  │
                    │    .toBatch()         │
                    │       │               │
                    │       ▼               │
                    │  SparkBatch           │
                    │    .planInputParts()  │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │  ICEBERG SCAN PLANNING│   DRIVER
                    │                       │
                    │  DataTableScan        │
                    │    .planFiles()       │
                    │       │               │
                    │       ▼               │
                    │  ManifestGroup        │
                    │    .planFiles()       │
                    │       │               │
                    │       ▼               │
                    │  FileScanTask[]       │
                    │  (grouped into        │
                    │   ScanTaskGroup[])    │
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

```
Spark SQL Parser
  └─► SparkCatalog.loadTable(ident)                              [spark/source]
        └─► Catalog.loadTable(TableIdentifier)                   [api]
              └─► returns Table (BaseTable)                      [core]
                    └─► SparkTable wraps Table                   [spark/source]

(Alternatively, when using the DataSourceV2 `format("iceberg")` path:
  IcebergSource.getTable(StructType, Transform[], Map<String,String> options)
    └─► resolves to SparkTable via SparkCatalog)
```

### Phase 2: Scan Building (Driver)

```
SparkTable.newScanBuilder(options)                               [spark/source/SparkTable]
  └─► new SparkScanBuilder(spark, table, readConf, schema, ...)
        │
        │  Spark optimizer calls pushdown methods:
        ├─► pushPredicates(Predicate[])                          filter pushdown
        ├─► pruneColumns(StructType)                             column pruning
        └─► pushAggregation(Aggregation)                         aggregate pushdown
        │
        └─► build()                                              [SparkScanBuilder]
              └─► new SparkBatchQueryScan(...)
                    │
                    │  Internally creates Iceberg TableScan:
                    └─► table.newScan()                          [api/Table]
                          .filter(icebergFilter)
                          .select(columns)
                          .caseSensitive(...)
```

### Phase 3: Partition Planning (Driver)

```
SparkBatchQueryScan.toBatch()                                    [spark/source/SparkScan]
  └─► new SparkBatch(...)

SparkBatch.planInputPartitions()                                 [spark/source/SparkBatch]
  │
  ├─► tableScan.planFiles()                                      [core/DataTableScan]
  │     └─► ManifestGroup.planFiles()                            [core/ManifestGroup]
  │           │
  │           ├─► for each ManifestFile in snapshot:
  │           │     ├─► ManifestEvaluator.eval(manifest)         partition pruning
  │           │     │     (skip manifest if partition stats
  │           │     │      don't match filter)
  │           │     │
  │           │     └─► ManifestReader.open(manifest)
  │           │           └─► for each ManifestEntry:
  │           │                 ├─► InclusiveMetricsEvaluator     column stats pruning
  │           │                 │     .eval(dataFile)
  │           │                 │
  │           │                 └─► yield FileScanTask
  │           │                       ├─ file: DataFile
  │           │                       ├─ deletes: List<DeleteFile>
  │           │                       ├─ residual: Expression
  │           │                       └─ spec: PartitionSpec
  │           │
  │           └─► returns CloseableIterable<FileScanTask>
  │
  ├─► group into ScanTaskGroup[] (bin-packing by split size)
  │
  └─► wrap each group as SparkInputPartition                     [spark/source]
        ├─ taskGroup: ScanTaskGroup<FileScanTask>
        ├─ broadcast table reference
        └─ expected schema

SparkBatch.createReaderFactory()                                 [spark/source/SparkBatch]
  └─► new SparkRowReaderFactory(...)                             (or SparkColumnarReaderFactory)
```

### Phase 4: Data Reading (Executors)

```
SparkRowReaderFactory.createReader(inputPartition)               [spark/source]
  └─► new RowDataReader(partition)

RowDataReader extends BaseRowReader<FileScanTask>                [spark/source/RowDataReader]
  extends BaseReader<InternalRow, FileScanTask>                  [spark/source/BaseReader]

BaseReader.next()                                                [spark/source/BaseReader]
  │
  ├─► if currentIterator.hasNext() → return true
  │
  └─► while tasks.hasNext():
        ├─► task = tasks.next()                                  (next FileScanTask)
        ├─► currentIterator = open(task)                         [RowDataReader.open(FileScanTask)]
        │
        └─► open(FileScanTask task)                              [RowDataReader]
              │
              ├─► Build SparkDeleteFilter for this file:
              │     deleteFilter = new SparkDeleteFilter(
              │       filePath, task.deletes(), counter(), true)
              │     requiredSchema = deleteFilter.requiredSchema()
              │     idToConstant   = constantsMap(task, requiredSchema)
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
              └─► deleteFilter.filter(iter).iterator()           [SparkDeleteFilter (inner class of BaseReader)]
                    ├─ apply position deletes / deletion vectors
                    └─ apply equality deletes
```

The per-format reader function (e.g. `SparkParquetReaders.buildReader(...)`,
`VectorizedSparkParquetReaders.buildReader(...)`, `SparkOrcReader`, ...) is no
longer wired inline in `RowDataReader`. It is registered once per JVM in
`SparkFormatModels.register()` and looked up by `(format, InternalRow.class)` /
`(format, ColumnarBatch.class)` from `FormatModelRegistry`.

### Phase 5: Parquet File Reading (Executors)

```
ParquetReader<InternalRow>.iterator()                            [parquet/ParquetReader]
  └─► new FileIterator<InternalRow>()
        │
        ├─► ParquetFileReader.open(inputFile)                    (Apache parquet-mr)
        │     └─► reads file footer, row group metadata
        │
        ├─► applyFilter(filter)                                  row group pruning
        │     └─► skip row groups where stats don't match
        │
        └─► for each matching RowGroup:
              ├─► readNextRowGroup()
              │     └─► ParquetFileReader.readRowGroup(i)
              │           └─► reads column chunks into memory
              │
              ├─► build ParquetValueReader<InternalRow>          [spark/data/SparkParquetReaders]
              │     └─► creates reader tree:
              │           StructReader (InternalRow)
              │             ├─► IntReader (column 0)
              │             ├─► StringReader (column 1)
              │             ├─► DecimalReader (column 2)
              │             ├─► TimestampReader (column 3)
              │             └─► ... (one reader per projected column)
              │
              └─► for each record in RowGroup:
                    └─► reader.read(reuse)
                          └─► returns InternalRow
                                → back to Spark SQL execution engine
```

---

## 3. Key Classes Reference

| Step           | Class                       | Module       | Key Method                                  |
|----------------|-----------------------------|--------------|---------------------------------------------|
| Entry          | `IcebergSource`             | spark/source | `getTable(schema, partitioning, options)`   |
| Table          | `SparkTable`                | spark/source | `newScanBuilder()`                          |
| Scan build     | `SparkScanBuilder`          | spark/source | `build()`, `pushPredicates()`               |
| Logical scan   | `SparkBatchQueryScan`       | spark/source | `toBatch()`                                 |
| Physical batch | `SparkBatch`                | spark/source | `planInputPartitions()`                     |
| Partition      | `SparkInputPartition`       | spark/source | serialized task group                       |
| Reader factory | `SparkRowReaderFactory`     | spark/source | `createReader()`                            |
| Row reader     | `RowDataReader`             | spark/source | `open(FileScanTask)`                        |
| Base reader    | `BaseReader`                | spark/source | task iteration loop (`next()`)              |
| File open      | `BaseRowReader`             | spark/source | `newIterable()` (format-agnostic dispatch)  |
| Iceberg scan   | `DataTableScan`             | core         | `planFiles()`                               |
| Manifest eval  | `ManifestGroup`             | core         | `planFiles()`                               |
| Manifest read  | `ManifestReader`            | core         | reads manifest Avro files                   |
| Stats eval     | `InclusiveMetricsEvaluator` | api          | `eval(DataFile)`                            |
| Format lookup  | `FormatModelRegistry`       | core/formats | `readBuilder(format, type, file)`           |
| Spark formats  | `SparkFormatModels`         | spark/source | `register()` (one-time at startup)          |
| Parquet read   | `ParquetReader`             | parquet      | `iterator()` → `FileIterator`               |
| Value decode   | `SparkParquetReaders`       | spark/data   | `buildReader()` (registered via FormatModel) |
| Delete filter  | `SparkDeleteFilter`         | spark/source | inner class of `BaseReader`; `filter()`     |

---

## 4. Filter Pushdown Pipeline

```
Spark Expression (SQL WHERE clause)
       │
       ▼
SparkScanBuilder.pushPredicates(Predicate[])
       │
       ├─► convert Spark Predicate → Iceberg Expression
       │
       ▼
DataTableScan.filter(icebergExpression)
       │
       ├─► ManifestEvaluator (partition-level pruning)
       │     skip entire manifests based on partition summaries
       │
       ├─► InclusiveMetricsEvaluator (file-level pruning)
       │     skip files based on column min/max stats
       │
       ├─► Parquet RowGroup filter (row-group-level pruning)
       │     skip row groups based on Parquet column stats
       │
       └─► ResidualEvaluator (row-level filtering)
             residual filter applied row-by-row in reader
```
