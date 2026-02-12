# Apache Iceberg - Spark Integration Configuration Reference

This document provides a comprehensive reference for all configuration options available when using Iceberg with Spark.

## Table of Contents

1. [Catalog Configuration](#catalog-configuration)
2. [Read Configuration](#read-configuration)
3. [Write Configuration](#write-configuration)
4. [Session Configuration](#session-configuration)
5. [Table Properties](#table-properties)
6. [Configuration Precedence](#configuration-precedence)

## Catalog Configuration

Configure Iceberg catalogs in Spark using `spark.sql.catalog.<catalog-name>.*` properties.

### Basic Catalog Setup

```properties
# Register an Iceberg catalog
spark.sql.catalog.iceberg=org.apache.iceberg.spark.SparkCatalog

# Catalog type (required)
spark.sql.catalog.iceberg.type=hive|hadoop|rest|glue|jdbc|nessie

# Or use custom catalog implementation
spark.sql.catalog.iceberg.catalog-impl=org.example.CustomCatalog
```

### Catalog Types

| Type | Description | Required Properties |
|------|-------------|---------------------|
| `hive` | Hive Metastore catalog | `uri` (HMS Thrift URI) |
| `hadoop` | Hadoop-based catalog | `warehouse` (HDFS/S3 path) |
| `rest` | REST catalog | `uri` (REST endpoint) |
| `glue` | AWS Glue catalog | AWS credentials |
| `jdbc` | JDBC-based catalog | `uri`, `jdbc.user`, `jdbc.password` |
| `nessie` | Nessie catalog | `uri`, `ref` |

### Hive Metastore Catalog

```properties
spark.sql.catalog.hive_catalog=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.hive_catalog.type=hive
spark.sql.catalog.hive_catalog.uri=thrift://localhost:9083
spark.sql.catalog.hive_catalog.warehouse=s3://bucket/warehouse
```

### Hadoop Catalog

```properties
spark.sql.catalog.hadoop_catalog=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.hadoop_catalog.type=hadoop
spark.sql.catalog.hadoop_catalog.warehouse=hdfs://namenode:8020/warehouse
```

### REST Catalog

```properties
spark.sql.catalog.rest_catalog=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.rest_catalog.type=rest
spark.sql.catalog.rest_catalog.uri=https://catalog-server:8181
spark.sql.catalog.rest_catalog.credential=client_id:client_secret
spark.sql.catalog.rest_catalog.token=<bearer-token>
spark.sql.catalog.rest_catalog.oauth2-server-uri=https://auth-server/token
```

### Session Catalog Extension

To use Iceberg with Spark's default session catalog:

```properties
spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog
spark.sql.catalog.spark_catalog.type=hive
```

### Catalog Properties Reference

| Property | Description | Default |
|----------|-------------|---------|
| `type` | Catalog type (hive, hadoop, rest, etc.) | Required |
| `catalog-impl` | Custom catalog implementation class | - |
| `uri` | Catalog service URI | Required for most types |
| `warehouse` | Default warehouse location | - |
| `io-impl` | Custom FileIO implementation | Auto-detected |
| `default-namespace` | Default namespace for unqualified tables | - |
| `cache-enabled` | Enable table caching | `true` |
| `cache.case-sensitive` | Case-sensitive cache keys | `true` |
| `cache.expiration-interval-ms` | Cache TTL in milliseconds | `600000` (10 min) |

### Table Property Defaults and Overrides

Set default or enforced properties for all tables in a catalog:

```properties
# Default properties (can be overridden by table)
spark.sql.catalog.iceberg.table-default.write.format.default=parquet
spark.sql.catalog.iceberg.table-default.write.parquet.compression-codec=zstd

# Override properties (enforced, cannot be overridden)
spark.sql.catalog.iceberg.table-override.write.metadata.delete-after-commit.enabled=true
```

## Read Configuration

Read options can be specified via DataFrame API or session configuration.

### DataFrame Read Options

```scala
spark.read
  .format("iceberg")
  .option("option-name", "value")
  .load("catalog.db.table")
```

### Time Travel Options

| Option | Description | Example |
|--------|-------------|---------|
| `snapshot-id` | Read specific snapshot | `123456789` |
| `as-of-timestamp` | Read as of timestamp (ms since epoch) | `1704067200000` |
| `branch` | Read from branch | `main`, `dev` |
| `tag` | Read from tag | `v1.0`, `release-2024` |
| `start-snapshot-id` | Start snapshot for incremental reads | `123456789` |
| `end-snapshot-id` | End snapshot for incremental reads | `987654321` |

### Scan Options

| Option | Description | Default |
|--------|-------------|---------|
| `split-size` | Target split size in bytes | `134217728` (128MB) |
| `lookback` | Split lookback for planning | `10` |
| `file-open-cost` | Cost for opening a file | `4194304` (4MB) |
| `vectorization-enabled` | Enable vectorized reads | `true` (Parquet) |
| `batch-size` | Batch size for vectorized reads | `4096` |
| `locality` | Report data locality to Spark | `false` |

### Filter and Projection Options

| Option | Description | Default |
|--------|-------------|---------|
| `aggregate-push-down-enabled` | Push down COUNT/MIN/MAX | `true` |

### Example: Time Travel Read

```scala
// Read specific snapshot
val df1 = spark.read
  .format("iceberg")
  .option("snapshot-id", 123456789L)
  .load("iceberg.db.table")

// Read as of timestamp
val df2 = spark.read
  .format("iceberg")
  .option("as-of-timestamp", "1704067200000")
  .load("iceberg.db.table")

// Read from branch
val df3 = spark.read
  .format("iceberg")
  .option("branch", "dev")
  .load("iceberg.db.table")

// Incremental read
val df4 = spark.read
  .format("iceberg")
  .option("start-snapshot-id", 123456789L)
  .option("end-snapshot-id", 987654321L)
  .load("iceberg.db.table")
```

### SQL Time Travel

```sql
-- Version as of snapshot ID
SELECT * FROM iceberg.db.table VERSION AS OF 123456789;

-- Timestamp as of
SELECT * FROM iceberg.db.table TIMESTAMP AS OF '2024-01-01 00:00:00';

-- Using FOR SYSTEM_TIME AS OF
SELECT * FROM iceberg.db.table FOR SYSTEM_TIME AS OF '2024-01-01 00:00:00';
```

## Write Configuration

Write options control how data is written to Iceberg tables.

### DataFrame Write Options

```scala
df.writeTo("iceberg.db.table")
  .option("option-name", "value")
  .append()
```

### Distribution Options

| Option | Description | Default |
|--------|-------------|---------|
| `distribution-mode` | Data distribution strategy | `hash` |
| `fanout-enabled` | Enable fanout writes | `false` |

**Distribution Modes:**
- `none` - No distribution; each writer handles all partitions
- `hash` - Hash distribute by partition key (default)
- `range` - Range distribute for globally sorted output

### Sort Options

| Option | Description | Example |
|--------|-------------|---------|
| `sort-order` | Sort order for writes | `id ASC, ts DESC` |

### File Format Options

| Option | Description | Default |
|--------|-------------|---------|
| `write-format` | Output file format | `parquet` |
| `target-file-size-bytes` | Target file size | `536870912` (512MB) |

### Parquet Options

| Option | Description | Default |
|--------|-------------|---------|
| `parquet-compression` | Compression codec | `zstd` |
| `parquet-compression-level` | Compression level | codec default |
| `parquet-row-group-size-bytes` | Row group size | `134217728` (128MB) |
| `parquet-page-size-bytes` | Page size | `1048576` (1MB) |
| `parquet-dict-size-bytes` | Dictionary page size | `2097152` (2MB) |

### ORC Options

| Option | Description | Default |
|--------|-------------|---------|
| `orc-compression` | Compression codec | `zstd` |
| `orc-stripe-size-bytes` | Stripe size | `67108864` (64MB) |
| `orc-block-size-bytes` | Block size | `268435456` (256MB) |

### Branch/Tag Options

| Option | Description |
|--------|-------------|
| `branch` | Write to specific branch |

### Example: Write with Options

```scala
// Write with custom distribution and sorting
df.writeTo("iceberg.db.table")
  .option("distribution-mode", "range")
  .option("sort-order", "id ASC NULLS FIRST, ts DESC")
  .option("target-file-size-bytes", "268435456")
  .option("parquet-compression", "zstd")
  .append()

// Write to branch
df.writeTo("iceberg.db.table")
  .option("branch", "feature-branch")
  .append()
```

## Session Configuration

Session-level configuration using `spark.sql.iceberg.*` properties.

### Vectorization

```properties
# Enable/disable vectorized reads
spark.sql.iceberg.vectorization.enabled=true

# Parquet reader type: iceberg (default) or native
spark.sql.iceberg.parquet.reader-type=iceberg
```

### Data Planning

```properties
# Planning mode: LOCAL or DISTRIBUTED
spark.sql.iceberg.data-planning-mode=LOCAL
spark.sql.iceberg.delete-planning-mode=LOCAL

# Preserve data grouping from shuffle
spark.sql.iceberg.planning.preserve-data-grouping=false
```

### Write Settings

```properties
# Default distribution mode
spark.sql.iceberg.distribution-mode=hash

# Check nullability during writes
spark.sql.iceberg.check-nullability=true

# Check field ordering during writes
spark.sql.iceberg.check-ordering=true

# Advisory partition size for clustering
spark.sql.iceberg.advisory-partition-size=134217728
```

### Push Down

```properties
# Aggregate push down (COUNT, MIN, MAX)
spark.sql.iceberg.aggregate-push-down.enabled=true
```

### Caching

```properties
# Enable executor-side caching
spark.sql.iceberg.executor-cache.enabled=true

# Cache delete files on executors
spark.sql.iceberg.executor-cache.delete-files.enabled=true

# Enable locality tracking in cache
spark.sql.iceberg.executor-cache.locality.enabled=true
```

### Write-Audit-Publish (WAP)

```properties
# WAP workflow ID
spark.wap.id=audit-2024-01-15

# WAP branch
spark.wap.branch=audit-branch
```

### All Session Properties Reference

| Property | Description | Default |
|----------|-------------|---------|
| `spark.sql.iceberg.vectorization.enabled` | Enable vectorized reads | `true` |
| `spark.sql.iceberg.parquet.reader-type` | Parquet reader (iceberg/native) | `iceberg` |
| `spark.sql.iceberg.distribution-mode` | Default write distribution | `hash` |
| `spark.sql.iceberg.check-nullability` | Check nullability on write | `true` |
| `spark.sql.iceberg.check-ordering` | Check field ordering on write | `true` |
| `spark.sql.iceberg.planning.preserve-data-grouping` | Preserve shuffle grouping | `false` |
| `spark.sql.iceberg.aggregate-push-down.enabled` | Push down aggregates | `true` |
| `spark.sql.iceberg.executor-cache.enabled` | Enable executor caching | `true` |
| `spark.sql.iceberg.data-planning-mode` | Scan planning mode | `LOCAL` |
| `spark.sql.iceberg.delete-planning-mode` | Delete planning mode | `LOCAL` |
| `spark.sql.iceberg.advisory-partition-size` | Clustering partition size | `134217728` |
| `spark.wap.id` | WAP workflow ID | - |
| `spark.wap.branch` | WAP branch name | - |

## Table Properties

Iceberg table properties that affect Spark behavior.

### Read Properties

| Property | Description | Default |
|----------|-------------|---------|
| `read.split.target-size` | Target split size | `134217728` |
| `read.split.metadata-target-size` | Metadata split size | `33554432` |
| `read.split.planning-lookback` | Planning lookback | `10` |
| `read.split.open-file-cost` | File open cost | `4194304` |
| `read.parquet.vectorization.enabled` | Vectorized Parquet reads | `true` |
| `read.parquet.vectorization.batch-size` | Vectorized batch size | `4096` |
| `read.orc.vectorization.enabled` | Vectorized ORC reads | `true` |
| `read.orc.vectorization.batch-size` | Vectorized batch size | `4096` |

### Write Properties

| Property | Description | Default |
|----------|-------------|---------|
| `write.format.default` | Default file format | `parquet` |
| `write.target-file-size-bytes` | Target file size | `536870912` |
| `write.parquet.compression-codec` | Parquet compression | `zstd` |
| `write.parquet.compression-level` | Compression level | codec default |
| `write.parquet.row-group-size-bytes` | Row group size | `134217728` |
| `write.parquet.page-size-bytes` | Page size | `1048576` |
| `write.parquet.dict-size-bytes` | Dictionary size | `2097152` |
| `write.orc.compression-codec` | ORC compression | `zstd` |
| `write.orc.stripe-size-bytes` | Stripe size | `67108864` |
| `write.orc.block-size-bytes` | Block size | `268435456` |

### Distribution Properties

| Property | Description | Default |
|----------|-------------|---------|
| `write.distribution-mode` | Write distribution mode | `hash` |
| `write.delete.distribution-mode` | Delete distribution mode | `hash` |
| `write.update.distribution-mode` | Update distribution mode | `hash` |
| `write.merge.distribution-mode` | Merge distribution mode | `hash` |

### Delete Properties

| Property | Description | Default |
|----------|-------------|---------|
| `write.delete.mode` | Delete mode (copy-on-write/merge-on-read) | `copy-on-write` |
| `write.update.mode` | Update mode | `copy-on-write` |
| `write.merge.mode` | Merge mode | `copy-on-write` |

### Metadata Properties

| Property | Description | Default |
|----------|-------------|---------|
| `write.metadata.compression-codec` | Metadata file compression | `gzip` |
| `write.metadata.delete-after-commit.enabled` | Delete old metadata | `false` |
| `write.metadata.previous-versions-max` | Max previous versions | `100` |

### Example: Setting Table Properties

```sql
-- Create table with properties
CREATE TABLE iceberg.db.table (
  id BIGINT,
  data STRING
) USING iceberg
TBLPROPERTIES (
  'write.format.default' = 'parquet',
  'write.parquet.compression-codec' = 'zstd',
  'write.distribution-mode' = 'hash',
  'read.split.target-size' = '134217728'
);

-- Alter existing table
ALTER TABLE iceberg.db.table SET TBLPROPERTIES (
  'write.target-file-size-bytes' = '268435456'
);
```

## Configuration Precedence

Configuration values are resolved in the following order (highest to lowest priority):

### Read Configuration Precedence

1. **Read options** - Options passed via `.option()` in DataFrame API
2. **Session configuration** - `spark.sql.iceberg.*` properties
3. **Table metadata properties** - Table-level properties

### Write Configuration Precedence

1. **Write options** - Options passed via `.option()` in DataFrame API
2. **Session configuration** - `spark.sql.iceberg.*` properties
3. **Table metadata properties** - Table-level properties

### Example: Precedence Demonstration

```scala
// Table property: write.distribution-mode = none
// Session config: spark.sql.iceberg.distribution-mode = hash
// Write option: distribution-mode = range

// This write uses "range" (write option wins)
df.writeTo("iceberg.db.table")
  .option("distribution-mode", "range")
  .append()

// This write uses "hash" (session config wins over table property)
df.writeTo("iceberg.db.table")
  .append()
```

### Catalog Table Defaults vs Overrides

- **table-default.***: Applied when table doesn't specify a value; table property wins
- **table-override.***: Always applied; overrides any table property

```properties
# Default: table can override
spark.sql.catalog.iceberg.table-default.write.format.default=parquet

# Override: always enforced
spark.sql.catalog.iceberg.table-override.write.metadata.delete-after-commit.enabled=true
```
