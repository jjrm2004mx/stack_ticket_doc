# GemIA — Preguntas en lenguaje natural sobre los datos (Text-to-SQL)

> **Sustituido el 2026-11-11 por `PLAN_PREGUNTAS_SOBRE_LOS_DATOS.md` (versión 3).** Wren ya no se instala. Este documento se conserva como registro de las versiones 1 y 2.

**Fecha:** 2026-11-11
**Versión del plan:** 2 (segunda verificación; reemplaza a la tabla aprobada el mismo día)
**Estado:** Decisión de rumbo tomada; **sin código**. Tabla v2 aprobada. **Etapa 1 terminada el 2026-11-11** (`ETAPA_1_REVISION_WREN.md`); sus decisiones D1 a D5 esperan visto bueno.
**Antecedentes:** `GEMIA_Plan_Maestro_Catalogo_Semantico_Text_to_SQL_Fases_1_a_8.md` y `GEMIA_Evaluacion_WrenAI_Candidato_POC_v1.0.md`.

---

## 1. Decisión

**Tomar Wren AI como base. No construir desde cero.**

Wren AI (`Canner/WrenAI`) se usa como **librería** dentro de un servicio Python propio, delgado y aislado. Su nombre provisional es `ticket-insights`.

- **Catálogo semántico:** el MDL de Wren (YAML), con `knowledge/rules/` para las reglas de negocio y `knowledge/sql/` para los pares pregunta→SQL confirmados.
- **Motor:** `wren-core`, que expande modelos, relaciones y joins; la política SQL y el conector a PostgreSQL.
- **Lo que sigue siendo de GemIA:** identidad, rol, tenant, auditoría, límites y el apagado cuando el cliente tiene la IA desactivada.

Esta decisión **reemplaza** la IR y el compilador propios del plan maestro. El LLM escribe SQL **contra los modelos del MDL**, nunca contra las tablas. Wren lo expande, le aplica el control por fila y la política, y lo traduce a PostgreSQL.

## 2. Evidencia revisada en el código de Wren

Copia leída en el commit `6e8ac81` (2026-11-10). Fue una lectura estática: no se instaló ni se ejecutó nada.

| Tema | Hallazgo | Archivo |
|---|---|---|
| Licencia | `core/`, `sdk/`, `skills/` y `examples/` son Apache-2.0; `docs/` es CC-BY-4.0. Hay un `LICENSE-AGPL-3.0` reservado para módulos futuros. | `LICENSE` |
| Aislamiento por tenant | El MDL tiene control de acceso por fila y columna (`row_level_access_controls`, CLAC). Hay un ejemplo multitenant con `session_tenant_id`. | `core/wren-core/wren-example/examples/row-level-access-control.rs` |
| Tenant desde nuestro código | `query`, `dry_plan` y `dry_run` del toolkit reciben `properties`. El filtro por fila se inyecta al planear. | `core/wren/src/wren/engine.py`, `sdk/wren-langchain/src/wren_langchain/_toolkit.py` |
| Edición abierta vs. de pago | La abierta incluye RLAC/CLAC en el MDL. La de pago agrega cuentas, roles, SSO, RLS por usuario, auditoría, MCP/REST hospedado y UI. No la necesitamos. | `docs/core/concepts/oss_vs_commercial.md` |
| Política SQL | El bloqueo de escrituras está siempre activo (lista blanca sobre el AST). El `strict_mode` (solo tablas del MDL; bloquea `dblink` y lectores de archivos) es opcional y **se activará siempre**. | `core/wren/src/wren/policy.py` |
| Concurrencia | El toolkit es síncrono y usa un `Lock`. Hay que medirlo. | `_toolkit.py` |

### Hallazgos de la segunda verificación (cambian el plan)

