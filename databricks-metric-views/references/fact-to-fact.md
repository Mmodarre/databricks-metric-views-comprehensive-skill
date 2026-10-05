# Fact-to-Fact Joins

When two tables are both facts at different grains — header/line, or two independent facts conformed by date — naive star-schema modeling produces wrong totals. There are three patterns; pick by what your two facts share and what grain the queries want.

## Pattern selection

| Both facts share row-level key (`invoice_id`, `claim_id`)? | Query grain | Pattern |
|---|---|---|
| Yes | Child grain (line attributes dominate) | **A — anchor on fine grain** |
| Yes | Parent grain, or multiple children of one parent | **B — anchor on parent, child as `cardinality: one_to_many`** |
| No (only conformed by date / channel / period) | Shared dimension | **C — anchor on shared dim, all facts as `cardinality: one_to_many`** |

The mechanic that makes B and C work: the join's `cardinality:` field tells the planner that this join expands rows from the source side. The planner stores it as a per-join table property and scopes per-measure aggregation to the right grain — header-grain measures dedupe over parent keys; child-grain measures sum normally over the join's grain. Without `cardinality: one_to_many`, the engine assumes N:1 from the source side and silently first-match-truncates a 1:N join.

## Pattern A — Anchor on the fine grain

Use when the two facts share a row-level key AND the natural query grain is the child (line attributes are dimensions, header attributes reachable via N:1 lookup, every line-level dimension matters).

`source:` is the child fact. The header is N:1 from the line side — many lines per invoice, exactly the contract joins satisfy by default.

```yaml
version: 1.1
comment: 'Invoice analytics anchored at line grain'
source: sales.gold.invoice_line

joins:
  - name: invoice
    source: sales.gold.invoice
    'on': source.invoice_id = invoice.invoice_id
  - name: customer                      # snowflake hop through invoice
    source: sales.gold.customer
    'on': invoice.customer_id = customer.customer_id

dimensions:
  - { name: invoice_date,     expr: invoice.invoice_date }
  - { name: invoice_status,   expr: invoice.status }
  - { name: customer_segment, expr: customer.segment }
  - { name: product_id,       expr: source.product_id }

measures:
  # Line-grain — natural, no special handling
  - name: line_count
    expr: COUNT(1)
  - name: line_revenue
    expr: SUM(source.line_amount)

  # Header-grain — must dedupe by invoice_id
  - name: invoice_count
    expr: COUNT(DISTINCT source.invoice_id)
  - name: avg_invoice_value
    expr: SUM(source.line_amount) / NULLIF(COUNT(DISTINCT source.invoice_id), 0)

  # Header-stored value not decomposed onto lines (shipping fee, freight, header discount)
  - name: total_shipping_fee
    expr: SUM(invoice.shipping_fee) FILTER (WHERE source.line_seq = 1)
```

### Header-grain measure techniques

When a measure is naturally header-grain and the anchor is line, three deduplication options:

- **Sum a line-grain quantity** — if `invoice_total = SUM(line_amount)` by construction, just sum at line grain. Fastest and simplest.
- **`COUNT(DISTINCT)` on the header key** — for invoice_count, distinct_customers_invoiced.
- **Header-stored values** — for fields like shipping fee that aren't decomposable onto lines, use either `SUM(...) FILTER (WHERE source.line_seq = 1)` (cleanest if a first-line flag exists) or window-based dedup: `SUM(invoice.shipping_fee) / NULLIF(COUNT(*) OVER (PARTITION BY source.invoice_id), 0)`. The window runs per-row before SUM, so this is not a nested aggregate. See [gotchas.md](gotchas.md#nested-aggregates).

## Pattern B — Anchor on the parent fact

Use when the two facts share a row-level key AND header semantics dominate the query (claim_status, claim_type, invoice header dimensions) — especially when there are multiple children sharing the parent key (lines + medical_service, or lines + shipments).

`source:` is the header fact. Each child fact is a `cardinality: one_to_many` join. Header-grain measures (`SUM(source.x)`, `COUNT(source.id)`) deduplicate per parent automatically. Child-grain measures (`SUM(child.x)`, `COUNT(child.id)`) sum normally over each join's grain. Multiple children compose cleanly — neither child's row count multiplies the other.

```yaml
version: 1.1
comment: 'Invoice analytics anchored at header grain'
source: sales.gold.invoice
filter: source.customer_id IS NOT NULL

joins:
  - name: lines
    source: sales.gold.invoice_line
    'on': source.invoice_id = lines.invoice_id
    cardinality: one_to_many

  - name: shipments
    source: sales.gold.shipment
    'on': source.invoice_id = shipments.invoice_id
    cardinality: one_to_many

dimensions:
  - { name: invoice_status, expr: source.status }
  - { name: invoice_type,   expr: source.invoice_type }

measures:
  # Header-grain — engine deduplicates these per parent key
  - name: invoice_count
    expr: COUNT(DISTINCT source.invoice_id)
  - name: invoice_total
    expr: SUM(COALESCE(source.invoice_total, 0))

  # Line-grain — sums over lines join's grain, not the cross-product
  - name: line_count
    expr: COUNT(lines.line_id)
  - name: line_revenue
    expr: SUM(COALESCE(lines.line_amount, 0))

  # Shipment-grain — sums over shipments' grain, independent of lines
  - name: shipment_count
    expr: COUNT(shipments.shipment_id)
  - name: total_freight
    expr: SUM(COALESCE(shipments.freight_amount, 0))
```

