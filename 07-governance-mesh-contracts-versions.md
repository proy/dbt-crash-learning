# Module 7 — Governance at Scale: dbt Mesh, Model Versions, Groups & Access
**Time: 60 minutes | Priority: Advanced (this is where senior/staff interviews differentiate candidates)**

## Learn (30 min)

### The problem dbt Mesh solves

Once an organization has hundreds of models and multiple teams, a single monolithic dbt project breaks down: parse/compile time balloons, teams step on each other's models, ownership is unclear, and a change anywhere can silently break something owned by another team. **dbt Mesh** isn't one feature — it's a pattern built from several converging governance features that let you split a project into multiple, independently-deployed dbt projects that can still safely reference each other. Requires **Enterprise or Enterprise+** on dbt Platform for cross-project `ref()`.

### Groups

A `group` clusters models under a named owner, typically mapping to a team:

```yaml
groups:
  - name: finance
    owner:
      name: Finance Team
      email: finance-data@company.com

models:
  - name: fct_revenue
    config:
      group: finance
```

A model in a group can only be referenced by other models in the *same* group **unless** it's explicitly exposed via an access modifier — groups are the ownership boundary that access modifiers then punch controlled holes through.

### Access modifiers

Every model has an `access` setting controlling who can `ref()` it across group/project boundaries:

- `private` — only referenceable within its own group.
- `protected` (default) — referenceable anywhere within the *same* dbt project, but not from other projects.
- `public` — referenceable from other groups **and** other dbt projects (this is the one that matters for cross-project Mesh references).

```yaml
models:
  - name: dim_customers
    config:
      access: public
      group: core
```

The mental model to say out loud: **groups control internal ownership boundaries; `access` controls the blast radius of who's allowed to build a dependency on a model at all** — treating your public models as a versioned API surface, not "just SQL files."

### Model contracts as the API layer (tie-back to Module 4)

Any model you mark `public` and expect other teams/projects to depend on should almost always also have `contract: enforced: true` — access controls *who* can depend on it, contracts control *what breaks* if you change it. Together they're how dbt Mesh treats models as genuine internal APIs.

### Model versions

When a public/contracted model's shape needs to change in a breaking way, you don't just edit it — you version it, so existing downstream consumers keep working against the old version while new consumers adopt the new one:

```yaml
models:
  - name: dim_customers
    latest_version: 2
    versions:
      - v: 1
        defined_in: dim_customers_v1
      - v: 2
        defined_in: dim_customers_v2
```

Downstream `ref('dim_customers')` resolves to `latest_version` unless a consumer explicitly pins `ref('dim_customers', version=1)`. This gives you a deprecation window instead of a flag-day breaking change — a genuinely important production pattern to be able to describe, because "how do you evolve a schema without breaking fifty downstream dashboards" is a real senior-level question.

### Cross-project references

With Mesh, `{{ ref('other_project', 'public_model') }}` works across independently-deployed projects, using the *public* model's contract as the interface — the referencing project doesn't need the upstream project's source code, only its published public/contracted models via the Catalog's cross-project metadata. This is what actually enables org-scale decomposition without every team needing repo access to every other team's project.

### When to adopt Mesh (and when not to)

Adopt when you're seeing: parse/compile times degrading noticeably, teams needing genuinely independent deploy cadences, or real security/governance requirements for isolation between domains. dbt Labs' own explicit guidance: **don't** architect a greenfield project as multi-project Mesh from day one — start monolithic, add structure incrementally as scale actually demands it. Citing this "don't over-engineer early" nuance unprompted is a strong signal in an interview, because the naive answer is "always use Mesh for governance," which is wrong.

## Practice (20 min)

1. Design a two-project split for an org with a `core` project (owns `dim_customers`, `dim_products`) and a `finance` project (owns `fct_revenue`, which needs `dim_customers`). Write the `access`/`group`/`contract` config for `dim_customers` that makes this legal, and the cross-project `ref()` call `fct_revenue` would use.
2. Write a `versions:` block bumping `dim_customers` from v1 to v2 where v2 renames `email` → `email_address`, and describe how you'd communicate/migrate downstream consumers before removing v1.
3. One sentence: why would you mark an intermediate model `private` even within your own team's project?

## Recall (7 min flashcards)

- dbt Mesh = pattern, not a single feature: cross-project refs + Catalog + groups + access + contracts + versions.
- Requires Enterprise/Enterprise+ for cross-project `ref()`.
- Groups = ownership boundary (team). Access = `private` (same group only) / `protected` (default, same project) / `public` (cross-group, cross-project).
- Public models should almost always be contracted too — access controls who can depend, contracts control what breaks.
- Model versions give a deprecation window: `latest_version`, `ref(name, version=N)` to pin.
- Don't build multi-project Mesh on day one — add it as scale actually demands.

## Interview (3 min)

**Q: "Two teams both need a `customer_ltv` calculation and each built their own, slightly different. How does dbt's governance tooling prevent this going forward?"**
A `public`, `contract`-enforced model owned by a clear `group`, published from a project both teams can cross-reference, turns "everyone reinvents this metric" into "everyone refs the one governed model" — and if it's genuinely a metric (not raw data), the Semantic Layer (Module 8) is arguably an even better home for it than a plain model.

**Q: "What actually breaks if you skip model versioning and just edit a public contracted model's columns directly?"**
The contract catches it at build time (assuming `enforced: true`) and the build **errors** rather than silently shipping broken data — but that's still a hard stop for the owning team with no graceful migration path for consumers; versioning is what turns a hard stop into a managed deprecation.

**Common mistake:** treating `access: public` as a purely technical toggle instead of an organizational commitment — once a model is public, downstream teams build on it as an API, and breaking it without process (grep who's calling it, communicate, version, deprecate) causes real incident-level pain, which is exactly why interviewers ask about this.
