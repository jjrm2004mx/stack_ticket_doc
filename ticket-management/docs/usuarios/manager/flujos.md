---
rol: manager
tipo: flujos
audiencia: usuario_final
fuente: ticket-management
---

# Flujos principales — Manager

Como Manager tienes acceso completo al ciclo de vida de los tickets: puedes editarlos, asignarlos, cambiar su estado, gestionar adjuntos, revisar borradores y ver métricas del equipo.

---

## 1. Gestionar el estado de un ticket

Cambiar el estado es la acción más frecuente de un Manager. Controla el progreso del ticket a través del flujo.

**Pasos para cambiar el estado:**
1. Abre el detalle del ticket
2. Busca el selector de estado (normalmente en la parte superior del detalle)
3. Selecciona el nuevo estado válido
4. Confirma el cambio

**Transiciones válidas:** las que configuró tu organización en el workflow; el menú "Estado" del detalle ofrece cada estado al que se puede pasar desde el estado actual. Con el workflow de fábrica: Abierto → En progreso, Abierto → Cerrado, En progreso → En revisión, En revisión → En progreso y En revisión → Cerrado. Un ticket cerrado se reabre solo si la organización habilitó una transición que salga del estado Cerrado.

**Qué registra el sistema:** cada cambio de estado queda en el historial con tu nombre y la fecha/hora exacta.

---

## 2. Asignar un ticket a un miembro del equipo

**Cuándo hacerlo:** cuando el ticket está ABIERTO y necesitas designar quién lo resolverá.

**Pasos:**
1. Abre el detalle del ticket
2. En el campo **"Asignado a"** selecciona al usuario responsable
3. Guarda el cambio

**Notas:**
- Puedes reasignar un ticket en cualquier estado, también cerrado
- Al asignar, el campo `assignedTo` del ticket se actualiza con el UUID del usuario seleccionado
- La asignación queda registrada en el historial de eventos
- Si el Admin configuró reglas de asignación por área y categoría, los correos que entran activos ya llegan asignados (en el historial: "⚙ Asignado automáticamente"); puedes reasignarlos igual. Al aprobar un borrador o crear un ticket, "Asignar a" viene propuesto con la persona de la regla

---

## 3. Editar un ticket

Como Manager puedes modificar los campos de un ticket existente.

**Campos que puedes editar:**
- Título (máximo 255 caracteres)
- Descripción (máximo 10,000 caracteres)
- Prioridad (Baja / Media / Alta / Crítica)
- Fecha límite (o quitarla)
- Solicitante: nombre, correo, casilla "Enviar notificaciones al solicitante" y medio
- Horas estimadas
- Área y categoría
- Asignado a

**Pasos:**
1. Abre el detalle del ticket
2. Haz clic en **"Editar"** o directamente sobre el campo que quieres modificar
3. Realiza los cambios
4. Guarda

**Restricción:** no puedes editar tickets en un estado de categoría Cerrado.

---

## 4. Gestionar adjuntos

Como Manager tienes control total sobre los adjuntos de un ticket.

### Ver y descargar adjuntos
1. Abre el detalle del ticket
2. Usa **"Adjuntos"** en la barra de acciones del detalle (o "Ver todos" de su sección de adjuntos)
3. Verás la lista de archivos con nombre, tamaño y quién los subió
4. Haz clic en un archivo para descargarlo o visualizarlo

### Subir un adjunto nuevo
1. Abre la sección de adjuntos del ticket
2. Haz clic en **"Subir archivo"** o arrastra el archivo
3. Selecciona el archivo desde tu computadora
4. Espera a que se complete la carga

**Restricciones al subir:**
- Tamaño máximo por archivo: **10 MB**
- Si el archivo supera 10 MB, la carga será rechazada
- Tipos de archivo: el sistema acepta los formatos estándar de documentos e imágenes

### Eliminar un adjunto
1. En la lista de adjuntos, busca el que quieres eliminar
2. Haz clic en el ícono de eliminar (papelera)
3. Confirma la acción