A single header row with 7 lines and 3 shipments contributes once to `invoice_total`, 7 times to `line_revenue` (over `lines`' grain), and 3 times to `total_freight` (over `shipments`' grain). The 7×3 cross-product is never materialised.

## Pattern C — Anchor on a shared dimension

Use when two or more facts share NO row-level key — only conformed by date, channel, period, or some other shared key. The classic "premium ⨯ marketing-spend at month grain" or "claim-benefit ⨯ payment at FY grain" case.

`source:` is the conforming dimension (calendar / channel-month / etc.). Each fact is a `cardinality: one_to_many` join keyed by the shared column. Each fact aggregates at its own grain; composed `MEASURE()` ratios across the two atomic measures behave correctly at any rollup (per-month, per-FY, all-time) because each underlying measure is already grain-aware.

```yaml
version: 1.1
comment: 'Per-month payments vs paid-claim benefits'
source: sales.gold.dim_date
filter: source.day_of_month = 1                  # one row per month

joins:
  - name: payments
    source: billing.gold.fact_payment
    'on': |
      payments.payment_date IS NOT NULL
      AND payments.is_successful = true
      AND date_trunc('MONTH', payments.payment_date) = source.date
    cardinality: one_to_many

  - name: paid_claims
    source: claims.gold.fact_claim
    'on': |
      paid_claims.current_claim_status = 'Paid'
      AND paid_claims.paid_ts IS NOT NULL
      AND date_trunc('MONTH', CAST(paid_claims.paid_ts AS DATE)) = source.date
    cardinality: one_to_many

dimensions:
  - { name: month_start,     expr: source.date }
  - { name: financial_year,  expr: source.financial_year_label }

measures:
  - { name: payment_count,         expr: COUNT(payments.payment_id) }
  - { name: total_paid_amount,     expr: SUM(COALESCE(payments.payment_amount, 0)) }
  - { name: paid_claim_count,      expr: COUNT(paid_claims.claim_id) }
  - { name: total_claim_benefit,   expr: SUM(COALESCE(paid_claims.total_benefit, 0)) }

  - name: claim_benefit_to_payment_ratio
    expr: MEASURE(total_claim_benefit) / NULLIF(MEASURE(total_paid_amount), 0)
    format: { type: number, decimal_places: { type: exact, places: 4 } }
```

This pattern replaces the inline-SQL pre-aggregation workaround that earlier versions of this skill recommended. No more hand-written `WITH s AS (...), m AS (...) SELECT ... FROM s FULL OUTER JOIN m USING (month, channel)` source. The native `cardinality: one_to_many` planner does per-measure subqueries scoped to each join's grain — it does not materialise the cross-product. A worst-case month with 4k payments × 3k claims (~12M cross-product rows) returns instantly because no cross-product is ever built.

## Pattern D — `rely.at_most_one_match` (perf hint, not a correctness gate)

For plain N:1 joins (Pattern A's lookup case), the planner adds defensive deduplication logic to handle potential M:N data. When you've empirically verified the join is genuinely many-to-one, this hint asserts the planner can skip that logic.

```yaml
joins:
  - name: invoice
    source: sales.gold.invoice
    'on': source.invoice_id = invoice.invoice_id
    rely:
      at_most_one_match: true
```

> Behaviour described per documentation; empirical verification of the perf delta on this workspace is pending. Validate cardinality before declaring `rely` (see "Cardinality validation" below) — if the data ever violates the contract, you get silent first-match truncation with no recourse.

This hint is a side note for Pattern A. It does not enable fact-to-fact joins.

## Cardinality validation before you ship

Validate empirically — the failure modes differ between patterns:

```sql
-- Pattern A / B parent-side: the "header" key must be unique
SELECT invoice_id, COUNT(*) FROM sales.gold.invoice
GROUP BY invoice_id HAVING COUNT(*) > 1;
-- Must return zero rows.

-- Pattern A: orphan child rows (LEFT OUTER keeps them with NULL header attrs)
SELECT COUNT(*) FROM sales.gold.invoice_line l
LEFT JOIN sales.gold.invoice i ON l.invoice_id = i.invoice_id
WHERE i.invoice_id IS NULL;
-- Usually a data-quality bug. Fix at source rather than papering over.

-- Pattern B / C: confirm child-side keys repeat per parent (1:N as expected)
SELECT MAX(c) FROM (
  SELECT invoice_id, COUNT(*) c FROM sales.gold.invoice_line
  GROUP BY invoice_id
);
-- Should be > 1 if the data really is 1:N.
```

What `cardinality: one_to_many` does NOT fix:

- **Genuine M:N from the join side.** The directive declares "1:N from source to join" — i.e., one parent maps to many children. If your data is actually M:N (multiple parents share the same children), declaring `one_to_many` does not satisfy any cardinality contract and the engine has nothing safe to do. Restructure the data instead.
- **Forgetting the directive on 1:N data.** Without `cardinality: one_to_many`, the engine assumes N:1 from the source side. A 1:N join silently first-match-truncates — no error, just wrong totals.

If a cardinality check fails, fix upstream data rather than adding hints to suppress the engine's defensive logic. Silent first-match truncation is correctness debt that compounds.
