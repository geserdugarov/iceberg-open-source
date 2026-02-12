# Iceberg Spark DataSource V2 Read Path Blueprint

This document provides a comprehensive guide to Apache Iceberg's Spark DataSource V2 (DSv2) read implementation.

## Table of Contents

1. [End-to-End Flow Overview](#end-to-end-flow-overview)
2. [Key Classes by Spark Version](#key-classes-by-spark-version)
3. [Pushdowns Implementation](#pushdowns-implementation)
4. [Planning Logic](#planning-logic)
5. [Test Pointers](#test-pointers)

---

## End-to-End Flow Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SPARK DRIVER SIDE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  IcebergSource.getTable()                                                   │
│       │                                                                     │
│       ▼                                                                     │
│  SparkTable (implements SupportsRead)                                       │
│       │                                                                     │
│       │ newScanBuilder(options)                                             │
│       ▼                                                                     │
│  SparkScanBuilder                                                           │
│       │                                                                     │
│       ├── pushPredicates() ──► SparkV2Filters.convert() ──► Iceberg Expr   │
│       ├── pruneColumns() ────► SparkSchemaUtil.prune()                      │
│       ├── pushAggregation() ─► COUNT/MIN/MAX pushdown                       │
│       └── pushLimit() ───────► Limit optimization                           │
│       │                                                                     │
│       │ build()                                                             │
│       ▼                                                                     │
│  SparkBatchQueryScan (extends SparkPartitioningAwareScan)                   │
│       │                                                                     │
│       ├── tasks() ───────────► scan.planFiles() ──► FileScanTask[]         │
│       ├── taskGroups() ──────► TableScanUtil.planTaskGroups()              │
│       └── estimateStatistics() ► row count, size, column stats             │
│       │                                                                     │
│       │ toBatch()                                                           │
│       ▼                                                                     │
│  SparkBatch                                                                 │
│       │                                                                     │
│       ├── planInputPartitions() ─► SparkInputPartition[]                   │
│       │       └── Broadcasts table metadata                                 │
│       │       └── Computes preferred locations                              │
│       │                                                                     │
│       └── createReaderFactory() ─► SparkColumnarReaderFactory              │
│                                    or SparkRowReaderFactory                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          SPARK EXECUTOR SIDE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SparkInputPartition (deserialized on executor)                             │
│       │                                                                     │
│       │ via PartitionReaderFactory                                          │
│       ▼                                                                     │
│  PartitionReader                                                            │
│       │                                                                     │
│       ├── RowDataReader ─────────► InternalRow iteration                   │
│       │       └── Opens InputFile                                           │
│       │       └── Applies SparkDeleteFilter                                 │
│       │       └── Applies residual filter                                   │
│       │                                                                     │
│       └── BatchDataReader ───────► ColumnarBatch iteration                 │
│               └── Vectorized Parquet/ORC reads                              │
│               └── Comet acceleration (if enabled)                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Flow Steps

1. **TableProvider Entry** - `IcebergSource.getTable()` returns a `SparkTable`
2. **Scan Builder Creation** - `SparkTable.newScanBuilder()` creates `SparkScanBuilder`
3. **Pushdown Application** - Spark calls pushdown methods on the builder
4. **Scan Object Creation** - `build()` creates `SparkBatchQueryScan`
5. **Task Planning** - `tasks()` and `taskGroups()` plan the scan
6. **Batch Creation** - `toBatch()` creates `SparkBatch`
7. **Partition Planning** - `planInputPartitions()` creates executor work units
8. **Reader Factory** - `createReaderFactory()` selects row vs columnar reader
9. **Executor Reading** - Readers iterate files and apply filters

---

## Key Classes by Spark Version

All classes follow the path pattern:
```
spark/v{version}/spark/src/main/java/org/apache/iceberg/spark/source/{ClassName}.java
```

### Entry Point: IcebergSource

| Version | Path |
|---------|------|
| 3.4 | `spark/v3.4/spark/src/main/java/org/apache/iceberg/spark/source/IcebergSource.java` |
| 3.5 | `spark/v3.5/spark/src/main/java/org/apache/iceberg/spark/source/IcebergSource.java` |
| 4.0 | `spark/v4.0/spark/src/main/java/org/apache/iceberg/spark/source/IcebergSource.java` |
| 4.1 | `spark/v4.1/spark/src/main/java/org/apache/iceberg/spark/source/IcebergSource.java` |

**Purpose:** DataSource V2 registration point. Implements `DataSourceRegister`, `SupportsCatalogOptions`, `SessionConfigSupport`.

**Key Methods:**
- `shortName()` - Returns "iceberg"
- `inferSchema()` - Returns null (schema from table)
- `getTable()` - Returns SparkTable instance

---

### SparkTable

**Purpose:** Iceberg table wrapper implementing Spark's Table interface.

**Implements:**
- `org.apache.spark.sql.connector.catalog.Table`
- `SupportsRead` (primary read interface)
- `SupportsWrite`
- `SupportsDeleteV2`
- `SupportsRowLevelOperations`
- `SupportsMetadataColumns`

**Key Fields:**
```java
Table icebergTable;       // Underlying Iceberg table
Long snapshotId;          // Optional snapshot for time-travel
String branch;            // Branch reference
boolean refreshEagerly;   // Eager metadata refresh
```

**Key Methods:**
- `newScanBuilder(CaseInsensitiveStringMap options)` - Creates SparkScanBuilder
- `schema()` - Returns Spark StructType
- `partitioning()` - Returns Spark partitioning

---

### SparkScanBuilder

**Purpose:** Builds scan with all pushdowns applied.

**Implements:**
- `ScanBuilder`
- `SupportsPushDownV2Filters` - Filter pushdown
- `SupportsPushDownRequiredColumns` - Column pruning
- `SupportsPushDownAggregates` - Aggregate pushdown
- `SupportsReportStatistics` - Statistics reporting
- `SupportsPushDownLimit` - Limit pushdown

**Key Fields:**
```java
SparkSession spark;
Table table;
Schema schema;                          // Current (pruned) schema
SparkReadConf readConf;
List<Expression> filterExpressions;     // Pushed Iceberg expressions
Predicate[] pushedPredicates;           // Successfully pushed predicates
Set<String> metaColumns;                // Requested metadata columns
```

**Key Methods:**
- `pushPredicates(Predicate[])` - Converts and pushes filters
- `pruneColumns(StructType)` - Prunes schema to required columns
- `pushAggregation(Aggregation)` - Pushes COUNT/MIN/MAX
- `pushLimit(int)` - Sets row limit
- `build()` - Creates SparkBatchQueryScan

---

### SparkBatchQueryScan

**Purpose:** Represents the planned scan with all tasks.

**Extends:** `SparkPartitioningAwareScan<PartitionScanTask>`

**Implements:** `SupportsRuntimeV2Filtering` - Runtime filter application

**Key Methods:**
- `tasks()` - Returns list of FileScanTasks (lazy)
- `taskGroups()` - Returns grouped ScanTaskGroups (lazy)
- `estimateStatistics()` - Returns Statistics for CBO
- `filter(Predicate[])` - Applies runtime filters from broadcast joins
- `toBatch()` - Creates SparkBatch

---

### SparkBatch

**Purpose:** Batch execution coordinator.

**Implements:** `Batch`

**Key Methods:**
- `planInputPartitions()` - Creates InputPartition array
  - Broadcasts table metadata
  - Computes preferred locations (block or executor locality)
  - Creates SparkInputPartition per task group
- `createReaderFactory()` - Returns appropriate reader factory:
  - `SparkColumnarReaderFactory` for Parquet/ORC vectorized
  - `SparkRowReaderFactory` for row-level reads

---

### SparkInputPartition

**Purpose:** Serializable work unit sent to executors.

**Implements:**
- `InputPartition`
- `HasPartitionKey` - For partition key reporting

**Key Fields:**
```java
ScanTaskGroup<T> taskGroup;           // Tasks for this partition
Broadcast<Table> tableBroadcast;      // Broadcasted table
String expectedSchemaString;          // JSON schema
String[] preferredLocations;          // Location hints
```

**Key Methods:**
- `preferredLocations()` - Returns location hints
- `partitionKey()` - Returns grouping key value
- `taskGroup()` - Returns task group
- `table()` - Dereferences broadcast

---

### Reader Factories

#### SparkRowReaderFactory

**Purpose:** Creates row-level readers.

**Methods:**
- `createReader()` - Creates RowDataReader
- `supportColumnarReads()` - Returns false

#### SparkColumnarReaderFactory

**Purpose:** Creates vectorized columnar readers.

**Constructor Variants:**
- `SparkColumnarReaderFactory(ParquetBatchReadConf)` - Parquet
- `SparkColumnarReaderFactory(OrcBatchReadConf)` - ORC

**Methods:**
- `createColumnarReader()` - Creates BatchDataReader
- `supportColumnarReads()` - Returns true

---

### Readers

#### RowDataReader

**Extends:** `BaseRowReader<FileScanTask>`
**Implements:** `PartitionReader<InternalRow>`

**Key Method:**
```java
protected CloseableIterator<InternalRow> open(FileScanTask task)
```
- Opens InputFile via newStream()
- Creates SparkDeleteFilter for delete handling
- Sets InputFileBlockHolder for filename() function
- Returns filtered InternalRow iterator

#### BatchDataReader

**Extends:** `BaseBatchReader`
**Implements:** `PartitionReader<ColumnarBatch>`

- Handles vectorized Parquet (Iceberg/Comet) and ORC
- Manages batch allocation and columnar layout
- Applies delete filters at columnar level

---

### Supporting Classes

#### SparkV2Filters
**Path:** `spark/v{version}/spark/src/main/java/org/apache/iceberg/spark/SparkV2Filters.java`

Converts Spark predicates to Iceberg expressions:
| Spark | Iceberg |
|-------|---------|
| `=`, `<=>` | `EQ` |
| `<>` | `NOT_EQ` |
| `<`, `<=`, `>`, `>=` | Comparisons |
| `IN`, `NOT IN` | `IN`, `NOT_IN` |
| `IS NULL`, `IS NOT NULL` | `IS_NULL`, `NOT_NULL` |
| `STARTS_WITH` | `STARTS_WITH` |
| `AND`, `OR`, `NOT` | Logical ops |

Supported transform functions: `years`, `months`, `days`, `hours`, `bucket`, `truncate`

#### SparkReadConf
**Path:** `spark/v{version}/spark/src/main/java/org/apache/iceberg/spark/SparkReadConf.java`

Configuration precedence: Read options > Session config > Table metadata

Key methods:
- `branch()`, `snapshotId()`, `asOfTimestamp()` - Time travel
- `localityEnabled()`, `executorCacheLocalityEnabled()` - Locality
- `parquetVectorizationEnabled()`, `orcVectorizationEnabled()` - Vectorization
- `parquetReaderType()` - ICEBERG, COMET, or NATIVE
- `splitSize()`, `splitLookback()` - Split planning
- `preserveDataGrouping()` - Partition key grouping
- `distributedPlanningEnabled()` - Distributed planning

---

## Pushdowns Implementation

### Filter Pushdown

**Location:** `SparkScanBuilder.pushPredicates()` (lines ~151-195)

**3-Way Filter Classification:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Filter Classification                           │
├─────────────────┬───────────────────────┬───────────────────────────┤
│ Type            │ Evaluated By          │ Example                   │
├─────────────────┼───────────────────────┼───────────────────────────┤
│ Partition-level │ Iceberg only          │ date = '2024-01-01'       │
│                 │ (file pruning)        │ (where date is partition) │
├─────────────────┼───────────────────────┼───────────────────────────┤
│ Record-level    │ Iceberg + Spark       │ status = 'active'         │
│                 │ (residual evaluation) │ (data column with stats)  │
├─────────────────┼───────────────────────┼───────────────────────────┤
│ Unsupported     │ Spark only            │ udf(col) > 10             │
│                 │ (post-scan)           │ (complex expressions)     │
└─────────────────┴───────────────────────┴───────────────────────────┘
```

**Flow:**
1. Spark calls `pushPredicates(Predicate[])`
2. Each predicate converted via `SparkV2Filters.convert()`
3. `ExpressionUtil.selectsPartitions()` checks if filter selects entire partitions
4. Partition-level filters stored in `filterExpressions`
5. Record-level filters returned for Spark to evaluate post-scan

### Column Pruning

**Location:** `SparkScanBuilder.pruneColumns()` (lines ~329-346)

**Flow:**
1. Spark calls `pruneColumns(StructType requestedSchema)`
2. Metadata columns extracted separately
3. `SparkSchemaUtil.prune()` determines minimal schema
4. Schema includes columns needed for filters
5. Pruned schema stored for scan planning

### Statistics Reporting

**Location:** `SparkScan.estimateStatistics()` (lines ~183-252)

**Statistics Provided:**
- Row count (from snapshot summary or task counts)
- Size in bytes
- Column statistics (NDV from puffin files if CBO enabled)

### Aggregation Pushdown

**Location:** `SparkScanBuilder.pushAggregation()` (lines ~207-271)

**Supported Aggregates:**
- `COUNT(*)`
- `COUNT(column)`
- `MIN(column)`
- `MAX(column)`

**Requirements:**
- No delete files present
- Metrics mode supports aggregation
- Creates `SparkLocalScan` for result

### Limit Pushdown

**Location:** `SparkScanBuilder.pushLimit()`

Sets minimum rows needed, optimizes file planning.

---

## Planning Logic

### How Iceberg Turns Table State + Filters into File/Scan Tasks

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PLANNING PIPELINE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DataTableScan.planFiles()                                                  │
│       │                                                                     │
│       ▼                                                                     │
│  ManifestGroup (core/src/.../ManifestGroup.java)                           │
│       │                                                                     │
│       ├── ManifestEvaluator ──► Partition stats pruning                    │
│       │       └── Uses ManifestFile partition field summaries              │
│       │       └── Skips manifests that cannot match                        │
│       │                                                                     │
│       ├── For each manifest entry:                                          │
│       │   ├── File Evaluator ──► DataFile struct filtering                 │
│       │   ├── InclusiveMetricsEvaluator ──► Column stats pruning           │
│       │   │       └── Uses min/max/null counts per column                  │
│       │   └── StrictMetricsEvaluator ──► Guaranteed matches                │
│       │                                                                     │
│       ├── DeleteFileIndex ──► Associates delete files                      │
│       │       └── Partition-keyed position/equality deletes                │
│       │       └── Global deletes                                           │
│       │                                                                     │
│       └── createFileScanTasks() ──► BaseFileScanTask[]                     │
│               └── Includes residual evaluator per partition                 │
│               └── Includes associated delete files                          │
│                                                                             │
│       │                                                                     │
│       ▼                                                                     │
│  TableScanUtil.planTaskGroups() (core/src/.../util/TableScanUtil.java)     │
│       │                                                                     │
│       ├── Split files by targetSplitSize                                    │
│       │                                                                     │
│       └── Bin-pack splits into CombinedScanTask                            │
│               └── Weight function considers file size + delete files       │
│               └── Uses splitLookback for optimal packing                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Core Planning Classes

#### ManifestGroup
**Path:** `core/src/main/java/org/apache/iceberg/ManifestGroup.java`

**Configuration:**
```java
manifestGroup
  .schemasById(Map<Integer, Schema>)
  .specsById(Map<Integer, PartitionSpec>)
  .filterData(Expression)         // Row filter
  .filterFiles(Expression)        // DataFile struct filter
  .filterPartitions(Expression)   // Partition filter
  .caseSensitive(boolean)
  .planWith(ExecutorService)      // Parallel planning
```

**Key Methods:**
- `planFiles()` - Returns `CloseableIterable<FileScanTask>`
- `entries()` - Returns raw manifest entries

#### ResidualEvaluator
**Path:** `api/src/main/java/org/apache/iceberg/expressions/ResidualEvaluator.java`

Computes remaining filter after partition evaluation:
```
Filter: ts >= a AND ts <= b
Partition: day(ts) = d

If day(a) < d < day(b) → residual = TRUE (all rows match)
If d == day(a)         → residual = ts >= a
If d == day(b)         → residual = ts <= b
```

#### ManifestEvaluator
**Path:** `api/src/main/java/org/apache/iceberg/expressions/ManifestEvaluator.java`

Evaluates filter against manifest partition statistics:
- Returns `true` if manifest MAY contain matches
- Returns `false` if manifest CANNOT contain matches
- Uses partition field summary (min/max/containsNull)

#### InclusiveMetricsEvaluator
**Path:** `api/src/main/java/org/apache/iceberg/expressions/InclusiveMetricsEvaluator.java`

File-level pruning using column statistics:
- Returns `true` if file MAY contain matches
- Uses column min/max bounds, null counts, NaN counts

#### StrictMetricsEvaluator
**Path:** `api/src/main/java/org/apache/iceberg/expressions/StrictMetricsEvaluator.java`

Strict file-level evaluation:
- Returns `true` only if ALL rows in file match
- Used for guaranteed filtering

#### DeleteFileIndex
**Path:** `core/src/main/java/org/apache/iceberg/DeleteFileIndex.java`

Associates delete files with data files:
- Global equality deletes
- Partition-specific equality deletes
- Position deletes by partition
- Position deletes by path
- Deletion vectors

---

## Test Pointers

### Scan Planning Tests

| Test File | Location | Coverage |
|-----------|----------|----------|
| `ScanTestBase.java` | `spark/v{version}/spark/src/test/.../source/` | Base scan infrastructure |
| `SparkDistributedDataScanTestBase.java` | `spark/v{version}/spark/src/test/.../source/` | Distributed planning |
| `TestSparkDistributedDataScanFilterFiles.java` | `spark/v{version}/spark/src/test/.../source/` | File filtering in distributed scans |
| `TestScanTaskSerialization.java` | `spark/v{version}/spark/src/test/.../source/` | Task serialization |

### Filter Pushdown Tests

| Test File | Location | Coverage |
|-----------|----------|----------|
| `TestSparkV2Filters.java` | `spark/v{version}/spark/src/test/.../spark/` | V2 filter conversion |
| `TestSparkFilters.java` | `spark/v{version}/spark/src/test/.../spark/` | Filter to expression conversion |
| `TestFilterPushDown.java` | `spark/v{version}/spark/src/test/.../sql/` | SQL-level pushdown |
| `TestFilteredScan.java` | `spark/v{version}/spark/src/test/.../source/` | Filter application during scan |
| `TestRuntimeFiltering.java` | `spark/v{version}/spark/src/test/.../source/` | Runtime filter optimization |
| `TestSparkReaderWithBloomFilter.java` | `spark/v{version}/spark/src/test/.../source/` | Bloom filter reads |

### Column Pruning Tests

| Test File | Location | Coverage |
|-----------|----------|----------|
| `TestReadProjection.java` | `spark/v{version}/spark/src/test/.../source/` | Column projection |
| `TestSparkReadProjection.java` | `spark/v{version}/spark/src/test/.../source/` | Read-time projection |
| `TestSparkMetadataColumns.java` | `spark/v{version}/spark/src/test/.../source/` | Metadata columns |
| `TestSparkParquetReadMetadataColumns.java` | `spark/v{version}/spark/src/test/.../source/` | Parquet metadata columns |
| `TestSparkOrcReadMetadataColumns.java` | `spark/v{version}/spark/src/test/.../source/` | ORC metadata columns |

### Schema Evolution Tests

| Test File | Location | Coverage |
|-----------|----------|----------|
| `TestSparkScan.java` | `spark/v{version}/spark/src/test/.../source/` | Schema evolution scenarios |
| `TestSparkSchemaUtil.java` | `spark/v{version}/spark/src/test/.../spark/` | Schema utilities |

### Partition Pruning Tests

| Test File | Location | Coverage |
|-----------|----------|----------|
| `TestPartitionPruning.java` | `spark/v{version}/spark/src/test/.../source/` | Partition elimination |
| `TestIdentityPartitionData.java` | `spark/v{version}/spark/src/test/.../source/` | Identity partition handling |
| `TestMetadataTablesWithPartitionEvolution.java` | `spark/v{version}/spark/src/test/.../source/` | Partition evolution |
| `TestPartitionValues.java` | `spark/v{version}/spark/src/test/.../source/` | Partition value extraction |

### Batch Reader Tests

| Test File | Location | Coverage |
|-----------|----------|----------|
| `TestParquetScan.java` | `spark/v{version}/spark/src/test/.../source/` | Parquet scans |
| `TestParquetVectorizedScan.java` | `spark/v{version}/spark/src/test/.../source/` | Vectorized Parquet |
| `TestParquetCometVectorizedScan.java` | `spark/v{version}/spark/src/test/.../source/` | Comet acceleration |
| `TestAvroScan.java` | `spark/v{version}/spark/src/test/.../source/` | Avro scans |
| `TestBaseReader.java` | `spark/v{version}/spark/src/test/.../source/` | Base reader implementation |
| `TestParquetVectorizedReads.java` | `spark/v{version}/spark/src/test/.../data/vectorized/parquet/` | Vectorized read details |

### Delete Handling Tests

| Test File | Location | Coverage |
|-----------|----------|----------|
| `TestSparkReaderDeletes.java` | `spark/v{version}/spark/src/test/.../source/` | Delete vectors in reads |
| `TestPositionDeletesTable.java` | `spark/v{version}/spark/src/test/.../source/` | Position delete handling |
| `TestPositionDeletesReader.java` | `spark/v{version}/spark/src/test/.../source/` | Position delete reading |

---

## Quick Reference: File Paths

### Spark Module (v3.4/v3.5/v4.0/v4.1)

```
spark/v{version}/spark/src/main/java/org/apache/iceberg/spark/
├── source/
│   ├── IcebergSource.java           # DataSource entry point
│   ├── SparkTable.java              # Table implementation
│   ├── SparkScanBuilder.java        # Scan builder with pushdowns
│   ├── SparkScan.java               # Base scan class
│   ├── SparkBatchQueryScan.java     # Batch query scan
│   ├── SparkPartitioningAwareScan.java # Task planning
│   ├── SparkBatch.java              # Batch coordinator
│   ├── SparkInputPartition.java     # Executor work unit
│   ├── SparkRowReaderFactory.java   # Row reader factory
│   ├── SparkColumnarReaderFactory.java # Columnar reader factory
│   ├── RowDataReader.java           # Row-level reader
│   ├── BatchDataReader.java         # Batch reader
│   └── Stats.java                   # Statistics implementation
├── SparkV2Filters.java              # Filter conversion
├── SparkReadConf.java               # Read configuration
└── SparkSchemaUtil.java             # Schema utilities
```

### Core Module

```
core/src/main/java/org/apache/iceberg/
├── DataTableScan.java               # Main table scan
├── BaseTableScan.java               # Base implementation
├── SnapshotScan.java                # Snapshot-based scan
├── BaseScan.java                    # Scan base class
├── ManifestGroup.java               # Manifest processing
├── DeleteFileIndex.java             # Delete file tracking
├── BaseFileScanTask.java            # File scan task
├── BaseCombinedScanTask.java        # Combined tasks
└── util/
    └── TableScanUtil.java           # Planning utilities
```

### API Module (Expressions)

```
api/src/main/java/org/apache/iceberg/expressions/
├── Expression.java                  # Expression interface
├── Expressions.java                 # Expression factory
├── Binder.java                      # Expression binding
├── Evaluator.java                   # Row evaluation
├── ResidualEvaluator.java           # Residual computation
├── ManifestEvaluator.java           # Manifest pruning
├── InclusiveMetricsEvaluator.java   # Inclusive file pruning
├── StrictMetricsEvaluator.java      # Strict file pruning
└── Projections.java                 # Partition projection
```
