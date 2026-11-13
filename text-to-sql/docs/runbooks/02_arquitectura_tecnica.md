# 02 — Arquitectura técnica

Fuente de verdad: `openspec/` (cambio `motor-inicial` hasta su archivado). Diseño razonado:
`openspec/changes/motor-inicial/design.md` (D1–D17).

## Piezas

```
src/
  api/app.py            rutas, llave, cupos, request id, respuesta de errores
  api/access.py         KeyRing (comparación en tiempo constante) y RateLimiter (ventana de 60 s)
  api/observability.py  logs JSON con campos en lista blanca, métricas tts_*
  api/schemas.py        cuerpos de petición (campos desconocidos se ignoran)
  catalog/model.py      formato del catálogo (pydantic, sin campos extra)
  catalog/loader.py     carga y validación al arrancar (filtros, referencias, ejemplos, contrato)
  catalog/contract.py   contrato publicado y su generador
  sql/rules.py          listas blancas y códigos de motivo (portado de Wren, ver NOTICE)
  sql/validator.py      validación, filtro por CTE, límite de filas, relectura
  sql/render.py         dialecto que escribe :nombre o ? (no el %(nombre)s de psycopg)
  pipeline/prompt.py    prompts de SQL y de respuesta, limpieza del texto
  pipeline/pipeline.py  tubería fija con un reintento y presupuesto de tiempo
  pipeline/providers.py SqlDraft (salida estructurada), tiempo por llamada, fábrica de adaptadores
  adapters/             copiados tal cual de ticket-agent (ORIGEN.md)
catalogs/<aplicación>/  catalog.yaml, rules.md, examples.yaml, contract.json
```

## Tubería de `POST /v1/sql`

1. Llave (`401`), cuerpo (`400`), aplicación de la llave (`403`), catálogo cargado (`503`), versión
   MAYOR (`409`), objetos permitidos y valores (`400`), cupo (`429`). Ninguno llama al modelo.
2. Prompt: reglas de escritura de SQL, reglas del catálogo, objetos permitidos con columnas visibles,
   objetos no disponibles (solo nombre y descripción), relaciones, ejemplos cuyos objetos están
   permitidos, ejemplos sin SQL, fecha de hoy y, como **datos** codificados en JSON: valores del
   cliente, turno anterior, error de la base y la pregunta.
3. Salida estructurada `SqlDraft {status, sql, period, assumptions, message}`, temperatura 0, sin
   herramientas, con tope `MODEL_TIMEOUT_S`.
4. `CLARIFY` / `OUT_OF_SCOPE`: se devuelve el mensaje; cualquier SQL que traiga se descarta.
5. `SQL`: validación y reescritura. Si se rechaza, **un** reintento con el motivo — solo si queda en
   `SQL_BUDGET_S` al menos el tiempo de una llamada. Si vuelve a fallar: `INVALID` con el motivo.
6. Una falla o tiempo agotado del proveedor: `503 MODEL_UNAVAILABLE`, sin repetir.

## Validador (orden de evaluación)

