# ClickHouse Performance Analysis Report (2026-01-08 before 10:00)

## 1. Summary

This report summarizes the performance analysis of the ClickHouse server `chi-ai-ai-0-0-0` (version 25.12.2.54) for the period of 2026-01-08 from 00:00:00 to 10:00:00.

**Key Findings:**

*   **Memory Pressure:** The server is experiencing significant memory pressure, leading to failed merges and killed queries. This is the most critical issue to address.
*   **Merge Errors:** Merges are failing due to "memory limit exceeded" errors.
*   **Logical Errors:** There are "Block structure mismatch" errors during merges, which could indicate data corruption or a software bug.
*   **Slow Queries:** A specific complex query is consistently appearing as the longest-running query.

## 2. Detailed Findings

### 2.1. Errors

No errors were found in the `system.query_log`. However, the `system.part_log` and `system.text_log` revealed two major issues:

*   **Memory Limit Exceeded (Code: 241):** This error appears frequently in the logs, indicating that merge operations are consuming more memory than the configured limit (12.60 GiB). This is a major cause of performance degradation as it prevents data from being efficiently merged.

*   **Block Structure Mismatch (Code: 49):** This logical error occurs during merges and points to an inconsistency in the data structure of the parts being merged. The error message "Block structure mismatch in patch parts stream: different names of columns: specialty LowCardinality(String) LowCardinality(size = 0, UInt8(size = 0), Unique(size = 1, String(size = 1))) last_updated DateTime UInt32(size = 0)" suggests a problem with the `specialty` and `last_updated` columns.

### 2.2. Query Performance

*   **SELECT Queries:** The top 5 longest-running queries are all instances of the same query with a `normalized_query_hash` of `50895899387004963`. The query is a complex one with multiple CTEs and joins, and it takes around 86-87 seconds to complete. While this may not be a problem in itself, it's worth investigating for potential optimizations.

*   **INSERT Queries:** The longest-running INSERT statements are for the `fact_encounters` table. These queries are part of a data loading process and take 5-7.5 seconds to complete. This is not necessarily a problem, but it's good to be aware of the ingestion performance.

### 2.3. Merges and Mutations

*   **Merges:** There are no currently running merges, but the logs show that many merges have failed due to memory limit exceeded errors.
*   **Mutations:** There are no running or stuck mutations.

## 3. Recommendations

### 3.1. High Priority

*   **Increase Memory:** The most critical issue is the lack of memory. The server's memory limit should be increased to handle the merge load. A good starting point would be to double the `max_server_memory_usage` to 24GiB. This can be set in the `users.xml` file.
*   **Investigate Block Structure Mismatch:** The "Block structure mismatch" error needs immediate attention as it can lead to data corruption.
    *   **Identify the problematic table:** The error message doesn't explicitly name the table. It is necessary to investigate which table is causing this error. It is likely one of the tables involved in the slow query.
    *   **Check for schema changes:** Review recent schema changes for the affected table.
    *   **Contact support:** If the issue persists, contact ClickHouse support and provide the error logs.

### 3.2. Medium Priority

*   **Optimize Slow Query:** The long-running query should be analyzed for potential optimizations. This could involve:
    *   Adding or modifying indexes.
    *   Rewriting the query to be more efficient.
    *   Denormalizing data to avoid joins.
*   **Review Merge Tree Settings:** Review the `merge_tree` settings for the tables with high merge activity. It might be possible to tune the merge strategy to reduce memory consumption.

### 3.3. Low Priority

*   **Monitor Ingestion Performance:** Keep an eye on the `INSERT` performance to ensure it doesn't degrade over time.

## 4. Next Steps

1.  Increase the `max_server_memory_usage` to at least 24GiB.
2.  Investigate the "Block structure mismatch" error to identify the root cause.
3.  Analyze and optimize the slow-running query.
## 5. Related GitHub Issue

A relevant GitHub issue has been found that describes a similar "Block structure mismatch" error:

*   **Issue:** [#89472 Block structure mismatch in patch parts stream](https://github.com/ClickHouse/ClickHouse/issues/89472)
*   **Status:** Open
*   **Description:** The issue describes the same error message and is related to lightweight updates. The ClickHouse team is still investigating the issue and has requested more information for reproduction. It is recommended to monitor this issue for any updates or workarounds.