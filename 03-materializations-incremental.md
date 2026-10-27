# Module 3 — Materializations & Incremental Models Deep Dive
**Time: 70 minutes | Priority: Core**

## Learn (35 min)

### The five materializations

| Materialization | What it does | When to use | Cost |
|---|---|---|---|
| `view` | `CREATE VIEW AS` every run | Lightweight, low-query-volume, always-fresh data | Cheap to build, expensive to query repeatedly |
| `table` | `CREATE TABLE AS` (full rebuild) every run | Small/medium data, complex transforms queried often | Expensive to build, cheap to query |
| `incremental` | Only processes new/changed rows | Large, append-mostly, time-series data | Cheap to build after first run, adds complexity |
| `ephemeral` | Interpolated as a CTE into referencing models, never persisted | Reusable snippets that don't need to be queried directly | No storage, but can't be tested/queried standalone, and can bloat compiled SQL if overused |
| `materialized view` | Warehouse-native MV with (often automatic) refresh | Near-real-time aggregates where the warehouse supports incremental refresh natively | GA on most adapters (Postgres, BigQuery, Databricks, Redshift); **Snowflake uses Dynamic Tables instead**, not this materialization |

Python models (`.py`) are supported on Snowflake, Databricks, and BigQuery, and are restricted to `table` and `incremental` — no `view`/`ephemeral` for Python, because those materializations rely on SQL-level view/CTE semantics.

### Incremental strategies — the part interviewers actually probe hardest

An incremental model without `is_incremental()` logic just re-runs a full table build every time, which defeats the purpose. The classic pattern:

```sql
{{ config(materialized='incremental', unique_key='order_id', incremental_strategy='merge') }}

select * from {{ ref('stg_orders') }}
{% if is_incremental() %}
  where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```

Strategies, and which platforms use which under the hood:

- **`append`** — just inserts, no dedup. Fine for pure event logs with no late-arriving duplicates.
- **`merge`** — `MERGE` / upsert on `unique_key`. Most common, requires warehouse `MERGE` support (Snowflake, BigQuery, Databricks, Postgres via emulation).
- **`delete+insert`** — deletes matching rows then inserts new ones; used where native `MERGE` isn't available or preferred (Redshift default).
- **`insert_overwrite`** — replaces whole partitions atomically; BigQuery/Spark, requires `partition_by`.

**Common mistake:** forgetting `unique_key` with `merge`/`delete+insert` → silent duplicate rows instead of updates. Also: forgetting the `is_incremental()` guard entirely means the "incremental" model just does a full scan every run and provides zero performance benefit while adding complexity — worse than just using `table`.

### Microbatch — the newest incremental strategy (dbt Core 1.9+, now GA)

Microbatch solves a specific pain: manually writing lookback-window logic for late-arriving data, and being unable to cheaply retry/parallelize a failed backfill. Instead of one big incremental query with manual date filtering, you declare the batching parameters and dbt handles slicing:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='session_start',
    batch_size='day',
    begin='2020-01-01',
    lookback=1,
    full_refresh=false
) }}

select * from {{ ref('stg_events') }}
```

- `event_time` must also be set on the **upstream models** you reference, or dbt can't auto-filter them and falls back to full scans per batch — this "set it on upstreams too" gotcha is a favorite trick question.
- `lookback` batches are reprocessed each run to catch late-arriving data (default 1).
- `dbt run --event-time-start ... --event-time-end ...` backfills a specific range; `dbt retry` reprocesses only failed batches — this is the single biggest operational win over hand-rolled incremental logic (no more "re-run the whole 2-year history because 3 days failed").
- Batch replacement mechanism is adapter-native under the hood: `merge` (Postgres), `delete+insert` (Redshift/Snowflake), `insert_overwrite` (BigQuery/Spark), `replace_where` (Databricks) — you don't choose this, dbt picks it per adapter.
- Timezone note: dbt currently assumes UTC for `event_time`, `begin`, and the CLI backfill flags.
- **When NOT to use it:** no reliable event-time column, or the model is dimension-style data updated by upsert rather than append (e.g. `dim_customers` where attributes change in place) — microbatch is for time-partitioned fact-style data, not slowly-changing dimensions (that's what snapshots are for — Module 6).

### Full refresh

`dbt run --full-refresh` drops and rebuilds an incremental model from scratch — necessary after a schema change, a backfill of corrected historical logic, or when you suspect drift. Good practice: set `full_refresh=false` in config for microbatch models where a full historical rebuild would be prohibitively expensive, and provide an explicit backfill command instead.

## Practice (25 min)

Write three variants of the same incremental model against a hypothetical `stg_events(event_id, event_time, user_id, event_type)` table:

1. A `merge`-strategy incremental keyed on `event_id`, filtered with a manual `is_incremental()` block.
2. The same model rewritten as `microbatch` with `event_time`, `batch_size='day'`, `begin`, and `lookback=2`.
3. Explain in one sentence each: what happens on the very first run of each; what happens if you run each one twice in a row with no new data; what happens if 3 days of source data arrive 2 days late.

## Recall (7 min flashcards)

- 5 materializations: view, table, incremental, ephemeral, materialized view (+ python models limited to table/incremental).
- Snowflake materialized views = Dynamic Tables, not the `materialized view` config.
- Incremental strategies: append (no dedup), merge (upsert), delete+insert, insert_overwrite (partition swap).
- Missing `unique_key` on merge/delete+insert = duplicate rows.
- Microbatch (1.9+, GA): declare `event_time`+`batch_size`+`begin`, no manual `is_incremental()` needed, auto-retries failed batches, needs `event_time` set on upstream models too.
- `dbt retry` reprocesses only failed microbatches. `--event-time-start/--end` backfills a range.
- Full refresh drops+rebuilds; set `full_refresh=false` on microbatch models where that's too costly.

## Interview (3 min)

**Q: "A stakeholder says yesterday's incremental run duplicated rows. Walk me through your debugging steps."**
Strong answer: check whether `unique_key` is set and actually unique in source data (test it!); check whether the merge/delete+insert predicate window is too narrow and missed the update but caught a re-insert; check for late-arriving data outside the lookback window; check whether someone ran `--full-refresh` concurrently with an incremental run causing a race.

**Q: "When would microbatch be the wrong choice, even though your data is large and time-based?"**
When updates aren't append-only within a time bucket — e.g., a support ticket that gets reopened and updated weeks after its `created_at` — because microbatch reprocesses only `lookback` batches near the latest, an update far outside that window won't be picked up. That's a real production gotcha, not just documentation trivia, and naming it shows depth.

**Q: "Table vs. incremental — how do you decide, concretely?"**
Data volume growth rate, source mutability (pure inserts vs. updates), acceptable rebuild cost/time, and freshness SLA. A good rule stated out loud: start with `table`, move to `incremental` only when full rebuilds become a measurable cost or latency problem — premature incrementalization adds real complexity for no benefit on small tables.
