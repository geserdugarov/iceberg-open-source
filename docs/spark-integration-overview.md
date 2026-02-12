# Apache Iceberg - Spark Integration Overview

This document provides an overview of how Apache Iceberg integrates with Apache Spark, enabling Spark to work with Iceberg tables using the DataSource V2 API.

## Table of Contents

1. [Introduction](#introduction)
2. [Supported Spark Versions](#supported-spark-versions)
3. [Module Structure](#module-structure)
4. [Quick Start](#quick-start)
5. [Key Features](#key-features)
6. [Related Documentation](#related-documentation)

## Introduction

Apache Iceberg provides a native Spark integration through the DataSource V2 API. This integration allows Spark applications to:

- Read and write Iceberg tables with full ACID guarantees
- Perform time travel queries using snapshots, timestamps, branches, and tags
- Execute row-level operations (UPDATE, DELETE, MERGE)
- Use Iceberg-specific SQL extensions and stored procedures
- Work with multiple catalog implementations (Hive, REST, Hadoop, etc.)

## Supported Spark Versions

The Iceberg project maintains separate modules for different Spark versions:

| Spark Version | Scala Versions | Module Path |
|---------------|----------------|-------------|
| Spark 3.4 | 2.12, 2.13 | `spark/v3.4/` |
| Spark 3.5 | 2.12, 2.13 | `spark/v3.5/` |
| Spark 4.0 | 2.13 only | `spark/v4.0/` |
| Spark 4.1 | 2.13 only | `spark/v4.1/` |

Default versions are configured in `gradle.properties`:
- `defaultSparkVersions=4.1`
- `defaultScalaVersion=2.12`

Build with specific versions using: `-DsparkVersions=3.4,3.5 -DscalaVersion=2.13`

## Module Structure

Each Spark version has three submodules:

```
spark/
└── v4.1/
    ├── spark/              # Core integration (Java)
    │   └── src/main/java/org/apache/iceberg/spark/
    │       ├── SparkCatalog.java
    │       ├── SparkTable.java
    │       ├── source/     # Read/write implementations
    │       ├── procedures/ # SQL stored procedures
    │       ├── functions/  # SQL functions
    │       └── actions/    # Maintenance actions
    │
    ├── spark-extensions/   # SQL extensions (Scala)
    │   └── src/main/scala/org/apache/iceberg/spark/extensions/
    │       ├── IcebergSparkSessionExtensions.scala
    │       └── catalyst/   # Parser, analyzer, optimizer rules
    │
    └── spark-runtime/      # Integration tests & runtime
```

## Quick Start

### 1. Configure Spark Session

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder()
  .appName("IcebergExample")
  // Register Iceberg SQL extensions
  .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
  // Configure an Iceberg catalog
  .config("spark.sql.catalog.iceberg", "org.apache.iceberg.spark.SparkCatalog")
  .config("spark.sql.catalog.iceberg.type", "hive")
  .config("spark.sql.catalog.iceberg.uri", "thrift://localhost:9083")
  .getOrCreate()
```

### 2. Create and Query Tables

```sql
-- Create an Iceberg table
CREATE TABLE iceberg.db.sample (
  id BIGINT,
  data STRING,
  ts TIMESTAMP
) USING iceberg
PARTITIONED BY (days(ts));

-- Insert data
INSERT INTO iceberg.db.sample VALUES
  (1, 'a', timestamp '2024-01-01 10:00:00'),
  (2, 'b', timestamp '2024-01-02 12:00:00');

-- Query the table
SELECT * FROM iceberg.db.sample;

-- Time travel query
SELECT * FROM iceberg.db.sample VERSION AS OF 123456789;
SELECT * FROM iceberg.db.sample TIMESTAMP AS OF '2024-01-01 00:00:00';
```

### 3. Use DataFrame API

```scala
// Read from Iceberg table
val df = spark.read
  .format("iceberg")
  .load("iceberg.db.sample")

// Read with options (time travel)
val historicalDf = spark.read
  .format("iceberg")
  .option("snapshot-id", 123456789L)
  .load("iceberg.db.sample")

// Write to Iceberg table
df.writeTo("iceberg.db.sample")
  .option("distribution-mode", "hash")
  .append()
```

## Key Features

### Catalog Integration

Iceberg provides two catalog implementations for Spark:

- **SparkCatalog**: A standalone catalog for Iceberg tables only
- **SparkSessionCatalog**: Extends Spark's session catalog to support both Iceberg and non-Iceberg tables

### Time Travel

Query historical data using:
- Snapshot ID: `VERSION AS OF <snapshot-id>`
- Timestamp: `TIMESTAMP AS OF '<timestamp>'`
- Branch: Read from named branches
- Tag: Read from named tags

### Row-Level Operations

Full support for SQL DML operations:
- `UPDATE table SET col = value WHERE condition`
- `DELETE FROM table WHERE condition`
- `MERGE INTO table USING source ON condition ...`

### SQL Extensions

Iceberg-specific DDL operations:
- Partition evolution: `ALTER TABLE ... ADD PARTITION FIELD`
- Schema evolution: `ALTER TABLE ... ADD COLUMN`
- Branch/tag management: `ALTER TABLE ... CREATE BRANCH`
- Identifier fields: `ALTER TABLE ... SET IDENTIFIER FIELDS`

### Stored Procedures

Maintenance operations accessible via SQL:
- `CALL iceberg.system.expire_snapshots(...)`
- `CALL iceberg.system.rewrite_data_files(...)`
- `CALL iceberg.system.remove_orphan_files(...)`
- `CALL iceberg.system.rewrite_manifests(...)`

### Write-Audit-Publish (WAP)

Stage and validate changes before publishing:
```scala
spark.conf.set("spark.wap.branch", "audit_branch")
// Write to audit branch
df.writeTo("iceberg.db.sample").append()
// Validate data
// Publish when ready
spark.sql("CALL iceberg.system.fast_forward_branch('db.sample', 'main', 'audit_branch')")
```

## Related Documentation

- [Spark Integration Architecture](spark-integration-architecture.md) - Detailed architecture and components
- [Spark Integration Configuration](spark-integration-configuration.md) - Complete configuration reference
- [Apache Iceberg Official Documentation](https://iceberg.apache.org/docs/latest/spark-getting-started/)
