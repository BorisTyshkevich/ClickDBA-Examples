# ClickHouse performance analysis — 2026-01-08 before 10:00 (UTC)

## Scope

- **Server:** `chi-ai-ai-0-0-0` (ClickHouse `25.12.2.54`)
- **Time zone:** server/session timezone is **UTC**
- **Analysis window:** **2026-01-08 00:00:00Z** ≤ `event_time` < **2026-01-08 10:00:00Z**
- **Focus:** errors, SELECT/INSERT performance, ingestion behavior, merges & mutations (incl. lightweight update/delete), root cause analysis (RCA), and upstream GitHub matches.

## Executive summary

Two independent problem classes dominated this window:

1) **Severe CPU + memory pressure during a benchmark/load run**, leading to hundreds of query failures with `MEMORY_LIMIT_EXCEEDED` (code **241**) and additional failures/cancellations as clients dropped/reset connections.

2) **Patch-parts (lightweight update/delete) related logical errors (code 49)** that:
   - Broke reads (and therefore several INSERT pipelines that read from affected tables),
   - And created a **stuck background MERGE_PARTS task** for `fhir.dim_practitioners` that has continued retrying thousands of times.

If you address only one thing, address **(2)** first: it causes persistent background errors and blocks merges/queue progress on `fhir.dim_practitioners`.

---

## System/resource context (from system logs)

- **OS memory total:** ~**15.31 GiB** (`system.asynchronous_metric_log` / `OSMemoryTotal`)
- **Resident memory (window):** **max 12.39 GiB**, **avg 2.96 GiB**, **min 700.86 MiB**
- **CPU quota:** `CGroupMaxCPU = 3.8` (effectively ~4 vCPU)
- **Key settings (current):**
  - `max_memory_usage = 13529146982` (~**12.6 GiB** per query)
  - `enable_lightweight_update = 1`, `enable_lightweight_delete = 1`
  - `allow_experimental_lightweight_update = 1`, `allow_experimental_lightweight_delete = 1`
  - `apply_patch_parts = 1`

Interpretation:

- The box has ~15 GiB RAM but appears CPU-constrained (~4 vCPU). Under high concurrency and “scan-heavy” queries, CPU saturation + memory pressure are expected.

---

## Errors and anomalies

### Query-level exceptions (system.query_log)

Exceptions in-window (by `exception_code`):

- **241** — `MEMORY_LIMIT_EXCEEDED`: **318** occurrences
- **49** — `LOGICAL_ERROR` (patch parts / RangeReader / patch stream): **78** occurrences
- **236** — client dropped connection (`ABORTED`): **40** occurrences
- **47** — unknown identifier (ad-hoc query): **3** occurrences
- **735** — query cancelled by client: **2** occurrences

Time concentration:

- Errors were concentrated almost entirely in **08:00Z–09:00Z**.

### Server log errors (system.text_log)

`system.text_log` had **27,009** `Error` messages in the window, dominated by network errors:

- `ServerErrorHandler`: **25,554** errors (mostly `Code: 210` “Socket is not connected” / “Connection reset by peer”)
- `TCPHandler` / `executeQuery`: hundreds of `Code: 236` (client dropped connection, query cancelled)

Interpretation:

- This pattern is typical of load generators/clients timing out or abruptly closing connections while the server is overloaded (not necessarily a server-side networking fault).

### Merge/replication queue errors (system.part_log + system.replication_queue)

`fhir.dim_practitioners` had repeated merge failures:

- **79** failed `MergeParts` tasks in-window for a single target part name: `202312_0_28_3_41`
- Failure signature: **Code 49** — “Block structure mismatch in patch parts stream: different names of columns …”

This also shows up as a **stuck replication queue entry**:

- `system.replication_queue` (`fhir.dim_practitioners`): `MERGE_PARTS` to `202312_0_28_3_41`
  - `parts_to_merge`: `['202312_0_13_2_24', '202312_16_28_1_29']`
  - `num_tries`: **4109** (as of **2026-01-11 09:55Z**)
  - `last_exception`: `Code: 49 … Block structure mismatch in patch parts stream …`

Impact:

- Persistent background retries (log spam, wasted CPU)
- Merge progress blocked for the affected range on `dim_practitioners`
- Elevated risk of **part explosion** (eventually `TOO_MANY_PARTS`), because merges can’t reduce parts effectively.

