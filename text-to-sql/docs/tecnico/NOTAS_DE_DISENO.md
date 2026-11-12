# Notas de diseño para la etapa 2

**Fecha:** 2026-11-11
**Para qué:** dejar escrito el razonamiento técnico que no está en el plan, para redactar con él las propuestas OpenSpec y las pruebas de la compuerta 1.
**Estado de cada afirmación:** lo marcado **[verificado]** se leyó en el código. Lo marcado **[por probar]** es conocimiento de PostgreSQL o del controlador que debe confirmarse ejecutando en la etapa 3.

---

## 1. Capa de base de datos en TM

### Roles

| Rol | Tipo | Permisos |
|---|---|---|
| `ticket_insights_owner` | Sin acceso de entrada, sin saltarse la seguridad por fila | `SELECT` solo sobre las columnas de las tablas base que usan las vistas. Dueño de las vistas |
| `ticket_insights` | Con acceso de entrada, sin saltarse la seguridad por fila | Uso del esquema `insights` y `SELECT` sobre sus vistas. Nada sobre `public` |

Ajustes fijados en `ticket_insights`: tiempo máximo de consulta, transacciones de solo lectura por defecto, tiempo máximo de transacción ociosa, zona horaria del negocio, `search_path = insights` y un límite bajo de conexiones. La contraseña entra por `afterMigrate.sql`, como las de los otros roles **[verificado: patrón existente]**.

### Trampas

| # | Trampa | Qué hacer |
|---|---|---|
| T1 | Flyway corre como superusuario **[verificado]**. Una vista sin opciones aplica las políticas de **su dueño**, y un superusuario se las salta: la vista mostraría todos los clientes **[por probar]** | En la misma migración, pasar cada vista a `ticket_insights_owner`. Prueba automática: todo objeto de `insights` pertenece a ese rol |
| T2 | La alternativa `security_invoker = true` obliga a dar permisos sobre las tablas base al rol que consulta | Descartada: ese rol leería las tablas directamente |
| T3 | La política lee `current_setting('app.tenant_id')` **[verificado]**. Aplicada "como el dueño de la vista", debe seguir leyendo el valor de la sesión **[por probar]** | Caso de prueba: mismo rol, dos tenants, cada uno ve solo lo suyo a través de la vista |
| T4 | `set_config` cambia el tenant dentro de un `SELECT`, y TM la usa **[verificado: `TenantAwareJpaTransactionManager` línea 24]** | Variantes A o B, abajo |
| T5 | El controlador JDBC puede ejecutar varias sentencias separadas por `;` en una sola llamada **[por probar]**. Envolver la consulta en `SELECT * FROM ( … ) q LIMIT n` no basta: `) q; SET LOCAL …; SELECT * FROM (` cierra el paréntesis y sigue | TM rechaza cualquier `;`. El motor garantiza una sola sentencia. Probar si el controlador de verdad lo ejecutaría |
| T6 | Todos los roles pueden usar el esquema `public` por defecto. Una función de ese esquema que nadie haya cerrado sería invocable. Las sensibles ya tienen `REVOKE … FROM PUBLIC` **[verificado: V33 a V38]** | Prueba automática: `ticket_insights` no tiene permiso de ejecución sobre ninguna función de `public` |
| T7 | `pg_sleep`, `generate_series` y los bloqueos consultivos no filtran datos, pero estorban | Lista blanca del motor y tiempo máximo de consulta |
| T8 | Si las fechas se guardan como `timestamp` sin zona en UTC, convertirlas mal corre el día | Confirmar el tipo en la etapa 1 y normalizar en la vista |

### El tenant no se puede cambiar: dos variantes

**Variante A — quitar `set_config`.**
`REVOKE EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) FROM PUBLIC` y concederlo a `ticket_app`, `ticket_platform` y `ticket_relay`. En la conexión de solo lectura, TM fija el tenant con `SET LOCAL app.tenant_id = '<entero>'`, que es una orden y no una función; el entero viene del token y se valida como tal.
Riesgo: es un cambio para toda la base. Se corre la suite completa de TM. El superusuario y las funciones con privilegios del dueño no se ven afectados **[por probar]**.

**Variante B — filtro que la sesión no puede falsificar.**
Cada vista agrega `AND tenant_id = (SELECT tenant_id FROM insights.session_scope WHERE backend_pid = pg_backend_pid())`. Antes de cada consulta, TM escribe "esta conexión → este tenant" con su rol normal, a través de una función que solo `ticket_app` puede ejecutar, y lo borra al terminar. Si alguien cambiara la variable, las dos condiciones no coincidirían y saldría vacío.
Costo: una tabla y dos idas más a la base por pregunta.

**Preferencia:** A, por simple. B si la suite muestra que A rompe algo.

### Ejecución en TM

