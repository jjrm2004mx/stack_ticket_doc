## Context

Plan version 4 (`docs/tecnico/PLAN_PREGUNTAS_SOBRE_LOS_DATOS.md`) gives this repository all the knowledge and logic of the function "Preguntas sobre los datos" and gives each application only an executor, its own security and its screens. The engine receives a question and returns SQL; the application runs it. Three documents feed this design: `ETAPA_1_REVISION_WREN.md` (what was learned from Wren AI's code and why it is not installed), `NOTAS_DE_DISENO.md` (the reasoned validator and filter) and `ETAPA_1_DEFINICION_DE_VISTAS.md` (the `ticket-management` views the first catalog describes). The cross-repo view of stage 2 is `ETAPA_2_DISENO.md`.

Facts that shape it:
- Wren's strict mode passes schema-qualified names as physical tables, blocks functions by black list, ignores an optional rule that receives no value and, when it cannot re-parse the expanded SQL, lets it through. Each of those is a rule here, inverted.
- `ticket-agent` already calls OCI (`openai.gpt-oss-20b`) with structured output through `oci_adapter.py`; it has never used tool calling with that model.
- The application's database is reachable only from the application (`infra-monitoring`, runbook 11): an engine with a database connection would undo that.
- Rocky has 2 cores; the model takes seconds per call.

## Goals / Non-Goals

**Goals:**
- A question becomes a SQL that can only read the objects of the catalog, always through their filter, or a clear "I need to know more" / "that is out of scope".
- The engine cannot leak or change data even if the model misbehaves: it has no database, and what it returns is checked by a white list and re-read before leaving.
- A second application adopts it with a catalog folder and a key, without code changes.
- Everything testable without a model.

**Non-Goals:**
- Executing SQL, holding credentials or reading any database, in any form.
- Other SQL dialects, a client library, automatic drafting of a catalog (when a second application arrives; they do not change the contract).
- Deciding whether a question is help or data, saving confirmed examples from the portal, token metering.
- Unifying model access across `ticket-agent`, `ticket-classification` and this engine (follow-up outside the plan).

## Decisions

**D1. A service that never touches a database.** Stateless FastAPI; no driver in its dependencies. *Alternative:* let the engine read the application's database to check or run the SQL. Rejected by the owner (2026-11-12) and by the network design: it would need credentials and a route to `ticket-db`.

**D2. Python with SQLGlot; Wren is a source, not a dependency.** The validator and the filter are a few hundred lines over SQLGlot's tree. From Wren (Apache-2.0) the rules of `read_only.py` and `policy.py` are ported with their notice; `wrenai` is not installed (it brings duckdb, pandas, pyarrow and a Rust core for features this engine does not use). *Alternative:* a validator in Java inside the application. Rejected with plan version 3: every application would repeat it.

**D3. White list everywhere, and fail closed.** Unknown node, function, operator, type or literal form → refused. A catalog with an object without filter → not loaded. An output that cannot be re-parsed → `INVALID`. This is the opposite of the three places where Wren fails open.

**D4. The final SQL is rendered from the tree.** The database executes the engine's rendering of what it validated, not the model's text. It removes comments and any difference between "what the parser understood" and "what the text says" as an attack surface. *Trade-off:* SQLGlot's rendering of an exotic construct could differ from PostgreSQL's reading; the white list keeps the constructs few, and the reference questions are executed against a real database in the application's tests.

**D5. Mandatory filter by engine-written CTE.** For each object used, a CTE with the object's name, the physical source and the filter, placed before the model's CTEs. With unqualified names (rule 5) and no shadowing (rule 8), every reference binds to it. *Alternative:* add the filter to each `FROM` occurrence. Rejected: joins, subqueries and set operations multiply the cases; one CTE per object has one. *Alternative:* trust the application's row-level security. Rejected: an application without it must still be safe, and in `ticket-management` it is a second barrier, tested alone in stage 6.

**D6. Unbound parameters, two styles.** The filter's value never travels to the engine. `named` (`:tenant_id`) for clients that bind by name; `qmark` (`?` plus an ordered list) for JDBC, so `ticket-management` binds by position without parsing. *Alternative:* only `named`. Rejected: Spring's named-parameter parser also treats `&name` as a parameter, which would make the application's executor depend on the SQL's content.

**D7. A fixed pipeline, no agent.** Prompt → structured output → validate → rewrite → at most one retry with the reason. No tools, so the model cannot choose a table, a tenant or an action; two calls per question in the normal case (SQL and wording), three with the retry. *Alternative:* an agent with tools (Wren's LangChain toolkit). Rejected: its engine is created with strict mode off and its tools do not carry the tenant.

**D8. The catalog lives here, one folder per application, with a version and a published contract.** The application knows only the names it needs for its permissions and the version its database implements. `MAJOR` ties the catalog to the application's views; the request's `catalogVersion` turns a silent drift into a `409`, and `contract.json` lets the application test its views without the engine running.

**D9. The caller's objects and the client's values come in each request.** `allowedObjects` narrows the prompt and the validator (rule 7); the objects left out are named to the model as not available, so the answer is a clear `OUT_OF_SCOPE` and not a guess. The values (names of states, areas, people) are the client's vocabulary; they are data in the prompt, bounded by the catalog's `maxItems`, and are never stored or logged.

**D10. `;` is refused even inside a literal.** The application's driver may execute several statements from one text; refusing every `;` lets the application keep a guard that needs no parser. A question whose literal would need a `;` fails as `INVALID`: accepted.

**D11. `LIMIT rowLimit + 1`.** The engine bounds the rows and leaves the application the extra row to know the result was cut; the application also bounds at its driver.

**D12. One key per application; limits per application and per opaque usage key.** The key decides which catalog may be used. The usage key lets an application spread its quota among its own clients without the engine knowing what a client is; it is never a metric label (cardinality, and it would reveal the application's clients).

**D13. Model adapters copied from `ticket-agent`.** As they are, with `PROVIDER` = `oci` or `ollama`. *Alternative:* a shared library or the classifier's gateway. Not now: the gateway returns free text on an internal network, and a shared library touches two repositories in use; it is a follow-up of the plan (§11).

**D14. Logs and metrics carry no content.** No question, value, SQL, row or answer in logs at any level; metrics by application only. The application's own log (`insights_questions` in `ticket-management`) is where a question can be traced, joined by `requestId`.

**D15. Time budgets inside the application's.** `MODEL_TIMEOUT_S` 20, `SQL_BUDGET_S` 40 (under the 45 s the application waits), `ANSWER_BUDGET_S` 18 (under 20). Nothing is repeated after a time-out; the retry of the pipeline exists only for an invalid SQL and only if a call still fits.

**D16. Where it runs.** `~/stack_ticket/text-to-sql`, container `text-to-sql`, `127.0.0.1:8003`, only on `ticket-management-network`, started by `infra-monitoring/startup.sh` as a sixth project. *Alternative:* `~/stack_agent` next to `ticket-agent`. Rejected: that stack is started by `support-console`, and the plan gives the orchestration and monitoring of the engine to `infra-monitoring`.

**D17. Layout.** `src/api` (routes, auth, limiter, request id), `src/catalog` (loader, validation, contract), `src/sql` (validator, filter, renderer), `src/pipeline` (prompt, structured output, retry), `src/adapters` (copied), `catalogs/ticket-management/`, `tests/`, `examples/executor/`, `adoption-tests/`, `docs/`.

## Risks / Trade-offs

- [The validator misses a construct that reads outside the catalog] → White list; negative cases as tests; the application's database is a second barrier and is tested without the engine.
- [The filter misses a reference] → One mechanism (CTE by name) instead of per-occurrence rewriting; the re-parse check; stage 6 runs the reference questions with row-level security off.
- [SQLGlot renders something PostgreSQL reads differently] → Few constructs allowed; reference questions executed for real in the application's tests; SQLGlot pinned.
- [The model writes wrong SQL that is valid] → Accuracy is measured in stage 6 against expected results; examples and rules are the levers; the table shown to the user is the truth.
- [`openai.gpt-oss-20b` is slow or weak at SQL] → Measured in stage 6 before building the portal; the provider can change without touching the contract.
- [Prompt size with a large client (five lists of up to 200 names)] → `maxItems` per set; measured in stage 6.
- [Rate counters in memory, single instance] → Enough for one server; a restart resets them.
- [Catalog and application drift] → Version handshake and contract file.
- [Names of people and ticket titles reach the provider] → Owner decision 3; no e-mails, requester data or free text; nothing is stored or logged here.

## Migration Plan

1. **Stage 4.** Create the repository with its template and this change as `openspec/changes/motor-inicial/` (moved from `stack_ticket_doc/text-to-sql/openspec/`). Build the catalog loader, the `ticket-management` catalog, the validator, the filter, the pipeline with a fake provider, the adapters, keys and limits, the four routes, tests without a model, runbooks, container. Run locally and on Rocky (with the owner's permission; clean old images first).
2. **Stage 6.** With `ticket-management` deployed off: negative cases with the model, the filter alone, a sample of the reference questions, time per step.
3. **Stage 9.** Orchestration and monitoring (`infra-monitoring`), adoption guide, example executor and test package, full run, archive with the siblings.

**Rollback.** The engine holds nothing: stopping and removing the container and its key in `ticket-management` removes it. Until stage 5 nothing depends on it.

## Open Questions

- The repository's final name (`text-to-sql` is provisional) — owner.
- Whether `string_agg` and the percentile functions stay in the white list after measuring what the questions really need (stage 6).