---

## SELECT performance (system.query_log, QueryFinish)

In-window SELECT summary:

- **Finished:** 4,608
- **Failed:** 360
- **Latency:** avg **8.08s**, p50 **4.37s**, p95 **29.35s**, max **87.08s**
- **Read size per query:** avg **227 MiB**, p95 **1.51 GiB**, max **3.69 GiB**
- **Total read (SELECT finished):** **~1022 GiB**

Top tables by total read (note: queries may count multiple tables, so totals can overlap):

- `fhir.fact_procedures`: **828.58 GiB**
- `fhir.fact_encounters`: **698.27 GiB**
- `fhir.fact_medications`: **633.16 GiB**
- `fhir.fact_conditions`: **622.29 GiB**

Dominant “heavy hitter” query shapes:

- A small set of normalized queries accounted for a large fraction of IO, e.g. the top normalized query hash ran **105** times and read **~385.83 GiB** total.

Schema observation relevant to these queries:

- `fhir.fact_*` tables are ordered by `(patient_id, <id>)` and partitioned by `toYYYYMM(snowflakeIDToDateTime(<id>))`.
- Many benchmark queries aggregate by *domain datetime columns* (e.g., `performed_datetime`, `recorded_date`) rather than by `<id>` timestamp or by `patient_id`, leading to **weak partition pruning** and **poor primary-key alignment**, i.e. large scans.

---

## INSERT performance + ingestion behavior

In-window INSERT summary:

- **Finished:** 108
- **Failed:** 79
- **Latency:** avg **696ms**, p95 **4.57s**
- **Total written:** **13.05 MiB** / **205,823 rows**

INSERT failures were dominated by **Code 49**:

- “Min/max part offset must be set in RangeReader for reading patch parts … MergeTreeSelect(pool: ReadPoolInOrder …)”
- These failures occurred in INSERTs that **read from tables with patch parts enabled** (e.g., reading `fact_encounters` during INSERT into another fact table).

### Part creation patterns (system.part_log, NewPart)

Signs of micro-batching / part explosion:

- `fhir.dim_practitioners`: **1078** new parts, avg **42 rows/part** (avg part size ~**2.49 KiB**)
- `fhir.dim_locations`: **1078** new parts, avg **41 rows/part**
- `fhir.dim_patients`: **1029** new parts, avg **46 rows/part**
- Fact tables had fewer parts but still relatively small per-part row counts in some cases (partition fan-out contributes).

Current active parts snapshot (now, not time-windowed):

- `fhir.dim_practitioners`: **1140** active parts across **72** partitions (plus **999** active patch parts), despite only ~29 MiB on disk.

Interpretation:

- Extremely small inserts (tens of rows) create merge pressure and operational risk.
- With a stuck merge task, `dim_practitioners` is at elevated risk of accumulating parts faster than merges can reduce them.

---

## Updates / Deletes (lightweight update/delete behavior)

This workload performed many `UPDATE` and `DELETE` statements:

- **UPDATE:** 188 finished / 2 failed; avg **1.39s**, p95 **5.04s**, total read **68.21 GiB**
- **DELETE:** 187 finished / 0 failed; avg **1.05s**, p95 **3.37s**, total read **63.83 GiB**

Notably:

- `system.part_log` showed **no `MutatePart` events** in-window, consistent with **lightweight update/delete producing patch parts** rather than classic mutations.
- Patch parts in `fhir.dim_practitioners` include different “shapes”, e.g.:
  - Patch parts that include updated columns (e.g. `specialty` with `specialty.dict` + `specialty` substreams),
  - And patch parts that include only internal columns (`_row_exists`, `_part_offset`, etc.), consistent with delete markers.

This heterogeneity is strongly correlated with the observed “patch parts stream mismatch” merge failure.

---

## Root cause analysis (RCA)

### RCA #1 — Memory pressure + CPU saturation during benchmark

Evidence:

- `MEMORY_LIMIT_EXCEEDED` (code 241) was the most common query failure.
- Resident memory peaked at **12.39 GiB** in-window, close to `max_memory_usage` (~**12.6 GiB**).
- `CGroupMaxCPU = 3.8` indicates ~4 vCPU available; load average peaked at **32**, consistent with a saturated run queue.
- Query patterns were highly scan-heavy (p95 read ~**1.51 GiB** per SELECT; max **3.69 GiB**) and ran at high frequency.

