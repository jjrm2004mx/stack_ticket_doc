## ADDED Requirements

### Requirement: One key per application
Every route under `/v1` SHALL require the header `X-Service-Key`. Each application has its own key, configured by environment (`APP_KEY_<APPLICATION>`, the application's name in upper case with `_` for `-`; for the first one, `APP_KEY_TICKET_MANAGEMENT`), compared in constant time. A missing or unknown key → `401`; a valid key used with the `application` of another → `403`; an application with a catalog but without a key cannot be called. The engine SHALL refuse to start when no application has both a valid catalog and a key. Keys never appear in logs, responses or metrics. `/health` and `/metrics` need no key: the port is published only on the loopback interface and the shared network.

#### Scenario: Missing key
- **WHEN** a request to `/v1/sql` has no `X-Service-Key`
- **THEN** the response is `401` and no model call is made

#### Scenario: Key of another application
- **WHEN** the key of `ticket-management` sends `application = "otra-app"`
- **THEN** the response is `403`

#### Scenario: Rotating a key
- **WHEN** `APP_KEY_TICKET_MANAGEMENT` changes in the engine and in the application and both restart
- **THEN** the old key gets `401`

### Requirement: Rate limits by application and by usage key
The engine SHALL limit the questions (`POST /v1/sql`) per minute: `RATE_LIMIT_APP_PER_MINUTE` (default 60) for each application and `RATE_LIMIT_USAGE_PER_MINUTE` (default 10) for each `usageKey` of an application. `usageKey` is opaque for the engine: the application decides what it stands for (for `ticket-management`, one client), and the engine uses it only to count — never to filter data, and never as a metric label. Beyond a limit the response is `429` with `Retry-After`, and no model call is made. `POST /v1/answer`, `POST /v1/validate` and `GET /v1/catalogs/{application}` are not counted. The counters live in memory: a restart clears them, and the engine runs as a single instance.

#### Scenario: One client asks too much
- **WHEN** the same `usageKey` sends an eleventh question within a minute
- **THEN** it gets `429` with `Retry-After`, while another `usageKey` of the same application is answered

#### Scenario: No usage key
- **WHEN** an application sends no `usageKey`
- **THEN** only the application's limit applies
