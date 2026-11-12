# Etapa 1 — Definición de las vistas `insights`

**Fecha:** 2026-11-12
**Plan:** `PLAN_PREGUNTAS_SOBRE_LOS_DATOS.md` (versión 4), etapa 1
**Complementa a:** `ETAPA_1_ALCANCE_Y_CATALOGO.md` (significados y preguntas de referencia)
**Estado:** columnas confirmadas contra el código de TM y definición de cada vista escrita. **Nada de esto se ha ejecutado:** el SQL es el texto que irá en la migración de la etapa 3, y se prueba ahí.
**Leído en TM** (copia local del 11-12, migraciones hasta `V72`): `V1`, `V21`, `V28`, `V31` a `V35`, `V37`, `V44`, `V45`, `V54`, `V56`, `V57`, `V59`, `afterMigrate.sql`, `DashboardService.java`, `BusinessTime.java`, `Ticket.java`, `StatusChange.java`, `ArchiveService.java`, `UserService.java`, las entidades de persistencia y el portal (`ListView.tsx`, `types/index.ts`).

---

## 1. Qué se confirmó y qué cambió respecto a lo supuesto

### Columnas reales

| Dato lógico | Columna real en TM | Nota |
|---|---|---|
| Cliente | `tenant_id` en todas las tablas que se usan | Todas tienen seguridad por fila forzada (`V34`, `V37`) |
| Título | `tickets.title` | |
| Prioridad | `tickets.priority` | `BAJA`, `MEDIA`, `ALTA`, `CRITICA` |
| Estado actual | `tickets.status_id` → `workflow_states` (`name`, `category`, `outcome`, `position`, `is_active`) | La categoría y el desenlace de un estado no se pueden editar; el nombre sí. Un estado nunca se borra, solo se desactiva |
| Área | `tickets.classification_id` → `ticket_classifications.name` | En la base el área se llama "classification" |
| Categoría | `tickets.category_id` → `ticket_categories.name`; su área es `ticket_categories.classification_id` | |
| Responsable | `tickets.assigned_to_id` → `users.full_name` | |
| Borrador | `status_id IS NULL` y `is_archived = FALSE` | Igual que la bandeja de borradores |
| Archivado | `is_archived = TRUE`, con `archived_at`, `archive_reason` y `archive_by_rule` (`V45`) | Incluye borradores descartados |
| Cierre | `tickets.closed_at` y `tickets.closed_outcome` | Sí se mantienen siempre: ver abajo |
| Historial de estados | `ticket_status_history`: `from_state_id`, `to_state_id`, `to_category`, `to_outcome`, `changed_at` | `from_status` y `to_status` son la foto del nombre en ese momento y no se usan |
| Origen | `tickets.channel_id` (no nulo = llegó por un buzón) → `tenant_channels.name` | |
| Folio | Los primeros 8 caracteres del `id`, en mayúsculas | Así lo forma el portal: `id.slice(0, 8).toUpperCase()` |
| Personas | `users`: `full_name`, `role` (`USER`, `MANAGER`, `ADMIN`), `status` (`ACTIVE`, `INACTIVE`, `INVITED`, `SYSTEM`) | Los usuarios de sistema son los de `status = 'SYSTEM'` |

### Hallazgos que cambian el diseño

