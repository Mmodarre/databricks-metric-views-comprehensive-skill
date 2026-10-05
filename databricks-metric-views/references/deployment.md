# Deployment

How to ship metric views to production: catalog/schema layout, grants, naming, and CI/CD via Databricks Asset Bundles (DABs).

## Catalog and schema layout

Put metric views in a **dedicated schema**, separate from the gold-layer tables they read from. This makes access control straightforward — data engineers own the gold layer, metric views provide a governed read-only interface for dashboard builders and Genie Spaces.

```sql
-- One-time setup
CREATE SCHEMA IF NOT EXISTS reporting.metrics
COMMENT 'Governed metric views for consistent KPI definitions across teams';

-- Read-only access for analytics consumers
GRANT USE CATALOG ON CATALOG reporting TO `data-consumers`;
GRANT USE SCHEMA ON SCHEMA reporting.metrics TO `data-consumers`;
GRANT SELECT ON SCHEMA reporting.metrics TO `data-consumers`;
```

The gold layer stays in `gold.<domain>.*` (or wherever your team puts it). Metric views live in `reporting.metrics.*` and `source:` from the gold tables. Authors who can deploy metric views need `CREATE TABLE` + `USE SCHEMA` on `reporting.metrics`; consumers need only `SELECT`.

## Naming conventions

| Object | Convention | Example |
|---|---|---|
| Metric view name | `snake_case`, domain-prefixed | `trade_orders_metrics`, `treasury_settlement_metrics` |
| Dimensions | `snake_case`, descriptive | `trade_date`, `partner_name`, `account_type` |
| Measures | `snake_case`, describes the aggregation | `total_notional_value`, `fill_rate`, `unique_accounts` |
| `display_name` | Title Case, business-friendly | `'Total Notional Value'`, `'Fill Rate'` |
| Synonyms | Lowercase, how each team naturally asks | `'gross notional'`, `'cash position'`, `'fill rate'` |

Domain-prefixing the metric view name (`trading_*`, `treasury_*`, `support_*`) prevents collisions when multiple teams have a `total_revenue` and makes governance audits easier.

## CI/CD with Databricks Asset Bundles

DABs do not yet have a dedicated `metric_view` resource type. The stable production pattern is a DABs-managed job whose tasks each run `CREATE OR REPLACE VIEW` SQL files via a SQL warehouse.

### Project structure

```
metric-views/
├── databricks.yml                       # bundle config
├── resources/
│   └── deploy_metric_views_job.yml      # job definition with SQL tasks
└── src/
    └── sql/
        ├── 00_setup_schema.sql           # schema + grants (idempotent)
        ├── 01_trade_orders_metrics.sql   # one file per metric view
        ├── 02_trade_partner_metrics.sql
        └── 03_treasury_metrics.sql
```

### `databricks.yml`

```yaml
bundle:
  name: metric-views

variables:
  catalog:
    description: 'Target catalog for metric views'
  schema:
    description: 'Target schema'
    default: metrics
  gold_catalog:
    description: 'Source catalog with gold tables'
  warehouse_id:
    description: 'SQL warehouse executing the DDL'

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://YOUR_DEV_WORKSPACE.cloud.databricks.com
    variables:
      catalog: dev_reporting
      gold_catalog: dev_gold

  staging:
    workspace:
      host: https://YOUR_STAGING_WORKSPACE.cloud.databricks.com
    variables:
      catalog: staging_reporting
      gold_catalog: staging_gold

  prod:
    workspace:
      host: https://YOUR_PROD_WORKSPACE.cloud.databricks.com
    variables:
      catalog: reporting
      gold_catalog: gold
```

### `resources/deploy_metric_views_job.yml`

```yaml
resources:
  jobs:
    deploy_metric_views:
      name: 'deploy-metric-views-${bundle.target}'
      description: 'Deploy metric view definitions to ${var.catalog}.${var.schema}'
      tasks:
        - task_key: setup_schema
          sql_task:
            warehouse_id: ${var.warehouse_id}
            file: { path: src/sql/00_setup_schema.sql }

        - task_key: trade_orders_metrics
          depends_on: [{ task_key: setup_schema }]
          sql_task:
            warehouse_id: ${var.warehouse_id}
            file: { path: src/sql/01_trade_orders_metrics.sql }

        - task_key: trade_partner_metrics
          depends_on: [{ task_key: setup_schema }]
          sql_task:
            warehouse_id: ${var.warehouse_id}
            file: { path: src/sql/02_trade_partner_metrics.sql }

        - task_key: treasury_metrics
          depends_on: [{ task_key: setup_schema }]
          sql_task:
            warehouse_id: ${var.warehouse_id}
            file: { path: src/sql/03_treasury_metrics.sql }
```

### Parameterized SQL files

Each `.sql` file uses bundle variables so the same definition deploys to dev/staging/prod:

```sql
-- src/sql/01_trade_orders_metrics.sql
CREATE OR REPLACE VIEW ${var.catalog}.${var.schema}.trade_orders_metrics
WITH METRICS
LANGUAGE YAML
AS $$
version: 1.1
comment: 'Trade order KPIs for operations and compliance reporting'
source: ${var.gold_catalog}.trading.fact_trade_orders
filter: trade_date >= '2024-01-01'

dimensions:
  - name: trade_date
    expr: trade_date
    display_name: 'Trade Date'
    format: { type: date, date_format: year_month_day }

  - name: order_status
    expr: |
      CASE
        WHEN status = 'FILLED' THEN 'Filled'
        WHEN status = 'PARTIAL' THEN 'Partially Filled'
        WHEN status = 'CANCELLED' THEN 'Cancelled'
      END
    display_name: 'Order Status'

measures:
  - name: total_orders
    expr: COUNT(1)
    display_name: 'Total Orders'

  - name: filled_orders
    expr: COUNT(1) FILTER (WHERE status = 'FILLED')

  - name: fill_rate
    expr: MEASURE(filled_orders) / NULLIF(MEASURE(total_orders), 0)
    display_name: 'Fill Rate'
    format:
      type: percentage
      decimal_places: { type: exact, places: 2 }
$$;
```

### Deploy and run

```bash
databricks bundle validate -t dev
databricks bundle deploy -t dev
databricks bundle run deploy_metric_views -t dev
```

Promote to staging/prod by changing the target.

## Performance hygiene

- **Filter early.** Use the metric view's `filter:` to scope to recent data. Querying a 10-year history when consumers only ever look at the last 90 days wastes cycles.
- **Materialize hot paths.** Dashboard queries that hit the same dim/measure combos repeatedly benefit from `materialization:` — see [materialization.md](materialization.md).
- **Validate cardinality before joining** — silent first-match truncation is a correctness debt, not a performance issue. See [fact-to-fact.md](fact-to-fact.md#cardinality-validation-before-you-ship).
