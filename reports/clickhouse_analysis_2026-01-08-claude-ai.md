# ClickHouse Server Performance Analysis Report

**Server:** chi-ai-ai-0-0-0  
**Version:** 25.12.2.54  
**Analysis Period:** 2026-01-08 00:00 - 10:00 UTC  
**Report Generated:** 2026-01-11  

---

## Executive Summary

The analysis of the ClickHouse server on January 8, 2026 revealed **three critical issues**:

1. **CRITICAL BUG: Patch Parts / Lightweight Updates Bug** - A severe bug in version 25.12.2.54 is causing merge failures and query errors related to patch parts
2. **Memory Pressure** - Heavy benchmark workload caused memory limit exceeded errors (318 OOM failures)
3. **Network Errors** - High volume of connection reset/socket disconnection errors (25,545 errors)

---

## 1. Critical Bug: Patch Parts LOGICAL_ERROR (Code 49)

### Finding

The server is experiencing a critical bug related to **patch parts** (lightweight updates feature) causing two types of failures:

- **Query failures (78 instances):** "Min/max part offset must be set in RangeReader for reading patch parts"
- **Merge failures (79+ instances):** "Block structure mismatch in patch parts stream: different names of columns"

### Evidence

**Error Distribution:**
| Error Code | Count | Description |
|------------|-------|-------------|
| 241 | 318 | MEMORY_LIMIT_EXCEEDED |
| 49 | 78 | LOGICAL_ERROR (patch parts bug) |
| 236 | 40 | ABORTED (client dropped connection) |

**Affected Tables with Patch Parts:**
| Database | Table | Patch Parts | Regular Parts | Total |
|----------|-------|-------------|---------------|-------|
| fhir | dim_practitioners | 999 | 141 | 1140 |
| fhir | dim_organizations | 69 | 24 | 93 |
| fhir | dim_locations | 67 | 24 | 91 |
| fhir | fact_observations | 4 | 70 | 74 |
| fhir | fact_procedures | 4 | 74 | 78 |
| fhir | fact_encounters | 3 | 72 | 75 |

**Stuck Merge in Replication Queue:**
```
Table: fhir.dim_practitioners
Part: 202312_0_28_3_41
Retry Count: 4,408 attempts
Error: Block structure mismatch in patch parts stream: different names of columns:
       specialty LowCardinality(String) vs last_updated DateTime
```

### Root Cause Analysis

The issue stems from tables configured with lightweight updates enabled:
```sql
SETTINGS enable_block_number_column = 1, enable_block_offset_column = 1
```

The `dim_practitioners` table has these settings which create **patch parts** when UPDATEs are performed. A bug in version 25.12.2.54 causes:

1. **During queries:** The RangeReader fails to properly handle min/max part offsets when reading patch parts
2. **During merges:** Block structure validation fails due to column order mismatch between the base part and patch parts

### GitHub Issue Reference

This appears to be related to the following issues in ClickHouse:
- Lightweight updates / patch parts feature introduced in v24.8+
- Known instabilities in early versions of this feature
- Similar issues reported for "Block structure mismatch" during merges with patch parts

**Note:** GitHub API was unreachable during analysis; manual search recommended for:
- Issues containing "patch parts LOGICAL_ERROR"
- Issues containing "Block structure mismatch patch"
- Issues containing "RangeReader patch parts"

### Recommendations

**Immediate Actions:**
1. **Downgrade or upgrade ClickHouse** - Move away from 25.12.2.54 to a patched version
2. **Disable lightweight updates** on affected tables if possible:
   ```sql
   ALTER TABLE fhir.dim_practitioners MODIFY SETTING 
     enable_block_number_column = 0, 
     enable_block_offset_column = 0;
   ```
3. **Force merge patch parts** to eliminate them (may require restart if merges are stuck)

**For dim_practitioners specifically:**
1. Consider rebuilding the table without patch parts
2. Clear the replication queue entry manually if possible
3. Monitor merge queue for similar failures

---

## 2. Memory Pressure and OOM Errors

### Finding

During the benchmark period (08:20-09:10), the server experienced severe memory pressure with 318 memory limit exceeded errors.

### Evidence

