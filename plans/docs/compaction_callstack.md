# Iceberg Compaction (RewriteDataFiles) — Call Stack

This document traces the complete compaction flow from user invocation through file planning, rewriting, and metadata commit.

---

## 1. High-Level Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│  USER INVOCATION                                                     │
│                                                                      │
│  Spark SQL:  CALL catalog.system.rewrite_data_files('db.table')      │
│  Java API:   SparkActions.get(spark).rewriteDataFiles(table)         │
│                .filter(expr).binPack().execute()                     │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                    ┌───────────▼─────────────────┐
                    │  1. INITIALIZATION          │   DRIVER
                    │                             │
                    │  RewriteDataFilesSparkAction│
                    │    .execute()               │
                    │       │                     │
                    │       ├─► init(snapshotId)  │
                    │       ├─► pick planner      │
                    │       │   (Shuffling vs     │
                    │       │    BinPack)         │
                    │       ├─► default runner    │
                    │       │   to BinPack if     │
                    │       │   not set           │
                    │       └─► validateAndInit-  │
                    │           Options()         │
                    └───────────┬─────────────────┘
                                │
                    ┌───────────▼─────────────────┐
                    │  2. FILE PLANNING           │   DRIVER
                    │                             │
                    │  BinPackRewriteFilePlanner  │
                    │  (or SparkShufflingData-    │
                    │   RewritePlanner)           │
                    │       │                     │
                    │       ├─► scan table files  │
                    │       ├─► filter by size/   │
                    │       │   delete thresholds │
                    │       ├─► group by          │
                    │       │   partition         │
                    │       ├─► BinPacking        │
                    │       │   algorithm         │
                    │       ├─► sort groups by    │
                    │       │   rewrite-job-order │
                    │       └─► FileRewritePlan   │
                    │           (groups of files) │
                    └───────────┬─────────────────┘
                                │
                    ┌───────────▼─────────────────┐
                    │  3. FILE REWRITING          │   SPARK JOBS
                    │                             │
                    │  For each RewriteFileGroup  │
                    │  (parallel via              │
                    │   ExecutorService):         │
                    │                             │
                    │  ┌──────────────────────┐   │
                    │  │ SparkBinPackFile-    │   │
                    │  │ RewriteRunner        │   │
                    │  │   .rewrite(group)    │   │
                    │  │                      │   │
                    │  │  READ:               │   │
                    │  │   spark.read         │   │
                    │  │    .format("iceberg")│   │
                    │  │    .load(groupId)    │   │
                    │  │                      │   │
                    │  │  WRITE:              │   │
                    │  │   df.write           │   │
                    │  │    .format("iceberg")│   │
                    │  │    .mode("append")   │   │
                    │  │    .save(groupId)    │   │
                    │  └──────────────────────┘   │
                    └───────────┬─────────────────┘
                                │
                    ┌───────────▼─────────────────┐
                    │  4. METADATA COMMIT         │   DRIVER
                    │                             │
                    │  RewriteDataFilesCommit-    │
                    │  Manager                    │
                    │    .commitOrClean()         │
                    │    (or CommitService for    │
                    │     partial progress)       │
                    │       │                     │
                    │       ▼                     │
                    │  table.newRewrite()         │
                    │    .deleteFile(old)         │
                    │    .addFile(new)            │
                    │    .commit()                │
                    │       │                     │
                    │       ▼                     │
                    │  NEW SNAPSHOT (REPLACE)     │
                    └───────────┬─────────────────┘
                                │
                    ┌───────────▼─────────────────┐
                    │  5. POST-COMPACTION         │   DRIVER (optional)
                    │                             │
                    │  if remove-dangling-deletes:│
                    │    RemoveDanglingDeletes-   │
                    │      SparkAction.execute()  │
                    └─────────────────────────────┘
```

---

## 2. Detailed Call Stack

### Phase 1: Entry Point & Initialization

```
-- Bin-pack (default strategy; no sort_order)
CALL catalog.system.rewrite_data_files(                          [Spark SQL]
  table => 'db.table',
  strategy => 'binpack',
  options => map('target-file-size-bytes','536870912'),
  where => 'date > "2024-01-01"',
  branch => 'main'             (optional)
)

