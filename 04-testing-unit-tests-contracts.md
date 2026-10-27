# Module 4 — Testing: Data Tests, Unit Tests & Contracts
**Time: 60 minutes | Priority: Core**

## Learn (30 min)

### Data tests (the original dbt testing model)

Data tests run **after** a model is materialized and assert something about the actual data. Two flavors:

- **Generic tests** — reusable, parameterized (`unique`, `not_null`, `accepted_values`, `relationships` are dbt's built-ins; `dbt_utils`/`dbt_expectations` packages add dozens more like `dbt_utils.at_least_one`, `dbt_expectations.expect_column_values_to_be_between`).
  ```yaml
  models:
    - name: dim_customers
      columns:
        - name: customer_id
          data_tests:
            - unique
            - not_null
        - name: status
          data_tests:
            - accepted_values:
                arguments:
                  values: ['active', 'churned', 'trial']
  ```
  Note the modern syntax: test parameters nest under `arguments:` (this replaced the older flat top-level-kwargs style — the `require_generic_test_arguments_property` behavior flag controls which is enforced). Also note the property is `data_tests:`, not the older bare `tests:` key.
- **Singular tests** — a one-off SQL file in `tests/` that returns failing rows (a query returning 0 rows = pass). Used for business-logic assertions that don't generalize into a reusable macro.

Failure handling: `severity: warn` vs `error`, `error_if`/`warn_if` thresholds, and `store_failures: true` to persist failing rows to a table for debugging instead of just a pass/fail count — a genuinely important production feature because "test failed" alone doesn't tell you *which* rows.

### Unit tests (dbt 1.8+) — different purpose, often confused with data tests

Unit tests validate your **SQL logic** against **static, hand-crafted mock input**, before the model ever touches real production data. This is test-driven development applied to SQL — closer to a traditional software unit test than a data-quality check.

```yaml
unit_tests:
  - name: test_is_valid_email_address
    model: dim_customers
    given:
      - input: ref('stg_customers')
        rows:
          - {email: cool@example.com, tld: example.com}
          - {email: badgmail.com, tld: gmail.com}
    expect:
      rows:
        - {email: cool@example.com, is_valid_email: true}
        - {email: badgmail.com, is_valid_email: false}
```

Key distinctions to state clearly in an interview:

| | Data tests | Unit tests |
|---|---|---|
| Runs against | Real materialized data | Static mock rows you define |
| Timing | After the model builds | Can run before building (with `--empty`), tests logic in isolation |
| Catches | Data quality drift in production | Logic bugs (regex, CASE statements, date math, joins) before merge |
| Belongs in | `tests/` dir (singular) or `.yml` columns (generic) | `.yml` under `models/`, referencing the model |

Best practice: unit test **complex logic** (regex parsing, conditional branching, date/window functions, anything with a bug history), not warehouse built-ins like `sum()` or `min()` — the warehouse vendor already tests those. Unit tests are recommended for dev + CI, explicitly **not** production (`dbt test --exclude-resource-type unit_test` in prod jobs) because they add no value against real data and just cost compute. Materialized views, Python models, and recursive SQL are **not** supported.

### Model contracts (governance-adjacent, but fundamentally a testing/reliability concept)

A contract is an **upfront, build-time guarantee** about a model's shape (column names, types, optionally constraints), enforced *before* the model is even allowed to build — different from a data test, which checks content *after* the model already exists.

```yaml
models:
  - name: dim_customers
    config:
      contract:
        enforced: true
    columns:
      - name: customer_id
        data_type: int
        constraints:
          - type: not_null
      - name: customer_name
        data_type: string
```

Mechanics: every column must be declared (no partial contracts), name+type must match exactly (order doesn't matter), and if the compiled query would violate the contract the **model fails to build at all** — this is a stronger guarantee than a data test, which lets bad data land and then flags it. `not_null` is broadly enforced at DDL level; `primary_key`/`foreign_key`/`unique` are metadata-only (documented, not enforced) on most platforms; `check` constraints aren't supported. Contracts detect breaking changes (dropped column, changed type, dropped/renamed contracted model as of 1.9+) and either error (on versioned models) or warn (unversioned).

Use contracts on **public models** — anything another team, another project (dbt Mesh), a BI dashboard, or an external consumer depends on as a stable interface. Don't bother on internal staging/intermediate models that only you touch — contracts add friction (harder rollbacks, more upfront rigor) that isn't worth it until a model's "shape" is actually a promise to someone else.

## Practice (20 min)

1. Write a `data_tests` block for a `fct_orders` model testing: `order_id` unique+not_null, `status` in `('placed','shipped','delivered','cancelled')`, and a `relationships` test that every `customer_id` exists in `dim_customers`.
2. Write one unit test for a hypothetical macro/model that classifies orders as `'large'` if `order_total > 100` else `'small'` — include at least one boundary-value mock row (exactly 100).
3. Write a `contract: enforced: true` block for a two-column public model `dim_customers(customer_id int, email string)`, then describe in one sentence what happens if someone renames `email` to `email_address` without bumping a model version.

## Recall (7 min flashcards)

- Data tests = content, post-build. Unit tests = logic, pre-build, static mocks. Contracts = shape, build-time gate.
- Generic tests are reusable/parameterized; singular tests are one-off SQL files in `tests/`.
- Modern syntax: `data_tests:` key, params nested under `arguments:`.
- `store_failures: true` persists failing rows for debugging.
- Unit tests: `given`/`expect`, `format: dict|csv|sql`, only SQL models, no materialized views, don't run in prod.
- Contracts: all columns must be declared, `not_null` enforced, `primary_key`/`unique`/`foreign_key` metadata-only on most platforms, `check` unsupported, breaking change on a contracted+versioned model = build error.
- Contracts belong on public/cross-team models, not internal staging models.

## Interview (3 min)

**Q: "Your CI catches a failing `not_null` test on a production model. What's your triage process?"**
Check `store_failures` output (or manually query) to see *which* rows are null and since when; check whether an upstream source schema changed; check whether this is a genuine data quality issue vs. a legitimately-nullable business case that means the test itself is wrong; decide warn-vs-error severity going forward if the null rate is expected to be nonzero.

**Q: "Why would you add a unit test *and* a data test for the same model?"**
They catch different failure classes — a unit test proves your CASE/regex logic is correct against known inputs (catches logic bugs before merge, fast, no warehouse cost); a data test proves the *actual* data continues to satisfy invariants over time (catches upstream drift, new edge cases you didn't anticipate). Passing unit tests doesn't guarantee clean production data, and passing data tests today doesn't guarantee your logic handles a new edge case tomorrow.

**Q: "What's the tradeoff of enforcing contracts on every model in your project?"**
Every column rename/type change anywhere becomes a breaking, build-blocking event, which is safety you want on public interfaces but pure friction on internal models mid-iteration — over-applying contracts is a real anti-pattern worth naming unprompted.
