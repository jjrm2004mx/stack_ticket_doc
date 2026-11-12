## ADDED Requirements

### Requirement: Validation rules
The validator SHALL accept a SQL only when every rule below holds, checked on the syntax tree SQLGlot builds for the catalog's dialect (`postgres` in this version), and SHALL refuse it with the first reason code that applies. It works by white list: a node, function, operator or type it does not know is refused.

| # | Rule | Reason code |
|---|---|---|
| 1 | The text parses, and to exactly one statement | `UNPARSEABLE`, `NOT_SINGLE_STATEMENT` |
| 2 | The text contains no `;` at all, not even inside a literal, and no comment survives | `SEMICOLON` |
| 3 | The root is a `SELECT` or a set operation (`UNION`, `INTERSECT`, `EXCEPT`) of them, with an optional non-recursive `WITH` | `NOT_A_QUERY` |
| 4 | No node that writes, defines, changes the session or locks, anywhere in the tree: `INSERT`, `UPDATE`, `DELETE`, `MERGE`, DDL, `SET`, `RESET`, `SHOW`, `COPY`, `CALL`, `DO`, `EXPLAIN`, `LOCK`, `FOR UPDATE/SHARE`, `SELECT … INTO`, `TABLESAMPLE`, a recursive CTE | `FORBIDDEN_STATEMENT` |
| 5 | Every table name is unqualified (no schema, no database) | `QUALIFIED_NAME` |
| 6 | Every table name is an object of the catalog or a CTE defined in the same statement; `VALUES` lists are allowed, functions in `FROM` are not | `UNKNOWN_OBJECT` |
| 7 | Every catalog object named is among the request's allowed objects | `OBJECT_NOT_ALLOWED` |
| 8 | No CTE or alias of a table has the name of a catalog object (ignoring case) | `CTE_SHADOWS_OBJECT` |
| 9 | Every function is in the white list of the dialect and is written without schema; the session and system keywords (`CURRENT_USER`, `SESSION_USER`, `CURRENT_SCHEMA`, `CURRENT_CATALOG`, `CURRENT_TIMESTAMP`, `NOW`) are refused; `CURRENT_DATE` and `LOCALTIMESTAMP` are allowed | `UNKNOWN_FUNCTION` |
| 10 | Every column reference resolves to a visible column of an allowed object in scope, to an output of a CTE or subquery, or to an alias; `*` is allowed | `UNKNOWN_COLUMN` |
| 11 | No parameter or placeholder written by the model (`:name`, `$1`, `?`, `%s`, `@name`) | `PARAMETER_IN_SQL` |
| 12 | Only the operators of the white list: arithmetic, comparison, logical, `LIKE`/`ILIKE`, `IN`, `BETWEEN`, `IS [NOT] NULL/TRUE/FALSE`, `IS DISTINCT FROM`, `EXISTS`, `ANY`/`ALL` over a subquery, and text concatenation; regular-expression, JSON and array operators are refused | `FORBIDDEN_OPERATOR` |
| 13 | Casts only to `integer`, `bigint`, `numeric`, `double precision`, `text`, `boolean`, `date`, `timestamp` and `interval` | `FORBIDDEN_TYPE` |
| 14 | Text literals in standard quotes only (no `E'…'`, `U&'…'` nor dollar quoting); a quoted identifier must be exactly a name in scope | `FORBIDDEN_LITERAL` |
| 15 | The model's SQL is at most 4 000 characters and the final SQL at most 20 000 | `TOO_LONG` |

The white list of functions for `postgres` SHALL be: aggregates `count`, `sum`, `avg`, `min`, `max`, `bool_and`, `bool_or`, `string_agg`, `percentile_cont`, `percentile_disc` (with `FILTER` and `WITHIN GROUP`); window functions `row_number`, `rank`, `dense_rank`, `ntile`, `lag`, `lead`, `first_value`, `last_value`; scalars `coalesce`, `nullif`, `greatest`, `least`, `round`, `ceil`, `floor`, `trunc`, `abs`, `lower`, `upper`, `initcap`, `length`, `trim`, `btrim`, `ltrim`, `rtrim`, `substring`, `left`, `right`, `concat`, `replace`, `strpos`, `position`; dates `date_trunc`, `date_part`, `extract`, `age`, `to_char`, `make_date`; and `CASE`. A catalog cannot extend it.

The rules that come from Wren AI's `read_only.py` and `policy.py` (1, 3, 4, 5 and the white list approach of 9) are ported with their Apache-2.0 notice.

#### Scenario: Two statements
- **WHEN** the SQL is `SELECT 1; SELECT 2`
- **THEN** it is refused with `NOT_SINGLE_STATEMENT`

