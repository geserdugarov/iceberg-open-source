# Apache Iceberg Architecture Overview

This document provides a high-level architecture overview of Apache Iceberg, excluding the Native Layer modules.

---

## 1. Layered Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        QUERY ENGINES                               │
│   ┌───────────┐  ┌───────────┐  ┌────────────┐  ┌──────────────┐   │
│   │  Spark    │  │  Flink    │  │  MapReduce │  │  Hive        │   │
│   │  3.5/4.0  │  │  1.20/    │  │            │  │              │   │
│   │  /4.1     │  │  2.0/2.1  │  │            │  │              │   │
│   └─────┬─────┘  └─────┬─────┘  └──────┬─────┘  └──────┬───────┘   │
│         │              │               │               │           │
└─────────┼──────────────┼───────────────┼───────────────┼───────────┘
          │              │               │               │
┌─────────▼──────────────▼───────────────▼───────────────▼───────────┐
│                   ENGINE CONNECTORS                                │
│   ┌───────────┐  ┌───────────┐  ┌────────────┐  ┌──────────────┐   │
│   │ spark/    │  │ flink/    │  │ mr/        │  │ hive-        │   │
│   │  source/  │  │           │  │            │  │ metastore/   │   │
│   └─────┬─────┘  └─────┬─────┘  └──────┬─────┘  └──────┬───────┘   │
│         │              │               │               │           │
└─────────┼──────────────┼───────────────┼───────────────┼───────────┘
          │              │               │               │
          └──────────────┴───────┬───────┴───────────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│                     ICEBERG API (api/)                             │
│                                                                    │
│   Table, Schema, PartitionSpec, SortOrder, Snapshot                │
│   TableScan, AppendFiles, RewriteFiles, DeleteFiles                │
│   Catalog, Expression, Types, Transforms                           │
│                                                                    │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│                   ICEBERG CORE (core/)                             │
│                                                                    │
│   TableMetadata, BaseTable, BaseTableScan, ManifestGroup           │
│   SnapshotProducer, MergingSnapshotProducer                        │
│   FastAppend, MergeAppend, BaseRewriteFiles, BaseRowDelta          │
│   ManifestReader, ManifestWriter, ManifestListWriter, ManifestFiles│
│   TableMetadataParser, SchemaParser, MetadataUpdateParser          │
│                                                                    │
└────────────┬───────────────────┬───────────────────┬───────────────┘
             │                   │                   │
┌────────────▼─────┐  ┌──────────▼────────┐  ┌───────▼──────────────┐
│ FILE FORMATS     │  │ DATA MODULE       │  │ COMMON UTILITIES     │
│                  │  │ (data/)           │  │ (common/)            │
│ ┌─────────────┐  │  │                   │  │                      │
│ │ parquet/    │  │  │ IcebergGenerics   │  │ Shared utilities     │
│ │ Parquet.java│  │  │ GenericReader     │  │ across modules       │
│ │ ParquetRdr  │  │  │ GenericAppender-  │  │ (DynClasses,         │
│ │ ParquetWtr  │  │  │   Factory         │  │  DynMethods, etc.)   │
│ ├─────────────┤  │  │ GenericFileWriter-│  │                      │
│ │ orc/        │  │  │   Factory         │  │                      │
│ ├─────────────┤  │  │ DeleteFilter /    │  │                      │
│ │ arrow/      │  │  │ BaseDeleteLoader  │  │                      │
│ └─────────────┘  │  │                   │  │                      │
└──────────────────┘  └───────────────────┘  └──────────────────────┘
             │
┌────────────▼───────────────────────────────────────────────────────┐
│                     FILE I/O LAYER                                 │
│                                                                    │
│   InputFile, OutputFile, FileIO                                    │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│   │ aws/     │  │ azure/   │  │ gcp/     │  │ hadoop (local/   │   │
│   │ S3FileIO │  │ ADLSFile │  │ GCSFile  │  │  HDFS)           │   │
│   │          │  │ IO       │  │ IO       │  │                  │   │
│   └──────────┘  └──────────┘  └──────────┘  └──────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
             │
