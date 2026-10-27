# Module 2 — Project Structure, Sources & Staging Conventions
**Time: 35 minutes | Priority: Core**

## Learn (18 min)

### The layered architecture (staging → intermediate → marts)

This is dbt Labs' own recommended structure (from their official "How we structure our dbt projects" guide) and it comes up in almost every interview as "walk me through how you'd organize a new project":

- **`sources`** — not files you write SQL in; YAML declarations of raw tables already loaded into the warehouse (`source('jaffle_shop', 'orders')`). Sources let you: (a) test raw data freshness/quality, (b) document lineage back to the true origin, (c) swap the underlying raw table without touching every downstream model.
- **`staging/`** — one model per source table, thin 1:1 renaming/casting/light cleanup layer. No joins across sources here. Naming convention: `stg_<source>__<entity>.sql` (e.g. `stg_stripe__payments.sql`). This is where you standardize column names, data types, and boolean/timestamp casting so everything downstream is consistent.
- **`intermediate/`** — optional layer for reusable joins/logic that don't belong in a single mart and aren't meant to be queried directly by BI tools. Convention: `int_<entity>s_<verb>.sql` (e.g. `int_payments_pivoted_to_orders.sql`). Often materialized as ephemeral or view.
- **`marts/`** — the business-facing, wide, denormalized final models (`dim_customers`, `fct_orders`) that BI tools and analysts actually query. Usually `table` or `incremental`.

### Sources in depth

```yaml
sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    tables:
      - name: orders
        loaded_at_field: _etl_loaded_at
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 24, period: hour}
```

`dbt source freshness` checks whether the raw data is stale — a genuinely common production interview question is "how do you know if an upstream pipeline silently broke?" and freshness checks are part of that answer (paired with anomaly/volume tests downstream).

### `dbt_project.yml`, `packages.yml`, `profiles.yml` — know which does what

- **`dbt_project.yml`** — project-level config: model paths, materialization defaults per directory (`+materialized: table` under a folder path), variables, hooks, quoting.
- **`packages.yml`** (or newer **`dependencies.yml`**, which can also pull in other dbt *projects* for cross-project `ref()` under dbt Mesh) — third-party package dependencies like `dbt_utils`, `dbt_expectations`, `codegen`.
- **`profiles.yml`** — connection credentials (warehouse host, user, warehouse/role, target). Lives outside the project repo (in `~/.dbt/` for dbt Core) — this separation is deliberate so credentials never get committed. On dbt Platform, this is replaced by environment/connection configuration in the UI. In 2026 release notes, a new **Profiles** feature (project-level connection management) also appeared in the Fusion tooling.

### Node selection syntax (you must know this cold)

```bash
dbt run --select stg_orders          # exact model
dbt run --select stg_orders+         # model + everything downstream
dbt run --select +stg_orders         # model + everything upstream
dbt run --select +stg_orders+        # both directions
dbt run --select tag:nightly
dbt run --select path:models/marts/finance
dbt run --select stg_orders,stg_customers   # union
dbt run --select stg_orders --exclude stg_payments
dbt run --select state:modified+     # everything changed vs. a stored manifest, plus downstream (covered in Module 9)
```

Note: `--models`/`-m` and `--model` are **deprecated** in favor of `--select`/`-s` — using the old flag in an interview answer is a small but noticeable tell that your knowledge is a few years stale.

## Practice (10 min)

Sketch (on paper or in a scratch file) the staging → marts layering for a project with three raw sources: `stripe.payments`, `shopify.orders`, `shopify.customers`. Name every model you'd create (`stg_stripe__payments`, `stg_shopify__orders`, `stg_shopify__customers`, then `int_orders_joined_payments`, then `fct_orders`, `dim_customers`). Write the `sources.yml` block for `shopify.orders` including a freshness check.

## Recall (5 min flashcards)

- Layering: sources (YAML only) → staging (1:1 per source table, thin) → intermediate (optional, reusable joins) → marts (wide, BI-facing).
- Naming: `stg_<source>__<entity>`, `int_<entity>s_<verb>`, `dim_`/`fct_` for marts.
- `dbt_project.yml` = project config/defaults. `packages.yml`/`dependencies.yml` = external package (and cross-project) deps. `profiles.yml` = credentials, kept out of the repo.
- `--select`/`-s` is current; `--models`/`-m` is deprecated.
- `+` before = upstream, `+` after = downstream.

## Interview (2 min)

**Q: "Why not just write one big model straight from source to `fct_orders`?"**
Because you lose testability at each stage, lineage granularity, reusability of intermediate logic across multiple marts, and the ability to isolate where a bug was introduced. This is the staging-layer justification interviewers are listening for.

**Q: "When would you skip the intermediate layer?"**
Small/simple projects where a direct staging→mart join adds no real reuse value — over-layering a five-model project is itself a mistake worth naming; dbt Labs' own guidance is "add structure as complexity demands it," not upfront for its own sake.

**Common mistake:** putting business logic (joins across sources, aggregations) into the staging layer instead of keeping it a thin renaming/casting pass — this is one of the most common real-world dbt code review complaints and a good "mistake you've seen" interview answer.
