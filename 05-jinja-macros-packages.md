# Module 5 — Jinja, Macros & Packages
**Time: 45 minutes | Priority: Core**

## Learn (22 min)

### Jinja fundamentals as dbt actually uses them

dbt SQL files are Jinja templates rendered into raw SQL before execution. The three building blocks:

- `{{ }}` — expression, outputs a value (`{{ ref('stg_orders') }}` → the fully-qualified relation name).
- `{% %}` — statement, control flow, no direct output (`{% if %}`, `{% for %}`, `{% set %}`).
- `{# #}` — comment, stripped entirely at compile time.

`ref()` and `source()` are not string interpolation — they're how dbt builds the DAG. Never hardcode a schema/table name where `ref()`/`source()` belongs; doing so breaks dependency tracking, environment promotion (dev/staging/prod schema swapping), and `state:modified` selection.

### Macros

Macros are Jinja functions, defined with `{% macro %}...{% endmacro %}`, callable from any model:

```sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100)::numeric(16,2)
{% endmacro %}

-- usage in a model:
select {{ cents_to_dollars('amount_cents') }} as amount_dollars from {{ ref('stg_payments') }}
```

Macros are how you: write custom generic tests (a macro named `test_<name>` becomes a reusable test), DRY up repeated cross-database SQL (e.g. `dbt_utils.date_trunc()` normalizes syntax differences between Snowflake/BigQuery/Postgres), and implement hooks (`pre-hook`/`post-hook`, `on-run-start`/`on-run-end` — e.g. granting permissions after a table rebuild).

**Custom generic test example:**
```sql
{% test positive_value(model, column_name) %}
select * from {{ model }} where {{ column_name }} < 0
{% endtest %}
```
Used in YAML exactly like a built-in: `data_tests: [positive_value]`.

### Packages

`packages.yml` (or `dependencies.yml`, which additionally supports pulling in other **dbt projects** for cross-project `ref()` under dbt Mesh — see Module 7) pulls in shared macro/model libraries:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
```

The essential packages to know by name for interviews:
- **`dbt_utils`** — cross-database compatible macros (`date_trunc`, `surrogate_key`, `pivot`, `unpivot`) and generic tests (`unique_combination_of_columns`, `at_least_one`, `not_constant`).
- **`dbt_expectations`** — Great-Expectations-style richer data tests (distribution checks, regex, `between`).
- **`codegen`** — generates YAML/SQL boilerplate (e.g. `dbt_codegen.generate_source()` from an information_schema scan) — a real time-saver worth mentioning as a productivity habit.
- **`dbt-labs/audit_helper`** — compares two relations row-by-row, invaluable for validating a refactor produces identical output to the model it's replacing.
- **Native private packages** (GA as of ~2026) — hosting internal packages without a public git repo, relevant if asked about sharing macros across teams privately.

`dbt deps` installs everything into `dbt_packages/` (renamed from the older `dbt_modules/` default — a small but real deprecated-path gotcha).

## Practice (16 min)

1. Write a macro `safe_divide(numerator, denominator)` that returns `null` instead of erroring on divide-by-zero, using a Jinja `{% macro %}` wrapping a `case when ... then null else ... end`.
2. Convert it into a **custom generic test** named `test_no_divide_by_zero_error` — think through what the test SQL needs to select to represent a *failure* (rows, not booleans).
3. Write the `packages.yml` entry to install `dbt_utils` pinned to the 1.x line, and name one `dbt_utils` macro and one `dbt_utils` generic test you'd actually use on a `fct_orders` model.

## Recall (5 min flashcards)

- `{{ }}` expression, `{% %}` statement, `{# #}` comment.
- Always use `ref()`/`source()`, never hardcode schema.table — breaks DAG + environment promotion + state selection.
- Macro = Jinja function; a macro named `test_x` becomes a reusable generic test callable as `data_tests: [x]`.
- `dbt_utils` = cross-db macros + generic tests. `dbt_expectations` = richer data-quality tests. `codegen` = YAML/boilerplate generation. `audit_helper` = row-level diff between two relations.
- Packages live in `dbt_packages/` (current), not `dbt_modules/` (deprecated default).
- Hooks: `pre-hook`/`post-hook` (model-scoped), `on-run-start`/`on-run-end` (project-scoped).

## Interview (2 min)

**Q: "You see the same 12-line CASE statement copy-pasted across 6 models. What do you do?"**
Extract it into a macro (or, if it's really a metric definition rather than raw SQL reuse, consider whether it belongs in the Semantic Layer instead — Module 8). Name the DRY benefit but also the real cost: macros can make SQL harder to read/debug for less-Jinja-fluent teammates if overused, so reach for them for genuine repetition, not reflexively.

**Q: "What's a concrete case where over-using macros/Jinja hurt a project?"**
Deeply nested `{% if %}`/macro-of-macros logic that makes `dbt compile` output unreadable and debugging painful — a senior answer names "compiled SQL should still be legible" as a design constraint, not just "macros are good, use them everywhere."

**Common mistake:** hardcoding `from raw.orders` instead of `{{ source('raw','orders') }}` — breaks lineage, freshness checks, and environment overrides silently; this is one of the most common junior-code-review findings and a strong "mistake I've caught" interview anecdote.
