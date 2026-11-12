# GemIA — Preguntas sobre los datos: plan versión 4

**Fecha:** 2026-11-12
**Estado:** reparto y decisiones de negocio aprobados. Etapas 1 y 2 hechas el 11-12; las propuestas OpenSpec de la etapa 2 tienen el visto bueno del dueño (2026-11-12). Sigue la etapa 3 (compuerta 1). **No se escribe código sin hablarlo antes con el dueño.**
**Sustituye a:** las versiones 3 y 3.1 de este documento y a `PLAN_TEXT_TO_SQL_WREN.md`.
**Documentos de apoyo:** `ETAPA_1_ALCANCE_Y_CATALOGO.md` (vistas, significados y preguntas de referencia), `ETAPA_1_DEFINICION_DE_VISTAS.md` (columnas confirmadas y definición de cada vista), `ETAPA_2_DISENO.md` (mapa de las tres propuestas OpenSpec, decisiones que cruzan repos, tiempos, casos negativos y diferencias con este plan), `NOTAS_DE_DISENO.md` (diseño técnico razonado), `PASO_A_REVISION_STACK.md` (cómo está el stack) y `ETAPA_1_REVISION_WREN.md` (qué se aprendió de Wren).

---

## 1. Qué es

Una función contratable por cliente: un usuario con permiso escribe una pregunta sobre los tickets de su empresa ("¿cuántos tickets abiertos tiene Nómina este mes?") y recibe la respuesta en texto, con la tabla de datos que la respalda.

**Entra en la primera versión:** preguntas sobre tickets activos, borradores y archivados, con sus estados, áreas, categorías, prioridades, responsables y fechas: conteos, listados, filtros, agrupaciones y tiempos.

**No entra en la primera versión:**
- Texto de comentarios y de correos, y datos del solicitante.
- Preguntas de SLA y vencidos: hoy `due_date` no tiene quién lo escriba.
- Detección automática de si la pregunta es de ayuda o de datos.
- Guardar una respuesta como ejemplo confirmado desde el portal.

## 2. Principio y reparto

**Principio:** la consulta se ejecuta donde viven los datos y con los permisos de quien pregunta. El componente de IA propone el SQL y nunca entra a la base. Es como lo resuelven Oracle Select AI y Snowflake Cortex Analyst. No se abre ninguna excepción a "la base de TM solo se alcanza desde TM".

| Repo | Responsabilidad | Alcance en esta implementación |
|---|---|---|
| **Motor nuevo** (`text-to-sql`, nombre provisional) | Todo el conocimiento y la lógica de la función | Catálogo de cada aplicación, escritura y validación del SQL, filtro obligatorio, redacción de la respuesta, proveedores de modelo, métricas técnicas |
| **`ticket-management` (TM)** | Dueño de los datos de tickets, de los permisos y del portal | Rol de solo lectura y vistas, un ejecutor pequeño y genérico, registro y consumo, permiso por rol, interruptor del operador, panel y ayudas |
| **`infra-monitoring`** | Orquestación y monitoreo | Sumar el motor al arranque, los chequeos y las métricas |
| **`stack_ticket_doc`** | Documentación | Guías de usuario y copias de runbooks |
| `ticket-agent`, `ticket-classification`, `notification-service`, `ticket-ingestion-light`, `support-console` | — | No se tocan |

### Flujo

```
Portal ─► TM  POST /insights/query        (JWT: cliente y rol; interruptor; permisos)
            │   TM lee los valores del cliente (nombres de estados, áreas, responsables)
            │
            ├─► motor  POST /v1/sql        (aplicación, pregunta, valores, objetos permitidos)
            │        └─► SQL validado con filtros obligatorios y parámetros sin valor,
            │            o petición de aclaración, o "fuera de alcance"
            │
            ├─ ejecutor de TM              (pone el cliente del token en los parámetros;
            │                               rol de solo lectura; transacción del cliente;
            │                               límite de filas y de tiempo)
            ├─ TM registra la pregunta
            │
            ├─► motor  POST /v1/answer     (pregunta + filas)
            │        └─► texto de la respuesta
            ▼
         Portal: respuesta, tabla y periodo
```

