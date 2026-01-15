# ClickHouse Health Audit Report

**Cluster:** ai-demo  
**Host:** chi-ai-ai-0-0-0  
**Version:** 25.12.3.21  
**Report Time:** 2026-01-14T20:22 UTC  
**Uptime:** 7 minutes 45 seconds (recently restarted)

---

## Executive Summary

| Severity | Finding | Impact |
|----------|---------|--------|
| 🔴 **CRITICAL** | Patch Parts Bug (v25.12) | Merges blocked, parts accumulating |
| 🔴 **CRITICAL** | MaxPartCountForPartition = 122 | Approaching 300 limit, insert failures imminent |
| 🟠 **MAJOR** | 1,140 parts in dim_practitioners | 999 are unmergeable patch parts |
| 🟡 **MODERATE** | System trace_log consuming 3.4GB | Log rotation may be needed |
| 🟢 **OK** | Disk space 84.4% free | Healthy |
| 🟢 **OK** | Memory usage 20.3% | Healthy |

---

## Critical Finding: Patch Parts Merge Bug

### Evidence

The cluster is affected by the **known ClickHouse 25.12 patch parts bug** affecting lightweight updates. This is the bug you've been investigating.

**Error Pattern:**
```
Code: 49. DB::Exception: Block structure mismatch in patch parts stream: 
different names of columns:
  specialty LowCardinality(String)
  last_updated DateTime
```

**Error Frequency (last 24h):**
| Error Type | Count |
|------------|-------|
| Block structure mismatch in patch parts | 522 |
| Socket/connection errors (merge coordination) | 19,151 |

### Affected Tables

| Table | Total Parts | Patch Parts | Regular Parts | Severity |
|-------|-------------|-------------|---------------|----------|
| fhir.dim_practitioners | 1,140 | 999 (87.6%) | 141 | 🔴 CRITICAL |
| fhir.dim_organizations | 93 | 69 (74.2%) | 24 | 🟠 MAJOR |
| fhir.dim_locations | 91 | 67 (73.6%) | 24 | 🟠 MAJOR |
| fhir.fact_observations | 69 | 4 | 65 | 🟡 MODERATE |
| fhir.fact_procedures | 72 | 4 | 68 | 🟡 MODERATE |

### Partition Breakdown (dim_practitioners)

The patch parts create separate partitions that cannot merge:
| Partition | Parts | Size |
|-----------|-------|------|
| patch-f18f7271629a324b0d26b6ad0b83a6c2-202312 | 122 | 2.54 MiB |
| 202312 (regular) | 118 | 9.37 MiB |
| patch-116ec7151bbfe7ea649b23811e916438-202406 | 58 | 356 KiB |
| ... (many more patch partitions) | | |

### Root Cause

The table `fhir.dim_practitioners` uses:
- `enable_block_number_column = 1` and `enable_block_offset_column = 1` settings
- JSON column with type hints: `JSON(active UInt8, family_name String, ...)`
- Lightweight updates (`ALTER UPDATE`) creating patch parts

The bug causes a **column order mismatch** during merge when patch parts reference columns in different order (`specialty` vs `last_updated`).

---

## Schema Issues

### Table Design Review: fhir.dim_practitioners

```sql
ENGINE = ReplicatedMergeTree
PARTITION BY toYYYYMM(snowflakeIDToDateTime(practitioner_id))
ORDER BY practitioner_id
SETTINGS enable_block_number_column = 1, enable_block_offset_column = 1
```

**Issues Identified:**

1. **Complex Partition Key**: Using `snowflakeIDToDateTime()` extracts month from ID, creating many partitions
2. **Lightweight Updates Enabled**: The `enable_block_number_column` + `enable_block_offset_column` settings enable patch parts for updates
3. **JSON Column**: The typed JSON column (`d JSON(...)`) with DEFAULT expressions from `d.*` paths adds complexity

### All User Tables by Parts Count

| Database | Table | Parts | Tiny Parts (<16MB) | Size |
|----------|-------|-------|-------------------|------|
| fhir | dim_practitioners | 1,140 | 1,140 (100%) | 29.43 MiB |
| fhir | dim_organizations | 93 | 93 (100%) | 799.72 KiB |
| fhir | dim_locations | 91 | 91 (100%) | 4.49 MiB |
| fhir | fact_procedures | 72 | 19 (26%) | 9.22 GiB |
| fhir | fact_observations | 69 | 19 (28%) | 10.75 GiB |

---

## Storage Analysis

### Disk Usage

| Disk | Total | Free | Free % | Status |
|------|-------|------|--------|--------|
| default | 196.68 GiB | 166.07 GiB | 84.4% | 🟢 Healthy |

### Largest Tables

| Database | Table | Size | Parts | Avg Part |
|----------|-------|------|-------|----------|
| fhir | fact_observations | 10.75 GiB | 69 | 159.52 MiB |
| fhir | fact_procedures | 9.22 GiB | 72 | 131.10 MiB |
| system | trace_log_0 | 2.97 GiB | 41 | 74.12 MiB |
| fhir | fact_encounters | 1.87 GiB | 38 | 50.48 MiB |
| fhir | fact_medications | 1.76 GiB | 44 | 40.90 MiB |