**Advertencia importante:** la eliminación de adjuntos es **irreversible**. Una vez eliminado, el archivo no puede recuperarse. Asegúrate de que el adjunto ya no es necesario antes de eliminarlo.

---

## 5. Agregar y ver comentarios

Los comentarios son el canal de comunicación dentro de un ticket.

**Agregar un comentario:**
1. Abre el detalle del ticket
2. Ve a la sección de comentarios
3. Escribe tu mensaje
4. Haz clic en **"Enviar"**

**Como Manager puedes:** ver todos los comentarios de todos los tickets, agregar comentarios en cualquier ticket (no solo los asignados a ti).

---

## 6. Ver el historial de eventos

El historial muestra un registro completo y auditable de todo lo que ocurrió con el ticket.

**Qué incluye el historial:**
- Cambios de estado (de quién a quién, fecha y hora)
- Cambios de asignación
- Modificaciones de campos (prioridad, fecha límite, solicitante, área y categoría)
- Comentarios agregados
- Adjuntos subidos o eliminados

**Cómo acceder:**
1. Abre el detalle del ticket
2. Haz clic en **"Historial"** en la barra de acciones (arriba de la columna izquierda)

**Campos que muestra cada evento:**
- Tipo de evento
- Usuario que lo realizó
- Fecha y hora exacta
- Valor anterior y valor nuevo (cuando aplica)

---

## 7. Revisar y aprobar borradores

Los borradores son tickets que llegaron al sistema (normalmente desde correo electrónico) y necesitan revisión antes de activarse.

**Cómo acceder a los borradores:**
1. En el menú de la izquierda haz clic en **"Borradores"** (su contador dice cuántos hay; solo aparece si tu rol tiene "Revisar borradores")
2. Se abre la **Bandeja de revisión**. Junto al título dice **"Se aprueban como ● <estado inicial>"**: el estado en que entrarán al aprobarlos

**Qué ves en cada borrador:**
- El chip **BORRADOR**, el solicitante y su correo, el título, la descripción y la fecha de creación
- Arriba a la derecha, **"Correo original"** (si llegó por correo: abre el mensaje tal como llegó) y **"Adjuntos"** con su número (si trae archivos). Los adjuntos son **solo de consulta**: aquí no se suben ni se eliminan
- Un aviso en rojo si el clasificador pidió un área o una categoría que no existe o está inactiva

**Qué decides**, con los mismos campos y en el mismo orden que **"Nuevo ticket"**:
- **Prioridad**: en la lista, con su punto de color
- **Área · Categoría**: un solo selector; si falta lo que pidió el clasificador, se marca en rojo con "Solicitada: …" debajo
- **Asignar a** (si tu rol puede asignar): si el área o la categoría tienen regla de asignación, viene propuesta la persona que le toca (en verde); si eliges a otra, **"Usar la sugerencia"** vuelve a ella

**Cuándo existe un borrador:** cuando el sistema de ingesta automática (ticket-ingestion-light/n8n) procesa un correo entrante y lo crea como borrador para revisión humana.

**Pasos para aprobar:**
1. Revisa el correo original y los adjuntos si hace falta
2. Ajusta prioridad, área y categoría y, si aplica, la asignación
3. Haz clic en **"Aprobar"**
4. El ticket pasa al estado inicial y aparece en el tablero y la lista (al refrescar)

**Nota:** los borradores solo se aprueban; no se pueden rechazar ni eliminar desde el portal (tampoco el Admin).

**Si el borrador es basura** (spam, publicidad, un duplicado o algo que no debe entrar al flujo): no lo apruebes tú. Avisa a un **Admin**: él lo aprueba y enseguida lo archiva desde su detalle, así no queda en el tablero ni en la lista. Como Manager no tienes el permiso para archivar tickets, así que si lo apruebas quedaría activo hasta que un Admin lo archive. Mientras tanto, el borrador se queda en la bandeja sin afectar nada.

