---
rol: admin
tipo: glosario
audiencia: usuario_final
fuente: ticket-management
---

# Glosario completo — Admin

Definiciones de todos los términos del sistema con contexto técnico y de negocio para el rol Admin.

---

**Ticket**
Registro formal de una solicitud, problema o tarea. Modelo central del sistema. Campos clave: `id` (UUID), `title`, `description`, `status`, `priority`, `type`, `createdBy`, `assignedTo`, `createdAt`, `updatedAt`, `resolvedAt`, `resolvedBy`, `isArchived`, `isDraft`, `classificationId`, `categoryId`, `requesterEmail`, `requesterName`.

**Status (Estado)**
Enum con cuatro valores: OPEN, IN_PROGRESS, IN_REVIEW, CLOSED. Campo `status` en el modelo Ticket. Puede ser NULL cuando el ticket es un borrador.

**isDraft**
Campo derivado del modelo Ticket. Es `true` cuando `status == null`. No es un valor del enum Status sino una condición especial que indica que el ticket no ha entrado al flujo normal. Los borradores se gestionan en la pantalla de Revisión de Borradores.

**isArchived**
Campo booleano del ticket (`false` por defecto). Cuando es `true`, el ticket está oculto de vistas normales. Solo Admin puede cambiarlo. Diferente al cierre: un ticket puede estar ABIERTO y archivado simultáneamente.

**Estado inicial (isInitial)**
Estado con el que comienzan todos los tickets: OPEN. Definido en la entidad `WorkflowStateEntity` y en la tabla `workflow_states`.

**Estado terminal (isTerminal)**
Estado del que no hay transiciones de salida: CLOSED. Una vez que un ticket llega a CLOSED, no puede cambiar de estado bajo ninguna circunstancia ni con ningún rol.

**Transición (WorkflowTransition)**
Cambio permitido de un estado a otro. Definida en la tabla `workflow_transitions` con campos: `fromState`, `toState`, `requiredRole` (nullable), `description`. El servicio `TicketService.changeStatus()` valida que la transición exista en la base de datos antes de ejecutarla.

**requiredRole**
Campo de una transición que define qué rol puede ejecutarla. Valores: `NULL` (cualquier rol con CHANGE_STATUS), `MANAGER` (Manager o Admin), `ADMIN` (solo Admin). Configurable desde la pantalla de workflow.

**resolvedAt / resolvedBy**
Campos del ticket que se rellenan automáticamente cuando el ticket pasa a estado CLOSED. `resolvedAt` es la fecha/hora exacta, `resolvedBy` es el UUID del usuario que ejecutó el cierre.

**Borrador (Draft)**
Ticket con `status = NULL`. Creado por el sistema de ingesta automática (ticket-ingestion-light/n8n) cuando procesa correos entrantes que necesitan revisión humana. Los borradores están en la tabla `tickets` con `status_id = NULL`. Manager puede aprobarlos; Admin puede aprobarlos o eliminarlos.

**approveDraft()**
Método en `TicketService` (línea 61) que activa un borrador: cambia el status de NULL a OPEN, aplica los campos editados en la revisión (prioridad, clasificación, asignado), y registra la acción. Requiere permiso REVIEW_DRAFTS.

**WorkflowState**
Entidad persistida en la tabla `workflow_states`. Representa un estado del sistema con toda su configuración: nombre, descripción, color, isInitial, isTerminal, position, notifyRequester. Cada workflow_state tiene un `id` UUID que es referenciado por los tickets.

**WorkflowTransition**
Entidad persistida en `workflow_transitions`. Representa una transición válida entre dos WorkflowStates. Campos: `fromState` (FK), `toState` (FK), `requiredRole`, `description`.

**notifyRequester**
Campo booleano de WorkflowState. Si es `true`, el sistema envía una notificación automática al solicitante del ticket cuando este entra a ese estado. Configurable por Admin.

**Action (Permiso)**
Enum con 15 valores que representan las acciones posibles en el sistema: VIEW_TICKETS, CREATE_TICKET, EDIT_TICKET, CHANGE_STATUS, ARCHIVE_TICKET, ADD_COMMENT, VIEW_ATTACHMENTS, UPLOAD_ATTACHMENT, DELETE_ATTACHMENT, VIEW_TICKET_EVENTS, VIEW_DASHBOARD, MANAGE_USERS, REVIEW_DRAFTS, MANAGE_CLASSIFICATIONS, MANAGE_WORKFLOW.