| # | Hallazgo | Consecuencia |
|---|---|---|
| H1 | **Todas las fechas son `TIMESTAMP` sin zona y están en UTC.** El backend fija la zona de la JVM en UTC (`TicketManagementApplication`) y convierte a la zona del negocio al mostrar (`BusinessTime`) | Cada vista convierte a hora local con la misma expresión que usa el tablero. Cierra la trampa T8 de `NOTAS_DE_DISENO.md` |
| H2 | **Fijar la zona horaria en el rol no basta.** El controlador JDBC manda la zona de la JVM (UTC) al abrir la conexión, y ese valor gana sobre el que tenga el rol **[por probar en la etapa 3]** | El ejecutor de TM fija la zona en cada transacción con `SET LOCAL TIME ZONE`, igual que fija el cliente. Las vistas leen la zona de la sesión, así que las fechas y `CURRENT_DATE` siempre van juntas. Corrige la regla 5 de `ETAPA_1_ALCANCE_Y_CATALOGO.md` |
| H3 | `closed_at` y `closed_outcome` se llenan al entrar a un estado cerrado y se limpian al reabrir. El único camino para cambiar de estado es `Ticket.changeStatus`, y lo usan también los cambios automáticos | El cierre actual sale de `tickets`, sin recorrer el historial |
| H4 | `resolved_at` ya no existe en `tickets` (se quitó en `V32`). El tablero toma la resolución del historial: el **último cierre resuelto**, aunque el ticket se haya reabierto después | `resolution_days` copia esa fórmula. Un ticket reabierto conserva su último tiempo de resolución hasta que se resuelva otra vez; `closed_at` sí queda vacío |
| H5 | Un ticket creado directo en el portal **no deja fila en el historial** por su estado inicial. Un borrador aprobado sí, con origen vacío | `ticket_state_changes` no sirve para contar creaciones. Regla del catálogo: las creaciones se cuentan con `tickets.created_date` |
| H6 | Los archivados incluyen borradores descartados, que nunca tuvieron estado | `archived_tickets` lleva la columna `was_draft` y el estado puede venir vacío |
| H7 | `archive_detail` es texto libre: un motivo escrito a mano, un folio o una nota de la IA | No se expone, por la regla de no llevar texto libre de personas. Se expone el motivo de lista (`archive_reason`) |
| H8 | El tablero cuenta como reapertura cualquier salida de un estado cerrado, también el paso de un cerrado a otro cerrado | `is_reopen` es más estricto: sale de cerrado **y** entra a uno no cerrado. Difiere del tablero solo en ese caso raro |
| H9 | El tablero mide la antigüedad en tiempo transcurrido ("más de 7 días" es más de 168 horas) | `age_days` va con decimales para dar el mismo conteo que el tablero. Lo mismo `resolution_days` y `waiting_days`. Regla del catálogo: al mostrar días, redondear a un decimal |

### Cambios en los nombres lógicos

| Cambio | Motivo |
|---|---|
| `states.position` pasa a llamarse `display_order` | `POSITION` es una función de SQL y el validador trabaja con lista blanca de funciones: se evita la confusión |
| Columna nueva `channel_name` en `tickets`, `drafts` y `archived_tickets` | Con varios buzones por cliente, "¿cuántos llegaron por el buzón X?" es una pregunta natural. Es el nombre del canal, no su dirección de correo |
| `ticket_state_changes` suma `from_state_id` y `to_state_id` | Permite unir con `states` |
| `archived_tickets` suma `was_draft`, `archive_reason` y `archived_by_rule` | Hallazgos H6 y H7 |
| `origin` queda confirmado: `CORREO` o `PORTAL` | Sale de `channel_id` |

## 2. Reglas comunes a todas las vistas

- **Esquema:** `insights`. Las tablas base se nombran con su esquema (`public.`), porque el rol de consulta tendrá `search_path = insights`.
- **Hora local:** una fecha guardada se convierte con
  `columna AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone')`.
  Es la expresión de `BusinessTime.sqlLocal`, con la zona de la sesión en lugar de un texto fijo.
- **"Ahora" en UTC,** para restar contra columnas guardadas: `now() AT TIME ZONE 'UTC'`. No depende de la zona de la sesión.
- **Los joins repiten `tenant_id`.** La seguridad por fila ya lo garantiza; repetirlo hace que la vista siga siendo correcta en la prueba de la etapa 6 que apaga la seguridad por fila.
- **Días con decimales:** segundos entre dos momentos, divididos entre 86 400.
- **Dueño:** cada vista pasa a `ticket_insights_owner` en la misma migración (trampa T1).

## 3. Definición de cada vista

### 3.1 `insights.tickets` — un ticket activo por fila

