# Etapa 1 — Revisión dirigida de Wren AI

**Fecha:** 2026-11-11
**Plan:** `PLAN_TEXT_TO_SQL_WREN.md` (versión 2), etapa 1
**Alcance:** lectura estática del código. No se instaló ni se ejecutó nada.
**Código leído:** `Canner/WrenAI` en el commit `6e8ac81` (versión 0.18.0, 2026-11-10), comparado con la etiqueta `wren-v0.17.1`; y el adaptador OCI de `ticket-agent`.

---

## 1. Resultado

**Wren sigue siendo viable como base.** Ninguno de los tres motivos de descarte se cumple con lo leído.

Tres supuestos del plan cambian:

1. **No conviene usar `wren-langchain`.** Su toolkit crea el motor con el modo estricto apagado y sin forma de encenderlo, y sus herramientas no pasan el tenant. Se usa el motor (`WrenEngine`) directamente.
2. **El modo estricto no basta.** Deja pasar nombres con esquema (`esquema.tabla`) como tablas físicas y solo bloquea funciones por lista negra. Hace falta el modo `read_only` del motor, que solo existe desde la 0.18.0, más una comprobación propia.
3. **La capa de PostgreSQL no puede depender de una variable de sesión.** El conector no permite fijarla por consulta, y cualquier sesión puede cambiarla. Se propone un rol de PostgreSQL por tenant.

Lo que no se puede saber leyendo queda listado en §5 y se comprueba en la etapa 4.

## 2. Respuesta a cada punto de la etapa

| # | Punto | Respuesta | Estado |
|---|---|---|---|
| 1 | Versión a fijar | `wrenai==0.18.0` exacta (o su parche 0.18.x más reciente al llegar a la etapa 4), **sin** `wren-langchain`. Ver §4. | Decisión propuesta |
| 2 | Control por fila desde YAML | La clave `row_level_access_controls` del YAML se convierte a `rowLevelAccessControls` al compilar. El filtro se inyecta dentro de la expansión de cada modelo, así que aplica aunque el modelo se use en un join o una subconsulta. No hay pruebas del camino completo YAML → PostgreSQL en el paquete Python; las pruebas están en el núcleo. | Confirmado en código; falta ejecutarlo |
| 3 | Sin tenant, ¿falla o deja pasar? | Depende de cómo se declare la propiedad. Con `required: true`, falla. Con `required: false` y sin valor por defecto, **la regla se ignora y no se filtra nada**. | Confirmado en código y en una prueba de Wren |
| 4 | Cómo entra el valor del tenant | Se analiza como expresión SQL, pero solo se aceptan literales; una subconsulta o una función se rechazan. Un texto debe llegar con sus comillas. | Confirmado en código |
| 5 | Variable de sesión de PostgreSQL | El conector abre **una conexión de larga vida, compartida**, y solo acepta parámetros al conectar. No hay forma de fijar una variable por consulta. | Confirmado en código |
| 6 | Compatibilidad con OCI | El adaptador de `ticket-agent` usa `ChatOCIGenAI` con `openai.gpt-oss-20b` y ya expone salida estructurada. No usa llamadas a herramientas en ningún lado. El diseño propuesto no las necesita. | Confirmado en código; falta medir latencia |
| 7 | Modo estricto | Es un campo de `WrenConfig` que se pasa al crear el motor. El toolkit de LangChain no lo pasa. | Confirmado en código |
| 8 | Licencias de dependencias | `wrenai` y `wren-core-py`: Apache-2.0. `psycopg` (el controlador de PostgreSQL) es LGPL-3.0: usarlo sin modificar como librería es compatible con uso comercial. El resto (sqlglot, duckdb, pyarrow, pandas, pydantic) son licencias permisivas. | De memoria; se confirma con un inventario al instalar |

## 3. Hallazgos

La numeración continúa la del plan (H1 a H6).

### H7 — El toolkit de LangChain deja el modo estricto apagado

`WrenToolkit._build_engine` crea el motor sin pasar configuración, y el motor usa entonces los valores por defecto (`strict_mode = False`). El toolkit no ofrece ningún parámetro para cambiarlo.

- Evidencia: `sdk/wren-langchain/src/wren_langchain/_toolkit.py` líneas 151–166; `core/wren/src/wren/engine.py` línea 84; `core/wren/src/wren/config.py` línea 32.
- Consecuencia: con el toolkit, una consulta puede nombrar tablas que no están en el catálogo.

### H8 — Un nombre con esquema pasa como tabla física

El propio motor lo dice en un comentario: *"A qualified name never resolves to a model or a CTE: the planner passes it through to the connector as a physical table"*. La comprobación del modo estricto mira solo el nombre de la tabla, no el esquema.

