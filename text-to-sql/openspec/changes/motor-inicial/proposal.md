## Why

The owner approved (plan version 4, 2026-11-12) the function "Preguntas sobre los datos": a user asks in natural language about the data of its company and receives the answer with the table behind it. The knowledge of that function — what each object and column means, how the SQL is written and checked, which filter is mandatory, how the answer is worded — must not live inside each application. This repository is that knowledge as a service: an engine that turns a question into validated, filtered SQL and rows into text, and that **never enters a database**. The application that owns the data runs the query with its own permissions (the way Oracle Select AI and Snowflake Cortex Analyst work). `ticket-management` is the first application; a second one adopts the engine by delivering its catalog and mounting a small executor.

Installing Wren AI was discarded after reading its code (`ETAPA_1_REVISION_WREN.md`): its strict mode lets schema-qualified names through, an optional rule fails open and its connector cannot carry a per-request tenant. Its ideas and some of its rules (Apache-2.0) are ported instead.

## What Changes

New repository `text-to-sql` (provisional name), a stateless FastAPI service:

- **`POST /v1/sql`** — application, question, the client's values and the objects the caller may use → SQL validated and rewritten with the mandatory filters and unbound parameters, or a request for clarification, or "out of scope". One internal retry with the validator's error. It accepts, optionally, the error the database returned and the previous question.
- **`POST /v1/answer`** — question plus rows → the text of the answer.
- **`POST /v1/validate`** — validates a SQL against a catalog and returns the final SQL with its filters, without calling a model. It is what the tests and an adopting application use.
- **`GET /v1/catalogs/{application}`** — the objects, columns and types the engine expects, so the application can compare them with its database in an automatic test.
- **Catalogs** — one folder per application, with a published and versioned format: objects, columns, meanings, synonyms, relationships, rules, question→SQL examples and the mandatory filter of every object. The first catalog is `ticket-management` (the eight `insights` views).
- **Validator and mandatory filter** — Python with SQLGlot: one read-only statement, only objects of the catalog, no schema-qualified names, a white list of functions, columns that exist, no parameters written by the model; then every object the query names is replaced by an engine-written CTE that applies its filter. It fails closed.
- **Model providers** — `PROVIDER` = `oci` or `ollama`, with the adapters copied as they are from `ticket-agent`; every response reports provider and model.
- **Keys and quotas** — one key per application; rate limits per application and per an opaque usage key the application sends.
- **Operation** — `/health`, `/metrics`, request id, logs without questions or rows; one light container on the shared network, its port only on `127.0.0.1`; no database, no data of any client, no credentials of other databases.
- **Adoption kit** — guide for a new application, an example executor and a test package the application runs against its own database.
- Repository standards from the first commit: `CLAUDE.md`, `openspec/`, skills in `.claude/skills`, runbooks 01 to 03.

## Capabilities

### New Capabilities
- `sql-generation`: `POST /v1/sql`, its request and response, the fixed pipeline (prompt → model → validation → rewrite → one retry) and its statuses.
- `sql-validation`: the rules a SQL must pass, their reason codes and `POST /v1/validate`.
- `mandatory-filter`: the filter every object declares, how the engine applies it and how its parameters are returned unbound.
- `answer-writing`: `POST /v1/answer`.
- `application-catalog`: the catalog format, its validation at start-up, its version, the published contract, `GET /v1/catalogs/{application}` and the content of the `ticket-management` catalog.
- `model-providers`: provider selection, adapters, structured output, time limits and what is reported.
- `application-access`: keys per application and rate limits.
- `service-operations`: health, metrics, request id, logs, container, statelessness and repository standards.
- `adoption-kit`: the guide, the example executor and the test package.

### Modified Capabilities
(none — the repository is new)

## Impact

- New repository and container (`text-to-sql`, port `127.0.0.1:8003`, network `ticket-management-network` only). Until the repository exists, this change lives in `stack_ticket_doc/text-to-sql/openspec/` and moves to the repository with its first commit (stage 4).
- Dependencies: Python 3.11, FastAPI, SQLGlot (pinned), `langchain-oci` and `langchain-ollama` as in `ticket-agent`, `prometheus-fastapi-instrumentator` (pinned). Rules ported from Wren AI (`read_only.py`, `policy.py`) keep their Apache-2.0 notice in `NOTICE`.
- Siblings: `ticket-management` (`preguntas-sobre-los-datos`: executor, views, permissions, switch, log) and `infra-monitoring` (`motor-en-el-ecosistema`: start-up, checks and metrics). `ticket-agent` is only the source of the copied adapters; it is not changed.
- OCI: two model calls per question, three when the validator asks for a retry. In the stages only samples are run; the full run goes at the end.
- Rocky: one light container without volumes. Before deploying it, clean old images (13.8 GB recoverable, measured 2026-11-11).
