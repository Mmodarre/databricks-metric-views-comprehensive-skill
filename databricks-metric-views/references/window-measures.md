# Window Measures

Window measures enable moving averages, running totals, period-over-period changes, and semi-additive aggregations. They are queried with the same `MEASURE()` syntax as ordinary measures.

> **Status**: Experimental. **Use `version: 1.1`** — confirmed by Databricks docs as of April 2026.

## Anatomy

```yaml
measures:
  - name: <measure_name>
    expr: <AGGREGATE>(...)
    window:
      - order: <dimension_name>      # dimension that orders the window
        range: <range_value>          # see table below
        semiadditive: <first | last>  # value when order dim absent from GROUP BY
```

### Range values

| Range | Behavior |
|-------|----------|
| `current` | Only rows whose ordering value equals the current row |
| `cumulative` | All rows up to and including the current row |
| `trailing N <unit>` | N units before current row (**excludes** current) |
| `leading N <unit>` | N units after current row |
| `all` | All rows regardless of ordering — used for coarser LOD (see [level-of-detail.md](level-of-detail.md)) |

`<unit>` accepts `day`, `days`, `month`, `months`, `year`, `years`, etc.

### Semiadditive

When the `order` dimension is **not** in the query's `GROUP BY`, the engine must collapse the windowed results to a single value. `semiadditive: last` returns the last value (most recent date for date-ordered windows); `first` returns the first.

For balance-style measures (bank balance, inventory level), `last` is what you want — when summing across customers without a date GROUP BY, you get the most recent balance per customer summed across all customers, not a meaningless sum-across-time.

## Pattern: Trailing 7-day distinct customers

```yaml
version: 1.1
source: catalog.schema.orders
filter: order_date > DATE'2024-01-01'

dimensions:
  - name: date
    expr: order_date

measures:
  - name: t7d_customers
    expr: COUNT(DISTINCT customer_id)
    window:
      - order: date
        range: trailing 7 day
        semiadditive: last
```

`trailing 7 day` covers the 7 days **before** each date, **excluding** the current date. To include the current date, combine with a `current` window or compute differently in the source.

## Pattern: Cumulative running total

```yaml
measures:
  - name: running_total_sales
    expr: SUM(total_price)
    window:
      - order: date
        range: cumulative
        semiadditive: last
```

## Pattern: Period-over-period (day-over-day growth)

Compose two window measures and derive a third:

```yaml
measures:
  - name: previous_day_sales
    expr: SUM(total_price)
    window:
      - order: date
        range: trailing 1 day
        semiadditive: last

  - name: current_day_sales
    expr: SUM(total_price)
    window:
      - order: date
        range: current
        semiadditive: last

  - name: day_over_day_growth
    expr: (MEASURE(current_day_sales) - MEASURE(previous_day_sales)) / NULLIF(MEASURE(previous_day_sales), 0)
    format:
      type: percentage          # engine multiplies by 100 for display — store as raw ratio
      decimal_places: { type: exact, places: 2 }
```

The derived `day_over_day_growth` measure does **not** need its own `window:` block — it inherits windowing through the `MEASURE()` references.

## Pattern: Year-to-date (multi-window)

A single measure can have **multiple window specs** that compose into period-to-date semantics:

```yaml
dimensions:
  - name: date
    expr: order_date
  - name: year
    expr: DATE_TRUNC('year', order_date)

measures:
  - name: ytd_sales
    expr: SUM(total_price)
    window:
      - order: date
        range: cumulative
        semiadditive: last
      - order: year
        range: current
        semiadditive: last
```

The first window sums cumulatively over `date`. The second restricts scope to the `current` year. Together they produce a YTD total that resets each year.

## Pattern: Semi-additive bank balance

For measures that should not sum across time:

```yaml
dimensions:
  - name: date
    expr: balance_date
  - name: customer
    expr: customer_id

measures:
  - name: balance
    expr: SUM(balance_amount)
    window:
      - order: date
        range: current
        semiadditive: last
```

The measure **still aggregates across non-time dimensions** like `customer`. Grouped by date, you get total balance across customers for that day. Not grouped by date, you get the total of each customer's most-recent balance.

## Querying window measures

```sql
SELECT
  date,
  MEASURE(t7d_customers)         AS trailing_7d,
  MEASURE(running_total_sales)   AS running_total
FROM catalog.schema.activity_metrics
WHERE date >= DATE '2024-06-01'
GROUP BY ALL
ORDER BY date
```

No special syntax — the engine knows the measure has a `window:` and applies it.