**Memory Usage Timeline:**
| Time (UTC) | Memory Resident | OS Available | Status |
|------------|-----------------|--------------|--------|
| 08:15 | 1.5 GiB | 13.4 GiB | Normal |
| 08:25 | 8.7 GiB | 7.0 GiB | High load |
| 08:30 | 9.1 GiB | 6.5 GiB | Critical |
| 08:45 | 9.3 GiB | 6.3 GiB | Peak pressure |
| 09:00 | 8.9 GiB | 6.8 GiB | High |
| 09:10 | 2.6 GiB | 13.1 GiB | Recovered |

**Server Memory Configuration:**
- Total RAM: 16.4 GiB
- Memory Limit: 12.6 GiB
- Peak Usage: ~9.3 GiB resident (but virtual memory much higher)

**Top Memory-Consuming Queries:**
1. Patient summary queries consuming ~2.2 GiB each
2. Dictionary lookups (`patients_dict_source`) using 2+ GiB per query
3. Concurrent execution of multiple heavy queries

### Root Cause

The benchmark workload ran highly concurrent complex queries that exceeded memory limits:
- 400-600 SELECT queries per 5-minute interval during peak
- Multiple concurrent queries each using 2+ GiB
- Total memory demand exceeded 12.6 GiB limit

### Recommendations

1. **Increase memory limit** if hardware allows:
   ```sql
   SET max_memory_usage = 10000000000; -- per query
   SET max_memory_usage_for_user = 12000000000;
   ```

2. **Add query concurrency limits:**
   ```sql
   SET max_concurrent_queries_for_user = 5;
   ```

3. **Optimize heavy queries:**
   - The `patients_dict_source` lookups are consuming excessive memory
   - Consider pre-aggregating or using JOIN hints

4. **Schedule benchmarks during off-peak hours**

---

## 3. SELECT Query Performance

### Finding

Query performance was severely degraded during the benchmark period with p95 latency reaching 30+ seconds.

### Evidence

**Query Latency by Period:**
| Time (UTC) | Queries | Failed | Avg (ms) | P95 (ms) | Max (ms) |
|------------|---------|--------|----------|----------|----------|
| 08:25 | 457 | 18 | 11,233 | 34,966 | 76,564 |
| 08:30 | 403 | 12 | 11,953 | 36,508 | 79,025 |
| 08:35 | 613 | 27 | 8,824 | 29,894 | 87,078 |
| 08:45 | 570 | 16 | 7,515 | 30,380 | 83,481 |
| 09:00 | 489 | 40 | 8,971 | 27,366 | 86,577 |
| 09:10 | 8 | 0 | 88 | 128 | 129 |

**Heaviest Query Patterns by Data Read:**

| Pattern | Executions | Total Read | Avg Duration |
|---------|------------|------------|--------------|
| Patient summary (6-month) | 105 | 385.83 GiB | 68 sec |
| Patient summary (2-week) | 135 | 89.04 GiB | 27 sec |
| Procedure analysis | 175 | 57.73 GiB | 11 sec |
| Emergency visit analysis | 316 | 34.00 GiB | 9 sec |

### Recommendations

1. **Optimize the patient summary queries** - these read 3.6+ GiB per execution
2. **Consider materialized views** for frequently run analytical queries
3. **Add appropriate indices** or projections for common filter patterns
4. **Implement query result caching** for repeated queries

---

## 4. INSERT Performance

### Finding

INSERT operations experienced failures due to the patch parts bug.

### Evidence

**Insert Activity (during benchmark):**
| Time (UTC) | Inserts | Failed | Avg (ms) | Rows Written |
|------------|---------|--------|----------|--------------|
| 08:35 | 26 | 12 | 834 | 30,578 |
| 08:40 | 40 | 17 | 1,202 | 41,360 |
| 08:45 | 42 | 18 | 875 | 41,380 |
| 08:50 | 45 | 22 | 956 | 51,125 |
| 08:55 | 28 | 9 | 774 | 31,104 |

**High failure rate: 40-50% of inserts failing** due to patch parts bug when reading from source tables.

### Root Cause

INSERT ... SELECT queries reading from tables with patch parts fail with LOGICAL_ERROR when the RangeReader encounters patch parts without proper offset handling.

### Recommendations

