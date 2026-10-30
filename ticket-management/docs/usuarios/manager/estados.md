---
rol: manager
tipo: estados
audiencia: usuario_final
fuente: ticket-management
---

# Estados y transiciones — Manager

Los estados de los tickets los define cada organización. Como Manager puedes ver el estado de todos los tickets y moverlos por las transiciones que configuró tu organización, si tu rol tiene el permiso "Cambiar estado".

---

## Nombre y categoría

Cada estado tiene un **nombre** que eligió tu organización y una **categoría** que entiende el sistema:

| Categoría | Significa |
|---|---|
| Nuevo | El ticket acaba de entrar. Todo ticket nuevo empieza en el **estado inicial**, que es de esta categoría. |
| En curso | Alguien trabaja en el ticket. |
| En espera | Se espera a un tercero (cliente, proveedor). La fecha límite se pausa. |
| Cerrado · Resuelto | Terminó con solución. |
| Cerrado · Cancelado | Terminó sin solución. |

En el tablero cada estado activo es una columna con el icono de su categoría: 📋 Nuevo, ⚙️ En curso, ⏸ En espera, ✅ Resuelto y ✖ Cancelado.

## Workflow de fábrica

Abierto (Nuevo, inicial) → En progreso (En curso) → En revisión (En curso) → Cerrado (Cerrado · Resuelto). Transiciones: Abierto → En progreso, Abierto → Cerrado, En progreso → En revisión, En revisión → En progreso y En revisión → Cerrado. Tu organización puede renombrarlos, agregar otros (por ejemplo "Esperando cliente" o "Cancelado") y cambiar las transiciones.

## Cambiar el estado de un ticket

- **Tablero:** arrastra la tarjeta a otra columna. Si la transición no está configurada aparece "Transición no permitida en el flujo configurado: <origen> → <destino>".
- **Detalle:** el campo "Estado", arriba de la columna derecha, ofrece cada estado al que se puede pasar desde el estado actual.
- **Reabrir:** un ticket en un estado Cerrado solo se puede reabrir si tu organización habilitó una transición que salga de ese estado.

Un ticket archivado no cambia de estado. Un ticket en un estado Cerrado no se edita (título, descripción, prioridad, fecha límite, solicitante, área y categoría), pero sí acepta comentarios, adjuntos y asignación.

## Fecha límite y En espera

Mientras el ticket está En espera, su fecha límite está pausada: no se marca como vencido y el tablero muestra "⏸". Al salir de la espera, la fecha límite se recorre los días que esperó y el historial de eventos lo muestra como "Fecha límite cambiada".

## Métricas por categoría

- **Abiertos:** tickets que no están en un estado Cerrado.
- **Cerrados:** tickets en un estado Cerrado; el subtítulo indica cuántos se cancelaron.
- **Cumplimiento SLA y MTTR:** solo cuentan los cierres Resueltos; el MTTR descuenta el tiempo En espera.
- **Vencidos:** fecha límite pasada y ticket que no está Cerrado ni En espera.

---

## Preguntas frecuentes

**¿Por qué no veo una columna "Cerrado"?**
Con "Activos" (verde) se ocultan los tickets y las columnas de los estados Cerrados; cambia a "Todos" (azul).

**¿Cancelar es lo mismo que archivar?**
No. Cancelar es mover el ticket a un estado Cerrado · Cancelado: cuenta como cerrado. Archivar lo oculta de las vistas sin cambiar su estado.

**¿Quién configura los estados?**
Un rol con el permiso "Gestionar workflow" (de fábrica, el Admin).
