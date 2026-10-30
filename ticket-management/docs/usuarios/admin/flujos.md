---
rol: admin
tipo: flujos
audiencia: usuario_final
fuente: ticket-management
---

# Flujos principales — Admin

El Admin tiene acceso completo a todas las funciones del sistema: gestión de tickets, usuarios, áreas y categorías, workflow y permisos. Este documento cubre todas las operaciones disponibles.

---

## 1. Gestión de tickets (todas las operaciones)

Como Admin heredas todas las capacidades de Manager más las exclusivas de Admin.

**Capacidades completas sobre tickets:**
- Ver todos los tickets del sistema
- Crear tickets nuevos
- Editar cualquier campo de cualquier ticket
- Cambiar estado por las transiciones configuradas en el workflow
- Asignar y reasignar tickets
- Agregar comentarios
- Ver y descargar adjuntos
- Subir adjuntos
- Eliminar adjuntos
- Ver historial completo de eventos
- Ver dashboard de métricas
- Revisar y aprobar borradores (no se pueden rechazar ni eliminar)
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
3. Haz clic en **"Crear y enviar invitación"**

El usuario aparece como **"✉ Invitado"** y recibe por correo una invitación (válida 72 horas) para definir su propia contraseña. Tú nunca la ves ni la eliges. Hasta que acepte la invitación no puede iniciar sesión.

**Nota:** el rol del usuario determina qué permisos tiene en el sistema según la matriz de `role_permissions`.

### Reenviar una invitación
Si un invitado no recibió el correo o su enlace venció, haz clic en **"Reenviar invitación"** en su fila. Le llega un enlace nuevo y el anterior deja de funcionar.

### Un usuario no puede iniciar sesión
Haz clic en **"Enviar enlace para restablecer contraseña"** en su fila y confirma. El usuario recibe en su propio correo un enlace (válido 60 minutos, de un solo uso) para definir una nueva contraseña. Al usarlo, todas sus sesiones abiertas terminan y, si su cuenta estaba bloqueada por intentos fallidos, se desbloquea. Puedes enviar hasta 10 enlaces por hora al mismo usuario.

El usuario también puede pedirlo solo con **"¿Olvidaste tu contraseña?"** en la página de inicio de sesión.

**Bloqueo por intentos:** después de 5 contraseñas incorrectas en 15 minutos la cuenta queda bloqueada 15 minutos (el inicio de sesión responde "Credenciales inválidas" aunque la contraseña sea correcta). Se desbloquea sola o al restablecer la contraseña.

### Editar un usuario existente
1. Busca el usuario en la lista
2. Haz clic en **"Editar"**
3. Puedes modificar: nombre, rol, estado (activo/inactivo)
4. **Cambiar el rol** de un usuario tiene efecto inmediato en sus permisos desde la próxima vez que inicie sesión

### Desactivar un usuario
Si un usuario ya no debe tener acceso al sistema, desactívalo con **"Desactivar"** (no lo elimines — se preserva el historial de sus acciones). Ya no podrá iniciar sesión y sus sesiones abiertas terminan en menos de un minuto.

Al presionar **"Desactivar"** se abre un diálogo que revisa qué afecta:

- **Sus tickets abiertos** (sin archivar y fuera de un estado Cerrado). Eliges qué hacer con ellos:
  - **Reasignarlos con la regla de su área y categoría** (opción por defecto): cada ticket va con quien le toque según la pestaña "👥 Asignación"; los que no tengan regla quedan sin asignar.
  - **Asignarlos a** una persona activa que elijas.
  - **Dejarlos sin asignar**, para repartirlos después.
  - **Dejarlos asignados a él**: nadie más los tendrá como suyos; solo los verá quien tenga "Ver todos los tickets".
- **Las reglas de asignación** en las que recibe tickets. Con **"Quitarlo de esas reglas"** (marcado por defecto) sale de todas; una regla que se queda sin nadie se borra: el área queda en "Nadie" y la categoría vuelve a "Igual que el área". Si no lo quitas, las reglas se lo saltan mientras esté inactivo, pero **si lo reactivas vuelve a recibir tickets**.

Los tickets cerrados o archivados conservan a la persona como responsable. Cada reasignación queda en el historial del ticket a tu nombre y, si "Asignación" está encendida en Notificaciones, el solicitante recibe el aviso con el nombre de la nueva persona. Si algo falla (por ejemplo, la persona elegida ya no está activa), no se desactiva nada.

---

## 3. Gestionar áreas y categorías (MANAGE_CLASSIFICATIONS)

