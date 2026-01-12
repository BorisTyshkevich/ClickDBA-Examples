# LOGICAL_ERROR deep-dive: patch parts (SQL UPDATE/DELETE) vs on-the-fly mutations — 2026-01-08 before 10:00 UTC

Scope: ClickHouse server `chi-ai-ai-0-0-0` (tz `UTC`), version `25.12.2.54`. Time window: **2026-01-08 00:00:00–09:59:59 UTC**.

This report is *only* about the `LOGICAL_ERROR` failures related to patch parts (lightweight SQL UPDATE/DELETE) and the merge/read paths that interact with patch parts.

## 1) First: terminology / features (what’s what)

There are two related—but distinct—features in ClickHouse:

### A) SQL-style UPDATE/DELETE using **patch parts** (“lightweight updates/deletes”)

From `summaries/path-updates-howto.md` (operator summary of ClickHouse’s SQL-style updates):

- **Goal:** make `UPDATE ... WHERE ...` and `DELETE FROM ... WHERE ...` visible immediately without waiting for classic mutation materialization.
- **Mechanism:** write **patch parts** + apply them during reads (and optionally during merges), then later **materialize** them via `ALTER TABLE ... APPLY PATCHES`.
- **Prereqs:** MergeTree family tables with:
  - `enable_block_number_column = 1`
  - `enable_block_offset_column = 1`
- **Feature flags:** `allow_experimental_lightweight_update=1` (and delete-related flags/settings).
- **Gotcha explicitly called out:** patch parts accumulating → `TOO_MANY_PARTS`; and patch application can fall back to a hash join on `(_block_number, _block_offset)` (more memory).

### B) **On-the-fly mutations** (docs page you linked)

From the ClickHouse docs page you linked (`on-the-fly mutations`):

- This is about applying **classic mutations** (e.g., `ALTER TABLE ... UPDATE/DELETE`) “on the fly” during SELECT, controlled by `apply_mutations_on_fly` (and a couple of mutation execution settings for subqueries/nondeterminism).
- It’s not the same as patch parts, but it matters because both features are “mutations visible at read time” and they share some conceptual/operational constraints (subqueries/nondeterminism can be tricky; applying changes at read time has perf impact).

**Important fact for this server:** `apply_mutations_on_fly = 0`. So, the `LOGICAL_ERROR` we saw is not because on-the-fly mutations were enabled; it is occurring in the patch-part implementation paths.

## 2) What “LOGICAL_ERROR” occurred (and where)

In the window, `system.query_log` shows exactly one `exception_code=49` pattern:

### A) Query-time LOGICAL_ERROR (read path used by INSERT … SELECT)

**Error text:** `Min/max part offset must be set in RangeReader for reading patch parts`

Counts (initial queries, before 10:00 UTC):

- `exception_code=49`: **78** events
- All 78 are **INSERT** queries failing while reading MergeTree data with patch parts (message includes `MergeTreeSelect(pool: ReadPoolInOrder, algorithm: InOrder)`).

Grouped examples (from `system.query_log`), all in the same error family:

```sql
-- 22 failures
INSERT INTO fact_conditions (...)
... FROM fact_encounters ...
-- LOGICAL_ERROR: Min/max part offset must be set in RangeReader for reading patch parts
```

```sql
-- 20 failures
INSERT INTO fact_observations (...)
... FROM fact_encounters ...
-- LOGICAL_ERROR: Min/max part offset must be set in RangeReader for reading patch parts
```

```sql
-- 19 failures
INSERT INTO fact_procedures (...)
... FROM fact_encounters ...
-- LOGICAL_ERROR: Min/max part offset must be set in RangeReader for reading patch parts
```

```sql
-- 17 failures
INSERT INTO fact_medications (...)
... FROM fact_encounters ...
-- LOGICAL_ERROR: Min/max part offset must be set in RangeReader for reading patch parts
```

Note: none of the query-log `LOGICAL_ERROR` events in this window mention `dim_practitioners` in their `tables` list; they all involve `fact_encounters` + the destination fact table.

### B) Merge-time LOGICAL_ERROR (background merges)

Separately, `system.part_log` shows merge failures with `Code: 49`:

- `fhir.dim_practitioners`: **79** `MergeParts` failures, exception starts with:
  - `Block structure mismatch in patch parts stream: different names of columns: specialty ..., last_updated ...`

This is a background merge path failure (not a direct query failure in `query_log`).

## 3) Why it “touches only dim_practitioners” (it doesn’t—there are 2 issues)

There are two different `LOGICAL_ERROR` problems:

1) **RangeReader “min/max part offset”** failures hit the **fact ingestion** queries (`INSERT … SELECT` reading `fact_encounters`).
2) **Merge-time “block structure mismatch”** failures hit **merges** for `dim_practitioners`.

