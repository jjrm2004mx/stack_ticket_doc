## ADDED Requirements

### Requirement: Every object declares its filter
Each object of a catalog SHALL declare either a `filter` — a boolean expression over the object's own columns and the catalog's declared parameters (for `ticket-management`, `tenant_id = :tenant_id` on its eight objects) — or `shared: true`, stating on purpose that its rows are the same for everybody. A catalog with an object that has neither, with a filter that names an undeclared parameter or a column the object does not have, or with a filter that is not a plain comparison of columns, parameters and literals (no function, no subquery) SHALL be refused when it loads (`application-catalog`): the engine fails closed and never serves an object without its filter. A filter cannot be optional or depend on the request.

#### Scenario: Object without filter
- **WHEN** a catalog declares an object with no `filter` and no `shared`
- **THEN** the catalog does not load and every request of that application gets `503` `CATALOG_INVALID`

#### Scenario: Shared object declared on purpose
- **WHEN** an object is declared `shared: true`
- **THEN** the catalog loads and the object is served without filter

### Requirement: The engine applies the filter to every use of an object
For each catalog object a validated statement names, the engine SHALL prepend to the statement's `WITH` a CTE with the object's own name: `<object> AS (SELECT <visible columns> FROM <physical source> WHERE <filter>)`, before any CTE of the model. Because the model's table names are never qualified and can never be the name of a CTE of its own (`sql-validation`, rules 5 and 8), every reference to the object — in the main query, a subquery, a branch of a set operation or a lateral join — binds to the engine's CTE. Only the engine writes the physical, schema-qualified name. The columns the catalog marks `hidden` (the filter's own, such as `tenant_id`) are not projected, so the model cannot select them. A shared object gets the same CTE without `WHERE`. The filter is always applied: there is no request field, catalog option or model output that removes it.

#### Scenario: Object used twice
- **WHEN** a statement names `tickets` in the main query and in a subquery
- **THEN** the final SQL has one CTE `tickets` with the filter, and both references read it

#### Scenario: Join of two objects
- **WHEN** a statement joins `states` and `tickets`
- **THEN** the final SQL has the CTEs `states` and `tickets`, each with its filter, and two parameters to bind

#### Scenario: Model's own CTE
- **WHEN** the statement has `WITH por_area AS (SELECT area_name, count(*) AS n FROM tickets GROUP BY 1) SELECT * FROM por_area`
- **THEN** the final `WITH` lists the engine's `tickets` first and `por_area` after it

#### Scenario: The filter alone isolates
- **WHEN** the final SQL of any reference question runs on a database whose row-level security is off, with `tenant_id` bound to one client
- **THEN** it returns only rows of that client

### Requirement: Parameters are returned unbound
The filter's parameters SHALL appear in the final SQL as placeholders and SHALL be listed in `parameters`; the engine never receives, stores or logs their values. With `paramStyle = "named"` each parameter is written `:name` and `parameters` lists the distinct names; with `"qmark"` each occurrence is written `?` and `parameters` lists one name per `?`, in the order they appear, so an application can bind them by position without parsing the SQL. The application supplies the values from its own trusted source (for `ticket-management`, the tenant of the token). No placeholder other than the filters' can exist in the final SQL (`sql-validation`, rule 11).

#### Scenario: qmark with two objects
- **WHEN** a statement over `states` and `tickets` is returned with `paramStyle = "qmark"`
- **THEN** the SQL has two `?` and `parameters = ["tenant_id", "tenant_id"]`

#### Scenario: named
- **WHEN** the same statement is returned with `paramStyle = "named"`
- **THEN** the SQL has `:tenant_id` twice and `parameters = ["tenant_id"]`
