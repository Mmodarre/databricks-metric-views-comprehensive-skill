---
name: databricks-metric-views
description: "Unity Catalog metric views: define, create, alter, and query governed business metrics in YAML. Use this skill whenever the user mentions metric views, semantic layers, KPI definitions, the MEASURE() function, the WITH METRICS / LANGUAGE YAML syntax, fact-to-fact joins, fan-out or level-of-detail problems, materialization of metric views, or wants reusable business metrics shared across Genie, Lakeview, and SQL — even if they don't explicitly say 'metric view'."
---

# Unity Catalog Metric Views

Define reusable, governed business metrics in YAML. Measures and dimensions are declared once on top of your gold-layer tables; consumers query any combination at runtime.

## When to use this skill

- Defining standardized KPIs (revenue, fill rate, churn, fulfillment) shared across Genie, Lakeview Dashboards, and SQL
- Star/snowflake schemas where joins should live in the metric definition rather than every query
- Composed metrics — ratios, rates, and derived KPIs built on atomic aggregations
- Window measures — moving averages, running totals, period-over-period, YTD, semi-additive balances
- Level-of-detail problems — % of total, dataset-wide aggregates, filter-aware coarser aggregations
- Multi-fact problems — `invoice` ↔ `invoice_line`, `sales` ↔ `marketing`, or any two facts conformed by date / period — using `cardinality: one_to_many` joins
- Authoring metric views from MCP tools, the SQL editor, or a DABs CI/CD pipeline


## Quickstart

```sql
CREATE OR REPLACE VIEW catalog.schema.orders_metrics
WITH METRICS
LANGUAGE YAML
AS $$
version: 1.1
comment: "Orders KPIs for sales analysis"
source: catalog.schema.orders
filter: order_date > '2020-01-01'

dimensions:
  - name: order_month
    expr: DATE_TRUNC('MONTH', order_date)
    display_name: 'Order Month'
  - name: order_status
    expr: |
      CASE
        WHEN status = 'O' THEN 'Open'
        WHEN status = 'P' THEN 'Processing'
        WHEN status = 'F' THEN 'Fulfilled'
      END
    display_name: 'Order Status'

measures:
  # Atomic
  - name: order_count
    expr: COUNT(1)
    display_name: 'Order Count'
  - name: total_revenue
    expr: SUM(total_price)
    display_name: 'Total Revenue'
    format:
      type: currency
      currency_code: USD
      decimal_places:
        type: exact
        places: 2

  # Composed
  - name: avg_order_value
    expr: MEASURE(total_revenue) / NULLIF(MEASURE(order_count), 0)
    display_name: 'Average Order Value'
    format:
      type: currency
      currency_code: USD
$$
```

Query:

```sql
SELECT
  order_month,
  order_status,
  MEASURE(order_count),
  MEASURE(total_revenue),
  MEASURE(avg_order_value)
FROM catalog.schema.orders_metrics
WHERE EXTRACT(YEAR FROM order_month) = 2024
GROUP BY ALL
ORDER BY 1
```

## Authoring workflow

1. **Inspect the source** — call `get_table_stats_and_schema` (MCP) or `DESCRIBE EXTENDED` to see available columns.
2. **Define dimensions** — categorical attributes for GROUP BY / WHERE. Use `name: snake_case` + `display_name: 'Title Case'`.
3. **Define atomic measures first** — simple `SUM`, `COUNT`, `AVG`, `COUNT(DISTINCT)` aggregations.
4. **Compose derived measures** — ratios and rates built with `MEASURE()` references. *If `total_revenue` ever changes (e.g., excludes tax), `avg_order_value` automatically uses the updated definition.*
5. **Add joins** — see [references/joins.md](references/joins.md) for star/snowflake; for two facts at different grains see [references/fact-to-fact.md](references/fact-to-fact.md).
6. **Add semantic metadata** — `display_name`, `synonyms`, `format` for Genie/Lakeview. See [references/semantic-metadata.md](references/semantic-metadata.md).
7. **Verify** — query the view, check totals, cross-check against a known-good number from the underlying tables.

