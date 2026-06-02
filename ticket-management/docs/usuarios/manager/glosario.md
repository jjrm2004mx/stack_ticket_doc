---
rol: manager
tipo: glosario
audiencia: usuario_final
fuente: ticket-management
---

# Glosario — Manager

Definiciones completas de todos los términos del sistema, con contexto específico para las responsabilidades de un Manager.

---

**Ticket**
Registro formal de una solicitud, problema o tarea. Cada ticket tiene: identificador único (UUID), estado, prioridad, tipo, asignado, creador, clasificación y categoría. Es la unidad central del sistema.

**Estado (Status)**
Etapa actual del ticket en el flujo de trabajo. Los cuatro estados son: OPEN (Abierto), IN_PROGRESS (En Progreso), IN_REVIEW (En Revisión), CLOSED (Cerrado). Como Manager puedes cambiar entre todos excepto la transición directa ABIERTO→CERRADO.

**Estado inicial**
OPEN (Abierto) — todos los tickets nuevos comienzan aquí automáticamente.

**Estado terminal**
CLOSED (Cerrado) — no tiene transiciones de salida. Es irreversible.

**Transición**
Cambio de un estado a otro. El sistema valida que la transición sea permitida antes de ejecutarla. Las transiciones están definidas en la tabla `workflow_transitions` de la base de datos.

**Historial de estados (StatusHistory)**
Registro de todos los cambios de estado del ticket. Cada entrada incluye: estado anterior (`from_status`), estado nuevo (`to_status`), usuario que hizo el cambio (`changed_by`), fecha y hora exacta (`changed_at`).

**Evento (TicketEvent)**
Registro de cualquier acción realizada sobre el ticket: cambio de estado, edición de campos, comentarios, adjuntos, asignación. Cada evento incluye tipo de evento, usuario, fecha/hora, y valores anterior/nuevo.

**Borrador (Draft)**
Ticket sin estado activo (`status = NULL`). Creado automáticamente cuando llega un ticket por correo electrónico y necesita revisión humana. No aparece en el flujo normal hasta ser aprobado. Como Manager eres responsable de revisarlos y aprobarlos.

**Aprobar borrador**
Acción que activa un borrador y lo convierte en ticket ABIERTO. Al aprobar, el ticket entra al flujo normal con la prioridad, clasificación y asignación que definiste en la revisión.

**REVIEW_DRAFTS**
Permiso que permite revisar y aprobar borradores. Activo por defecto para Manager.

**Prioridad**
Urgencia del ticket:
- `BAJA` — puede esperar
- `MEDIA` — atención en tiempo razonable
- `ALTA` — afecta el trabajo, atención pronta
- `CRÍTICA` — bloqueante total, atención inmediata

**Tipo de ticket**
- `TASK` (Tarea) — acción a realizar
- `BUG` — problema o falla
- `FEATURE` — solicitud de mejora
- `DOCS` — solicitud de documentación

**Asignado a (assignedTo)**
Usuario del equipo responsable de resolver el ticket. Campo UUID que referencia a un usuario del sistema. Como Manager puedes asignar y reasignar tickets.

**Solicitante (Requester)**
Persona que originó la solicitud. Puede ser un Usuario interno (`createdBy`) o un solicitante externo identificado por email (`requesterEmail`) cuando el ticket llegó por correo.

**requesterEmail / requesterName**
Campos del ticket que identifican al solicitante externo cuando el ticket fue creado por integración de correo (no por un usuario interno del sistema).

**Adjunto (Attachment)**
Archivo vinculado a un ticket. Límite: 10 MB por archivo. Como Manager puedes ver, descargar, subir y eliminar adjuntos. La eliminación es irreversible.

**UPLOAD_ATTACHMENT / DELETE_ATTACHMENT**
Permisos que te dan control total sobre los archivos del ticket. Ambos activos por defecto para Manager.

**Comentario (Comment)**
Mensaje de texto asociado a un ticket. Visible para todos los usuarios con acceso al ticket. Incluye autor y fecha/hora. No se pueden eliminar comentarios una vez enviados.

**Dashboard**
Pantalla de métricas y estadísticas del equipo. Requiere permiso VIEW_DASHBOARD (activo para Manager). Muestra distribución de tickets por estado, prioridad, asignado y tiempo de resolución.

**VIEW_TICKET_EVENTS**
Permiso para ver el historial completo de eventos de un ticket. Activo para Manager. No disponible para rol Usuario.

**Clasificación (Classification)**
Dominio al que pertenece el ticket. Las cuatro clasificaciones son:
- `IT` — tecnología e infraestructura
- `cliente` — atención y soporte al cliente
- `operaciones` — procesos y logística
- `otro` — sin clasificación específica

**Categoría (Category)**
Subcategoría dentro de la clasificación. Ejemplo: dentro de IT existen hardware, software, red, acceso, correo, impresora, VPN, servidor, base_de_datos, seguridad.

**classificationHint / categoryHint**
Campos del ticket que guardan la clasificación/categoría sugerida por el sistema de IA cuando no encontró una coincidencia exacta en el catálogo. Útil para revisar borradores y entender la intención original del clasificador.

**Fecha límite (dueDate)**
Fecha de resolución esperada del ticket. Configurable al crear o editar. Usada para priorización.

**Horas estimadas (estimatedHours)**
Estimación de esfuerzo para resolver el ticket. Campo editable por Manager.

**Horas reales (actualHours)**
Tiempo efectivo de resolución. Campo editable por Manager al cerrar el ticket.

**resolvedAt / resolvedBy**
Campos que registran la fecha/hora de cierre y el usuario que cerró el ticket. Se rellenan automáticamente cuando el ticket pasa a CLOSED.

**CHANGE_STATUS**
Permiso que permite ejecutar transiciones de estado. Activo para Manager con las restricciones de workflow definidas (no puede hacer ABIERTO→CERRADO).

**EDIT_TICKET**
Permiso que permite modificar campos del ticket (título, descripción, prioridad, tipo, fecha límite, horas). Activo para Manager.

**ARCHIVE_TICKET**
Permiso para marcar un ticket como archivado (oculto de vistas normales). **No disponible para Manager** — exclusivo de Admin.

**isArchived**
Campo booleano del ticket. Cuando es `true`, el ticket está archivado y no aparece en las vistas estándar. Solo Admin puede cambiar este campo.

**Tenant**
Organización o entidad propietaria de los datos. El sistema es multi-tenant: cada organización tiene sus propios tickets, usuarios y configuraciones. Los permisos se aplican por tenant.

**Matriz de permisos (role_permissions)**
Tabla en base de datos que define qué acciones puede realizar cada rol. Configurable por el Admin desde la pantalla de Gestión de Permisos. Los cambios se reflejan en el sistema tras recargar la sesión.