### Por qué así

- **TM no carga el conocimiento de la función.** No sabe de catálogos, de modelos ni de cómo se escribe el SQL. Solo ejecuta lo que llega validado y aplica su propia seguridad.
- **El motor no recibe credenciales ni entra a la red de la base.** Tampoco guarda datos de ningún cliente.
- **El cliente sale del token y nunca sale de TM** como dato en el que confiar: el motor devuelve un parámetro vacío y TM lo llena.
- **La frontera final es la base de TM.** Aunque el motor fallara, la base no entrega filas de otro cliente ni columnas no permitidas.
- **La tabla es la verdad y el texto la acompaña.** Si la redacción falla o tarda de más, TM entrega la tabla con una frase fija.
- **Otra aplicación lo adopta** entregando su catálogo al motor y montando su propio ejecutor pequeño.

## 3. Reglas de la función

### Contratación: interruptor del operador

- Es una función más de "Funciones de IA" en la consola del operador: **"Preguntas sobre los datos"**, por cliente, con motivo, línea en el historial de IA y registro en auditoría.
- **Nace apagada** para clientes nuevos y existentes. Las demás funciones nacen encendidas.
- Apagada: nadie del cliente la ve, la ruta responde `403` y TM no llama al motor.
- El administrador del cliente la ve en solo lectura.
- No existe apagado global: se controla cliente por cliente. Así se representan los planes contratados.

### Permisos por rol

Acción nueva en la matriz, grupo «Insights»: `ASK_DATA` — "Preguntar por los datos".

| Rol | Valor inicial | ¿Lo cambia el administrador? |
|---|---|---|
| ADMIN | Encendido | No, queda fijo |
| MANAGER | Apagado | Sí |
| USER | Apagado | No, nunca se le puede dar |

| Para preguntar sobre | Se necesita |
|---|---|
| Tickets activos | `ASK_DATA` y `VIEW_ALL_TICKETS` |
| Borradores y archivados | Lo anterior y además `REVIEW_DRAFTS` |

- Con el interruptor apagado, la acción cuenta como negada para todos, no aparece en el editor de permisos y los valores guardados no cambian.
- La migración inserta la acción para los clientes existentes, y el alta de clientes la incluye para los nuevos.

### Otras reglas

- **Sesión de soporte:** no puede usar la función.
- **Registro y consumo:** TM guarda cada pregunta (cliente, usuario, fecha, texto, SQL, resultado, filas, duración, proveedor y modelo). Con eso muestra el consumo por cliente y por mes en la consola del operador. Se borra y se exporta con el cliente.
- **Ventana:** el mismo panel «¿Necesitas ayuda?», con dos modos que elige el usuario: "Ayuda del portal" y "Mis datos". Cada modo aparece solo si el cliente lo tiene contratado y el rol tiene permiso.
- **Sin periodo en la pregunta:** se usa todo el histórico y la respuesta lo dice.
- **Tiempo de resolución:** como el tablero, sobre los tickets creados en el periodo.
- **Nombres de personas:** al modelo llegan los nombres de los responsables y los títulos de los tickets. No llegan correos ni datos del solicitante.

## 4. Seguridad: capas, de afuera hacia adentro

