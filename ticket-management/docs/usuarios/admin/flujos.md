---
rol: admin
tipo: flujos
audiencia: usuario_final
fuente: ticket-management
---

# Flujos principales — Admin

El Admin tiene acceso completo a todas las funciones del sistema: gestión de tickets, usuarios, clasificaciones, workflow y permisos. Este documento cubre todas las operaciones disponibles.

---

## 1. Gestión de tickets (todas las operaciones)

Como Admin heredas todas las capacidades de Manager más las exclusivas de Admin.

**Capacidades completas sobre tickets:**
- Ver todos los tickets del sistema
- Crear tickets nuevos
- Editar cualquier campo de cualquier ticket
- Cambiar estado (incluyendo ABIERTO → CERRADO directamente)
- Asignar y reasignar tickets
- Agregar comentarios
- Ver y descargar adjuntos
- Subir adjuntos
- Eliminar adjuntos
- Ver historial completo de eventos
- Ver dashboard de métricas
- Revisar y aprobar borradores
- **Eliminar borradores** (exclusivo Admin)
- **Archivar tickets** (exclusivo Admin)

**Para los flujos de gestión de tickets (crear, editar, cambiar estado, adjuntos, comentarios, borradores):** consulta el documento `manager/flujos.md` — los pasos son idénticos para Admin.

---

## 2. Gestionar usuarios (MANAGE_USERS)

Esta es una función exclusiva de Admin. Permite crear, modificar y controlar los accesos de todos los usuarios del sistema.

**Cómo acceder:** menú principal → **"Gestión de Usuarios"**

### Crear un usuario nuevo
1. En la pantalla de Gestión de Usuarios haz clic en **"Nuevo Usuario"**
2. Completa los campos requeridos:
   - Nombre completo
   - Email (será el identificador de inicio de sesión)
   - Rol: `USER`, `MANAGER` o `ADMIN`
   - Contraseña inicial (el usuario deberá cambiarla)
3. Guarda el usuario

**Nota:** el rol del usuario determina qué permisos tiene en el sistema según la matriz de `role_permissions`.

### Editar un usuario existente
1. Busca el usuario en la lista
2. Haz clic en **"Editar"**
3. Puedes modificar: nombre, rol, estado (activo/inactivo)
4. **Cambiar el rol** de un usuario tiene efecto inmediato en sus permisos desde la próxima vez que inicie sesión

### Desactivar un usuario
Si un usuario ya no debe tener acceso al sistema, desactívalo (no lo elimines — se preserva el historial de sus acciones).

**Precaución:** desactivar a un usuario no reasigna automáticamente sus tickets abiertos. Debes reasignarlos manualmente antes o después de desactivarlo.

---

## 3. Gestionar clasificaciones (MANAGE_CLASSIFICATIONS)

Las clasificaciones y categorías determinan cómo se organiza el conocimiento de los tickets.

**Cómo acceder:** menú principal → **"Gestión de Clasificaciones"**

