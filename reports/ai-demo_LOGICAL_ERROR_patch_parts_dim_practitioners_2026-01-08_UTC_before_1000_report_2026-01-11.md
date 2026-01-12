# LOGICAL_ERROR analysis (patch parts / LWU): `fhir.dim_practitioners` and related failures

Date written: **2026-01-11**  
Server (ai-demo connector): `chi-ai-ai-0-0-0`, ClickHouse **25.12.2.54**, timezone **UTC**.

This report focuses on **2026-01-08 before 10:00 UTC** and the **LOGICAL_ERROR (Code: 49)** family observed with the new SQL-style UPDATE/DELETE (“patch parts”, aka lightweight update/delete).

## Background: two different “mutation” features

ClickHouse has two separate concepts that are easy to mix up:

1) **On-the-fly mutations** (docs page the user referenced): this is about `ALTER TABLE ... UPDATE/DELETE` (classic mutations) and the ability to apply them during SELECTs without materializing the full mutated part (`apply_mutations_on_fly` / `apply_mutations_on_fly_timeout`).  
2) **SQL-style `UPDATE` / `DELETE` using patch parts (LWU)**: this is the newer feature (experimental in ~25.7, evolving quickly) that writes **patch parts** and applies them on reads/merges. It requires table settings:
   - `enable_block_number_column = 1`
   - `enable_block_offset_column = 1`
   and is commonly used with `lightweight_delete_mode = 'lightweight_update_force'`.

The LOGICAL_ERRORs in this report are clearly from **(2)** (“patch parts”) based on the exception texts and stack traces.

## What failed before 10:00 UTC on 2026-01-08

There were **two distinct Code 49 problems** active in this window:

### A) Read-path Code 49 (RangeReader / in-order reads)

`system.query_log` shows **78 exceptions** between **08:37:36Z and 09:06:34Z**:

> `Min/max part offset must be set in RangeReader for reading patch parts: While executing MergeTreeSelect(pool: ReadPoolInOrder, algorithm: InOrder). (LOGICAL_ERROR)`

These were thrown while running **INSERT ... SELECT** into fact tables, where the SELECT side used MergeTree “InOrder” reading (likely because of `optimize_read_in_order=1` and/or indexes). Example (truncated from `system.query_log`):

```sql
INSERT INTO fact_conditions (...) WITH (SELECT count() FROM fact_encounters) ...
-- Exception: Min/max part offset must be set in RangeReader for reading patch parts ...
```

Upstream tracking: **ClickHouse/ClickHouse#89836** (“Min/max part offset ... RangeReader ... with LWU and index”).

### B) Merge-path Code 49 (patch stream block structure mismatch) on `dim_practitioners`

`system.part_log` shows **79 failed merges** on `fhir.dim_practitioners` between **08:51:47Z and 09:59:30Z** (within the requested window), all with:

> `Block structure mismatch in patch parts stream: different names of columns: specialty ... last_updated ... (LOGICAL_ERROR)`

And this continued through the day (806 merge error events total on 2026-01-08).

This is a **background merge failure**, so it won’t necessarily show up as a failed user query unless you run an `OPTIMIZE` / `ALTER` that waits on the merge.

Upstream tracking: **ClickHouse/ClickHouse#89472** (“Block structure mismatch in patch parts stream”).

## Why does the merge bug “touch only” `dim_practitioners`?

Schema and query *shape* across the `dim_*` tables is very similar (all are ReplicatedMergeTree, same block columns enabled, same style of UPDATE+DELETE workload). The standout difference is **part topology and patch pressure**, especially in partition **202312**:

### Evidence: `system.parts` (active parts in base month 202312)

```sql
-- snapshot taken on 2026-01-11
WITH parts AS (
  SELECT
    table,
    if(startsWith(partition, 'patch-'), extract(partition, '-(\\d{6})$'), partition) AS base_month,
    startsWith(partition, 'patch-') AS is_patch,
    count() AS parts,
    sum(rows) AS rows
  FROM system.parts
  WHERE database='fhir'
    AND table IN ('dim_practitioners','dim_patients','dim_locations','dim_organizations')
    AND active
  GROUP BY table, base_month, is_patch
)
SELECT
  table, base_month,
  sumIf(parts, is_patch=0) AS base_parts,
  sumIf(rows,  is_patch=0) AS base_rows,
  sumIf(parts, is_patch=1) AS patch_parts,
  sumIf(rows,  is_patch=1) AS patch_rows
FROM parts
WHERE base_month='202312'
GROUP BY table, base_month
ORDER BY table;
```

Result (key rows):

- `dim_locations`: base_parts=**1**, patch_parts=**0**
- `dim_organizations`: base_parts=**1**, patch_parts=**0**
- `dim_patients`: base_parts=**2**, patch_parts=**2**
- `dim_practitioners`: base_parts=**118**, patch_parts=**136**

Further breakdown for `dim_practitioners` in 202312 (two patch “kinds”):