1. Fix the underlying patch parts issue (see Section 1)
2. Use OPTIMIZE FINAL on source tables before bulk inserts
3. Consider disabling lightweight updates on source tables used in INSERT ... SELECT

---

## 5. Merge Activity

### Finding

Merge operations are partially blocked due to the patch parts bug.

### Evidence

**Part Log Activity (00:00-10:00):**
| Event Type | Count |
|------------|-------|
| RemovePart | 24,909 |
| NewPart | 18,784 |
| MergeParts | 6,882 |
| MergePartsStart | 6,875 |

**Tables with Merge Failures:**
| Time | Database | Table | Merges | Failures |
|------|----------|-------|--------|----------|
| 09:00 | fhir | dim_practitioners | 57 | 57 |
| 08:00 | fhir | dim_practitioners | 168 | 22 |
| 08:00 | fhir | dim_locations | 219 | 4 |

**100% merge failure rate on dim_practitioners** during the 09:00 hour.

### Recommendations

1. Priority: Resolve the patch parts bug
2. Monitor replication_queue for stuck entries
3. Consider SYSTEM RESTART REPLICA after fixing the underlying issue

---

## 6. Mutation Status

### Finding

No stuck mutations detected - all mutations are complete.

### Evidence

```sql
SELECT * FROM system.mutations WHERE NOT is_done;
-- Returns: 0 rows
```

---

## 7. Network Errors

### Finding

High volume of network errors (25,545 total) indicating client connection issues.

### Evidence

| Error Type | Count |
|------------|-------|
| Socket not connected | 21,706 |
| Connection reset by peer | 3,839 |

**Source IPs:** 10.129.57.5 and 10.129.57.15 (likely benchmark clients)

### Root Cause

These errors correlate with:
1. Memory limit exceeded causing query termination
2. High server load causing timeouts
3. Benchmark clients disconnecting abruptly

### Recommendations

1. Investigate benchmark client configuration
2. Increase client timeout settings
3. Implement proper connection retry logic in clients

---

## 8. Storage Status

### Finding

Storage is healthy with adequate free space.

### Evidence

| Disk | Used | Free | Total |
|------|------|------|-------|
| default | 28.11 GiB | 168.57 GiB | 196.68 GiB |

**Top Tables by Size:**
| Table | Parts | Compressed Size |
|-------|-------|-----------------|
| fhir.fact_observations | 74 | 10.75 GiB |
| fhir.fact_procedures | 78 | 9.22 GiB |
| system.trace_log | 45 | 2.74 GiB |
| fhir.fact_encounters | 75 | 1.96 GiB |
| fhir.fact_medications | 55 | 1.75 GiB |

### Note

`dim_practitioners` has 1,140 parts (999 patch parts) which is abnormally high and contributes to the merge issues.

---

## Summary of Recommendations

### Critical (Immediate Action Required)

1. **Upgrade/downgrade ClickHouse** from version 25.12.2.54 to address the patch parts bug
2. **Disable lightweight updates** on affected tables or rebuild them without patch parts
3. **Clear stuck merge** from replication queue on dim_practitioners

### High Priority

4. **Increase memory limits** or reduce query concurrency during benchmarks
5. **Optimize heavy queries** especially patient summary reports
6. **Monitor replication queue** for new failures

### Medium Priority

7. **Implement query caching** for repeated analytical queries
8. **Review benchmark client configuration** to prevent connection drops
9. **Consider materialized views** for frequently executed reports

---

## Appendix: Key Queries for Ongoing Monitoring

```sql
-- Check for patch parts
SELECT database, table, countIf(name LIKE 'patch-%') as patch_parts
FROM system.parts WHERE active = 1
GROUP BY database, table HAVING patch_parts > 0;

-- Check replication queue for errors
SELECT database, table, type, num_tries, substring(last_exception, 1, 100)
FROM system.replication_queue WHERE last_exception != '';

-- Monitor memory pressure
SELECT metric, value FROM system.asynchronous_metrics
WHERE metric IN ('MemoryResident', 'OSMemoryAvailable');

-- Check for failing queries
SELECT exception_code, count() as failures
FROM system.query_log
WHERE event_time > now() - INTERVAL 1 HOUR AND type LIKE 'Exception%'
GROUP BY exception_code ORDER BY failures DESC;
```
