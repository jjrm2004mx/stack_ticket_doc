## ADDED Requirements

### Requirement: Provider selection
The engine SHALL call the language model through one provider chosen by the variable `PROVIDER` — `oci` (OCI Generative AI, the provider on Rocky) or `ollama` (local development) —, using the adapters of `ticket-agent/src/adapters/` (`base.py`, `oci_adapter.py`, `ollama_adapter.py`) copied as they are, with a note of their origin. Configuration by environment: for `oci`, `OCI_CONFIG_FILE`, `OCI_PROFILE`, `OCI_COMPARTMENT_ID`, `OCI_SERVICE_ENDPOINT`, `OCI_MODEL` (default `openai.gpt-oss-20b`), `OCI_MAX_TOKENS` and `OCI_REASONING_EFFORT`, with the credentials read from the mounted `~/.oci` directory; for `ollama`, `OLLAMA_BASE_URL` and `OLLAMA_LLM_MODEL`. An unknown `PROVIDER`, or a provider whose required variables are missing, SHALL keep the engine from starting. The engine SHALL NOT use `ticket-classification`'s `langchain-api` gateway nor any other service to reach the model. Adding a provider is adding an adapter; no route or contract changes.

#### Scenario: Rocky
- **WHEN** the engine starts with `PROVIDER=oci` and its OCI variables
- **THEN** `/health` reports `provider = "oci"` and the configured model

#### Scenario: Unknown provider
- **WHEN** `PROVIDER=otro`
- **THEN** the engine does not start and logs the accepted values

### Requirement: Structured output, fixed temperature and time limits
`POST /v1/sql` SHALL ask the model for a structured output with the fields of `sql-generation` ("Fixed pipeline with one retry") and `POST /v1/answer` for plain text; both with temperature 0 and without tools. Each model call SHALL be bounded by `MODEL_TIMEOUT_S` (default 20 s) and each request by its budget: `SQL_BUDGET_S` (default 40 s) for `/v1/sql`, which covers the first call and the retry, and `ANSWER_BUDGET_S` (default 18 s) for `/v1/answer`; both stay below the time the application waits (45 s and 20 s for `ticket-management`). A call that fails or times out SHALL NOT be repeated by the engine: the request ends with `503` `MODEL_UNAVAILABLE`. A structured output that cannot be read counts as an invalid first attempt and takes the single retry of the pipeline.

#### Scenario: Model slower than its limit
- **WHEN** the provider has not answered after `MODEL_TIMEOUT_S`
- **THEN** the request ends with `503` `MODEL_UNAVAILABLE` and the call is not repeated

#### Scenario: Budget does not allow the retry
- **WHEN** the first call took 25 s and its SQL is invalid
- **THEN** the engine answers `INVALID` without a second call, because less than one model call is left of the 40 s

### Requirement: Provider, model and call count are reported
Every response of `/v1/sql` and `/v1/answer` SHALL carry `provider`, `model` and `modelCalls` (how many model calls that request made, failed ones included), so the application can record its consumption without the engine storing anything. The engine SHALL expose, per application, counters of model calls and a histogram of their duration by route and provider (`service-operations`); it does not measure tokens in this version.

#### Scenario: Question with a retry
- **WHEN** a `/v1/sql` request needs the retry
- **THEN** its response has `modelCalls = 2`
