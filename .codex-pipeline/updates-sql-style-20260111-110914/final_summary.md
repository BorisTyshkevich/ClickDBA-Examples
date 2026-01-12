## ClickHouse UPDATE/DELETE: practical how-to

### 1) Pick the mechanism you’re using
- **Classic mutation (MergeTree):** `ALTER TABLE … UPDATE` / `ALTER TABLE … DELETE` (mutation-based). Runs **asynchronously**; results may not be visible until the background mutation finishes. Heavyweight because affected columns are rewritten for matching parts.
- **On-the-fly mutations:** can make updates **visible immediately** by applying expressions at read time while parts rewrite in the background; too many accumulated updates can slow `SELECT` and has limits around subqueries/non-determinism.
- **Lightweight UPDATE/DELETE (patch parts):** SQL `UPDATE … SET … WHERE …` and `DELETE … WHERE …` create **patch parts** (row identifiers + new values). Visibility is **immediate to queries** (“patch-on-read”), with later background merges/materialization applying changes to data parts.

### 2) Enable lightweight UPDATE if you’re on ClickHouse 25.7
In **25.7**, lightweight updates are **experimental** (expected **beta** in **25.8**) and must be enabled, plus table settings:
```sql
SET allow_experimental_lightweight_update = 1;

ALTER TABLE orders
  MODIFY SETTING enable_block_number_column = 1, enable_block_offset_column = 1;
```

### 3) Run UPDATE / DELETE
**UPDATE (standard SQL syntax):**
```sql
UPDATE orders
SET quantity = 60, discount = 0.20
WHERE order_id = 1001 AND item_id = 'mouse';
```

**DELETE (lightweight-update mode uses `_row_exists = 0` via patch parts):**
```sql
DELETE FROM t
WHERE /* predicate */ ;
```

### 4) Optional operational knobs (ordering/concurrency/consistency)
- `update_parallel_mode`:
  - `auto` (default): serializes dependent updates; runs others in parallel.
  - `sync`: runs updates one at a time.
- `update_sequential_consistency = 1`: each update sees the latest visible state (performance cost). Default is `0`.
- `lightweight_delete_mode = lightweight_update`: makes `DELETE` behave as patch-part delete (`_row_exists = 0`) until merged/materialized.

### 5) Materialize patch parts when needed
If patch parts accumulate and you want to force them into data parts:
```sql
ALTER TABLE t APPLY PATCHES;
```

### 6) Key gotchas to plan for
- Patch parts are **regular parts** and can contribute to `TOO_MANY_PARTS` (per partition); they’re cleaned up after materialization/merges.
- If original parts get merged away before patch application, ClickHouse may fall back to a **hash join** on `(_block_number, _block_offset)`, which is slower and more memory-hungry (patch must fit in memory).
- Multiple UPDATEs touching different column sets can create separate patch-part groupings (part counts can rise in unexpected ways).

## Verification
- Check row visibility immediately after UPDATE:
```sql
SELECT *
FROM orders
WHERE order_id = 1001 AND item_id = 'mouse';
```
- For broad predicates, spot-check a few qualifying rows (e.g., `WHERE quantity >= 40`) to confirm changes.
- For lightweight deletes, confirm queries no longer return deleted rows right away (ignored via `_row_exists = 0`), and run `ALTER TABLE … APPLY PATCHES` if you need to ensure changes are fully materialized.