If your question is specifically “why are merge failures only on `dim_practitioners`?”:

### Key differentiator: `dim_practitioners` is a “hot partition with many small parts”, driving merges + patch application

Current state of active **base parts** by partition (from `system.parts`, excluding patch parts):

- `fhir.dim_practitioners`:
  - partition `202312`: **118 parts**, **225,626 rows**
  - other partitions: ~**1 part** each, ~8–9k rows
- `fhir.dim_locations`: partitions are ~**1 part** each (very few merges needed)
- `fhir.dim_organizations`: partitions are ~**1 part** each (very few merges needed)

So `dim_practitioners` is the only dimension table where a single partition (`202312`) accumulated **many parts**, causing the background merge machinery to run frequently *while patch parts exist* and to execute “apply patches on merge” code paths often.

In contrast, other dimension tables *also* had patch parts in the time window, but their base partitions were largely already consolidated (1 part/partition), so merges weren’t constantly combining patched parts in the same way.

## 4) “Correct usage” verification checklist (schema + settings + query patterns)

### A) Schema prerequisites for patch parts (per howto)

Verified for relevant `fhir.*` tables (examples shown in `SHOW CREATE TABLE` output for `dim_practitioners` and other tables):

- `ENGINE = ReplicatedMergeTree(...)`
- Table settings include:
  - `enable_block_number_column = 1`
  - `enable_block_offset_column = 1`

Also verified at the physical-part level (e.g. `system.parts_columns` for base parts) that `_block_number` and `_block_offset` exist and are stored in parts.

### B) Session/server settings relevant to patch parts & on-the-fly mutations

From `system.settings` (global defaults):

- Patch parts feature flags:
  - `allow_experimental_lightweight_update = 1`
  - `allow_experimental_lightweight_delete = 1`
  - `enable_lightweight_update = 1`
  - `enable_lightweight_delete = 1`
  - `apply_patch_parts = 1`
- On-the-fly mutations:
  - `apply_mutations_on_fly = 0`  (so not in play here)
- Mutation execution knobs (relevant when subqueries/nondeterminism exist):
  - `mutations_execute_subqueries_on_initiator = 0`
  - `mutations_execute_nondeterministic_on_initiator = 0`
- Update concurrency knobs:
  - `update_parallel_mode = auto`
  - `update_sequential_consistency = 1`

Additionally, **the UPDATE/DELETE sessions in the window set**:

- `lightweight_delete_mode = lightweight_update_force`

…which is consistent with “force patch-part behavior for DELETE where possible”.

### C) Query pattern differences across tables (examples for discussion)

All of these were observed in the 00:00–10:00 window as successful `QueryFinish` entries in `system.query_log`.

**Dimension tables (random row selection via nondeterministic subquery):**

```sql
UPDATE dim_practitioners
SET
    specialty = constAt('spec', (cityHash64(practitioner_id, toUInt64(now64()) + 7) % constLen('spec')) + 1),
    last_updated = now()
WHERE practitioner_id IN (
    SELECT practitioner_id
    FROM dim_practitioners
    ORDER BY rand()
    LIMIT 1000
);

DELETE FROM dim_practitioners
WHERE practitioner_id IN (
    SELECT practitioner_id
    FROM dim_practitioners
    ORDER BY rand()
    LIMIT 1000
);
```

```sql
UPDATE dim_locations
SET
    status = constAt('status', (cityHash64(location_id, toUInt64(now64()) + 5) % constLen('status')) + 1),
    last_updated = now()
WHERE location_id IN (SELECT location_id FROM dim_locations ORDER BY rand() LIMIT 1000);

DELETE FROM dim_locations
WHERE location_id IN (SELECT location_id FROM dim_locations ORDER BY rand() LIMIT 1000);
```

```sql
UPDATE dim_organizations
SET
    status = constAt('status', (cityHash64(organization_id, toUInt64(now64()) + 7) % constLen('status')) + 1),
    last_updated = now()
WHERE organization_id IN (SELECT organization_id FROM dim_organizations ORDER BY rand() LIMIT 1000);

DELETE FROM dim_organizations
WHERE organization_id IN (SELECT organization_id FROM dim_organizations ORDER BY rand() LIMIT 1000);
```

**Fact tables (partition-filtered deletes; minmax indexes exist):**

```sql
DELETE FROM fact_observations
WHERE observation_id IN (
    SELECT observation_id
    FROM fact_observations
    WHERE toYYYYMM(snowflakeIDToDateTime(observation_id)) = '202412'
    ORDER BY rand()
    LIMIT 10000
);
```

**Ingestion queries that triggered the RangeReader LOGICAL_ERROR (reads `fact_encounters`):**