#### Scenario: Write inside a CTE
- **WHEN** the SQL is `WITH d AS (DELETE FROM tickets RETURNING *) SELECT count(*) FROM d`
- **THEN** it is refused with `FORBIDDEN_STATEMENT`

#### Scenario: Schema-qualified names
- **WHEN** the SQL names `insights.tickets`, `public.users`, `pg_catalog.pg_roles` or `information_schema.tables`
- **THEN** each is refused with `QUALIFIED_NAME`

#### Scenario: Table outside the catalog in a subquery, a union or a lateral join
- **WHEN** the SQL names `users` inside a subquery of the `WHERE`, in the second branch of a `UNION` or in a `LATERAL` join
- **THEN** it is refused with `UNKNOWN_OBJECT`

#### Scenario: Dangerous functions
- **WHEN** the SQL calls `set_config`, `current_setting`, `query_to_xml`, `pg_read_file`, `pg_sleep`, `dblink` or `generate_series`
- **THEN** each is refused with `UNKNOWN_FUNCTION`

#### Scenario: CTE with the name of an object
- **WHEN** the SQL is `WITH tickets AS (SELECT 1 AS n) SELECT * FROM tickets`, or the same with `"Tickets"`
- **THEN** it is refused with `CTE_SHADOWS_OBJECT`

#### Scenario: Parameter written by the model
- **WHEN** the SQL contains `WHERE tenant_id = :tenant_id` or `= $1`
- **THEN** it is refused with `PARAMETER_IN_SQL`

#### Scenario: Semicolon inside a literal
- **WHEN** the SQL is `SELECT count(*) FROM tickets WHERE title LIKE '%;%'`
- **THEN** it is refused with `SEMICOLON`

#### Scenario: Reference questions pass
- **WHEN** each expected SQL of the reference questions of the `ticket-management` catalog is validated
- **THEN** all of them are accepted

### Requirement: The final SQL is rendered from the validated tree
The SQL the engine returns SHALL be generated from the syntax tree it validated and rewrote, never from the model's text: comments, odd spacing and anything the parser did not keep cannot reach the database. After the rewrite (`mandatory-filter`) the engine SHALL parse its own output again and check that it is one statement, that every schema-qualified table reference is inside a CTE the engine wrote, and that the number of parameters equals the number of filters applied; when the output cannot be parsed or a check fails, the result is `INVALID` with `REWRITE_CHECK_FAILED`. There is no path that returns a SQL the engine could not re-read.

#### Scenario: Comment in the model's SQL
- **WHEN** the model's SQL carries `/* … */` or `-- …`
- **THEN** the final SQL has no comment

#### Scenario: Output that cannot be re-read
- **WHEN** the engine's rendering of a statement does not parse back
- **THEN** the response is `INVALID` with `REWRITE_CHECK_FAILED`, never the SQL

### Requirement: Row limit
The final SQL SHALL never return more than `rowLimit + 1` rows: a root without `LIMIT`, or with a larger one, gets `LIMIT rowLimit + 1`; a smaller `LIMIT` written by the model is kept. The extra row lets the application tell a truncated result from a complete one. `LIMIT` expressions that are not integer literals are refused (`FORBIDDEN_OPERATOR`).

#### Scenario: Listing without limit
- **WHEN** the model's SQL lists tickets without `LIMIT` and `rowLimit` is 200
- **THEN** the final SQL ends with `LIMIT 201`

#### Scenario: Top five
- **WHEN** the model's SQL has `LIMIT 5`
- **THEN** it is kept

### Requirement: Validation endpoint without a model
`POST /v1/validate` `{application, catalogVersion?, sql, allowedObjects?, rowLimit?, paramStyle?}` SHALL run the same validation and rewrite as `POST /v1/sql`, without calling a model, and answer `200` with `{valid, reason?, detail?, sql?, parameters?, objects?, catalogVersion, requestId}`: for a valid SQL the same final statement `POST /v1/sql` would return; for an invalid one, `valid = false`, the reason code and a short detail. It uses the same key and catalog rules as the other routes and does not count as a question for the rate limits of `application-access`. It is what the engine's tests, the applications' tests and the adoption kit use to exercise the validator and the filter without spending on a model.

#### Scenario: Valid SQL
- **WHEN** an application sends `SELECT count(*) FROM tickets WHERE is_open`
- **THEN** the response has `valid = true`, the final SQL with the filter CTE and its parameters

#### Scenario: Invalid SQL
- **WHEN** an application sends `SELECT * FROM public.users`
- **THEN** the response has `valid = false` and `reason = "QUALIFIED_NAME"`, with status `200`
