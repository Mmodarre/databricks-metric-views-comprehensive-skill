# Worked Examples

End-to-end YAML examples covering the common shapes. Copy and adapt.

## 1. Simple KPI view (single table, atomic + composed measures)

```yaml
version: 1.1
comment: 'Orders KPIs for sales analysis'
source: samples.tpch.orders
filter: o_orderdate > '1990-01-01'

dimensions:
  - name: order_month
    expr: DATE_TRUNC('MONTH', o_orderdate)
    display_name: 'Order Month'

  - name: order_status
    expr: |
      CASE
        WHEN o_orderstatus = 'O' THEN 'Open'
        WHEN o_orderstatus = 'P' THEN 'Processing'
        WHEN o_orderstatus = 'F' THEN 'Fulfilled'
      END
    display_name: 'Order Status'

  - name: order_priority
    expr: SPLIT(o_orderpriority, '-')[1]
    display_name: 'Order Priority'

measures:
  # Atomic
  - name: order_count
    expr: COUNT(1)
    display_name: 'Order Count'
    format: { type: number, hide_group_separator: false }

  - name: total_revenue
    expr: SUM(o_totalprice)
    display_name: 'Total Revenue'
    format:
      type: currency
      currency_code: USD
      decimal_places: { type: exact, places: 2 }

  - name: revenue_open
    expr: SUM(o_totalprice) FILTER (WHERE o_orderstatus = 'O')
    display_name: 'Revenue (Open Orders)'

  # Composed
  - name: revenue_per_customer
    expr: MEASURE(total_revenue) / NULLIF(COUNT(DISTINCT o_custkey), 0)
    display_name: 'Revenue per Customer'
    format:
      type: currency
      currency_code: USD
```

DDL wrapper:

```sql
CREATE OR REPLACE VIEW catalog.schema.orders_metrics
WITH METRICS
LANGUAGE YAML
AS $$
-- paste YAML here
$$;
```

## 2. Star schema (orders + customer + nation)

```yaml
version: 1.1
comment: 'Revenue by geography'
source: samples.tpch.orders

joins:
  - name: customer
    source: samples.tpch.customer
    'on': source.o_custkey = customer.c_custkey
  - name: nation
    source: samples.tpch.nation
    'on': customer.c_nationkey = nation.n_nationkey

dimensions:
  - name: order_month
    expr: DATE_TRUNC('MONTH', o_orderdate)
    display_name: 'Order Month'
  - name: customer_name
    expr: customer.c_name
    display_name: 'Customer'
  - name: nation_name
    expr: nation.n_name
    display_name: 'Country'

measures:
  - name: total_revenue
    expr: SUM(o_totalprice)
    display_name: 'Total Revenue'
  - name: unique_customers
    expr: COUNT(DISTINCT customer.c_custkey)
    display_name: 'Unique Customers'
```

## 3. Snowflake schema (orders → customer → nation → region)

```yaml
version: 1.1
source: samples.tpch.orders

joins:
  - name: customer
    source: samples.tpch.customer
    'on': source.o_custkey = customer.c_custkey
    joins:
      - name: nation
        source: samples.tpch.nation
        'on': customer.c_nationkey = nation.n_nationkey
        joins:
          - name: region
            source: samples.tpch.region
            'on': nation.n_regionkey = region.r_regionkey

dimensions:
  - name: customer_name
    expr: customer.c_name
  - name: nation_name
    expr: nation.n_name                  # 2 hops, flat reference
  - name: region_name
    expr: region.r_name                  # 3 hops, flat reference

measures:
  - name: total_revenue
    expr: SUM(o_totalprice)
  - name: order_count
    expr: COUNT(1)
```

Query example (rolls up across hierarchy levels):

```sql
SELECT region_name, MEASURE(total_revenue), MEASURE(order_count)
FROM catalog.schema.geo_metrics
GROUP BY ALL
ORDER BY 2 DESC;
```

## 4. Composable measures (atomic-first, ratios on top)

```yaml
version: 1.1
source: catalog.schema.orders

measures:
  # Atomic
  - name: total_revenue
    expr: SUM(amount)
  - name: total_orders
    expr: COUNT(1)
  - name: fulfilled_orders
    expr: COUNT(1) FILTER (WHERE status = 'fulfilled')
  - name: unique_customers
    expr: COUNT(DISTINCT customer_id)

  # Composed
  - name: avg_order_value
    expr: MEASURE(total_revenue) / NULLIF(MEASURE(total_orders), 0)
    format:
      type: currency
      currency_code: USD

  - name: fulfillment_rate
    expr: MEASURE(fulfilled_orders) / NULLIF(MEASURE(total_orders), 0)
    format:
      type: percentage
      decimal_places: { type: exact, places: 1 }

  - name: revenue_per_customer
    expr: MEASURE(total_revenue) / NULLIF(MEASURE(unique_customers), 0)
    format:
      type: currency
      currency_code: USD
```

When `total_revenue` definition changes (e.g., excludes tax), `avg_order_value` and `revenue_per_customer` automatically use the updated definition. That's the point of composability.

## 5. SQL query as source (use when you need pre-aggregation or complex joins)

