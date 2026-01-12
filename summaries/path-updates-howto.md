## ClickHouse SQL-style UPDATE/DELETE (patch parts): operator HOWTO

### Prerequisites / compatibility (as described in the article)
- **Version:** introduced as **experimental in 25.7** (the article notes a plan for **beta in 25.8**).
- **Tables:** MergeTree-family tables; per-table settings must add internal “row id” columns used by patch parts:
  - `enable_block_number_column = 1`
  - `enable_block_offset_column = 1`
- **Feature flags (session-level):** `allow_experimental_lightweight_update = 1` (and enable delete behavior as applicable in your version).

### Safe rollout checklist
1) **Canary first:** enable on one table (or one shard/replica set) and run your real UPDATE/DELETE workload.
2) **Set hard limits:** cap concurrency and memory for update jobs; keep OLAP queries isolated if possible.
3) **Watch part growth:** patch parts are still “parts”; alert on rising parts-per-partition and merge backlog.
4) **Plan cleanup:** schedule/trigger patch materialization (`APPLY PATCHES`) during low-traffic windows.
5) **Fallback plan:** know how to disable the feature flags and revert to mutation-based `ALTER TABLE … UPDATE/DELETE` if errors appear.

### Enable (minimal)
```sql
SET allow_experimental_lightweight_update = 1;

ALTER TABLE orders
  MODIFY SETTING enable_block_number_column = 1, enable_block_offset_column = 1;
```

### Use (minimal examples)
```sql
UPDATE orders
SET quantity = 60, discount = 0.20
WHERE order_id = 1001 AND item_id = 'mouse';
```

```sql
DELETE FROM orders
WHERE order_id = 1001 AND item_id = 'mouse';
```

### Optional operational knobs (when you need stricter behavior)
- `update_parallel_mode`:
  - `auto` (default): serializes dependent updates; runs others in parallel.
  - `sync`: runs updates one at a time.
- `update_sequential_consistency = 1`: each update sees the latest visible state (costs performance).
- `lightweight_delete_mode = lightweight_update`: makes `DELETE` behave as patch-part delete (`_row_exists = 0`) until materialized.

### Materialize / clean up patch parts
```sql
ALTER TABLE orders APPLY PATCHES;
```

### Verification (operator queries)
- **Spot-check visibility (should be immediate to SELECTs):**
```sql
SELECT quantity, discount
FROM orders
WHERE order_id = 1001 AND item_id = 'mouse';
```
- **Track patch/part pressure (core risk):** monitor parts per partition and merge backlog for the canary table(s).

### Gotchas / limits called out in the article
- Patch parts can drive `TOO_MANY_PARTS` if they accumulate (they’re cleaned up after materialization/merges).
- If original parts are merged away before patch application, ClickHouse may fall back to a **hash join** on `(_block_number, _block_offset)` (slower + more memory-hungry; patch must fit in memory).
- Multiple UPDATEs that touch different column sets can create more patch parts than you expect (part counts can rise).
