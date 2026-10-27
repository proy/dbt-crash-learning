# Advanced dbt Interview Preparation Course

**For:** engineers who already have hands-on dbt Core experience and have an interview coming up.
**Not for:** people who have never used dbt — this assumes you know what a model, a `ref()`, and `dbt run` are.
**Research basis:** dbt Labs' official Developer Hub, release notes, and blog, current as of **August 2026**. Every module flags what's dbt Core (Python, pre-2026), dbt Core v2 / dbt Fusion (Rust engine), dbt Platform (formerly dbt Cloud), and whether a feature is GA, beta/preview, or deprecated.

## Why this course exists

Most dbt tutorials teach the 2020 mental model: dbt Core CLI, Jinja, a `target/` folder, and stop there. Since 2023, dbt Labs has shipped unit tests, model contracts and versioning, dbt Mesh, microbatch incremental models, a rewritten Semantic Layer (MetricFlow), Advanced CI, and — starting in 2025 — a full architectural pivot: the **Rust-based Fusion engine**, a **dbt Core v2** rewrite, and a rebrand of dbt Cloud into **dbt Platform** (with Explorer → Catalog, Cloud IDE → Studio IDE, Visual Editor → Canvas). An interviewer in 2026 expects you to speak fluently about both the fundamentals *and* this evolution.

## How to use this course

Each module follows **Learn → Practice → Recall → Interview**:

- **Learn** — the concept, why it exists, how it works, current version/status.
- **Practice** — a short hands-on task or code snippet you should actually run or type out, not just read.
- **Recall** — a compressed, flashcard-style summary for your day-before-the-interview pass.
- **Interview** — real-style questions with the key points a strong answer hits, including "when NOT to use this" and common mistakes, which is what separates senior answers from junior ones.

Do the modules in order — later modules (governance, Semantic Layer, platform architecture) assume you're solid on materializations and testing. If you're very short on time, modules marked **[Core]** below are the minimum revision set; modules marked **[Advanced]** are what differentiates a senior/staff-level interview answer.

## Module map and time budget

| # | Module | Time | Priority |
|---|--------|------|----------|
| 1 | Modern dbt Architecture & Mental Model | 45 min | Core |
| 2 | Project Structure, Sources & Staging Conventions | 35 min | Core |
| 3 | Materializations & Incremental Models Deep Dive | 70 min | Core |
| 4 | Testing: Data Tests, Unit Tests & Contracts | 60 min | Core |
| 5 | Jinja, Macros & Packages | 45 min | Core |
| 6 | Snapshots & Slowly Changing Dimensions | 30 min | Core |
| 7 | Governance at Scale: dbt Mesh, Contracts, Versions, Groups & Access | 60 min | Advanced |
| 8 | Semantic Layer & MetricFlow | 50 min | Advanced |
| 9 | Orchestration, State, CI/CD & Advanced CI | 60 min | Core |
| 10 | The Modern dbt Platform: Fusion Engine, dbt Core v2, Studio IDE & AI Tooling | 45 min | Advanced |
| 11 | Performance, Debugging & Production Best Practices | 45 min | Core |
| 12 | Interview Simulation & Rapid Recall | 45 min | Core |

**Total: ~590 minutes ≈ 9 hours 50 minutes.**

Suggested split across days: Day 1 = Modules 1–5 (~4h15m), Day 2 = Modules 6–9 (~3h20m), Day 3 = Modules 10–12 (~2h15m). Adjust to your interview date.

## Terminology cheat sheet (read this first)

The naming changed a lot in 2025. If you see the old term in an older blog post or in your own memory, here's the current mapping as of August 2026:

| Old name | Current name (2026) | Notes |
|---|---|---|
| dbt Cloud | **dbt Platform** | Rebranded ~May 2025. "dbt Cloud" is now colloquial, not the product name. |
| dbt Cloud IDE | **Studio IDE** | Browser-based, now runs on the Fusion engine. |
| Explorer | **Catalog** | Metadata/lineage browser, now with cross-project global search. |
| Visual Editor | **Canvas** | Drag-and-drop model building, GA since May 2025. |
| dbt Core (Python) | **dbt Core v1.x** | Still maintained, Apache 2.0, PyPI `dbt-core`. Latest as of Aug 2026: **1.12.x**. |
| — | **dbt Core v2** | Rust/ADBC rewrite, Apache 2.0, released to beta in 2026. Shares its foundation with Fusion. |
| — | **dbt Fusion engine** | Rust-based engine (built on former SDF Labs technology), free CLI, adds proprietary IDE/CI features on top of Core v2. |

You do not need to memorize exact dates — you need to be able to say, correctly, which product a feature belongs to and roughly when it showed up. That's what interviewers are actually checking.