| Paso | Reglas | Códigos |
|---|---|---|
| Texto | largo ≤ 4 000; sin `\`; empieza con `SELECT`, `WITH` o `(`; solo textos `'…'`; sin `:x`, `$1`, `?`, `%s`, `@x` | `TOO_LONG`, `FORBIDDEN_LITERAL`, `NOT_A_QUERY`, `PARAMETER_IN_SQL` |
| Lectura | una sola sentencia; ningún `;` en el texto | `UNPARSEABLE`, `NOT_SINGLE_STATEMENT`, `SEMICOLON` |
| Árbol (por número de regla) | raíz de consulta; nodos con efectos o fuera de la lista blanca; nombres con esquema; objetos del catálogo o CTE visibles; objetos permitidos; CTE o alias con nombre de objeto; funciones de la lista blanca; operadores; tipos | 3 `NOT_A_QUERY`, 4 `FORBIDDEN_STATEMENT`, 5 `QUALIFIED_NAME`, 6 `UNKNOWN_OBJECT`, 7 `OBJECT_NOT_ALLOWED`, 8 `CTE_SHADOWS_OBJECT`, 9 `UNKNOWN_FUNCTION`, 11, 12 `FORBIDDEN_OPERATOR`, 13 `FORBIDDEN_TYPE` |
| Columnas | cada columna resuelve a una columna visible de un objeto en alcance, a una salida de CTE o subconsulta, o a un alias | 10 `UNKNOWN_COLUMN` |
| Reescritura | CTE por objeto + límite + relectura | 15 `TOO_LONG`, `REWRITE_CHECK_FAILED` |

Decisiones finas:
- **Barra invertida rechazada:** SQLGlot la lee como escape dentro de `'…'` y PostgreSQL (con
  `standard_conforming_strings`) no. Rechazarla quita esa diferencia de lectura.
- **Comandos sin gramática** (`SHOW`, `RESET`, `CALL`, `DO`, `LOCK`, `EXPLAIN`): SQLGlot los vuelve un
  texto opaco; se rechazan por la primera palabra.
- **Funciones por lista blanca:** cada nombre permitido se lee una vez con SQLGlot y se admite la
  clase que produce; un nombre que SQLGlot no conoce se admite solo si está en la lista. Nunca se
  admite `Anonymous` en general, ni una función con esquema (`pg_catalog.count`).
- **Visibilidad de CTE** como PostgreSQL: un nombre sin esquema se liga al CTE visible más cercano;
  dentro de un CTE, solo los anteriores del mismo `WITH`.
- **CTE del motor `NOT MATERIALIZED`:** PostgreSQL lo integra en cada referencia y puede omitir
  columnas costosas (`resolution_days`). El filtro queda dentro del CTE en cualquier caso.
- **Relectura:** el SQL final se vuelve a leer y debe dar exactamente el mismo texto (punto fijo),
  tener como únicas tablas con esquema las fuentes de los CTE del motor y tantos parámetros como
  filtros aplicados.
- **Parámetros:** `named` devuelve los nombres distintos; `qmark` uno por `?`, en el orden del texto
  (el de los CTE). Un `?` dentro de un texto no cuenta: JDBC lo distingue.

## Límites de tiempo

| Variable | Defecto | Dónde |
|---|---|---|
| `MODEL_TIMEOUT_S` | 20 | Cada llamada al modelo |
| `SQL_BUDGET_S` | 40 | `/v1/sql` completo (TM espera 45) |
| `ANSWER_BUDGET_S` | 18 | `/v1/answer` (TM espera 20) |

La llamada al adaptador corre en un hilo con `asyncio.wait_for`; al agotarse el tiempo, la respuesta
sale `503` aunque el hilo termine después.

## Llaves y cupos

- `APP_KEY_<APLICACIÓN>` por cada carpeta de `catalogs/`. Sin al menos una aplicación con catálogo
  válido y llave, el motor no arranca.
- Cupos en memoria, ventana deslizante de 60 s: `RATE_LIMIT_APP_PER_MINUTE` (60) por aplicación y
  `RATE_LIMIT_USAGE_PER_MINUTE` (10) por `usageKey`. Solo cuenta `/v1/sql`. Un reinicio los limpia.

## Observabilidad

- Logs JSON a la salida estándar (journald, etiqueta `text-to-sql`). Campos permitidos: `ts`,
  `level`, `logger`, `requestId`, `msg`, `application`, `route`, `status`, `outcome`, `reason`,
  `reasons`, `durationMs`, `modelCalls`, `provider`, `model`, `catalogVersion`, `rows`, `objects`,
  `code`. El registro de SQLGlot está apagado porque escribe el texto que no entiende.
- Métricas: las de `prometheus-fastapi-instrumentator` y `tts_requests_total`,
  `tts_validation_failures_total`, `tts_model_calls_total`, `tts_model_call_seconds`,
  `tts_rate_limited_total`, todas por `application`, nunca por `usageKey`.
- `X-Request-Id`: el entrante si es válido (`[A-Za-z0-9._-]{1,64}`), si no uno nuevo; va en la
  cabecera, en el cuerpo y en cada línea de log. TM lo guarda en `insights_questions.engine_request_id`.

## Contenedor y red

`python:3.11-slim`, `uvicorn --factory src.api.app:create_app_from_env`, puerto 8003 publicado solo
en `127.0.0.1`, red `ticket-management-network` (externa), `~/.oci` de solo lectura, journald,
`restart: unless-stopped`, chequeo de salud con `/health`. Sin volúmenes. El motor no está en la red
de `ticket-db`.

## Pruebas

`py -m pytest` corre sin modelo ni red. `--evidence=<archivo.md>` escribe la tabla de casos M1–M27
(`docs/evidencia/`). Los casos se marcan con `@pytest.mark.case("M7")`.
