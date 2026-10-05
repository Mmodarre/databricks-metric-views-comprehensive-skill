# Joins in Metric Views

Joins let a metric view's `source:` (the fact / anchor grain) reach dimension columns from other tables. Joins default to LEFT OUTER and many-to-one (N:1 from the source side); declare `cardinality: one_to_many` on a join entry when the data is 1:N from source to join.

## Mental model

- `source` defines the anchor grain. Every measure aggregates at this grain.
- A `joins:` entry is **N:1 from the source side by default** — many source rows match one joined row, never the reverse. Declare `cardinality: one_to_many` to flip the contract: one source row matches many joined rows. The planner then scopes per-measure aggregation to each join's grain. See [fact-to-fact.md](fact-to-fact.md) for when to use it.
- Joins are evaluated **lazily**: the engine only joins tables whose columns are referenced by the active query. If a query selects only `customer.nation.region.r_name`, the `customer` and `nation` joins are also pulled in (since the path traverses them).
- Joins are **silent on cardinality violations**. The default N:1 contract silently first-match-truncates 1:N or M:N data — no error, wrong totals. `cardinality: one_to_many` handles 1:N correctly but does not fix genuine M:N. Validate cardinality before you join — see [gotchas.md](gotchas.md#silent-first-match-truncation).

## Star schema (single-level joins)

```yaml
source: catalog.schema.fact_orders

joins:
  # ON expression
  - name: customer
    source: catalog.schema.dim_customer
    'on': source.customer_id = customer.customer_id

  # USING — when the FK column has the same name in both tables
  - name: product
    source: catalog.schema.dim_product
    using:
      - product_id

dimensions:
  - name: customer_segment
    expr: customer.segment              # join_name.column
  - name: product_category
    expr: product.category
  - name: order_month
    expr: DATE_TRUNC('MONTH', order_date)   # source columns need no prefix

measures:
  - name: total_revenue
    expr: SUM(amount)                   # source columns ok
  - name: unique_customers
    expr: COUNT(DISTINCT customer.customer_id)
```

### `'on':` MUST be quoted

YAML 1.1 parses bare `on` as the boolean `true`. The join condition is silently dropped. Always single-quote: `'on': source.fk = dim.pk`.

## Snowflake schema (nested joins, DBR 17.1+)

When a dimension table itself joins to another dimension, nest the joins. Each hop must be N:1.

```yaml
source: samples.tpch.orders

joins:
  - name: customer
    source: samples.tpch.customer
    'on': source.o_custkey = customer.c_custkey
    joins:
      - name: nation                    # dim → dim, N:1
        source: samples.tpch.nation
        'on': customer.c_nationkey = nation.n_nationkey
        joins:
          - name: region                # 3 hops deep
            source: samples.tpch.region
            'on': nation.n_regionkey = region.r_regionkey

dimensions:
  - name: customer_name
    expr: customer.c_name               # 1 hop
  - name: nation_name
    expr: customer.nation.n_name        # 2 hops — hierarchical path through parent
  - name: region_name
    expr: customer.nation.region.r_name # 3 hops — full path from top-level join
```

### Reference path is hierarchical, not flat

Nested join columns are referenced by **traversing the join hierarchy** from the top-level join down. Reference `customer.nation.n_name`, not bare `nation.n_name`. The path mirrors the nesting structure declared in `joins:`.

### Sibling joins cannot reference each other

Top-level joins are all anchored to `source`. If two dimension tables share a key but neither is a natural parent of the other, you must pick one as the top-level join and **nest** the other. There is no "side join" between sibling top-level joins.

## ON vs USING

| Clause | When to use |
|--------|-------------|
| `'on': <expression>` | Most cases. The FK column has a different name from the PK column, or the predicate is non-trivial. |
| `using: [<column1>, <column2>]` | The shared column has the same name on both sides. Equivalent to `source.col = join.col`. |

You cannot use both on the same join.

## Joined column reference rules

- **From a dimension or measure expression:** `join_name.column_name` for top-level joins; `parent.child.column_name` for nested joins — follow the hierarchical path.
- **In a join's `'on':` predicate:** the source side is referenced as `source.column`; joined sides use their `name:`.
- **Default reference** (no prefix) inside a `'on':` predicate is the join table itself, not the source. Best practice: always prefix explicitly to avoid confusion.

## Constraints and limitations

- All joins are **LEFT OUTER**. Source rows with no match get `NULL` for joined columns; source rows are never dropped.
- Joins are **many-to-one from the source side by default**. Set `cardinality: one_to_many` on a join entry when the data is 1:N from source to join (header→child, or shared-dim→fact). See [fact-to-fact.md](fact-to-fact.md). Genuine M:N data is not supported by either default — restructure upstream.
- **MAP-type columns are not allowed in joined tables.** They will fail at create time.
- **No query-time joins** — every join must be in the YAML. To combine two metric views at query time, wrap each in a CTE and join the CTE results in a downstream SQL query.
- **JOIN with inline-SQL `source:` doesn't work.** If `source` is a query, top-level `joins:` are ignored. Push the join into the source query, or build an ETL view first.

## Name-shadowing trap

A dimension whose `name:` matches a join's `name:` shadows the join. Subsequent expressions referencing `<name>.column` will resolve to the dimension (a string) rather than the join, producing `INVALID_EXTRACT_BASE_FIELD_TYPE`.

```yaml
# WRONG
joins:
  - name: channel
    source: catalog.schema.dim_channel
    'on': source.channel_id = channel.id
dimensions:
  - name: channel                       # shadows the join alias
    expr: channel.pychannel             # resolves to dimension, not join → ERROR

# RIGHT — use a different dimension name; surface the natural label via display_name + synonyms
dimensions:
  - name: channel_name
    expr: channel.pychannel
    display_name: 'Channel'
    synonyms: ['channel']
```
