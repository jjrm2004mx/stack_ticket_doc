## ADDED Requirements

### Requirement: Catalog format
The engine SHALL read one catalog per application from `catalogs/<application>/`, versioned with the code:

- `catalog.yaml`:
  - `application`, `version` (`MAJOR.MINOR`), `dialect` (`postgres`), `language` (default language of messages);
  - `parameters`: the filter parameters the application binds, each with `name` and `type`;
  - `valueSets`: the lists of the client's own values the application sends with each question, each with `name`, `description`, `maxItems` and the fields of an item (`name` plus optional attributes);
  - `objects`: each with `name` (the name the model uses, unqualified), `source` (the physical, schema-qualified name), `description`, `grain` (what one row is), `group` (a label an application can use to decide who may use it), `filter` or `shared: true` (`mandatory-filter`), and `columns`: each with `name`, `type` (`integer`, `number`, `text`, `boolean`, `date`, `datetime`, `uuid`), `description`, optional `synonyms`, optional `values` (a fixed list, each value with an optional label), optional `valueSet` (the set whose names this column holds) and optional `hidden: true`;
  - `relationships`: how two objects join (`from`, `to`, `on`, cardinality).
- `rules.md`: the meanings and conventions in text (what "abierto" means, how a period is read, what to do without a period, how listings are limited, what is out of scope and the message for it, how the answer is worded).
- `examples.yaml`: question → SQL pairs, each with the objects it uses, plus questions that must not produce SQL and their expected status.
- `contract.json`: the published contract (see "Published contract").

The format SHALL be documented in the repository (`docs/catalogo.md`) and is the same for every application. A catalog holds no credential, no host and no data of any client.

#### Scenario: Adding an application
- **WHEN** a folder `catalogs/otra-app/` with the four files is added and a key for `otra-app` is configured
- **THEN** the engine serves that application without any code change

### Requirement: Catalog validation at start-up
When it starts, the engine SHALL load every catalog and refuse the ones that are not valid, logging why: missing or unknown fields; duplicated object or column names (ignoring case); an object without `filter` and without `shared: true`, or with an invalid filter (`mandatory-filter`); a `valueSet`, relationship or filter that names something that does not exist; a `source` that is not schema-qualified; an example whose SQL does not pass the validator with the objects it declares; a `contract.json` that does not match `catalog.yaml`. An application whose catalog was refused gets `503` `{code: "CATALOG_INVALID"}` on every route, while the other applications keep working; with no valid catalog at all the engine does not start.

#### Scenario: Example that no longer validates
- **WHEN** a column is renamed in `catalog.yaml` and an example still uses the old name
- **THEN** the catalog is refused at start-up and the log names the example and the reason code

#### Scenario: One bad catalog among two
- **WHEN** the catalog of one application is invalid and another is valid
- **THEN** the engine starts, serves the valid one and answers `503` `CATALOG_INVALID` to the other

### Requirement: Catalog version
`version` SHALL be `MAJOR.MINOR`. `MAJOR` changes when something the application's database must match changes: an object, a visible or hidden column, a type, a `source`, a parameter or a filter. `MINOR` changes for anything else: descriptions, synonyms, fixed values and their labels, rules, examples. Every response of `/v1/sql`, `/v1/validate` and `/v1/catalogs/{application}` carries the loaded version, and a request whose `catalogVersion` names another major version is answered `409` (`sql-generation`). A new major version of a catalog is released together with the application's change of its database.

#### Scenario: New synonym
- **WHEN** a synonym is added to a column
- **THEN** the version goes from `1.3` to `1.4` and applications that send `catalogVersion = "1"` keep working

#### Scenario: New column
- **WHEN** a column is added to an object
- **THEN** the version goes to `2.0`, and an application still on `1` gets `409` until its database and its declared version follow

