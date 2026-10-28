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
Registro formal de una solicitud, problema o tarea. Modelo central del sistema. Campos clave: `id` (UUID), `title`, `description`, `status`, `priority`, `type`, `createdBy`, `assignedTo`, `createdAt`, `updatedAt`, `closedAt`, `closedBy`, `closedOutcome`, `isArchived`, `isDraft`, `classificationId`, `categoryId`, `requesterEmail`, `requesterName`.

**Estado**
Etapa del ticket en el workflow de la organización. Cada estado tiene un nombre libre, un código técnico fijo (`status` en la API, por ejemplo `OPEN`) y una categoría del sistema: Nuevo, En curso, En espera o Cerrado (con resultado Resuelto o Cancelado). El comportamiento sigue la categoría. Es nulo cuando el ticket es un borrador.

**isDraft**
Campo derivado del modelo Ticket. Es `true` cuando `status == null`. No es un estado sino una condición especial que indica que el ticket no ha entrado al flujo normal. Los borradores se gestionan en la pantalla de Revisión de Borradores.

**isArchived**
Campo booleano del ticket (`false` por defecto). Cuando es `true`, el ticket está oculto de vistas normales. Solo Admin puede cambiarlo. Diferente al cierre: un ticket archivado conserva su estado y ya no cambia de estado ni se edita.

**Estado inicial (isInitial)**
Estado donde empiezan todos los tickets nuevos (portal, correo y borradores aprobados). Debe ser de categoría Nuevo; se elige con "Hacer inicial" en "⚙️ Gestión de Workflow".

**Categoría Cerrado (isTerminal)**
Un ticket está cerrado si su estado es de categoría Cerrado, sea cual sea su nombre. `isTerminal` es un valor derivado de la categoría. Se puede reabrir si la organización habilitó una transición que salga de ese estado.

**Transición (WorkflowTransition)**
Cambio permitido de un estado a otro. Definida en la tabla `workflow_transitions` con campos: `fromState`, `toState`, `requiredRole` (nullable), `description`. El servicio `TicketService.changeStatus()` valida que la transición exista en la base de datos antes de ejecutarla.

**requiredRole**
Columna de `workflow_transitions` (valores de fábrica: `NULL`, `MANAGER`, `ADMIN`). **No se aplica**: cualquier rol con CHANGE_STATUS ejecuta las transiciones activas. La pantalla de workflow no la muestra ni la edita, y al reactivar una transición queda en `NULL`.

**closedAt / closedBy / closedOutcome**
Datos de cierre: se guardan al entrar a un estado Cerrado (fecha, usuario y resultado Resuelto o Cancelado) y se limpian al reabrir.

**Borrador (Draft)**
Ticket con `status = NULL`. Creado por el sistema de ingesta automática (ticket-ingestion-light/n8n) cuando procesa correos entrantes que necesitan revisión humana. Los borradores están en la tabla `tickets` con `status_id = NULL`. Manager puede aprobarlos; Admin puede aprobarlos o eliminarlos.

**approveDraft()**
Método en `TicketService` (línea 61) que activa un borrador: pasa el ticket al estado inicial de la organización, aplica los campos editados en la revisión (prioridad, área, categoría, asignado), y registra la acción. Requiere permiso REVIEW_DRAFTS.

**WorkflowState**
Entidad persistida en la tabla `workflow_states`. Representa un estado del workflow de una organización: código, nombre, descripción, color, categoría, resultado, isInitial, isActive, position y notifyRequester. Cada workflow_state tiene un `id` entero que referencian los tickets (`status_id`). Los estados no se borran: se desactivan moviendo sus tickets a otro estado.

**WorkflowTransition**
Entidad persistida en `workflow_transitions`. Representa una transición válida entre dos WorkflowStates. Campos: `fromState` (FK), `toState` (FK), `requiredRole`, `description`.

**notifyRequester**
Campo booleano de WorkflowState ("Avisar al entrar"). Si es `true`, el solicitante recibe correo cuando el ticket entra a ese estado, siempre que el ticket tenga activados los avisos al solicitante y este tenga correo. La creación, la asignación y los comentarios tienen sus propios interruptores. Configurable por Admin en "⚙️ Gestión de Workflow" › Notificaciones.

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

**Área (Classification)**
Primer nivel del catálogo: el área a la que pertenece un ticket. Tabla: `ticket_classifications`. Campos: nombre, color, descripción, activa, tenant_id. Las cuatro áreas por defecto son IT, cliente, operaciones, otro. El nombre es único sin distinguir mayúsculas.

> **Equivalencia de nombres:** "Área" es el nombre en pantalla; en la API y la base de datos se llama *classification* y el clasificador automático la llama *dominio*. Es el mismo dato.


**Categoría (Category)**
Segundo nivel del catálogo, dentro de un área. Tabla: `ticket_categories`. Vinculada a su área por FK. Ejemplos dentro de IT: hardware, software, red, acceso, vpn.

**classificationHint / categoryHint**
Campos del ticket que guardan el valor original de área/categoría cuando el clasificador no encontró una coincidencia exacta en el catálogo. Útiles para detectar la necesidad de agregar nuevas categorías al sistema.

**ticket_classification_review (bitácora del clasificador)**
Guarda, por cada ticket llegado por correo, el área y la categoría que propuso el clasificador, su confianza y la decisión humana final (quién y cuándo). No se muestra en las pantallas de tickets; se exporta desde "🏷️ Áreas y categorías" → "⬇ Exportar revisión del clasificador".

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

**dueDate**
Fecha límite del ticket (LocalDate). Opcional; no puede ser anterior a hoy (zona `app.timezone`, por defecto America/Mexico_City). Alimenta el SLA y los vencidos del dashboard.

**notifyRequester / contactMedium**
Avisos al solicitante del ticket (casilla; siempre falsa sin correo) y medio por el que llegó (PHONE, IN_PERSON, EMAIL, CHAT, OTHER). Se notifica solo si el estado avisa, la casilla está marcada y hay correo; REQUESTER_CHANGED y DUE_DATE_CHANGED nunca se notifican.

**estimatedHours / actualHours**
Campos de esfuerzo del ticket (Double). `estimatedHours` es la estimación; `actualHours` es el tiempo real registrado al cierre.

**usePermissionStore**
Store de permisos en el frontend (React/Redux). Métodos: `fetchPermissions()` carga la matriz desde el backend; `can(role, action)` verifica un permiso en el cliente; `updatePermissions()` actualiza permisos (UI + backend). Los permisos se cargan al inicio de la sesión.

**helpContent.ts**
Archivo del frontend que contiene el contenido de ayuda contextual por pantalla. Exporta `HELP_CONTENT` indexado por `screen` (ej: newTicket, ticketDetail, attachments). Cada entrada tiene `title` y `sections` con `items`. Este contenido aparece cuando el usuario hace clic en el botón `?` de cada pantalla.

**HelpButton.tsx**
Componente React que renderiza el botón `?` en las pantallas con ayuda contextual. El título del modal es "Ayuda de esta pantalla". Las pantallas con este botón son: TicketModal, TicketDetail, AttachmentModal, EventsModal, DraftReview, ClassificationManagement, UserManagement, PermissionManagement.
