---
title: "Spark Integration Architecture"
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

# Spark Integration Architecture

The Iceberg Spark integration is organized as a multi-version module system under the `spark/` directory. Each supported Spark version (3.4, 3.5, 4.0, 4.1) has its own directory (`spark/v3.4/`, `spark/v3.5/`, `spark/v4.0/`, `spark/v4.1/`) containing three submodules that together provide the full integration.

## Submodule Structure

Each Spark version directory contains three submodules:

### `spark/` — Core DataSource V2 Implementation

The core submodule implements Spark's DataSource V2 API and provides:

- **Catalog integration** — `SparkCatalog` and `SparkSessionCatalog` implement Spark's catalog interfaces, enabling Spark SQL to manage Iceberg tables by name.
- **Table representation** — `SparkTable` wraps an Iceberg table and exposes it to Spark's query planning infrastructure.
- **Data source** — `IcebergSource` is the DataSource V2 entry point registered under the `iceberg` format name.
- **Scan planning** — Scan builders that translate Spark filter expressions into Iceberg predicates, supporting predicate pushdown, column projection, and partition pruning.
- **Write operations** — Write builders for INSERT INTO, INSERT OVERWRITE, MERGE INTO, UPDATE, and DELETE.
- **Stored procedures** — Built-in procedures for table maintenance (snapshot expiration, compaction, orphan file removal, etc.).
- **Actions** — Programmatic APIs for operations like rewriting data files, rewriting manifests, and migrating tables.
- **Functions** — Spark SQL function implementations for Iceberg transforms (bucket, truncate, years, months, days, hours).

### `spark-extensions/` — SQL Parser Extensions

The extensions submodule provides ANTLR-based custom SQL parser extensions for Iceberg-specific syntax that goes beyond standard Spark SQL. This includes:

- Partition management (ADD PARTITION FIELD, DROP PARTITION FIELD, REPLACE PARTITION FIELD)
- Branch operations (CREATE BRANCH, DROP BRANCH, REPLACE BRANCH)
- Tag operations (CREATE TAG, DROP TAG, REPLACE TAG)
- View operations
- Other Iceberg-specific SQL statements

The entry point is `IcebergSparkSessionExtensions` (Scala class), which injects custom parser rules, analyzer rules, optimizer rules, and planner strategies into the Spark session.

### `spark-runtime/` — Shaded Runtime JAR

The runtime submodule produces a shaded fat JAR using the Gradle Shadow plugin. This JAR bundles the core Spark integration, SQL extensions, and all required Iceberg dependencies into a single artifact with relocated packages to avoid classpath conflicts. This is the JAR that users deploy to their Spark clusters.

## DataSource V2 Integration

The key classes that implement Spark's DataSource V2 API are:

| Class | Package | Role |
|-------|---------|------|
| `SparkCatalog` | `o.a.i.spark` | Implements Spark's `TableCatalog`, `SupportsNamespaces` — primary catalog for Iceberg tables |
| `SparkSessionCatalog` | `o.a.i.spark` | Wraps Spark's built-in session catalog to add Iceberg table support alongside non-Iceberg tables |
| `SparkTable` | `o.a.i.spark.source` | Implements Spark's `Table` interface, wrapping an Iceberg `Table` |
| `IcebergSource` | `o.a.i.spark.source` | DataSource V2 provider registered as the `iceberg` format |

(Package prefix `o.a.i` = `org.apache.iceberg`)

## SQL Extensions Pipeline

When `IcebergSparkSessionExtensions` is registered via `spark.sql.extensions`, it injects the following into the Spark session:

1. **Parser extensions** — Custom ANTLR grammar rules that parse Iceberg-specific SQL syntax into logical plan nodes.
2. **Analyzer extensions** — Rules that resolve Iceberg-specific logical plan nodes during analysis.
3. **Optimizer extensions** — Rules that optimize query plans for Iceberg tables.
4. **Planner strategies** — Physical plan strategies that translate Iceberg logical plans into executable physical plans.

Configuration:

```plain
spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
```

## Multi-Version Module System

Spark modules are conditionally included in the build based on the `-DsparkVersions` system property. By default, only Spark 4.1 is built. The Gradle build system handles this through:

- **Conditional inclusion** — The `settings.gradle` file reads `-DsparkVersions` and includes only the requested version directories.
- **Project naming convention** — Gradle projects are named `iceberg-spark-{version}_{scalaVersion}` (e.g., `iceberg-spark-4.1_2.13`).
- **Version-specific build files** — Each version directory has its own `build.gradle` that may contain version-specific dependencies or configurations.

To include all Spark versions in the build:

```bash
./gradlew -DsparkVersions=3.4,3.5,4.0,4.1 build -x test -x integrationTest
```

Or use the `-DallModules` flag to include all engine versions:

```bash
./gradlew -DallModules build -x test -x integrationTest
```

## Dependency Diagram

The following diagram shows how the runtime JAR is assembled from the submodules and Iceberg core libraries:

```plain
spark-runtime (shaded fat JAR)
 ├── spark (core DataSource V2 implementation)
 │    ├── iceberg-api
 │    ├── iceberg-common
 │    ├── iceberg-core
 │    ├── iceberg-data
 │    ├── iceberg-parquet
 │    ├── iceberg-orc
 │    ├── iceberg-arrow
 │    └── iceberg-bundled-guava
 ├── spark-extensions (SQL parser extensions)
 └── catalog/storage integrations (AWS, GCP, Azure, etc.)
```

All transitive dependencies are relocated (shaded) to avoid conflicts with dependencies already present on the Spark classpath.

For build and development configuration details, see [Build and Development Configuration](spark-integration-configuration.md). For a high-level overview, see [Spark Integration Overview](spark-integration-overview.md).
