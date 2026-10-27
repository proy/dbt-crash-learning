# Module 9 — Orchestration, State, CI/CD & Advanced CI
**Time: 60 minutes | Priority: Core**

## Learn (30 min)

### `defer` and `state:` — the foundation of efficient CI

Running a full `dbt build` on every PR against a huge project is slow and expensive. dbt's state comparison lets you build/test **only what changed**, while still letting unchanged downstream models resolve against production data:

```bash
dbt build --select state:modified+ --defer --state ./prod-manifest
```

- **`state:modified`** — compares your current project's manifest against a stored (usually production) manifest and selects only nodes whose SQL/config/YAML actually changed.
- **`--defer`** — for any node **not** selected to run, resolve its `ref()` against the *deferred* (production) manifest's actual relation instead of failing because it doesn't exist in your dev/CI schema. This is what makes "just build the 3 models I changed" actually work when they depend on 200 unchanged upstream models.
- This combo is what "Slim CI" means in dbt terminology — it's the baseline, included-in-most-tiers CI pattern, distinct from the paid **Advanced CI** feature below.

### Advanced CI (Enterprise/Enterprise+, dbt Platform)

Slim CI tells you *whether the code ran successfully*. Advanced CI's **Compare Changes** tells you *what actually changed in the data* as a result — genuinely different information:

- Compares the last-applied production state against the PR's changes: added/removed rows, changed columns, primary key differences.
- Surfaces as a **Compare tab** in the job run and as a comment directly on the Git PR — reviewers see data impact without leaving their PR workflow.
- Uses cached sample data (up to 100 records per modified model, encrypted, ~30-day retention) rather than re-querying full production data every time, to control cost.
- `event_time`-aware: if your CI job only processes a recent time window (say, last 7 days) while prod has full history, Advanced CI understands the overlapping window instead of falsely reporting "all older rows were deleted."
- **Limitation to know:** CI and production must live on the *same* database host/connection — Compare Changes can't diff across genuinely separate warehouse connections.
- The dbt Fusion-powered VS Code extension brings an equivalent **Compare Changes** *locally*, before you even open a PR — comparing your working copy against your local target, on demand in the editor.

**State-Aware Orchestration** (introduced 2025, evolving through 2026) goes a layer further at the *job* level: it can skip/clone nodes automatically when it detects neither logic nor upstream data actually changed since the last successful run — cost optimization on top of the state-comparison primitive, enabled via account/job settings ("Enable Fusion cost optimization features").

### Deployment environments and jobs (dbt Platform)

Standard pattern: a `dev` environment per developer (or one shared, non-authoritative), a `CI` environment that runs on every PR (ephemeral, deferred against prod), and a `prod` environment running scheduled jobs that materialize the tables BI tools actually query. Jobs can chain: e.g., a nightly `prod` job followed by a `dbt docs generate`/Catalog refresh. Release tracks (**Latest / Compatible / Extended**, GA March 2025) let you choose how aggressively an environment picks up new dbt engine versions — Latest for early access to new features, Compatible for stability, Extended for long-term-support-style conservatism.

### dbt Cloud CLI vs. dbt Core CLI vs. Fusion CLI — a genuinely confusing area, get it straight

- **dbt Core CLI** — the original, connects directly to your warehouse using local credentials in `profiles.yml`, runs entirely on your machine, free, works with any dbt Core-compatible orchestrator (Airflow, Dagster, cron).
- **dbt Cloud CLI** (a distinct product from dbt Core's CLI) — same commands, but authenticates against dbt Platform and can leverage Platform-side features (like Platform-hosted state/deferral) while still running locally.
- **Fusion CLI (`dbtf`)** — the new Rust engine's CLI, free for local use even without a Platform account, and the path being consolidated toward as dbt Core v2 and Fusion converge.
The practical interview-safe framing: know that multiple CLIs with overlapping-but-different capabilities have existed simultaneously through this transition period, and that the direction of travel in 2026 is consolidation onto the Fusion/Core-v2-based tooling — don't confidently claim one is "the" current CLI without that caveat.

### Orchestrating dbt from outside dbt Platform

Many real production stacks run dbt (Core or Fusion) as one task inside a broader orchestrator — Airflow (via `cosmos` or plain `BashOperator`/`KubernetesPodOperator`), Dagster (native dbt integration with asset-level lineage), or Prefect — specifically because ingestion, ML training, and reverse-ETL steps need to be sequenced alongside dbt, and dbt Platform's own scheduler only orchestrates dbt itself. Being able to name this tradeoff (dbt Platform = simplest for dbt-only orchestration + built-in CI/governance; external orchestrator = necessary once dbt is one node in a larger DAG) is a strong senior-level signal.

## Practice (20 min)

1. Write the exact CLI invocation for a CI job that should build and test only models changed vs. production, plus everything downstream, deferring unchanged upstream refs to prod.
2. Explain, in your own words, the difference between what Slim CI (state + defer) tells a reviewer and what Advanced CI's Compare Changes additionally tells them — write one sentence each.
3. Sketch (bullet list is fine) a 3-environment setup (dev/CI/prod) for a mid-size team, naming what triggers each environment's jobs and what each defers against.

## Recall (7 min flashcards)

- `state:modified` selects changed nodes vs. a stored manifest; `--defer` resolves unselected refs against that deferred manifest's real relations. Together = Slim CI.
- Advanced CI (Enterprise+) = Compare Changes: actual data diffs (rows/columns/keys) on the PR, not just "did it run" — cached samples, `event_time`-aware, same-connection requirement.
- State-Aware Orchestration = job-level skip/clone when nothing actually changed — cost optimization layered on state comparison.
- Release tracks: Latest (early access) / Compatible (stable) / Extended (LTS-style).
- Three CLIs coexisted through the transition: dbt Core CLI (local, free), dbt Cloud CLI (Platform-authenticated), Fusion CLI `dbtf` (new Rust engine, free, convergence target).
- External orchestrators (Airflow/Dagster/Prefect) are common when dbt is one node among ingestion/ML/reverse-ETL steps — dbt Platform's scheduler is dbt-only.

## Interview (3 min)

**Q: "A PR passes CI (all tests green) but production data still looks wrong after merge. How is that possible given Slim CI?"**
Slim CI validates that *changed* models build and pass *tests that exist* — it doesn't validate against full production data volume/distribution (CI often runs on a sample or fresh schema), doesn't catch missing tests (untested logic can still be wrong), and basic Slim CI (unlike Advanced CI) doesn't surface *data*-level diffs at all, only pass/fail. This is exactly the gap Advanced CI's Compare Changes is built to close.

**Q: "Why would a team on dbt Platform still stand up Airflow alongside it?"**
Because dbt Platform orchestrates dbt jobs only — once you need to sequence ingestion → dbt → ML feature pipeline → reverse ETL as one coherent DAG with cross-tool dependency awareness, an external orchestrator becomes the source of truth for scheduling, with dbt as one task type within it.

**Common mistake:** conflating "our tests passed" with "our data is correct" — tests only check what someone thought to test; naming this distinction unprompted is a strong maturity signal.
