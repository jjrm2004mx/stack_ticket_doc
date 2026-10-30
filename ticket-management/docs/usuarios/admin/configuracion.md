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
1. Haz clic en **"+ Nuevo Usuario"**
2. Ingresa: nombre completo, email y rol
3. Haz clic en **"Crear y enviar invitación"**
4. El usuario queda **"✉ Invitado"** y recibe un correo para definir su propia contraseña (válido 72 horas); podrá iniciar sesión cuando la defina. Si no le llegó, usa **"Reenviar invitación"**

### Restablecer la contraseña de un usuario
Nadie más que el usuario conoce su contraseña. Si no puede entrar, haz clic en **"Enviar enlace para restablecer contraseña"** en su fila: le llega a su correo un enlace de un solo uso, válido 60 minutos. También puede pedirlo él mismo con **"¿Olvidaste tu contraseña?"** en el inicio de sesión.

### Cambiar el rol de un usuario
1. Busca el usuario en la lista
2. Edita su perfil
3. Cambia el rol
4. Guarda — el nuevo rol aplica desde la próxima sesión del usuario

**Implicaciones del cambio de rol:**
- USER → MANAGER: obtiene EDIT_TICKET, CHANGE_STATUS, UPLOAD_ATTACHMENT, DELETE_ATTACHMENT, VIEW_TICKET_EVENTS, VIEW_DASHBOARD, REVIEW_DRAFTS
- MANAGER → ADMIN: obtiene ARCHIVE_TICKET, MANAGE_USERS, MANAGE_CLASSIFICATIONS y MANAGE_WORKFLOW
- ADMIN → MANAGER: pierde los 4 permisos exclusivos de Admin (las transiciones no dependen del rol)

### Desactivar un usuario
Cuando un usuario deja de tener acceso (por ejemplo, un empleado que se va de la empresa):
1. Busca el usuario en la lista
2. Haz clic en **"Desactivar"** y confirma
3. Ya no podrá iniciar sesión y sus sesiones abiertas terminan en menos de un minuto

**Antes de desactivar:** reasigna manualmente los tickets que tiene asignados. El sistema no los reasigna automáticamente.

---

## 2. Gestión de áreas y categorías

**Pantalla:** Áreas y categorías, en el menú de la izquierda (`screen: classificationManagement`)
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
- **Área:** haz clic en el lápiz para cambiar nombre, descripción o color y presiona **"Guardar"**. El nuevo nombre se ve en todos los tickets que ya la tienen
- **Categoría:** haz clic en el lápiz dentro del chip, escribe el nuevo nombre y presiona Enter (Escape cancela)
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

**Pantalla:** "⚙️ Gestión de Workflow" ("Workflow" en el menú de la izquierda; tiene ayuda "?")
**Permiso requerido:** MANAGE_WORKFLOW

El workflow define por qué estados pasa un ticket y a quién se le informa. Se aplica a todos los tickets de la organización, también a los que llegan por correo.

### Estados

Cada estado tiene un nombre que eliges y una categoría fija que entiende el sistema: Nuevo, En curso, En espera o Cerrado (Resuelto o Cancelado). Las métricas, los filtros y el cierre siguen la categoría (ver "Estados y transiciones — Admin").

1. **"+ Nuevo estado":** nombre, categoría, resultado (solo Cerrado), descripción y color.
2. **"Editar":** nombre, descripción y color; **↑/↓** cambia el orden de las columnas del tablero.
3. **"Hacer inicial":** elige el estado de categoría Nuevo donde entran los tickets nuevos.
4. **"Desactivar":** mueve los tickets del estado al que elijas (sin correo a los solicitantes) y lo desactiva; **"Activar"** lo vuelve a usar.

### Transiciones

Cada interruptor activa o desactiva un movimiento de un estado a otro (desde el tablero o el detalle) y se aplica de inmediato para todos. Una transición que sale de un estado Cerrado permite reabrir. La pantalla rechaza los cambios que dejan un estado sin camino a un estado Cerrado y muestra la regla que se rompería.

