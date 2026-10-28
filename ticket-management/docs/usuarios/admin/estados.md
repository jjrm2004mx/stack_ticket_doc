---
rol: admin
tipo: estados
audiencia: usuario_final
fuente: ticket-management
---

# Estados y transiciones — Admin

Cada organización define sus propios estados de ticket. Como Admin (con el permiso "Gestionar workflow") configuras los estados, las transiciones entre ellos y a quién se le avisa. Este documento explica el modelo y la pantalla de configuración.

---

## Nombre y categoría de un estado

Cada estado tiene:

- **Nombre:** lo eliges tú y lo ve todo el mundo (por ejemplo "Esperando cliente" o "En diagnóstico"). Lo puedes cambiar cuando quieras.
- **Categoría:** lo que entiende el sistema. Es fija desde que creas el estado:

| Categoría | Significa | Ejemplos |
|---|---|---|
| Nuevo | El ticket acaba de entrar | Abierto, Recibido |
| En curso | Alguien trabaja en el ticket | En progreso, En revisión, En diagnóstico |
| En espera | Se espera a un tercero (cliente, proveedor) | Esperando cliente |
| Cerrado · Resuelto | El ticket terminó con solución | Cerrado, Resuelto |
| Cerrado · Cancelado | El ticket terminó sin solución | Cancelado, Duplicado |

Las métricas, los filtros, los contadores del tablero y las reglas de edición siguen la **categoría**, nunca el nombre. Renombrar "Cerrado" a "Finalizado" no cambia nada del comportamiento.

Para "cambiar" la categoría de un estado, crea uno nuevo con la categoría correcta y desactiva el anterior moviendo sus tickets.

## Estado inicial

Todo ticket nuevo empieza en el **estado inicial**: los creados en el portal, los que llegan por correo y los borradores aprobados. Debe ser de categoría Nuevo. Lo eliges con "Hacer inicial".

## Workflow de fábrica

| Estado | Categoría | Avisa al solicitante |
|---|---|---|
| Abierto (inicial) | Nuevo | sí |
| En progreso | En curso | no |
| En revisión | En curso | no |
| Cerrado | Cerrado · Resuelto | sí |

Transiciones de fábrica: Abierto → En progreso, Abierto → Cerrado, En progreso → En revisión, En revisión → En progreso y En revisión → Cerrado.

---

## Pantalla "⚙️ Gestión de Workflow"

**Acceso:** "Workflow" en el menú de la izquierda; tiene ayuda "?". **Permiso:** "Gestionar workflow".

### Estados

- **"+ Nuevo estado":** nombre (hasta 50 caracteres, sin repetir), categoría, resultado (solo para Cerrado), descripción y color.
- **"Editar":** cambia nombre, descripción y color. La categoría y el resultado se muestran bloqueados.
- **↑ / ↓:** cambia el orden; es el orden de las columnas del tablero y de la línea de tiempo pública.
- **"Hacer inicial":** en estados activos de categoría Nuevo.
- **"Desactivar":** pregunta a qué estado mover los tickets que están ahí, los mueve y desactiva el estado. Los movimientos **no envían correo** a los solicitantes y quedan en el historial. El historial conserva el nombre anterior. No se puede desactivar el estado inicial ni el último estado Cerrado · Resuelto.
- **"Activar":** vuelve a usar un estado inactivo; después habilita sus transiciones.

Los estados nunca se borran.

### Transiciones

Una cuadrícula "Desde ↓ \ Hacia →" con un interruptor por par de estados activos. Cualquier rol con el permiso "Cambiar estado" puede mover un ticket por las transiciones activas, desde el tablero o desde el detalle. **Una transición que sale de un estado Cerrado permite reabrir** tickets cerrados.

### Notificaciones

El interruptor "Avisar al entrar" de un estado decide si el solicitante recibe correo cuando el ticket entra a ese estado. La creación, la asignación y los comentarios tienen sus propios interruptores en Notificaciones; lo que no esté encendido no se envía. Siempre hace falta que el ticket tenga activados los avisos y que el solicitante tenga correo. Por defecto solo avisan los estados Cerrado. Al cerrar, el solicitante recibe un correo distinto si el cierre fue Resuelto o Cancelado.

### Reglas que la pantalla hace cumplir

El banner superior muestra en rojo lo que impide guardar y en ámbar los avisos. La pantalla rechaza los cambios que dejarían:

1. sin estado inicial, o con un inicial que no sea de categoría Nuevo;
2. sin ningún estado Cerrado · Resuelto activo;
3. un estado (o el inicial) sin camino posible a un estado Cerrado;
4. más de 30 estados.

Un estado recién creado y todavía sin conectar solo genera aviso: agrega sus transiciones en el orden que quieras.

---

## En espera pausa la fecha límite

Mientras un ticket está en un estado de categoría En espera, su fecha límite está **pausada**: no se marca como vencido y en el tablero aparece "⏸". Al salir de la espera, la fecha límite se recorre los días que esperó y queda registrado como "Fecha límite cambiada". El tiempo en espera no cuenta en el MTTR.

## Estados especiales

- **Borrador:** un ticket sin estado (por ejemplo, un correo pendiente de revisión). Al aprobarlo entra al estado inicial.
- **Archivado:** no es un estado. Un ticket archivado conserva su estado, desaparece de las vistas y métricas y ya no cambia de estado ni se edita. Se puede archivar un ticket en cualquier estado, también cerrado.

---

## Preguntas frecuentes

**¿Puedo reabrir un ticket cerrado?**
Sí, si habilitas una transición que salga del estado Cerrado (por ejemplo, Cerrado → En progreso). Con el workflow de fábrica no hay ninguna.

**¿Qué diferencia hay entre Cancelado y archivado?**
Cancelado es un cierre (cuenta como cerrado y no entra en el SLA ni en el MTTR). Archivar oculta el ticket de las vistas sin cambiar su estado.

**¿Qué pasa con los tickets si desactivo un estado?**
Se mueven al estado que elijas, sin correo a los solicitantes. Los archivados conservan el estado inactivo.

**¿Cambiar el nombre de un estado afecta las integraciones?**
No. Cada estado tiene además un código técnico fijo (por ejemplo `OPEN`) que usan las integraciones y que no cambia.