┌────────────▼───────────────────────────────────────────────────────┐
│                        STORAGE                                     │
│    S3  |  ADLS  |  GCS  |  HDFS  |  Local FS                       │
└────────────────────────────────────────────────────────────────────┘
```

---

## 2. Module Structure

```
iceberg/
├── api/                    Public API interfaces and contracts
├── core/                   Reference implementation (incl. avro, rest,
│                           jdbc, hadoop, view, puffin, encryption)
├── common/                 Shared utilities across modules
├── bom/                    Maven BOM for downstream dependency management
├── bundled-guava/          Shaded Guava used by every module
│
├── parquet/                Parquet file format reader/writer
├── orc/                    ORC file format support
├── arrow/                  Apache Arrow columnar (vectorized) integration
├── data/                   Generic JVM read/write — Records, DeleteFilter,
│                           BaseDeleteLoader, *FileWriterFactory
│
├── spark/                  Spark DataSourceV2 integration
│   ├── v3.5/               (spark, spark-extensions, spark-runtime)
│   ├── v4.0/
│   └── v4.1/
│
├── flink/                  Flink integration
│   ├── v1.20/
│   ├── v2.0/
│   └── v2.1/
│
├── mr/                     Hadoop MapReduce InputFormat (also used by Hive)
├── hive-metastore/         HiveCatalog (Thrift client to Hive Metastore)
│
├── aws/                    S3FileIO, GlueCatalog, DynamoDbCatalog, KMS
├── aws-bundle/             Shaded AWS runtime
├── azure/                  ADLSFileIO (Gen2)
├── azure-bundle/           Shaded Azure runtime
├── gcp/                    GCSFileIO, GCP utilities
├── gcp-bundle/             Shaded GCP runtime
├── aliyun/                 Alibaba Cloud OSS integration
├── dell/                   Dell ECS catalog + FileIO
│
├── bigquery/               BigQuery Metastore catalog integration
├── snowflake/              SnowflakeCatalog (read-only)
├── nessie/                 NessieCatalog (versioned branches/tags)
├── delta-lake/             Delta Lake → Iceberg migration helpers
│
├── kafka-connect/          Kafka Connect sink (kafka-connect,
│                           kafka-connect-runtime, kafka-connect-events,
│                           kafka-connect-transforms)
├── open-api/               REST catalog OpenAPI spec + conformance tests
│
├── format/                 Iceberg format specification (spec.md, view-spec,
│                           puffin-spec, udf-spec, gcm-stream-spec)
├── docs/                   Versioned MkDocs documentation
└── site/                   Top-level docs site (built with MkDocs)
```

---

## 3. Table Metadata Structure

```
                    ┌─────────────────────────┐
                    │   metadata.json         │   (default v2, supported up
                    │                         │    to v4 — see TableMetadata
                    │  format-version         │    DEFAULT_TABLE_FORMAT_VERSION
                    │  table-uuid             │    and SUPPORTED_TABLE_FORMAT_-
                    │  location               │    VERSION)
                    │  last-sequence-number   │
                    │  next-row-id (v3+)      │
                    │  schemas[]              │
                    │  partition-specs[]      │
                    │  sort-orders[]          │
                    │  current-snapshot-id ───┼──┐
                    │  snapshots[] ───────────┼──┤
                    │  refs{} (branches/tags) │  │
                    │  statistics[] /         │  │
                    │  partition-statistics[] │  │
                    │  properties{}           │  │
                    │  metadata-log[]         │  │
                    └─────────────────────────┘  │
                                                 │
                    ┌────────────────────────────┘
                    ▼
            ┌───────────────────┐
            │    Snapshot       │
            │                   │
            │  snapshot-id      │
            │  parent-id        │
            │  sequence-number  │
            │  timestamp-ms     │
            │  operation        │
            │  summary{}        │
            │  first-row-id     │   (v3+, row lineage)
            │  added-rows       │   (v3+, row lineage)
            │  manifest-list ───┼──┐
            └───────────────────┘  │
                                   │
                    ┌──────────────┘
                    ▼
          ┌──────────────────────┐
          │  Manifest List       │    Avro file: snap-<snapshotId>-
          │                      │    <attempt>-<commitUUID>.avro (see
          │  ┌────────────────┐  │    SnapshotProducer.manifestListPath)
          │  │ ManifestFile 1 ├──┼──┐
          │  ├────────────────┤  │  │
          │  │ ManifestFile 2 │  │  │   Each entry contains:
          │  ├────────────────┤  │  │   - manifest path
          │  │ ManifestFile N │  │  │   - partition spec id
          │  └────────────────┘  │  │   - added/existing/deleted counts
          └──────────────────────┘  │   - partition field summaries
                                    │     (min/max for partition pruning)
                                    │   - first-row-id (v3+, assigned by
                                    │     ManifestListWriter for DATA
                                    │     manifests)
                    ┌───────────────┘
                    ▼
          ┌──────────────────────┐
          │  Manifest File       │    <uuid>-m<N>.<ext>
          │                      │    Avro by default; Parquet from v4
          │  ┌────────────────┐  │    (TableMetadata.MIN_FORMAT_VERSION_-
          │  │ ManifestEntry 1├──┼──┐ PARQUET_MANIFESTS — selected in
          │  ├────────────────┤  │  │ SnapshotProducer).
          │  │ ManifestEntry 2│  │  │
          │  ├────────────────┤  │  │ Each entry contains:
          │  │ ManifestEntry N│  │  │ - status (ADDED/EXISTING/DELETED)
          │  └────────────────┘  │  │ - snapshot-id
          └──────────────────────┘  │ - content-file reference (DataFile
                                    │   or DeleteFile; manifest content
                                    │   = DATA or DELETES)
                                    │
                    ┌───────────────┘
                    ▼
          ┌──────────────────────┐
          │  ContentFile         │    (DataFile, DeleteFile)
          │                      │
          │  file-path           │    Actual data / delete files:
          │  file-format         │    - Parquet (.parquet)
          │  content             │    - ORC (.orc)
          │    DATA              │    - Avro (.avro)
          │    POSITION_DELETES  │    - Puffin (DV blobs, v3)
          │    EQUALITY_DELETES  │
          │  partition           │    Per-column stats:
          │  record-count        │    - null counts
          │  file-size-in-bytes  │    - NaN counts
          │  column-sizes{}      │    - lower/upper bounds
          │  value-counts{}      │
          │  null-value-counts{} │    Row lineage (v3, data files):
          │  nan-value-counts{}  │    - first-row-id
          │  lower-bounds{}      │
          │  upper-bounds{}      │    DeleteFile / DV extras:
          │  equality-ids[]      │    - referenced-data-file
          │  sort-order-id       │    - content-offset
          │                      │    - content-size-in-bytes
          └──────────────────────┘
