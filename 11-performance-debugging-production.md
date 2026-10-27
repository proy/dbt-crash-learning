# Module 11 — Performance, Debugging & Production Best Practices
**Time: 45 minutes | Priority: Core**

## Learn (22 min)

### Diagnosing slow dbt runs — a structured approach interviewers want to hear

1. **Isolate**: is it slow to *parse/compile* (project-structure/macro-complexity problem) or slow to *execute* (warehouse-side problem)? `dbt compile` alone tells you parse time separate from execution time.
2. **Check materialization strategy**: a `view` queried repeatedly by BI tools is often actually a warehouse-cost problem disguised as a "dbt is slow" complaint — the fix is often `table`/`incremental`, not a dbt-level tweak.
3. **Check incremental logic**: an incremental model doing a full scan every run because the `is_incremental()` filter is missing or ineffective (e.g., filtering on a column without a matching partition/cluster key) — this is one of the most common real production perf bugs.
4. **Warehouse-side tuning**: partitioning/clustering keys (`partition_by`/`cluster_by` configs on BigQuery, Snowflake clustering keys) aligned with your actual filter/join columns; warehouse-level concurrency/queueing (a job may be "slow" purely from queueing behind other workloads, not from the SQL itself).
5. **DAG-level parallelism**: `dbt run --threads N` controls how many independent-branch models run concurrently — too low wastes warehouse headroom; too high can create warehouse-side contention. Worth naming as a tunable, not just a constant.

### Common production failure modes and how to talk about them

- **Schema drift** — an upstream source added/renamed/removed a column outside of dbt's control. Contracts (Module 4/7) catch this at build time for contracted models; for everything else, `dbt source freshness` plus data tests are your early-warning system, and `on_schema_change` config (`ignore`/`append_new_columns`/`sync_all_columns`/`fail`) controls how incremental models react when the query's column set changes.
- **Silent partial failure** — a job "succeeded" but a test with `severity: warn` quietly failed; production monitoring needs to actually surface warn-level results, not just hard failures, or they get ignored indefinitely.
- **Full-refresh accidents** — someone runs `--full-refresh` on a huge incremental model in prod during business hours, causing a long lock/rebuild window; mitigations include restricting who can trigger full-refresh jobs and scheduling large full-refreshes off-peak.
- **Credential/permission drift** — a service account loses a grant after a warehouse-side security change, and the failure surfaces as a cryptic SQL permission error rather than an obvious "dbt problem" — knowing to check grants first, not just dbt config, when errors look database-side is a practical debugging habit worth stating.

### Write-Audit-Publish (WAP) pattern

A production-hardening pattern where a model builds into a **staging/audit** location first, data tests run against that staging output, and only on success is it **published** (swapped/promoted) into the production-facing table — preventing bad data from ever being visible to consumers, versus "build directly into prod, then test, then maybe roll back." This is more manual/custom in dbt Core than a built-in feature, but it's a real pattern experienced teams implement (via blue/green schema swaps, custom macros, or platform-level equivalents), and citing it shows production maturity beyond "just run `dbt build`."

### Cost management

- **`--sample` flag** (2025+) — time-based sampling for `run`/`build` to cut compute cost/time in dev/CI without processing full history.
- **Cost Insights** (GA 2026, Snowflake/BigQuery/Databricks, Snowflake Adaptive Warehouse support added) — surfaces per-model/per-job warehouse compute cost directly in dbt Platform, turning "which model is expensive" from a warehouse-console archaeology exercise into a first-class dbt-side metric.
- Materialization choice itself is often the biggest cost lever: an over-materialized `table` that's rarely queried wastes compute on every rebuild; an under-materialized `view` queried constantly wastes compute on every *query* instead — the right choice depends on write-frequency vs. read-frequency, not a fixed rule.

### Documentation and testability as production hygiene

`dbt docs generate`/Catalog, `description:` fields on models/columns, and consistent tagging (`tags: ['finance','nightly']`) aren't cosmetic — they're what makes impact analysis ("what breaks if I change this") tractable at scale, and what onboarding a new team member to a large project actually depends on.

## Practice (18 min)

Work through this scenario and write your actual answer (not just read it): *"A nightly job that used to take 20 minutes now takes 2 hours. Nothing in the dbt project changed in the last week. Diagnose."* Structure your answer using the 5-step approach above — what would you check first, second, third, and what tool/command would you actually run at each step (e.g., compare `run_results.json` timing across recent runs to see which specific model(s) regressed, check warehouse query history for that model's compiled SQL, check upstream source row-count growth).

## Recall (5 min flashcards)

- Diagnose slow runs: parse vs. execute time → materialization fit → incremental filter effectiveness → warehouse tuning (partition/cluster keys) → thread concurrency.
- `on_schema_change`: ignore / append_new_columns / sync_all_columns / fail — controls incremental reaction to upstream schema drift.
- `severity: warn` tests can fail silently if nobody's monitoring them — surfacing warns matters as much as erroring on failures.
- WAP (Write-Audit-Publish): build to staging → test → promote to prod, prevents bad data ever being visible.
- `--sample` flag and Cost Insights = 2025-2026 cost-control additions.
- Docs/descriptions/tags aren't cosmetic — they're the substrate for impact analysis and onboarding at scale.

## Interview (2 min)

**Q: "How do you decide between fixing a slow model at the dbt level vs. the warehouse level?"**
If the compiled SQL itself is inefficient (unnecessary full scans, missing filter pushdown, redundant joins) — that's a dbt/model-authoring fix. If the SQL is reasonable but the warehouse lacks the physical structures to execute it efficiently (no partitioning/clustering aligned to the filter pattern) — that's a warehouse-config fix, often expressed *through* dbt's `partition_by`/`cluster_by` configs, which is the actual overlap point between the two layers.

**Q: "What's your philosophy on `severity: warn` vs. `severity: error`?"**
`error` for anything that should block a deploy/build (real data integrity risk); `warn` for known, accepted edge cases you still want visibility into over time — but only if something actually reads and acts on warnings; a warn nobody monitors is worse than no test, because it creates false confidence that "it's covered."

**Common mistake:** treating a warehouse permission error as "a dbt bug" and spending time debugging dbt config before checking grants/roles — naming this as a real debugging trap you've hit shows practical experience.
