---
rol: admin
tipo: permisos
audiencia: usuario_final
fuente: ticket-management
---

# Permisos — Admin

La pantalla **🔐 Gestión de Permisos** define qué puede hacer cada rol en tu organización. Solo el rol Admin puede abrirla y editarla. Este documento explica cómo está organizada, qué valores trae por defecto y qué reglas son fijas.

---

## Cómo está organizada la pantalla

Las acciones están agrupadas por la parte del portal que controlan. Cada grupo tiene un título y una explicación corta; debajo están sus acciones con las columnas **👤 Usuario**, **👔 Manager** y **🔑 Admin**.

### 🎫 Acceso a tickets — qué tickets puede ver y crear el rol

| Acción | Qué permite | Usuario | Manager | Admin |
|---|---|---|---|---|
| Ver tickets (VIEW_TICKETS) | Abrir el tablero, la lista y el detalle de los tickets que le corresponden | ✅ Fijo | ✅ Fijo | ✅ Fijo |
| Ver tickets sin asignar (VIEW_UNASSIGNED_TICKETS) | Ver también los tickets que aún no tienen responsable | ❌ | ✅ | ✅ Fijo |
| Ver todos los tickets (VIEW_ALL_TICKETS) | Ver todos los tickets de la organización, no solo los asignados al usuario | ❌ | ✅ | ✅ Fijo |
| Crear tickets (CREATE_TICKET) | Botón «+ Nuevo ticket» | ✅ | ✅ | ✅ Fijo |

### 🧾 Trabajo en el ticket — acciones dentro del detalle de un ticket

| Acción | Qué permite | Usuario | Manager | Admin |
|---|---|---|---|---|
| Editar tickets (EDIT_TICKET) | Cambiar título, descripción, prioridad, área y categoría, fecha límite y datos del solicitante | ❌ | ✅ | ✅ Fijo |
| Asignar tickets (ASSIGN_TICKET) | Elegir o quitar al responsable (incluye «Asignar a» al crear y al aprobar un borrador) | ❌ | ✅ | ✅ Fijo |
| Cambiar estado (CHANGE_STATUS) | Mover el ticket entre estados, desde el detalle o arrastrando en el tablero | ❌ | ✅ | ✅ Fijo |
| Agregar comentarios (ADD_COMMENT) | Escribir y editar comentarios | ✅ | ✅ | ✅ Fijo |
| Archivar tickets (ARCHIVE_TICKET) | Ocultar un ticket de las vistas (también cerrados) | 🔒 Solo Admin | 🔒 Solo Admin | ✅ Fijo |

### 📎 Adjuntos e historial — archivos del ticket y registro de lo que pasó

| Acción | Qué permite | Usuario | Manager | Admin |
|---|---|---|---|---|
| Ver adjuntos (VIEW_ATTACHMENTS) | Ver y descargar archivos | ✅ | ✅ | ✅ Fijo |
| Subir adjuntos (UPLOAD_ATTACHMENT) | Agregar archivos a un ticket | ❌ | ✅ | ✅ Fijo |
| Eliminar adjuntos (DELETE_ATTACHMENT) | Borrar archivos de un ticket | ❌ | ✅ | ✅ Fijo |
| Ver historial de eventos (VIEW_TICKET_EVENTS) | Ver quién hizo qué y cuándo en el ticket | ❌ | ✅ | ✅ Fijo |

### 📋 Bandeja de revisión — tickets que llegan por correo y esperan revisión

| Acción | Qué permite | Usuario | Manager | Admin |
|---|---|---|---|---|
| Revisar borradores (REVIEW_DRAFTS) | Abrir la Bandeja de revisión y aprobar borradores | ❌ | ✅ | ✅ Opcional |

### 📊 Métricas y asistente

| Acción | Qué permite | Usuario | Manager | Admin |
|---|---|---|---|---|
| Ver dashboard de métricas (VIEW_DASHBOARD) | Vista «Métricas» con KPIs, gráficas y rendimiento por agente | ❌ | ✅ | ✅ Fijo |
| Usar asistente (USE_AGENT) | Widget del asistente con respuestas de la documentación | ✅ | ✅ | ✅ Fijo |

### ⚙️ Administración — configuración de la organización

| Acción | Qué permite | Usuario | Manager | Admin |
|---|---|---|---|---|
| Gestionar usuarios (MANAGE_USERS) | Invitar y desactivar usuarios | 🔒 Solo Admin | 🔒 Solo Admin | ✅ Fijo |
| Gestionar áreas y categorías (MANAGE_CLASSIFICATIONS) | Crear, editar y desactivar áreas y categorías, y exportar la revisión del clasificador | 🔒 Solo Admin | 🔒 Solo Admin | ✅ Opcional |
| Gestionar workflow (MANAGE_WORKFLOW) | Estados, transiciones y notificaciones | 🔒 Solo Admin | 🔒 Solo Admin | ✅ Opcional |
| Gestionar permisos | Editar esta matriz | — | — | ✅ Siempre |