### Composability best practices

When building composed measures, follow this order:

1. **Define atomic measures first** — `SUM`, `COUNT`, `AVG`, `COUNT(DISTINCT)`. One aggregation per measure. Keep the SQL minimal.
2. **Use `MEASURE()` for references** — NEVER inline the aggregation a second time in another measure. If `total_revenue` changes (e.g., excludes tax), `avg_order_value` automatically follows.
3. **Prioritize readability** — write composed measures as the literal business formula: `MEASURE(gross_profit) / MEASURE(total_revenue)`, not a re-derivation from raw columns.
4. **Apply semantic metadata to composed measures** — `format: type: percentage` for rates, `format: type: currency` for value-derived metrics. Genie/Lakeview render correctly only when the format is declared on the composed result.

## Querying rules

- Wrap every measure in `MEASURE()`. `SELECT *` is **not** supported.
- `GROUP BY ALL` is supported and idiomatic.
- Joins at query time are not supported — declare them in the YAML, or wrap two metric views in CTEs and join the CTE results.
- Modern v1.1 names (snake_case identifiers) need no backticks. Names with spaces require backticks: `` MEASURE(`Total Revenue`) ``.

## Operations

```sql
-- Create
CREATE OR REPLACE VIEW mv WITH METRICS LANGUAGE YAML AS $$ ... $$;

-- Alter (replace YAML)
ALTER VIEW mv AS $$ ... $$;

-- Describe (human)
DESCRIBE TABLE EXTENDED mv;

-- Describe (JSON, for programmatic use)
DESCRIBE TABLE EXTENDED mv AS JSON;

-- Grant
GRANT SELECT ON mv TO `analytics-consumers`;

-- Drop
DROP VIEW mv;
```

For programmatic authoring, the `manage_metric_views` MCP tool exposes `create`, `alter`, `describe`, `query`, `drop`, and `grant` actions — see [references/examples.md](references/examples.md).

## Gotchas you must know before authoring

These are correctness or parse-error traps. Read [references/gotchas.md](references/gotchas.md) before debugging unexpected results.

- **Quote `'on':` in joins.** Bare `on:` is parsed as boolean `true` in YAML 1.1 — silent join-condition loss.
- **Silent first-match truncation in 1:N joins without `cardinality: one_to_many`.** Joins default to N:1 from the source side; 1:N data (header→child, dim→fact) silently keeps only the first match per source row unless you declare `cardinality: one_to_many`. Genuine M:N data isn't fixed by either — restructure upstream. Validate cardinality before you join.
- **`decimal_places` is a nested object**, not a scalar. `decimal_places: 2` fails with a deserialization error.
- **Use `hide_group_separator: false`**, not `use_thousands_separator: true`.
- **Dimension names must not collide with join names** — a dimension shadows the join alias and downstream `join_name.col` references break.
- **YAML `#` comments are stripped on save** — use the `comment:` field instead.
- **No nested aggregates** — `MAX(... collect_list(x) ...)` won't compile; split into a CTE in an upstream view.
- **Top-level `joins:` are ignored when `source:` is an inline SQL query** — pre-join in the SQL or build an ETL view first.
- **Measure names must not collide with source-table column names** — silently suppresses the measure on the standalone surface, breaks chained consumers loudly with `MISSING_ATTRIBUTES.RESOLVED_ATTRIBUTE_APPEAR_IN_OPERATION`. Use `total_<thing>_amount` / `<thing>_count` prefixes.
- **Inherited measures from a chained MV are not callable from outside** — `MEASURE(<inherited>) FROM consumer` returns `UNRESOLVED_COLUMN`. They're only reachable from inside the consumer's local measure expressions, or via a thin pass-through re-export.
- **Window measures need their `order:` column in the GROUP BY** — otherwise the rolling frame returns 0/null silently. Derived columns of the same date (e.g. YYYYMM int derived from a DATE) do NOT align.

