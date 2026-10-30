# 14 — Gestión por correo
## ticket-management · Cómo los correos crean, ordenan y mueven los tickets

> Para quién: el operador de la plataforma y el admin de un cliente con buzón de correo.
> Cambios de OpenSpec: `hilo-del-correo`, `aplicar-si-es-ticket`, `motivo-al-archivar`,
> `intencion-de-la-respuesta`, `estado-para-ligar`, `dueno-del-seguimiento`, `gestion-por-correo`
> (y en ticket-classification `decidir-antes-de-entregar`, `ligar-por-encabezados`).

---

## 1. La idea en una línea

El equipo trabaja desde su correo; la plataforma lee cada correo y **decide si es ticket, a quién le toca
y en qué estado va**, según quién escribió, a quién y para qué. La app se usa para lo que la IA no resuelve sola.

```
Correo nuevo ──▶ ¿Es ticket? ──▶ tablero / Borradores / Archivados
                     │
Respuesta ──▶ ligada a su ticket (conversación, folio o encabezados)
                     │
                     ▼
            Hilo del correo (quién → a quién · qué · intención)
                     │
                     ▼
            Gestión por correo: sugerencias A/B/C (y las automáticas, si está encendida)
```

## 2. Interruptores (por cliente, en /plataforma › detalle del cliente)

| Interruptor | Apagado | Encendido |
|---|---|---|
| Seguimiento del equipo | Lo que el equipo pide afuera es un correo más | Es un ticket de **quien lo mandó** (asignado a él) |
| Aplicar ¿es ticket? | La IA solo registra su decisión; todo se crea como siempre | No es ticket → **Archivados** ("IA: …"); dudoso o avance → **Borradores** con marca |
| Gestión por correo | La IA **sugiere** en el hilo (con botón) | La IA **mueve** el ticket (A1–A4, A6) y asigna a quien contesta (B3) |

Los tres piden motivo, quedan en la Auditoría y **nunca mandan correos** por sí mismos.

## 3. Roles de los estados (Gestión de Workflow › Gestión por correo)

La IA no usa el nombre del estado sino su **rol**; el admin elige qué estado cumple cada uno:

| Rol | Categoría que admite | Ejemplo en Shared Services |
|---|---|---|
| El equipo atiende (`IN_PROGRESS`) | En curso | En progreso |
| En espera del solicitante (`WAITING_REQUESTER`) | En espera | En espera del solicitante |
| En espera de un tercero (`WAITING_THIRD_PARTY`) | En espera | En espera de tercero |
| Resuelto, falta confirmación (`RESOLVED_PENDING`) | En curso o En espera | Resuelto, por confirmar |
| Cerrado al confirmar (`CLOSED`) | Cerrado · Resuelto | Cerrado |

- Un rol sin estado = ese caso **no se sugiere ni se hace**.
- La IA solo mueve por **transiciones permitidas**: si falta una transición, ese movimiento no aparece.
- Las esperas son categoría **En espera**: pausan la fecha límite y no cuentan contra el equipo en Métricas.

## 4. Los casos

**A — Estado** (según la última respuesta)

| | Cuándo | Movimiento | Automático* |
|---|---|---|---|
| A1 | El equipo le pide algo **al solicitante** | → En espera del solicitante | Sí |
| A2 | El equipo le pide algo **a un tercero** (también al crear un seguimiento del equipo) | → En espera de un tercero | Sí |
| A3 | Llega lo que se pidió (de fuera del equipo) estando en espera o resuelto | → El equipo atiende | Sí |
| A4 | Le piden algo al equipo estando en espera o resuelto | → El equipo atiende | Sí |
| A5 | El equipo entrega información | → Resuelto, por confirmar | Propuesto |
| A6 | El **solicitante** confirma que quedó (o el tercero, en un seguimiento del equipo) | → Cerrado | Sí |
| A6 | Otro tercero confirma | → Cerrado | Propuesto |
| A7 | El equipo avisa que ya quedó, o lleva 5 días resuelto sin respuesta | → Resuelto / Cerrado | Propuesto |
| A8 | Escriben después del cierre | → Reabrir | Propuesto |

**B — Asignación**

| | Cuándo | Movimiento | Automático* |
|---|---|---|---|
| B1 | Seguimiento del equipo | Asignado a quien lo mandó (al crear) | Sí (con "Aplicar ¿es ticket?") |
| B2 | Asignado por regla, pero otra persona del equipo es la que contesta | Reasignar a quien atiende | Propuesto |
| B3 | Sin asignado y contesta alguien del equipo | Asignar a quien contestó | Sí |
| B4 | El equipo le pide apoyo a otra persona del equipo | Reasignar a esa persona | Propuesto |
| B5 | El asignado ya no está activo | Reasignar (a quien contesta o por la regla del área) | Propuesto |