| # | Capa | Dónde | Qué impide |
|---|---|---|---|
| 1 | Interruptor del cliente y permisos del rol | TM | Que use la función quien no la contrató o no tiene permiso |
| 2 | El cliente y el rol salen del token | TM | Que la petición o el modelo elijan el cliente |
| 3 | TM dice al motor qué objetos puede usar ese rol | TM y motor | Preguntar por borradores o archivados sin `REVIEW_DRAFTS` |
| 4 | Validación del SQL: una sola consulta de lectura, solo objetos del catálogo, sin nombres con esquema, lista blanca de funciones, columnas que existen | Motor | Escrituras, tablas fuera del catálogo y funciones peligrosas como `set_config` |
| 5 | Filtro obligatorio por objeto, agregado siempre por el motor; TM pone el valor del cliente | Motor y TM | Ver filas de otro cliente aunque la base no filtrara |
| 6 | Guardas sin analizador en el ejecutor: sin `;`, largo máximo, consulta envuelta con límite de filas | TM | Dos sentencias en una y resultados gigantes, aunque el motor fallara |
| 7 | Rol de base de solo lectura que solo ve las vistas `insights` | Base de TM | Leer contraseñas, tokens o tablas no permitidas |
| 8 | Seguridad por fila que ya existe | Base de TM | Ver filas de otro cliente |
| 9 | El rol de solo lectura no puede cambiar el cliente de la transacción | Base de TM | Que una consulta se salte la capa 8 |
| 10 | Límite de tiempo y transacción de solo lectura, fijados en el rol | Base de TM | Consultas que cuelguen la base |
| 11 | Registro de cada pregunta | TM | Uso sin rastro |

**Lo más delicado es la capa 9.** La seguridad por fila depende de una variable que hoy cualquier consulta puede cambiar con la función `set_config`. Hay dos formas de cerrarlo en la base, descritas en `NOTAS_DE_DISENO.md`; la compuerta 1 decide cuál.

## 5. Piezas por repo

### Motor nuevo

| Pieza | Detalle |
|---|---|
| Catálogos | Una carpeta por aplicación. La de TM describe las vistas: columnas, significados, sinónimos, relaciones, reglas, ejemplos pregunta→SQL y el filtro obligatorio de cada objeto. Formato publicado y versionado |
| `POST /v1/sql` | Aplicación + pregunta + valores del cliente + objetos permitidos → SQL validado con parámetros con nombre, aclaración o "fuera de alcance". Un reintento interno. Acepta, opcionales, el error que devolvió la base y la pregunta anterior |
| `POST /v1/answer` | Pregunta + filas → texto |
| `POST /v1/validate` | Valida un SQL contra un catálogo y devuelve el SQL final con sus filtros, sin modelo |
| `GET /v1/catalogs/{aplicación}` | Lista los objetos y columnas que el motor espera. La aplicación lo compara contra su base en una prueba automática |
| Validador y filtros | Python con SQLGlot; reglas portadas de Wren con su aviso de licencia. Es la pieza más delicada del repo |
| Modelo | Varios proveedores con la variable `PROVIDER` (`oci`, `ollama`), con los adaptadores copiados tal cual de `ticket-agent/src/adapters/`. Cada respuesta informa proveedor y modelo |
| Llaves | Una por aplicación, con su cupo y sus métricas |
| Estado | Ninguno: sin base, sin datos de clientes, sin credenciales de bases ajenas |
| Operación | `/health`, `/metrics`, identificador de petición, logs sin preguntas ni filas. Un contenedor liviano en la red compartida; puerto solo en `127.0.0.1` |
| Adopción | Guía de pasos para una aplicación nueva, ejecutor de ejemplo y paquete de pruebas que esa aplicación corre contra su propia base |

### `ticket-management`