Los valores de Usuario y Manager son los que trae una organización nueva; tú puedes cambiarlos. "Gestionar permisos" es una fila informativa: la regla la aplica el sistema por rol y no se guarda en la matriz.

---

## Reglas fijas del sistema

Estas reglas no se pueden cambiar desde la pantalla. Las casillas fijas se ven con borde punteado y muestran su valor real.

- **Ver tickets** está siempre activo para todos los roles.
- **Solo Admin:** Archivar tickets, Gestionar usuarios, Gestionar áreas y categorías y Gestionar workflow no se pueden dar a Usuario ni a Manager.
- **Admin tiene fijas** todas las acciones excepto tres opcionales: Revisar borradores, Gestionar áreas y categorías y Gestionar workflow.
- **Gestionar permisos** es solo del rol Admin.

> **Precaución:** si quitas a Admin «Gestionar áreas y categorías» o «Gestionar workflow», nadie podrá abrir esas pantallas hasta que lo vuelvas a activar aquí.

---

## Asignar es independiente de editar

**Asignar tickets** es una acción propia, separada de **Editar tickets**:

- Con **Asignar tickets**, el rol ve los selectores **"Asignado a"** (detalle), **"Asignar a"** (Nuevo ticket) y **"Asignar a"** (Bandeja de revisión). Los selectores solo ofrecen usuarios activos de tu organización; el responsable actual sigue apareciendo aunque ya no esté activo.
- Sin **Asignar tickets**, el detalle muestra el responsable sin selector, Nuevo ticket no tiene "Asignar a" y los borradores se aprueban sin responsable.
- Un rol puede tener una sin la otra. Por ejemplo, un Manager con "Asignar tickets" y sin "Editar tickets" puede repartir el trabajo pero no cambiar prioridad, área ni categoría.

Al separar las dos acciones, cada rol conservó en "Asignar tickets" el mismo valor que tenía en "Editar tickets", así que nadie ganó ni perdió la capacidad de asignar. Revisa ambas al delegar trabajo.

---

## Cómo modificar permisos

1. Abre **Permisos** en el menú de la izquierda (solo administradores).
2. Busca la acción en su grupo y activa o desactiva el interruptor del rol.
3. Presiona **Guardar cambios**. Si sales sin guardar, los cambios se pierden. Si el guardado falla verás "No se pudieron guardar los cambios" con el motivo.

**Ejemplo:** para que los Usuarios puedan subir adjuntos, activa "Subir adjuntos" en la columna Usuario del grupo Adjuntos e historial y guarda.

**Cuándo aplica:** el servidor aplica la matriz en cuanto guardas; quien intente una acción que ya no tiene recibe "No tienes permiso para esta acción". Los botones y pantallas de los demás usuarios se actualizan la próxima vez que recarguen el portal o inicien sesión.

---

## Cambios de estado y permisos

Para cambiar el estado de un ticket hace falta el permiso CHANGE_STATUS ("Cambiar estado") y que exista una transición habilitada desde el estado actual en "⚙️ Gestión de Workflow". Las transiciones no tienen restricciones por rol: cualquier rol con el permiso puede usar cualquier transición activa. De fábrica, Manager y Admin tienen el permiso y Usuario no.

---

## Registro de cambios en permisos

Cada casilla guarda quién la cambió por última vez y cuándo. No se conserva el historial de valores anteriores.

---

## Preguntas frecuentes sobre permisos

**¿Puedo darle a un Manager el permiso de archivar tickets?**
No. Archivar tickets es exclusivo de Admin y no puede asignarse a otro rol.

**¿Se pueden archivar tickets cerrados?**
Sí. "🗄 Archivar ticket" aparece en el detalle en cualquier estado, incluido Cerrado.

**¿Puedo delegar la edición de permisos a un Manager?**
No. La fila "Gestionar permisos" muestra que solo Admin puede editar esta matriz.

**¿Puedo quitarle a un Manager el permiso de cambiar estados?**
Sí. Si lo desactivas, los Managers no podrán ejecutar ninguna transición de estado.

**¿Qué pasa si desactivo "Crear tickets" para el rol Usuario?**
Los usuarios no verán "+ Nuevo ticket". Seguirán pudiendo ver y comentar los tickets que les corresponden.

**¿Los permisos aplican al usuario específico o al rol?**
Al **rol**. Todos los usuarios del mismo rol tienen exactamente los mismos permisos. Para dar capacidades especiales a una persona, cambia su rol.

**¿Qué es el rol Soporte?**
Es un rol reservado: existe en el sistema pero hoy no tiene usuarios ni columna en la matriz. Se activará en un cambio futuro.

**¿Puedo tener dos Admins?**
Sí. Puede haber varios usuarios con rol Admin y todos tienen las mismas capacidades.