**C — Separar, ligar y duplicados**

| | Cuándo | Movimiento | Automático* |
|---|---|---|---|
| C1 | Una respuesta trae una **nueva solicitud** | Crear ticket con ella (Borradores, "Separado de #…") | Propuesto |
| C2 | Un borrador "Parece avance de otro ticket" y se encuentra cuál | Ligarlo al hilo de ese ticket (con adjuntos) | Propuesto |
| C3 | Respuesta en otra conversación que cita un correo de un ticket abierto | Se liga sola (clasificador) | Sí |
| C4 | Mismo identificador (5+ dígitos) o asunto muy parecido que otro ticket abierto | Archivar como duplicado | Propuesto |

\* Automático solo con "Gestión por correo" encendido; apagado, todo aparece como sugerencia con su botón.

Cómo se identifica "a quién se pide": el primer destinatario directo (Para) que no es el buzón. Del equipo
→ B4; el solicitante → A1; cualquier otro → A2. La intención la da el LLM en la misma llamada que ya
evaluaba cada respuesta; quién y a quién salen del correo (sin LLM).

## 5. Qué ve el equipo

- **Detalle › Hilo del correo:** cada respuesta con su intención (morado) y, si la IA hizo algo, "IA: Pasó a «…» · Se asignó a …".
  Arriba, las sugerencias vigentes con su botón (Aplicar, Reasignar, Crear ticket, Archivar como duplicado).
- **Borradores:** marca amarilla (¿Es ticket? / Parece avance) y, si se encontró el ticket, "Ligar a #…".
- **Archivados:** "Archivado por: IA"; filtro "Lo archivó una regla o la IA". Restaurar regresa a Borradores.
- **Historial:** todo movimiento de la IA queda como hecho por el usuario de integración, sin aviso al solicitante.

## 6. Operación

**Encender por fases** (como se hizo con ¿es ticket?):
1. Roles y transiciones configurados (sección 3).
2. Un día con "Gestión por correo" apagada: revisar en los hilos si las sugerencias acertaron.
3. Encender en /plataforma con motivo.

**Revisar lo que hizo la IA** (solo lectura):

```sql
BEGIN; SET LOCAL app.tenant_id = <tenant>;
SELECT left(ticket_id::text, 8) AS ticket, received_at, from_address, intent, auto_action
FROM ticket_channel_replies WHERE tenant_id = <tenant> AND auto_action IS NOT NULL
ORDER BY received_at DESC LIMIT 50;
COMMIT;
```

**Apagar:** el interruptor "Gestión por correo" en /plataforma. Lo ya movido queda; se deshace cambiando estado
o asignado a mano.

**No aparece una sugerencia que debería:** revisar (1) que el caso tenga estado en Gestión de Workflow, (2)
que la transición del estado actual al destino esté habilitada, (3) que la respuesta tenga intención (las
anteriores al 2026-10-29 no la tienen).

## 7. Configuración aplicada en Shared Services (tenant 1, 2026-10-30)

| Pos. | Estado | Categoría | Rol |
|---|---|---|---|
| 1 | Abierto | Nuevo | — |
| 2 | En progreso | En curso | El equipo atiende |
| 3 | En espera de tercero | En espera | En espera de un tercero |
| 4 | En espera del solicitante | En espera | En espera del solicitante |
| 5 | Resuelto, por confirmar (antes "En revisión") | En espera | Resuelto, falta confirmación |
| 6 | Cerrado | Cerrado · Resuelto | Cerrado al confirmar |

Transiciones: entre En progreso y las dos esperas en ambos sentidos, de cualquiera a Resuelto y a Cerrado,
de Resuelto a En progreso, y de Cerrado a En progreso (reabrir). "En revisión" cambió de categoría En curso a
En espera por SQL (la pantalla no cambia categorías): no tenía tickets activos.

## 8. Datos

| Columna | Para qué |
|---|---|
| `workflow_states.email_role` | Rol del estado (uno por cliente) |
| `tenants.email_flow` / `apply_triage` / `follow_up_by_sender` | Interruptores |
| `tickets.triage_type` / `triage_note` | Tipo que decidió la IA / marca de Borradores |
| `tickets.merged_into` | Ticket al que se ligó (C2) o del que es duplicado (C4); sus respuestas van allá |
| `ticket_channel_replies.intent` / `auto_action` / `split_ticket_id` | Intención, lo que hizo la IA, ticket separado (C1) |