| # | Hallazgo | Evidencia | Consecuencia |
|---|---|---|---|
| H1 | Las herramientas que ve el LLM (`wren_query(sql, limit)`, `wren_dry_plan(sql)`) **no reciben ni reenvían `properties`**. | `sdk/wren-langchain/src/wren_langchain/_tools.py` | El LLM no puede fijar el tenant (bien). Pero las herramientas de fábrica **no sirven con control por fila**: hay que escribir herramientas propias que tomen el tenant del contexto de la petición. Es poco código, pero es código de seguridad. |
| H2 | Si la regla declara una propiedad obligatoria y no llega, el planeador **falla** ("required … but not found"). Hay una rama distinta para propiedades **no** obligatorias. | `core/wren-core/core/src/logical_plan/analyze/access_control/mod.rs` | Falla cerrado solo si la propiedad se declara obligatoria. Hay que comprobar qué pasa cuando no lo es, y prohibirlo en nuestro MDL. |
| H3 | El valor de la propiedad se pasa **como texto SQL** (el ejemplo manda `"'1acdef01-…'"` con sus comillas). | `row-level-access-control.rs` | El valor se interpola en la consulta. El tenant debe validarse como entero estricto en nuestro código antes de pasarlo. |
| H4 | El control por fila casi no aparece en la referencia del MDL para proyectos YAML; lo documentado y probado está en el núcleo Rust. | `docs/core/reference/mdl.md`, `core/wren-core/core/tests/access_control.rs` | El camino YAML → Python → control por fila hay que **probarlo de punta a punta antes** de invertir en el catálogo. |
| H5 | Wren publica una versión menor casi cada semana (0.16.0 el 16-oct, 0.17.0 el 23-oct, 0.17.1 el 30-oct, 0.18.0 el 10-nov) y sigue en 0.x. | `core/wren/CHANGELOG.md` | Fijar versión exacta. Candidata: 0.17.1, si incluye el paso de `properties`. La 0.18.0 agrega un agente y un MCP que no usaremos. |
| H6 | El control por fila de Wren es una capa de aplicación. Si tuviera un fallo, el usuario de solo lectura vería todos los tenants. | Diseño | Hace falta una **segunda capa en PostgreSQL**, independiente de Wren (§4). |

**Buenas prácticas que adoptamos, con o sin Wren** (`docs/core/reference/architecture.md`): elegir solo el esquema relevante, perfilar valores (qué significa cada estado), detectar ambigüedad, mostrar la traza (`dry-plan`), reintentar y reparar con errores estructurados, y evaluación golden contra regresiones. La lista de huecos semánticos (`skills_content/enrich-context/references/gap_catalog.md`) se usa como checklist.

## 3. Costo

| Concepto | Costo |
|---|---|
| Licencia | **Cero.** Apache-2.0; la edición de pago no hace falta. |
| Infraestructura | **Baja.** Un contenedor más en Rocky, entre cientos de MB y 1–2 GB de imagen (cifra a medir). Sin base de datos nueva y sin LanceDB al inicio. |
| Rocky (medido el 11-11) | `/home`: 9.8 GB libres (78 % usado); 13.8 GB recuperables en imágenes sin uso. RAM: 9.3 GB disponibles. **CPU: 2 núcleos.** Antes del primer contenedor se limpia con `podman image prune -a`, con autorización. |
| Llamadas a OCI | **El costo variable real.** De 2 a 5 llamadas por pregunta. Muestra en el POC; corrida completa (30–100 preguntas) solo al final. |
| Trabajo | El POC completo (etapas 1 a 9) se estima en **unas 2 semanas hábiles**, no en "pocos días". Ver §6. |

## 4. Reglas de seguridad (no negociables)

1. El tenant y el rol salen del token de GemIA y **nunca** del LLM. El LLM solo ve herramientas propias que no aceptan tenant como argumento (H1).
2. El tenant se valida como **entero estricto** antes de entrar a Wren (H3).
3. Toda regla de control por fila declara su propiedad como **obligatoria**; sin tenant, la consulta falla (H2).
4. **Doble capa de aislamiento** (H6):
   - Capa 1, PostgreSQL: el usuario de solo lectura no tiene permiso sobre las tablas de TM. Solo lee vistas en un esquema aparte (`insights`), que filtran por el tenant fijado en la sesión y exponen solo las columnas permitidas. Se borra con `DROP SCHEMA insights CASCADE`.
   - Capa 2, Wren: control por fila en el MDL sobre esas vistas.
   - Si en la etapa 1 resulta que el conector de Wren no permite fijar la variable de sesión, la capa 1 se rediseña en la etapa 3; no se elimina.
5. Comprobación determinista antes de ejecutar: el SQL expandido (`dry_plan`) debe contener el filtro del tenant; si no, no se ejecuta.
6. `strict_mode` siempre encendido. Límite de filas, timeout y auditoría (usuario, pregunta, SQL, duración y resultado).
7. **Datos que llegan al LLM:** los resultados de la consulta se envían a OCI para redactar la respuesta. Las vistas excluyen contraseñas, llaves, tokens y cuerpos de correo. Nombres y correos de personas: se decide en la etapa 3.
8. **Alcance por rol:** el POC y la primera versión son solo para el **administrador del tenant**. Los demás roles tienen reglas de visibilidad propias dentro del tenant y quedan para después. Soporte nunca ve datos de negocio.
9. Sin MCP y sin GenBI. Cliente con IA apagada: la función no existe para él.
10. El texto de los tickets es contenido no confiable: puede traer instrucciones. Las capas anteriores limitan el daño a datos del propio tenant; el agente no tiene herramientas de escritura.