-- Sort, using the table's declared sort order (no sort_order argument)
CALL catalog.system.rewrite_data_files(
  table => 'db.table',
  strategy => 'sort'
)

-- Sort with an explicit sort order
CALL catalog.system.rewrite_data_files(
  table => 'db.table',
  strategy => 'sort',
  sort_order => 'col1 ASC NULLS LAST, col2 DESC NULLS FIRST',
  options => map('target-file-size-bytes','536870912')
)

-- sort_order alone implies the sort strategy (strategy defaults to 'sort')
CALL catalog.system.rewrite_data_files(
  table => 'db.table',
  sort_order => 'col1 ASC NULLS LAST'
)

-- Z-order (strategy => 'sort', sort_order => 'zorder(...)')
CALL catalog.system.rewrite_data_files(
  table => 'db.table',
  strategy => 'sort',
  sort_order => 'zorder(c1, c2)'
)

-- Notes on procedure validation (RewriteDataFilesProcedure.checkAndApplyStrategy):
--   * strategy => 'sort' with no sort_order calls action.sort(); SparkSortFileRewriteRunner
--     throws only if the table itself has no declared sort order.
--   * sort_order with no strategy is accepted; the procedure treats strategy as 'sort'
--     and applies the parsed order (or zOrder(...) when the expression is zorder()).
--   * strategy => 'binpack' together with sort_order is rejected: the procedure calls
--     binPack() and then sort(...) on purpose to surface the action's
--     "Cannot set rewrite mode, it has already been set to BIN-PACK" error.

RewriteDataFilesProcedure.call(args)                             [spark/procedures]
  │
  └─► SparkActions.get(spark)
        .rewriteDataFiles(table)                                 [spark/actions]
          └─► new RewriteDataFilesSparkAction(spark, table)

RewriteDataFilesSparkAction                                      [spark/actions]
  │
  ├─► .filter(expression)          optional: limit to partitions
  ├─► .binPack()                   strategy: SparkBinPackFileRewriteRunner
  │   .sort()/.sort(sortOrder)     strategy: SparkSortFileRewriteRunner
  │   .zOrder(col1, col2, ...)     strategy: SparkZOrderFileRewriteRunner
  ├─► .toBranch(branch)            optional: target a non-main branch
  │
  └─► .execute()
        │
        ├─► early-exit if currentSnapshot == null
        ├─► resolve startingSnapshotId = table.snapshot(branch).snapshotId()
        │
        ├─► init(startingSnapshotId)
        │     ├─► pick planner based on runner type:
        │     │     runner instanceof SparkShufflingFileRewriteRunner
        │     │       ? new SparkShufflingDataRewritePlanner(...)
        │     │       : new BinPackRewriteFilePlanner(...)
        │     │
        │     ├─► default runner to SparkBinPackFileRewriteRunner if unset
        │     └─► validateAndInitOptions()
        │           ├─► verify option keys are in
        │           │   runner.validOptions ∪ planner.validOptions
        │           │   ∪ action VALID_OPTIONS
        │           ├─► planner.init(options)
        │           └─► runner.init(options)
        │
        └─► plan = planner.plan()
              early-exit on plan.totalGroupCount() == 0
