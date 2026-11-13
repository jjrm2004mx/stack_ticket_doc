# Paso A — Revisión del stack actual para la función de preguntas sobre los datos

**Fecha:** 2026-11-11
**Alcance:** solo lectura de specs, código y configuración de los repos. No se ejecutó ni se cambió nada.
**Para qué:** decidir dónde vive la función y qué se reutiliza, antes de escribir el plan versión 3 (motor propio delgado, sin Wren instalado).
**Criterio:** encajar en lo que existe; una pieza nueva solo entra con su justificación.

> **Nota del 2026-11-11:** el reparto recomendado aquí (opción D, con la parte del modelo dentro de `ticket-agent` y la validación en Java) cambió en el plan versión 3: el modelo y la validación van en un repo nuevo y genérico. Los hallazgos y las restricciones de este documento siguen vigentes.

---

## 1. Resultado

**La función cabe en el stack sin servicio, contenedor ni red nuevos.** Se reparte entre dos repos que ya se hablan:

- **`ticket-management` (TM):** dueño de los datos. Pone el permiso, el interruptor, el catálogo, la validación del SQL, la ejecución y la pantalla.
- **`ticket-agent`:** dueño del modelo de lenguaje. Escribe el SQL y redacta la respuesta. No toca la base.

Tres hallazgos cambian lo que traíamos pensado:

1. **El aislamiento por tenant ya existe en PostgreSQL.** TM tiene seguridad por fila forzada en todas las tablas de tenant desde el cambio `aislamiento-entre-tenants`. No hace falta un rol por tenant ni vistas que filtren por tenant.
2. **La base de TM no se alcanza desde fuera de TM.** `ticket-db` está solo en la red interna de TM, y la regla del ecosistema es que cada servicio guarda y recupera lo suyo. La consulta tiene que ejecutarse dentro de TM.
3. **Las métricas ya tienen definición.** El tablero «Métricas» calcula SLA, MTTR, vencidos y reapertura con reglas escritas en su spec. Las respuestas de la función nueva deben dar los mismos números.

## 2. Lo que ya existe y se reutiliza

| Pieza | Dónde está | Para qué sirve aquí |
|---|---|---|
| Seguridad por fila forzada, con política `tenant_id = app.tenant_id` | TM, migraciones `V34`, `V35`, `V37` y siguientes | El aislamiento por tenant, sin construir nada |
| Tenant fijado por transacción desde el token | TM, `TenantAwareJpaTransactionManager` y `JwtAuthFilter` | El tenant sale del JWT y nunca de la petición ni del LLM |
| Roles de base por función (`ticket_app`, `ticket_platform`, `ticket_relay`), cada uno con su conexión | TM, `V33`, `V35`, `afterMigrate.sql`, `application.properties` | Patrón para agregar un rol de solo lectura con permisos mínimos |
| Matriz de permisos por rol; grupo «Insights» con `VIEW_DASHBOARD` y `USE_AGENT` | TM, `Action.java`, spec `rbac-permissions` | Una acción nueva para la función |
| Regla de visibilidad de tickets (`VIEW_ALL_TICKETS`) | TM, spec `ticket-visibility` | Decide quién puede preguntar sobre todos los tickets |
| Interruptores de IA por cliente, con historial y auditoría | TM, `V63`, specs `platform-operator` y `ai-configuration` | Un interruptor nuevo, apagable por el operador |
| Llamada TM → agente con llave de servicio; rol y tenant tomados del JWT | TM `AgentClient`; agente `api/auth.py` y `routes/query.py` | El mismo canal y la misma llave para las rutas nuevas |
| Límite de peticiones por tenant, identificador de petición, métricas y logs | `ticket-agent`, `api/limiter.py`, `api/main.py` | Se heredan en las rutas nuevas |
| Adaptador de OCI con salida estructurada | `ticket-agent`, `adapters/oci_adapter.py` | El LLM entrega el SQL como estructura; ya está probado con OCI |
| Definición de cada métrica y de la zona horaria | TM, `DashboardService`, spec `dashboard-analytics` | Fuente de las reglas del catálogo |
| SQL nativo con `JdbcTemplate` | TM, 46 clases lo usan | La ejecución de la consulta sigue un patrón conocido |
| Migraciones Flyway | TM | Crean el rol y las vistas, versionados |
| Pantalla «Métricas» y widget del asistente | TM, specs `dashboard-ui` y `help-and-agent-ui` | Lugar natural de la función en el portal |
| Flujo OpenSpec en cada repo | Todos | La función entra como cambio normal |