Likely mechanism:

- Multiple concurrent scan-heavy SELECT/UPDATE/DELETE queries competed for limited CPU, increased wall time, and pushed memory toward the total limit, causing OvercommitTracker / memory limit failures and client-side timeouts/disconnects.

### RCA #2 — Patch parts (lightweight update/delete) bug leading to logical errors and stuck merges

Evidence:

- Repeated code **49** exceptions:
  - Query-side: “Min/max part offset must be set in RangeReader for reading patch parts … ReadPoolInOrder …”
  - Merge-side: “Block structure mismatch in patch parts stream: different names of columns …”
- A single stuck `MERGE_PARTS` entry in `system.replication_queue` for `fhir.dim_practitioners` has retried **4100+** times since **2026-01-08 08:51:47Z**.
- Patch parts exist in multiple tables; `dim_practitioners` is the extreme case with **~1000** active patch parts.

Likely mechanism:

- A known bug in the patch-parts / lightweight update/delete pipeline triggers:
  - A read-path logical error in the “read in order” execution path, and
  - A merge-path incompatibility when patch parts of different “shapes” (updated columns vs delete markers) are combined, producing a stream schema mismatch.

---

## Upstream GitHub matches

These errors match existing upstream reports:

- `ClickHouse/ClickHouse#89836` — “Logical error: 'Min/max part offset must be set in RangeReader for reading patch parts' …” (open)
- `ClickHouse/ClickHouse#89472` — “Block structure mismatch in patch parts stream” (open)

Both point to the patch-parts / lightweight updates area as the likely upstream root for the code 49 errors.

---

## Deep dive: LOGICAL_ERROR (49) — patch parts usage verification

This section verifies whether the cluster/table configuration is *consistent with intended patch UPDATE/DELETE usage*, and documents why the observed errors are likely upstream bugs rather than schema misuse.

### 1) Feature flags and core settings (server/session)

At the time of analysis (current settings on the node):

- Patch updates/delete are enabled:
  - `enable_lightweight_update = 1`
  - `enable_lightweight_delete = 1`
  - `allow_experimental_lightweight_update = 1`
  - `allow_experimental_lightweight_delete = 1`
  - `apply_patch_parts = 1` (patch parts are applied on SELECT)
- Read-in-order optimization is enabled:
  - `optimize_read_in_order = 1`
- Lightweight delete mode default (server) is:
  - `lightweight_delete_mode = alter_update`
  - (but the workload frequently overrode this per-query; see below)

Conclusion: patch parts were enabled in the expected way, and the read path was using the in-order algorithm that appears in the exception stack traces.

### 2) Table prerequisites: row-id columns present

All `fhir` MergeTree-family tables involved in SQL-style UPDATE/DELETE had the required MergeTree settings embedded in `create_table_query`:

- `enable_block_number_column = 1`
- `enable_block_offset_column = 1`

Conclusion: schema-level prerequisites for patch parts were met.

### 3) Code 49 “Min/max part offset must be set in RangeReader…” (read path)

The failing INSERT pipeline queries show the exact failure signature:

- `Min/max part offset must be set in RangeReader for reading patch parts`
- `MergeTreeSelect(pool: ReadPoolInOrder, algorithm: InOrder)`

Additionally, `system.query_log` shows that the failing INSERTs carried (as changed settings):

- `lightweight_delete_mode = lightweight_update_force`
- `optimize_read_in_order` is enabled server-side (`= 1`)

Interpretation:

- This aligns with `ClickHouse/ClickHouse#89836`, which reports the same logical error in the patch parts reader on recent versions.

Operational mitigation (diagnostic + potential workaround):

- For sessions that *read* tables with patch parts (including INSERT…SELECT pipelines), try:
  - `set optimize_read_in_order = 0;`
  - If the error disappears, it confirms the issue is in the in-order read path.

### 4) Code 49 “Block structure mismatch in patch parts stream…” (merge/materialization path)

`fhir.dim_practitioners` accumulated two disjoint classes of active patch parts (as of analysis time):

- **588** patch parts containing updated columns: `{specialty, last_updated}` (update patches)
- **411** patch parts containing only `'_row_exists'` (delete patches)

In partition `202312`, patch parts were split across two patch-partition ids:

