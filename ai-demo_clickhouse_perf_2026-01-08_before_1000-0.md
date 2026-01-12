# ClickHouse performance & incident analysis (ai-demo) — 2026-01-08 before 10:00 UTC

Generated: 2026-01-11  
Connector: `ai-demo`  
Node: `chi-ai-ai-0-0-0` (ClickHouse `25.12.2.54`, timezone `UTC`)

## Scope and data availability

### Time window analyzed
- Requested: **2026-01-08 before 10:00 UTC**
- Server start time (uptime-based): **2026-01-08 07:29:15 UTC**
- Effective window used: **2026-01-08 07:29:15 → 09:59:59 UTC**

### System log coverage in this window
- `system.text_log`: present for the full window (rows: **28,307**).
- `system.part_log`: present for the full window (used throughout).
- `system.query_log`: starts at **2026-01-08 08:24:41 UTC** (rows in-window: **11,104**). Anything before 08:24:41 relies on `text_log` / `part_log` / metric logs.

## Executive summary

1) **Severe server-wide memory pressure** during ~08:20–09:05 UTC led to repeated `(total) memory limit exceeded` errors (Code **241**) impacting both user queries and background/system-log work.

2) **Lightweight delete / patch-parts workload dominated part creation** (thousands of tiny `patch-*` parts), and hit **two distinct LOGICAL_ERROR signatures (Code 49)**:
   - `Min/max part offset must be set in RangeReader for reading patch parts` (observed during INSERT…SELECT reads).
   - `Block structure mismatch in patch parts stream` (observed as repeated merge failures for `fhir.dim_practitioners`).

3) **Merge/replication fallout persisted**: a `fhir.dim_practitioners` replication queue merge created at **2026-01-08 08:51:47 UTC** remains stuck (still failing with Code 49 as of 2026-01-11), and the table currently holds **999 active patch parts**.

4) **Query workload was extremely scan-heavy** (multi‑GiB reads per query, hundreds of GiB per 5‑minute bucket) with p95 latencies often **20–36s** and max ~**87s**, likely contributing to the memory pressure when run concurrently.

## Findings (with evidence)

### A) Errors and their impact

#### A1) Server memory limit exceeded (Code 241)
- `system.server_settings.max_server_memory_usage` = **13,529,146,982 bytes (~12.60 GiB)**.
- Query failures in `system.query_log` (08:24:41–10:00): **318** exceptions with Code **241**.
- `system.part_log` also shows merge failures with Code **241** (e.g., `fhir.dim_locations` merge errors; plus system log table merges failing during flush/merge).
- `system.asynchronous_metric_log` shows `MemoryResident` jumping from ~1–1.6 GiB to **~9.2 GiB at 08:20**, then **~12.9–13.3 GiB between 08:25 and 09:05**, and dropping back to ~2.3–2.6 GiB after **09:10**.

Interpretation:
- The memory failures were not isolated to a single query; this is consistent with **concurrent scan-heavy SELECTs** plus background activity hitting the **server-wide** memory cap (`max_server_memory_usage`), triggering the OvercommitTracker to kill/cancel work.

#### A2) Patch-parts related LOGICAL_ERROR (Code 49)
Observed in both query execution and background merges.

1) INSERT failures (query_log): **78** failures with Code **49** (INSERT…SELECT patterns reading from `fhir.fact_encounters` and other tables).
   - Example error: `Min/max part offset must be set in RangeReader for reading patch parts` (LOGICAL_ERROR).

2) Merge failures (part_log): `fhir.dim_practitioners`
   - `system.part_log` (07:29–10:00): **225** merge attempts, **79** merge errors (Code **49**).
   - Example error: `Block structure mismatch in patch parts stream: different names of columns … last_updated DateTime UInt32 …` (LOGICAL_ERROR).

Interpretation / likely root cause:
- The workload executed many `DELETE FROM ...` statements and produced a very large number of `patch-*` parts (see sections C and D).
- The error messages match known upstream issues in ClickHouse around patch parts / lightweight updates (see “GitHub triage”).

