# Module 8 — Semantic Layer & MetricFlow
**Time: 50 minutes | Priority: Advanced**

## Learn (25 min)

### The problem: metric definitions scattered and inconsistent

Without a semantic layer, "revenue" gets redefined slightly differently in every BI dashboard, every analyst's ad hoc query, and every downstream tool — a classic "why do Tableau and our exec deck disagree by 3%" incident. The dbt **Semantic Layer**, powered by **MetricFlow**, lets you define metrics **once**, in version-controlled YAML sitting on top of your existing dbt models, and query them consistently from SQL, the GraphQL API, or BI tool integrations (Tableau, Power BI, Sheets), instead of every consumer re-deriving the aggregation logic.

### Core concepts

- **Semantic models** — metadata layered onto an existing dbt model (not a new physical table), declaring:
  - **Entities** — join keys (`primary`, `foreign`) that tell MetricFlow how semantic models relate to each other.
  - **Dimensions** — how you slice/group (categorical, or time with a `granularity`).
  - **Measures** — raw aggregatable columns (e.g. `sum(order_total)`) that metrics are built from.
- **Metrics** — the actual named, queryable business definitions, five types:
  - **Simple** — direct aggregation of one measure.
  - **Ratio** — one metric divided by another (e.g. `food_revenue_per_customer`).
  - **Derived** — an expression combining other metrics.
  - **Cumulative** — aggregated over a rolling/cumulative time window.
  - **Conversion** — tracks a base event and a subsequent conversion event for the same entity within a time window (funnel-style metrics).

```yaml
models:
  - name: orders
    semantic_model:
      enabled: true
      name: orders_semantic_model
    agg_time_dimension: ordered_at
    columns:
      - name: order_id
        entity: {type: primary, name: order_id}
      - name: customer_id
        entity: {type: foreign, name: customer}
      - name: ordered_at
        granularity: day
        dimension: {type: time}
    metrics:
      - name: order_total
        type: simple
        agg: sum
        expr: order_total
```

This is the **new Semantic Layer YAML spec** (live on the Latest release track through 2026) which lets you define `semantic_model:`/`metrics:` directly inside a model's own `.yml` properties, rather than in fully separate semantic-model files — worth mentioning that this simplified the authoring ergonomics versus the earlier (2023–2024) syntax.

### How it's queried

- **MetricFlow CLI** — `mf query --metrics order_total --group-by metric_time__day` for local development/validation.
- **GraphQL API** — how BI tools and the `queryRecords`/paginated metadata endpoints actually fetch data in production.
- **Saved queries** — a named, reusable, pre-defined combination of metrics + dimensions + filters, so common questions ("weekly revenue by region") don't need to be re-specified by every consumer:
  ```yaml
  saved_queries:
    - name: weekly_revenue_by_region
      query_params:
        metrics: [order_total]
        group_by: [metric_time__week, customer__region]
  ```
- 2025's Semantic Layer expansion added: Postgres and Trino platform support, Power BI and Tableau Cloud integrations, and querying the Semantic Layer with natural language / without writing code via dbt Insights.

### Where it fits vs. plain models/metrics-as-SQL

If you find yourself writing the same aggregation with slightly different `GROUP BY`s across five marts, that's a signal the logic belongs in the Semantic Layer as a metric, not duplicated as SQL in each mart — the same DRY instinct as macros (Module 5), but for *business* logic rather than *transformation* logic. The tradeoff: it requires BI tools/consumers to actually query through the Semantic Layer (GraphQL/MetricFlow) rather than querying a plain table directly, which is an adoption cost, and MetricFlow doesn't currently support arbitrary dbt macros/packages inside metric definitions — it's intentionally a constrained DSL, not general SQL.

## Practice (15 min)

1. Define a semantic model on top of a hypothetical `fct_orders(order_id, customer_id, order_total, ordered_at, is_refunded)` with `order_id` as primary entity, `customer_id` as foreign entity, `ordered_at` as the time dimension (day granularity), and `is_refunded` as a categorical dimension.
2. Define two metrics: `gross_revenue` (simple, sum of `order_total`), and `net_revenue_rate` (ratio: revenue where `is_refunded = false` divided by `gross_revenue`).
3. Write a `saved_queries` entry for "monthly net revenue rate by customer region" (assume a `region` dimension exists via a joined entity).

## Recall (6 min flashcards)

- Semantic Layer = metrics defined once in YAML on top of existing models, queried consistently everywhere (GraphQL, BI tools, MetricFlow CLI).
- Semantic model = entities (join keys) + dimensions (slice by) + measures (raw aggregatable columns), attached to an existing dbt model, not a new table.
- 5 metric types: simple, ratio, derived, cumulative, conversion.
- New (2026) Semantic Layer YAML spec: `semantic_model:`/`metrics:` inline in a model's own properties file.
- Saved queries = named, reusable metric+dimension+filter combos.
- MetricFlow doesn't support arbitrary macros/packages — constrained DSL by design.
- Compatible with dbt v1.6+; platform support expanded through 2025 (Postgres, Trino added).

## Interview (4 min)

**Q: "Marketing and Finance report different 'active customers' numbers off the same warehouse. How does the Semantic Layer prevent this?"**
Define `active_customers` once as a metric (with its exact filter/window logic) in the semantic layer; both teams query the *same* metric definition through their BI tool's Semantic Layer integration instead of each writing their own `WHERE`/`GROUP BY`, so the definition can only drift if someone deliberately edits the governed YAML (which is then visible in git history and code review) rather than silently in fifty different dashboards.

**Q: "When would you NOT put a metric in the Semantic Layer?"**
One-off, exploratory, or single-consumer analysis that doesn't need cross-tool consistency — forcing every ad hoc analytical question through a governed metrics layer is overhead without payoff; the Semantic Layer earns its cost when a definition needs to be consistent across *multiple* consumers/tools.

**Q: "How is a `ratio` metric different from just computing two simple metrics and dividing in your BI tool?"**
The division happens inside MetricFlow's query generation with correct handling of grouping/granularity (e.g. avoiding the classic mistake of averaging pre-aggregated ratios instead of recomputing numerator/denominator at the target grain) — doing it downstream in a BI tool risks exactly that kind of silent aggregation error, which is the underlying reliability argument for the whole Semantic Layer.