```

---

## 4. Key API Interfaces

```
                        ┌──────────────┐
                        │   Catalog    │
                        │              │
                        │ loadTable()  │
                        │ createTable()│
                        │ dropTable()  │
                        └──────┬───────┘
                               │ returns
                               ▼
                        ┌──────────────┐
                        │    Table     │
                        │              │
                        │ schema()     │──────► Schema ──► StructType ──► NestedField[]
                        │ spec()       │──────► PartitionSpec ──► PartitionField[]
                        │ sortOrder()  │──────► SortOrder ──► SortField[]
                        │ currentSnap- │──────► Snapshot
                        │   shot()     │
                        │ properties() │
                        │ io()         │──────► FileIO
                        │ location()   │
                        │              │
                        │ newScan()  ──┼──┐    READS
                        │              │  │
                        │ newAppend()──┼──┼─┐  WRITES / MUTATIONS
                        │ newOverwrite │  │ │
                        │ newRewrite() │  │ │
                        │ newDelete()  │  │ │
                        │ newRowDelta  │  │ │
                        └──────────────┘  │ │
                               ┌──────────┘ │
                               ▼            │
                     ┌──────────────────┐   │
                     │   TableScan      │   │
                     │                  │   │
                     │ select(columns)  │   │
                     │ filter(expr)     │   │
                     │ planFiles()    ──┼───┼──► CloseableIterable<FileScanTask>
                     │ planTasks()    ──┼───┼──► CloseableIterable<CombinedScanTask>
                     └──────────────────┘   │
                                            │
                               ┌────────────┘
                               ▼
                     ┌──────────────────────┐
                     │ PendingUpdate<T>     │       (root API)
                     │                      │
                     │  apply() → T         │  preview uncommitted changes
                     │  commit() ──────────────────► atomic metadata update
                     ├──────────────────────┤
                     │ SnapshotUpdate<T>    │       (produces a new snapshot;
                     │                      │        adds set(prop, value)
                     │                      │        + commit hooks)
                     │ ├ AppendFiles        │  add new data files
                     │ ├ OverwriteFiles     │  replace data files (filter-based)
                     │ ├ RewriteFiles       │  swap old files for new (compaction)
                     │ ├ DeleteFiles        │  delete data files by expression
                     │ ├ RowDelta           │  add data + delete files (MoR)
                     │ ├ ReplacePartitions  │  dynamic partition overwrite
                     │ └ RewriteManifests   │  optimize manifest files
                     ├──────────────────────┤
                     │ Other PendingUpdates │  (not part of SnapshotUpdate)
                     │                      │
                     │ ├ ManageSnapshots    │  PendingUpdate<Snapshot>
                     │ │                    │   (rollback, cherry-pick,
                     │ │                    │    branch/tag management;
                     │ │                    │    may produce a snapshot)
                     │ ├ ExpireSnapshots    │  PendingUpdate<List<Snapshot>>
                     │ │                    │   (removes snapshots; no new
                     │ │                    │    snapshot is produced)
                     │ ├ UpdateSchema       │  metadata-only: schema evolution
                     │ ├ UpdatePartitionSpec│  metadata-only: spec evolution
                     │ ├ UpdateProperties   │  metadata-only: table properties
                     │ └ UpdateLocation     │  metadata-only: table location
                     └──────────────────────┘
