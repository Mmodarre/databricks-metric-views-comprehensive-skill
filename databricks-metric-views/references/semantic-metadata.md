# Semantic Metadata

Semantic metadata makes metric views work well with Genie's natural-language interface and Lakeview's auto-formatting. Requires `version: 1.1` and DBR 17.2+.

## Field summary

| Field | Where used | Purpose |
|-------|------------|---------|
| `display_name` | dimensions, measures | Title-Case label shown in Genie, Lakeview, Unity Catalog browser. Max 255 chars. |
| `synonyms` | dimensions, measures | Alternative names Genie matches against natural-language questions. Max 10 per field, 255 chars each. |
| `format` | measures, date dimensions | Renders values as currency, percentage, number, or date. |
| `comment` | metric view, dimensions, measures | Documentation. Surfaced in UC browser; persists across ALTER VIEW. |

## Display names and synonyms

```yaml
dimensions:
  - name: order_date
    expr: o_orderdate
    display_name: 'Order Date'
    synonyms:
      - 'order time'
      - 'date of order'

measures:
  - name: total_revenue
    expr: SUM(o_totalprice)
    display_name: 'Total Revenue'
    synonyms: ['revenue', 'total sales', 'gross revenue']     # flow style is fine
```

Synonyms should match how each team naturally asks questions. Trading ops calls it "gross notional"; treasury calls it "cash position"; finance calls it "revenue." Add 2–4 per measure.

### Cross-team synonym examples

The same measure often has different names across teams. Maintain one canonical `name:` and `display_name:`, and use `synonyms:` to cover each team's vocabulary.

| Measure shape       | Trading Ops                            | Treasury                                  | Support                          |
|---------------------|----------------------------------------|-------------------------------------------|----------------------------------|
| Total value         | "gross notional", "total value"        | "cash position", "settlement amount"      | —                                |
| Count               | "order count", "number of trades"      | "settlement count"                        | "ticket count", "case volume"    |
| Rate / success      | "fill rate", "execution rate"          | "success rate", "completion rate"         | "resolution rate", "SLA rate"    |

Genie matches synonyms by exact string after lowercasing — list the phrasings each team naturally uses, not dictionary synonyms.

## Format types

Four format types: `currency`, `percentage`, `number`, `date`.

### Currency

```yaml
format:
  type: currency
  currency_code: USD               # ISO 4217 — required
  decimal_places:                  # optional
    type: exact
    places: 2
  hide_group_separator: false      # optional — show 1,234,567 not 1234567
```

### Percentage

```yaml
format:
  type: percentage
  decimal_places:
    type: exact
    places: 1
```

The metric view multiplies the value by 100 before display — store rates as decimals (`0.95`, not `95`).

### Number

```yaml
format:
  type: number
  decimal_places:
    type: exact
    places: 0
  hide_group_separator: false
```

### Date

```yaml
format:
  type: date
  date_format: year_month_day      # also: month_day_year, day_month_year, etc.
  leading_zeros: true              # optional — 01 vs 1
```

## `decimal_places` is a nested object

This is the single most common format error. **Scalar forms fail at create time** with a `DecimalPlaces` deserialization error.

```yaml
# WRONG
decimal_places: 2
decimal_places: "2"

# RIGHT
decimal_places:
  type: exact         # or 'max' or 'all'
  places: 2           # required for exact/max; omit for all
```

| `type` | Behavior | `places` |
|--------|----------|----------|
| `exact` | Always renders exactly N decimals (pads with zeros) | required, 0–10 |
| `max` | Renders up to N decimals (trims trailing zeros) | required, 0–10 |
| `all` | Renders all available decimals | omit |

## `hide_group_separator`, not `use_thousands_separator`

The field is **`hide_group_separator`** (inverted semantics: `false` to show separators). The intuitive name `use_thousands_separator` will fail with an unrecognized-field error.

```yaml
# WRONG
use_thousands_separator: true

# RIGHT (show 1,234,567)
hide_group_separator: false
```

Valid `number`/`currency` format fields: `decimal_places`, `hide_group_separator`, `currency_code` (currency only), `abbreviation`.

## Full example

```yaml
version: 1.1
comment: 'Sales KPIs with full semantic metadata'
source: catalog.schema.orders

dimensions:
  - name: order_date
    expr: o_orderdate
    display_name: 'Order Date'
    format:
      type: date
      date_format: year_month_day
    synonyms: ['order time', 'date of order']

  - name: customer_segment
    expr: |
      CASE
        WHEN o_totalprice > 100000 THEN 'Enterprise'
        WHEN o_totalprice > 10000 THEN 'Mid-market'
        ELSE 'SMB'
      END
    display_name: 'Customer Segment'
    synonyms: ['customer tier', 'customer type']

measures:
  - name: total_revenue
    expr: SUM(o_totalprice)
    display_name: 'Total Revenue'
    comment: 'Sum of all order values'
    format:
      type: currency
      currency_code: USD
      decimal_places: { type: exact, places: 2 }
    synonyms: ['revenue', 'total sales', 'gross revenue']

  - name: order_count
    expr: COUNT(1)
    display_name: 'Order Count'
    format:
      type: number
      hide_group_separator: false
    synonyms: ['number of orders', 'total orders']

  - name: fulfillment_rate
    expr: MEASURE(fulfilled_orders) / NULLIF(MEASURE(order_count), 0)
    display_name: 'Fulfillment Rate'
    format:
      type: percentage
      decimal_places: { type: exact, places: 1 }
```