- `patch-116ec7151bbfe7ea649b23811e916438-202312`: **14** parts (update patches)
- `patch-f18f7271629a324b0d26b6ad0b83a6c2-202312`: **122** parts (delete patches)

The background/replication merge is stuck on a specific MERGE_PARTS task:

- `system.replication_queue` (`fhir.dim_practitioners`):
  - `type = MERGE_PARTS`
  - `new_part_name = 202312_0_28_3_41`
  - `num_tries ≈ 4100+` (observed **4109** on 2026-01-11 09:55Z)
  - `last_exception`: `Code: 49 … Block structure mismatch in patch parts stream …`

Interpretation:

- The table is mixing patch parts that represent **updates** (new values for specific columns) and patch parts that represent **deletes** (`_row_exists` markers).
- This aligns with `ClickHouse/ClickHouse#89472`, which reports the same “block structure mismatch” while selecting/reading with patch parts enabled, and in similar versions.

Operational mitigations:

- Avoid `lightweight_delete_mode = lightweight_update_force` for now; prefer `lightweight_update` (fallback) or `alter_update` to reduce exposure to patch-delete edge cases.
- Periodically materialize patch parts during low-traffic windows:
  - `alter table <t> apply patches`
- If this is benchmark/test data: drop & recreate the affected table(s) to clear stuck queue entries and patch-part buildup.

---

## Recommendations

### Immediate (today)

1) **Stop the bleeding from the stuck merge**
   - If this is a test dataset: **drop & recreate** `fhir.dim_practitioners` (fastest way to clear the stuck queue entry and patch parts).
   - If not test data: temporarily **stop replication queues** for this table to stop retry storms, then plan a controlled remediation (upgrade or data repair).

2) **Disable lightweight update/delete (patch parts) for now**
   - Set (session or globally): `enable_lightweight_update=0`, `enable_lightweight_delete=0`
   - Consider also setting `apply_patch_parts=0` only as a diagnostic tool (not as a long-term correctness-preserving setting).
   - Track/upgrade once upstream fixes land (see GitHub issues above).

3) **Reduce benchmark concurrency to match the environment**
   - The server appears CPU-capped (~4 vCPU). Running many concurrent scan-heavy queries will amplify latency and memory pressure.
   - Cap concurrency and/or enforce `max_threads` at ~4 for heavy queries to reduce run-queue contention.

### Short-term (this week)

1) **Upgrade ClickHouse to a build that includes fixes for patch-parts issues**
   - Re-test the exact workload that triggered code 49 after upgrade.

2) **Eliminate micro-batch ingestion**
   - Change loaders to insert larger batches (target **10k–1M rows** per insert for fact tables; avoid “tens of rows” inserts for dimensions).
   - Avoid inserting data that fans out across many partitions in a single INSERT (group/flush by partition key).

3) **Add guardrails**
   - Set query-level memory caps and concurrency limits appropriate to 15 GiB RAM / 4 vCPU.
   - Add alerting on:
     - `system.replication_queue` entries with high `num_tries`
     - `system.part_log` merge failures (`error != 0`)
     - growth in active parts (`system.parts`) and patch parts

### Long-term (schema & workload alignment)

1) **Align partitioning/ordering to dominant query predicates**
   - Current fact tables are ordered by `(patient_id, <id>)`, but many analytics queries filter/aggregate by domain event times (`performed_datetime`, `recorded_date`, etc.).
   - Consider:
     - Repartitioning by the dominant event-time column (monthly) to improve pruning, and/or
     - Adding projections keyed by event time to accelerate month/day aggregates.

2) **Replace scan-heavy “KPI” queries with pre-aggregations**
   - Materialized views or rollups by month/city/org can reduce per-query reads from GiB to MiB.

---

## Appendix: key evidence queries (for reproducibility)

- Exceptions: `system.query_log` filtered to the window and `type like 'Exception%'`
- Latency/IO summaries: `system.query_log` aggregated by `query_kind`, `normalized_query_hash`, and table sets
- Ingestion/merges: `system.part_log` (`NewPart`, `MergeParts`) and active parts (`system.parts`)
- Stuck merge: `system.replication_queue` for `fhir.dim_practitioners` (type `MERGE_PARTS`, high `num_tries`)
- Resources: `system.asynchronous_metric_log` (`MemoryResident`, `OSMemoryTotal`, `LoadAverage*`, `CGroupMaxCPU`)