La pantalla no muestra ni edita un rol requerido por transición: cualquier rol con el permiso "Cambiar estado" puede ejecutar las transiciones activas.

Transiciones de fábrica: Abierto → En progreso, Abierto → Cerrado, En progreso → En revisión, En revisión → En progreso y En revisión → Cerrado.

### Notificaciones

La pantalla **Notificaciones** es la lista completa de lo que puede recibir el solicitante por correo; lo que no esté encendido no se envía:
- **Avisos por tipo de evento:** "Creación", "Asignación" y "Comentario".
- **Cambio de estado: avisar al entrar a…:** un interruptor por estado; se avisa cuando el ticket entra a ese estado.

Siempre hace falta que el ticket tenga activados los avisos al solicitante y que este tenga correo. Nunca se notifican las correcciones (solicitante, fecha límite, área/categoría), los adjuntos, los movimientos al desactivar un estado ni lo que el solicitante agrega al responder el correo. Una organización nueva solo avisa la **creación** y la **entrada a un estado Cerrado**.

**Precaución:** apagar "Creación" silencia los correos de creación, también los de tickets que llegan por correo.

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

El Admin revisa y aprueba borradores igual que el Manager, en la **Bandeja de revisión** (menú **"Borradores"**; ver `manager/flujos.md`, sección 7): con "Correo original" y los adjuntos (solo de consulta) a la vista, decide prioridad, área y categoría y, si aplica, a quién asignarlo, y presiona **"Aprobar"**.

Los borradores **no se pueden rechazar ni eliminar** desde el portal, tampoco el Admin. Un borrador que no debe entrar al flujo (spam o duplicado) se aprueba y después se archiva desde su detalle (sección 6).

---

## 6. Archivar tickets

**Permiso requerido:** ARCHIVE_TICKET (exclusivo Admin)

**Pasos:**
1. Abre el detalle del ticket a archivar
2. En la barra de acciones haz clic en **"⋯"** y elige **"Archivar ticket"**
3. Confirma con **"Archivar"** en el mismo menú

**Resultado:** el ticket deja de aparecer en el tablero, la lista y las búsquedas, pero permanece en la base de datos con su estado y su historial.

**Para ver un ticket archivado:** ábrelo con su enlace (`/tickets/<id>`); el detalle muestra **"Ticket archivado"** y el botón **"Desarchivar"**, que lo devuelve a las vistas en el estado que tenía. No hay un filtro de archivados en la lista.

---

## 7. Canales de entrada (📨 Canales)

Los **canales** son los buzones de correo de tu organización desde los que se crean tickets automáticamente. Los da de alta tu proveedor; tú no ves sus llaves ni puedes crearlos o borrarlos.

En **📨 Canales** ves, por buzón: su estado (Activo, Pausado o Deshabilitado), el último mensaje recibido, cuántos están en proceso, los errores y la hora del último reporte del clasificador (en rojo si lleva más de 15 minutos sin reportar).

- **Pausar:** los correos nuevos de ese buzón no se convierten en tickets y quedan sin leer hasta que lo reanudes.
- **Reanudar:** vuelve a crear tickets con los correos pendientes.

Los tickets que llegan de un canal muestran en su detalle **Origen: ✉ canal (buzón)**, y el remitente queda como solicitante con medio "Correo".

---

## Buenas prácticas para Admin

1. **Cambios de workflow:** siempre comunica al equipo antes de modificar transiciones o estados
2. **Cambios de permisos:** documenta por qué cambiaste un permiso — usa los comentarios del sistema si están disponibles
3. **Desactivación de usuarios:** reasigna tickets antes de desactivar
4. **Borradores dudosos:** revisa "Correo original" y los adjuntos antes de aprobar; si no deben entrar al flujo, apruébalos y archívalos
5. **Áreas y categorías:** antes de agregar una nueva, verifica que no existe una similar con otro nombre
6. **Múltiples Admins:** es recomendable tener al menos dos Admins para evitar puntos únicos de falla en la administración