```yaml
version: 1.1
source: |
  SELECT o.*, c.c_name, c.c_mktsegment
  FROM samples.tpch.orders o
  LEFT JOIN samples.tpch.customer c ON o.o_custkey = c.c_custkey

dimensions:
  - name: market_segment
    expr: c_mktsegment
  - name: order_month
    expr: DATE_TRUNC('MONTH', o_orderdate)

measures:
  - name: total_revenue
    expr: SUM(o_totalprice)
  - name: order_count
    expr: COUNT(1)
```

> **Reminder**: Top-level `joins:` are ignored when `source:` is an inline query. If you need both pre-aggregation and YAML joins, build an ETL view first.

## 6. Metric view as source (chained MVs and the SSOT pattern)

When multiple metric views share atomic-grain measures over the same source table, extract those into a base MV and have the consumers chain it via `source:`.

```yaml
# Base — at catalog.schema.claim_base
version: 1.1
source: catalog.schema.fact_claim
filter: source.policy_id <> '00000000-0000-0000-0000-000000000000'
dimensions:
  - { name: claim_type, expr: source.claim_type }
  - { name: claim_status, expr: source.current_claim_status }
  - { name: lodgement_date, expr: source.lodgement_date }
measures:
  - { name: claim_count, expr: COUNT(source.claim_id) }
  - { name: paid_claim_count, expr: COUNT_IF(source.current_claim_status = 'Paid') }
  - { name: total_charge_amount, expr: CAST(SUM(source.total_charge) AS DOUBLE) }
```

```yaml
# Consumer — adds joins + windowed measures, inherits atomic measures via composition
version: 1.1
source: catalog.schema.claim_base   # <-- chained MV-as-source

joins:
  - name: date_lodged
    source: catalog.schema.dim_date
    'on': date_lodged.date = `lodgement_date`   # base dimension by backticked name

dimensions:
  - { name: lodged_month, expr: date_lodged.calendar_ym }

measures:
  - name: paid_completion_rate
    expr: MEASURE(paid_claim_count) / NULLIF(MEASURE(claim_count), 0)
    # MEASURE() calls reference the BASE's measures from inside a local definition.
```

### The SSOT boundary rule

A base MV should contain **only**:

- **Atomic measures** — single aggregation per measure (`SUM`, `COUNT`, `COUNT_IF`).
- **Source-bound expressions** — no `MEASURE()` over inherited measures (the base has no parent to inherit from).
- **No joins** — joins are grain-specific and belong in consumers.
- **No window measures** — windows are grain-specific too.
- **No materialization** — same reason.

Joined dimensions, windowed measures, atomic-ratio composition, and materialization all live in the consumer that needs them.

### What inherits and what doesn't

| Surface | Visible on consumer? | How to use |
|---|---|---|
| Base **dimensions** | Yes, by backticked name | `` `lodgement_date` `` in joins / filters / local dimensions |
| Base **measures** | **No, not directly callable** | Reachable only inside a *local* measure's `expr:` via `MEASURE(<base_name>)` |
| Base **physical source columns** | As `__mv_src_<col>` aliases | Avoid — use the base's exposed dimension instead |

If you need to expose an inherited measure on the consumer's public surface, declare a thin local pass-through: `expr: MEASURE(<inherited_name>)`. See [gotchas.md → Inherited measures are not callable from a chained consumer](gotchas.md#inherited-measures-are-not-callable-from-a-chained-consumer).

### Naming discipline

Per [gotchas.md → Measure name collides with source-table column name](gotchas.md#measure-name-collides-with-source-table-column-name) — measure names in a base MV must not match physical columns of its source. Follow the `total_<thing>_amount` / `<thing>_count` pattern. Once a base is chained, any collision will break consumers with a `MISSING_ATTRIBUTES` error.

### Deployment ordering

The live consumer keeps binding to the *deployed* base, not your local YAML. When fixing a chain bug, redeploy the base BEFORE re-querying any consumer. A consumer-only redeploy is a no-op for inheritance purposes.

Keep composition to 2–3 layers — deeper nesting becomes hard to debug.

## 7. MCP tool authoring

```python
manage_metric_views(
    action="create",
    full_name="catalog.schema.orders_metrics",
    source="catalog.schema.orders",
    or_replace=True,
    comment="Orders KPIs for sales analysis",
    filter_expr="order_date > '2020-01-01'",
    dimensions=[
        {"name": "order_month", "expr": "DATE_TRUNC('MONTH', order_date)",
         "display_name": "Order Month"},
        {"name": "order_status", "expr": "status"},
    ],
    measures=[
        {"name": "order_count", "expr": "COUNT(1)"},
        {"name": "total_revenue", "expr": "SUM(total_price)",
         "display_name": "Total Revenue"},
    ],
)

manage_metric_views(
    action="query",
    full_name="catalog.schema.orders_metrics",
    query_measures=["total_revenue", "order_count"],
    query_dimensions=["order_month"],
    where="EXTRACT(YEAR FROM order_month) = 2024",
    order_by="ALL",
    limit=100,
)
```

Available actions: `create`, `alter`, `describe`, `query`, `drop`, `grant`. The `alter` action takes the same shape as `create` and replaces the entire YAML.