```

### Phase 2: File Planning

```
BinPackRewriteFilePlanner.plan()                                 [core/actions]
  extends SizeBasedFileRewritePlanner
  │
  ├─► Scan table for all files:
  │     TableScan scan = table.newScan()
  │       .filter(userFilter)
  │       .caseSensitive(caseSensitive)
  │       .ignoreResiduals()
  │     if (snapshotId != null) scan = scan.useSnapshot(snapshotId)
  │     CloseableIterable<FileScanTask> tasks = scan.planFiles()
  │
  ├─► Group tasks by partition (groupByPartition):
  │     tasks whose specId != current spec id are bucketed
  │     under an empty struct (treated as unpartitioned).
  │     └─► StructLikeMap<StructLike, List<FileScanTask>>
  │
  ├─► For each partition, pick rewrite candidates (filterFiles):
  │     │
  │     ├─► Size-based selection (outsideDesiredFileSizeRange):
  │     │     file.length() < min-file-size-bytes
  │     │       (default 0.75 * target)
  │     │     file.length() > max-file-size-bytes
  │     │       (default 1.80 * target)
  │     │
  │     ├─► Delete-based selection:
  │     │     deletes.size() >= delete-file-threshold
  │     │     deleteRatio   >= delete-ratio-threshold (0.3)
  │     │     (deleteRatio counts only file-scoped deletes)
  │     │
  │     └─► If rewrite-all=true, all files pass without filtering.
  │
  ├─► BinPacking algorithm:                                      [core/util/BinPacking]
  │     BinPacking.ListPacker(maxGroupSize, lookback=1,
  │                            largestBinFirst=false,
  │                            maxItemsPerBin=maxGroupCount)
  │       .pack(candidateTasks, ContentScanTask::length)
  │         └─► PackingIterable consumes the input iterable in its
  │             existing order (no pre-sort). It keeps a sliding
  │             window of `lookback` open bins; each incoming task
  │             goes into the first open bin that still has room
  │             (bounded by max-file-group-size-bytes and
  │              max-file-group-input-files), otherwise the oldest
  │             open bin is emitted and a new bin is opened for the
  │             task. With lookback=1, this collapses to a single
  │             rolling bin that is emitted whenever the next task
  │             would overflow it.
  │             └─► returns List<List<FileScanTask>>
  │
  ├─► Filter groups (filterFileGroups):
  │     a group is kept if any of these hold:
  │       enoughInputFiles      (>= min-input-files and size > 1)
  │       enoughContent         (inputSize > target-file-size-bytes)
  │       tooMuchContent        (inputSize > max-file-size-bytes)
  │       any task tooManyDeletes
  │       any task tooHighDeleteRatio
  │
  ├─► Build RewriteFileGroups (honors max-files-to-rewrite cap):
  │     RewriteFileGroup
  │       ├─ info: FileGroupInfo(globalIdx, partitionIdx, partition)
  │       ├─ fileScanTasks: List<FileScanTask>
  │       ├─ outputSpecId: int        (output-spec-id or current spec)
  │       ├─ writeMaxFileSize: long   (target + (max-target)/2)
  │       ├─ inputSplitSize: long
  │       └─ expectedOutputFiles: int
  │
  └─► Return FileRewritePlan:
        groups sorted by RewriteFileGroup.comparator(rewriteJobOrder)
        totalGroupCount and groupsInPartition map exposed for jobs
```

`SparkShufflingDataRewritePlanner` extends `BinPackRewriteFilePlanner` and adds a `compression-factor` option that scales `inputSize` when picking the expected number of output files.

### Phase 3: File Rewriting (Per Group)

```
RewriteDataFilesSparkAction.execute() (continued)               [spark/actions]
  │
  ├─► partialProgressEnabled
  │     ? doExecuteWithPartialProgress(plan, commitManager)
  │     : doExecute(plan, commitManager)
  │
  └─► doExecute(plan, commitManager):
        │
        ├─► rewriteService =
        │     fixed-thread-pool(maxConcurrentFileGroupRewrites)
        │
        └─► Tasks.foreach(plan.groups())
              .executeWith(rewriteService)
              .stopOnFailure().noRetry()
              .run(group → rewrittenGroups.add(rewriteFiles(plan, group)))

rewriteFiles(plan, RewriteFileGroup group)                       [spark/actions]
  │
  ├─► describe job with newJobGroupInfo("REWRITE-DATA-FILES", desc)
  ├─► addedFiles = runner.rewrite(fileGroup)
  ├─► fileGroup.setOutputFiles(addedFiles)
  └─► return fileGroup