| Pieza | Patrón que sigue |
|---|---|
| Migración: roles `ticket_insights` y `ticket_insights_owner`; esquema `insights` con las vistas de tickets, borradores, archivados, cambios de estado y catálogos del cliente; columna `tenants.ai_data_questions` (apagada); acción `ASK_DATA`; tabla `insights_questions` | Flyway; roles como `ticket_platform`; acción como `V64`; función como `V63` |
| Ejecutor: conexión propia con el rol de solo lectura; recibe SQL y parámetros, ejecuta dentro de la transacción del cliente y acota el resultado | `PlatformDatabase` |
| Ruta `POST /insights/query` y cliente HTTP del motor | `AgentController` y `AgentClient` |
| Lectura de los valores del cliente | Servicios existentes |
| Interruptor del operador, historial, auditoría y consumo | `TenantPlatformService`, `AiHistory`, "AI usage of a client" |
| Acción en la matriz, oculta con el interruptor apagado | `PermissionService`, `DefaultPermissions` |
| Tabla de preguntas con seguridad por fila, incluida en el borrado y la exportación | `ticket_decision_traces` |
| Portal: modos del panel, fila en el editor de permisos, vista de solo lectura del administrador | `AgentPanel`, `PermissionManagement` |
| Tiempo de espera propio para la ruta en nginx | `/api/v1/agent/` |
| Pruebas: aislamiento entre clientes, permisos, interruptor, y que las vistas coincidan con lo que el motor espera | `RowLevelSecurityIntegrationTest`, `CrossTenantEndpointsIntegrationTest` |

### Ayudas, runbooks y OpenSpec

| Qué | Dónde | Etapa |
|---|---|---|
| Ayuda «?» de cada pantalla que cambia: panel del asistente con el modo "Mis datos", editor de permisos, sección «IA» del administrador, y en la consola el detalle del cliente ("Funciones de IA" y consumo) | TM, `helpContent.ts`, con el skill `regenerar-ayudas` | 8 |
| Guías de usuario por rol | `stack_ticket_doc/ticket-management/docs/usuarios/`, y re-ingesta en el asistente | 9 |
| Runbooks del motor: 01 para principiantes, 02 arquitectura técnica, 03 guía operacional | Repo del motor y su copia en `stack_ticket_doc` | 4 y 9 |
| Runbook operacional de la función: encender por cliente, rotar la contraseña del rol, qué hacer si el motor no responde, cómo quitarla | TM, `docs/runbooks/` | 5 y 9 |
| Runbook de puertos y redes y guía operacional del ecosistema | `infra-monitoring` | 9 |
| OpenSpec en el motor desde su primer commit: `openspec/specs/`, skills en `.claude/skills` y `CLAUDE.md` | Repo del motor | 2 y 4 |
| Cambio OpenSpec en TM (capacidad nueva y deltas de `rbac-permissions`, `platform-operator`, `help-and-agent-ui`, `multi-tenancy`, `data-model`, `stack-deployment`) y en `infra-monitoring` | TM e `infra-monitoring` | 2; archivado en la 9 |

## 6. Qué se toma de Wren

Wren no se instala. Su licencia (Apache-2.0) permite copiar código conservando el aviso de origen.

| Qué | De dónde | Cómo |
|---|---|---|
| Una sola consulta de lectura y lista blanca de funciones | `core/wren/src/wren/read_only.py` | Se porta al validador |
| Sentencias prohibidas y lectores de archivos bloqueados | `core/wren/src/wren/policy.py` | Se porta al validador |
| Acotado del resultado | `core/wren/src/wren/governed_query.py` | Se aplica en el ejecutor |
| Filtro por fila declarado en el catálogo | Su control de acceso por fila y `mdl/cte_rewriter.py` | Se rehace en pequeño: un filtro por objeto, siempre obligatorio |
| Catálogo con reglas en texto y pares pregunta→SQL | Estructura `knowledge/` | Forma del catálogo |
| Lista de huecos semánticos | `gap_catalog.md` | Checklist al armar el catálogo |
| Seis piezas de exactitud y reglas de redacción de SQL | `architecture.md` y sus prompts | Guía del diseño y base del prompt |

## 7. Plan por etapas

> **No se escribe código sin hablarlo antes.** Antes de cada etapa se avisa para ajustar el esfuerzo del modelo.
> Los tamaños son estimaciones con margen amplio, no compromisos.