#### A3) Network-level errors (Code 210) and dropped connections (Code 236)
- `system.error_log` (07:29–10:00): **17,970** occurrences of Code **210** (`NETWORK_ERROR`) with messages like “Socket is not connected / connection reset by peer”.
- `system.query_log`: **40** exceptions Code **236** (`Client has dropped the connection, cancel the query`) around 09:06 UTC, during the peak instability window.

Interpretation:
- These are most consistent with **clients aborting connections** (timeouts / retries / load generators terminating requests) while the server was overloaded and/or terminating queries due to memory pressure.

### B) SELECT query performance (08:24:41–10:00)

#### B1) Time-bucket trend
From ~08:25 to ~09:05, SELECTs show:
- Typical p95 in **~21–36s** (5‑minute buckets)
- Max up to **~87s**
- Massive aggregate read volume per 5 minutes (often **~100–150 GiB**)

After ~09:10, the workload sharply drops to a small number of low-latency queries (p95 ~0.1s).

#### B2) Top offender query shapes
By total read volume, the dominant offender was a recurring “encounter_summary” query shape:
- Executions: **179**
- Failures: **74**
- Avg: **~68s**, p95: **~83s**, max: **~87s**
- Total reads: **~414 GiB**
- Typical single-execution reads: **~3.65–3.69 GiB**, rows read: **~216–224M**

This query family, combined with concurrency, is consistent with the observed server-wide memory saturation.

### C) INSERT performance and ingestion behavior (08:24:41–10:00)

#### C1) Insert successes
- Inserts completed (query_log): **108** (`QueryFinish` + `query_kind='Insert'`).
- Insert latency trend (5‑minute buckets): avg **~0.56–1.17s**, p95 **~1.9–5.0s**.
- Largest/slowest successful inserts observed were `INSERT INTO fhir.fact_encounters ...` writing **10,000 rows** and taking **~2–7s** each (with high per-query `memory_usage` reported).

#### C2) Insert failures
- Insert exceptions: **78** failures with Code **49** (patch-parts RangeReader error), and **1** failure with Code **241** (memory).
- The Code 49 failures line up with load scripts like `load_fact_conditions.sql`, `load_fact_procedures.sql`, `load_fact_observations.sql`, `load_fact_medications.sql`, and `load_fact_encounters.sql` variants that perform INSERT…SELECT over MergeTree tables.

Impact:
- These failures likely prevented a portion of the intended ingestion workload from completing successfully during the window.

### D) Parts, merges, and lightweight deletes (patch parts)

#### D1) Deletes were a first-class workload
In query_log (08:24:41–10:00), finished queries by kind:
- `Select`: **4,608**
- `Insert`: **108**
- `Delete`: **187**
- `Update`: **188**

The Delete statements include patterns like:
- `DELETE FROM dim_practitioners WHERE practitioner_id IN (SELECT ... ORDER BY rand() LIMIT 1000)`
- Similar for `dim_locations`, `dim_organizations`, `dim_patients`, and fact tables.

#### D2) Patch part creation dominated `part_log` NewPart events
From `system.part_log` (07:29–10:00), for several tables almost all “new parts” were `patch-*` parts:
- `fhir.dim_practitioners`: **1,078** new parts, **1,056** patch parts (p50 ~**2.48 KiB**)
- `fhir.dim_locations`: **1,078** new parts, **1,056** patch parts (p50 ~**2.21 KiB**)
- `fhir.dim_organizations`: **1,127** new parts, **1,104** patch parts (p50 ~**2.22 KiB**)
- `fhir.dim_patients`: **1,029** new parts, **1,008** patch parts (p50 ~**3.71 KiB**)

This indicates the delete workload touched many parts, generating a large number of tiny patch parts and forcing compaction/merge work.

#### D3) Current state (parts inventory)
`system.parts` (active parts now):
- `fhir.dim_practitioners`: **1,140** active parts, **999** active patch parts (patch bytes ~**12.24 MiB** of ~**29.43 MiB** total)
- `fhir.dim_locations`: **91** active parts, **67** patch parts
- `fhir.dim_organizations`: **93** active parts, **69** patch parts