### Clasificaciones disponibles por defecto:
- `IT` — tecnología e infraestructura (color #0066FF)
- `cliente` — atención al cliente (color #FF6600)
- `operaciones` — procesos y logística (color #009900)
- `otro` — sin clasificación específica (color #999999)

### Categorías por clasificación:
- **IT:** hardware, software, red, acceso, correo, impresora, vpn, servidor, base_de_datos, seguridad
- **cliente:** facturacion, reclamo, consulta, devolucion, garantia, soporte, pedido, envio
- **operaciones:** logistica, compras, inventario, mantenimiento, produccion, calidad, proveedores
- **otro:** general, sin_clasificar

### Agregar una clasificación nueva
1. Haz clic en **"Nueva Clasificación"**
2. Define: nombre, color (hex), descripción
3. Guarda

### Agregar una categoría a una clasificación
1. Selecciona la clasificación
2. Haz clic en **"Agregar Categoría"**
3. Define el nombre de la categoría
4. Guarda

**Impacto:** las nuevas clasificaciones y categorías quedan disponibles de inmediato para asignar a tickets y para el sistema de clasificación automática (IA).

---

## 4. Configurar el workflow (MANAGE_WORKFLOW)

El workflow define los estados del sistema y las transiciones permitidas entre ellos.

**Cómo acceder:** menú principal → **"Configuración de Workflow"**

### Estados configurables:
Por cada estado puedes modificar:
- Nombre visible en la interfaz
- Descripción
- Color (hex)
- Si notifica automáticamente al solicitante al llegar a este estado (`notifyRequester`)
- Posición de visualización

### Transiciones configurables:
Por cada transición puedes definir:
- Estado de origen
- Estado de destino
- Rol requerido (`NULL` = cualquier rol con CHANGE_STATUS, o especificar `MANAGER` / `ADMIN`)
- Descripción

**Advertencia crítica:** los cambios en el workflow afectan a TODOS los tickets activos en tiempo real. Una transición eliminada o modificada puede bloquear tickets en progreso. Planifica los cambios de workflow fuera de horario de trabajo y comunica los cambios al equipo antes de aplicarlos.

**Configuración actual de transiciones (defaults del sistema):**

| De | A | Rol requerido |
|---|---|---|
| OPEN | IN_PROGRESS | Sin restricción |
| IN_PROGRESS | IN_REVIEW | Sin restricción |
| IN_REVIEW | IN_PROGRESS | Sin restricción |
| IN_REVIEW | CLOSED | MANAGER |
| OPEN | CLOSED | ADMIN |

---

## 5. Gestionar permisos (pantalla Gestión de Permisos)

La matriz de permisos define qué puede hacer cada rol. Como Admin puedes modificarla.

**Cómo acceder:** menú principal → **"Gestión de Permisos"**

### Cómo funciona la matriz:
- Hay una fila por cada combinación de rol + acción
- Cada combinación tiene un valor `allowed` (true/false)
- Los cambios se aplican a TODOS los usuarios con ese rol

### Permisos que PUEDES cambiar para roles Manager y Usuario:
CREATE_TICKET, EDIT_TICKET, CHANGE_STATUS, ADD_COMMENT, VIEW_ATTACHMENTS, UPLOAD_ATTACHMENT, DELETE_ATTACHMENT, VIEW_TICKET_EVENTS, VIEW_DASHBOARD, REVIEW_DRAFTS

### Permisos que NO PUEDES cambiar (bloqueados por el sistema):
- `VIEW_TICKETS` — siempre TRUE para todos, no modificable
- `ARCHIVE_TICKET` — solo Admin, no transferible a Manager ni Usuario
- `MANAGE_USERS` — solo Admin, no transferible
- `MANAGE_CLASSIFICATIONS` — solo Admin, no transferible
- `MANAGE_WORKFLOW` — solo Admin, no transferible

### Actualizar un permiso:
1. En la pantalla de Gestión de Permisos verás la matriz completa
2. Activa o desactiva el permiso del rol deseado
3. Guarda los cambios
4. Los usuarios afectados verán los nuevos permisos en su próxima sesión

**Registro:** todos los cambios en la matriz de permisos quedan registrados con quién los hizo (`updatedBy`) y cuándo.

---

## 6. Archivar tickets (ARCHIVE_TICKET)

Archivar oculta un ticket de las vistas normales sin eliminarlo ni cambiar su estado.

**Cuándo archivar:**
- Tickets muy antiguos que ya no son relevantes operativamente
- Limpieza de la interfaz sin perder el historial
- Casos cerrados hace tiempo que no necesitan visibilidad diaria

**Pasos:**
1. Abre el detalle del ticket
2. Busca la opción **"Archivar"** en el menú de acciones
3. Confirma la acción

**Efecto:** el campo `isArchived` pasa a `true`. El ticket deja de aparecer en búsquedas y listas estándar pero sigue en la base de datos.

**Importante:** archivar no cierra el ticket. Si quieres archivar y cerrar, primero ciérralo y luego archívalo.

---

## 7. Eliminar borradores

A diferencia de Manager (que solo puede aprobar), Admin puede eliminar borradores permanentemente.

**Cuándo eliminar un borrador:**
- Correos basura (spam) que llegaron al sistema de ingesta
- Solicitudes duplicadas que ya tienen un ticket activo
- Borradores creados por error de configuración del sistema de ingesta

**Pasos:**
1. Ve a la sección **"Revisión de Borradores"**
2. Abre el borrador que quieres eliminar
3. Selecciona **"Eliminar"** en las opciones
4. Confirma — la acción es irreversible

**Advertencia:** la eliminación de un borrador es permanente. A diferencia de los tickets archivados, un borrador eliminado no puede recuperarse.

---

## Resumen de permisos exclusivos de Admin

| Acción exclusiva de Admin | Descripción |
|---|---|
| ABIERTO → CERRADO directamente | Cerrar sin pasar por EN PROGRESO ni EN REVISIÓN |
| ARCHIVE_TICKET | Archivar cualquier ticket |
| MANAGE_USERS | Crear, editar y desactivar usuarios |
| MANAGE_CLASSIFICATIONS | Gestionar clasificaciones y categorías |
| MANAGE_WORKFLOW | Configurar estados y transiciones del sistema |
| Eliminar borradores | Los Managers solo pueden aprobar |
| Actualizar matriz de permisos | Cambiar qué puede hacer cada rol |
