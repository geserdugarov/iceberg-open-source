---
title: "Spark Build and Development Configuration"
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

# Spark Build and Development Configuration

This page covers building, testing, and developing the Iceberg Spark integration modules.

## Prerequisites

- **Gradle wrapper** — Use `./gradlew` from the repository root. No separate Gradle installation is needed.
- **JDK requirements** vary by Spark version:

| Spark Version | Supported JDK Versions |
|---------------|------------------------|
| 3.4           | 8, 11, 17              |
| 3.5           | 8, 11, 17              |
| 4.0           | 17, 21                 |
| 4.1           | 17, 21                 |

## Build Commands

The following table shows the build command for each Spark version. All commands exclude tests for faster builds; remove `-x test -x integrationTest` to include them.

| Spark Version | Build Command |
|---------------|---------------|
| 3.4           | `./gradlew -DsparkVersions=3.4 build -x test -x integrationTest` |
| 3.5           | `./gradlew -DsparkVersions=3.5 build -x test -x integrationTest` |
| 4.0           | `./gradlew -DsparkVersions=4.0 build -x test -x integrationTest` |
| 4.1 (default) | `./gradlew build -x test -x integrationTest` |

Since Spark 4.1 is the default version, it does not require the `-DsparkVersions` flag.

### Building Multiple Versions

To build specific versions together:

```bash
./gradlew -DsparkVersions=3.4,3.5,4.0,4.1 build -x test -x integrationTest
```

To include all engine versions (Spark, Flink, Kafka):

```bash
./gradlew -DallModules build -x test -x integrationTest
```

## Scala Compatibility

| Spark Version | Default Scala | Supported Scala Versions | Override Flag |
|---------------|---------------|--------------------------|---------------|
| 3.4           | 2.12          | 2.12, 2.13              | `-DscalaVersion=2.13` |
| 3.5           | 2.12          | 2.12, 2.13              | `-DscalaVersion=2.13` |
| 4.0           | 2.13          | 2.13 only               | N/A           |
| 4.1           | 2.13          | 2.13 only               | N/A           |

Example building Spark 3.4 with Scala 2.13:

```bash
./gradlew -DsparkVersions=3.4 -DscalaVersion=2.13 build -x test -x integrationTest
```

## Running Tests

### All Tests for a Spark Version

```bash
# Spark 4.1 (default)
./gradlew :iceberg-spark:iceberg-spark-4.1_2.13:test

# Spark 3.5
./gradlew -DsparkVersions=3.5 :iceberg-spark:iceberg-spark-3.5_2.12:test

# Spark 3.4
./gradlew -DsparkVersions=3.4 :iceberg-spark:iceberg-spark-3.4_2.12:test

# Spark 4.0
./gradlew -DsparkVersions=4.0 :iceberg-spark:iceberg-spark-4.0_2.13:test
```

### Single Test Class

```bash
./gradlew :iceberg-spark:iceberg-spark-4.1_2.13:test --tests "org.apache.iceberg.spark.TestClassName"
```

### Extension Tests

Extensions tests are in a separate submodule:

```bash
./gradlew :iceberg-spark:iceberg-spark-extensions-4.1_2.13:test
```

### Test Configuration

- **Test framework**: JUnit 5 with Mockito and AssertJ
- **Test logs**: Written to `build/testlogs/{module}.log`
- **Parallelism**: Controlled via `-DtestParallelism=auto`
- **JVM heap**: Configured via `org.gradle.jvmargs` in `gradle.properties` (default: `-Xmx1536m`)

## Code Formatting

The project uses Google Java Format 1.22.0 for Java sources and Scalafmt 3.9.7 for Scala sources. All source files require the Apache License header.

### Applying Formatting

```bash
# Default versions only
./gradlew spotlessApply

# All module versions
./gradlew spotlessApply -DallModules
```

### Quick Mode

Adding `-Pquick=true` skips Checkstyle and Error Prone checks, which speeds up development iteration:

```bash
./gradlew -DsparkVersions=4.1 build -Pquick=true
```

CI runs the full checks without quick mode.

## Module Naming Convention

Gradle project paths and Maven artifact names follow a consistent pattern:

| Component | Gradle Path | Maven Artifact |
|-----------|-------------|----------------|
| Core | `:iceberg-spark:iceberg-spark-{version}_{scala}` | `iceberg-spark-{version}_{scala}` |
| Extensions | `:iceberg-spark:iceberg-spark-extensions-{version}_{scala}` | `iceberg-spark-extensions-{version}_{scala}` |
| Runtime | `:iceberg-spark:iceberg-spark-runtime-{version}_{scala}` | `iceberg-spark-runtime-{version}_{scala}` |

For example, the Spark 4.1 core module:
- Gradle: `:iceberg-spark:iceberg-spark-4.1_2.13`
- Maven: `org.apache.iceberg:iceberg-spark-4.1_2.13`

## Gradle Properties Reference

Key properties in `gradle.properties` that affect the Spark build:

| Property | Default | Description |
|----------|---------|-------------|
| `systemProp.defaultSparkVersions` | `4.1` | Spark versions included in the default build |
| `systemProp.knownSparkVersions` | `3.4,3.5,4.0,4.1` | All recognized Spark versions |
| `systemProp.defaultScalaVersion` | `2.12` | Default Scala version for Spark 3.x builds |
| `systemProp.knownScalaVersions` | `2.12,2.13` | All recognized Scala versions |

These can be overridden on the command line using `-D` flags (e.g., `-DsparkVersions=3.4,3.5`).

For the technical architecture of these modules, see [Spark Integration Architecture](spark-integration-architecture.md). For a high-level overview, see [Spark Integration Overview](spark-integration-overview.md).