- Update patch partition `patch-116ec7151bbfe7ea649b23811e916438-202312`: **14 parts** (columns include `specialty`, `last_updated`)
- Delete patch partition `patch-f18f7271629a324b0d26b6ad0b83a6c2-202312`: **122 parts** (columns include `_row_exists`)

### Interpretation (likely root cause trigger)

The “block structure mismatch” error is triggered during merge-time patch application. While the underlying ClickHouse bug is upstream, it needs a **triggering shape**:

- many **Compact** parts in the same partition,
- lots of accumulated **patch parts** (both update + delete),
- constant churn (new patch parts created while merge selection keeps trying).

`dim_practitioners` is the only `dim_*` table that reached this high-pressure state in 202312, so it’s the only one that reliably trips the merge-time bug.

## Verify: schema/settings and workload queries (correct LWU usage?)

### Table settings

`fhir.dim_practitioners` has the required per-table settings enabled:

```sql
SHOW CREATE TABLE fhir.dim_practitioners;
-- includes: SETTINGS enable_block_number_column = 1, enable_block_offset_column = 1
```

So the table is configured correctly to *use* patch parts.

### Workload queries (examples)

From `system.query_log` on **2026-01-08** (examples are consistent across all `dim_*` tables; differences are only the updated column name and the PK name):

`dim_practitioners` UPDATE (note: `now64()` + `now()` are nondeterministic):

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
```

`dim_practitioners` DELETE (note: `ORDER BY rand()` is nondeterministic):

```sql
DELETE FROM dim_practitioners
WHERE practitioner_id IN (
    SELECT practitioner_id
    FROM dim_practitioners
    ORDER BY rand()
    LIMIT 1000
);
```

The same query structure exists for `dim_patients`, `dim_locations`, `dim_organizations`.

### Practical correctness notes (even if not the root cause)

- LWU supports nondeterminism only when explicitly allowed; your workload sets `allow_nondeterministic_mutations=1` and uses `lightweight_delete_mode='lightweight_update_force'`. That is internally consistent.
- Operationally, nondeterministic UPDATE/DELETE patterns can increase patch churn and make results harder to reason about during incident response. They are not *required* for benchmarking the feature.

## Repro attempt (ai-demo) and current status

Goal: reproduce merge-time Code 49 by running inserts + updates + deletes concurrently with merges (as suggested).

Because `clickhouse-client` cannot reach the server directly from this host, the repro was executed via the **ai-demo SQL connector** using a dedicated database: `repro_patchparts`.

### What was tried

1) Targeted table with patch parts and forced merge:
   - created multiple base parts
   - created update patch parts (`specialty`, `last_updated`) and delete patch parts (`_row_exists`)
   - ran `OPTIMIZE ... FINAL`
   - result: **no Code 49**

2) “Race” attempt: run `OPTIMIZE ... FINAL` concurrently with UPDATE/DELETE/INSERT using parallel requests:
   - result: **no Code 49**

3) Compact-only table attempt (all parts forced `Compact`) with 40 base parts, then concurrent `OPTIMIZE` + multiple updates/deletes:
   - result: **no Code 49**

### Conclusion so far

- The production failure is real and well evidenced in `system.part_log` and aligns with upstream issue **#89472**.
- A deterministic, minimal repro *on this single node* has **not** been found yet; the trigger seems more specific than “any concurrent merge + patch churn”.

## Recommendations

### Immediate mitigations (stability-first)

1) **Disable LWU for `dim_practitioners`** (or stop using SQL `UPDATE/DELETE`) until upgrading to a version with confirmed fixes for #89472/#89836. Use classic mutations (`ALTER TABLE ... UPDATE/DELETE`) if you must, and/or move deletes to TTL where possible.
2) Reduce patch/part pressure:
   - schedule `ALTER TABLE ... APPLY PATCHES` during low-traffic windows (after validating it completes successfully in your version),
   - reduce UPDATE/DELETE frequency and batch more work per statement if possible.
3) Avoid “in-order reads” on tables that currently have patch parts until the RangeReader bug is fixed (see #89836). If your workload forces this (e.g. `optimize_read_in_order=1`), disable it temporarily for affected queries/jobs.

### Medium-term (once stable)

1) **Upgrade ClickHouse** (or apply backport) once upstream fixes for:
   - `Block structure mismatch in patch parts stream` (**#89472**)
   - `Min/max part offset must be set in RangeReader...` (**#89836**)
   are available in a release you can deploy.
2) Add operational SLOs/alerts specific to LWU:
   - patch parts count per partition,
   - merge failures in `system.part_log` (error!=0),
   - “ReadPoolInOrder” exceptions in `system.query_log`.

## Upstream references (GitHub)

- ClickHouse/ClickHouse#89472 — “Block structure mismatch in patch parts stream” (issue)  
  https://github.com/ClickHouse/ClickHouse/issues/89472
- ClickHouse/ClickHouse#89836 — “Min/max part offset must be set in RangeReader for reading patch parts … with LWU and index” (issue)  
  https://github.com/ClickHouse/ClickHouse/issues/89836