SparkBinPackFileRewriteRunner.rewrite(group)                     [spark/actions]
  extends SparkDataFileRewriteRunner
  │
  ├─► String groupId = UUID.randomUUID()
  │
  ├─► Stage files for coordinated read:
  │     tableCache.add(groupId, table)
  │     taskSetManager.stageTasks(table, groupId, fileScanTasks)
  │
  ├─► doRewrite(groupId, group)
  │     │
  │     ├─► READ phase (Spark job):
  │     │     spark.read()
  │     │       .format("iceberg")
  │     │       .option(SCAN_TASK_SET_ID, groupId)       ◄── Spark 3.5 / 4.0 only
  │     │       .option(SparkReadOptions.SPLIT_SIZE, inputSplitSize)
  │     │       .option(SparkReadOptions.FILE_OPEN_COST, "0")
  │     │       .load(groupId)
  │     │         │
  │     │         └─► path-as-groupId is resolved against the staged
  │     │             tasks/table that this runner registered before
  │     │             calling doRewrite (see "Stage files" above).
  │     │             ├─ Spark 3.5 / 4.0: SparkStagedScanBuilder picks
  │     │             │   the staged set via the SCAN_TASK_SET_ID option.
  │     │             └─ Spark 4.1+: the staged identifier is carried by
  │     │                 a rewrite-only catalog/table view
  │     │                 (SparkRewriteTableCatalog / SparkRewriteTable)
  │     │                 keyed off load(groupId), so the explicit
  │     │                 SCAN_TASK_SET_ID option is no longer set.
  │     │
  │     └─► WRITE phase (Spark job):
  │           scanDF.write()
  │             .format("iceberg")
  │             .option(REWRITTEN_FILE_SCAN_TASK_SET_ID, groupId)            ◄── Spark 3.5 / 4.0 only
  │             .option(TARGET_FILE_SIZE_BYTES, group.maxOutputFileSize())
  │             .option(DISTRIBUTION_MODE,                                   ◄── NONE if input spec
  │                     distributionMode(group).modeName())                  │   matches output spec,
  │             .option(OUTPUT_SPEC_ID, group.outputSpecId())                │   RANGE otherwise
  │             .mode("append")
  │             .save(groupId)
  │               │
  │               └─► normal write path → new Parquet files.
  │                   Spark 3.5 / 4.0 route the write through
  │                   FileRewriteCoordinator using the explicit
  │                   REWRITTEN_FILE_SCAN_TASK_SET_ID option; Spark 4.1
  │                   relies on the rewrite-catalog binding from the
  │                   matching save(groupId). In both cases the new
  │                   data files are registered with
  │                   FileRewriteCoordinator and NOT committed yet.
  │
  ├─► addedFiles = coordinator.fetchNewFiles(table, groupId)
  │     (only reached when doRewrite returns normally — on failure
  │      the exception propagates and addedFiles is never fetched)
  │
  └─► finally:
        tableCache.remove(groupId)
        taskSetManager.removeTasks(table, groupId)
        coordinator.clearRewrite(table, groupId)
        (the finally block only releases staged state; it does NOT
         fetch new files)
```

**Sort and ZOrder runners share a base class and differ from BinPack in the WRITE phase:**

```
SparkShufflingFileRewriteRunner.doRewrite()                      [spark/actions]
  abstract, extends SparkDataFileRewriteRunner
  │
  ├─► READ phase (no split size override):
  │     spark.read().format("iceberg")
  │       .option(SCAN_TASK_SET_ID, groupId)   ◄── Spark 3.5 / 4.0 only;
  │       .load(groupId)                         Spark 4.1 binds the staged
  │                                              tasks through the rewrite
  │                                              catalog at load(groupId).
  │
  ├─► sortedDF = sortedDF(scanDF, sortFunction(...))
  │     plans a logical sort + an OrderedWrite that requests an
  │     OrderedDistribution and required ordering, optionally
  │     coalescing by shuffle-partitions-per-file.
  │
  └─► WRITE phase:
        sortedDF.write().format("iceberg")
          .option(REWRITTEN_FILE_SCAN_TASK_SET_ID, groupId)   ◄── Spark 3.5 / 4.0 only
          .option(TARGET_FILE_SIZE_BYTES, group.maxOutputFileSize())
          .option(USE_TABLE_DISTRIBUTION_AND_ORDERING, "false")
          .option(OUTPUT_SPEC_ID, group.outputSpecId())
          .option(OUTPUT_SORT_ORDER_ID, matchingTableSortOrderId)
          .mode("append").save(groupId)

SparkSortFileRewriteRunner    sortOrder() returns user/table SortOrder
                              sortedDF() applies the sort directly

SparkZOrderFileRewriteRunner  sortOrder() returns Z_SORT_ORDER over a
                              synthetic binary column ICEZVALUE
                              sortedDF() interleaves byte values
                              (SparkZOrderUDF) and drops the helper column
