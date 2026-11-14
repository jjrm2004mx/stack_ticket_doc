---
rol: admin
tipo: flujos
audiencia: usuario_final
fuente: ticket-management
---

# Preguntas sobre los datos — Admin

«Preguntas sobre los datos» deja preguntar en lenguaje natural por los tickets de tu empresa ("¿cuántos tickets abiertos tiene Nómina?") y recibir la respuesta en texto con la tabla de datos que la respalda. Es una función que se contrata: la enciende el operador de la plataforma para tu organización. Mientras está apagada nadie la ve, ni siquiera el Admin.

---

## Dónde está

1. En el menú de la izquierda, la tarjeta **«¿Necesitas ayuda?»** › **«Abrir asistente»**.
2. Arriba del panel, elige **«Mis datos»**. La otra opción, **«Ayuda del portal»**, es el asistente que explica cómo usar el portal.

Si solo tienes uno de los dos modos, el panel abre directo en ese y no muestra el selector. El panel recuerda en este navegador el último modo que usaste. Cada modo tiene su propia conversación; se pierde al recargar la página o al cerrar sesión.

El panel nunca adivina de qué trata tu pregunta: si preguntas en «Mis datos» cómo se usa algo del portal, te pide cambiarte a «Ayuda del portal».

---

## Quién la ve

| Rol | ¿Puede preguntar? |
|---|---|
| Admin | Siempre, mientras la función esté encendida |
| Manager | Solo si le das **«Preguntar por los datos»** en 🔐 Permisos (viene apagado) y tiene **«Ver todos los tickets»** |
| Usuario | Nunca |

Para preguntar por **borradores** y **tickets archivados** también hace falta **«Revisar borradores»**. Sin ese permiso, la respuesta es «No tienes permiso para preguntar por borradores ni por tickets archivados.»

La fila «Preguntar por los datos» del editor de permisos (grupo Métricas y asistente) solo aparece mientras la función está encendida. Si la plataforma la apaga, la fila desaparece y lo que diste se conserva para cuando se vuelva a encender.

En «IA» › «Funciones de IA» ves si la función está **Encendida** o **Apagada**; no la puedes cambiar desde ahí.

---

## Qué puedes preguntar

Sobre los tickets de tu empresa, por ejemplo:

- Conteos: "¿Cuántos tickets abiertos hay?", "¿Cuántos tickets se crearon este mes?"
- Por grupo: "¿Cuántos tickets abiertos tiene cada área?", "¿Cuántos tickets hay en cada estado?"
- Por persona: "¿Quién tiene más tickets abiertos?", "¿Cuántos tickets tiene Ana López?"
- Tiempos: "¿Cuánto tardamos en resolver los tickets de Nómina este año?", "¿Cuántos tickets abiertos llevan más de 7 días?"
- Periodos: hoy, esta semana, la semana pasada, este mes, el mes pasado, este año, los últimos N días.
- Listados: "Lista los tickets abiertos de Finanzas" (hasta 50, los más recientes primero).
- Con «Revisar borradores»: "¿Cuántos borradores hay por revisar?", "¿Por qué motivo se archivaron los tickets este año?"

Usa los nombres de tus estados, áreas, categorías y personas tal como están en el portal. Si un nombre coincide con varias personas (dos "Ana"), el asistente te pregunta a cuál te refieres; contesta en el siguiente mensaje.

---

## Cómo se lee la respuesta

- **El texto** responde la pregunta con los números de los datos.
- **Periodo:** el rango de fechas que se usó. "ahora" quiere decir la foto actual (por ejemplo, los abiertos de hoy); "todo el histórico" quiere decir que no pediste un periodo.
- **Notas en cursiva:** los supuestos que se tomaron (por ejemplo, qué cuenta como cerrado).
- **La tabla:** los datos detrás de la respuesta. Se ven 10 filas; «Ver las n filas» muestra el resto. Si la consulta trajo más filas de las que se pueden mostrar, lo dice. Un solo número no lleva tabla.

"Abiertos" son los tickets que no están en un estado Cerrado. "Cerrados" suma resueltos y cancelados.

---

## Qué no responde

| Pregunta | Respuesta |
|---|---|
| Vencidos, SLA o fecha límite | Aún no está disponible |
| Comentarios, correos, adjuntos o descripción de un ticket | Esa información no está disponible |
| Datos de los solicitantes | No están disponibles |
| Usuarios, permisos o configuración del portal | No se consulta desde aquí |
| Otra empresa o cliente | Solo responde sobre tu propia empresa |
| Borrar, asignar, cerrar o cambiar algo | Solo consulta; no cambia nada |
| Cómo usar el portal | Cámbiate a «Ayuda del portal» |

Si no entiende la pregunta, dice "No pude convertir tu pregunta en una consulta. Intenta decirla de otra forma." Si la consulta tarda demasiado, pide acotar el periodo.

---

## Avisos

- **No hay reintentos automáticos:** si algo falla, el panel lo dice una vez y espera a que vuelvas a preguntar.
- "Se están haciendo demasiadas preguntas. Intenta en un momento." — hay un límite de preguntas por minuto para toda la empresa.
- "Las preguntas sobre los datos no están disponibles en este momento." — el servicio no responde; intenta más tarde.

---

## Privacidad y consumo

- Solo se consultan los datos de tu empresa; nunca los de otra.
- No se usan datos de contacto de los solicitantes, ni el texto de los correos, ni los comentarios.
- Cada pregunta queda registrada (quién, cuándo, la pregunta y si se respondió) para medir el consumo que la plataforma cobra; la respuesta y las filas no se guardan.
- Cada pregunta usa el modelo de IA, normalmente dos veces.
