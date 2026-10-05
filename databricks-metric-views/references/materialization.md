# Materialization

Pre-compute metric view aggregations using Lakeflow Spark Declarative Pipelines under the hood. Speeds up high-frequency dashboard queries.

> **Status**: Experimental. Requires DBR 17.2+ and serverless compute enabled.

## When to materialize

- Dashboard queries that hit the same dimension/measure combinations repeatedly.
- Source views whose evaluation is expensive (many joins, complex transformations).
- Latency-sensitive consumers where re-aggregating from raw facts on every query is too slow.

If query latency is acceptable without materialization, don't bother — materialization adds operational cost (a Lakeflow pipeline to maintain).

## Anatomy

```yaml
materialization:
  schedule: every 6 hours              # same syntax as MV REFRESH SCHEDULE
  mode: relaxed                        # only "relaxed" is supported today
  materialized_views:
    - name: baseline
      type: unaggregated               # full data model: source + joins + filter
    - name: daily_category_summary
      type: aggregated                 # pre-computed for specific dim/measure combo
      dimensions: [category, day]
      measures: [revenue, transactions]
```

## Materialization types

| Type | What it stores | When to use |
|------|----------------|-------------|
| `unaggregated` | The full source + joins + filter as a Delta table. No aggregation. | Source views are expensive (many joins) but query patterns are unpredictable. |
| `aggregated` | Pre-computed values for the named dimensions × measures. | Predictable hot paths — daily-region revenue, monthly category counts, etc. |

You can declare multiple materialized views in one block, mixing types.

## Schedule syntax

Same syntax as standard materialized-view `REFRESH SCHEDULE`:

```yaml
schedule: every 1 hour
schedule: every 6 hours
schedule: every 1 day
schedule: cron '0 0 * * * ?'         # quartz-style cron
```

## Constraints

- Serverless compute must be enabled in the workspace.
- Databricks Runtime 17.2+ required for both the metric view and consumers.
- `TRIGGER ON UPDATE` is **not** supported.
- Mode is `relaxed` only — strict semantics are not yet available.

## Refresh

The materialized view is backed by a Lakeflow pipeline. Refresh on demand:

```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
w.pipelines.start_update(pipeline_id='your-pipeline-id')
```

The pipeline ID is visible in the metric view's metadata via `DESCRIBE TABLE EXTENDED <mv> AS JSON`.

## Full example

```yaml
version: 1.1
source: catalog.schema.fact_sales
filter: sale_date >= '2023-01-01'

joins:
  - name: customer
    source: catalog.schema.dim_customer
    'on': source.customer_id = customer.id
  - name: product
    source: catalog.schema.dim_product
    'on': source.product_id = product.id

dimensions:
  - name: sale_month
    expr: DATE_TRUNC('MONTH', sale_date)
  - name: customer_segment
    expr: customer.segment
  - name: product_category
    expr: product.category

measures:
  - name: total_revenue
    expr: SUM(amount)
  - name: transaction_count
    expr: COUNT(1)
  - name: unique_customers
    expr: COUNT(DISTINCT customer_id)

materialization:
  schedule: every 1 hour
  mode: relaxed
  materialized_views:
    - name: hourly_segment
      type: aggregated
      dimensions: [sale_month, customer_segment]
      measures: [total_revenue, transaction_count]
    - name: full_model
      type: unaggregated
```