## Reference index

Each file is a self-contained deep dive. Load when relevant.

- **[references/yaml-reference.md](references/yaml-reference.md)** — Full top-level YAML field reference: `version`, `source` (table/view/SQL/MV), `filter`, `dimensions`, `measures`, `joins`, `materialization`. Load when authoring or debugging YAML structure.
- **[references/joins.md](references/joins.md)** — Star and snowflake (nested) joins, `ON` vs `USING`, joined-column reference syntax, MAP-column constraint, name-shadowing trap. Load when modeling dimensional data.
- **[references/fact-to-fact.md](references/fact-to-fact.md)** — Three patterns for two facts at different grains: anchor on fine grain (Pattern A), anchor on parent with `cardinality: one_to_many` children (Pattern B), and anchor on a shared dim with multiple facts as `cardinality: one_to_many` joins (Pattern C). Plus the `rely.at_most_one_match` perf hint. Load whenever the user mentions two fact tables, a header/line relationship, or independent facts conformed by date/period.
- **[references/semantic-metadata.md](references/semantic-metadata.md)** — `display_name`, `synonyms`, `format` (currency, percentage, number, date), `decimal_places` nested object, `hide_group_separator`. Load when adding Genie/Lakeview-friendly metadata.
- **[references/window-measures.md](references/window-measures.md)** — Trailing/cumulative/leading windows, semiadditive measures, period-over-period, YTD via multi-window. All examples use `version: 1.1`. Load when the user wants moving averages, running totals, or balance-style measures.
- **[references/level-of-detail.md](references/level-of-detail.md)** — Fixed LOD via window functions in source SQL; coarser LOD via `range: all`. The "% of total" pattern. Load when the user mentions partition-by, % of total, dataset aggregates, or independent grain.
- **[references/materialization.md](references/materialization.md)** — Pre-compute aggregations via Lakeflow. Schedule, mode, aggregated/unaggregated types, refresh. Load when query latency is a concern or the user mentions materialization.
- **[references/etl-vs-metric-view.md](references/etl-vs-metric-view.md)** — When to push logic into an upstream ETL view rather than the metric YAML (dedup, ROW_NUMBER+LAG, complex multi-table joins). Load when the source has window functions, dedup, or non-trivial transformations, or you're combining inline-SQL source: with YAML joins: and hitting silently-ignored behavior.
- **[references/gotchas.md](references/gotchas.md)** — Every YAML trap and silent-correctness bug in one file. Load when debugging unexpected behavior.
- **[references/examples.md](references/examples.md)** — Worked end-to-end YAML examples: simple KPI, star schema, snowflake, composable, SQL-query-as-source, MV-as-source, MCP authoring. Load when scaffolding a new metric view from scratch.
- **[references/deployment.md](references/deployment.md)** — Catalog/schema layout, GRANTS, naming conventions, CI/CD with Databricks Asset Bundles (job + SQL-task pattern, parameterized variables). Load when productionizing or deploying via DABs.

## Resources

- [Metric Views Documentation](https://docs.databricks.com/en/business-semantics/metric-views/)
- [YAML Syntax Reference](https://docs.databricks.com/en/business-semantics/metric-views/syntax)
- [Joins](https://docs.databricks.com/en/business-semantics/metric-views/joins)
- [Advanced Techniques](https://docs.databricks.com/aws/en/business-semantics/metric-views/advanced-techniques)
- [Level of Detail](https://docs.databricks.com/aws/en/business-semantics/metric-views/level-of-detail)
- [Materialization](https://docs.databricks.com/en/business-semantics/metric-views/materialization)
- [MEASURE() Function](https://docs.databricks.com/en/sql/language-manual/functions/measure)
