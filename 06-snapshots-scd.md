# Module 6 — Snapshots & Slowly Changing Dimensions
**Time: 30 minutes | Priority: Core**

## Learn (15 min)

### Why snapshots exist

Your source system usually only shows you the *current* state of a row — if a customer's `status` changes from `trial` to `active`, the source table just updates in place; history is lost. Snapshots capture that change over time by turning mutable source data into an append-only **Type 2 Slowly Changing Dimension (SCD)** table, so you can later answer "what was this customer's status as of March?"

### Syntax changed in dbt 1.9 — this is a real "what's new" interview topic

- **Legacy (dbt ≤1.8):** a `{% snapshot %}...{% endsnapshot %}` Jinja block inside a `.sql` file under `snapshots/`.
- **Current (dbt 1.9+):** plain **YAML**, no Jinja block required:

```yaml
snapshots:
  - name: orders_snapshot
    relation: source('jaffle_shop', 'orders')
    config:
      schema: snapshots
      unique_key: id
      strategy: timestamp
      updated_at: updated_at
      dbt_valid_to_current: "to_date('9999-12-31')"
```

Being able to say "this moved from a Jinja block to declarative YAML in 1.9" is exactly the kind of dated, specific knowledge that signals you keep up with the docs rather than working from a 3-year-old mental model.

### Strategies

- **`timestamp`** — uses an `updated_at` column; preferred when available because it's resilient to schema changes (new/dropped columns don't require touching the snapshot config) and cheap to evaluate (one column comparison).
- **`check`** — compares specific columns (`check_cols: [status, plan]` or `check_cols: all`) for any source system without a reliable updated-at column; more fragile to schema drift and more expensive (compares N columns every run).

### Meta-columns every snapshot produces

`dbt_valid_from`, `dbt_valid_to` (nullable/`dbt_valid_to_current` for the current row), `dbt_scd_id` (unique version id), `dbt_updated_at`. This is the literal SCD Type 2 shape — a query like `where dbt_valid_from <= '2026-03-01' and (dbt_valid_to > '2026-03-01' or dbt_valid_to is null)` reconstructs point-in-time state.

### Hard deletes (1.9+ `hard_deletes` config, replacing `invalidate_hard_deletes: true`)

If a row is physically deleted from the source, does your snapshot notice? Three options:
- `ignore` (default) — deleted source rows are simply never updated again in the snapshot; stale but harmless if you don't care about deletions.
- `invalidate` — sets `dbt_valid_to` on the now-missing row (legacy `invalidate_hard_deletes: true` behavior).
- `new_record` — adds an explicit new snapshot row with `dbt_is_deleted: true`, so deletion itself becomes a queryable event in history, not just an inferred absence.

## Practice (10 min)

Write a current-syntax YAML snapshot for `source('crm', 'deals')` using the `timestamp` strategy on an `updated_at` column, `unique_key: deal_id`, `hard_deletes: new_record`. Then write the point-in-time reconstruction query: "what was `deal_id = 42`'s `stage` on 2026-01-15?"

## Recall (3 min flashcards)

- Snapshots = Type 2 SCD, capture change over time from mutable source tables.
- 1.9+ syntax: YAML, `snapshots:` key with `relation:` and `config:` — no more Jinja block (legacy).
- `timestamp` strategy (uses `updated_at`, resilient, cheap) vs `check` strategy (`check_cols`, no updated_at needed, more fragile/costly).
- Meta-columns: `dbt_valid_from`, `dbt_valid_to`, `dbt_scd_id`, `dbt_updated_at`.
- `hard_deletes: ignore | invalidate | new_record` — replaces old `invalidate_hard_deletes: true` boolean.
- Snapshots should run on a schedule (hourly-to-daily) — if you only run them occasionally, you lose any state changes that happened and reverted between runs.

## Interview (2 min)

**Q: "A dimension changes 40 times a day. Would you snapshot it?"**
Probably not at that granularity via scheduled snapshots — you'd either increase the snapshot schedule to match (real infra cost) or reconsider whether you actually need full history at that resolution vs. just current state, or capture the change stream at source (CDC) instead of inferring it from periodic snapshots, which can *miss* rapid intermediate changes entirely — a genuinely important limitation to name: **snapshots only see state at the moment they run; changes that happen and revert between runs are invisible.**

**Q: "Snapshot vs. incremental model with a `check` strategy of your own — what's actually different?"**
Snapshots are purpose-built with the SCD meta-columns, hard-delete handling, and are meant for *source* mutation tracking; a hand-rolled incremental doing similar comparison work reinvents that machinery without the built-in guarantees, and dbt explicitly recommends snapshotting raw/source-like data, not already-transformed marts (snapshotting a heavily-transformed model couples your history capture to logic that might itself change).

**Common mistake:** snapshotting a *mart* instead of something close to source — if you later fix a bug in the transformation logic feeding that mart, your snapshot history now contains the bug baked in permanently, since snapshots are append-only and can't be retroactively corrected the way a `table` materialization can just be rebuilt.