| # | Etapa | Qué incluye | Entregable | Tamaño | Esfuerzo | OCI | ¿Código? |
|---|---|---|---|---|---|---|---|
| 1 | Alcance y catálogo inicial — **hecha el 11-12** | Vistas lógicas, significados, preguntas de referencia, columnas reales de TM confirmadas y definición de cada vista (`ETAPA_1_DEFINICION_DE_VISTAS.md`). Los resultados esperados de las preguntas pasaron a la etapa 3 por decisión del dueño (11-12): se calculan cuando las vistas existan | Definición de vistas; preguntas de referencia | Hecha | Alto | No | No |
| 2 | Diseño y propuestas OpenSpec — **hecha y aprobada el 11-12** | Tres cambios escritos y validados: `preguntas-sobre-los-datos` (TM), `motor-inicial` (motor, provisionalmente en `stack_ticket_doc/text-to-sql/openspec/`) y `motor-en-el-ecosistema` (`infra-monitoring`). Mapa y casos negativos en `ETAPA_2_DISENO.md` | Cambios OpenSpec listos para aprobar | Hecha | **Extra alto** | No | No |
| 3 | **Compuerta 1:** aislamiento en la base | Rama de TM con la migración y las pruebas de los casos negativos (B1 a B14 de `ETAPA_2_DISENO.md` §8). Resultados esperados de las preguntas de referencia. Se corre en Rocky en una carpeta temporal, sin tocar el stack, con la suite completa de TM | Evidencia pasa/falla; variante elegida. **Si no hay forma de aislar, se detiene aquí sin nada fusionado** | 1–2 días | **Alto** | No | Sí, en rama |
| 4 | Motor | Repo nuevo con su plantilla y OpenSpec. Catálogo de TM. Validador portado de Wren. Filtro obligatorio y parámetros. Adaptadores de modelo. Llave por aplicación. Cuatro rutas. Pruebas con todos los casos negativos, sin modelo. Runbooks. Contenedor | Motor corriendo en local y en Rocky | 6–7 días | **Alto**; validador y filtros, **extra alto** | Poco | Sí |
| 5 | TM, parte de servidor | Migración definitiva. Ejecutor. Ruta y cliente del motor. Valores del cliente. Permisos. Interruptor del operador. Registro y consumo. Pruebas de aislamiento, permisos, interruptor, sesión de soporte y coincidencia con el catálogo | Función disponible por API, apagada para todos | 3–4 días | **Alto**; ejecutor y aislamiento, **extra alto** | No | Sí |
| 6 | **Compuerta 2:** punta a punta | En Rocky, encendida solo en el cliente de prueba. Casos negativos con el modelo. El filtro del motor a solas, con la seguridad por fila apagada en una base de prueba. Muestra de las preguntas de referencia. Tiempos por paso | Seguridad al 100 %; exactitud y latencia medidas | 1–2 días | **Alto** | **Sí (muestra)** | Ajustes |
| 7 | Dictamen | Seguir, ajustar o descartar | Decisión documentada | Horas | Alto | No | No |
| 8 | Portal y consola del operador | Modos del panel. Fila en el editor de permisos. Función y consumo en "Funciones de IA". Vista de solo lectura del administrador. Ayudas «?» | Función usable en el portal | 3–4 días | **Alto** | Poco | Sí |
| 9 | Cierre | Motor en la orquestación y el monitoreo. Runbooks. Guía de adopción y paquete de pruebas. Guías de usuario y re-ingesta. Corrida completa. Archivado OpenSpec | Función lista para ofrecer; motor listo para una segunda aplicación | 3–4 días | Medio; la corrida final, alto | **Sí (completa)** | Poco |

**Total estimado:** de 4 a 5 semanas y media. Frente a la versión 3.1 el total no cambia: el motor crece porque ahora lleva el catálogo, y TM se reduce.

**Compuertas:**
- **Compuerta 1 (etapa 3):** si la base no puede garantizar el aislamiento, el plan se detiene antes de construir el motor.
- **Compuerta 2 (etapa 6):** la seguridad debe pasar al 100 %. La pantalla no se construye hasta el dictamen.

### Cómo se descarta

