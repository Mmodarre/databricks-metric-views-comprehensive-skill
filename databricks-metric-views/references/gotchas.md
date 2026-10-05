# Gotchas — Silent Bugs and Parse Errors

Read this when:
- Your metric view created without error but the numbers are wrong.
- Your metric view fails to create with a confusing error.
- You're about to ship a metric view to production.

## Correctness gotchas (silent bugs)

These don't error — they return wrong results. The worst kind.

### Silent first-match truncation

Joins default to LEFT OUTER and **many-to-one from the source side**. If your data is 1:N from source to join (header→child, dim→fact) and you forget `cardinality: one_to_many`, only the first matching row is kept per source row. No warning, no error — just wrong totals.

The directive flips the contract: declaring `cardinality: one_to_many` tells the planner the join expands rows, and per-measure aggregation is scoped to the right grain. See [fact-to-fact.md](fact-to-fact.md) for when each pattern applies.

This applies in two failure modes:

- **1:N data, default cardinality** — silently first-match-truncates. Add `cardinality: one_to_many`.
- **Genuine M:N data** (multiple parents map to the same children) — neither the default N:1 nor `cardinality: one_to_many` is correct. Declaring `one_to_many` does not satisfy any cardinality contract — restructure the data upstream rather than papering over it.

**Validate cardinality empirically before declaring a join.** Run a count comparison; if the data isn't strictly N:1 or 1:N as expected, fix the data or restructure the model. See [fact-to-fact.md](fact-to-fact.md#cardinality-validation-before-you-ship).

### Unquoted `on:` in joins

YAML 1.1 parses bare `on` as the boolean `true`. The join's predicate is silently dropped, the engine returns weird/wrong results, and you don't get an error.

```yaml
# WRONG — silent
joins:
  - name: customer
    source: catalog.schema.dim_customer
    on: source.customer_id = customer.customer_id

# RIGHT — single-quoted key
joins:
  - name: customer
    source: catalog.schema.dim_customer
    'on': source.customer_id = customer.customer_id
```

### Nested-join reference path is flat

`nation.n_name`, not `customer.nation.n_name`. The nesting only describes the join graph; column references use each join's `name:` directly. Writing `customer.nation.n_name` may resolve to a struct-extraction error or — worse — to some other column with the same prefix.

### YAML `#` comments stripped on save

```yaml
# This will disappear after the first ALTER VIEW
comment: "This won't"
```

YAML `#` comments are convenient when drafting in your editor but are not persisted across an ALTER VIEW round-trip. Use the `comment:` field for anything that should survive.

### Composed-measure aggregation order

Composed measures using `MEASURE()` correctly re-aggregate at the query's grain — that's the point. But this means **a composed measure's value at one grain is not the average of its values at a finer grain**. If you query `fill_rate` grouped by region, you get region-level fill rate; if you remove the GROUP BY and aggregate to a single row, you get the dataset's fill rate, **not** the average of regional fill rates.

This is correct behavior, but it occasionally surprises users coming from BI tools that pre-compute and naively average.

### Measure name collides with source-table column name

A `measures:` entry whose `name:` matches a physical column on the `source:` table is silently suppressed from the metric view's public measure surface. A direct `MEASURE(<colliding_name>)` query returns `UNRESOLVED_COLUMN` — the resolver's "did you mean" suggestion lists other measures, but never the colliding one.

When this metric view is then *chained* (used as `source:` by another MV), the failure mode is louder and more confusing:

```text
[MISSING_ATTRIBUTES.RESOLVED_ATTRIBUTE_APPEAR_IN_OPERATION]
Resolved attribute(s) "claim_count", "paid_claim_count", ... missing from "claim_id", ...
Attribute(s) with the same name appear in the operation: "excess_applied", "co_payment_applied".
```

The chained planner expands the base into one Project that carries both the underlying source's physical columns AND the base's measures. Colliding names break binding, and *every* measure in that Project becomes unresolvable as a secondary fault — even non-colliding ones.

**Rule**: measure names MUST NOT match column names in the `source:` table.
**Convention**: prefix dollar-total measures with `total_` (e.g. `total_excess_amount` instead of `excess_applied`); prefix counts with `<thing>_count`.

The error message's trailing line (`Attribute(s) with the same name appear in the operation`) is the diagnostic — when you see it, look for a measure name that shadows a source column.

### Inherited measures are not callable from a chained consumer

When MV B has `source: catalog.schema.A` (where A is itself a metric view), B inherits A's *dimensions* into its public surface — but **not A's measures**. `MEASURE(<A_measure>) FROM B` returns `UNRESOLVED_COLUMN`.

Inherited measures *are* reachable from inside B's locally-defined measure expressions:

```yaml
# In B (consumer of A):
measures:
  - name: paid_completion_rate
    expr: MEASURE(paid_claim_count) / NULLIF(MEASURE(claim_count), 0)
    # Both MEASURE() calls reference A's measures — this works.
```

