---
rol: usuario
tipo: estados
audiencia: usuario_final
fuente: ticket-management
---

# Estados de un ticket

Un ticket siempre tiene exactamente un estado. Los nombres de los estados los define tu organización; cada uno pertenece a una de estas categorías:

| Categoría | Icono | Qué significa |
|---|---|---|
| Nuevo | 📋 | El ticket acaba de entrar y espera que alguien lo tome. Todo ticket nuevo empieza en el estado inicial de tu organización. |
| En curso | ⚙️ | Alguien está trabajando en el ticket. |
| En espera | ⏸ | Se espera una respuesta de un tercero (por ejemplo, del cliente). Mientras tanto, la fecha límite se pausa. |
| Cerrado · Resuelto | ✅ | El ticket se atendió y terminó con solución. |
| Cerrado · Cancelado | ✖ | El ticket terminó sin solución (por ejemplo, un duplicado). |

Con el workflow de fábrica los estados son Abierto, En progreso, En revisión y Cerrado.

---

## Qué puedes hacer en cada estado

- **Mientras el ticket no está Cerrado:** ver el detalle, comentar y ver adjuntos. Si tu rol tiene "Editar tickets", también puedes corregir la fecha límite y el solicitante.
- **Cuando está Cerrado:** sigues viendo el ticket y puedes comentar, pero ya no se edita. Solo se reabre si tu organización permite pasar de ese estado a otro.
- **Cambiar el estado:** solo si tu rol tiene el permiso "Cambiar estado", y solo hacia los estados que tu organización configuró.

## La fecha límite En espera

Si el ticket pasa a un estado En espera, el tablero y la lista muestran la fecha límite con un ícono de pausa y el ticket no aparece como vencido. Cuando sale de la espera, la fecha límite se recorre los días que esperó.

## Ocultar los tickets cerrados

El interruptor "Activos" (verde) oculta los tickets en estados Cerrados; "Todos" (azul) los vuelve a mostrar.
