# Level of Detail (LOD) Expressions

LOD expressions decouple the **aggregation grain** from the **query grain**, enabling "% of total", dataset-wide aggregates, and partition-relative measures. Two distinct mechanisms cover different needs.

## When to use which

| Need | Mechanism | Section |
|------|-----------|---------|
| Aggregate at a **fixed** grain regardless of query — independent of query filters | Window function in source SQL | [Fixed LOD](#fixed-lod) |
| Aggregate at a **coarser** grain than the query, **respecting query filters** | `range: all` window measure | [Coarser LOD](#coarser-lod) |

## Fixed LOD

Embed a window function in the inline-SQL `source:` and expose the result as a dimension. The aggregate is computed once at source-evaluation time, before any query-time filter is applied.

```yaml
version: 1.1
source: |
  SELECT
    o_orderkey,
    o_orderpriority,
    o_totalprice,
    o_orderdate,
    SUM(o_totalprice) OVER (PARTITION BY o_orderpriority) AS priority_total_price
  FROM samples.tpch.orders

dimensions:
  - name: order_priority
    expr: o_orderpriority
    display_name: 'Order Priority'
  - name: order_date
    expr: o_orderdate
    display_name: 'Order Date'
  - name: priority_total_price
    expr: priority_total_price        # exposed as dimension so measures can read it

measures:
  - name: total_sales
    expr: SUM(o_totalprice)
    display_name: 'Total Sales'

  - name: pct_of_priority_total
    expr: SUM(o_totalprice) / NULLIF(ANY_VALUE(priority_total_price), 0)
    display_name: '% of Priority Total'
    format:
      type: percentage
      decimal_places: { type: exact, places: 2 }
```

### When Fixed LOD is the right tool

Reach for Fixed LOD when:

- **Static partitioning across all uses** — a value computed at a fixed grain that must remain stable regardless of how consumers query (e.g., "this priority's all-time total" anchored to the source dataset, not the user's current filter).
- **Dataset-wide aggregates as comparison anchors** — global totals or averages that row-level metrics are compared against. Use empty `OVER ()`.
- **Multi-level hierarchies with detail and rollup** — when a single view needs both detail rows and a coarser pre-computed roll-up (e.g., per-priority total alongside per-order detail) and the coarser value should not respond to query filters.

If your "% of total" or aggregate-of-aggregate needs to respect the user's WHERE clause, jump to **Coarser LOD** below instead.

### Three rules for Fixed LOD

1. **Wrap LOD dimensions in an aggregate when used in a measure.** Use `ANY_VALUE` when the value is constant within a group: `SUM(x) / ANY_VALUE(priority_total_price)`. Without an aggregate wrapper, the SQL won't compile inside a measure expression.
2. **Filters do not narrow the partition.** A query-time `WHERE priority_total_price > 1000000` does **not** re-compute the partition — the value was set when the source SQL ran. To filter pre-aggregation, embed the WHERE clause in the source SQL itself.
3. **Dataset-wide aggregates use empty `OVER ()`.** Useful for "% of grand total" patterns:

   ```sql
   SUM(o_totalprice) OVER () AS grand_total_price
   ```

## Coarser LOD

Define a measure with a `window: range: all` block. The measure aggregates ignoring the query's GROUP BY on the named dimension — and **respects query-time filters**, unlike Fixed LOD.

```yaml
version: 1.1
source: samples.tpch.orders

dimensions:
  - name: order_priority
    expr: o_orderpriority

measures:
  - name: total_sales
    expr: SUM(o_totalprice)

  - name: all_priorities_sales              # ignores order_priority groupings
    expr: SUM(o_totalprice)
    window:
      - order: order_priority
        range: all
        semiadditive: last

  - name: pct_of_total_sales
    expr: SUM(o_totalprice) / NULLIF(MEASURE(all_priorities_sales), 0)
    format:
      type: percentage
      decimal_places: { type: exact, places: 2 }
```

When the user queries with `GROUP BY order_priority`, `total_sales` is per-priority but `all_priorities_sales` is the unconditional total — so `pct_of_total_sales` correctly returns each priority's share.

If the query also has `WHERE order_date >= '2024-01-01'`, `all_priorities_sales` reflects only 2024 orders. **This is the key behavioral difference from Fixed LOD.**

### Multiple dimensions to ignore

Use multiple window entries:

```yaml
- name: all_categories_all_regions_sales
  expr: SUM(amount)
  window:
    - order: category
      range: all
      semiadditive: last
    - order: region
      range: all
      semiadditive: last
```

## Choosing between fixed and coarser

- **Fixed LOD** = compute once at source-load time. Faster on repeat queries. Use for stable partitions that should not respond to user filters (e.g., "this priority's all-time total" for context).
- **Coarser LOD** = compute at query time, filter-aware. Use for "% of total in the user's selection" — the standard BI requirement.

When in doubt, prefer coarser. Fixed LOD's filter-blind behavior is a frequent surprise for end users.
