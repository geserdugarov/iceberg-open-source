<!--
  - Licensed to the Apache Software Foundation (ASF) under one
  - or more contributor license agreements.  See the NOTICE file
  - distributed with this work for additional information
  - regarding copyright ownership.  The ASF licenses this file
  - to you under the Apache License, Version 2.0 (the
  - "License"); you may not use this file except in compliance
  - with the License.  You may obtain a copy of the License at
  -
  -   http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing,
  - software distributed under the License is distributed on an
  - "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  - KIND, either express or implied.  See the License for the
  - specific language governing permissions and limitations
  - under the License.
  -->

# Apache Iceberg - Spark 3.4 Integration

This module provides the Apache Iceberg integration for Apache Spark 3.4, implementing the Spark DataSource V2 API to enable Spark to read and write Iceberg tables.

## Submodules

- **`spark/`** - Core Spark integration including DataSource V2 implementation, scan planning, and table operations.
- **`spark-extensions/`** - Spark SQL extensions providing ANTLR-based custom SQL parser extensions for Iceberg-specific SQL syntax.
- **`spark-runtime/`** - Shaded runtime JAR (fat jar) with relocated dependencies for deployment.

## Scala Compatibility

Spark 3.4 supports both Scala 2.12 (default) and 2.13. Use `-DscalaVersion=2.13` to build with Scala 2.13.

## Build Commands

```bash
# Build this module (default Scala 2.12)
./gradlew -DsparkVersions=3.4 build -x test -x integrationTest

# Build with Scala 2.13
./gradlew -DsparkVersions=3.4 -DscalaVersion=2.13 build -x test -x integrationTest

# Run tests
./gradlew -DsparkVersions=3.4 :iceberg-spark:iceberg-spark-3.4_2.12:test

# Run a single test class
./gradlew -DsparkVersions=3.4 :iceberg-spark:iceberg-spark-3.4_2.12:test --tests "org.apache.iceberg.spark.TestClassName"
```