```sql
CREATE VIEW insights.tickets AS
SELECT
    t.tenant_id,
    t.id                                    AS ticket_id,
    upper(left(t.id::text, 8))              AS ticket_ref,
    t.title,
    t.priority,
    ws.id                                   AS state_id,
    ws.name                                 AS state_name,
    ws.category                             AS state_category,
    (ws.category <> 'DONE')                 AS is_open,
    c.id                                    AS area_id,
    c.name                                  AS area_name,
    k.id                                    AS category_id,
    k.name                                  AS category_name,
    u.id                                    AS assignee_id,
    u.full_name                             AS assignee_name,
    CASE WHEN t.channel_id IS NOT NULL THEN 'CORREO' ELSE 'PORTAL' END AS origin,
    ch.name                                 AS channel_name,
    (t.created_at AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))        AS created_at,
    (t.created_at AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))::date  AS created_date,
    (t.closed_at  AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))        AS closed_at,
    (t.closed_at  AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))::date  AS closed_date,
    t.closed_outcome,
    (EXTRACT(EPOCH FROM (r.resolved_at - t.created_at)) - r.wait_secs_resolved) / 86400.0 AS resolution_days,
    COALESCE(r.wait_secs_total, 0) / 86400.0                                              AS waiting_days,
    EXTRACT(EPOCH FROM ((now() AT TIME ZONE 'UTC') - t.created_at)) / 86400.0             AS age_days
FROM public.tickets t
JOIN public.workflow_states ws
       ON ws.id = t.status_id AND ws.tenant_id = t.tenant_id
LEFT JOIN public.ticket_classifications c
       ON c.id = t.classification_id AND c.tenant_id = t.tenant_id
LEFT JOIN public.ticket_categories k
       ON k.id = t.category_id AND k.tenant_id = t.tenant_id
LEFT JOIN public.users u
       ON u.id = t.assigned_to_id AND u.tenant_id = t.tenant_id
LEFT JOIN public.tenant_channels ch
       ON ch.id = t.channel_id AND ch.tenant_id = t.tenant_id
LEFT JOIN (
    SELECT x.ticket_id,
           MAX(x.resolved_at) AS resolved_at,
           COALESCE(SUM(EXTRACT(EPOCH FROM (LEAST(COALESCE(x.next_at, x.resolved_at), x.resolved_at) - x.changed_at)))
                    FILTER (WHERE x.to_category = 'WAITING' AND x.changed_at < x.resolved_at), 0) AS wait_secs_resolved,
           COALESCE(SUM(EXTRACT(EPOCH FROM (COALESCE(x.next_at, now() AT TIME ZONE 'UTC') - x.changed_at)))
                    FILTER (WHERE x.to_category = 'WAITING'), 0)                                  AS wait_secs_total
    FROM (
        SELECT h.ticket_id, h.changed_at, h.to_category,
               LEAD(h.changed_at) OVER (PARTITION BY h.ticket_id ORDER BY h.changed_at) AS next_at,
               MAX(h.changed_at) FILTER (WHERE h.to_category = 'DONE' AND h.to_outcome = 'RESOLVED')
                   OVER (PARTITION BY h.ticket_id)                                      AS resolved_at
        FROM public.ticket_status_history h
    ) x
    GROUP BY x.ticket_id
) r ON r.ticket_id = t.id
WHERE t.is_archived = FALSE
  AND t.status_id IS NOT NULL;
```

| Columna | Tipo | Origen y significado |
|---|---|---|
| `tenant_id` | entero | Cliente |
| `ticket_id` | uuid | `tickets.id` |
| `ticket_ref` | texto | Folio del portal, sin el `#` |
| `title` | texto | `tickets.title` |
| `priority` | texto | `BAJA`, `MEDIA`, `ALTA`, `CRITICA` |
| `state_id`, `state_name`, `state_category` | entero, texto, texto | Estado actual con su nombre vigente. Categoría: `NEW`, `ACTIVE`, `WAITING`, `DONE` |
| `is_open` | booleano | La categoría no es `DONE`. Misma regla que el tablero |
| `area_id`, `area_name` | entero, texto | Vacíos si el ticket no tiene área |
| `category_id`, `category_name` | entero, texto | Vacíos si no tiene categoría |
| `assignee_id`, `assignee_name` | uuid, texto | Vacíos si está sin asignar |
| `origin` | texto | `CORREO` si llegó por un buzón, `PORTAL` si se capturó a mano |
| `channel_name` | texto | Nombre del buzón; vacío en los de portal |
| `created_at`, `created_date` | fecha y hora, fecha | Creación, hora local |
| `closed_at`, `closed_date` | fecha y hora, fecha | Cierre vigente; vacío si está abierto |
| `closed_outcome` | texto | `RESOLVED` o `CANCELLED`; vacío si está abierto |
| `resolution_days` | número | Días de la creación al último cierre resuelto, sin las esperas anteriores a ese cierre. Vacío si nunca se resolvió. Fórmula del MTTR del tablero |
| `waiting_days` | número | Días acumulados en estados de categoría `WAITING`, hasta ahora. Cero si nunca esperó |
| `age_days` | número | Días transcurridos desde la creación |

**Coincidencia con el tablero.** La población es la misma (`is_archived = FALSE` y con estado). La subconsulta `r` reproduce `RESOLUTION_CTE`: el último cierre resuelto por ticket y los tramos de espera recortados a ese cierre. `resolution_days` es `RESOLUTION_DAYS` tal cual.

### 3.2 `insights.ticket_state_changes` — un cambio de estado por fila