#### D4) Merge and replication symptoms
- `fhir.dim_practitioners`:
  - Merge errors (07:29–10:00): **79** (Code 49, patch parts stream mismatch).
  - Replication queue currently shows a stuck merge task created **2026-01-08 08:51:47 UTC** (`MERGE_PARTS` → new part `202312_0_28_3_41`), still failing with the same error.
- `fhir.dim_locations`:
  - Merge errors: **4**, all Code 241 (memory pressure spillover).

## Root cause assessment

High confidence:
- The system was driven into a failure mode by a combination of:
  1) **Highly concurrent, scan-heavy analytics queries** (multi‑GiB reads, tens of seconds each) causing resident memory to surge into the **~13 GiB** range, exceeding `max_server_memory_usage` (~12.6 GiB).
  2) **Aggressive row-level deletes** that created **thousands of tiny `patch-*` parts**, increasing merge pressure and exercising patch-parts codepaths.
  3) **Patch-parts related bugs/edge cases** (Code 49) that caused both query failures (RangeReader) and repeated merge failures (`dim_practitioners`), leaving the replica unable to complete at least one merge for days.

Moderate confidence:
- The flood of `NETWORK_ERROR` entries is downstream fallout (clients resetting connections due to timeouts/retries while the server was overloaded and/or cancelling work).

## Recommendations

### Immediate mitigations (today/this week)
1) **Reduce concurrent heavy SELECT load** to stay under the server-wide memory cap:
   - Consider setting server limits like `max_concurrent_select_queries` and/or `max_memory_usage_for_all_queries` to keep aggregate memory below `max_server_memory_usage`.
   - Alternatively, raise `max_server_memory_usage` only if the host has sufficient RAM and you accept higher resident usage (validate with `asynchronous_metric_log`).

2) **Pause or redesign the row-level DELETE workload** driving patch part churn:
   - Prefer **partition-level operations** (e.g., drop partition) or **TTL-based cleanup** where possible.
   - If you must delete/update rows, reduce frequency and batch size; avoid “ORDER BY rand() LIMIT …” patterns that touch many parts.

3) **Address the patch-parts LOGICAL_ERROR**:
   - Upgrade ClickHouse to a build that includes fixes for the patch-parts issues below (or backport if you maintain your own builds).
   - Expect that the stuck `fhir.dim_practitioners` queue entry will not clear until the underlying bug/condition is resolved.

### Follow-ups (next iteration)
1) **Validate table design against query patterns** (especially the `encounter_summary` family):
   - Most pain comes from large scans; align `ORDER BY`/primary key and add selective skipping indexes where appropriate.

2) **Re-test ingestion without patch-part churn**:
   - Run the same load with deletes disabled; compare p95 and error rates to isolate whether patch parts are the primary instability trigger.

## GitHub triage (upstream)

These upstream issues match the observed error strings and are likely relevant for ClickHouse `25.12.2.54`:
- ClickHouse/ClickHouse#89836 — “Logical error: 'Min/max part offset must be set in RangeReader for reading patch parts' with LWU and index” (OPEN)  
  https://github.com/ClickHouse/ClickHouse/issues/89836
- ClickHouse/ClickHouse#89472 — “Block structure mismatch in patch parts stream” (OPEN)  
  https://github.com/ClickHouse/ClickHouse/issues/89472

## Appendix: key queries used (examples)

All queries were time-bounded and avoided `SELECT *`.

```sql
-- server identity + start time
select hostName(), version(), timezone(), now(), uptime(), now() - toIntervalSecond(uptime()) as server_start_time_utc;

-- exceptions by code (window)
select exception_code, count()
from system.query_log
where event_time >= toDateTime('2026-01-08 07:29:15')
  and event_time <  toDateTime('2026-01-08 10:00:00')
  and type like 'Exception%'
group by exception_code;

-- patch part creation volume
select database, table, count()
from system.part_log
where event_time >= toDateTime('2026-01-08 07:29:15')
  and event_time <  toDateTime('2026-01-08 10:00:00')
  and event_type = 'NewPart'
  and part_name like 'patch-%'
group by database, table;
```