- Evidencia: `engine.py` líneas 138–145; `core/wren/src/wren/policy.py` líneas 344–384.
- Consecuencia: si un modelo se llama igual que su tabla o vista física, `SELECT * FROM esquema.nombre` podría saltarse el control por fila aun con el modo estricto encendido. Leyendo no se puede asegurar; se prueba en la etapa 4.
- El modo `read_only` del motor rechaza cualquier nombre con esquema.

### H9 — Una regla con propiedad opcional falla abierta

Confirma H2. `validate_rule` devuelve "ignorar la regla" cuando la propiedad es opcional, no llega y no tiene valor por defecto. La prueba `test_session_context` de Wren lo muestra: sin propiedades, la consulta sale sin filtro.

- Evidencia: `core/wren-core/core/src/logical_plan/analyze/access_control/mod.rs` líneas 505–551; `core/wren-core-py/tests/test_modeling_core.py` líneas 54–65 y 125–132.
- Consecuencia: toda regla nuestra declara `required: true`, y un validador propio lo exige antes de publicar el catálogo. Wren exporta `validate_rlac_rule` a Python, que sirve de apoyo.

### H10 — El valor del tenant solo puede ser un literal

Afina H3. El valor se analiza como expresión y `prevent_invalid_expr` rechaza todo lo que no sea un literal.

- Evidencia: `access_control/mod.rs` líneas 469–503; prueba `test_rlac` (líneas 326–337), donde el filtro queda como `WHERE customer.c_name = 'test_user'` dentro del modelo.
- Consecuencia: el riesgo de inyección es bajo. Aun así, nuestro código convierte el tenant a entero antes de pasarlo.

### H11 — Las funciones se bloquean por lista negra, salvo en el modo `read_only`

En modo estricto, las funciones que leen datos externos se bloquean por nombre. El propio código avisa que fuera del `FROM` es una lista negra que hay que mantener. Funciones de PostgreSQL como `query_to_xml` (ejecuta SQL dado como texto) o `set_config` (cambia variables de sesión) no están en la lista.

El modo `read_only` es otra cosa: una sola sentencia, sin nombres con esquema, modo estricto forzado y una **lista blanca** de funciones analíticas; cualquier función desconocida se rechaza.

- Evidencia: `policy.py` líneas 50–62 y 63–118; `core/wren/src/wren/read_only.py` líneas 16–31 y 42–75; `engine.py` líneas 128–156.
- **`read_only.py` no existe en la 0.17.1.** Entró en la 0.18.0.

### H12 — El conector usa una sola conexión compartida y sin límite de tiempo

`PostgresConnector` abre una conexión con `autocommit` y la reutiliza toda la vida del proceso. Ejecuta el SQL como texto. Si se le pasa un límite de filas, envuelve la consulta en `SELECT * FROM (...) LIMIT n`. No fija un tiempo máximo de consulta.

- Evidencia: `core/wren/src/wren/connector/postgres.py` líneas 289–311 y 314–328.
- Consecuencias:
  - No sirve una variable de sesión por petición (punto 5).
  - El límite de tiempo se fija en el rol de PostgreSQL (`ALTER ROLE … SET statement_timeout`), no en la aplicación.
  - Nuestro código pasa siempre un límite de filas.
  - Todas las consultas de un motor van en fila por la misma conexión; con pocos usuarios no importa.

### H13 — El camino "gobernado" de Wren no pasa el tenant

`governed_query.py` ejecuta cada consulta en un proceso hijo con plazo de 90 s y acota el resultado (100 filas, 20 columnas, textos de 256 caracteres). Llama a `engine.query(sql, limit, read_only=True)` **sin** `properties`, así que no sirve con control por fila.

- Evidencia: `core/wren/src/wren/governed_query.py` líneas 262–299 y 310.
- Se copia la idea de acotar el resultado, no el módulo.

### H14 — La comprobación del SQL ya expandido falla abierta si no se puede analizar

`validate_planned_sql` vuelve a revisar el SQL final, pero si sqlglot no logra analizarlo, lo deja pasar. Está documentado así en el código.

- Evidencia: `policy.py` líneas 271–289.
- Consecuencia: es una red secundaria; la frontera real son los permisos de PostgreSQL.

## 4. Decisiones que propongo

### D1 — No usar `wren-langchain`; usar el motor directamente con un grafo fijo

En lugar de un agente con herramientas, un grafo de LangGraph con pasos fijos:

1. Elegir el contexto del catálogo para la pregunta (código).
2. El LLM entrega el SQL como **salida estructurada**, o pide una aclaración.
3. Comprobación propia, planeación y validación sin ejecutar (código).
4. Si falla, un reintento con el error (LLM).
5. Ejecutar con el tenant de la petición (código).
6. El LLM redacta la respuesta con los datos devueltos.

