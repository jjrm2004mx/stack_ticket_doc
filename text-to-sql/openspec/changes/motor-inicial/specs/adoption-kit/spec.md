## ADDED Requirements

### Requirement: Adoption guide
The repository SHALL include `docs/adopcion.md`, the steps for an application that wants questions about its own data, in the order they are done: (1) design views with names next to ids, local dates and computed metrics, each carrying the column its filter needs; (2) create a read-only database role that sees only those views and cannot change the variable or identity the filter depends on; (3) write the catalog (`application-catalog`) with a mandatory filter on every object; (4) ask for a key; (5) mount an executor that binds the parameters from its own trusted source, bounds rows and time and runs in a read-only transaction; (6) record each question in the application; (7) run the test package. The guide SHALL state what the engine guarantees (validated and filtered SQL) and what it does not: the application's database must hold on its own, because the engine never sees it.

#### Scenario: Second application
- **WHEN** a team that does not know the engine follows the guide
- **THEN** it knows which pieces are its own (views, role, executor, permissions, log) and which the engine gives (catalog format, SQL, wording)

### Requirement: Example executor
The repository SHALL include a small example executor (`examples/executor/`, Python, under 150 lines) that an application can copy: it calls `POST /v1/sql` with `paramStyle = "qmark"`, refuses a SQL that contains `;` or a parameter name it does not know, opens a read-only transaction, binds the parameters by position from values the application supplies, limits rows and time, and calls `POST /v1/answer`. It is an example, not a library: the engine publishes no client package in this version.

#### Scenario: Reading the example
- **WHEN** a developer reads the example
- **THEN** every step of the flow is visible in one file, with the guards before the execution

### Requirement: Test package for the application's database
The repository SHALL include a test package (`adoption-tests/`, pytest) that an application runs against its own database with its own read-only role, given a connection string, the name of its filter variable and two tenants with data: the role cannot read a table outside its views, sees nothing without the filter value, sees only its tenant with it, cannot change the filter value from inside a query, cannot write or create objects, cannot run a second statement, and is cancelled by its time limit; and, through `POST /v1/validate`, that the engine's final SQL for the application's examples returns only the bound tenant's rows on a copy of the database without row-level security. The package reports pass or fail per case and changes nothing in the database.

#### Scenario: Application with a role that can read a base table
- **WHEN** the package runs against a database whose read-only role has `SELECT` on a base table
- **THEN** the case "tabla fuera de las vistas" fails and names the table