---

## 8. Ver el dashboard de métricas

El dashboard muestra estadísticas del estado del equipo.

**Cómo acceder:** en el menú principal busca **"Dashboard"** o el ícono de métricas.

**Qué puedes ver:**
- Tickets abiertos, cerrados (y cuántos se cancelaron), en curso y en espera
- Tickets por prioridad
- Tickets asignados a cada miembro del equipo
- Tickets creados en un período de tiempo
- Tiempo promedio de resolución

---

## Resumen de permisos del Manager

| Acción | ¿Puedes? |
|---|---|
| Ver todos los tickets | ✅ |
| Crear tickets | ✅ |
| Editar tickets | ✅ |
| Cambiar estado (transiciones configuradas) | ✅ |
| Agregar comentarios | ✅ |
| Ver adjuntos | ✅ |
| Subir adjuntos | ✅ |
| Eliminar adjuntos | ✅ |
| Ver historial de eventos | ✅ |
| Ver dashboard de métricas | ✅ |
| Revisar y aprobar borradores | ✅ |
| Asignar tickets | ✅ |
| Archivar tickets | ❌ — solo Admin |
| Gestionar usuarios | ❌ — solo Admin |
| Gestionar áreas y categorías | ❌ — solo Admin |
| Configurar workflow | ❌ — solo Admin |

---

## Preguntas frecuentes

**¿Puedo editar un ticket que no me está asignado?**
Sí. Como Manager puedes editar cualquier ticket del sistema, no solo los asignados a ti.

**¿Puedo cambiar la prioridad de un ticket cerrado?**
No. Los tickets cerrados no son editables.

**¿Qué pasa si subo un archivo de más de 10 MB?**
El sistema rechazará la carga y mostrará un mensaje de error. Debes comprimir el archivo o dividirlo antes de intentarlo de nuevo.

**¿Puedo ver los borradores si no me los asignaron?**
Sí. Todos los Managers ven todos los borradores pendientes.

**¿Cómo cambio o recupero mi contraseña?**
Para cambiarla: clic en tu nombre o tus iniciales, arriba a la derecha → **"Cambiar contraseña"** (mínimo 12 caracteres; tus otros dispositivos cierran sesión). Si la olvidaste: **"¿Olvidaste tu contraseña?"** en el inicio de sesión, o pide a tu administrador que te envíe el enlace. Después de 5 intentos fallidos en 15 minutos la cuenta se bloquea 15 minutos.

**¿Cuándo debo usar "EN REVISIÓN" vs cerrar directamente?**
Usa EN REVISIÓN cuando quieras que alguien (el solicitante u otro Manager) confirme que el problema fue resuelto antes del cierre formal. Si estás seguro de que está resuelto, puedes cerrar directamente desde EN REVISIÓN.

---

## 8. Revisar los archivados

En el menú de la izquierda, **"Archivados"** (solo aparece si tu rol tiene "Revisar borradores") lista los tickets archivados con el motivo por el que se archivaron, por ejemplo un aviso automático de la mesa de soporte o una copia de un correo del propio equipo. Filtra por motivo con los contadores de arriba, busca por título, solicitante o detalle, y usa **"Abrir"** para ver el detalle. Restaurar un ticket es solo del Admin: si uno no debió archivarse, avísale.

---

## 9. Palabras clave (si el Admin te dio el permiso)

Si tu Admin te dio **"Gestionar palabras clave"** y la plataforma las encendió para tu organización, en el menú aparece **"Áreas y categorías"** y ahí solo verás la pestaña **"Palabras clave"**. Despliega un área con ▶; en cada categoría escribe una frase de 1 a 4 palabras en "Nueva frase..." y presiona Enter. Las frases ayudan a la IA a reconocer la categoría de un correo (el asunto pesa más que el cuerpo), pero decide la IA. Cada frase pertenece a una sola categoría; corrige con el lápiz del chip y quita con la ✕.
