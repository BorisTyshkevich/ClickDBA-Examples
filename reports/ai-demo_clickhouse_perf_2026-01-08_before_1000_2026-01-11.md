# ClickHouse performance analysis (ai-demo) — 2026-01-08 before 10:00 UTC

Scope: ClickHouse server `chi-ai-ai-0-0-0` (tz `UTC`), version `25.12.2.54`, time window **2026-01-08 00:00:00–09:59:59 UTC** (i.e., “before 10:00”). Data sources: `system.query_log`, `system.part_log`, `system.text_log`, `system.error_log`, `system.asynchronous_metric_log`, `system.replicas`, `system.replication_queue`, `system.parts`, `system.parts_columns`, `system.mutations`.

## Executive summary

Primary issues in this window:

1) **Lightweight update/delete (“patch parts”) appears broken in 25.12.2.54 under this workload**, causing:
   - Query failures during reads: **78** initial queries failed with `Code: 49` (`Min/max part offset must be set in RangeReader for reading patch parts`) during **08:35–09:05**.
   - Merge failures: **79** merge failures for `fhir.dim_practitioners` with `Code: 49` (`Block structure mismatch in patch parts stream`) repeatedly during **08:50–09:59**.
   - Replication/merge backlog symptom (current state): `fhir.dim_practitioners` has a stuck `MERGE_PARTS` queue entry with **>4k retries** (see “Current impact”).

2) **Severe memory pressure from analytical SELECTs**:
   - **318** initial queries failed with `Code: 241 (MEMORY_LIMIT_EXCEEDED)` before 10:00.
   - `QueriesMemoryUsage` peaked at **~12.6 GiB** (near server cap `12.60 GiB`) and `LoadAverage1` peaked at **~32** on a 4-core node.
   - Many clients canceled/aborted: `Code: 236 (ABORTED)` **40** times; plus pervasive socket disconnect errors in `text_log`.

3) **Operational hygiene / restarts**:
   - The server received a termination signal at **2026-01-08 01:03:07Z**.
   - A new server start is logged at **2026-01-08 07:29:18Z** with “unclean restart” and ownership mismatch warnings.

Disk capacity looked healthy during the window (default disk free ~181–184 GB, used ~27–30 GB).

## Server + table context

- Host/version: `chi-ai-ai-0-0-0` / `25.12.2.54`, tz `UTC`.
- Node sizing from server log at `2026-01-08 07:29:18Z`: **14.00 GiB RAM**, **4 logical cores**, `max_server_memory_usage` set to **12.60 GiB**; merges/mutations soft limit set to **7.00 GiB**.
- Key tables are `ReplicatedMergeTree` with `enable_block_number_column=1` and `enable_block_offset_column=1` (example: `fhir.dim_practitioners`).

## Errors (what happened, evidence, likely causes)

### A) Patch parts / lightweight update-delete failures (root cause)

**What we saw (windowed counts):**

- `system.query_log` initial-query exceptions (00:00–10:00):
  - `Code 241 (MEMORY_LIMIT_EXCEEDED)`: **318**
  - `Code 49 (LOGICAL_ERROR)`: **78** — `Min/max part offset must be set in RangeReader for reading patch parts`
  - `Code 236 (ABORTED)`: **40**
- `system.part_log` merge errors (00:00–10:00):
  - `fhir.dim_practitioners`: **79** merge failures with `Code 49` — `Block structure mismatch in patch parts stream: different names of columns ...`

**Timeline correlation (selected 5-min buckets):**

- Patch-read logical errors (query `Code 49`) appear mainly in **08:35–08:55** (and a single event at 09:05).
- Merge failures for `fhir.dim_practitioners` repeat throughout **08:50–09:59** (4–5 failures every 5 minutes).

**Patch parts created in the window (current active parts whose `modification_time` is within the window):**

- Two patch IDs were created for `fhir.dim_practitioners` between ~08:48 and ~09:06:
  - `patch_id=116ec7151bbfe7ea649b23811e916438`: **5 parts** (08:48:44–08:51:46)
  - `patch_id=f18f7271629a324b0d26b6ad0b83a6c2`: **10 parts** (08:51:47–09:06:09)

**Critically, those two patch IDs have *different patch-part schemas* (observed via `system.parts_columns`):**

- Example `patch-116ec715...`: contains user columns (`last_updated`, `specialty`) + internal patch columns (`_part`, `_part_offset`, `_part_data_version`, `_block_number`, `_block_offset`).
- Example `patch-f18f727...`: contains only internal patch columns and `_row_exists` (no `last_updated`/`specialty`).

This strongly suggests **mixing lightweight UPDATE and lightweight DELETE patch streams** in the same partition, then:

- Reads fail with the RangeReader “min/max part offset” logical error.
- Background merges fail when applying patch parts during merge (`apply_patches_on_merge=1` by default), due to inconsistent patch-part block structure.

**Likely cause (high confidence):** a ClickHouse bug in the lightweight updates/deletes (“patch parts”) implementation for 25.10+ (and present in 25.12.2.54), especially when patch parts exist in combination and/or with in-order reading paths.