Las áreas y sus categorías organizan los tickets; el clasificador automático asigna cada correo a un área y una categoría de este catálogo.

**Cómo acceder:** menú de la izquierda → **"Áreas y categorías"**

### Áreas disponibles por defecto:
- `IT` — tecnología e infraestructura (color #0066FF)
- `cliente` — atención al cliente (color #FF6600)
- `operaciones` — procesos y logística (color #009900)
- `otro` — sin clasificación específica (color #999999)

### Categorías por área:
- **IT:** hardware, software, red, acceso, correo, impresora, vpn, servidor, base_de_datos, seguridad
- **cliente:** facturacion, reclamo, consulta, devolucion, garantia, soporte, pedido, envio
- **operaciones:** logistica, compras, inventario, mantenimiento, produccion, calidad, proveedores
- **otro:** general, sin_clasificar

### Agregar un área nueva
1. Haz clic en **"+ Nueva área"**
2. Define: nombre, descripción y color
3. Presiona **"Crear"**

### Agregar una categoría a un área
1. Despliega el área con la flecha ▶
2. Escribe el nombre en **"Nueva categoría..."**
3. Presiona **"+ Agregar"** o Enter

**Impacto:** las nuevas áreas y categorías quedan disponibles de inmediato para asignar a tickets y en el catálogo que consulta el clasificador automático. Los nombres no se pueden repetir aunque cambien las mayúsculas. Nada se elimina: un área o categoría se desactiva y sus correos llegan como borradores a la Bandeja de revisión. El detalle completo está en `configuracion.md`.

### Asignación automática por área y categoría (pestaña "👥 Asignación")

Define quién recibe los tickets que entran por correo, según su área y categoría. Sin reglas, los tickets entran sin asignar, como siempre.

**Cómo acceder:** menú de la izquierda → **"Reglas de asignación"** (o "Áreas y categorías" → pestaña **"Asignación"**). La tabla muestra una fila por área ("toda el área") y debajo una por cada categoría, con quién recibe sus tickets y la modalidad.

**Modalidades:**
- **Responsable directo:** una sola persona recibe todos los tickets de esa fila. La misma persona puede ser responsable de varias áreas y categorías.
- **Menor carga:** entre varias personas, recibe el ticket quien tenga menos tickets abiertos asignados (si empatan, por nombre).

**Ejemplo:** Ana atiende todo IT, salvo lo de red, que va con Carlos.
1. Fila **IT (toda el área)** → **Editar** → "Responsable directo" → Ana → **Guardar**. Las categorías de IT muestran "↳ igual que IT · Ana".
2. Fila **red** → **Editar** → "Responsable directo" → Carlos → **Guardar**.
3. Un correo IT/vpn se asigna a Ana; uno IT/red, a Carlos.

**Reglas:**
- La regla de una categoría manda sobre la de su área. Para quitarla, en la fila de la categoría elige **"Igual que el área"**.
- Un área con **"Nadie"** (⚠) deja sus tickets sin asignar, para que el administrador los asocie.
- Una persona inactiva aparece tachada y se salta; si una categoría se queda sin nadie activo, se usa la regla de su área.
- **"Ver por persona"** muestra en qué filas recibe tickets alguien (útil antes de desactivarlo).
- Solo se asignan solos los correos que entran **activos**. Los borradores no se asignan solos: al aprobarlos en la Bandeja de revisión, "Asignar a" viene propuesto con la persona de la regla. En **"Nuevo ticket"** también se propone al elegir "Área · Categoría"; en ambos casos puedes cambiarla ("Usar la sugerencia" la vuelve a poner).
- Reclasificar un ticket activo no cambia su asignado.
- En el historial del ticket, una asignación hecha por la regla aparece como **"⚙ Asignado automáticamente"**. Si "Asignación" está encendida en Notificaciones, el solicitante recibe el aviso con el nombre de la persona asignada.

### Palabras clave (pestaña "Palabras clave")

Frases que ayudan a la IA a reconocer a qué categoría pertenece un correo. La pestaña solo aparece si la plataforma encendió las palabras clave para tu organización; apagadas, todo funciona como siempre y las frases se conservan.

**Cómo acceder:** "Áreas y categorías" → pestaña **"Palabras clave"**. Despliega un área con ▶: cada categoría activa muestra sus frases como chips.

**Cómo funcionan:**
- El clasificador busca las frases en el asunto y en el cuerpo de cada correo y le dice a la IA a qué área y categoría apuntan. El asunto pesa más que el cuerpo.
- Deciden la IA y el contenido del correo: si el correo no pide lo que sugiere la frase, se ignora. Un correo dudoso sigue llegando como borrador.
- Una frase tiene de 1 a 4 palabras. No importan mayúsculas ni acentos y solo cuenta la frase completa ("nomina" no coincide con "nominal").
- Cada frase pertenece a una sola categoría; si ya está en otra, el aviso dice dónde.

**Ejemplo:** en la categoría **Viáticos** escribe "reembolso de viaticos" en "Nueva frase..." y presiona Enter. Un correo con asunto "RE: Reembolso de Viáticos octubre" llega a la IA con la pista "Viáticos".

**Consejos:** prefiere frases que distingan una categoría de otra ("factura es correcta", "reembolso de viaticos") sobre palabras generales ("factura"). Corrige con el lápiz del chip y quita con la ✕.

**Quién:** tú siempre; al Manager se le da con el permiso **"Gestionar palabras clave"** en Permisos (al Usuario nunca).

---

## 4. Configurar el workflow (MANAGE_WORKFLOW)

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

### Gestión por correo

Según las respuestas del correo, la IA sugiere (o hace, si la plataforma lo encendió) el cambio de estado del
ticket. En la sección **"Gestión por correo"** eliges qué estado de tu flujo usa en cada caso:

| Caso | Estado que debe ser | Ejemplo |
|---|---|---|
| El equipo atiende | En curso | En progreso |
| En espera del solicitante | En espera | En espera del solicitante |
| En espera de un tercero | En espera | En espera de tercero |
| Resuelto, falta confirmación | En curso o En espera | Resuelto, por confirmar |
| Cerrado al confirmar | Cerrado · Resuelto | Cerrado |

Solo aparecen los estados que encajan y cada estado sirve para un caso. **"— Ninguno —"** apaga ese caso.
Si no tienes el estado (por ejemplo, una espera), créalo arriba con su categoría y **habilita sus
transiciones**: la IA solo mueve por transiciones permitidas. Las esperas pausan la fecha límite.

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

`MANAGE_KEYWORDS` ("Gestionar palabras clave") se le puede dar al **Manager**, nunca al Usuario. Solo aparece en la matriz si la plataforma encendió las palabras clave para tu organización.

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
2. En la barra de acciones haz clic en **"⋯"** y elige **"Archivar ticket"**
3. Confirma con **"Archivar"** en el mismo menú

**Efecto:** el ticket deja de aparecer en el tablero, la lista y las búsquedas, pero sigue en la base de datos con su estado y su historial. Se abre con su enlace y su detalle ofrece **"Desarchivar"**.

**Importante:** archivar no cierra el ticket. Si quieres archivar y cerrar, primero ciérralo y luego archívalo.

### Pantalla "Archivados"

En el menú de la izquierda, **"Archivados"** (junto a Borradores) lista los tickets archivados con el **motivo** por el que se archivaron: Archivado a mano, Aviso automático, Copia del equipo, No requiere acción, Para conocimiento, Duplicado o Prueba. Lo que archivas desde el detalle queda como **"Archivado a mano"**; los demás motivos los pone un proceso automático (por ejemplo, el recálculo de tickets), y en la columna **"Por"** aparece **"Regla"**.

- Arriba, cada motivo muestra cuántos tickets tiene; haz clic para ver solo esos. El buscador y el filtro **"Regla o persona"** acotan la lista.
- **"Abrir"** muestra el detalle; el aviso **"Ticket archivado"** dice su motivo.
- **"Restaurar"** (solo Admin) lo devuelve a las vistas en el estado que tenía y borra su motivo.

---

## 7. Borradores que no deben entrar al flujo

Los borradores **no se pueden rechazar ni eliminar** desde el portal (tampoco el Admin): la Bandeja de revisión solo los aprueba. Si un borrador es correo basura o un duplicado, apruébalo y después archívalo desde su detalle (sección 6). Así queda el registro de que llegó y de quién lo descartó.

---

## Resumen de permisos exclusivos de Admin

| Acción exclusiva de Admin | Descripción |
|---|---|
| ARCHIVE_TICKET | Archivar cualquier ticket |
| MANAGE_USERS | Crear, editar y desactivar usuarios |
| MANAGE_CLASSIFICATIONS | Gestionar áreas y categorías y exportar la revisión del clasificador |
| MANAGE_WORKFLOW | Configurar estados y transiciones del sistema |
| Actualizar matriz de permisos | Cambiar qué puede hacer cada rol |