## 3. Restricciones que fija el stack

| # | Restricción | Evidencia | Consecuencia |
|---|---|---|---|
| R1 | La base de TM solo es alcanzable desde TM | `infra-monitoring/docs/runbooks/11_puertos_y_redes.md` §4 y §5 | La consulta se ejecuta en TM. Conectar otro servicio a `ticket-db` desharía `aislar-redes-y-puertos` |
| R2 | Solo los servicios Python llaman al modelo; TM no habla con OCI | `AgentClient`, `classifier.base-url` | Los pasos con LLM van en `ticket-agent` |
| R3 | TM espera al agente 60 s por llamada; el portal espera 90 s en `/api/v1/agent/`; el agente admite 10 peticiones por minuto por tenant | spec `ai-agent-integration`; `frontend/nginx.conf` línea 54; `routes/query.py` | Dos llamadas al agente por pregunta deben caber en esos tiempos. Hay que medirlo |
| R4 | La seguridad por fila depende de una variable de la transacción (`app.tenant_id`) que cualquier consulta puede cambiar con la función `set_config`. TM la usa para fijar el tenant, así que no se puede prohibir en la base | `V34`; `TenantAwareJpaTransactionManager` línea 24 | El SQL del LLM solo puede usar funciones de una lista blanca. Es la comprobación más importante del diseño |
| R5 | No todos los usuarios ven todos los tickets | spec `ticket-visibility` | La función es solo para roles con `VIEW_ALL_TICKETS` y la acción nueva. Los agregados del tablero ya siguen esta idea |
| R6 | Una sesión de soporte no puede usar el agente | spec `support-access` | Tampoco esta función |
| R7 | `ticket_app` puede leer todas las tablas, incluidas contraseñas y tokens | `V33` línea 61 | La consulta no puede correr con ese rol. Hace falta un rol de solo lectura que vea únicamente lo permitido |
| R8 | `due_date` no tiene quién lo escriba, así que SLA y vencidos solo reflejan datos sembrados | spec `dashboard-analytics`, Known Gaps | El catálogo debe decirlo, o esas preguntas quedan fuera del primer alcance |
| R9 | Rocky tiene 2 núcleos y el pool de TM es de 10 conexiones | Medición del 11-11; `application.properties` | Límite de tiempo y de filas por consulta, y una conexión aparte para esta función |

## 4. Dónde vive la función

| Opción | Encaje | Problema |
|---|---|---|
| A. Todo en `ticket-agent`, conectado a la base de TM | Reutiliza el LLM | Rompe R1. El agente tendría credenciales de la base de TM y conocería su esquema |
| B. Servicio nuevo | Aislado | Rompe R1 igual, y suma contenedor, red, llaves y despliegue sin dar nada que no den los repos actuales |
| C. Todo en TM | Respeta R1 | TM tendría que llamar a OCI, y hoy no lo hace (R2). Duplicaría el adaptador en Java |
| **D. Repartida: TM ejecuta, `ticket-agent` redacta** | Respeta R1 y R2 y usa el canal que ya existe | Dos llamadas al agente por pregunta (R3) |

**Recomiendo la D.**

### Flujo propuesto

```
Portal ─► TM  POST /insights/query            (JWT: tenant, rol, permiso, interruptor)
            │
            ├─► ticket-agent  "escribe el SQL"  (pregunta + catálogo + valores del tenant)
            │        └─► devuelve SQL o una petición de aclaración
            │
            ├─ TM valida el SQL                 (una sola consulta, solo vistas del catálogo,
            │                                    lista blanca de funciones)
            ├─ TM ejecuta                       (rol de solo lectura, transacción del tenant,
            │                                    límite de filas y de tiempo)
            ├─ TM audita                        (usuario, pregunta, SQL, duración, filas)
            │
            ├─► ticket-agent  "redacta"         (pregunta + filas)
            │        └─► devuelve la respuesta
            ▼
         Portal muestra respuesta y tabla
```