## 5. POC descartable: cómo se borra sin dejar basura

**Aislamiento desde el inicio:**
- Un repo nuevo y aparte (`ticket-insights-poc`). No se toca TM, `ticket-agent` ni ningún repo existente.
- Un contenedor propio con su propia red de compose y **sin puertos publicados a la LAN**.
- En PostgreSQL, solo dos objetos nuevos: el usuario de solo lectura y el esquema `insights` con sus vistas.
- Sin migraciones Flyway, sin tablas nuevas y sin cambios en las tablas de TM.
- Sin cambios en el portal ni en los otros servicios.
- Las llaves de OCI en un `.env` del propio repo, que está en el gitignore.

**Si el POC falla, se borra así:**
1. `podman-compose down --rmi all`, que apaga el contenedor y libera la imagen.
2. `DROP SCHEMA insights CASCADE` y `DROP ROLE` del usuario de solo lectura.
3. Archivar o borrar el repo del POC.
4. Borrar la copia local de Wren de la carpeta temporal.

**Lo que se conserva aunque se descarte:** el inventario del esquema de TM, el MDL, el set de preguntas con su SQL esperado y el informe de por qué falló. Todo queda como documentación en este repo.

**Motivos para descartar Wren y volver a construir desde cero:**
- El control por fila no aísla a los tenants en las pruebas negativas (compuerta 1, etapa 4).
- El MDL no puede expresar los estados configurables por tenant.
- No funciona con OCI sin forzarlo, o el ciclo completo no cabe en el tiempo de espera del portal (60 s).

## 6. Plan y esfuerzo requerido (versión 2)

> **Nada se programa hasta aprobar esta tabla.** Antes de cada etapa se avisa para ajustar el esfuerzo del modelo.
> Los tamaños son estimaciones con margen de error, no compromisos.

**Qué cambió frente a la versión 1:**
- La prueba de aislamiento pasa **antes** de armar el catálogo (era la etapa 5; ahora es la 4, con un modelo mínimo). Si Wren no aísla, se descarta sin haber invertido en el MDL.
- Se agrega la revisión del esquema real de TM como etapa propia (la 2): el diseño del aislamiento depende de ella.
- La etapa 1 sube de esfuerzo medio a alto: ahora verifica supuestos de seguridad (H1 a H5).
- El montaje incluye herramientas propias (H1) y la doble capa (H6).
- Las pruebas negativas se corren dos veces: sin LLM (etapa 4) y con el LLM intentando saltarse el filtro (etapa 7).

