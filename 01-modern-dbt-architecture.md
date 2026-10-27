# Module 1 — Modern dbt Architecture & Mental Model
**Time: 45 minutes | Priority: Core**

## Learn (25 min)

### What dbt actually does

dbt is **not** an ETL/orchestration tool and it does **not** move data. It is a **transformation and modeling layer** that runs entirely inside your warehouse's compute (Snowflake, BigQuery, Databricks, Redshift, Postgres, etc.). dbt takes SQL (or, for a subset of platforms, Python) that you write, compiles it (resolving Jinja, `ref()`, `source()`, macros, variables), and executes the compiled SQL against the warehouse using the warehouse's own engine.

Three separate verbs matter here, and interviewers like to probe whether you actually know the difference:

- **`dbt compile`** — resolves Jinja/refs into raw SQL, writes it to `target/compiled/`, does **not** touch the warehouse.
- **`dbt run`** — compiles *and* executes model SQL (materializes tables/views).
- **`dbt build`** — the modern default: runs models, tests, snapshots, and seeds together, respecting the DAG, and stops downstream nodes if a test fails upstream (unlike running `dbt run` then `dbt test` separately, where a failed test doesn't stop anything because materialization already happened).

### The DAG is the whole point

Every model, source, seed, snapshot, and test is a node in a directed acyclic graph, built from `ref()` and `source()` calls. This is what gives you: safe parallel execution, `--select`/`--exclude` targeting, lineage visualization, and impact analysis ("what breaks if I change this column"). If you can't explain how dbt builds the DAG from static analysis of your project files (it parses `ref()`/`source()` calls without executing SQL), you'll struggle with any question about selectors or state comparison later in this course.

### ELT, not ETL

dbt assumes data is already **loaded** into the warehouse (by Fivetran, Airbyte, custom pipelines, etc.) and its job starts *after* that — hence ELT (Extract-Load-**T**ransform) rather than ETL. This is a common interview framing question: "why did the industry move from ETL to ELT?" — cheap elastic cloud warehouse compute made it viable to load raw data first and transform it declaratively in SQL, instead of transforming in a separate ETL engine before load.

### The product landscape in 2026 — get this exactly right

This is the single most common area candidates get wrong or out of date on:

- **dbt Core (v1.x)** — the original, Apache 2.0, Python-based, pip/PyPI-installed CLI. Still maintained; latest is the 1.12.x line. Free, open source, runs anywhere you can run Python.
- **dbt Core v2** — a from-scratch rewrite of the *open-source* foundation in **Rust**, using **ADBC** and **Parquet** artifacts instead of Python/JSON. Went to beta in 2026. Still Apache 2.0. Removes the Python virtual-environment install pain and gives large parse-time improvements.
- **dbt Fusion engine** — a Rust-based engine built on the dbt Core v2 foundation, developed from technology dbt Labs acquired from **SDF Labs**. It is a true SQL compiler — it *understands* your SQL (types, column-level lineage, errors) rather than just templating and shipping it to the warehouse. Free to use locally (even without a dbt Platform account), and it also powers the paid IDE/CI features. As of 2026 the "two-engine era" (separate Core-Python vs Fusion-Rust codebases) is explicitly being retired in favor of one shared foundation.
- **dbt Platform** (renamed from **dbt Cloud** in 2025) — the managed SaaS: hosted orchestration/scheduling, the browser **Studio IDE** (was: Cloud IDE), **Catalog** (was: Explorer) for lineage/metadata, **Canvas** (was: Visual Editor) for no-code model building, Advanced CI, the Semantic Layer service, RBAC/SSO/SCIM, and AI features (Copilot, Wizard, Developer Agent).

The interview-safe way to say this: *"dbt Core is the free open-source engine you can run anywhere; dbt Platform is the managed service built around it for scheduling, collaboration, governance, and enterprise features; and since 2025 dbt Labs has been rewriting the underlying engine itself in Rust — first as the standalone Fusion engine, now converging with dbt Core v2 into one shared foundation used by both the free CLI and the platform."*

### Why the Rust rewrite happened (not just trivia)

Two things Python-based dbt Core structurally couldn't fix:

1. **Performance** — Python's parse/compile loop doesn't scale well on large projects (thousands of models); this is felt directly as slow `dbt compile`/`dbt run` startup.
2. **SQL comprehension** — dbt Core renders Jinja into SQL text but never *understands* the SQL; it can't tell you that a column is a string being summed, or give live type errors, because it isn't a SQL compiler. Fusion is.

## Practice (10 min)

If you have dbt installed locally, run these and actually look at the output — this is the kind of muscle memory interviewers probe for:

```bash
dbt compile --select stg_orders     # inspect target/compiled/.../stg_orders.sql — no warehouse call
dbt run --select stg_orders         # now it executes
dbt build --select stg_orders+      # models + tests + snapshots downstream of stg_orders, dag-aware
dbt ls --select stg_orders+ --resource-type model   # list only the models in that selection, no execution
```

If you don't have a project handy, at minimum read through a `manifest.json` from any project (`target/manifest.json` after a compile) and identify: `nodes`, `sources`, `child_map`/`parent_map`. That's literally what powers lineage and selection.

## Recall (5 min flashcards)

- dbt transforms, doesn't move data → ELT, not ETL.
- `compile` = render SQL only. `run` = render + execute. `build` = run + test + snapshot + seed, DAG-ordered, fails fast downstream.
- DAG is built statically from `ref()`/`source()` — no SQL execution needed to know the graph shape.
- dbt Core v1.x (Python) → still alive, 1.12.x latest, Apache 2.0.
- dbt Core v2 (Rust/ADBC) → open-source foundation rewrite, beta in 2026.
- dbt Fusion → Rust engine built on Core v2, from SDF Labs tech, true SQL compiler, free CLI.
- dbt Cloud is now called **dbt Platform**. Explorer→Catalog, Cloud IDE→Studio IDE, Visual Editor→Canvas.

## Interview (5 min)

**Q: "Walk me through what happens when I run `dbt build`."**
Strong answer covers: DAG resolution → topological execution order → for each node, Jinja compilation (ref/source resolved to real relation names, macros expanded) → SQL sent to warehouse → for models, materialization strategy applied (create/replace/merge); for tests, a query is run and failures counted; if a node fails, dbt marks descendants as skipped rather than running them blind.

**Q: "What's the practical difference between dbt Core and dbt Platform, and when would you choose self-hosted orchestration (Airflow/Dagster) over dbt Platform's own scheduler?"**
Good answer distinguishes the free CLI/engine from the paid orchestration+governance layer, and notes that many teams run dbt Core (or Fusion) *inside* Airflow/Dagster/Prefect precisely because they want unified orchestration across dbt and non-dbt tasks (ingestion, ML, reverse ETL) — dbt Platform's scheduler is dbt-only.

**Q: "Explain the difference between dbt Fusion and dbt Core v2 in your own words."**
Watch for candidates who conflate them. The correct distinction: Core v2 is the shared open-source (Apache 2.0) Rust foundation; Fusion is dbt Labs' enhanced binary built on that foundation with additional proprietary capabilities (IDE integration, IDE-side compare-changes, etc.) under a more permissive-but-not-fully-open license. Saying "they're the same thing" is a red flag answer; saying "Fusion is dbt Core v2 plus dbt Labs' value-add layer" is correct at interview depth.

**Common mistake to avoid saying out loud:** describing dbt as "an ETL tool" or "a data pipeline orchestrator" — both signal you don't actually understand the T-only, warehouse-native execution model.
