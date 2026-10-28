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
1. En la barra de la vista haz clic en **"+ Nuevo ticket"**. Se abre un panel a la derecha que indica en qué estado empezará ("Se crea como …")
2. Completa el formulario:
   - **Título** *(obligatorio)*: describe el problema en una frase corta y clara. Máximo 255 caracteres. Ejemplo: "No puedo acceder al sistema desde mi computadora"
   - **Descripción** *(opcional pero recomendada)*: explica el problema con detalle. Incluye qué pasó, cuándo ocurrió, qué intentaste hacer. Máximo 10,000 caracteres
   - **Solicitante** (recuadro de la derecha): la persona a nombre de quien se crea el ticket (por ejemplo, un cliente que llamó): nombre, correo y **Medio de contacto** (Teléfono, Presencial, Correo, Chat u Otro). Si escribes un correo válido se marca sola la casilla **"Puede recibir avisos por correo"**; puedes desmarcarla. Quien crea el ticket no recibe avisos por crearlo
   - **Prioridad** (cuatro botones; Media por defecto): indica la urgencia
     - `Baja` — puede esperar, no es urgente
     - `Media` — necesita atención en tiempo razonable
     - `Alta` — afecta tu trabajo, necesita atención pronto
     - `Crítica` — bloqueante total, necesita atención inmediata
   - **Área · Categoría** *(opcional)*: elige el área y, si aplica, su categoría en un solo selector
   - **Fecha límite** *(opcional)*: si hay una fecha en la que necesitas resolución, con los atajos "Hoy", "Mañana" y "+3 días"; no puede ser anterior a hoy. No aparece si tu organización no usa fecha límite
   - **Adjuntos** *(si tu rol puede subirlos)*: arrastra, pega (Ctrl+V) o elige archivos; los que no se pueden subir se marcan en rojo
3. Haz clic en **"Crear ticket"** (o Ctrl+Enter). Si falta el título, el aviso aparece junto al campo. Con **"Crear otro al terminar"** el formulario queda vacío para el siguiente
4. El ticket queda en el **estado inicial** de tu organización (de fábrica, "Abierto"), aparece el aviso "Ticket #… creado" y se abre su detalle

**Qué pasa después:** el equipo revisará el ticket, lo clasificará si es necesario, lo asignará a alguien y comenzará a trabajarlo.

**Nota sobre el área:** los tickets que llegan por correo los clasifica automáticamente el sistema en un área (IT, Cliente, Operaciones, Otro) y una categoría. Si no es correcta, indícalo en los comentarios; quien tenga el permiso "Editar tickets" puede corregirla desde el detalle.

---

## 2. Ver y seguir tus tickets

**Cómo ver la lista de tickets:**
- La pantalla principal muestra todos tus tickets con su estado actual, prioridad y fecha de creación
- Puedes filtrar por estado, prioridad o asignado, y buscar por título, solicitante o folio (escribe `#` al inicio) en la única caja de búsqueda

**Cómo abrir el detalle de un ticket:**
- Haz doble clic sobre cualquier ticket de la lista o del tablero para ver su detalle completo
- En el detalle verás: título, descripción, estado actual, persona asignada, prioridad, solicitante, fecha límite, fecha de creación, comentarios y adjuntos

**Campos que puedes ver en el detalle:**
| Campo | Qué muestra |
|---|---|
| Estado | Estado actual, con el nombre que usa tu organización |
| Asignado a | Nombre de quien trabaja tu ticket |
| Prioridad | Baja / Media / Alta / Crítica |
| Creado por | Tu nombre |
| Fecha de creación | Cuándo creaste el ticket |
| Solicitante | A nombre de quién se creó, si recibe avisos y el medio |
| Fecha límite | La fecha que indicaste (si la pusiste) |
| Área / Categoría | Área y categoría asignadas (IT, Cliente, etc.) |

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
El ticket aparecerá en tu lista inmediatamente en el estado inicial de tu organización. Eso confirma que fue registrado correctamente.

**¿Puedo crear un ticket para otra persona?**
El ticket quedará registrado a tu nombre. Si es para otra persona, indícalo en la descripción.

**¿Qué hago si mi ticket urgente no ha sido atendido?**
Agrega un comentario en el ticket explicando la urgencia. También puedes contactar directamente a tu Manager.

**¿Se me notifica cuando cambia el estado de mi ticket?**
Depende de la configuración del sistema. Algunos estados envían notificación automática al solicitante. Consulta con tu equipo si no recibes notificaciones.

**¿Cómo cambio mi contraseña?**
Haz clic en tu nombre o tus iniciales, arriba a la derecha, y elige **"Cambiar contraseña"**. Escribe la actual y la nueva dos veces (mínimo 12 caracteres; puedes usar una frase). Tu sesión sigue abierta y se cierran las de tus otros dispositivos.

**Olvidé mi contraseña.**
En la página de inicio de sesión haz clic en **"¿Olvidaste tu contraseña?"**, escribe tu correo y abre el enlace que te llega (válido 60 minutos, de un solo uso). Si no llega, revisa tu carpeta de spam o pide a tu administrador que te envíe el enlace.

**Mi cuenta dice "Credenciales inválidas" aunque la contraseña es correcta.**
Después de 5 intentos fallidos en 15 minutos la cuenta se bloquea 15 minutos. Espera o restablece tu contraseña con "¿Olvidaste tu contraseña?".

**¿Por qué me pide iniciar sesión de nuevo?**
Si cambiaste o restableciste tu contraseña en otro dispositivo, las demás sesiones terminan y verás "Tu sesión terminó. Inicia sesión de nuevo.".
