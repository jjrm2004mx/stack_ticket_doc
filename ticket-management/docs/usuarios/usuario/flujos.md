---
rol: usuario
tipo: flujos
audiencia: usuario_final
fuente: ticket-management
---

# Flujos principales — Usuario

Este documento describe paso a paso todo lo que puedes hacer como Usuario en el sistema de tickets.

---

## 1. Crear un ticket nuevo

Un ticket es una solicitud formal al equipo de soporte. Puedes crear uno cuando tengas un problema, necesites ayuda o quieras reportar algo.

**Pasos:**
1. En la pantalla principal haz clic en el botón **"Nuevo Ticket"** (ícono `+`)
2. Completa el formulario:
   - **Título** *(obligatorio)*: describe el problema en una frase corta y clara. Máximo 255 caracteres. Ejemplo: "No puedo acceder al sistema desde mi computadora"
   - **Descripción** *(opcional pero recomendada)*: explica el problema con detalle. Incluye qué pasó, cuándo ocurrió, qué intentaste hacer. Máximo 10,000 caracteres
   - **Tipo**: clasifica tu solicitud
     - `Tarea` — algo que necesitas que hagan
     - `Bug` — algo que no funciona correctamente
     - `Feature` — una mejora o funcionalidad nueva
     - `Documentación` — necesitas información o documentación
   - **Prioridad**: indica la urgencia
     - `Baja` — puede esperar, no es urgente
     - `Media` — necesita atención en tiempo razonable
     - `Alta` — afecta tu trabajo, necesita atención pronto
     - `Crítica` — bloqueante total, necesita atención inmediata
   - **Fecha límite** *(opcional)*: si hay una fecha en la que necesitas resolución
3. Haz clic en **"Crear"** o **"Guardar"**
4. El ticket queda en estado **ABIERTO** y el equipo lo verá de inmediato

**Qué pasa después:** el equipo revisará el ticket, lo clasificará si es necesario, lo asignará a alguien y comenzará a trabajarlo.

**Nota sobre clasificación:** cuando se crea el ticket, el sistema puede clasificarlo automáticamente en una categoría (IT, Cliente, Operaciones, Otro). Si la clasificación no es correcta, puedes indicarlo en los comentarios.

---

## 2. Ver y seguir tus tickets

**Cómo ver la lista de tickets:**
- La pantalla principal muestra todos tus tickets con su estado actual, prioridad y fecha de creación
- Puedes filtrar por estado, tipo o prioridad usando los controles de la parte superior

**Cómo abrir el detalle de un ticket:**
- Haz clic sobre cualquier ticket de la lista para ver su detalle completo
- En el detalle verás: título, descripción, estado actual, persona asignada, prioridad, tipo, fecha de creación, comentarios y adjuntos

**Campos que puedes ver en el detalle:**
| Campo | Qué muestra |
|---|---|
| Estado | Estado actual (Abierto, En Progreso, En Revisión, Cerrado) |
| Asignado a | Nombre de quien trabaja tu ticket |
| Prioridad | Baja / Media / Alta / Crítica |
| Tipo | Tarea / Bug / Feature / Documentación |
| Creado por | Tu nombre |
| Fecha de creación | Cuándo creaste el ticket |
| Fecha límite | La fecha que indicaste (si la pusiste) |
| Clasificación | Dominio y categoría asignados (IT, Cliente, etc.) |

---

## 3. Agregar comentarios a un ticket

Los comentarios son la forma de comunicarte con el equipo sobre un ticket específico.

**Cuándo usar comentarios:**
- Para dar información adicional que olvidaste incluir al crear el ticket
- Para responder preguntas que el equipo te haga
- Para indicar si el problema sigue ocurriendo o si ya se resolvió
- Para dar seguimiento si el ticket lleva mucho tiempo sin avanzar

**Cómo agregar un comentario:**
1. Abre el detalle del ticket
2. Ve a la sección de comentarios en la parte inferior
3. Escribe tu mensaje en el campo de texto
4. Haz clic en **"Enviar"** o **"Comentar"**

**Importante:** los comentarios son visibles para todo el equipo que tiene acceso al ticket. No incluyas información sensible como contraseñas.

---

## 4. Ver adjuntos de un ticket

Los adjuntos son archivos (imágenes, documentos, capturas de pantalla) asociados al ticket.

**Cómo ver adjuntos:**
1. Abre el detalle del ticket
2. Busca la sección **"Adjuntos"** o el ícono de clip/archivo
3. Verás la lista de archivos adjuntos con su nombre y tamaño
4. Haz clic en cualquier adjunto para descargarlo o visualizarlo

**Qué tipos de archivos puedes ver:**
- Imágenes (PNG, JPG, GIF)
- Documentos (PDF, Word, Excel)
- Otros archivos adjuntados por el equipo

**Como Usuario:** puedes ver y descargar adjuntos existentes, pero NO puedes subir ni eliminar adjuntos. Si necesitas enviar un archivo al equipo, hazlo por comentario indicando que tienes un archivo y el equipo te indicará cómo enviarlo.

---

## 5. Consultar el historial del ticket

El historial muestra todos los cambios que tuvo el ticket desde que fue creado.

**Qué muestra el historial:**
- Cambios de estado (quién lo cambió y cuándo)
- Cambios de asignación
- Modificaciones de prioridad
- Cualquier otra actualización relevante

**Cómo acceder:** en el detalle del ticket busca la pestaña o sección **"Eventos"** o **"Historial"**.

**Nota:** como Usuario puedes ver el historial pero no tienes acceso a todas las acciones de auditoría detalladas — eso es solo para Managers y Admins.

---

## Resumen: qué puedes y no puedes hacer

| Acción | ¿Puedes? |
|---|---|
| Ver todos tus tickets | ✅ |
| Crear un ticket nuevo | ✅ |
| Agregar comentarios | ✅ |
| Ver adjuntos del ticket | ✅ |
| Descargar adjuntos | ✅ |
| Editar el título o descripción de un ticket | ❌ |
| Cambiar el estado del ticket | ❌ |
| Subir archivos adjuntos | ❌ |
| Eliminar adjuntos | ❌ |
| Ver el dashboard de métricas | ❌ |
| Gestionar usuarios | ❌ |
| Archivar tickets | ❌ |

---

## Preguntas frecuentes

**¿Puedo editar mi ticket después de crearlo?**
No. Una vez creado, solo el equipo (Manager o Admin) puede editar los campos del ticket. Si cometiste un error al crearlo, agrega un comentario con la corrección.

**¿Cómo sé que mi ticket fue recibido?**
El ticket aparecerá en tu lista inmediatamente con estado ABIERTO. Eso confirma que fue registrado correctamente.

**¿Puedo crear un ticket para otra persona?**
El ticket quedará registrado a tu nombre. Si es para otra persona, indícalo en la descripción.

**¿Qué hago si mi ticket urgente no ha sido atendido?**
Agrega un comentario en el ticket explicando la urgencia. También puedes contactar directamente a tu Manager.

**¿Se me notifica cuando cambia el estado de mi ticket?**
Depende de la configuración del sistema. Algunos estados envían notificación automática al solicitante. Consulta con tu equipo si no recibes notificaciones.
