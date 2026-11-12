## ADDED Requirements

### Requirement: Stateless service without database access
The engine SHALL keep no state between requests other than its in-memory rate counters: it has no database, no volume and no cache of questions, values, SQL, rows or answers, and it stores no data of any client. It SHALL NOT hold credentials, hosts or drivers of any application's database, and its code SHALL contain no database client: the SQL it returns is always executed by the application. A restart loses nothing.

#### Scenario: Looking for a way to the application's database
- **WHEN** the engine's dependencies, configuration and compose file are inspected
- **THEN** there is no database driver, no connection string and no network shared with a database

#### Scenario: Restart
- **WHEN** the container is recreated
- **THEN** it serves the same catalogs and no question, value or answer of before exists anywhere in it

### Requirement: Container, port and network
The repository SHALL ship a `Dockerfile` (Python 3.11 slim), a `docker-compose.yml` and a `start.sh` like the other repositories of the ecosystem (create `.env` from `.env.example` when missing, create `ticket-management-network` without subnet when missing, `podman-compose down`, then `up -d`). The single service `text-to-sql` runs as container `text-to-sql`, listens on port 8003 published only as `127.0.0.1:8003:8003`, joins only the external network `ticket-management-network`, mounts `${HOME}/.oci` read-only for the OCI credentials, logs through the `journald` driver with tag `text-to-sql`, uses `restart: unless-stopped` and declares a health check on `GET /health`. On the server the repository lives in `~/stack_ticket/text-to-sql` and is started, stopped, built and checked by `infra-monitoring` (`motor-en-el-ecosistema`).

#### Scenario: Port from the LAN
- **WHEN** a machine on the LAN connects to port 8003 of the server
- **THEN** the connection is refused, while `localhost:8003` on the server answers

#### Scenario: The engine tries a database
- **WHEN** the container resolves `ticket-db`
- **THEN** the name does not resolve, because the engine is not on that network

### Requirement: Health, metrics and request id
`GET /health` SHALL answer `200` `{"status": "ok", "provider", "model", "catalogs": [{application, version, valid}]}` without calling the model. `GET /metrics` SHALL expose the default metrics of `prometheus-fastapi-instrumentator` (version pinned) plus, labelled by `application` and never by `usageKey`: requests by route and outcome (`SQL`, `CLARIFY`, `OUT_OF_SCOPE`, `INVALID`, error codes), validation failures by reason code, model calls and their duration by provider, and rate-limited requests. Every request SHALL carry a request id — the incoming `X-Request-Id` when present, otherwise a new one — returned in the header `X-Request-Id` and in the body, and written in every log line of that request.

#### Scenario: Health
- **WHEN** `GET /health` is called with the `ticket-management` catalog loaded
- **THEN** the body lists it with its version and `valid = true`, and no model call is made

#### Scenario: Tracing a question
- **WHEN** `ticket-management` records the `requestId` of a response
- **THEN** the same id finds every log line of that request in the engine

### Requirement: Logs without questions or rows
The engine SHALL log in JSON to standard output, one line per event, with the request id, the application, the route, the outcome, reason codes, durations and counts. It SHALL never log the text of a question, the application's values, a SQL, a row, an answer, a `usageKey` or a key — at any log level. The prompt and the model's raw output are never written to disk or logs.

#### Scenario: Question with names of people
- **WHEN** a question mentions a person and the values carry the names of the client's agents
- **THEN** the log of that request has neither the question nor any name

#### Scenario: Invalid SQL
- **WHEN** the validator refuses a SQL
- **THEN** the log has the reason code and not the SQL

### Requirement: Tests without a model
The repository's test suite SHALL run without a model and without a network: the validator and the filter with every negative case of `stack_ticket_doc/text-to-sql/docs/tecnico/ETAPA_2_DISENO.md` §8, the catalog validation, the contract file, the pipeline with a fake provider (first answer invalid, retry valid; two invalid; clarification; out of scope; provider error and time-out), the keys and the rate limits. The questions with a real model are a separate, explicit run (a sample in the stages, the full set only at the end).

#### Scenario: Continuous run
- **WHEN** the suite runs on a machine without OCI credentials
- **THEN** it passes, and no test calls a provider

### Requirement: Repository standards
From its first commit the repository SHALL have `CLAUDE.md` (project standards and branch and commit conventions of the ecosystem), `openspec/` with these specs, the OpenSpec skills in `.claude/skills`, `README.md`, `.env.example` without secrets, `NOTICE` with the Apache-2.0 attribution of the rules ported from Wren AI, and `docs/runbooks/` with `01` (for beginners), `02` (technical architecture) and `03` (operational guide: start, stop, rotate a key, change the provider, add a catalog, what to do when the model does not answer), copied to `stack_ticket_doc/text-to-sql/` when they change.

#### Scenario: New person in the repository
- **WHEN** someone opens the repository for the first time
- **THEN** runbook 01 explains what the engine does and does not do, and `CLAUDE.md` points to `openspec/specs/` as the source of truth