But not from outside:

```sql
-- FAILS: UNRESOLVED_COLUMN — claim_count is in A's namespace, not B's
SELECT MEASURE(claim_count) FROM B;
```

**Workaround** — to expose an inherited measure on B's public surface, declare a thin pass-through:

```yaml
measures:
  - name: claim_count
    expr: MEASURE(claim_count)   # local re-export of A's measure
```

This is the inverse of how dimensions inherit — dimensions *are* callable on B by name; measures aren't. Plan the SSOT boundary with that asymmetry in mind.

### Window-measure GROUP BY must include the `order:` column

A measure with `window: [{order: <col>, range: <…>, semiadditive: last}]` returns 0 (or null) at every row when the GROUP BY does not include `<col>` at its native cardinality. Even GROUP BY columns *derived* from `<col>` (e.g. `CAST(date_format(date_col, 'yyyyMM') AS INT)`) do not align — the runtime treats them as independent dimensions and the rolling frame finds no rows.

```sql
-- Measure: claim_count_trailing_90d
--   order: lodged_anchor_date (DATE), range: trailing 90 days, semiadditive: last

-- WORKS: yields healthy ~6.5–9k/month
SELECT lodged_anchor_date, MEASURE(claim_count_trailing_90d)
FROM mv GROUP BY lodged_anchor_date;

-- ALL ZEROS: lodged_month is YYYYMM int, derived from lodged_anchor_date
-- but the resolver does not infer the alignment
SELECT lodged_month, MEASURE(claim_count_trailing_90d)
FROM mv GROUP BY lodged_month;
```

Document the required GROUP BY column alongside the measure (in `comment:`), or rename the order column to match the dimension consumers will actually group by.

## Parse-error gotchas (loud bugs)

These fail at create time with an error message. Knowing the real cause saves time.

### `decimal_places` scalar form

```yaml
# WRONG — fails with DecimalPlaces deserialization error
decimal_places: 2

# RIGHT
decimal_places: { type: exact, places: 2 }
```

### `use_thousands_separator` (or any other plausible name)

The field is named `hide_group_separator` (with inverted semantics). Any other name fails as an unrecognized field.

### Dimension/join name shadowing

```yaml
# WRONG — fails with INVALID_EXTRACT_BASE_FIELD_TYPE on second access
joins:
  - name: channel
    source: catalog.schema.dim_channel
    'on': source.channel_id = channel.id
dimensions:
  - name: channel               # shadows the join alias
    expr: channel.pychannel     # resolves to the dimension (a string), not the join
```

Workaround: rename the dimension and surface the natural label via `display_name` + `synonyms`.

### Nested aggregates

```yaml
# WRONG — SQL parse error
- name: had_pattern
  expr: MAX(CASE WHEN CAST(collect_list(url) AS STRING) LIKE '%pattern%' THEN 1 ELSE 0 END)
```

Push the inner aggregate into a CTE in an upstream ETL view. See [etl-vs-metric-view.md](etl-vs-metric-view.md#aggregate--pattern-matching).

### `joins:` ignored when `source:` is inline SQL

```yaml
# WRONG — joins are silently ignored
source: |
  SELECT * FROM facts WHERE year >= 2024
joins:
  - name: customer
    source: catalog.schema.dim_customer
    'on': source.customer_id = customer.id
```

Either inline the join into the SQL `source:`, or build an ETL view first and use it as `source` so YAML joins work.

## Formatting gotchas (YAML quirks)

### Names with spaces need backtick quoting in queries

```yaml
- name: 'Total Revenue'              # legacy style
```

Queried as `` MEASURE(`Total Revenue`) ``. Modernize to `name: total_revenue` + `display_name: 'Total Revenue'` to avoid the backtick gymnastics — the migration is purely cosmetic.

### Expressions starting with backtick

If your column name itself starts with a backtick (rare), wrap the entire `expr:` in double quotes:

```yaml
expr: "\`My Col\`"
```

### Expressions with colons

YAML treats colons as key separators. Wrap the expression in double quotes:

```yaml
expr: "SPLIT(o_orderpriority, '-')[1]"
```

…or use the multi-line block scalar form:

```yaml
expr: |
  SPLIT(o_orderpriority, '-')[1]
```

### Window measures and `version`

Window measures work on `version: 1.1`. Older guidance citing `version: 0.1` for windows is outdated as of April 2026.

### Divide-by-zero in composed ratios

Composed ratios can return errors (or surprising NaN/Infinity) when the denominator is zero for a particular dimension slice. Wrap denominators in `NULLIF(..., 0)`:

```yaml
- name: fill_rate
  expr: MEASURE(filled_orders) / NULLIF(MEASURE(total_orders), 0)
```