Por qué:
- La salida estructurada ya funciona con OCI en `ticket-agent`; las llamadas a herramientas no se han usado nunca con ese modelo.
- El LLM no tiene herramientas, así que no puede fijar nada: resuelve H1 y H7 de raíz.
- Son 2 llamadas a OCI por pregunta en el caso normal y 3 con reintento. Cabe mejor en el límite de 60 s y cuesta menos.
- Se quita una dependencia (`wren-langchain`, que además exige `langchain>=1.0`).
- El texto de reglas para escribir SQL que trae Wren se puede reutilizar: es Apache-2.0.

### D2 — Fijar `wrenai==0.18.0`

Antes dije que convenía una versión con semanas de uso. Tras leer el código, recomiendo la 0.18.0 aunque sea nueva:

- Es la primera con el modo `read_only` (H11).
- Trae correcciones del reescritor que tocan a qué se liga cada tabla, entre ellas "respect CTE visibility when expanding semantic models" (#2833).
- La 0.17.1 no tiene ninguna de las dos cosas.

Lo que compensa que sea reciente: versión exacta con sus sumas de verificación, comprobación propia antes de Wren, permisos de PostgreSQL por debajo y la compuerta 1. Si al llegar a la etapa 4 hay un parche 0.18.x, se toma ese.

### D3 — Tres comprobaciones antes de ejecutar, en este orden

1. **Comprobación propia de GemIA:** una sola sentencia `SELECT`, sin nombres con esquema, sin funciones fuera de una lista blanca. Son pocas líneas y no dependen de Wren.
2. **Motor de Wren** con `strict_mode=True` y `read_only=True`, y el tenant en `properties`.
3. **Revisión del SQL expandido:** debe contener el filtro del tenant.

### D4 — Capa de PostgreSQL con un rol por tenant

Opciones para que PostgreSQL filtre por su cuenta:

| Opción | Cómo | Problema |
|---|---|---|
| Variable de sesión | Las vistas filtran por `current_setting('app.tenant_id')` | La sesión puede cambiarla (`set_config`). Depende de que las capas de arriba bloqueen esa función: deja de ser independiente. |
| **Rol por tenant** (recomendada) | Cada tenant tiene un rol de solo lectura. Las vistas filtran por el tenant asociado a `session_user`, que una sesión sin privilegios no puede cambiar. Un motor por tenant. | Hay que crear el rol al dar de alta al tenant. |

Para el POC son 2 roles. El costo para producción (crear el rol en el alta de tenants) se decide en la etapa 3.

### D5 — Reglas del catálogo

- Todo modelo lleva una regla de tenant con `required: true`. Un validador propio rechaza el catálogo si falta alguna.
- Los modelos no se llaman igual que las vistas físicas (H8).
- En el POC, sin campos calculados que crucen modelos: no pude confirmar leyendo que hereden el filtro del otro modelo.

## 5. Lo que solo se comprueba ejecutando (va a la etapa 4)

1. Que exista un paquete ya compilado de `wren-core-py` para Rocky con Python 3.11. Si no existe, habría que compilar Rust, y eso cambia el costo.
2. Que el control por fila declarado en YAML llegue hasta el SQL de PostgreSQL.
3. Si un nombre con esquema se salta el filtro (H8), con y sin modo `read_only`.
4. Qué pasa con mayúsculas en el nombre de la propiedad.
5. Que la lista blanca de funciones del modo `read_only` alcance para preguntas reales (fechas, conteos, agrupaciones).
6. El tamaño real de la imagen. `wrenai` trae duckdb, pandas, pyarrow, boto3 y opendal aunque no se usen.
7. La latencia de `openai.gpt-oss-20b` generando SQL como salida estructurada.

## 6. Casos negativos que se agregan a la etapa 4

Además de los ya previstos (sin tenant, tenant ajeno, tenant mal formado, tabla fuera del catálogo, escritura):

- Nombre con esquema: `insights.vista`, `public.tabla`, `pg_catalog.pg_roles`, `information_schema.tables`.
- Dos sentencias en una.
- Funciones: `set_config`, `current_setting`, `query_to_xml`, `pg_read_file`, `dblink`.
- Un CTE del usuario con el nombre de un modelo.
- Identificadores entre comillas y con mayúsculas distintas.
- Subconsulta en el `WHERE` que nombra una tabla física.
- `UNION` entre un modelo y una tabla física.
- Regla con `required: false` (debe rechazarla el validador del catálogo).
- Conexión con el rol de un tenant pidiendo datos de otro, directo en PostgreSQL y sin pasar por Wren.

## 7. Efecto en el plan

- **Etapa 3:** incluye D1 a D5 como base del diseño.
- **Etapa 4:** se agregan los casos de §6 y las comprobaciones de §5.
- **Etapa 6:** el montaje es el grafo fijo de D1, sin `wren-langchain`.
- **Esfuerzo y tamaños:** sin cambio.
- **Motivos de descarte:** ninguno se cumple. El de OCI baja de riesgo porque ya no se necesitan llamadas a herramientas.
