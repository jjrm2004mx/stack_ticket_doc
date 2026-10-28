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
Registro formal de una solicitud, problema o tarea. Cada ticket tiene: identificador único (UUID), estado, prioridad, asignado, creador, solicitante, área y categoría. Es la unidad central del sistema.

**Estado (Status)**
Etapa actual del ticket en el workflow de tu organización. Cada estado tiene un nombre que eligió la organización y una categoría: Nuevo, En curso, En espera o Cerrado (Resuelto o Cancelado). Con "Cambiar estado" puedes mover el ticket por las transiciones configuradas.

**Estado inicial**
El estado de categoría Nuevo donde empiezan todos los tickets nuevos (de fábrica, "Abierto").

**Estado Cerrado**
Cualquier estado de categoría Cerrado. Un ticket cerrado no se edita; se reabre solo si la organización configuró una transición que salga de ese estado.

**Transición**
Cambio de un estado a otro. El sistema valida que la transición sea permitida antes de ejecutarla. Las transiciones están definidas en la tabla `workflow_transitions` de la base de datos.

**Historial de estados (StatusHistory)**
Registro de todos los cambios de estado del ticket. Cada entrada incluye: estado anterior (`from_status`), estado nuevo (`to_status`), usuario que hizo el cambio (`changed_by`), fecha y hora exacta (`changed_at`).

**Evento (TicketEvent)**
Registro de cualquier acción realizada sobre el ticket: cambio de estado, edición de campos, comentarios, adjuntos, asignación. Cada evento incluye tipo de evento, usuario, fecha/hora, y valores anterior/nuevo.

**Borrador (Draft)**
Ticket sin estado activo (`status = NULL`). Creado automáticamente cuando llega un ticket por correo electrónico y necesita revisión humana. No aparece en el flujo normal hasta ser aprobado. Como Manager eres responsable de revisarlos y aprobarlos.

**Aprobar borrador**
Acción que activa un borrador: el ticket entra al estado inicial de la organización. Al aprobar, el ticket entra al flujo normal con la prioridad, el área, la categoría y la asignación que definiste en la revisión.

**REVIEW_DRAFTS**
Permiso que permite revisar y aprobar borradores. Activo por defecto para Manager.

**Prioridad**
Urgencia del ticket:
- `BAJA` — puede esperar
- `MEDIA` — atención en tiempo razonable
- `ALTA` — afecta el trabajo, atención pronta
- `CRÍTICA` — bloqueante total, atención inmediata

**Asignado a (assignedTo)**
Usuario del equipo responsable de resolver el ticket. Campo UUID que referencia a un usuario del sistema. Como Manager puedes asignar y reasignar tickets.

**Solicitante (Requester)**
Persona a nombre de quien se crea el ticket (por ejemplo, un cliente que llamó o escribió). Quien crea el ticket (`createdBy`) nunca es el solicitante. Si el ticket llegó por correo, el solicitante es el remitente.

**requesterEmail / requesterName**
Nombre y correo del solicitante. Se capturan al crear el ticket en el portal o vienen del correo; con "Editar tickets" se corrigen desde el detalle (evento "Solicitante actualizado"). Cambiar el solicitante no cambia quién ve el ticket.

**notifyRequester / contactMedium**
Casilla "Enviar notificaciones al solicitante" (solo con correo) y medio por el que llegó la solicitud (Teléfono, Presencial, Correo, Chat, Otro). Se avisa al solicitante solo si el estado del ticket envía avisos, la casilla está marcada y hay correo. Las correcciones de solicitante y fecha límite nunca se notifican.

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

**Área**
Primer nivel del catálogo: el área a la que pertenece el ticket. Las cuatro áreas por defecto son:
- `IT` — tecnología e infraestructura
- `cliente` — atención y soporte al cliente
- `operaciones` — procesos y logística
- `otro` — sin clasificación específica

> **Equivalencia de nombres:** "Área" es el nombre en pantalla; en la API y la base de datos se llama *classification* y el clasificador automático la llama *dominio*. Es el mismo dato.

**Categoría (Category)**
Segundo nivel del catálogo, dentro de un área. Ejemplo: dentro de IT existen hardware, software, red, acceso, correo, impresora, VPN, servidor, base_de_datos, seguridad.

**classificationHint / categoryHint**
Campos del ticket que guardan el área/categoría sugerida por el clasificador cuando no encontró una coincidencia exacta en el catálogo. Útil para revisar borradores y entender la intención original del clasificador.

**Fecha límite (dueDate)**
Fecha de resolución esperada del ticket. Se define al crear o desde el detalle (evento "Fecha límite cambiada"); no puede ser anterior a hoy. Alimenta el cumplimiento de SLA y los vencidos del dashboard.

**Horas estimadas (estimatedHours)**
Estimación de esfuerzo para resolver el ticket. Campo editable por Manager.

**Horas reales (actualHours)**
Tiempo efectivo de resolución. Campo editable por Manager al cerrar el ticket.

**closedAt / closedBy / closedOutcome**
Fecha, usuario y resultado (Resuelto o Cancelado) del cierre. Se guardan al entrar a un estado Cerrado y se limpian si el ticket se reabre.

**CHANGE_STATUS**
Permiso que permite ejecutar transiciones de estado. Permite mover tickets por las transiciones que configuró la organización; no hay restricciones por rol en las transiciones.

**EDIT_TICKET**
Permiso que permite modificar campos del ticket (prioridad, fecha límite, solicitante, avisos, medio, área y categoría). Activo para Manager.

**ARCHIVE_TICKET**
Permiso para marcar un ticket como archivado (oculto de vistas normales). **No disponible para Manager** — exclusivo de Admin.

**isArchived**
Campo booleano del ticket. Cuando es `true`, el ticket está archivado y no aparece en las vistas estándar. Solo Admin puede cambiar este campo.

**Tenant**
Organización o entidad propietaria de los datos. El sistema es multi-tenant: cada organización tiene sus propios tickets, usuarios y configuraciones. Los permisos se aplican por tenant.

**Matriz de permisos (role_permissions)**
Tabla en base de datos que define qué acciones puede realizar cada rol. Configurable por el Admin desde la pantalla de Gestión de Permisos. Los cambios se reflejan en el sistema tras recargar la sesión.