```sql
CREATE VIEW insights.ticket_state_changes AS
SELECT
    h.tenant_id,
    h.ticket_id,
    (h.changed_at AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))        AS changed_at,
    (h.changed_at AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))::date  AS changed_date,
    fs.id                                   AS from_state_id,
    fs.name                                 AS from_state_name,
    fs.category                             AS from_state_category,
    ts.id                                   AS to_state_id,
    ts.name                                 AS to_state_name,
    h.to_category                           AS to_state_category,
    h.to_outcome,
    (h.to_category = 'DONE')                AS is_closure,
    COALESCE(fs.category = 'DONE' AND h.to_category <> 'DONE', FALSE) AS is_reopen
FROM public.ticket_status_history h
JOIN public.tickets t
       ON t.id = h.ticket_id AND t.tenant_id = h.tenant_id
JOIN public.workflow_states ts
       ON ts.id = h.to_state_id AND ts.tenant_id = h.tenant_id
LEFT JOIN public.workflow_states fs
       ON fs.id = h.from_state_id AND fs.tenant_id = h.tenant_id
WHERE t.is_archived = FALSE
  AND t.status_id IS NOT NULL;
```

| Columna | Tipo | Origen y significado |
|---|---|---|
| `tenant_id`, `ticket_id` | entero, uuid | Cliente y ticket. Solo tickets que aparecen en `tickets` |
| `changed_at`, `changed_date` | fecha y hora, fecha | Cuándo, hora local |
| `from_state_id`, `from_state_name`, `from_state_category` | entero, texto, texto | Estado del que salió, con su nombre vigente. Vacíos cuando el cambio fue la aprobación de un borrador |
| `to_state_id`, `to_state_name`, `to_state_category`, `to_outcome` | entero, texto, texto, texto | Estado al que entró |
| `is_closure` | booleano | Entró a un estado de categoría `DONE` |
| `is_reopen` | booleano | Salió de un estado `DONE` y entró a uno que no lo es |

Los nombres son los vigentes y no la foto del historial, para que coincidan con `states` y con la lista de valores que TM manda al motor.

### 3.3 `insights.drafts` — un borrador en revisión por fila

```sql
CREATE VIEW insights.drafts AS
SELECT
    t.tenant_id,
    t.id                                    AS ticket_id,
    upper(left(t.id::text, 8))              AS ticket_ref,
    t.title,
    t.priority,
    c.id                                    AS area_id,
    c.name                                  AS area_name,
    k.id                                    AS category_id,
    k.name                                  AS category_name,
    CASE WHEN t.channel_id IS NOT NULL THEN 'CORREO' ELSE 'PORTAL' END AS origin,
    ch.name                                 AS channel_name,
    (t.created_at AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))        AS created_at,
    (t.created_at AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))::date  AS created_date,
    EXTRACT(EPOCH FROM ((now() AT TIME ZONE 'UTC') - t.created_at)) / 86400.0         AS age_days
FROM public.tickets t
LEFT JOIN public.ticket_classifications c
       ON c.id = t.classification_id AND c.tenant_id = t.tenant_id
LEFT JOIN public.ticket_categories k
       ON k.id = t.category_id AND k.tenant_id = t.tenant_id
LEFT JOIN public.tenant_channels ch
       ON ch.id = t.channel_id AND ch.tenant_id = t.tenant_id
WHERE t.is_archived = FALSE
  AND t.status_id IS NULL;
```

El área y la categoría son las propuestas: quien revisa puede cambiarlas al aprobar. No lleva estado, cierre ni responsable.

### 3.4 `insights.archived_tickets` — un ticket archivado por fila