| Momento | Qué hay que deshacer |
|---|---|
| Antes de la etapa 5 | Nada en producción: se borran las ramas y el repo del motor |
| Después de la etapa 5 | La función está apagada para todos. Para quitarla del todo: revertir los cambios, una migración que borre el esquema `insights`, los dos roles, la columna y la tabla, y apagar el contenedor del motor |

## 8. Costo

| Concepto | Costo |
|---|---|
| Licencias | Cero |
| Infraestructura | Un contenedor liviano más en Rocky, sin base ni volúmenes. Antes de desplegarlo conviene limpiar imágenes viejas (13.8 GB recuperables, medido el 11-11) |
| OCI | Dos llamadas por pregunta, tres si hay reintento. En las etapas, solo muestras; la corrida completa va al final |
| Operación | Una llave de servicio, una contraseña de base y un contenedor más que vigilar |

## 9. Decisiones de negocio (cerradas el 2026-11-12)

| # | Decisión | Resultado |
|---|---|---|
| 1 | Nombre | "Preguntas sobre los datos" en la consola; modo "Mis datos" en el panel |
| 2 | Permiso inicial por rol | ADMIN fijo, MANAGER apagado y concedible, USER nunca |
| 3 | Nombres de personas hacia el modelo | Responsables y títulos sí; correos y datos del solicitante no |
| 4 | SLA y vencidos | Fuera de la primera versión |
| 5 | Guardar las preguntas | Sí, en TM |
| 6 | Tiempo de resolución de un periodo | Como el tablero: tickets creados en el periodo |
| 7 | Pregunta sin periodo | Todo el histórico, y la respuesta lo dice |
| 8 | Borradores y archivados | Entran, en vistas propias y solo con `REVIEW_DRAFTS` |
| 9 | Registro y consumo | En TM; el motor solo guarda métricas técnicas |

Pendiente: confirmar el nombre del repo del motor (`text-to-sql`).

## 10. Riesgos técnicos abiertos

| Riesgo | Se cierra en |
|---|---|
| Que quitar `set_config` a todos los roles rompa algo de TM | Etapa 3 |
| Que la seguridad por fila no aplique como se espera a través de las vistas | Etapa 3 |
| Que al filtro obligatorio del motor se le escape algún caso | Etapas 4 y 6. En TM no es la única barrera |
| Que el catálogo del motor se desalinee de las vistas de TM, al estar en repos distintos | Etapa 5, con la prueba automática que los compara |
| Que dos llamadas al modelo más la consulta no quepan en el tiempo del portal | Etapa 6 |
| Exactitud del modelo `openai.gpt-oss-20b` escribiendo SQL | Etapa 6 |

## 11. Seguimientos fuera de este plan

| Seguimiento | Por qué no entra ahora |
|---|---|
| Unificar el acceso a los modelos: `ticket-agent` (dos proveedores, dentro del proceso) y `ticket-classification` (pasarela `langchain-api`, cinco proveedores, texto libre) lo resuelven distinto. Con el motor serán tres. Caminos: librería compartida o una sola pasarela, que además permitiría medir tokens en un lugar | Toca dos repos en uso y no hace falta para esta función |
| Detección automática de si la pregunta es de ayuda o de datos | Hacen falta preguntas reales para medir si acierta |
| Guardar una respuesta como ejemplo confirmado desde el portal | Mejora posterior |
| Preguntas de SLA y vencidos | Depende de que `due_date` tenga quién lo escriba |
| Otros dialectos de base, librería cliente y borrador automático de catálogo | Cuando llegue la segunda aplicación; no cambian el contrato |

## 12. Historia de las versiones

| Versión | Cambio principal |
|---|---|
| 1 y 2 | Instalar Wren como motor. Descartado tras leer su código |
| 3 | Motor propio; TM con catálogo, validación y ejecución |
| 3.1 | Filtro obligatorio, llave por aplicación, varios proveedores, guía de adopción |
| 4 | El catálogo y toda la lógica pasan al motor. TM queda con el ejecutor, el registro, los permisos, el interruptor y el panel. El motor nunca entra a la base. Borradores y archivados entran |