```

### Phase 4: Metadata Commit

```
RewriteDataFilesSparkAction (continued)                          [spark/actions]
  │
  ├─► doExecute (single commit):
  │     commitManager.commitOrClean(rewrittenGroups)
  │
  └─► doExecuteWithPartialProgress (streaming commits):
        groupsPerCommit = ceil(totalGroupCount / max-commits)
        commitService = commitManager.service(groupsPerCommit)
        commitService.start()
        for each rewritten group: commitService.offer(group)
        commitService.close()
        if failedCommits > max-failed-commits: throw

RewriteDataFilesCommitManager.commitOrClean(groups)              [core/actions]
  │
  ├─► commitFileGroups(groups)
  │
  └─► on CleanableFailure: groups.forEach(abortFileGroup)
        (CommitStateUnknownException is rethrown without cleanup)

RewriteDataFilesCommitManager.commitFileGroups(groups)
  │
  ├─► Aggregate files from all groups:
  │     rewrittenDataFiles += group.rewrittenFiles()              old files
  │     addedDataFiles     += group.addedFiles()                  new files
  │     danglingDVs        += group.danglingDVs()                 orphaned DVs
  │
  ├─► Create RewriteFiles transaction:
  │     RewriteFiles rewrite = table.newRewrite()                 [api/Table]
  │       .validateFromSnapshot(startingSnapshotId)
  │       └─► new BaseRewriteFiles(...)                           [core/BaseRewriteFiles]
  │             operation() == DataOperations.REPLACE
  │
  ├─► If use-starting-sequence-number:
  │     rewrite.dataSequenceNumber(
  │       table.snapshot(startingSnapshotId).sequenceNumber())
  │
  ├─► Remove old files:
  │     rewrittenDataFiles.forEach(rewrite::deleteFile)
  │
  ├─► Add new files:
  │     addedDataFiles.forEach(rewrite::addFile)
  │
  ├─► Remove dangling DVs left over after rewriting their targets:
  │     danglingDVs.forEach(rewrite::deleteFile)
  │
  ├─► Apply commit-summary snapshot properties (commitSummary())
  │     and rewrite.toBranch(branch) when branch != null
  │
  └─► rewrite.commit()
        └─► SnapshotProducer.commit()                            [core/SnapshotProducer]
              │
              ├─► validate(base, parent):
              │     - replaced data files must not have new row-level
              │       deletes added since startingSnapshotId
              │     - files-to-delete must still exist in current snapshot
              │
              ├─► Write new manifests:
              │     ├─ manifest with DELETED entries (old files)
              │     └─ manifest with ADDED entries (new files)
              │
              ├─► Write manifest list
              │
              ├─► Create Snapshot:
              │     operation = "replace"
              │     summary includes: added-data-files,
              │                       deleted-data-files,
              │                       added-records,
              │                       deleted-records, ...
              │
              └─► TableOperations.commit(base, updated)
                    └─► atomic CAS on metadata.json
                          (retry on conflict)
```

When `doExecute` aborts (rewrite failure before commit), already-written files are deleted via `commitManager.abortFileGroup(group)` to avoid orphan data files.

### Phase 5: Post-Compaction

```
RewriteDataFilesSparkAction.execute() (continued)
  │
  └─► if remove-dangling-deletes:
        new RemoveDanglingDeletesSparkAction(spark, table).execute()
          └─► removes delete files (equality and position) that no longer
              reference any live data files; bumped count is folded into
              the returned result.removedDeleteFilesCount()
```

`ExpireSnapshots` is run independently (it is not part of `RewriteDataFiles`):

```
SparkActions.get(spark)
  .expireSnapshots(table)
  .expireOlderThan(timestampMillis)
  .execute()
    └─► removes unreferenced snapshots, which allows GC of old data
        files that were replaced by compaction