### System Log Tables

| Table | Size | Parts | Oldest | Newest |
|-------|------|-------|--------|--------|
| trace_log_0 | 2.97 GiB | 41 | Jan 7 | Jan 14 |
| trace_log | 432 MiB | 14 | Jan 13 | Jan 14 |
| processors_profile_log | 171.75 MiB | 8 | Jan 7 | Jan 14 |
| part_log | 148.26 MiB | 26 | Jan 7 | Jan 14 |
| metric_log | 119.85 MiB | 8 | Jan 9 | Jan 14 |

**Recommendation:** Consider reducing TTL on `trace_log_0` (consuming 2.97 GiB).

---

## Performance Metrics

### Current Activity

| Metric | Value |
|--------|-------|
| Running Queries | 1 |
| Running Merges | 0 |
| Replication Sends | 0 |
| Replication Fetches | 0 |
| **MaxPartCountForPartition** | **122** ⚠️ |

### Merge Activity (Last 24h)

| Hour | Merges Completed | Merge Starts |
|------|-----------------|--------------|
| Jan 14 20:00 | 233 | 230 |
| Jan 13 23:00 | 99 | 98 |
| Jan 13 22:00 | 2,385 | 2,386 |
| Jan 13 21:00 | 2,147 | 2,147 |
| Jan 13 20:00 | 1,466 | 1,466 |

Merges are running for regular parts, but **patch parts cannot merge** due to the bug.

---

## Recommended Actions

### Immediate (Critical)

1. **Upgrade ClickHouse** to a version where the patch parts bug is fixed (check GitHub issues for 25.12 patch or wait for 25.13)

2. **Workaround Options:**
   
   a. **OPTIMIZE FINAL** (may not work due to bug):
   ```sql
   OPTIMIZE TABLE fhir.dim_practitioners FINAL;
   ```
   
   b. **Disable lightweight updates** for affected tables (prevents new patch parts):
   ```sql
   ALTER TABLE fhir.dim_practitioners 
   MODIFY SETTING enable_block_number_column = 0, 
                  enable_block_offset_column = 0;
   ```
   Note: Existing patch parts will remain until manually resolved.
   
   c. **Recreate table** without patch parts:
   ```sql
   -- Export data
   INSERT INTO fhir.dim_practitioners_new
   SELECT * FROM fhir.dim_practitioners;
   
   -- Swap tables
   RENAME TABLE fhir.dim_practitioners TO fhir.dim_practitioners_backup,
                fhir.dim_practitioners_new TO fhir.dim_practitioners;
   ```

### Short-term

3. **Monitor MaxPartCountForPartition** - current value is 122, limit is 300:
   ```sql
   SELECT value FROM system.asynchronous_metrics 
   WHERE metric = 'MaxPartCountForPartition';
   ```

4. **Review repro_patchparts database** - appears to be your test database for this bug

### Medium-term

5. **Reduce system log retention** for trace_log_0 (consuming 3GB):
   ```sql
   -- Check current TTL
   SELECT name, engine_full FROM system.tables 
   WHERE database = 'system' AND name LIKE 'trace%';
   ```

6. **Consider schema redesign** for dimension tables:
   - Avoid lightweight updates for dimension tables
   - Use regular INSERT + ReplacingMergeTree for SCD Type 1
   - Reserve patch parts for high-frequency update scenarios only

---

## Cluster Configuration

| Setting | Value |
|---------|-------|
| ZooKeeper | Active (zookeeper-c7095-0:2181) |
| Cluster Macro | Working (`{cluster}`) |
| Session Uptime | 456 seconds |
| Replica Path | `/clickhouse/{cluster}/tables/{shard}/{uuid}` |

---

## Appendix: Key Queries Used

```sql
-- Check patch parts status
SELECT database, table,
    countIf(partition_id LIKE 'patch-%') AS patch_partitions,
    countIf(partition_id NOT LIKE 'patch-%') AS regular_partitions
FROM system.parts WHERE active
GROUP BY database, table
HAVING patch_partitions > 0;

-- Monitor merge errors
SELECT count(), substring(message, position(message, 'Code:'), 100) AS error_type
FROM system.text_log
WHERE event_time > now() - INTERVAL 1 HOUR
  AND level = 'Error' AND message LIKE '%patch%'
GROUP BY error_type;

-- Check parts per partition (for 300 limit)
SELECT database, table, partition_id, count() AS parts
FROM system.parts WHERE active
GROUP BY database, table, partition_id
ORDER BY parts DESC LIMIT 20;
```

---

**Report Generated:** 2026-01-14T20:22 UTC  
**Audit Duration:** ~2 minutes  
**Agents Used:** overview, schema, storage, merges, errors, text_log