| # | Etapa | Qué incluye | Entregable | Tamaño | Esfuerzo | OCI | ¿Código? |
|---|---|---|---|---|---|---|---|
| 0 | Aprobación del plan v2 | Revisar hallazgos, reglas y tabla | Visto bueno | — | — | No | No |
| 1 | Revisión dirigida de Wren — **hecha el 11-11** | Fijar versión exacta (H5). Control por fila desde YAML de punta a punta (H4). Qué pasa sin propiedad o con propiedad no obligatoria (H2). Cómo se interpola el valor (H3). Si el conector permite fijar una variable de sesión de PostgreSQL (capa 1). Compatibilidad del agente con el adaptador OCI. Configuración de `strict_mode`. Licencias de las dependencias del extra de PostgreSQL | Informe con evidencia por punto + versión fijada | 0.5–1 día | **Alto** | No | No |
| 2 | Revisión del esquema real de TM | Solo lectura de migraciones y entidades: qué tablas tienen tenant y cuáles lo heredan de su padre, cómo se guardan los estados configurables, columnas sensibles, fechas y zona horaria | Inventario del alcance del POC (3–4 tablas) | 0.5–1 día | **Alto** | No | No |
| 3 | Diseño del aislamiento | Doble capa (§4), vistas y columnas expuestas, validación del tenant, herramientas propias, alcance solo administrador, qué datos llegan al LLM, **lista de casos negativos** | Documento de diseño + casos negativos | 1–2 días | **Extra alto** | No | No |
| 4 | Prueba mínima de aislamiento — **compuerta 1** | Limpieza de imágenes en Rocky (con autorización). Repo y contenedor aislados. Modelo mínimo de 2 vistas con control por fila. Casos negativos **sin LLM**: sin tenant, tenant ajeno, tenant mal formado, tabla fuera del MDL, escritura, subconsultas y uniones que intenten cruzar tenants | Evidencia pasa/falla. **Si falla algo: se descarta Wren aquí** | 1–2 días | **Alto** | No | **Sí**, con visto bueno |
| 5 | MDL y reglas de TM (alcance POC) | Modelos, relaciones, reglas en `knowledge/rules/`, estados configurables como contexto por tenant, checklist `gap_catalog`. Set golden: ~10 preguntas funcionales con SQL esperado y resultado esperado en **dos tenants**, más preguntas ambiguas y fuera de alcance | MDL + set golden | 2–3 días | **Extra alto** | No | YAML/doc |
| 6 | Montaje del agente | Herramientas propias con el tenant del contexto, comprobación del filtro en el SQL expandido, conexión a OCI, límites, timeout y traza | POC completo corriendo | 1–2 días | **Alto** | Poco | Sí |
| 7 | Pruebas negativas con LLM — **compuerta 2** | Preguntas que intentan ver otro tenant, pedir escritura, columnas no expuestas o instrucciones escondidas en datos | Evidencia pasa/falla al 100 % | 1 día | **Alto** | Sí (pocas) | Sí (pruebas) |
| 8 | Corrida de las preguntas | Ejecución con OCI, trazas, latencia por etapa frente al límite de 60 s, separar error de catálogo del error del LLM | Informe de resultados | 1–2 días | **Alto** | **Sí (muestra)** | Ajustes de MDL |
| 9 | Dictamen del POC | Seguir, ajustar o descartar (con la limpieza de §5) | Decisión documentada | Horas | Alto | No | No |
| 10 | Servicio `ticket-insights` | Solo si el POC pasa: tenant y rol desde el token, auditoría, IA apagada, cambio OpenSpec | Servicio desplegable | 1–2 semanas | **Extra alto** en autenticación y aislamiento; alto en el resto | Poco | Sí |
| 11 | MDL completo de TM | Todas las reglas de negocio y el resto de los roles | Catálogo completo | Lo más largo: depende de las reglas, no del código | Extra alto | No | YAML/doc |
| 12 | Integración con el portal y evaluación final | Pantalla de preguntas y corrida golden de 30–100 | Función en el portal | ~1 semana + corrida | Alto | **Sí (completa)** | Sí |

**Compuertas:**
- **Compuerta 1 (etapa 4):** si el aislamiento falla sin LLM, se descarta Wren antes de armar el catálogo.
- **Compuerta 2 (etapa 7):** debe pasar al 100 % antes de la corrida de preguntas.
- **Etapa 9:** decide si se hacen la 10 a la 12.

### Ajustes tras la etapa 1

Detalle y evidencia en `ETAPA_1_REVISION_WREN.md`. Propuestos, pendientes de visto bueno:

- **Sin `wren-langchain`.** Se usa el motor de Wren directamente, dentro de un grafo fijo. El LLM no tiene herramientas: entrega el SQL como salida estructurada y nuestro código lo valida y lo ejecuta con el tenant de la petición. Sustituye a las "herramientas propias" de la regla 1 y de la etapa 6.
- **Versión:** `wrenai==0.18.0` exacta (o su parche 0.18.x al llegar a la etapa 4). Es la primera con el modo `read_only`.
- **Tres comprobaciones antes de ejecutar:** la propia de GemIA (una sentencia, sin nombres con esquema, lista blanca de funciones); el motor con `strict_mode` y `read_only`; y la revisión del filtro en el SQL expandido. Amplía la regla 6.
- **Capa de PostgreSQL con un rol por tenant**, no con una variable de sesión: las vistas filtran por el tenant asociado a `session_user`. Sustituye al texto de la capa 1 en la regla 4. En el POC son 2 roles; se borran junto con el esquema `insights`.
- **Catálogo:** toda regla de tenant con `required: true`, comprobado por un validador propio; los modelos no se llaman igual que las vistas físicas.
- **Límite de tiempo** fijado en el rol de PostgreSQL.
- **Etapa 4:** más casos negativos y siete comprobaciones que solo se pueden hacer ejecutando.

**Dónde va el esfuerzo extra alto:** etapas 3, 5, 11 y la parte de autenticación de la 10. Son las que, si salen mal, exponen datos entre clientes o dan respuestas equivocadas con apariencia de correctas.

## 7. Abierto

- ¿Un MDL común con control por fila, o uno por tenant? Se inclina por uno común; se decide en la etapa 3.
- ¿Los nombres y correos de personas pueden llegar al LLM en los resultados? Se decide en la etapa 3.
- ¿Cómo se comporta el toolkit síncrono con varios usuarios a la vez y 2 núcleos? Se mide en la etapa 8.
- Los umbrales de aceptación de la corrida (exactitud mínima) se fijan en la etapa 5, antes de correr.
