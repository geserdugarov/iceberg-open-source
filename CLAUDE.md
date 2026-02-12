# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Apache Iceberg is a high-performance open table format for huge analytic datasets. This is the Java reference implementation. It enables engines like Spark, Trino, Flink, Presto, and Hive to work with the same tables concurrently.

## Build System

Gradle with wrapper. Requires JDK 17 or 21.

### Common Build Commands

```bash
# Build everything (with tests)
./gradlew build

# Build without tests
./gradlew build -x test -x integrationTest

# Core-only checks (no Spark/Flink/Kafka modules, faster)
./gradlew check -DsparkVersions= -DflinkVersions= -DkafkaVersions= -Pquick=true

# Build with all engine versions included
./gradlew -DallModules build -x test -x integrationTest
```

### Running Tests

```bash
# All tests for a specific module
./gradlew :iceberg-core:test

# Single test class
./gradlew :iceberg-core:test --tests "org.apache.iceberg.TestManifestReader"

# Single test method
./gradlew :iceberg-core:test --tests "org.apache.iceberg.TestManifestReader.testMethodName"

# Control parallel test execution
./gradlew test -DtestParallelism=auto
```

Test framework: JUnit 5 with Mockito and AssertJ. Test logs are written to `build/testlogs/{module}.log`.

### Code Formatting

```bash
# Check and apply formatting (default Spark/Flink versions only)
./gradlew spotlessApply

# Apply formatting across all module versions
./gradlew spotlessApply -DallModules
```

Java formatting uses Google Java Format 1.22.0. Scala uses Scalafmt 3.9.7. All source files require the Apache License header.

### Quick Mode

Adding `-Pquick=true` skips Checkstyle and Error Prone checks, useful during development iteration. CI runs the full checks.

## Module Architecture

**Core dependency chain:** `api` -> `common` -> `core` -> `data`

- **api** - Public interfaces and types (Schema, Table, Catalog, expressions, transforms)
- **common** - Shared utility classes
- **core** - Reference implementation (table metadata, manifests, scan planning, REST catalog client)
- **data** - Generic readers/writers for direct JVM use
- **bundled-guava** - Shaded/relocated Guava (`org.apache.iceberg.relocated.com.google.common`)

**File format modules:** `parquet`, `orc`, `arrow`

**Engine integrations** (multi-version, conditionally included via system properties):
- `spark/` - Spark 3.4, 3.5, 4.0, 4.1 (DataSource V2). Subdirectories: `v3.4/`, `v3.5/`, `v4.0/`, `v4.1/`
- `flink/` - Flink 1.20, 2.0, 2.1. Subdirectories: `v1.20/`, `v2.0/`, `v2.1/`
- `kafka-connect/` - Kafka Connect sink
- `mr/` - MapReduce/Hive input format

**Cloud storage:** `aws`, `gcp`, `azure`, `aliyun` (each with a `-bundle` shaded runtime variant)

**Catalog integrations:** `hive-metastore`, `nessie`, `snowflake`, `bigquery`, `dell`

**Other:** `delta-lake` (conversion), `open-api` (REST catalog OpenAPI spec), `bom` (bill of materials)

## Multi-Version Module System

Spark, Flink, and Kafka modules are conditionally included. Default versions are set in `gradle.properties`:
- `defaultSparkVersions=4.1`
- `defaultFlinkVersions=2.1`
- `defaultKafkaVersions=3`
- `defaultScalaVersion=2.12`

Override with: `-DsparkVersions=3.4,3.5` `-DflinkVersions=1.20,2.0` `-DscalaVersion=2.13`

Spark 4.x is Scala 2.13 only. Spark 3.x supports both 2.12 and 2.13.

## API Compatibility

RevAPI enforces binary compatibility on: `iceberg-api`, `iceberg-core`, `iceberg-parquet`, `iceberg-orc`, `iceberg-common`, `iceberg-data`. Breaking changes require a deprecation cycle per CONTRIBUTING.md semantic versioning rules.

## Code Style Notes

- Guava is shaded into `bundled-guava`; use `org.apache.iceberg.relocated.com.google.common` imports
- Guava is excluded from `compileClasspath` for all modules except `iceberg-bundled-guava`
- Error Prone is configured with project-specific rules in `baseline.gradle` (e.g., `LambdaMethodReference:ERROR`, `MissingOverride:ERROR`, `UnusedVariable:ERROR`)
- Logger convention: use `LOG` (not `log`) for logger field names
- Java source/target: 17