```sql
INSERT INTO fact_conditions (...)
WITH (SELECT count() FROM fact_encounters) AS encounter_total
SELECT ...
FROM fact_encounters AS enc
...
-- LOGICAL_ERROR: Min/max part offset must be set in RangeReader for reading patch parts
```

### D) Key structural difference: fact tables have minmax skip indexes; dim tables don’t

From `system.data_skipping_indices`:

- `fact_encounters`: `idx_encounter_start_day_mm` (minmax on `toStartOfDay(start_datetime)`)
- `fact_observations`: `idx_observation_effective_day_mm`
- `fact_procedures`: `idx_procedure_performed_day_mm`
- `fact_medications`: `idx_medication_authored_day_mm`
- `fact_conditions`: `idx_condition_recorded_day_mm`

No skipping indices are present for the `dim_*` tables.

This aligns extremely well with upstream bug reports that mention patch parts + indexes triggering read-path issues.

## 5) Root cause hypotheses (with confidence)

### Hypothesis 1 (high confidence): ClickHouse bug in patch-part reader when skip indexes / InOrder readers are involved

Evidence:

- Query-log error text is consistently: `Min/max part offset must be set in RangeReader for reading patch parts`
- The failing reads use `ReadPoolInOrder` / `InOrder`
- The tables being read (`fact_encounters`, etc.) have minmax skipping indexes

Upstream match:

- `ClickHouse/ClickHouse#89836` — *Logical error: 'Min/max part offset must be set in RangeReader for reading patch parts' with LWU and index* (open).

### Hypothesis 2 (high confidence): ClickHouse bug in patch application during merges (block structure mismatch)

Evidence:

- `system.part_log` shows repeated merge failures for `fhir.dim_practitioners` with `Code: 49` and `Block structure mismatch in patch parts stream ...`
- The error mentions exactly the updated columns (`specialty`, `last_updated`), consistent with patch-application code paths.

Upstream match:

- `ClickHouse/ClickHouse#89472` — *Block structure mismatch in patch parts stream* (open).

### Hypothesis 3 (medium): workload/shape amplifies bug visibility for `dim_practitioners`

Evidence:

- `dim_practitioners` has a “hot” partition with **many parts** → constant merges.
- Patch parts + constant merges means the “apply patches on merge” path is exercised a lot more than other dimension tables.

This doesn’t create the bug, but it makes it show up earlier and more consistently.

## 6) Suggested verification / next steps

### A) Confirm correct / safe query patterns for replicated execution

Your UPDATE/DELETE patterns use nondeterminism (`rand()`, `now64()`) and subqueries. Even if that’s “okay” for a demo workload, in real replicated setups it can cause correctness/replica divergence issues unless queries are rewritten or initiator execution settings are used.

Concrete follow-ups:

- Avoid `ORDER BY rand()` inside subqueries for mutations; replace with a deterministic sampling method (e.g., select IDs based on a stable hash and a fixed threshold).
- If you must keep subqueries/nondeterminism, evaluate mutation initiator settings (`mutations_execute_subqueries_on_initiator`, `mutations_execute_nondeterministic_on_initiator`) and verify behavior for SQL UPDATE/DELETE in your version (these settings are documented for mutations; confirm they apply to patch-part UPDATE/DELETE in your environment).

### B) Mitigate / isolate the two LOGICAL_ERROR families

1) For the **RangeReader** bug (fact ingestion reads):
   - Short term: avoid patch-part UPDATE/DELETE on `fact_encounters` while ingestion runs, or temporarily disable `apply_patch_parts` for ingestion sessions (if correctness allows; typically risky).
   - Medium term: upgrade to a ClickHouse version containing a fix for `#89836` (track backports).

2) For the **merge-time block mismatch** (dim_practitioners):
   - Reduce merge pressure: batch inserts so `dim_practitioners` does not create huge numbers of tiny parts in the same partition.
   - Materialize patches during low traffic: `ALTER TABLE fhir.dim_practitioners APPLY PATCHES` (validate in staging first).
   - Upgrade ClickHouse (track `#89472`).

### C) Minimal reproduction you can use for upstream / internal debugging

For `#89836`-style issues:

- Table with `enable_block_number_column=1`, `enable_block_offset_column=1`
- At least one minmax skipping index
- Perform a few SQL `UPDATE`/`DELETE` to create patch parts
- Run an in-order read path (queries that force `ReadPoolInOrder`), and capture whether `Min/max part offset...` reproduces.

For `#89472`-style issues:

- Create patch parts that update a LowCardinality column + DateTime (like `specialty` + `last_updated`)
- Ensure the table has ongoing merges in a single hot partition.

---

If you want, I can add an appendix with exact “copy/paste” SQL used to pull every number in this report (the diagnostic queries against `system.*`), so you can re-run it for any date range.

