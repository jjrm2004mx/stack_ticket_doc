---
rol: admin
tipo: estados
audiencia: usuario_final
fuente: ticket-management
---

# Estados y transiciones — Admin

Como Admin tienes acceso a todas las transiciones del sistema, incluyendo las que están restringidas para Manager y Usuario. Este documento detalla el mapa completo de estados y las capacidades exclusivas de Admin.

---

## Los cuatro estados del sistema

### OPEN — Abierto
- **Estado inicial:** sí, todos los tickets nuevos comienzan aquí
- **Es terminal:** no
- **notifyRequester:** según configuración del workflow
- **Posición en el flujo:** 1

### IN_PROGRESS — En Progreso
- **Estado inicial:** no
- **Es terminal:** no
- **Posición en el flujo:** 2

### IN_REVIEW — En Revisión
- **Estado inicial:** no
- **Es terminal:** no
- **Posición en el flujo:** 3

### CLOSED — Cerrado
- **Estado inicial:** no
- **Es terminal:** sí — ninguna transición sale de este estado
- **Registra:** `resolvedAt` (fecha/hora) y `resolvedBy` (usuario que cerró)
- **Posición en el flujo:** 4

---

## Mapa completo de transiciones

### Transiciones disponibles para Admin (todas)

| Desde | Hacia | Rol requerido | ¿Admin puede? |
|---|---|---|---|
| ABIERTO | EN PROGRESO | Sin restricción | ✅ |
| ABIERTO | CERRADO | **ADMIN** | ✅ Exclusiva de Admin |
| EN PROGRESO | EN REVISIÓN | Sin restricción | ✅ |
| EN PROGRESO | ABIERTO | Sin restricción | ✅ |
| EN REVISIÓN | CERRADO | MANAGER o ADMIN | ✅ |
| EN REVISIÓN | EN PROGRESO | Sin restricción | ✅ |

**Transiciones no permitidas (para ningún rol):**
- ABIERTO → EN REVISIÓN ❌
- EN PROGRESO → CERRADO ❌
- EN REVISIÓN → ABIERTO ❌
- CERRADO → cualquier estado ❌ (terminal)

### Transición exclusiva de Admin: ABIERTO → CERRADO

Esta transición permite cerrar un ticket directamente desde su estado inicial sin pasar por EN PROGRESO ni EN REVISIÓN.

**Cuándo usarla:**
- Tickets duplicados que deben cerrarse sin trabajarse
- Tickets creados por error
- Solicitudes que ya fueron resueltas por otro medio antes de que el equipo tomara el ticket
- Tickets obsoletos que ya no aplican

**Advertencia:** usar esta transición omite el flujo normal de trabajo. Queda registrado en el historial que Admin cerró el ticket directamente desde ABIERTO. Úsala solo cuando sea realmente justificado.

---

## Estados especiales

### BORRADOR (Draft — status = NULL)

Los borradores son tickets con campo `status` nulo en la base de datos. No son un estado del enum Status sino una condición especial del modelo Ticket.

**Cómo se crean:**
- Automáticamente por el sistema de ingesta (ticket-ingestion-light/n8n) cuando procesa correos entrantes
- Por usuarios o sistemas externos que crean tickets sin activarlos

**Como Admin puedes:**
- Ver todos los borradores en la sección de Revisión de Borradores
- Editar cualquier campo del borrador: prioridad, clasificación, categoría, asignado a
- **Aprobar** → activa el borrador como ticket ABIERTO
- **Eliminar** → borra permanentemente el borrador (exclusivo de Admin)

**Diferencia con Manager:** el Manager puede aprobar borradores pero no eliminarlos. El Admin puede hacer ambas cosas.

### ARCHIVADO (isArchived = true)

El archivado no es un estado del ticket sino un campo booleano (`isArchived`). Un ticket archivado:
- Mantiene su estado actual (puede estar CERRADO y archivado simultáneamente)
- No aparece en las vistas y búsquedas normales
- Es recuperable si un Admin lo desarchivaera (aunque la UI actual puede no exponer esto)
- Requiere permiso ARCHIVE_TICKET — exclusivo de Admin

**Cuándo archivar:**
- Tickets muy antiguos que ya no son relevantes pero deben conservarse por auditoría
- Limpieza de la lista de trabajo sin eliminar el registro histórico

---

## Configuración del workflow desde Admin

Como Admin puedes modificar las definiciones de estados y transiciones desde la pantalla de configuración de workflow (requiere permiso MANAGE_WORKFLOW).

**Qué puedes configurar por estado:**
- Nombre visible (`name`)
- Descripción
- Color en la interfaz (`color` hex)
- Si notifica al solicitante al entrar a este estado (`notifyRequester`)
- Posición de visualización (`position`)

**Qué puedes configurar por transición:**
- Estado origen y destino
- Rol requerido para ejecutar la transición (`requiredRole` — puede ser NULL para sin restricción)
- Descripción de la transición

**Advertencia crítica:** modificar el workflow afecta a todos los tickets activos del sistema. Un cambio incorrecto puede bloquear transiciones en progreso. Realiza cambios en workflow solo si tienes certeza de lo que estás modificando.

---

## Preguntas frecuentes para Admin

**¿Puedo cambiar un ticket de CERRADO a cualquier otro estado?**
No. CERRADO es un estado terminal absoluto. Ningún rol puede revertirlo. Si necesitas reactivar un caso cerrado, crea un ticket nuevo y referencia el anterior en la descripción.

**¿Qué diferencia hay entre archivar y cerrar?**
Cerrar resuelve el caso formalmente (cambia el estado a CLOSED, registra fecha y usuario). Archivar oculta el ticket de la vista normal sin cambiar su estado — puede archivarse un ticket en cualquier estado, incluso ABIERTO.

**¿Puedo archivar un ticket que está EN PROGRESO?**
Técnicamente sí, tienes el permiso ARCHIVE_TICKET. Pero hacerlo ocultaría un ticket activo del equipo. Úsalo con precaución.

**¿Cuándo usar ABIERTO → CERRADO?**
Solo cuando el ticket no requiere trabajo real: duplicados, errores de creación, o casos ya resueltos por otro medio.

**¿Puede un Admin ver todos los estados incluyendo borradores?**
Sí. Admin tiene visibilidad total: tickets en todos los estados activos y todos los borradores pendientes.