**role_permissions**
Tabla en base de datos que almacena la matriz de permisos: `(tenant_id, role, action, allowed)`. Un registro por cada combinación posible de rol y acción. Modificable por Admin desde la pantalla de Gestión de Permisos.

**isLockedAlwaysTrue()**
Método en el enum Action. Las acciones que retornan true aquí (VIEW_TICKETS) son siempre TRUE para todos los roles y no pueden ser modificadas.

**isAdminOnly()**
Método en Action. Las acciones que retornan true aquí (ARCHIVE_TICKET) solo pueden ser TRUE para Admin y no pueden asignarse a otros roles.

**isLockedForAdmin()**
Acciones que son fijas en el Admin (no configurables para Admin).

**isAdminOptional()**
Acciones que el Admin puede activar o desactivar para sí mismo.

**PermissionService**
Servicio backend (`application/PermissionService.java`) que gestiona la lógica de permisos. Métodos: `getMatrix()` retorna la matriz completa; `isAllowed(role, action)` verifica un permiso; `updateMatrix(updates, updatedBy)` actualiza permisos (solo Admin puede llamar este método).

**Tenant / TenantId**
Identificador de la organización. El sistema es multi-tenant: cada empresa u organización tiene sus propios datos, usuarios y configuraciones aislados. Los permisos en `role_permissions` están asociados a un `tenant_id`.

**Clasificación (Classification)**
Dominio de conocimiento al que pertenece un ticket. Tabla: `ticket_classifications`. Campos: nombre, color, descripción, tenant_id. Las cuatro clasificaciones por defecto son IT, cliente, operaciones, otro.

**Categoría (Category)**
Subcategoría dentro de una Clasificación. Tabla: `ticket_categories`. Vinculada a una clasificación por FK. Ejemplos dentro de IT: hardware, software, red, acceso, vpn.

**classificationHint / categoryHint**
Campos del ticket que guardan el valor original de clasificación/categoría cuando el sistema de IA no encontró una coincidencia exacta en el catálogo. Útiles para detectar la necesidad de agregar nuevas categorías al sistema.

**ticket_email_content**
Tabla auxiliar que guarda el HTML del correo original para tickets creados por integración de email. El campo `emailBodyHtml` en el modelo Ticket hace referencia a este contenido.

**requesterEmail / requesterName**
Campos del ticket para solicitudes externas. Cuando un correo electrónico genera un ticket via ticket-ingestion-light, el email del remitente se guarda en `requesterEmail` y su nombre en `requesterName`.

**StatusHistory (ticket_status_history)**
Tabla de auditoría que registra todos los cambios de estado: `ticket_id`, `from_status`, `to_status`, `changed_by` (UUID), `changed_at` (timestamp). Se crea una entrada por cada cambio de estado ejecutado.

**TicketEvent (ticket_events)**
Tabla de auditoría más amplia que registra cualquier cambio en el ticket: `event_type`, `performed_by` (UUID), `performed_at` (timestamp), `old_value`, `new_value`. Abarca cambios de estado, ediciones de campos, comentarios, adjuntos, asignaciones.

**Priority**
Enum con cuatro valores: BAJA, MEDIA, ALTA, CRITICA. Campo `priority` del ticket.

**TicketType**
Enum de tipo de ticket: TASK, BUG, FEATURE, DOCS.

**dueDate**
Fecha límite del ticket (LocalDate). Opcional. Usada para priorización del equipo.

**estimatedHours / actualHours**
Campos de esfuerzo del ticket (Double). `estimatedHours` es la estimación; `actualHours` es el tiempo real registrado al cierre.

**usePermissionStore**
Store de permisos en el frontend (React/Redux). Métodos: `fetchPermissions()` carga la matriz desde el backend; `can(role, action)` verifica un permiso en el cliente; `updatePermissions()` actualiza permisos (UI + backend). Los permisos se cargan al inicio de la sesión.

**helpContent.ts**
Archivo del frontend que contiene el contenido de ayuda contextual por pantalla. Exporta `HELP_CONTENT` indexado por `screen` (ej: newTicket, ticketDetail, attachments). Cada entrada tiene `title` y `sections` con `items`. Este contenido aparece cuando el usuario hace clic en el botón `?` de cada pantalla.

**HelpButton.tsx**
Componente React que renderiza el botón `?` en las pantallas con ayuda contextual. El título del modal es "Ayuda de esta pantalla". Las pantallas con este botón son: TicketModal, TicketDetail, AttachmentModal, EventsModal, DraftReview, ClassificationManagement, UserManagement, PermissionManagement.
