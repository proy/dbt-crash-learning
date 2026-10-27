# Module 10 — The Modern dbt Platform: Fusion Engine, dbt Core v2, Studio IDE & AI Tooling
**Time: 45 minutes | Priority: Advanced (this is the "have you kept up with 2025-2026" module)**

## Learn (25 min)

### Recap and deepen: the engine transition

From Module 1: dbt Core v1.x (Python) is stable and still maintained; dbt Fusion (Rust, from acquired SDF Labs technology) is the rewritten engine; dbt Core v2 (Rust/ADBC, beta in 2026) is the open-source foundation Fusion is now built on, ending the "two separate codebases" era. What to add here:

- **Why Rust specifically**: predictable performance without a garbage collector, strong static typing that suits building an actual SQL type-checker/compiler, and a mature ecosystem (Arrow/ADBC) for columnar data interchange — this is the kind of "why this tech choice" reasoning interviewers sometimes probe for engineering judgment, not just product trivia.
- **ADBC (Arrow Database Connectivity)** replaces the old Python DB-API-style adapter pattern — adapters become thinner, and data moves between dbt and the warehouse as Arrow columnar batches rather than row-by-row Python objects, which is both faster and what enables Parquet-based artifacts (vs. the old `manifest.json`/`run_results.json`, still supported but now with high-performance Parquet alternatives queryable directly via DuckDB).
- **SQL comprehension in practice**: because Fusion actually parses and type-checks SQL rather than just templating it, it can catch errors like summing a datetime column, offer function-aware autocomplete, show live compiled-SQL/CTE previews, and do column-level (not just table-level) lineage — none of which a pure text-templating engine (classic dbt Core) can do without literally sending the query to the warehouse first.

### dbt Lint (beta) and dbt Docs v2 (preview)

- **dbt Lint** — a built-in SQL linter reported to be dramatically faster than SQLFluff (the long-standing third-party tool teams bolted on for style enforcement) while staying SQLFluff-config-compatible, reducing the need for a separate linting toolchain in CI.
- **dbt Docs v2** — a next-generation catalog/docs experience: compact binary index for faster loads, a REST API (not just the static site), and column-level lineage as a first-class citizen rather than a table-level graph only.

### Studio IDE, Canvas, Catalog — rebrand plus real capability changes

Don't just memorize the renames (Module 1's table) — know what changed functionally:

- **Studio IDE** (was Cloud IDE) — now runs on the Fusion engine, so it inherits live error detection, near-instant single-file recompilation, and richer autocomplete; recent additions include a dedicated **Commands** tab (history/logs) and a **Problems** tab surfacing live Fusion errors, plus command-palette-style search/navigation.
- **Canvas** (was Visual Editor, GA ~2025) — drag-and-drop model building without writing SQL directly; positioned for less SQL-fluent contributors or rapid prototyping, generates real dbt model files under the hood (not a separate black-box format).
- **Catalog** (was Explorer) — global, cross-project search/navigation (a real capability add, not just a rename), plus the ability to ingest metadata about *non-dbt* warehouse resources so lineage isn't limited to what dbt itself manages.

### AI tooling — know what each one actually does, this is commonly conflated