**Upstream references (GitHub):**

- `ClickHouse/ClickHouse#89836` — *Logical error: 'Min/max part offset must be set in RangeReader for reading patch parts' with LWU and index* (open). https://github.com/ClickHouse/ClickHouse/issues/89836
- `ClickHouse/ClickHouse#89472` — *Block structure mismatch in patch parts stream* (open). https://github.com/ClickHouse/ClickHouse/issues/89472
- `ClickHouse/ClickHouse#86169` — *Apply lightweight delete only on horizontal stage of merge* (merged PR, 2025-09-11). https://github.com/ClickHouse/ClickHouse/pull/86169

### B) Memory pressure / OOMs (query-level)

**What we saw:**

- `Code 241 (MEMORY_LIMIT_EXCEEDED)` caused **318** initial-query failures before 10:00.
- Many exceptions mention `OvercommitTracker decision: ... While executing AggregatingTransform`.

**Resource telemetry (00:00–10:00):**

- `CGroupMemoryTotal`: **15,032,385,536 bytes (~14.0 GiB)**
- `QueriesMemoryUsage`:
  - min **0**
  - max **12,576,562,704 bytes (~11.7 GiB)**
  - peaked around **08:20–09:05**
- `MemoryResident` max **13,298,315,264 bytes (~12.4 GiB)**.
- Load averages:
  - `LoadAverage1` peaked at **32.32** (08:40 bucket) on 4 cores.

**Top OOM offenders by `log_comment` (initial query exceptions, `exception_code=241`):**

- `Q7_months. Wait Time by Specialty (months)`: **75** OOMs
- `Q9_months. Patient Journey (months)`: **64** OOMs
- `Q8_months. Antibiotic % by Facility/Specialty (months)`: **50** OOMs
- `Q1_months. Procedure Volume by City and Month (6 months)`: **49** OOMs

These are long-running scans (several tens of seconds) and in at least one case average reads in the **TB** range per query (example: `Q9_months` averaged ~3.7 TB read).

**Likely cause (medium-high confidence):** unbounded analytic queries (large time ranges, big group-bys) running on a small node (4 cores / 14 GiB) with insufficient pre-aggregation and/or insufficient filtering to leverage ordering/partitioning; memory spills are not preventing peak memory use; overcommit kills queries.

### C) Network/socket error spam in server logs

`system.text_log` (00:00–10:00) contains **27,009** `Error` rows; the dominant message is `Code: 210 (NETWORK_ERROR) ... Socket is not connected, while reading from socket` and “Connection reset by peer”.

Peer distribution (for the dominant pattern) is heavily concentrated in:

- `10.129.57.15` (~12,773 rows)
- `10.129.57.5` (~12,772 rows)

**Likely cause (medium confidence):** clients / upstream proxy repeatedly opening HTTP connections and disconnecting during request read (timeouts, health checks, or client cancellations). This is *amplified* during the heavy-load period and after restarts, but the raw count suggests a configuration/traffic pattern that’s inherently noisy.

### D) Restarts / config warnings (operational)

From `system.text_log`:

- `2026-01-08 01:03:07Z`: `Received termination signal (Terminated)` and “Closed all listening sockets...”.
- `2026-01-08 07:29:18Z`: server start; “unclean restart” status file existed; warning: “Effective user of the process (clickhouse) does not match the owner of the data (root).”
- Repeated warning: `Listen [0.0.0.0]:9005 failed ... Address already in use` (7 occurrences).

These don’t explain the patch-part bugs, but they do add instability and complicate post-incident recovery (ownership mismatch can break expected behavior across restarts).

## Query performance (SELECT / INSERT)

All stats below are for initial queries in the window (00:00–10:00).

### SELECT

- Finished SELECTs: **4,545**
- Exceptions: **360**
- Duration ms (p50/p90/p99/max): **4,744 / 23,398 / 69,644 / 87,078**

Worst offenders by max runtime (08:24:41–10:00):

- `Q9_months. Patient Journey (months)`: p50 ~68.8s, max ~87.1s, avg read ~3.7 TB
- `Q1_months. Procedure Volume...`: p50 ~34.8s, avg read ~1.6 TB
- `Q7_months. Wait Time...`: p50 ~22.7s
- Several other “days/months” queries range ~10–30s.

### INSERT

- Finished INSERTs: **108**
- Exceptions: **79**
- Duration ms (p50/p90/p99/max): **146 / 1,956 / 6,086 / 7,401**
- Written rows per insert (p50/p90/p99): **20 / 10,000 / 10,000** (high variance; many very small inserts)

Notably, most INSERT exceptions are `Code 49` patch-part read logical errors, suggesting ingestion is failing when selecting from tables that have patch parts.

## Ingestion + parts/merges behavior

### New parts (00:00–10:00) — evidence of small-batch writes

From `system.part_log` (`event_type='NewPart'`):

- Overall `NewPart`: **18,784** events in the window
  - p50 rows/part: **30**
  - avg rows/part: **447**
  - avg size/part: **~0.035 MiB**

