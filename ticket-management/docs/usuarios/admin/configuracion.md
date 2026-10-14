---
rol: admin
tipo: configuracion
audiencia: usuario_final
fuente: ticket-management
---

# Configuración del sistema — Admin

Este documento cubre todas las tareas de configuración que solo el Admin puede realizar: usuarios, áreas y categorías, workflow y permisos.

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

## 2. Gestión de áreas y categorías

**Pantalla:** 🏷️ Áreas y categorías (`screen: classificationManagement`)
**Permiso requerido:** MANAGE_CLASSIFICATIONS ("Gestionar áreas y categorías")

El catálogo tiene dos niveles: el **área** (por ejemplo, IT) y la **categoría** dentro de ella (por ejemplo, impresora). El clasificador automático de correos asigna cada ticket a un área y una categoría de este catálogo.

### Áreas del sistema (por defecto)

| Nombre | Color | Descripción |
|---|---|---|
| IT | #0066FF | Tecnología e infraestructura |
| cliente | #FF6600 | Atención y soporte al cliente |
| operaciones | #009900 | Procesos y logística |
| otro | #999999 | Sin clasificación específica |

### Categorías por área

**IT:** hardware, software, red, acceso, correo, impresora, vpn, servidor, base_de_datos, seguridad

**cliente:** facturacion, reclamo, consulta, devolucion, garantia, soporte, pedido, envio

**operaciones:** logistica, compras, inventario, mantenimiento, produccion, calidad, proveedores

**otro:** general, sin_clasificar

### Agregar un área nueva
1. Haz clic en **"+ Nueva área"**
2. Define: nombre, descripción y color
3. Presiona **"Crear"**
4. El área aparece de inmediato en los formularios de tickets y en el catálogo que consulta el clasificador

No puede haber dos áreas con el mismo nombre, aunque solo cambien las mayúsculas ("IT" e "it" son el mismo nombre).

### Agregar una categoría a un área
1. Despliega el área con la flecha ▶
2. Escribe el nombre en **"Nueva categoría..."** (recomendado: minúsculas y guion bajo si es compuesto)
3. Presiona **"+ Agregar"** o Enter

Dentro de un área tampoco puede haber dos categorías con el mismo nombre ignorando mayúsculas.

### Editar un área o una categoría
- **Área:** haz clic en ✏️ para cambiar nombre, descripción o color y presiona **"Guardar"**. El nuevo nombre se ve en todos los tickets que ya la tienen
- **Categoría:** haz clic en ✏️ dentro del chip, escribe el nuevo nombre y presiona Enter (Escape cancela)
- El color del área se muestra como un punto en las tarjetas del tablero, en la lista y en el detalle del ticket

### Desactivar un área o una categoría
- Usa el interruptor del área o el punto de color de la categoría. El sistema pide confirmación antes de desactivar
- Un área o categoría inactiva deja de ofrecerse al crear, revisar o corregir tickets, y los correos que el clasificador le asigne llegan como borradores a la Bandeja de revisión
- Los tickets existentes conservan su área y categoría
- Nada se puede eliminar: solo desactivar y volver a activar

### Exportar la revisión del clasificador
**"⬇ Exportar revisión del clasificador"** descarga un CSV con cada ticket que llegó por correo en el período elegido: el área y la categoría que propuso el clasificador, su confianza y el área y la categoría finales que dejó una persona. Sirve para medir cuánto acierta el clasificador y ajustarlo.

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
5. **Áreas y categorías:** antes de agregar una nueva, verifica que no existe una similar con otro nombre
6. **Múltiples Admins:** es recomendable tener al menos dos Admins para evitar puntos únicos de falla en la administración