- **dbt Copilot** — chat-based assistant inside Studio IDE for generating SQL/YAML from natural-language prompts, and can scaffold things like a `dbt-styleguide.md` for a project.
- **dbt Wizard** — an AI assistant available in the IDE (GA on Enterprise), with bring-your-own-Azure-OpenAI-key support — positioned similarly to Copilot but worth naming as a distinct, separately-shipped feature if asked to differentiate dbt's various AI surfaces (the exact Copilot/Wizard boundary has shifted across 2025 releases, so the safe interview answer is "dbt Platform has multiple AI-assisted authoring surfaces in the IDE, generating SQL/YAML/docs from natural language" rather than overclaiming precision on which name owns which exact feature today).
- **Developer Agent** (preview) — a more autonomous mode: building or refactoring models from natural-language instructions rather than just single-shot generation/autocomplete — the "does more of the work end-to-end" tier above chat-based Copilot-style assistance.
- **dbt MCP server** (GA) — dbt exposes itself as a **Model Context Protocol** server, meaning external AI agents/tools (not just dbt's own UI) can query dbt metadata and, with the Fusion-exclusive tools, things like `compile_sql` and `get_column_lineage` programmatically. This is the interoperability angle: dbt becoming something *other* AI tooling can plug into, not just a tool with its own bolted-on AI features.

### Iceberg and open table format support

Apache Iceberg catalog integration (Snowflake, BigQuery) reflects the broader industry move toward open, engine-agnostic table formats — relevant if asked about lakehouse architecture trends; dbt's role stays the same (defining transformation logic) but materializations can now target Iceberg-backed tables via catalog-linked databases on supporting platforms.

## Practice (12 min)

Write a short (4-6 sentence) answer, as if speaking it in an interview, to: *"I see you have dbt Core experience — are you familiar with what's changed in the last year or so?"* Your answer should name at minimum: the Fusion engine and why it was built (performance + SQL comprehension), the dbt Core v2 convergence, at least one rebrand (Cloud→Platform or Explorer→Catalog), and one AI-adjacent capability (Copilot, Developer Agent, or MCP server) — in one coherent narrative, not a bullet-dump.

## Recall (5 min flashcards)

- Fusion = Rust, from SDF Labs tech, true SQL compiler (types, column lineage, live errors), free CLI.
- Core v2 = Rust/ADBC open-source foundation (Apache 2.0), beta 2026, converging with Fusion.
- ADBC replaces old adapter pattern; Arrow columnar transfer; Parquet artifacts alongside/replacing JSON.
- dbt Lint (beta) = fast built-in linter, SQLFluff-config-compatible. dbt Docs v2 (preview) = binary index + REST API + column lineage.
- Studio IDE (was Cloud IDE, Fusion-powered), Canvas (was Visual Editor, no-code, GA), Catalog (was Explorer, cross-project search + non-dbt metadata ingestion).
- AI surfaces: Copilot (chat, in-IDE gen), Wizard (IDE AI assistant, BYO Azure key), Developer Agent (preview, more autonomous build/refactor), MCP server (GA — dbt as a tool *other* AI agents can call).
- Iceberg catalog integration on Snowflake/BigQuery reflects the open-table-format trend.

## Interview (3 min)

**Q: "If a candidate only knows 'classic' dbt Core, what's the single most important thing to update their mental model with?"** (a question you might get turned around on you, or should self-assess against)
That dbt is no longer just "a Python templating tool that ships SQL to the warehouse" — it's converging toward a genuine Rust-based SQL compiler with type-checking and column-level lineage, and the product itself has split into a free/open engine layer (Core v2 + Fusion) plus a managed governance/AI/collaboration layer (dbt Platform) that's been substantially rebranded and re-featured since 2025.

**Q: "What's a risk of adopting Fusion or Core v2 early, while they're in beta?"**
Beta software means potential breaking changes before GA, possibly incomplete adapter parity across all warehouses (new engines typically roll out adapter support incrementally, as Fusion did starting with Snowflake before expanding), and less community/Stack-Overflow-era troubleshooting content versus the mature Core v1.x line — a real production team should weigh migration timing against project criticality, not just chase newest-is-best.

**Common mistake:** presenting *every* 2025-2026 feature as universally GA — several of these (dbt Lint, dbt Docs v2, Developer Agent, dbt Core v2 itself) are explicitly beta/preview as of this course's research date; overclaiming GA status on a beta feature in an interview is the exact "outdated or overconfident mental model" failure this course is trying to prevent. When in doubt, say "I believe that's currently in beta/preview — worth confirming against the current docs" rather than guessing confidently.
