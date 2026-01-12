
# ClickHouse Performance Analysis Report (2026-01-08 before 10:00) - Final

## 1. Executive Summary

This report summarizes the performance analysis of the ClickHouse server `chi-ai-ai-0-0-0` (version 25.12.2.54) for the period of 2026-01-08 from 00:00:00 to 10:00:00.

**Key Findings:**

*   **Severe Memory Pressure:** The server experienced significant memory pressure, leading to failed merges and killed queries. This is the most critical issue to address.
*   **Merge and Logical Errors:** Merges are failing due to "memory limit exceeded" errors (Code 241) and "Block structure mismatch" logical errors (Code 49).
*   **Lightweight Deletes:** The workload includes a high number of lightweight deletes, which are creating a large number of small "patch" parts, increasing merge pressure and triggering bugs.
*   **Stuck Merges:** A merge for the `fhir.dim_practitioners` table has been stuck for days, failing with a "Block structure mismatch" error.
*   **Slow Queries:** The workload is dominated by scan-heavy `SELECT` queries with high latency, contributing to the memory pressure.

## 2. Detailed Findings

### 2.1. Memory Pressure and Memory Limit Exceeded Errors

The server's memory usage peaked at ~13.3 GiB, exceeding the configured limit of 12.60 GiB. This caused **318 query failures** with "memory limit exceeded" errors (Code 241) and also caused merges to fail. The memory pressure was likely caused by a combination of scan-heavy `SELECT` queries running concurrently with background merge and delete operations.

### 2.2. Lightweight Deletes and Patch-Parts Related Logical Errors

The workload includes a significant number of `DELETE` and `UPDATE` statements, which are creating a large number of small "patch" parts. For example, the `fhir.dim_practitioners` table has **999 active patch parts**. This high number of small parts increases the merge pressure on the server.

Furthermore, these patch parts are triggering two distinct logical errors (Code 49):
*   `Min/max part offset must be set in RangeReader for reading patch parts`
*   `Block structure mismatch in patch parts stream`

These errors are causing both `INSERT` and `MERGE` operations to fail.

### 2.3. Stuck Merges and Replication Issues

A merge for the `fhir.dim_practitioners` table, created on 2026-01-08 08:51:47 UTC, is still failing with a "Block structure mismatch" error. This is preventing the table from being properly merged and is a serious issue that needs to be addressed.

### 2.4. Slow Query Performance

The query workload is dominated by scan-heavy `SELECT` queries with p95 latencies between 20 and 36 seconds, and a max latency of around 87 seconds. These queries read a large amount of data (multi-GiB reads per query) and contribute significantly to the server's memory pressure.

## 3. Root Cause Analysis

The root cause of the performance issues is a combination of factors:

1.  **High-concurrency of scan-heavy analytics queries** that consume a large amount of memory.
2.  **Aggressive row-level deletes** that create a large number of small "patch" parts, increasing merge pressure and triggering bugs in ClickHouse.
3.  **Bugs in ClickHouse's lightweight updates/deletes implementation** that cause logical errors and stuck merges.

## 4. Recommendations

### 4.1. Immediate Mitigations

1.  **Reduce Concurrent Heavy SELECT Load:**
    *   Consider setting server limits like `max_concurrent_select_queries` and/or `max_memory_usage_for_all_queries` to keep aggregate memory below `max_server_memory_usage`.
    *   Alternatively, increase `max_server_memory_usage` if the host has sufficient RAM.

2.  **Pause or Redesign the Row-Level DELETE Workload:**
    *   Prefer partition-level operations (e.g., `ALTER TABLE ... DROP PARTITION`) or TTL-based cleanup where possible.
    *   If you must delete/update rows, reduce the frequency and batch size.

3.  **Address the Patch-Parts LOGICAL_ERROR:**
    *   Upgrade ClickHouse to a version that includes fixes for the patch-parts issues.
    *   The stuck merge for `fhir.dim_practitioners` will likely not clear until the underlying bug is resolved.

### 4.2. Follow-ups

1.  **Validate Table Design Against Query Patterns:**
    *   Align `ORDER BY`/primary key and add selective skipping indexes to reduce the amount of data scanned by the slow queries.

2.  **Re-test Ingestion Without Patch-Part Churn:**
    *   Run the same load with deletes disabled to isolate whether patch parts are the primary instability trigger.

## 5. Related GitHub Issues

The following upstream GitHub issues are likely related to the observed errors:

*   [#89836 - “Logical error: 'Min/max part offset must be set in RangeReader for reading patch parts' with LWU and index”](https://github.com/ClickHouse/ClickHouse/issues/89836) (OPEN)
*   [#89472 - “Block structure mismatch in patch parts stream”](https://github.com/ClickHouse/ClickHouse/issues/89472) (OPEN)

It is recommended to monitor these issues for any updates or workarounds.
