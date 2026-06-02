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

**Transiciones válidas para Manager:**
- ABIERTO → EN PROGRESO
- EN PROGRESO → EN REVISIÓN
- EN PROGRESO → ABIERTO (regresión)
- EN REVISIÓN → CERRADO ← única forma de cerrar un ticket como Manager
- EN REVISIÓN → EN PROGRESO (si hay problemas)

**Qué registra el sistema:** cada cambio de estado queda en el historial con tu nombre y la fecha/hora exacta.

---

## 2. Asignar un ticket a un miembro del equipo

**Cuándo hacerlo:** cuando el ticket está ABIERTO y necesitas designar quién lo resolverá.

**Pasos:**
1. Abre el detalle del ticket
2. En el campo **"Asignado a"** selecciona al usuario responsable
3. Guarda el cambio

**Notas:**
- Puedes reasignar un ticket en cualquier estado excepto CERRADO
- Al asignar, el campo `assignedTo` del ticket se actualiza con el UUID del usuario seleccionado
- La asignación queda registrada en el historial de eventos

---

## 3. Editar un ticket

Como Manager puedes modificar los campos de un ticket existente.

**Campos que puedes editar:**
- Título (máximo 255 caracteres)
- Descripción (máximo 10,000 caracteres)
- Prioridad (Baja / Media / Alta / Crítica)
- Tipo (Tarea / Bug / Feature / Documentación)
- Fecha límite
- Horas estimadas
- Clasificación y categoría
- Asignado a

**Pasos:**
1. Abre el detalle del ticket
2. Haz clic en **"Editar"** o directamente sobre el campo que quieres modificar
3. Realiza los cambios
4. Guarda

**Restricción:** no puedes editar tickets en estado CERRADO.

---

## 4. Gestionar adjuntos

Como Manager tienes control total sobre los adjuntos de un ticket.

### Ver y descargar adjuntos
1. Abre el detalle del ticket
2. Ve a la sección **"Adjuntos"** o usa el botón de clip
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
- Modificaciones de campos (prioridad, título, etc.)
- Comentarios agregados
- Adjuntos subidos o eliminados

**Cómo acceder:**
1. Abre el detalle del ticket
2. Ve a la pestaña **"Eventos"** o **"Historial"**

**Campos que muestra cada evento:**
- Tipo de evento
- Usuario que lo realizó
- Fecha y hora exacta
- Valor anterior y valor nuevo (cuando aplica)

---

## 7. Revisar y aprobar borradores

Los borradores son tickets que llegaron al sistema (normalmente desde correo electrónico) y necesitan revisión antes de activarse.

**Cómo acceder a los borradores:**
1. En el menú principal ve a **"Revisión de Borradores"**
2. Verás la lista de todos los tickets pendientes de revisión

**Qué puedes hacer con un borrador:**
- Ver el contenido original (incluyendo el email HTML si llegó por correo)
- Editar: prioridad, clasificación, categoría y asignado a
- **Aprobar** el borrador → se convierte en ticket ABIERTO y entra al flujo normal

**Cuándo existe un borrador:** cuando el sistema de ingesta automática (ticket-ingestion-light/n8n) procesa un correo entrante y lo crea como borrador para revisión humana.

**Pasos para aprobar:**
1. Abre el borrador
2. Revisa y ajusta clasificación, categoría, prioridad y asignación
3. Haz clic en **"Aprobar"**
4. El ticket pasa a estado ABIERTO y es visible en el flujo normal

**Nota:** no puedes eliminar borradores — esa acción es exclusiva de Admin.

---

## 8. Ver el dashboard de métricas

El dashboard muestra estadísticas del estado del equipo.

**Cómo acceder:** en el menú principal busca **"Dashboard"** o el ícono de métricas.

**Qué puedes ver:**
- Total de tickets por estado (Abiertos, En Progreso, En Revisión, Cerrados)
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
| Cambiar estado (excepto ABIERTO→CERRADO) | ✅ |
| Cerrar tickets (desde EN REVISIÓN) | ✅ |
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
| Gestionar clasificaciones | ❌ — solo Admin |
| Configurar workflow | ❌ — solo Admin |
| Cerrar ticket desde ABIERTO directamente | ❌ — solo Admin |

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

**¿Cuándo debo usar "EN REVISIÓN" vs cerrar directamente?**
Usa EN REVISIÓN cuando quieras que alguien (el solicitante u otro Manager) confirme que el problema fue resuelto antes del cierre formal. Si estás seguro de que está resuelto, puedes cerrar directamente desde EN REVISIÓN.
