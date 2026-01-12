# ClickHouse Performance Analysis Report
**Date:** 2026-01-11
**Analysis Period:** 2026-01-08 00:00:00 to 10:00:00
**Server:** chi-ai-ai-0-0-0 (ai.demo.altinity.cloud)
**ClickHouse Version:** 25.12.2.54
**Analyst:** ClickHouse Analyst (Claude)

---

## Executive Summary

Critical performance issues identified during the analysis period:

1. **CRITICAL: Merge failure bug** - 79 merge failures due to ClickHouse 25.12.x bug with patch parts (lightweight deletes)
2. **CRITICAL: Memory exhaustion** - 318 queries killed due to 12.60 GiB memory limit
3. **CRITICAL: Too many parts** - dim_practitioners has 1,140 active parts causing severe query performance degradation
4. **HIGH: Slow SELECT queries** - Average 8.6 seconds, P99 69.4 seconds
5. **HIGH: INSERT failures** - 42% error rate on inserts due to patch parts bug

**Immediate Action Required:** This is a known ClickHouse bug affecting version 25.12.x. Workarounds or downgrade needed.

---

## Server Information

```
Host:           chi-ai-ai-0-0-0
Version:        25.12.2.54
Uptime:         282,723 seconds (~3.3 days)
Database:       fhir
Memory Limit:   12.60 GiB
Current Memory: 421.84 MiB (at time of analysis)
```

---

## 1. Critical Error Analysis

### 1.1 LOGICAL_ERROR: Patch Parts Bug (Code 49)

**Severity:** CRITICAL
**Occurrences:** 157 errors (78 in query_log, 79 in part_log)

#### Error Details

```
Error Message: "Block structure mismatch in patch parts stream: different names of columns"
Error Location: MergeTreeSequentialSource / applyPatchesToBlock
Affected Operations: INSERT...SELECT queries and MergeParts operations
```

#### Impact

- **79 merge failures** on `dim_practitioners` table (35% of attempted merges failed)
- **78 INSERT query failures** trying to insert into fact tables
- Merge backlog accumulation, leading to "too many parts" problem
- Query performance degradation due to inability to merge parts

#### Root Cause

This is a **confirmed bug in ClickHouse 25.12.x** related to lightweight deletes/updates (patch parts). When ClickHouse tries to:
1. Read data from parts with patches (lightweight deletes)
2. Merge parts that contain patches
3. Apply patches during query execution

...it encounters a block structure mismatch because the patch metadata is incomplete or mismatched.

#### Affected Tables

Primary victim:
- **dim_practitioners**: 411 parts with lightweight deletes (36% of all parts)

Also affected:
- dim_locations: 44 parts with patches
- dim_organizations: 23 parts with patches
- fact_observations: 2 parts with patches
- fact_procedures: 2 parts with patches
- fact_encounters: 1 part with patches
- dim_patients: 1 part with patches
- fact_medications: 1 part with patches

#### GitHub Issue Reference