```sql
CREATE VIEW insights.archived_tickets AS
SELECT
    t.tenant_id,
    t.id                                    AS ticket_id,
    upper(left(t.id::text, 8))              AS ticket_ref,
    t.title,
    t.priority,
    (t.status_id IS NULL)                   AS was_draft,
    ws.id                                   AS state_id,
    ws.name                                 AS state_name,
    ws.category                             AS state_category,
    c.id                                    AS area_id,
    c.name                                  AS area_name,
    k.id                                    AS category_id,
    k.name                                  AS category_name,
    u.id                                    AS assignee_id,
    u.full_name                             AS assignee_name,
    CASE WHEN t.channel_id IS NOT NULL THEN 'CORREO' ELSE 'PORTAL' END AS origin,
    ch.name                                 AS channel_name,
    (t.created_at  AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))        AS created_at,
    (t.created_at  AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))::date  AS created_date,
    (t.closed_at   AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))        AS closed_at,
    (t.closed_at   AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))::date  AS closed_date,
    t.closed_outcome,
    (t.archived_at AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))        AS archived_at,
    (t.archived_at AT TIME ZONE 'UTC' AT TIME ZONE current_setting('TimeZone'))::date  AS archived_date,
    COALESCE(t.archive_reason, 'MANUAL')    AS archive_reason,
    t.archive_by_rule                       AS archived_by_rule
FROM public.tickets t
LEFT JOIN public.workflow_states ws
       ON ws.id = t.status_id AND ws.tenant_id = t.tenant_id
LEFT JOIN public.ticket_classifications c
       ON c.id = t.classification_id AND c.tenant_id = t.tenant_id
LEFT JOIN public.ticket_categories k
       ON k.id = t.category_id AND k.tenant_id = t.tenant_id
LEFT JOIN public.users u
       ON u.id = t.assigned_to_id AND u.tenant_id = t.tenant_id
LEFT JOIN public.tenant_channels ch
       ON ch.id = t.channel_id AND ch.tenant_id = t.tenant_id
WHERE t.is_archived = TRUE;
```

| Columna propia | Tipo | Significado |
|---|---|---|
| `was_draft` | booleano | Era un borrador que se descartó; nunca tuvo estado |
| `state_*` | | Estado que tenía al archivarse; vacío si `was_draft` |
| `archived_at`, `archived_date` | fecha y hora, fecha | Cuándo se archivó, hora local |
| `archive_reason` | texto | Motivo de lista. Sin motivo guardado cuenta como `MANUAL`, igual que en el portal |
| `archived_by_rule` | booleano | Lo archivó un proceso o la IA, no una persona |

No lleva `is_open`, `resolution_days`, `waiting_days` ni `age_days`: son medidas de tickets vivos.

Valores de `archive_reason` y cómo los nombra el portal:

| Valor | Etiqueta |
|---|---|
| `MANUAL` | Archivado a mano |
| `NO_ES_TICKET` | No es ticket |
| `AVANCE_ATENDIDO` | Avance o cierre ya atendido |
| `ATENDIDO_FUERA` | Se atendió por fuera |
| `YA_NO_SE_REQUIERE` | Ya no se requiere |
| `DUPLICADO` | Duplicado |
| `OTRO` | Otro |
| `AVISO_AUTOMATICO` | Aviso automático |
| `COPIA_EQUIPO` | Copia del equipo |
| `NO_REQUIERE_ACCION` | No requiere acción |
| `PARA_CONOCIMIENTO` | Para conocimiento |
| `PRUEBA` | Prueba |

### 3.5 Catálogos del cliente

```sql
CREATE VIEW insights.states AS
SELECT s.tenant_id,
       s.id        AS state_id,
       s.name      AS state_name,
       s.category  AS state_category,
       s.outcome   AS state_outcome,
       s.position  AS display_order,
       s.is_active
FROM public.workflow_states s;

CREATE VIEW insights.areas AS
SELECT c.tenant_id,
       c.id                 AS area_id,
       c.name               AS area_name,
       (c.is_active IS TRUE) AS is_active
FROM public.ticket_classifications c;

CREATE VIEW insights.categories AS
SELECT k.tenant_id,
       k.id                 AS category_id,
       k.name               AS category_name,
       c.id                 AS area_id,
       c.name               AS area_name,
       (k.is_active IS TRUE) AS is_active
FROM public.ticket_categories k
JOIN public.ticket_classifications c
       ON c.id = k.classification_id AND c.tenant_id = k.tenant_id;

CREATE VIEW insights.agents AS
SELECT u.tenant_id,
       u.id                  AS agent_id,
       u.full_name           AS agent_name,
       u.role,
       (u.status = 'ACTIVE') AS is_active
FROM public.users u
WHERE u.status <> 'SYSTEM';
```

- `states.display_order` puede venir vacío: la base lo permite.
- `areas.is_active` y `categories.is_active` usan `IS TRUE` porque la columna admite vacío.
- `categories.is_active` es el interruptor de la categoría. No mira si su área está activa.
- `agents` deja fuera a los usuarios de sistema. Un invitado que aún no acepta sale con `is_active` falso. Es la misma regla con la que TM decide a quién se le puede asignar un ticket. `role` es `USER`, `MANAGER` o `ADMIN`.

## 4. Columnas de las tablas base que leen las vistas

Es la lista para los permisos de `ticket_insights_owner`: `SELECT` solo sobre estas columnas.

