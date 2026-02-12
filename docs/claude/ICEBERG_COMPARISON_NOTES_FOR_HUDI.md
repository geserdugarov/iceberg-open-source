# Iceberg DSv2 Read Path: Comparison Notes for Hudi

This document analyzes Iceberg's Spark DataSource V2 read implementation to identify patterns that can be mirrored in Apache Hudi, versus those that require adaptation due to fundamental differences in timeline/metadata semantics.

## Table of Contents

1. [DSv2 Interface Patterns - Generally Applicable](#dsv2-interface-patterns---generally-applicable)
2. [Iceberg-Specific Patterns](#iceberg-specific-patterns)
3. [Hudi Adaptation Requirements](#hudi-adaptation-requirements)
4. [Migration Checklist](#migration-checklist)

---

## DSv2 Interface Patterns - Generally Applicable

These patterns from Iceberg's implementation are standard DSv2 practices that can be directly adopted in Hudi.

### 1. TableProvider / DataSourceRegister

**Iceberg Pattern:**
```java
public class IcebergSource implements DataSourceRegister,
    SupportsCatalogOptions, SessionConfigSupport {

  @Override
  public String shortName() { return "iceberg"; }

  @Override
  public Table getTable(StructType schema, Transform[] partitioning,
                        Map<String, String> properties) {
    // Returns SparkTable wrapping Iceberg table
  }
}
```

**What to Mirror in Hudi:**
- Implement `DataSourceRegister` with `shortName() = "hudi"`
- Implement `SupportsCatalogOptions` for catalog integration
- Implement `SessionConfigSupport` for session-level config prefix
- Return a `HudiSparkTable` from `getTable()`

**No Adaptation Needed:** This is pure DSv2 contract.

---

### 2. Table Interface Implementation

**Iceberg Pattern:**
```java
public class SparkTable implements Table, SupportsRead, SupportsWrite,
    SupportsDeleteV2, SupportsRowLevelOperations, SupportsMetadataColumns {

  @Override
  public ScanBuilder newScanBuilder(CaseInsensitiveStringMap options) {
    return new SparkScanBuilder(spark, icebergTable, options);
  }
}
```

**What to Mirror in Hudi:**
- Implement `SupportsRead` interface
- Implement `SupportsMetadataColumns` for `_hoodie_*` columns
- Create `newScanBuilder()` returning `HudiScanBuilder`

**Minor Adaptation:** Hudi's metadata columns (`_hoodie_commit_time`, `_hoodie_record_key`, etc.) differ from Iceberg's (`_file`, `_pos`, `_partition`).

---

### 3. ScanBuilder with Pushdowns

**Iceberg Pattern:**
```java
public class SparkScanBuilder implements ScanBuilder,
    SupportsPushDownV2Filters, SupportsPushDownRequiredColumns,
    SupportsPushDownAggregates, SupportsReportStatistics, SupportsPushDownLimit {

  @Override
  public Predicate[] pushPredicates(Predicate[] predicates) {
    // Convert, push what's supported, return remainder
  }

  @Override
  public void pruneColumns(StructType requestedSchema) {
    // Track required columns
  }
}
```

**What to Mirror in Hudi:**
- Implement all pushdown interfaces
- Filter classification (pushed vs post-filter)
- Column pruning with filter dependency tracking
- Statistics reporting for CBO

**Adaptation Required:** Filter-to-expression conversion must target Hudi's expression system, not Iceberg's.

---

### 4. Scan → Batch → InputPartition Flow

**Iceberg Pattern:**
```
SparkScan.toBatch() → SparkBatch
SparkBatch.planInputPartitions() → SparkInputPartition[]
SparkBatch.createReaderFactory() → PartitionReaderFactory
```

**What to Mirror in Hudi:**
- Create `HudiScan` implementing `Scan` with `toBatch()`
- Create `HudiBatch` implementing `Batch`
- Create `HudiInputPartition` implementing `InputPartition`
- Broadcast table metadata to executors
- Compute preferred locations

**Key Pattern:** Iceberg uses `SerializableTableWithSize.copyOf(table)` for efficient broadcast. Hudi should similarly serialize minimal metadata.

---

### 5. PartitionReaderFactory Selection

**Iceberg Pattern:**
```java
public PartitionReaderFactory createReaderFactory() {
  if (useColumnarBatchReads()) {
    return new SparkColumnarReaderFactory(batchReadConf);
  }
  return new SparkRowReaderFactory(readConf);
}
```

**What to Mirror in Hudi:**
- Vectorized readers for Parquet when applicable
- Row-level readers as fallback
- Reader selection based on file format and configuration

---

### 6. Statistics Reporting

**Iceberg Pattern:**
```java
public Statistics estimateStatistics() {
  return new Stats(
    OptionalLong.of(sizeInBytes),
    OptionalLong.of(numRows),
    columnStats
  );
}
```

**What to Mirror in Hudi:**
- Report row counts and sizes to Spark's CBO
- Optionally report column NDV statistics
- Use metadata for fast statistics when available

---

## Iceberg-Specific Patterns

These patterns are deeply tied to Iceberg's architecture and require significant adaptation for Hudi.

### 1. Manifest-Based File Discovery

**Iceberg Architecture:**
```
Snapshot → Manifest List → Manifests → DataFiles
                                    → DeleteFiles

ManifestGroup:
  - Filters manifests by partition bounds (ManifestEvaluator)
  - Filters files by column statistics (InclusiveMetricsEvaluator)
  - Associates delete files via DeleteFileIndex
```

**Hudi Equivalent:**
```
Timeline → Instants → FileGroups → FileSlices → BaseFile + LogFiles

Hudi must:
  - Query Timeline for committed instants
  - Resolve FileGroups from partition paths
  - Get latest FileSlice per FileGroup
  - Handle log file compaction state
```

**Key Difference:** Iceberg uses immutable manifests with statistics. Hudi uses mutable FileGroups with log-based updates.

---

### 2. Delete File Handling

**Iceberg Architecture:**
```java
DeleteFileIndex:
  - globalDeletes (equality deletes across all partitions)
  - eqDeletesByPartition (partition-scoped equality deletes)
  - posDeletesByPartition (position deletes by partition)
  - posDeletesByPath (position deletes by file path)
  - dvByPath (deletion vectors)

Applied at read time:
  FileScanTask.deletes() → merged delete files
  SparkDeleteFilter → applies deletes during iteration
```

**Hudi Equivalent:**
```
MOR Table Delete Handling:
  - Log files contain delete records
  - Resolved during log file merging
  - LogRecordMerger applies deletes

COW Table Delete Handling:
  - Deletes create new data files
  - Previous versions become invisible
```

**Key Difference:** Iceberg separates delete files explicitly. Hudi embeds deletes in log files (MOR) or rewrites files (COW).

---

### 3. Expression/Filter System

**Iceberg Expression System:**
```java
// Binding
Expression bound = Binder.bind(schema, expression, caseSensitive);

// Evaluation layers
ManifestEvaluator   → manifest partition stats
InclusiveMetricsEvaluator → file column stats
StrictMetricsEvaluator    → guaranteed matches
ResidualEvaluator   → remaining filter after partition eval
Evaluator           → row-level evaluation
```

**Hudi Equivalent:**
- Hudi has its own filter/expression system
- May use Spark's expressions directly in some paths
- Column stats in metadata table (if enabled)

**Adaptation Required:** Create Hudi-specific evaluators or leverage existing metadata-based filtering.

---

### 4. Partition Projection

**Iceberg Pattern:**
```java
Projections.inclusive(spec, caseSensitive)
  // Row filter: ts >= '2024-01-01'
  // Partition: day(ts)
  // Projected: day(ts) >= day('2024-01-01')
```

**Hudi Equivalent:**
- Hudi partitions are typically string paths (`dt=2024-01-01/`)
- Partition pruning is path-based, not projection-based
- May need to parse partition values from paths

---

### 5. Residual Filter Computation

**Iceberg Pattern:**
```java
ResidualEvaluator.residualFor(partitionData)
  // Original: ts >= A AND ts <= B
  // Partition: day(ts) = D
  // If day(A) < D < day(B): residual = TRUE
  // If D == day(A): residual = ts >= A
```

**Hudi Consideration:**
- If using identity partitions (e.g., `dt=value`), residuals may be simpler
- Transform-based partitions (bucket, truncate) need similar logic
- Could adopt similar residual computation for efficiency

---

### 6. Snapshot/Time-Travel

**Iceberg Pattern:**
```java
scan.useSnapshot(snapshotId)
scan.asOfTime(timestamp)
scan.useRef(branchOrTag)
```

**Hudi Equivalent:**
```java
// Hudi timeline-based time travel
beginInstantTime = "20240101000000"
// or
as.of.instant = "20240101000000"
```

**Key Difference:** Iceberg has snapshot IDs and refs (branches/tags). Hudi has linear timeline with instant times.

---

## Hudi Adaptation Requirements

### Timeline vs Snapshot Model

| Iceberg | Hudi | Adaptation |
|---------|------|------------|
| Snapshot ID | Instant Time | Map instant time to "virtual snapshot" |
| Snapshot contains file list | Timeline contains operations | Resolve file list from timeline |
| Manifest statistics | Metadata table | Query metadata for statistics |
| Branches/Tags | Timeline (no branches) | Hudi doesn't have branches; skip or add |

### File Indexing

| Iceberg | Hudi | Adaptation |
|---------|------|------------|
| ManifestFile with stats | FileGroup/FileSlice | Different indexing structure |
| DeleteFileIndex | Log file merging | Different delete handling |
| Column stats in manifest | Column stats in metadata table | Different stats location |

### Metadata Serialization

| Iceberg | Hudi | Adaptation |
|---------|------|------------|
| `SerializableTableWithSize` | Need similar pattern | Serialize minimal metadata |
| Schema as JSON string | Schema serialization | Same approach works |
| Broadcasted to executors | Same pattern | Implement broadcast wrapper |

### Filter Pushdown

| Iceberg | Hudi | Adaptation |
|---------|------|------------|
| `SparkV2Filters.convert()` | Hudi expression conversion | Create Hudi-specific converter |
| 3-way classification | Similar classification | Classify partition vs record level |
| Transform function support | Path-based partitions | Different partition handling |

---

## Migration Checklist

### Phase 1: Core DSv2 Structure

- [ ] **PR 1: TableProvider Implementation**
  - [ ] Create `HudiDataSource` implementing `DataSourceRegister`
  - [ ] Implement `SupportsCatalogOptions` for catalog integration
  - [ ] Implement `SessionConfigSupport` with `"hoodie"` prefix
  - [ ] Return `HudiSparkTable` from `getTable()`

- [ ] **PR 2: Table Interface**
  - [ ] Create `HudiSparkTable` implementing `Table`, `SupportsRead`
  - [ ] Implement `SupportsMetadataColumns` for `_hoodie_*` columns
  - [ ] Implement `schema()`, `partitioning()`, `capabilities()`
  - [ ] Create `newScanBuilder()` returning `HudiScanBuilder`

### Phase 2: Scan Building

- [ ] **PR 3: ScanBuilder Basic**
  - [ ] Create `HudiScanBuilder` implementing `ScanBuilder`
  - [ ] Implement `SupportsPushDownRequiredColumns` for column pruning
  - [ ] Track required columns including filter dependencies
  - [ ] Implement `build()` returning `HudiScan`

- [ ] **PR 4: Filter Pushdown**
  - [ ] Create `HudiV2Filters` for Spark-to-Hudi expression conversion
  - [ ] Implement `SupportsPushDownV2Filters`
  - [ ] Classify filters: partition-level vs record-level vs unsupported
  - [ ] Return post-scan predicates for Spark evaluation

- [ ] **PR 5: Statistics & Limits**
  - [ ] Implement `SupportsReportStatistics`
  - [ ] Query metadata table for row counts and sizes
  - [ ] Implement `SupportsPushDownLimit` for limit optimization

### Phase 3: Scan Execution

- [ ] **PR 6: Scan Object**
  - [ ] Create `HudiScan` implementing `Scan`
  - [ ] Implement task planning from timeline/file groups
  - [ ] Create `toBatch()` returning `HudiBatch`
  - [ ] Report statistics from planned tasks

- [ ] **PR 7: Batch & InputPartition**
  - [ ] Create `HudiBatch` implementing `Batch`
  - [ ] Create `HudiInputPartition` implementing `InputPartition`
  - [ ] Implement table metadata broadcasting
  - [ ] Compute preferred locations for data locality

- [ ] **PR 8: Reader Factory**
  - [ ] Create `HudiRowReaderFactory` for row-level reads
  - [ ] Create `HudiColumnarReaderFactory` for vectorized reads
  - [ ] Implement reader selection logic based on format/config

### Phase 4: Reader Implementation

- [ ] **PR 9: Row Reader**
  - [ ] Create `HudiRowDataReader` implementing `PartitionReader<InternalRow>`
  - [ ] Handle log file merging for MOR tables
  - [ ] Apply residual filters
  - [ ] Support `_hoodie_*` metadata columns

- [ ] **PR 10: Columnar Reader**
  - [ ] Create `HudiBatchDataReader` for vectorized reads
  - [ ] Support Parquet vectorization
  - [ ] Handle column pruning in vectorized path

### Phase 5: Advanced Features

- [ ] **PR 11: Time Travel**
  - [ ] Implement instant-time based time travel
  - [ ] Support `as.of.instant` read option
  - [ ] Handle incremental queries

- [ ] **PR 12: Runtime Filtering**
  - [ ] Implement `SupportsRuntimeV2Filtering` for broadcast join optimization
  - [ ] Filter file groups based on runtime predicates
  - [ ] Optimize partition pruning

- [ ] **PR 13: Aggregation Pushdown**
  - [ ] Implement `SupportsPushDownAggregates`
  - [ ] Support COUNT/MIN/MAX from metadata
  - [ ] Validate no pending compaction for aggregates

### Validation & Testing

- [ ] **PR 14: Test Framework**
  - [ ] Port relevant test patterns from Iceberg
  - [ ] Create `HudiScanTestBase` for scan testing
  - [ ] Test filter pushdown correctness
  - [ ] Test column pruning correctness

- [ ] **PR 15: Performance Testing**
  - [ ] Benchmark against current Hudi Spark integration
  - [ ] Validate statistics accuracy for CBO
  - [ ] Test vectorized read performance

---

## Key Takeaways

### What to Mirror Directly
1. DSv2 interface structure (TableProvider → Table → ScanBuilder → Scan → Batch → Reader)
2. Pushdown interface implementations
3. Statistics reporting pattern
4. Metadata broadcasting to executors
5. Reader factory selection logic
6. Input partition with preferred locations

### What Requires Adaptation
1. **File discovery:** Manifest-based → Timeline/FileGroup-based
2. **Delete handling:** Separate delete files → Log file merging
3. **Expression system:** Iceberg expressions → Hudi expressions
4. **Partition handling:** Transform-based → Path-based
5. **Time travel:** Snapshot/Branch → Instant time
6. **Statistics source:** Manifest stats → Metadata table

### Anti-Patterns to Avoid
1. Don't try to create "manifests" in Hudi - use native timeline
2. Don't implement DeleteFileIndex - use existing log merge logic
3. Don't copy Iceberg's expression classes - integrate with Hudi's
4. Don't ignore Hudi's MOR/COW distinction in reader selection