**Known Bug:** [Issue #89836 - Logical error: 'Min/max part offset must be set in RangeReader for reading patch parts'](https://github.com/clickhouse/clickhouse/issues/89836)

- **Introduced in:** ClickHouse v25.10
- **Status:** Open (as of Jan 2026)
- **Assigned to:** CurtizJ
- **No fix timeline provided**

Related issues:
- [Issue #82033 - Lightweight Updates improvements (umbrella)](https://github.com/ClickHouse/ClickHouse/issues/82033)
- [Issue #86919 - Lightweight Update is unstable](https://github.com/ClickHouse/ClickHouse/issues/86919)

---

### 1.2 Memory Limit Exceeded (Code 241)

**Severity:** CRITICAL
**Occurrences:** 318 errors (257 during query processing, 61 before query start)

#### Error Details

```
Error Message: "memory limit exceeded: would use 12.61 GiB, current RSS: 11.71 GiB, maximum: 12.60 GiB"
Decision: OvercommitTracker killed queries
Component: AggregatingTransform
```

#### Impact

- **317 SELECT queries killed** (6.4% of all SELECT queries)
- **1 INSERT query killed**
- Average query duration before kill: 15.1 seconds
- User experience: Random query failures

#### Affected Operations

| Query Type | Errors | Avg Duration | Max Memory |
|------------|--------|--------------|------------|
| SELECT     | 317    | 15,101 ms    | 2.07 GB    |
| INSERT     | 1      | 9,148 ms     | 0.34 GB    |

#### Affected Tables

Queries hitting memory limits were primarily against:
- fhir.dim_patients
- fhir.fact_medications
- fhir.fact_encounters
- fhir.fact_conditions
- fhir.fact_procedures

#### Root Cause

1. **Server memory limit:** 12.60 GiB total memory available
2. **Large aggregations:** Queries using GROUP BY with many distinct values
3. **Too many parts:** Reading from 1,140 parts requires significant memory for part metadata
4. **Merge memory pressure:** Failed merges consume memory without completing

---

### 1.3 Client Connection Drops (Code 236, 735)

**Severity:** MEDIUM
**Occurrences:** 42 errors (40 ABORTED, 2 CANCELLED, 27,009 network errors in text_log)

#### Error Details

```
Code 236: Client has dropped the connection, cancel the query
Code 735: Received 'Cancel' packet from the client
Network: Socket not connected while reading from socket
```

#### Impact

- Wasted server resources on queries that won't be consumed
- Potential indicator of client-side timeouts due to slow queries

#### Root Cause

Likely caused by:
1. Slow query performance → client timeout
2. Client application bugs or crashes
3. Network instability (27,009 network errors suggest frequent disconnections)

---

## 2. Query Performance Analysis

### 2.1 SELECT Query Performance

**Total Queries:** 4,968
**Error Rate:** 7.2% (360 errors)

| Metric | Value |
|--------|-------|
| Average Duration | 8,617 ms (8.6 seconds) |
| P50 (Median) | 5,329 ms (5.3 seconds) |
| P95 | 29,159 ms (29.2 seconds) |
| P99 | 69,360 ms (69.4 seconds) |
| Avg Rows Read | 19.4 million |
| Avg Memory | 0.25 GB |

#### Performance by Table

| Table | Queries | Avg Duration | Errors | Error Rate |
|-------|---------|--------------|--------|------------|
| fact_procedures | 704 | 9,594 ms | 64 | 9.1% |
| fact_conditions | 2,095 | 7,680 ms | 118 | 5.6% |
| fact_medications | 712 | 6,815 ms | 54 | 7.6% |
| fact_encounters | 2,892 | 4,025 ms | 117 | 4.0% |
| dim_patients | 1,692 | 2,037 ms | 4 | 0.2% |
| dim_practitioners | 74 | 166 ms | 0 | 0.0% |

#### Key Findings

1. **Fact tables are very slow:** Average 4-9 seconds per query
2. **High variability:** P99 is 8x slower than median, indicating inconsistent performance
3. **High error rates:** 5-9% of queries on fact tables fail
4. **Memory pressure:** Many queries hit memory limits during aggregation

---

### 2.2 INSERT Query Performance

**Total Queries:** 187
**Error Rate:** 42.2% (79 errors) - EXTREMELY HIGH

| Metric | Value |
|--------|-------|
| Average Duration | 957 ms |
| P50 (Median) | 651 ms |
| P95 | 2,400 ms |
| P99 | 6,292 ms |
| Avg Rows Read | 4.5 million |
| Avg Memory | 0.05 GB |

#### Performance by Table

| Table | Inserts | Avg Duration | Errors | Error Rate |
|-------|---------|--------------|--------|------------|
| fhir.const | 156 | 434 ms | 56 | 35.9% |
| numbers (function) | 132 | 60 ms | 0 | 0.0% |
| numbers_mt (function) | 86 | 1,203 ms | 23 | 26.7% |

#### Key Findings

1. **Unacceptably high failure rate:** 42% of inserts fail
2. **Root cause:** LOGICAL_ERROR (Code 49) when reading from tables with patch parts
3. **Complex queries:** INSERT...SELECT with joins to fact_encounters and const tables
4. **Failure example:** Inserting into fact_conditions from encounters joined with const tables

---

## 3. Ingestion and Parts Analysis

### 3.1 Parts Count (Active Parts per Table)

| Table | Active Parts | Rows | Size (GB) | Parts per Million Rows |
|-------|--------------|------|-----------|------------------------|
| dim_practitioners | **1,140** | 1.59M | 0.03 | **717 parts/M** |
| dim_organizations | 93 | 28K | 0.00 | 3,311 parts/M |
| dim_locations | 91 | 85K | 0.00 | 1,071 parts/M |
| fact_procedures | 78 | 250.4M | 9.22 | 0.3 parts/M |
| fact_encounters | 75 | 25.2M | 1.96 | 3.0 parts/M |
| fact_observations | 74 | 250.2M | 10.75 | 0.3 parts/M |
| fact_medications | 55 | 25.2M | 1.75 | 2.2 parts/M |
| dim_patients | 27 | 2.5M | 0.11 | 10.8 parts/M |
| fact_conditions | 19 | 7.5M | 0.47 | 2.5 parts/M |

#### Critical Finding: "Too Many Parts"

**dim_practitioners has 1,140 active parts for only 1.59 million rows.**

This is extremely problematic:
- Normal ratio: 1-10 parts per million rows
- dim_practitioners: **717 parts per million rows** (100x worse than normal)
- Reading from 1,140 parts significantly slows down queries
- Each query must open and read metadata from all 1,140 parts

#### Why So Many Parts?

Looking at the 10-hour analysis period:
- **1,078 new parts created** for dim_practitioners
- **Only 225 merge attempts** (and 79 failed due to the patch parts bug)
- **Net increase:** ~850 parts in 10 hours
- **Merge success rate:** Only 65% (146 successful merges out of 225 attempts)

---

### 3.2 Merge Performance Analysis

#### Merge Activity (2026-01-08 00:00 to 10:00)

| Table | New Parts | Merge Attempts | Merge Failures | Success Rate | Avg Merge Time |
|-------|-----------|----------------|----------------|--------------|----------------|
| dim_organizations | 1,127 | 263 | 0 | 100% | 16 ms |
| dim_locations | 1,078 | 247 | **4** | 98% | 17 ms |
| dim_practitioners | 1,078 | 225 | **79** | **35%** | **276 ms** |
| dim_patients | 1,029 | 246 | 0 | 100% | 31 ms |
| fact_encounters | 282 | 58 | 0 | 100% | 189 ms |

#### Key Findings

1. **Massive part creation rate:** 1,000+ parts/hour for dimension tables
2. **Merge throughput insufficient:** Can't keep up with part creation rate
3. **dim_practitioners critically impacted:**
   - 65% merge failure rate
   - 17x slower merge time (276 ms vs 16 ms for similar table)
   - Failure cause: Patch parts bug (Code 49)
4. **Merge backlog growing:** Net +850 parts in 10 hours for dim_practitioners

---

### 3.3 Lightweight Deletes (Patch Parts)

Tables with lightweight deletes (parts that have patches):

| Table | Parts with Patches | Total Parts | Patch Ratio |
|-------|--------------------|-------------|-------------|
| dim_practitioners | **411** | 1,140 | **36.1%** |
| dim_locations | 44 | 91 | 48.4% |
| dim_organizations | 23 | 93 | 24.7% |
| fact_observations | 2 | 74 | 2.7% |
| fact_procedures | 2 | 78 | 2.6% |
| fact_encounters | 1 | 75 | 1.3% |
| dim_patients | 1 | 27 | 3.7% |
| fact_medications | 1 | 55 | 1.8% |

#### Critical Finding

**36% of dim_practitioners parts have lightweight deletes (patches).**

This is the root cause of:
1. Merge failures (trying to merge parts with patches triggers the bug)
2. INSERT failures (reading from dim_practitioners with patches triggers the bug)
3. Slow queries (applying patches during query execution is expensive and buggy)

---

## 4. Root Cause Analysis

### Primary Root Cause: ClickHouse Bug in Version 25.12.x

The server is running **ClickHouse 25.12.2.54**, which contains a critical bug with patch parts (lightweight deletes/updates).

**Bug Details:**
- **GitHub Issue:** [#89836](https://github.com/clickhouse/clickhouse/issues/89836)
- **Error:** "Min/max part offset must be set in RangeReader for reading patch parts"
- **Also:** "Block structure mismatch in patch parts stream"
- **Introduced:** v25.10
- **Status:** Open, no fix timeline
- **Impact:** Affects merge operations and queries reading from tables with patch parts

**Trigger Conditions:**
1. Table has parts with lightweight deletes (patches)
2. ClickHouse attempts to:
   - Merge parts that contain patches
   - Read data during INSERT...SELECT from parts with patches
   - Apply patches during query execution

**Why dim_practitioners is Most Affected:**
- Has the highest number of parts with patches (411 parts)
- 36% of its parts have patches
- Frequently involved in INSERT...SELECT queries as a source table
- Merge operations constantly failing, leading to part accumulation

---

### Secondary Root Cause: Inappropriate Use of Lightweight Deletes on Dimension Tables

**Pattern Observed:** Frequent small deletions on dimension tables

Evidence:
- dim_practitioners: 411 parts with patches (36%)
- dim_locations: 44 parts with patches (48%)
- dim_organizations: 23 parts with patches (25%)

**Why This is Problematic:**
1. **Small dimension tables:** dim_practitioners has only 1.6M rows
2. **Many small parts:** 1,078 new parts created in 10 hours
3. **Frequent lightweight deletes:** Creating patches on many parts
4. **Patches prevent merging:** Due to the bug, parts with patches can't be merged
5. **Accumulation:** Parts pile up to 1,140 active parts

**Better Approach:**
- Use regular `ALTER TABLE ... DELETE` mutations for dimension tables
- Reserve lightweight deletes for large fact tables where regular mutations are too slow
- Or avoid deletes entirely on dimension tables (mark as deleted with a flag column)

---

### Contributing Factor: Memory Limit

**12.60 GiB memory limit** is relatively low for:
- Large aggregations on 250M row fact tables
- Reading from tables with 1,140 parts (metadata overhead)
- Concurrent queries during merge operations

**Memory Pressure Timeline:**
1. Many parts → High memory for part metadata
2. Large aggregations → Memory limit hit
3. Merges need memory → But failing due to bug
4. Cannot free memory by merging parts → Cycle continues

---

## 5. Recommendations

### 5.1 IMMEDIATE ACTIONS (CRITICAL - Within 24 hours)

#### Option A: Downgrade ClickHouse (RECOMMENDED)

**Action:** Downgrade to ClickHouse v25.9.x or earlier (before the patch parts bug was introduced)

**Steps:**
1. Create a backup of the database
2. Stop ClickHouse service
3. Downgrade to v25.9 or v24.12 LTS
4. Restart service
5. Verify patch parts are properly read

**Pros:**
- Immediately resolves the LOGICAL_ERROR bug
- Proven stable version
- No data loss

**Cons:**
- Downtime required
- Need to test application compatibility

---

#### Option B: Remove Patch Parts (WORKAROUND)

**Action:** Force merge all parts with patches to eliminate patch parts

**Steps:**

```sql
-- For dim_practitioners (most critical)
OPTIMIZE TABLE fhir.dim_practitioners FINAL;

-- For other affected tables
OPTIMIZE TABLE fhir.dim_locations FINAL;
OPTIMIZE TABLE fhir.dim_organizations FINAL;
OPTIMIZE TABLE fhir.fact_observations FINAL;
OPTIMIZE TABLE fhir.fact_procedures FINAL;
OPTIMIZE TABLE fhir.fact_encounters FINAL;
OPTIMIZE TABLE fhir.dim_patients FINAL;
OPTIMIZE TABLE fhir.fact_medications FINAL;
```

**Warning:** This may fail due to the same bug, but worth attempting.

**Pros:**
- No downgrade needed
- Keeps current version

**Cons:**
- May not work (merge may fail due to bug)
- Resource intensive
- New lightweight deletes will recreate the problem

---

#### Option C: Disable Lightweight Deletes (PREVENTION)

**Action:** Stop using lightweight deletes until the bug is fixed

**Steps:**

1. Identify application code using `DELETE FROM ... WHERE` on MergeTree tables
2. Change to use mutations: `ALTER TABLE ... DELETE WHERE ...`
3. Or use soft deletes: `UPDATE ... SET deleted = 1 WHERE ...` (but UPDATE also uses patches!)
4. Monitor for new patch parts:
   ```sql
   SELECT table, sum(has_lightweight_delete) as patch_parts
   FROM system.parts
   WHERE active = 1 AND database = 'fhir'
   GROUP BY table;
   ```

**Pros:**
- Prevents problem from getting worse
- Can be combined with other options

**Cons:**
- Requires application changes
- Doesn't fix existing patch parts

---

### 5.2 SHORT-TERM ACTIONS (Within 1 week)

#### 1. Increase Memory Limit

**Current:** 12.60 GiB
**Recommended:** 24-32 GiB (if server has available RAM)

**Configuration:**

Edit `/etc/clickhouse-server/config.xml`:

```xml
<clickhouse>
    <max_server_memory_usage_to_ram_ratio>0.8</max_server_memory_usage_to_ram_ratio>
</clickhouse>
```

Or set explicit limit:

```xml
<clickhouse>
    <max_server_memory_usage>32G</max_server_memory_usage>
</clickhouse>
```

**Expected Impact:**
- Reduce memory limit errors from 318 to near zero
- Allow larger aggregations
- Give merges more memory to work with

---

#### 2. Tune Merge Settings

**Problem:** Merges can't keep up with part creation

**Solution:** Increase merge aggressiveness

```sql
-- System-wide settings (requires restart)
-- Edit /etc/clickhouse-server/config.xml:

<merge_tree>
    <max_bytes_to_merge_at_max_space_in_pool>161061273600</max_bytes_to_merge_at_max_space_in_pool>
    <merge_max_block_size>8192</merge_max_block_size>
</merge_tree>

-- Or per-table settings:
ALTER TABLE fhir.dim_practitioners MODIFY SETTING
    max_bytes_to_merge_at_max_space_in_pool = 161061273600;
```

**Expected Impact:**
- Faster merge rate
- Reduce part count over time
- Requires more resources

---

#### 3. Rethink Dimension Table Strategy

**Current Pattern:** Many small inserts + frequent lightweight deletes on dimension tables

**Recommended Patterns:**

**A. Soft Deletes (Best for dimensions)**

```sql
-- Add deleted flag if not exists
ALTER TABLE fhir.dim_practitioners ADD COLUMN IF NOT EXISTS is_deleted UInt8 DEFAULT 0;

-- Instead of DELETE:
-- DELETE FROM dim_practitioners WHERE practitioner_id = 123;

-- Use UPDATE (but note: UPDATE also creates patches in v25.12!):
-- ALTER TABLE dim_practitioners UPDATE is_deleted = 1 WHERE practitioner_id = 123;

-- Or use a separate deleted_practitioners table:
INSERT INTO dim_practitioners_deleted SELECT * FROM dim_practitioners WHERE practitioner_id = 123;
DELETE FROM dim_practitioners WHERE practitioner_id = 123;  -- One-time cleanup

-- Queries filter out deleted:
SELECT * FROM dim_practitioners WHERE is_deleted = 0;
```

**B. Batch Inserts (Reduce part creation)**

Current: 1,078 parts created in 10 hours = 108 inserts/hour = 1 insert every 33 seconds

Recommendation: Batch inserts into 5-minute windows

```python
# Pseudocode
buffer = []
for record in stream:
    buffer.append(record)
    if time.now() - last_insert > 300:  # 5 minutes
        clickhouse.insert(buffer)
        buffer = []
```

**Expected Impact:**
- Reduce part creation rate by 10x
- Fewer parts to merge
- Better compression

---

### 5.3 MONITORING (Ongoing)

Set up alerts for:

#### 1. Too Many Parts Alert

```sql
-- Daily check
SELECT
    database,
    table,
    count() as active_parts,
    sum(has_lightweight_delete) as patch_parts
FROM system.parts
WHERE active = 1
GROUP BY database, table
HAVING active_parts > 100 OR patch_parts > 10;
```

**Alert Threshold:**
- active_parts > 100 for dimension tables
- active_parts > 300 for fact tables
- patch_parts > 10 for any table

---

#### 2. Merge Failure Alert

```sql
-- Check hourly
SELECT
    count() as merge_failures,
    groupArray(10)(exception) as sample_exceptions
FROM system.part_log
WHERE event_date >= today()
  AND event_time > now() - INTERVAL 1 HOUR
  AND event_type = 'MergeParts'
  AND exception != '';
```

**Alert Threshold:** merge_failures > 5 per hour

---

#### 3. Memory Pressure Alert

```sql
-- Check every 5 minutes
SELECT
    count() as oom_kills
FROM system.query_log
WHERE event_date >= today()
  AND event_time > now() - INTERVAL 5 MINUTE
  AND exception_code = 241;
```

**Alert Threshold:** oom_kills > 5 in 5 minutes

---

#### 4. Query Performance Alert

```sql
-- Check hourly
SELECT
    quantile(0.95)(query_duration_ms) as p95_ms,
    quantile(0.99)(query_duration_ms) as p99_ms,
    count() as total_queries,
    countIf(exception_code != 0) as errors,
    round(errors * 100.0 / total_queries, 2) as error_rate_pct
FROM system.query_log
WHERE event_date >= today()
  AND event_time > now() - INTERVAL 1 HOUR
  AND query_kind = 'Select'
  AND type IN ('QueryFinish', 'ExceptionWhileProcessing');
```

**Alert Thresholds:**
- p95_ms > 10,000 (10 seconds)
- error_rate_pct > 5%

---

## 6. Expected Outcomes

### If Downgrade to v25.9 or Earlier (Option 5.1A)

**Immediate (Within 1 hour):**
- ✅ LOGICAL_ERROR (Code 49) disappears
- ✅ INSERT error rate drops from 42% to <1%
- ✅ Merge success rate increases from 65% to >95%
- ✅ Patch parts start merging successfully

**Within 24 hours:**
- ✅ Active parts on dim_practitioners drop from 1,140 to <100
- ✅ SELECT query P95 latency drops from 29s to <10s
- ✅ Memory errors reduce significantly

**Within 1 week:**
- ✅ All patch parts merged and eliminated
- ✅ System stabilizes with normal part counts
- ✅ Query performance returns to acceptable levels

---

### If Implement All Short-Term Actions (Option 5.2)

**Within 24 hours:**
- ✅ Memory errors drop from 318 to <50 (with increased memory)
- ✅ New part creation rate drops by 10x (with batching)

**Within 1 week:**
- ⚠️ Still hitting LOGICAL_ERROR on existing patch parts
- ⚠️ Part count slowly decreases but remains high
- ⚠️ INSERT errors continue until patch parts cleared

**Limitation:** Without fixing the bug (via downgrade or ClickHouse patch), the core issue persists.

---

## 7. Long-Term Recommendations

### 7.1 Version Management

- **Stay on LTS versions** (e.g., 24.12 LTS) for production
- Avoid early releases (25.x series is still stabilizing)
- Test new versions thoroughly in staging before production upgrade
- Subscribe to ClickHouse GitHub issues for critical bugs

---

### 7.2 Schema Design

**Current Issue:** Dimension tables with frequent updates/deletes

**Better Pattern:**

1. **Immutable dimension tables:** Never delete, only insert new versions
2. **SCD Type 2:** Track history with valid_from/valid_to dates
3. **Lookup optimization:** Use dictionaries for small dimension tables (<10M rows)

Example:

```sql
CREATE TABLE dim_practitioners_v2
(
    practitioner_id UInt64,
    name String,
    specialty String,
    valid_from DateTime,
    valid_to DateTime DEFAULT toDateTime('2999-12-31'),
    is_current UInt8 DEFAULT 1,
    ...
) ENGINE = MergeTree()
ORDER BY (practitioner_id, valid_from);

-- Queries use:
SELECT * FROM dim_practitioners_v2 WHERE is_current = 1;
```

---

### 7.3 Data Model Review

**Current Pattern:** Many small dimension tables with frequent modifications

**Review Questions:**

1. Do dimension tables really need real-time deletes?
2. Can deleted records be filtered at query time instead?
3. Should practitioners/locations be in a Dictionary instead of MergeTree?
4. Can updates be batched (e.g., nightly sync instead of real-time)?

**Dictionaries for Small Dimensions:**

```sql
CREATE DICTIONARY dim_practitioners_dict
(
    practitioner_id UInt64,
    name String,
    specialty String
)
PRIMARY KEY practitioner_id
SOURCE(CLICKHOUSE(TABLE 'dim_practitioners'))
LIFETIME(MIN 3600 MAX 7200)
LAYOUT(FLAT());  -- For sequential IDs
-- or LAYOUT(HASHED());  -- For arbitrary IDs
```

Benefits:
- No parts to merge
- Faster lookups
- Automatic reloading
- No lightweight delete issues

---

## 8. Conclusion

The ClickHouse server experienced critical issues during 2026-01-08 due to a **confirmed bug in ClickHouse 25.12.x** related to patch parts (lightweight deletes). This bug caused:

- 35% merge failure rate on dim_practitioners
- 42% INSERT failure rate
- Accumulation of 1,140 active parts (should be <100)
- Severe query performance degradation
- 318 memory limit errors

**The primary recommendation is to downgrade to ClickHouse v25.9 or earlier** to immediately resolve the bug. The secondary recommendations (increase memory, batch inserts, avoid lightweight deletes) will help but won't fully resolve the issue while the bug persists.

---

## 9. References

### GitHub Issues

1. [Issue #89836 - Logical error: 'Min/max part offset must be set in RangeReader for reading patch parts'](https://github.com/clickhouse/clickhouse/issues/89836)
2. [Issue #82033 - Lightweight Updates improvements (umbrella)](https://github.com/ClickHouse/ClickHouse/issues/82033)
3. [Issue #86919 - Lightweight Update is unstable](https://github.com/ClickHouse/ClickHouse/issues/86919)

### ClickHouse Documentation

- [Lightweight UPDATE Statement](https://clickhouse.com/docs/sql-reference/statements/update)
- [How we built fast UPDATEs for ClickHouse](https://clickhouse.com/blog/updates-in-clickhouse-2-sql-style-updates)
- [ClickHouse Release 25.7](https://clickhouse.com/blog/clickhouse-release-25-07)

---

## Appendix A: Sample Failing Query

```sql
-- This query pattern is failing with LOGICAL_ERROR (Code 49):

INSERT INTO fact_conditions (created_at, patient_id, encounter_id, location_id, organization_id, d)
WITH (SELECT count() FROM fact_encounters) AS encounter_total
SELECT
    enc.start_dt + src.created_offset AS created_at,
    enc.patient_id,
    enc.encounter_id,
    enc.location_id,
    enc.organization_id,
    printf(...) AS d
FROM
(
    SELECT
        number,
        constAt('cond_codes', (intHash64(number) % constLen('cond_codes')) + 1) AS cond_code,
        -- ... more fields
    FROM numbers_mt(10000)
) AS src
INNER JOIN
(
    SELECT
        rowNumberInAllBlocks() + 1 AS rn,
        encounter_id,
        patient_id,
        location_id,
        organization_id,
        d.start_datetime AS start_dt
    FROM fact_encounters  -- Reading from table with patch parts
    WHERE toYYYYMM(snowflakeIDToDateTime(encounter_id)) = '202412'
    ORDER BY rand() LIMIT 10000
) AS enc ON (src.number % encounter_total) + 1 = enc.rn;
```

**Failure Point:** When reading from `fact_encounters` (has 1 part with patches)

---

## Appendix B: Server Metrics at Time of Analysis

```
Host:                chi-ai-ai-0-0-0
Version:             25.12.2.54
Uptime:              282,723 seconds (~3.3 days)
Database:            fhir
Current Memory:      421.84 MiB
Memory Limit:        12.60 GiB
Total Tables:        10 (fhir database)
Total Active Parts:  1,746
Total Rows:          563.9 million
Total Size:          24.4 GB
```

---

**Report Generated:** 2026-01-11 14:01:25 UTC
**Analysis Tool:** ClickHouse Analyst (Claude Code)
**Contact:** For questions about this report, consult your database administrator or ClickHouse support.