# ETL Views vs Metric Views — What Belongs Where

Not everything belongs in a metric view's YAML. The right architecture has an upstream ETL layer doing the data prep and the metric view doing the dimensional slicing.

## The boundary

| Concern | Belongs in upstream ETL view | Belongs in metric view YAML |
|---|---|---|
| Window functions for **dedup or sequencing** (`ROW_NUMBER`, `LAG`, `LEAD`) | yes | no — YAML `window:` is for cumulative/trailing/semiadditive, not row-by-row sequencing |
| Deduplication | yes | no |
| **Multi-step** transformations (CTE chains) | yes | no |
| Aggregate + LIKE patterns (`MAX(CASE WHEN ... collect_list(x) LIKE ...)`) | yes | no — nested aggregates not allowed |
| **Grain-change pre-aggregation** — promoting a child attribute up to parent grain via `MIN_BY` / `FIRST` / window-then-pick | yes | no |
| Cross-fact joins (header→child, or independent facts conformed by date/period) | no — declare in YAML | yes — see `cardinality: one_to_many` in [fact-to-fact.md](fact-to-fact.md) |
| Dimensional slicing (GROUP BY any combination) | no | yes — this is the whole point |
| Composed rates and ratios with `MEASURE()` | no | yes |
| Persistent filters (`WHERE` applied to every query) | no | yes — use the `filter:` field |
| Semantic metadata (`display_name`, `format`, `synonyms`) | no | yes |
| Star/snowflake joins to dimension tables | no (don't pre-join in ETL — let the metric view be lazy) | yes |

## The recommended architecture

```
Raw bronze tables
        │
        ▼
ETL view (SQL) — minimum transformation: dedup, event sequencing, outcome
                reclassification. Joins ONLY what's needed for the
                transformation. Passes foreign keys through for downstream.
        │
        ▼
Metric view (YAML) — `source:` is the ETL view. Star/snowflake `joins:` to
                     dimension tables. `filter:` for persistent scope.
                     Dimensions from source + joined tables. Measures use
                     `COUNT(DISTINCT ...) FILTER (WHERE ...)` for clean
                     atomic-then-composed metrics.
```

## Decision rule

> Ask: **"Is this part of the metric definition, or is it just data prep?"**
>
> - Part of the metric definition (how revenue is calculated, what counts as a "fulfilled" order, the rate's numerator and denominator, fact-to-fact composition) → metric view.
> - Just data prep (dedup, ordering, grain-changing pre-aggregation, working around source data quality) → ETL view.

When the answer isn't obvious, prefer the ETL view. It's easier to debug a SQL view than a metric view, and easier to materialize / observe / unit-test.

## Common ETL-required patterns

### Window-based dedup

```sql
-- ETL view: keep latest event per (id, type)
CREATE OR REPLACE VIEW gold.v_latest_events AS
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY id, type ORDER BY event_ts DESC) AS rn
  FROM silver.raw_events
)
SELECT *
FROM ranked
WHERE rn = 1;
```

The metric view sources from `gold.v_latest_events` and never sees the dedup logic.

### Event sequencing (LAG / LEAD)

```sql
-- ETL view: each row knows the previous outcome
CREATE OR REPLACE VIEW gold.v_event_sequence AS
SELECT
  id,
  event_ts,
  outcome,
  LAG(outcome) OVER (PARTITION BY entity_id ORDER BY event_ts) AS previous_outcome
FROM gold.v_latest_events;
```

### ROW_NUMBER + LAG ordering bug

A common mistake is putting both `ROW_NUMBER` (for dedup) and `LAG` (for sequencing) in the same CTE — `LAG` then sees the pre-dedup rows and produces phantom previous-outcome values. Always dedup first, then `LAG` on the deduped result:

```sql
-- WRONG
WITH cte AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY id ORDER BY ts DESC) AS rn,
         LAG(outcome) OVER (PARTITION BY entity_id ORDER BY ts) AS prev_outcome
  FROM ...
)
SELECT * FROM cte WHERE rn = 1;
-- Bug: prev_outcome was computed against pre-dedup rows.

-- RIGHT
WITH dedup AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY ts DESC) AS rn FROM ...
),
sequenced AS (
  SELECT *, LAG(outcome) OVER (PARTITION BY entity_id ORDER BY ts) AS prev_outcome
  FROM dedup WHERE rn = 1
)
SELECT * FROM sequenced;
```

### Aggregate + pattern matching

SQL forbids aggregates inside aggregates. If you need `LIKE`-on-a-collected-list logic, split into two ETL steps:

```sql
-- WRONG (in a view)
SELECT id, MAX(CASE WHEN CAST(collect_list(url) AS STRING) LIKE '%pattern%' THEN 1 ELSE 0 END)
FROM ... GROUP BY id;

-- RIGHT
WITH collected AS (
  SELECT id, CAST(collect_list(url) AS STRING) AS url_array
  FROM ... GROUP BY id
)
SELECT id, CASE WHEN url_array LIKE '%pattern%' THEN 1 ELSE 0 END AS matched
FROM collected;
```

The metric view sources from a view that already has the `matched` column.

## Why this boundary matters

Pushing complex SQL into the metric view's `source:` (as an inline query) **disables YAML `joins:`** — metric views ignore top-level joins when the source is a query. So a design with both pre-aggregation in `source:` and dimensional joins in YAML doesn't work; you must choose one or the other.

Cross-fact composition that used to require an inline-SQL `source:` (pre-aggregating two facts at a shared grain, then joining) is now solved by `cardinality: one_to_many` on join entries — see [fact-to-fact.md](fact-to-fact.md) Pattern C. The remaining ETL-only triggers are the ones above: row-by-row dedup/sequencing, multi-step CTE chains, aggregate+pattern-matching, and grain-changing pre-aggregation (e.g. `MIN_BY` to promote a child attribute up to parent grain).