```

---

## 3. Strategy Comparison

```
┌──────────────┬─────────────────┬──────────────────┬─────────────────┐
│              │    BIN-PACK     │     SORT         │    Z-ORDER      │
├──────────────┼─────────────────┼──────────────────┼─────────────────┤
│ Goal         │ Consolidate     │ Consolidate +    │ Consolidate +   │
│              │ small files     │ sort data        │ multi-dim sort  │
├──────────────┼─────────────────┼──────────────────┼─────────────────┤
│ Shuffle      │ None (RANGE if  │ Ordered          │ Ordered         │
│              │ output spec     │ distribution +   │ distribution on │
│              │ differs)        │ sort             │ interleaved bits│
├──────────────┼─────────────────┼──────────────────┼─────────────────┤
│ Speed        │ Fastest         │ Slower (shuffle) │ Slower (shuffle)│
├──────────────┼─────────────────┼──────────────────┼─────────────────┤
│ Read benefit │ Fewer files     │ Fewer files +    │ Fewer files +   │
│              │ to open         │ better min/max   │ multi-column    │
│              │                 │ pruning on sort  │ pruning         │
│              │                 │ columns          │                 │
├──────────────┼─────────────────┼──────────────────┼─────────────────┤
│ Use when     │ Many small      │ Queries filter   │ Queries filter  │
│              │ files, no       │ on known         │ on multiple     │
│              │ specific query  │ columns          │ columns equally │
│              │ pattern         │                  │                 │
├──────────────┼─────────────────┼──────────────────┼─────────────────┤
│ Runner class │ SparkBinPack-   │ SparkSort-       │ SparkZOrder-    │
│              │ FileRewrite-    │ FileRewrite-     │ FileRewrite-    │
│              │ Runner          │ Runner           │ Runner          │
├──────────────┼─────────────────┼──────────────────┼─────────────────┤
│ Runner base  │ SparkDataFile-  │ SparkShuffling-  │ SparkShuffling- │
│              │ RewriteRunner   │ FileRewrite-     │ FileRewrite-    │
│              │                 │ Runner           │ Runner          │
├──────────────┼─────────────────┼──────────────────┼─────────────────┤
│ Planner      │ BinPackRewrite- │ SparkShuffling-  │ SparkShuffling- │
│              │ FilePlanner     │ DataRewrite-     │ DataRewrite-    │
│              │                 │ Planner          │ Planner         │
└──────────────┴─────────────────┴──────────────────┴─────────────────┘
```

`SparkShufflingFileRewriteRunner` extends `SparkDataFileRewriteRunner`, so the staging/coordinator/cleanup behavior is shared across all three strategies.

---

## 4. Key Classes Reference

| Step         | Class                              | Module           | Key Method                              |
|--------------|------------------------------------|------------------|-----------------------------------------|
| SQL entry    | `RewriteDataFilesProcedure`        | spark/procedures | `call()`                                |
| API entry    | `SparkActions`                     | spark/actions    | `rewriteDataFiles()`                    |
| Orchestrator | `RewriteDataFilesSparkAction`      | spark/actions    | `execute()`, `init()`, `rewriteFiles()` |
| API options  | `RewriteDataFiles`                 | api/actions      | option constants, `binPack/sort/zOrder` |
| Planner      | `BinPackRewriteFilePlanner`        | core/actions     | `plan()`                                |
| Planner      | `SparkShufflingDataRewritePlanner` | spark/actions    | `plan()` (sort/zorder)                  |
| Size logic   | `SizeBasedFileRewritePlanner`      | core/actions     | file selection thresholds               |
| Bin packing  | `BinPacking.ListPacker`            | core/util        | `pack()`                                |
| Plan result  | `FileRewritePlan`                  | core/actions     | `groups()`, `totalGroupCount()`         |
| File group   | `RewriteFileGroup`                 | core/actions     | files + output info, `comparator()`     |
| Runner base  | `SparkDataFileRewriteRunner`       | spark/actions    | `rewrite()` (stage + doRewrite)         |
| Shuffle base | `SparkShufflingFileRewriteRunner`  | spark/actions    | `doRewrite()` (ordered write)           |
| BinPack run  | `SparkBinPackFileRewriteRunner`    | spark/actions    | `doRewrite()`                           |
| Sort run     | `SparkSortFileRewriteRunner`       | spark/actions    | `sortOrder()`, `sortedDF()`             |
| ZOrder run   | `SparkZOrderFileRewriteRunner`     | spark/actions    | `sortOrder()`, `sortedDF()` (Z bytes)   |
| Staging      | `ScanTaskSetManager`               | spark            | `stageTasks()`, `removeTasks()`         |
| Staging      | `SparkTableCache`                  | spark            | `add()`, `remove()`                     |
| Coordination | `FileRewriteCoordinator`           | spark            | `fetchNewFiles()`, `clearRewrite()`     |
| Commit mgr   | `RewriteDataFilesCommitManager`    | core/actions     | `commitOrClean()`, `commitFileGroups()` |
| Commit svc   | `RewriteDataFilesCommitManager.CommitService` | core/actions | `start()`, `offer()`, `close()`  |
| Rewrite op   | `BaseRewriteFiles`                 | core             | `deleteFile()`, `addFile()`, `commit()` |
| Snapshot     | `SnapshotProducer`                 | core             | `commit()` → new snapshot (REPLACE)     |
| Post-step    | `RemoveDanglingDeletesSparkAction` | spark/actions    | `execute()`                             |

---

## 5. Configuration Options

Action-level options (defined on the `RewriteDataFiles` API interface):

| Option                                 | Default          | Purpose                                                  |
|----------------------------------------|------------------|----------------------------------------------------------|
| `target-file-size-bytes`               | table property `write.target-file-size-bytes` | Target output file size           |
| `max-file-group-size-bytes`            | 100 GiB          | Max total bytes per rewrite group                        |
| `max-concurrent-file-group-rewrites`   | 5                | Parallel rewrite groups (driver-side executor pool)      |
| `partial-progress.enabled`             | false            | Commit groups in batches as they finish                  |
| `partial-progress.max-commits`         | 10               | Max commits when partial progress is enabled             |
| `partial-progress.max-failed-commits`  | value of `partial-progress.max-commits` | Tolerated commit failures         |
| `use-starting-sequence-number`         | true             | Preserve starting snapshot sequence number for new files |
| `remove-dangling-deletes`              | false            | Run `RemoveDanglingDeletes` after the commit             |
| `rewrite-job-order`                    | `none`           | Group ordering: `bytes-asc/desc`, `files-asc/desc`       |
| `output-spec-id`                       | current spec     | Partition spec ID used when writing rewritten files      |

Planner-level options (size-based and bin-pack planners):

| Option                       | Default                     | Purpose                                                  |
|------------------------------|-----------------------------|----------------------------------------------------------|
| `min-file-size-bytes`        | 0.75 × target               | Files smaller than this are rewrite candidates           |
| `max-file-size-bytes`        | 1.80 × target               | Files larger than this are rewrite candidates            |
| `min-input-files`            | 5                           | Min files in a group to justify rewrite                  |
| `rewrite-all`                | false                       | Force rewrite of every input file                        |
| `delete-file-threshold`      | `Integer.MAX_VALUE`         | Rewrite files whose attached delete count ≥ N            |
| `delete-ratio-threshold`     | 0.3                         | Rewrite files whose deleted-row ratio ≥ this fraction    |
| `max-files-to-rewrite`       | unset (rewrite all)         | Cap total number of files included in the plan           |

> Internal-only planner constant — **not** accepted by `RewriteDataFilesSparkAction`:
> `SizeBasedFileRewritePlanner.MAX_FILE_GROUP_INPUT_FILES` (`"max-file-group-input-files"`,
> default `Long.MAX_VALUE`) is read by `maxGroupCount()` to bound the `BinPacking.ListPacker`
> max-items-per-bin, but it is omitted from `SizeBasedFileRewritePlanner.validOptions()`, so
> passing it through the Spark action's `options` map fails the action's unknown-key check.

Shuffling planner adds:

| Option                | Default | Purpose                                                          |
|-----------------------|---------|------------------------------------------------------------------|
| `compression-factor`  | 1.0     | Scales `inputSize` when estimating expected output file count    |

Sort / Z-order runners (extend `SparkShufflingFileRewriteRunner`):

| Option                       | Default                              | Purpose                                                  |
|------------------------------|--------------------------------------|----------------------------------------------------------|
| `shuffle-partitions-per-file`| 1                                    | Split each output file across N shuffle partitions (requires Iceberg Spark extensions when > 1) |

Z-order runner adds:

| Option                       | Default                              | Purpose                                                  |
|------------------------------|--------------------------------------|----------------------------------------------------------|
| `max-output-size`            | `Integer.MAX_VALUE`                  | Max interleaved Z-value byte length                      |
| `var-length-contribution`    | `ZOrderByteUtils.PRIMITIVE_BUFFER_SIZE` | Bytes considered per variable-length input column     |
