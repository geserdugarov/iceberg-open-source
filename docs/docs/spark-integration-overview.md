---
title: "Spark Integration Overview"
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

# Spark Integration Overview

Apache Iceberg provides a deep integration with Apache Spark through the DataSource V2 API. This enables Spark to read and write Iceberg tables with full support for Iceberg's table format features, including schema evolution, hidden partitioning, time travel, and concurrent writes.

## Supported Versions

Iceberg maintains integrations for multiple Spark versions simultaneously. The following table summarizes the supported combinations:

| Spark Version | Scala Versions   | JDK Requirement | Default Build |
|---------------|------------------|-----------------|---------------|
| 3.4           | 2.12, 2.13       | 8, 11, or 17    | No            |
| 3.5           | 2.12, 2.13       | 8, 11, or 17    | No            |
| 4.0           | 2.13 only        | 17 or 21        | No            |
| 4.1           | 2.13 only        | 17 or 21        | Yes           |

Spark 4.1 is the default version included in the build. To build other versions, use the `-DsparkVersions` flag (see [Build and Development Configuration](spark-integration-configuration.md)).

## Key Capabilities

The Iceberg Spark integration provides the following capabilities:

- **Batch reads and writes** — Full scan planning with predicate pushdown, column projection, and partition pruning. Write support includes INSERT INTO, INSERT OVERWRITE, MERGE INTO, UPDATE, and DELETE.
- **Structured streaming** — Streaming reads and writes using Spark Structured Streaming (see [Structured Streaming](spark-structured-streaming.md)).
- **SQL DDL** — CREATE TABLE, ALTER TABLE, DROP TABLE, and other DDL commands through Spark SQL (see [DDL](spark-ddl.md)).
- **Stored procedures** — Built-in procedures for table maintenance operations such as snapshot expiration, data compaction, and metadata management (see [Procedures](spark-procedures.md)).
- **SQL extensions** — Custom SQL syntax for Iceberg-specific operations like partition management, branching, and tagging via `IcebergSparkSessionExtensions`.
- **Time travel** — Query historical table snapshots by timestamp or snapshot ID (see [Queries](spark-queries.md)).
- **Metadata tables** — Access table metadata (snapshots, manifests, data files, partitions) as queryable tables.

## Runtime Artifacts

Iceberg publishes shaded runtime JARs for each supported Spark version. These are self-contained fat JARs with relocated dependencies, suitable for direct deployment to Spark clusters.

The JAR naming pattern is:

```plain
iceberg-spark-runtime-{sparkVersion}_{scalaVersion}-{icebergVersion}.jar
```

For example: `iceberg-spark-runtime-4.1_2.13-1.8.0.jar`

The runtime JAR bundles the core Spark integration, SQL extensions, and all required Iceberg dependencies (core, data, file format modules, catalog integrations) into a single artifact with relocated packages to avoid classpath conflicts.

## Further Reading

- [Getting Started](spark-getting-started.md) — Quick start guide for using Iceberg with Spark
- [Configuration](spark-configuration.md) — Catalog configuration and Spark session properties
- [DDL](spark-ddl.md) — SQL DDL commands for managing Iceberg tables
- [Queries](spark-queries.md) — Reading data, time travel, metadata tables
- [Writes](spark-writes.md) — INSERT, MERGE INTO, UPDATE, DELETE operations
- [Procedures](spark-procedures.md) — Built-in stored procedures for table maintenance
- [Structured Streaming](spark-structured-streaming.md) — Streaming reads and writes
- [Architecture](spark-integration-architecture.md) — Technical architecture of the Spark integration modules
- [Build and Development Configuration](spark-integration-configuration.md) — Build commands, test execution, and module configuration