```

---

## 5. Catalog Subsystem

```
┌──────────────────────────────────────────────────────────────────┐
│                     Spark Catalog Layer                          │
│                                                                  │
│  ┌──────────────────┐     ┌──────────────────────────┐           │
│  │ SparkCatalog     │     │ SparkSessionCatalog      │           │
│  │ (TableCatalog)   │     │ (wraps built-in catalog) │           │
│  └────────┬─────────┘     └───────────┬──────────────┘           │
│           │                           │                          │
│           └─────────┬─────────────────┘                          │
│                     │ delegates to                               │
└─────────────────────┼────────────────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────────────────┐
│                 Iceberg Catalog (api/)                           │
│                                                                  │
│  interface Catalog                                               │
│  ├── loadTable(TableIdentifier) → Table                          │
│  ├── createTable(ident, schema, spec, location, props) → Table   │
│  ├── dropTable(ident, purge)                                     │
│  ├── renameTable(from, to)                                       │
│  └── listTables(namespace) → List<TableIdentifier>               │
│                                                                  │
│  interface SupportsNamespaces                                    │
│  ├── createNamespace(namespace, meta)                            │
│  ├── dropNamespace(namespace)                                    │
│  └── listNamespaces() → List<Namespace>                          │
│                                                                  │
│  interface ViewCatalog        (table-like operations for views)  │
│  ├── loadView(ident) → View                                      │
│  ├── buildView(ident) → ViewBuilder                              │
│  ├── dropView(ident) / renameView(from, to)                      │
│  └── listViews(namespace)                                        │
│                                                                  │
└─────────────────────┬────────────────────────────────────────────┘
                      │
        ┌─────────────┼──────────────┬────────────────┐
        ▼             ▼              ▼                ▼
  ┌────────────┐ ┌───────────┐ ┌────────────┐ ┌──────────────┐
  │ HiveCatalog│ │RESTCatalog│ │JdbcCatalog │ │HadoopCatalog │
  │ hive-      │ │ core/rest │ │ core/jdbc  │ │(no metastore)│
  │ metastore  │ │           │ │            │ │              │
  │ Hive       │ │ REST API  │ │ JDBC       │ │ filesystem   │
  │ Metastore  │ │ server    │ │ database   │ │ based        │
  └────────────┘ └───────────┘ └────────────┘ └──────────────┘

  Additional implementations: GlueCatalog (aws/glue), NessieCatalog
  (nessie/), SnowflakeCatalog (snowflake/), BigQueryMetastoreCatalog
  (bigquery/), DynamoDbCatalog (aws/dynamodb), EcsCatalog (dell/ecs).
  CachingCatalog (core/) decorates any of the above with a table cache.

       │             │             │                │
       └─────────────┴──────┬──────┴────────────────┘
                            ▼
                    ┌────────────────┐
                    │ TableOperations│   Atomic metadata updates
                    │                │   (HMS: lock+swap, REST: server
                    │ current()      │    side commit, Hadoop: rename,
                    │ refresh()      │    JDBC: row CAS, Glue: optimistic
                    │ commit(base,   │    locking on version-id)
                    │   updated)     │
                    └────────────────┘