### Requirement: Published contract
`GET /v1/catalogs/{application}` (same key rules as the other routes) SHALL return the contract of the loaded catalog: `{application, version, dialect, parameters: [{name, type}], valueSets: [{name, maxItems, fields}], objects: [{name, source, group, shared, columns: [{name, type, hidden}]}]}` — what the application's database must have, without descriptions, rules or examples. The same document SHALL be committed as `catalogs/<application>/contract.json`, generated from `catalog.yaml`; a test fails when they differ. The application copies that file into its own tests to compare it with its database without the engine running.

#### Scenario: Application checks its views
- **WHEN** `ticket-management` compares `contract.json` of version `1` with the columns of its schema `insights`
- **THEN** every object, column (hidden ones included) and logical type matches, or its test fails

#### Scenario: Another application's catalog
- **WHEN** the key of `ticket-management` asks for `GET /v1/catalogs/otra-app`
- **THEN** the response is `403`

### Requirement: The ticket-management catalog
The first catalog SHALL describe `ticket-management`, version `1.0`, dialect `postgres`, language `es`:
- **Parameter:** `tenant_id` (`integer`).
- **Objects**, with the columns of `stack_ticket_doc/text-to-sql/docs/tecnico/ETAPA_1_DEFINICION_DE_VISTAS.md` §3 and sources `insights.<name>`: `tickets`, `ticket_state_changes`, `states`, `areas`, `categories`, `agents` (group `base`) and `drafts`, `archived_tickets` (group `review`). Every object has the filter `tenant_id = :tenant_id`, and `tenant_id` is `hidden` in all of them.
- **Value sets:** `states` (name, category, outcome), `areas` (name), `categories` (name, area), `agents` (name) and `channels` (name), 200 items at most each; they feed `state_name`, `area_name`, `category_name`, `assignee_name`/`agent_name` and `channel_name`.
- **Fixed values:** `priority` (`BAJA`, `MEDIA`, `ALTA`, `CRITICA`), `state_category` (`NEW`, `ACTIVE`, `WAITING`, `DONE`), `closed_outcome` and `state_outcome` (`RESOLVED`, `CANCELLED`), `origin` (`CORREO`, `PORTAL`), `role` (`USER`, `MANAGER`, `ADMIN`) and `archive_reason` with the portal's label of each value.
- **Rules** (`ETAPA_1_ALCANCE_Y_CATALOGO.md` §3 and `ETAPA_1_DEFINICION_DE_VISTAS.md` §5): "ticket" means an active ticket, never a draft or an archived one unless the question names them; open is `is_open`, closed is category `DONE`, resolved and cancelled are the outcomes; a word that matches the name of a client's state wins over the category; without a period the whole history is used and the period says so; "este mes" is the calendar month, "esta semana" Monday to Sunday, "últimos N días" counts back from today; a period filters by creation date unless the question speaks of closed, resolved or reopened tickets; the resolution time of a period is that of the tickets created in it, as the dashboard; creations are counted in `tickets` by `created_date`, never in `ticket_state_changes`; people are chosen by their exact name from `agents`, and two matches ask for clarification; listings show the most recent first; days are compared unrounded and shown with one decimal; today is `CURRENT_DATE` and now is `LOCALTIMESTAMP`. Out of scope, each with its message: SLA and overdue tickets (not available yet), comments, e-mails, attachments, requesters, configuration, other clients' data, any change of data, and questions about how to use the portal (which suggest the portal's help mode).
- **Examples:** the 16 reference questions with their SQL and the 7 questions that must not produce SQL (N1 to N7).

It SHALL NOT describe nor allow any object that exposes free text of people, requester data, e-mail content or `due_date`.

#### Scenario: Contract of version 1
- **WHEN** `GET /v1/catalogs/ticket-management` is called
- **THEN** it lists the eight objects with source `insights.<name>`, the parameter `tenant_id`, the five value sets, and `tenant_id` as a hidden column of every object

#### Scenario: Reference questions
- **WHEN** the catalog is loaded
- **THEN** its 16 example SQL pass the validator, and each of the 7 negative questions is recorded with the status it must produce

#### Scenario: Question about SLA
- **WHEN** a user asks "¿Cuántos tickets vencidos hay?"
- **THEN** the rules make the answer `OUT_OF_SCOPE` with the message that it is not available yet