**Por qué orquesta TM y no el agente:**
- El tenant nunca sale de TM. No hay ida y vuelta en la que el agente pudiera devolver otro tenant.
- El agente no recibe credenciales ni acceso a la base: solo texto.
- Quien es dueño de los datos decide qué se ejecuta. El agente propone.

**Por qué una ruta aparte y no el widget «¿Necesitas ayuda?»:** en la primera versión no hace falta adivinar si la pregunta es de ayuda o de datos. Es más predecible y más fácil de permisar. Unirlas después es posible; el agente ya tiene un enrutador de intención.

## 5. Piezas nuevas y su justificación

| Pieza nueva | Repo | Por qué no basta lo existente |
|---|---|---|
| Rol de base de solo lectura y esquema de vistas `insights` | TM (Flyway) | `ticket_app` ve todo, contraseñas incluidas (R7). Las vistas además dan nombres claros y joins ya resueltos |
| Validador de SQL en Java | TM | La comprobación debe estar donde se ejecuta (R1, R4). Necesita una librería de análisis de SQL; la candidata es JSqlParser, con licencia por confirmar |
| Catálogo: descripción de vistas y columnas, sinónimos, reglas y ejemplos | TM | No existe. Va junto a las vistas que describe, versionado con ellas |
| Ruta `POST /insights/query`, acción de permiso e interruptor | TM | Son piezas nuevas, pero cada una sigue un patrón que ya existe |
| Dos rutas con llave de tenant: escribir SQL y redactar | `ticket-agent` | `/query` es para documentos y responde otra cosa. Se reutilizan el adaptador, la llave, el límite y las métricas |
| Sección en el portal | TM (frontend) | No existe |

**No entra:** servicio nuevo, contenedor nuevo, red nueva, puerto nuevo, base nueva, Wren instalado, grafo, representación intermedia ni compilador.

**Repos que no se tocan:** `notification-service`, `ticket-ingestion-light`, `support-console` e `infra-monitoring`. `ticket-classification` solo si el consumo de la función debe sumarse a su tabla de uso por cliente.

## 6. Qué cambia frente a lo conversado antes

| Antes | Ahora | Motivo |
|---|---|---|
| Rol de PostgreSQL por tenant | Un solo rol de solo lectura; el tenant lo filtra la seguridad por fila que ya existe | Hallazgo 1 |
| Validación en Python con SQLGlot, copiando código de Wren | Validación en Java, dentro de TM | R1 y R4. De Wren se portan las **listas y reglas** (funciones permitidas, sentencias prohibidas, acotado del resultado), no el código tal cual |
| POC en repo y contenedor aparte, sin tocar TM | Cambio OpenSpec en TM y en `ticket-agent`, con el interruptor apagado | Un POC fuera de TM no puede alcanzar la base (R1) ni probar la seguridad por fila real |
| Servicio `ticket-insights` | No hay servicio nuevo | Opción D |

Lo de Wren que sí se conserva: las seis piezas de exactitud, los pares pregunta→SQL confirmados, la lista de huecos semánticos como checklist y sus reglas de redacción de SQL para el prompt del agente.

## 7. Abierto para el paso B

1. **Cómo aplica la seguridad por fila a través de las vistas.** Depende de quién sea el dueño de la vista y de si se declara con permisos de quien consulta. Solo se confirma ejecutando.
2. **Tiempo total.** Dos llamadas al agente más la consulta deben caber en los 90 s del portal. Si no caben, hay que ajustar ese límite o partir la respuesta.
3. **Cómo se descarta si no funciona.** Con el interruptor apagado no afecta a nadie. Quitarlo del todo es revertir el cambio y una migración que borre el esquema `insights` y el rol.
4. **Dónde va en el portal:** dentro de «Métricas» o como sección propia.
5. **Consumo por cliente:** si las llamadas de esta función se suman a la tabla de uso que ve el operador.
6. **Preguntas de SLA y vencidos:** entran con aviso o quedan fuera hasta que `due_date` tenga quién lo escriba (R8).
7. **Nombres y correos de personas en los resultados que llegan a OCI.**