```

---

## 6. Spark DataSourceV2 Integration

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Spark SQL                                     │
│                                                                      │
│  SELECT * FROM catalog.db.table WHERE x > 10                         │
│  INSERT INTO catalog.db.table VALUES (...)                           │
│                                                                      │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  IcebergSource (TableProvider, SupportsCatalogOptions)               │
│  Registered via META-INF/services as "iceberg" format                │
│                                                                      │
│  getTable() → SparkTable                                             │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  SparkTable                                                          │
│  Implements: SupportsRead, SupportsWrite, SupportsDeleteV2,          │
│              SupportsRowLevelOperations, SupportsMetadataColumns     │
│                                                                      │
│  ┌──────────────────┐          ┌──────────────────────┐              │
│  │ newScanBuilder() │          │ newWriteBuilder()    │              │
│  └────────┬─────────┘          └──────────┬───────────┘              │
│           │                               │                          │
└───────────┼───────────────────────────────┼──────────────────────────┘
            │                               │
  ┌─────────▼──────────┐         ┌──────────▼───────────┐
  │ SparkScanBuilder   │         │ SparkWriteBuilder    │
  │ (filter/col push)  │         │ (append/overwrite)   │
  └─────────┬──────────┘         └──────────┬───────────┘
            │                               │
  ┌─────────▼──────────┐         ┌──────────▼───────────┐
  │ SparkBatchQueryScan│         │ SparkWrite           │
  │ (logical scan)     │         │ (physical write)     │
  └─────────┬──────────┘         └──────────┬───────────┘
            │                               │
  ┌─────────▼──────────┐         ┌──────────▼───────────┐
  │ SparkBatch         │         │ BatchAppend /        │
  │ (partition planning│         │ DynamicOverwrite     │
  │  + reader factory) │         │ (commit strategy)    │
  └─────────┬──────────┘         └──────────┬───────────┘
            │                               │
     ┌──────▼──────┐                 ┌──────▼──────┐
     │  EXECUTORS  │                 │  EXECUTORS  │
     │ RowDataRdr  │                 │ DataWriter  │
     │ BatchDtaRdr │                 │ FileWriter  │
     └─────────────┘                 └─────────────┘
```

---

## 7. File Format Integration

```
                    ┌────────────────────────────┐
                    │  Engine Layer              │
                    │                            │
                    │  SparkParquetReaders       │  (InternalRow <-> Parquet)
                    │  SparkParquetWriters       │
                    │  SparkOrcReader / Writer   │  (singular, OrcRowReader)
                    │  SparkPlannedAvroReader    │
                    │  SparkAvroWriter           │
                    │  VectorizedSparkParquet-   │
                    │    Readers (Arrow batches) │
                    └──────────┬─────────────────┘
                               │
                    ┌──────────▼────────────┐
                    │  Format Layer         │
                    │                       │
                    │  Parquet.read()       │──► ReadBuilder ──► ParquetReader
                    │  Parquet.writeData()  │──► DataWriteBuilder ──► ParquetWriter
                    │  ORC.read()           │──► ReadBuilder ──► OrcIterable
                    │  ORC.write()          │──► WriteBuilder ──► OrcFileAppender
                    │  Avro.read()          │──► ReadBuilder (record reader
                    │  Avro.writeData()     │     via PlannedDataReader in
                    │  Avro.writeDeletes()  │     data/avro; the legacy
                    │                       │     DataReader is @Deprecated)
                    │                       │
                    │  ParquetValueReader   │  (column-level decode)
                    │  ParquetValueWriter   │  (column-level encode)
                    │  Parquet variant      │  (shredded Variant; ORC stores
                    │    readers/writers    │   Variant as metadata/value
                    │                       │   struct, no shredding)
                    │  Puffin (core/puffin) │  (statistics + DV blobs)
                    └──────────┬────────────┘
                               │
                    ┌──────────▼────────────┐
                    │  Apache Libraries     │
                    │                       │
                    │  parquet-mr           │  (ParquetFileReader, ParquetFileWriter)
                    │  orc-core             │
                    │  avro                 │
                    │  arrow-vector         │  (vectorized read path)
                    └──────────┬────────────┘
                               │
                    ┌──────────▼────────────┐
                    │  FileIO               │
                    │                       │
                    │  InputFile.newStream()│
                    │  OutputFile.create()  │
                    └───────────────────────┘
```