For `fhir` tables (top by new parts):

- `fhir.dim_practitioners`: **1,078** new parts, avg **~42 rows**, avg **~2.49 KiB**
- `fhir.dim_locations`: **1,078** new parts, avg **~41 rows**, avg **~2.24 KiB**
- `fhir.dim_organizations`: **1,127** new parts, avg **~41 rows**, avg **~2.21 KiB**
- `fhir.dim_patients`: **1,029** new parts, avg **~46 rows**, avg **~3.92 KiB**

These are extremely tiny parts and will inevitably drive high merge churn.

### Merges (00:00–10:00)

From `system.part_log` (`event_type='MergeParts'`, successful merges `error=0`):

- Merge duration ms (p50/p90/p99): **18 / 154 / 1,351**
- Average merge read size: **~9.0 MiB**

Errors:

- `fhir.dim_practitioners`: **79** merge failures (`Code 49`, patch-part block mismatch)
- `fhir.dim_locations`: **4** merge failures (`Code 241`, OOM during merges)
- Some `system.*` log tables also hit OOM during merges while the node was under heavy query memory pressure.

### Mutations

`system.mutations` shows **no mutation rows created** in the window for inspected tables; this is consistent with using lightweight update/delete (patch parts) rather than heavyweight `ALTER UPDATE/DELETE` mutations.

## Current impact (outside the window, but directly attributable)

As of **2026-01-11** (now), `system.replication_queue` for `fhir.dim_practitioners` shows a `MERGE_PARTS` entry stuck with:

- `new_part_name='202312_0_28_3_41'`
- `parts_to_merge=['202312_0_13_2_24','202312_16_28_1_29']`
- `num_tries` **~4432**
- `last_exception` starts with the same `Code: 49 ... Block structure mismatch in patch parts stream ...`

This indicates the 2026-01-08 patch-part/merge issue can become a persistent background failure (retry storm) until mitigated.

## Recommendations

### 1) Stop the bleeding: mitigate lightweight update/delete patch-part bugs

Pick one (ordered by typical impact/urgency):

1. **Upgrade ClickHouse** to a build that includes fixes for the patch-part issues you’re hitting; track and test against:
   - https://github.com/ClickHouse/ClickHouse/issues/89836
   - https://github.com/ClickHouse/ClickHouse/issues/89472
2. **Temporarily disable lightweight update/delete** (avoid patch parts) on affected workloads/tables:
   - Prefer classic mutations (`ALTER TABLE ... UPDATE/DELETE`) if correctness is required.
   - If you must keep lightweight deletes, evaluate settings like `apply_patches_on_merge` (table-level MergeTree setting) to avoid failing merges, but validate impact: it may shift patch application cost to SELECTs.
3. **Stabilize replication/merges** for `fhir.dim_practitioners`:
   - Stop repeated failing work while investigating (`SYSTEM STOP MERGES fhir.dim_practitioners`), then plan a controlled recovery (e.g., detach/replace broken parts, or rebuild the table from a clean source).
   - Avoid ad-hoc part surgery without a rollback plan; wrong part operations can cause data loss.

### 2) Reduce memory blowups from analytics

- Gate expensive dashboards:
  - Use per-user settings / profiles (`max_memory_usage`, `max_bytes_to_read`, `max_execution_time`) and concurrency control.
- Make spilling actually help:
  - Consider **lowering** `max_bytes_before_external_group_by` / `max_bytes_before_external_sort` so big group-bys start spilling earlier (validate with representative queries; spilling trades RAM for disk IO).
- Add pre-aggregation:
  - Materialized views / rollups for KPIs and “months” dashboards; avoid scanning raw facts for repeated questions.
- Validate partition pruning:
  - Ensure time filters align with partition keys (`toYYYYMM(snowflakeIDToDateTime(...))`) and avoid functions that prevent pruning.

### 3) Fix ingestion shape (too many tiny parts)

- Batch inserts to produce fewer, larger parts:
  - Increase batch size in the writer, or use `async_insert`/buffering so each insert produces meaningful blocks.
- Monitor part/merge ratios:
  - Target fewer NewParts per minute and stable merge backlog; tiny-part patterns will continually waste CPU on merges.

### 4) Reduce noisy network errors and clean up restarts

- Investigate the two peer IPs producing the majority of `Code: 210` socket errors (`10.129.57.5` / `10.129.57.15`):
  - Check load balancer health checks, timeouts, and client retry patterns.
  - Ensure clients use HTTP keep-alive correctly; avoid half-open probes that generate error spam.
- Fix startup warnings:
  - Ensure `/var/lib/clickhouse` is owned by the ClickHouse user (not root).
  - Resolve port conflicts for `:9005` (remove duplicate listener, or fix sidecar/service collision).
  - Reduce unclean restarts (graceful shutdown; correct liveness/readiness timeouts during heavy load).

---

If you want, I can also generate a small “runbook” section with exact diagnostic queries to keep for future audits (merge errors, patch-part inventory, OOM top offenders) scoped to any date window.

