---
rol: admin
tipo: configuracion
audiencia: usuario_final
fuente: ticket-management
---

# Configuración del sistema — Admin

Este documento cubre todas las tareas de configuración que solo el Admin puede realizar: usuarios, clasificaciones, workflow y permisos.

---

## 1. Gestión de usuarios

**Pantalla:** Gestión de Usuarios (`screen: userManagement`)
**Permiso requerido:** MANAGE_USERS

### Roles disponibles
El sistema tiene tres roles:
- `USER` — usuario estándar, acceso básico
- `MANAGER` — supervisor/gestor, puede gestionar tickets del equipo
- `ADMIN` — administrador, acceso total al sistema

### Crear usuario
1. Haz clic en **"Nuevo Usuario"**
2. Ingresa: nombre completo, email, rol, contraseña inicial
3. Guarda
4. El usuario puede iniciar sesión inmediatamente

### Cambiar el rol de un usuario
1. Busca el usuario en la lista
2. Edita su perfil
3. Cambia el rol
4. Guarda — el nuevo rol aplica desde la próxima sesión del usuario

**Implicaciones del cambio de rol:**
- USER → MANAGER: obtiene EDIT_TICKET, CHANGE_STATUS, UPLOAD_ATTACHMENT, DELETE_ATTACHMENT, VIEW_TICKET_EVENTS, VIEW_DASHBOARD, REVIEW_DRAFTS
- MANAGER → ADMIN: obtiene ARCHIVE_TICKET, MANAGE_USERS, MANAGE_CLASSIFICATIONS, MANAGE_WORKFLOW + todas las transiciones de estado
- ADMIN → MANAGER: pierde los 4 permisos exclusivos de Admin y la transición ABIERTO→CERRADO

### Desactivar un usuario
Cuando un usuario deja de tener acceso (por ejemplo, un empleado que se va de la empresa):
1. Edita el usuario
2. Marca como inactivo
3. Guarda

**Antes de desactivar:** reasigna manualmente los tickets que tiene asignados. El sistema no los reasigna automáticamente.

---

## 2. Gestión de clasificaciones

**Pantalla:** Gestión de Clasificaciones (`screen: classificationManagement`)
**Permiso requerido:** MANAGE_CLASSIFICATIONS

### Clasificaciones del sistema (por defecto)

| Nombre | Color | Descripción |
|---|---|---|
| IT | #0066FF | Tecnología e infraestructura |
| cliente | #FF6600 | Atención y soporte al cliente |
| operaciones | #009900 | Procesos y logística |
| otro | #999999 | Sin clasificación específica |

### Categorías por clasificación

**IT:** hardware, software, red, acceso, correo, impresora, vpn, servidor, base_de_datos, seguridad

**cliente:** facturacion, reclamo, consulta, devolucion, garantia, soporte, pedido, envio

**operaciones:** logistica, compras, inventario, mantenimiento, produccion, calidad, proveedores

**otro:** general, sin_clasificar

