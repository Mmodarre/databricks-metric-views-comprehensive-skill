# Metric View YAML Reference

Complete reference for the Unity Catalog metric view YAML specification.

## Top-level fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `version` | No | string | Spec version. Use `'1.1'` (DBR 17.2+, default). `'0.1'` for DBR 16.4–17.0. |
| `source` | Yes | string | Source data — see [Source types](#source-types). |
| `comment` | No | string | Description of the metric view (v1.1+). Surfaced in Unity Catalog. |
| `filter` | No | string | SQL boolean applied as a global WHERE to every query. |
| `dimensions` | Yes | list | At least one dimension definition. |
| `measures` | Yes | list | At least one measure definition. |
| `joins` | No | list | Star/snowflake schema joins. See [joins.md](joins.md). |
| `materialization` | No | object | Pre-computation config. See [materialization.md](materialization.md). |

## Source types

The `source:` field accepts four kinds of input:

```yaml
# 1. Three-part table name (most common)
source: catalog.schema.fact_orders

# 2. Three-part view name
source: catalog.schema.v_orders_clean

# 3. Inline SQL query (for pre-aggregation, multi-fact pre-join, complex joins)
source: |
  SELECT o.*, c.c_name, c.c_mktsegment
  FROM samples.tpch.orders o
  LEFT JOIN samples.tpch.customer c ON o.o_custkey = c.c_custkey
  WHERE o.o_orderdate >= '1995-01-01'

# 4. Another metric view (cross-view composition / layered metrics)
source: catalog.schema.base_orders_metrics
```

### Important constraints

- **Top-level `joins:` does NOT work when `source:` is an inline SQL query.** YAML joins are silently ignored or error. If you need both pre-aggregation and dimensional joins, create an ETL view first and use it as `source:`. See [etl-vs-metric-view.md](etl-vs-metric-view.md).
- **MV-as-source is the recommended pattern for "department-specific" metric views** that build on a shared base layer. Keep composition to 2–3 layers — deeper nesting becomes hard to debug. See [examples.md](examples.md) for cross-view composition.

## Dimensions

```yaml
dimensions:
  - name: region                         # Identifier — snake_case, no spaces
    expr: region_name                    # Direct column ref
    display_name: 'Sales Region'         # Title Case for UIs (v1.1)
    comment: 'Geographic sales region'   # Documentation (v1.1)
    synonyms: ['territory', 'geo']       # Genie discovery (v1.1)

  - name: order_month
    expr: DATE_TRUNC('MONTH', order_date)
    display_name: 'Order Month'

  - name: order_year
    expr: EXTRACT(YEAR FROM order_month)  # Can reference earlier dimensions

  - name: customer_segment
    expr: |                              # Multi-line scalar with `|`
      CASE
        WHEN total_amount > 100000 THEN 'Enterprise'
        WHEN total_amount > 10000 THEN 'Mid-Market'
        ELSE 'SMB'
      END
    display_name: 'Customer Segment'

  - name: customer_name
    expr: customer.c_name                # Reference joined table column
```

### Dimension rules

- `name` is the queryable identifier. **Use snake_case** so queries don't need backticks.
- `expr` is a SQL expression that returns a scalar.
- Dimensions can reference: source columns, SQL functions, CASE expressions, **other earlier-defined dimensions** (by `name:`), and **joined columns** as `join_name.column` for top-level joins or `parent.child.column` for nested joins (hierarchical path — see [joins.md](joins.md)).
- Dimensions **cannot** use aggregate functions — those belong in measures.

## Measures

```yaml
measures:
  # Atomic — direct aggregation
  - name: total_revenue
    expr: SUM(total_price)
    display_name: 'Total Revenue'
    comment: 'Sum of all order totals'

  - name: order_count
    expr: COUNT(1)

  - name: unique_customers
    expr: COUNT(DISTINCT customer_id)

  # Filtered — conditional aggregation via FILTER (WHERE ...)
  - name: open_revenue
    expr: SUM(total_price) FILTER (WHERE status = 'O')
    comment: 'Revenue at risk from unfulfilled orders'

  # Filtered ratio — apply FILTER to EACH aggregate independently
  - name: avg_open_value
    expr: >
      SUM(total_price) FILTER (WHERE status = 'O')
      / NULLIF(COUNT(1) FILTER (WHERE status = 'O'), 0)

  # Composed — built from other measures via MEASURE()
  - name: avg_order_value
    expr: MEASURE(total_revenue) / NULLIF(MEASURE(order_count), 0)
    format:
      type: currency
      currency_code: USD
      decimal_places: { type: exact, places: 2 }

  - name: revenue_per_customer
    expr: MEASURE(total_revenue) / NULLIF(MEASURE(unique_customers), 0)
```

### Measure rules

- `name` is queried via `MEASURE(name)` (or `` MEASURE(`Display Name`) `` for legacy views with spaces in names).
- `expr` must contain an aggregate function (SUM, COUNT, AVG, MIN, MAX, COUNT DISTINCT, etc.) **or** be composed of other measures via `MEASURE()`.
- Composed measures **re-aggregate correctly at any grain**. If a user queries `avg_order_value` grouped by region, the engine computes `total_revenue` and `order_count` per region first, then divides — it does **not** average pre-computed averages.
- Wrap denominators in `NULLIF(..., 0)` to avoid divide-by-zero.
- `SELECT *` on metric views is **not supported**. Always use `MEASURE()` explicitly.

## Filter

A global WHERE applied to every query against this metric view.

```yaml
filter: order_date > '2020-01-01'

# Multiple conditions
filter: order_date > '2020-01-01' AND status != 'CANCELLED'

# Using joined columns
filter: customer.is_active = true
```

Use this for **scoping** — exclude test data, limit to recent years, drop deleted records. The filter is applied before any user-supplied WHERE clause.

## Comments

Two kinds of comments, with different lifetimes:

```yaml
# This YAML comment is STRIPPED on save (v1.1+) — do NOT use for documentation
comment: "This `comment:` field IS persisted and shown in Unity Catalog"
```

Always use the `comment:` field. YAML `#` comments are convenient while drafting but disappear after the first ALTER VIEW round-trip.

## Joins entry fields

A `joins:` entry accepts:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Alias used to reference joined columns (`name.column`) and the slot in nested-join paths. |
| `source` | Yes | Three-part name of the joined table or view. |
| `'on'` | One of `'on'` / `using` | SQL boolean predicate. **Must be quoted** — bare `on:` parses as `true`. |
| `using` | One of `'on'` / `using` | List of column names shared by source and join. |
| `cardinality` | No | `one_to_many` when the join is 1:N from source to join (header→child, shared dim→fact). Default is N:1 from source side. See [fact-to-fact.md](fact-to-fact.md). |
| `rely` | No | Object with `at_most_one_match: true` to assert N:1 and skip defensive dedup logic (perf hint, see [fact-to-fact.md](fact-to-fact.md)). |
| `joins` | No | Nested joins for snowflake schemas. Each hop must satisfy its own cardinality contract. |

## Joins, materialization, semantic metadata, windows, LOD

These have dedicated reference files:

- [joins.md](joins.md) — star/snowflake joins, ON vs USING, name-shadowing trap
- [fact-to-fact.md](fact-to-fact.md) — three patterns for two facts at different grains, `cardinality:` mechanics
- [materialization.md](materialization.md) — Lakeflow-backed pre-aggregation
- [semantic-metadata.md](semantic-metadata.md) — `display_name`, `synonyms`, `format`
- [window-measures.md](window-measures.md) — moving/cumulative/semiadditive measures
- [level-of-detail.md](level-of-detail.md) — fixed and coarser LOD patterns
