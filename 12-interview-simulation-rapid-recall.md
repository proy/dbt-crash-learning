# Module 12 — Interview Simulation & Rapid Recall
**Time: 45 minutes | Priority: Core (do this the day of/before your interview)**

## How to use this module

This is not new content — it's rehearsal. Work through each section by actually speaking or writing your answer before checking the "what a strong answer covers" notes. The goal is fluency under mild time pressure, which is what an interview actually tests.

## Part 1 — System design / scenario questions (20 min)

Give yourself 3-4 minutes per question, out loud if possible.

**1. "Design a dbt project for an e-commerce company with orders, customers, payments (from Stripe), and product catalog data, where Finance needs revenue reporting and Marketing needs customer segmentation. Walk me through your structure."**
Strong answer touches: sources for each raw system → staging layer (1:1 thin models) → intermediate joins where reused (e.g. orders+payments) → marts split by domain (`finance/fct_revenue`, `marketing/dim_customer_segments`) → consider whether Finance/Marketing eventually warrant separate groups/projects (Mesh) as the org scales → mention where Semantic Layer metrics (revenue, LTV) would live instead of being duplicated across marts.

**2. "A 500-model dbt project takes 45 minutes to `dbt build` on every PR. How do you speed up CI without buying more warehouse compute?"**
Strong answer: Slim CI (`state:modified+` with `--defer`) so PRs only build/test changed models plus downstream, not everything; check for unnecessary full-refreshes triggered in CI; consider `--sample` for dev/CI; check thread concurrency; consider whether some tests are redundantly expensive (e.g., `dbt_expectations` distribution checks on huge tables every PR) and whether they need to run on every PR vs. nightly only.

**3. "How would you migrate a monolithic 2000-model dbt project into a dbt Mesh architecture, with zero downtime for consumers?"**
Strong answer: identify natural domain boundaries (often by team ownership already); pick the first, most self-contained domain to extract; mark its boundary models `public` + `contract: enforced` *before* splitting anything physically, so consumers are already depending on a stable interface; use model versioning for any shape changes needed during the split; extract the project, wire up cross-project refs, validate via Advanced CI compare-changes that output is identical; repeat domain by domain rather than a single flag-day cutover.

**4. "A `dim_customers` model needs a breaking schema change (renaming a column used by 12 downstream dashboards). Walk me through, end to end."**
Strong answer: confirm it's actually contracted/public (if not, this is exactly why it should have been); bump `latest_version`, define the new version alongside the old; communicate a deprecation window to the 12 consumers (this is a people/process step, not just a YAML step — say this explicitly); once consumers migrate (`ref(..., version=2)` → default `ref()`), retire v1.

**5. "How do you know whether a table should be `incremental` with `microbatch`, `incremental` with `merge`, or just a plain `table`?"**
Strong answer: data volume/growth rate first (small enough to rebuild cheaply → `table`); if incremental is warranted, is it time-partitioned append-mostly data with a reliable event-time column and a need for cheap retries/backfills → `microbatch`; if it's upsert-heavy dimension-style data without a clean time-bucket pattern → `merge`. Explicitly reject microbatch for slowly-changing-dimension-style data (that's snapshots' job, not incremental's).

## Part 2 — Rapid recall flashcards (15 min, run through fast, don't linger)

- ELT not ETL; dbt = transform only, runs in warehouse compute.
- `compile` (render) < `run` (render+execute) < `build` (run+test+snapshot+seed, DAG-ordered, fail-fast).
- dbt Core v1.x (Python, 1.12.x) alive; dbt Core v2 (Rust/ADBC) beta 2026; Fusion (Rust, ex-SDF Labs) built on Core v2; dbt Cloud renamed dbt Platform (2025); Explorer→Catalog, Cloud IDE→Studio IDE, Visual Editor→Canvas.
- Layering: sources → staging (1:1 thin) → intermediate (reusable joins) → marts (BI-facing).
- `--select`/`-s` current; `--models`/`-m` deprecated.
- Materializations: view, table, incremental, ephemeral, materialized view (Snowflake = Dynamic Tables instead); python models = table/incremental only.
- Incremental strategies: append, merge (needs unique_key), delete+insert, insert_overwrite (needs partition_by).
- Microbatch (1.9+ GA): `event_time`+`batch_size`+`begin`, set event_time on upstreams too, `dbt retry` reprocesses failed batches only.
- Data tests (post-build, content) vs unit tests (pre-build, static mocks, logic) vs contracts (build-time shape gate).
- `data_tests:` key, params under `arguments:`. `store_failures: true` for debugging.
- Snapshots = Type 2 SCD; YAML syntax since 1.9 (was Jinja block); `timestamp` vs `check` strategy; `hard_deletes: ignore|invalidate|new_record`.
- Macros = Jinja functions; a `test_x` macro becomes a reusable generic test; `dbt_utils`/`dbt_expectations`/`codegen`/`audit_helper` by name.
- Mesh = cross-project ref + groups (ownership) + access (private/protected/public) + contracts + versions; Enterprise+ only; don't over-adopt early.
- Semantic Layer/MetricFlow: entities/dimensions/measures on semantic models; 5 metric types (simple/ratio/derived/cumulative/conversion); saved queries; new inline YAML spec (2026).
- `state:modified` + `--defer` = Slim CI. Advanced CI = Compare Changes (actual data diffs, Enterprise+, same-connection requirement, event_time-aware).
- Release tracks: Latest/Compatible/Extended.
- WAP pattern: stage → test → promote, prevents bad data reaching consumers.
- `on_schema_change`: ignore/append_new_columns/sync_all_columns/fail.

## Part 3 — Behavioral / experience questions (10 min)

These get asked in nearly every analytics engineer interview alongside the technical ones. Prepare one real (or realistic composite) story for each:

- "Tell me about a time a dbt test caught something important before it hit production." → structure as: what test, what it caught, what would have happened otherwise, what you changed afterward (added a test, added monitoring, changed a process).
- "Tell me about a time you had to refactor a messy dbt project." → structure as: what was wrong (layering violations, no tests, hardcoded refs, no docs), what you prioritized first and why, how you validated the refactor didn't change output (mention `audit_helper` or a manual row-diff approach).
- "Tell me about a disagreement with a stakeholder over a metric definition." → structure as: what the disagreement was, how you resolved it technically (often: this is exactly a Semantic Layer/governance problem — naming that connection is a strong close), what changed in your process afterward.
- "What's a dbt best practice you disagree with, or apply differently than the docs recommend?" → have a real, specific opinion ready (e.g., "I don't always build a full intermediate layer for small projects," or "I'm cautious about over-contracting internal models") — a generic "I always follow best practices" answer reads as inexperience, not diligence.

## Final check before your interview

Re-read the Module 1 terminology table and the Module 10 "common mistake" note one more time — the single highest-value thing this course can leave you with is: never state a feature's GA/beta status or version-introduction with false confidence. "I believe that landed around dbt 1.9, and I'd confirm the exact version against the current docs" is a *better* interview answer than a confident wrong date.