| Tabla | Columnas |
|---|---|
| `tickets` | `id`, `tenant_id`, `title`, `priority`, `status_id`, `classification_id`, `category_id`, `assigned_to_id`, `channel_id`, `is_archived`, `created_at`, `closed_at`, `closed_outcome`, `archived_at`, `archive_reason`, `archive_by_rule` |
| `ticket_status_history` | `tenant_id`, `ticket_id`, `changed_at`, `from_state_id`, `to_state_id`, `to_category`, `to_outcome` |
| `workflow_states` | `id`, `tenant_id`, `name`, `category`, `outcome`, `position`, `is_active` |
| `ticket_classifications` | `id`, `tenant_id`, `name`, `is_active` |
| `ticket_categories` | `id`, `tenant_id`, `classification_id`, `name`, `is_active` |
| `users` | `id`, `tenant_id`, `full_name`, `role`, `status` |
| `tenant_channels` | `id`, `tenant_id`, `name` |

**Quedan fuera a propósito:** de `tickets`, la descripción, el solicitante (`requester_*`), todo el correo (`email_*`, `thread_key`, `origin_message_ref`, `external_id`), `due_date`, `archive_detail`, `triage_note` y `triage_type`; de `users`, el correo y la contraseña; de `tenant_channels`, la dirección y la llave. Tampoco se tocan comentarios, adjuntos, eventos ni el contenido de los correos.

## 5. Ajustes a las preguntas de referencia

Las preguntas 1, 2 y 4 a 13 de `ETAPA_1_ALCANCE_Y_CATALOGO.md` quedan igual. Cambia la 3 y se agregan tres:

| # | Pregunta | SQL esperado |
|---|---|---|
| 3 | ¿Cuántos tickets hay en cada estado? | `SELECT s.state_name, COUNT(t.ticket_id) AS tickets FROM states s LEFT JOIN tickets t ON t.state_id = s.state_id WHERE s.is_active GROUP BY s.state_name, s.display_order ORDER BY s.display_order` |
| 14 | ¿Cuántos borradores hay por revisar? | `SELECT COUNT(*) AS borradores FROM drafts` |
| 15 | ¿Cuántos tickets se archivaron este mes? | `SELECT COUNT(*) AS tickets FROM archived_tickets WHERE archived_date >= date_trunc('month', CURRENT_DATE)::date` |
| 16 | ¿Por qué motivo se archivaron los tickets este año? | `SELECT archive_reason, COUNT(*) AS tickets FROM archived_tickets WHERE archived_date >= date_trunc('year', CURRENT_DATE)::date GROUP BY archive_reason ORDER BY 2 DESC` |

Reglas del catálogo que salen de esta revisión:

- Los días se muestran redondeados a un decimal (`ROUND(x, 1)`). Para comparar se usa el valor sin redondear.
- Las creaciones se cuentan en `tickets` por `created_date`, nunca en `ticket_state_changes`.
- "Hoy" es `CURRENT_DATE`. Para la hora actual, `LOCALTIMESTAMP`, que se compara bien con las columnas `*_at`.
- Los valores de `archive_reason` viajan con su etiqueta, para que el modelo traduzca "duplicados" o "no era ticket".

## 6. Lo que la etapa 2 debe recoger

| Tema | Qué hacer |
|---|---|
| Zona horaria (H2) | El ejecutor ejecuta `SET LOCAL TIME ZONE '<app.timezone>'` en cada transacción. Caso de prueba de la compuerta 1: con la conexión real de TM, `CURRENT_DATE` y `created_date` salen en hora local |
| `current_setting` | Las vistas y las políticas de seguridad por fila la usan. Debe seguir permitida para el rol de consulta; solo se quita `set_config` (variante A). El modelo no puede llamarla: no está en la lista blanca |
| Costo de `tickets` | La subconsulta de tiempos recorre el historial. PostgreSQL puede omitirla cuando la pregunta no usa `resolution_days` ni `waiting_days` **[por probar]**. Medirlo en la etapa 6 |
| Coincidencia con el catálogo | La prueba automática compara nombre y tipo de cada columna de estas vistas con lo que el motor espera |
| Resultados esperados | Falta calcularlos para las preguntas de referencia. Por decisión del dueño (11-12) se hace en la etapa 3, cuando las vistas existan |

Todo lo de esta sección quedó recogido en la etapa 2: ver `ETAPA_2_DISENO.md` y el cambio `preguntas-sobre-los-datos` de TM.