### Agregar una clasificación nueva
1. Haz clic en **"Nueva Clasificación"**
2. Define: nombre, color (formato hex ej. #FF0000), descripción
3. Guarda
4. La nueva clasificación aparece de inmediato en el selector de tickets y en el sistema de clasificación automática

### Agregar una categoría a una clasificación existente
1. Selecciona la clasificación
2. Haz clic en **"Agregar Categoría"**
3. Define el nombre (sin espacios, en minúsculas, usar guion bajo si es compuesto)
4. Guarda

### Editar una clasificación o categoría existente
- Cambiar el nombre de una clasificación afecta todos los tickets que ya la tienen asignada
- Cambiar el color solo afecta la visualización, no los datos
- Eliminar una clasificación podría afectar tickets existentes — el sistema puede no permitirlo si hay tickets activos con esa clasificación

---

## 3. Configuración del workflow

**Pantalla:** Configuración de Workflow (accesible desde Configuración del sistema)
**Permiso requerido:** MANAGE_WORKFLOW

**Advertencia crítica antes de modificar el workflow:** cualquier cambio afecta a todos los tickets activos en tiempo real. Planifica cambios en horario de baja actividad y notifica al equipo.

### Estados configurables

Por cada estado (OPEN, IN_PROGRESS, IN_REVIEW, CLOSED) puedes modificar:

| Campo | Descripción | Impacto del cambio |
|---|---|---|
| name | Nombre visible en la UI | Cambio cosmético, no afecta datos |
| description | Descripción del estado | Solo informativo |
| color | Color hex en la interfaz | Cambio cosmético |
| notifyRequester | Si notifica al solicitante al entrar a este estado | Activa/desactiva notificaciones automáticas |
| position | Orden de visualización | Solo afecta la presentación |

**Campos que NO debes cambiar:** isInitial e isTerminal — cambiarlos podría romper la lógica de creación y cierre de tickets.

### Transiciones configurables

Por cada transición puedes modificar:

| Campo | Descripción | Impacto |
|---|---|---|
| fromState | Estado de origen | Cambia el mapa del workflow |
| toState | Estado de destino | Cambia el mapa del workflow |
| requiredRole | NULL / MANAGER / ADMIN | Cambia quién puede ejecutar la transición |
| description | Descripción de la transición | Solo informativo |

**Ejemplo de uso:** si quieres que los Managers también puedan cerrar tickets directamente desde ABIERTO (en lugar de solo el Admin), cambia el `requiredRole` de la transición OPEN→CLOSED de `ADMIN` a `NULL`.

### Configuración actual de transiciones

| De | A | Rol requerido | Descripción |
|---|---|---|---|
| OPEN | IN_PROGRESS | NULL | Cualquier rol con CHANGE_STATUS |
| IN_PROGRESS | IN_REVIEW | NULL | Cualquier rol con CHANGE_STATUS |
| IN_REVIEW | IN_PROGRESS | NULL | Regresión si hay problemas |
| IN_REVIEW | CLOSED | MANAGER | Solo Manager o Admin cierran |
| OPEN | CLOSED | ADMIN | Cierre directo, exclusivo Admin |

---

## 4. Gestión de permisos

**Pantalla:** Gestión de Permisos (`screen: permissionManagement`)
**Permiso requerido:** implícito en ADMIN

Ver documento `admin/permisos.md` para la referencia completa.

**Flujo rápido:**
1. Abre Gestión de Permisos
2. Localiza el par (rol, acción) que quieres cambiar
3. Activa o desactiva
4. Guarda — queda registrado con tu usuario

---

## 5. Revisión de borradores (vista Admin)

Además de aprobar borradores (igual que Manager), Admin puede **eliminarlos**.

**Cuándo eliminar vs aprobar:**
- **Aprobar:** el borrador es una solicitud legítima que debe entrar al flujo
- **Eliminar:** el borrador es spam, duplicado, o fue creado por error del sistema de ingesta

**Pasos para eliminar:**
1. Ve a Revisión de Borradores
2. Abre el borrador
3. Selecciona **"Eliminar"**
4. Confirma — irreversible

---

## 6. Archivar tickets

**Permiso requerido:** ARCHIVE_TICKET (exclusivo Admin)

**Pasos:**
1. Abre el detalle del ticket a archivar
2. Selecciona **"Archivar"** en el menú de acciones
3. Confirma

**Resultado:** `isArchived = true`. El ticket deja de aparecer en búsquedas y listas normales pero permanece en la base de datos.

**Para ver tickets archivados:** en la lista principal activa el filtro **"Incluir archivados"** (si está disponible en la interfaz) o consulta directamente en la base de datos.

---

## Buenas prácticas para Admin

1. **Cambios de workflow:** siempre comunica al equipo antes de modificar transiciones o estados
2. **Cambios de permisos:** documenta por qué cambiaste un permiso — usa los comentarios del sistema si están disponibles
3. **Desactivación de usuarios:** reasigna tickets antes de desactivar
4. **Eliminación de borradores:** revisa el contenido antes de eliminar — puede ser una solicitud legítima mal formateada
5. **Clasificaciones:** antes de agregar una nueva, verifica que no existe una similar con otro nombre
6. **Múltiples Admins:** es recomendable tener al menos dos Admins para evitar puntos únicos de falla en la administración