- Conexión propia de dos o tres conexiones, con el patrón de `PlatformDatabase` **[verificado]**.
- Por pregunta, una transacción de solo lectura: fijar tenant, ejecutar con parámetros con nombre, leer como máximo N+1 filas, acotar el tamaño de cada celda.
- Antes de devolver filas, comprobar que `app.tenant_id` sigue siendo el esperado. Es barato y detecta un salto que las otras capas no hubieran parado.
- Si la redacción falla o tarda, se entrega la tabla con una frase fija. No se reintenta sola una petición que agotó el tiempo: hoy el widget del asistente repite la pregunta y duplica el costo **[verificado: spec `ai-agent-integration`, Known Gaps]**.

## 2. Validador del motor

Reglas, en este orden. Las tres primeras vienen de `read_only.py` y `policy.py` de Wren.

1. Exactamente una sentencia.
2. La raíz es un `SELECT` o una unión de ellos.
3. Ningún nodo de escritura, definición, orden de sesión, bloqueo ni `SELECT … INTO`. Ningún CTE recursivo.
4. **Tablas:** solo nombres sin esquema. Cada una debe ser un objeto del catálogo o un CTE definido en la misma consulta. Un CTE no puede llamarse como un objeto del catálogo.
5. **Funciones:** lista blanca por tipo. Se rechaza toda función desconocida y toda función con esquema.
6. **Columnas:** deben existir en el objeto. El error vuelve al modelo en el reintento.
7. **El modelo no escribe parámetros** (`:nombre`, `$1`, `?`). Solo el motor los agrega.
8. Se quitan los comentarios. Hay largo máximo. Se agrega o se recorta el `LIMIT`.

### Filtro obligatorio

- Por cada objeto del catálogo que la consulta nombra, el motor antepone un CTE con ese mismo nombre: `objeto AS (SELECT <columnas del catálogo> FROM <origen físico> WHERE <filtro>)`.
- Como las tablas del modelo no llevan esquema, toda referencia se liga al CTE, también dentro de subconsultas y uniones. Solo el motor escribe el nombre físico con esquema.
- La regla 4 cierra los dos escapes que tenía Wren: el nombre con esquema y el CTE del usuario que tapa al objeto.
- El filtro usa parámetros con nombre (`:tenant_id`). La respuesta lista los que la aplicación debe llenar. El motor nunca recibe el valor.
- **Falla cerrado:** un objeto sin filtro invalida el catálogo, salvo que se declare compartido a propósito. En Wren, una regla opcional sin valor se ignoraba y dejaba pasar todo **[verificado: `access_control/mod.rs` líneas 505–551]**.
- `/v1/validate` devuelve el mismo SQL final. Así se prueba el filtro sin gastar en el modelo.

## 3. Casos negativos para las pruebas

**En la base (compuerta 1, sin motor):**
- Con el rol `ticket_insights`: leer una tabla de `public`; leer una vista sin tenant fijado; con el tenant A, ver filas del B; ejecutar `set_config`; ejecutar una función de `public`; escribir; crear un objeto; pasar el tiempo máximo.
- Una vista creada sin cambiarle el dueño debe hacer fallar la prueba automática.
- Dos sentencias en una llamada del controlador.

**En el motor (sin modelo):**
- Dos sentencias; escritura; CTE con escritura; `SELECT … INTO`.
- Nombre con esquema: `insights.tickets`, `public.users`, `pg_catalog.pg_roles`, `information_schema.tables`.
- Funciones: `set_config`, `current_setting`, `query_to_xml`, `pg_read_file`, `pg_sleep`, `dblink`.
- CTE del usuario llamado como un objeto; identificador entre comillas con otra capitalización.
- Subconsulta, unión y `LATERAL` que nombran una tabla fuera del catálogo.
- Parámetro escrito por el modelo.
- Catálogo con un objeto sin filtro.

**De punta a punta (compuerta 2, con modelo):**
- Las preguntas N1 a N7 de `ETAPA_1_ALCANCE_Y_CATALOGO.md`.
- Un estado o un título de ticket con instrucciones escondidas.
- Con la seguridad por fila apagada en una base de prueba, el filtro del motor debe aislar por sí solo.

## 4. Contrato del motor, en borrador

| Ruta | Entra | Sale |
|---|---|---|
| `POST /v1/sql` | Pregunta, catálogo, valores, dialecto, idioma, fecha de hoy, límite de filas. Opcionales: error de la base con su SQL, y pregunta anterior con su SQL | Estado (`sql`, `aclarar`, `fuera_de_alcance`, `invalido`), SQL final, parámetros que faltan llenar, objetos usados, supuestos, proveedor y modelo |
| `POST /v1/answer` | Pregunta, SQL, columnas y filas acotadas, idioma | Texto, proveedor y modelo |
| `POST /v1/validate` | SQL, catálogo, dialecto | Válido o motivo; SQL final y parámetros |

- Autenticación: una llave por aplicación. El cupo se aplica por una clave de uso que manda la aplicación (para TM, el cliente), opaca para el motor.
- Los logs no llevan preguntas ni filas.
- El catálogo viaja en cada petición. El motor no lo guarda.
