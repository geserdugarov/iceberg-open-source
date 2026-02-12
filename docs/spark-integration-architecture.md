# Apache Iceberg - Spark Integration Architecture

This document describes the internal architecture of the Iceberg-Spark integration, including key components, data flows, and design patterns.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Catalog Integration](#catalog-integration)
3. [Table Representation](#table-representation)
4. [Read Path Architecture](#read-path-architecture)
5. [Write Path Architecture](#write-path-architecture)
6. [SQL Extensions](#sql-extensions)
7. [Stored Procedures](#stored-procedures)
8. [Functions](#functions)
9. [Actions](#actions)
10. [Design Patterns](#design-patterns)

## Architecture Overview

The Iceberg-Spark integration implements Spark's DataSource V2 API to provide native table format support. The integration consists of several layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Spark Application                          │
├─────────────────────────────────────────────────────────────────┤
│                    Spark SQL / DataFrame API                    │
├─────────────────────────────────────────────────────────────────┤
│  SQL Extensions    │   Catalog API    │    DataSource V2 API    │
│  (Parser/Analyzer) │  (TableCatalog)  │   (Table/Scan/Write)    │
├─────────────────────────────────────────────────────────────────┤
│                 Iceberg Spark Integration Layer                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ SparkCatalog │  │ SparkTable   │  │ SparkScan/SparkWrite │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      Iceberg Core Library                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Catalog    │  │    Table     │  │  Scan/Transaction    │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                    File I/O (Parquet/ORC/Avro)                  │
└─────────────────────────────────────────────────────────────────┘
```

## Catalog Integration

### SparkCatalog

`SparkCatalog` implements Spark's `TableCatalog` interface, wrapping an Iceberg `Catalog`.

**Key Responsibilities:**
- Table CRUD operations (create, load, alter, drop)
- Namespace management
- Table caching
- Metadata selector parsing (time travel, branches, tags)

**Class Hierarchy:**
```
TableCatalog (Spark interface)
    ↑
BaseCatalog (Iceberg abstract class)
    ↑
SparkCatalog (Iceberg implementation)
```

**Table Loading Flow:**
```
loadTable(identifier)
    │
    ├─→ Check if PathIdentifier (file path reference)
    │       └─→ HadoopTables.load(path)
    │
    └─→ Parse metadata selectors from identifier:
            • at_timestamp_<ms>
            • snapshot_id_<id>
            • branch_<name>
            • tag_<name>
            • changelog
            • rewrite
                │
                └─→ Return SparkTable | SparkChangelogTable
```

### SparkSessionCatalog

`SparkSessionCatalog<T>` extends Spark's `CatalogExtension` to support both Iceberg and non-Iceberg tables within the same catalog namespace.

**Integration Flow:**
```
SparkSession
    │
    ↓
CatalogManager
    │
    ↓
SparkSessionCatalog (CatalogExtension)
    ├─→ SparkCatalog (Iceberg tables)
    └─→ SessionCatalog (Non-Iceberg tables)
```

### Key Classes

| Class | Location | Purpose |
|-------|----------|---------|
| `SparkCatalog` | `spark/SparkCatalog.java` | Main Iceberg catalog implementation |
| `SparkSessionCatalog` | `spark/SparkSessionCatalog.java` | Session catalog extension |
| `BaseCatalog` | `spark/BaseCatalog.java` | Abstract base with common functionality |
| `SparkCachedTableCatalog` | `spark/SparkCachedTableCatalog.java` | Caching wrapper |
| `PathIdentifier` | `spark/PathIdentifier.java` | Path-based table loading |

## Table Representation

### SparkTable

`SparkTable` is the primary implementation of Spark's `Table` interface for Iceberg tables.

**Supported Capabilities:**
- `SupportsRead` - Batch and streaming reads
- `SupportsWrite` - Batch and streaming writes
- `SupportsDeleteV2` - DELETE operations
- `SupportsRowLevelOperations` - UPDATE, DELETE, MERGE
- `SupportsMetadataColumns` - Access to metadata columns

**Table Variants:**

| Class | Purpose |
|-------|---------|
| `SparkTable` | Standard Iceberg table |
| `SparkChangelogTable` | Changelog/CDC reading |
| `StagedSparkTable` | Transactional staging |
| `SparkView` | Iceberg views |
| `RollbackStagedTable` | Failed staging rollback |

## Read Path Architecture

The read path implements Spark's DataSource V2 scanning interfaces.

### Component Flow

```
IcebergSource (DataSourceRegister)
    │
    ↓ format("iceberg")
SparkTable.newScanBuilder()
    │
    ↓
SparkScanBuilder (ScanBuilder)
    │ - Applies filters (pushdown)
    │ - Handles column pruning
    │ - Configures scan options
    ↓
┌─────────────────────────────────────────────┐
│           Scan Type Selection               │
├─────────────────────────────────────────────┤
│ SparkBatchQueryScan   │ Standard queries    │
│ SparkChangelogScan    │ Incremental/CDC     │
│ SparkStagedScan       │ Uncommitted data    │
│ SparkCopyOnWriteScan  │ CoW optimization    │
└─────────────────────────────────────────────┘
    │
    ↓
SparkBatch (Batch)
    │
    ↓
┌─────────────────────────────────────────────┐
│           Reader Selection                  │
├─────────────────────────────────────────────┤
│ BatchDataReader       │ Standard row reader │
│ ChangelogRowReader    │ Changelog reader    │
│ PositionDeletesReader │ Delete file reader  │
│ SparkColumnarReader   │ Vectorized reader   │
└─────────────────────────────────────────────┘
```

### Scan Planning Modes

Scan planning can be performed on the driver or distributed to executors:

```
SparkReadConf determines planning mode
    │
    ├─→ driverMaxResultSize < 256MB → LOCAL (driver-side)
    │
    └─→ else → Check DATA_PLANNING_MODE / DELETE_PLANNING_MODE
            │
            ├─→ LOCAL: Planning on driver
            └─→ DISTRIBUTED: Planning on executors
```

### Key Read Classes

| Class | Purpose |
|-------|---------|
| `SparkScanBuilder` | Builds scan with filters and projections |
| `SparkBatchQueryScan` | Standard batch query implementation |
| `SparkChangelogScan` | Incremental changelog scanning |
| `SparkPartitioningAwareScan` | Base for partitioning and locality |
| `SparkReadConf` | Read configuration management |
| `SparkFilters` | Filter expression conversion |

## Write Path Architecture

The write path implements Spark's DataSource V2 write interfaces.

### Component Flow

```
SparkTable.newWriteBuilder()
    │
    ↓
SparkWriteBuilder (WriteBuilder)
    │ - Configures distribution mode
    │ - Sets up sort order
    │ - Handles write options
    ↓
SparkWrite (Write)
    │
    ↓
┌─────────────────────────────────────────────┐
│           Write Type Selection              │
├─────────────────────────────────────────────┤
│ BatchWrite            │ Batch insert/append │
│ StreamingWrite        │ Streaming writes    │
│ PositionDeltaWrite    │ Position-based dels │
│ CopyOnWriteOperation  │ CoW mutations       │
└─────────────────────────────────────────────┘
    │
    ↓
SparkFileWriterFactory
    │
    ↓
DataWriter (per partition)
    │
    ↓
Commit (driver aggregates results)
```

### Distribution Modes

| Mode | Description |
|------|-------------|
| `NONE` | No distribution; writers handle all partitions |
| `HASH` | Hash partition by partition key (default) |
| `RANGE` | Range partition for sorted output |

### Row-Level Operations

For UPDATE, DELETE, and MERGE operations:

```
SparkRowLevelOperationBuilder
    │
    ├─→ Copy-on-Write Strategy
    │       └─→ SparkCopyOnWriteOperation
    │           └─→ Rewrites affected files completely
    │
    └─→ Merge-on-Read Strategy
            └─→ SparkPositionDeltaWrite
                └─→ Writes position delete files
```

### Key Write Classes

| Class | Purpose |
|-------|---------|
| `SparkWriteBuilder` | Builds write configuration |
| `SparkWrite` | Main write implementation |
| `SparkWriteConf` | Write configuration management |
| `SparkFileWriterFactory` | Creates file writers |
| `SparkPositionDeltaWrite` | Position delete handling |

## SQL Extensions

SQL extensions are implemented in Scala and inject custom parsing and planning into Spark.

### Extension Registration

```scala
// IcebergSparkSessionExtensions.scala
class IcebergSparkSessionExtensions extends (SparkSessionExtensions => Unit) {
  override def apply(extensions: SparkSessionExtensions): Unit = {
    // Parser injection
    extensions.injectParser { case (_, parser) =>
      new IcebergSparkSqlExtensionsParser(parser)
    }

    // Analyzer rules
    extensions.injectResolutionRule { spark => ResolveViews(spark) }
    extensions.injectCheckRule { spark => CheckViews(spark) }

    // Optimizer rules
    extensions.injectOptimizerRule { _ => ReplaceStaticInvoke }

    // Planner strategies
    extensions.injectPlannerStrategy { spark =>
      ExtendedDataSourceV2Strategy(spark)
    }
  }
}
```

### Custom DDL Operations

Logical plans in `catalyst/plans/logical/`:

**Partitioning:**
- `AddPartitionField` - Add new partition field
- `DropPartitionField` - Remove partition field
- `ReplacePartitionField` - Replace partition transform

**Versioning:**
- `CreateOrReplaceBranch` - Create/replace branch
- `DropBranch` - Delete branch
- `CreateOrReplaceTag` - Create/replace tag
- `DropTag` - Delete tag

**Views:**
- `CreateIcebergView` - Create Iceberg view
- `DropIcebergView` - Drop Iceberg view
- `ShowIcebergViews` - List views

**Identity Fields:**
- `SetIdentifierFields` - Set identifier columns
- `DropIdentifierFields` - Remove identifier columns

## Stored Procedures

Stored procedures provide SQL-accessible maintenance operations.

### Procedure Registration

Procedures are registered in `SparkCatalog` and accessible via:
```sql
CALL catalog.system.procedure_name(args...)
```

### Available Procedures

**Snapshot Management:**
| Procedure | Purpose |
|-----------|---------|
| `expire_snapshots` | Remove old snapshots |
| `rollback_to_snapshot` | Rollback to specific snapshot |
| `rollback_to_timestamp` | Rollback to timestamp |
| `cherrypick_snapshot` | Cherry-pick changes |
| `set_current_snapshot` | Set current snapshot |

**Data Optimization:**
| Procedure | Purpose |
|-----------|---------|
| `rewrite_data_files` | Compact/sort data files |
| `rewrite_position_delete_files` | Optimize position deletes |
| `rewrite_manifests` | Optimize manifest files |
| `compute_table_stats` | Compute statistics |

**Table Management:**
| Procedure | Purpose |
|-----------|---------|
| `migrate_table` | Migrate from other formats |
| `snapshot_table` | Create snapshot copy |
| `register_table` | Register existing table |
| `remove_orphan_files` | Clean orphaned files |
| `create_changelog_view` | Create CDC view |

**Branch Operations:**
| Procedure | Purpose |
|-----------|---------|
| `fast_forward_branch` | Fast-forward branch |
| `publish_changes` | Publish staged changes |

### Example Usage

```sql
-- Expire snapshots older than 7 days
CALL iceberg.system.expire_snapshots(
  table => 'db.sample',
  older_than => TIMESTAMP '2024-01-01 00:00:00',
  retain_last => 10
)

-- Rewrite data files with sorting
CALL iceberg.system.rewrite_data_files(
  table => 'db.sample',
  strategy => 'sort',
  sort_order => 'id ASC NULLS FIRST'
)

-- Remove orphan files
CALL iceberg.system.remove_orphan_files(
  table => 'db.sample',
  older_than => TIMESTAMP '2024-01-01 00:00:00',
  dry_run => true
)
```

## Functions

Iceberg provides custom SQL functions accessible in Spark SQL.

### Available Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `iceberg_version()` | Get Iceberg version | `SELECT iceberg_version()` |
| `bucket(n, col)` | Bucket transform | `bucket(16, id)` |
| `truncate(width, col)` | Truncate transform | `truncate(10, name)` |
| `years(col)` | Year transform | `years(ts)` |
| `months(col)` | Month transform | `months(ts)` |
| `days(col)` | Day transform | `days(ts)` |
| `hours(col)` | Hour transform | `hours(ts)` |

### Function Implementation

Functions implement Spark's `UnboundFunction` interface:

```java
public class BucketFunction implements UnboundFunction {
  @Override
  public BoundFunction bind(StructType inputType) {
    // Return bound function for specific input types
  }
}
```

## Actions

Actions provide programmatic access to maintenance operations, leveraging Spark's distributed computing.

### Available Actions

```java
SparkActions actions = SparkActions.get(spark);

// Rewrite data files
actions.rewriteDataFiles(table)
    .filter(Expressions.equal("partition_col", "value"))
    .option("target-file-size-bytes", "134217728")
    .execute();

// Expire snapshots
actions.expireSnapshots(table)
    .expireOlderThan(System.currentTimeMillis() - TimeUnit.DAYS.toMillis(7))
    .retainLast(10)
    .execute();

// Remove orphan files
actions.removeOrphanFiles(table)
    .olderThan(System.currentTimeMillis() - TimeUnit.DAYS.toMillis(1))
    .execute();
```

### Action Classes

| Class | Purpose |
|-------|---------|
| `RewriteDataFilesSparkAction` | Compact data files |
| `RewriteManifestsSparkAction` | Optimize manifests |
| `RewritePositionDeleteFilesSparkAction` | Compact delete files |
| `DeleteReachableFilesSparkAction` | Delete reachable files |
| `ComputeTableStatsSparkAction` | Compute statistics |
| `SnapshotTableSparkAction` | Create table snapshot |

## Design Patterns

### Adapter Pattern

`SparkCatalog` adapts Iceberg's `Catalog` to Spark's `TableCatalog`:

```
Spark TableCatalog ←── SparkCatalog ──→ Iceberg Catalog
```

### Decorator Pattern

`SparkSessionCatalog` decorates Spark's session catalog with Iceberg support:

```
SessionCatalog ←── SparkSessionCatalog ──→ SparkCatalog
```

### Builder Pattern

Scan and write operations use builders for flexible configuration:

```java
SparkScanBuilder builder = new SparkScanBuilder(spark, table, options)
    .filter(filter)
    .prune(schema);
Scan scan = builder.build();
```

### Strategy Pattern

Different scan and write strategies for various operations:

```
ScanStrategy
    ├── SparkBatchQueryScan (standard)
    ├── SparkChangelogScan (incremental)
    └── SparkCopyOnWriteScan (mutations)

WriteStrategy
    ├── SparkWrite (standard)
    ├── SparkPositionDeltaWrite (deletes)
    └── SparkCopyOnWriteOperation (mutations)
```

### Visitor Pattern

Schema traversal using visitors:

```java
public class SparkTypeVisitor<T> {
  T struct(StructType struct, List<T> fieldResults);
  T field(StructField field, T typeResult);
  T list(ArrayType array, T elementResult);
  T map(MapType map, T keyResult, T valueResult);
  T primitive(DataType primitive);
}
```

### Factory Pattern

File writer creation uses factories:

```java
SparkFileWriterFactory factory = SparkFileWriterFactory.builderFor(table)
    .dataFileFormat(format)
    .dataSchema(schema)
    .build();

DataWriter<InternalRow> writer = factory.newDataWriter(partition, taskId);
```
