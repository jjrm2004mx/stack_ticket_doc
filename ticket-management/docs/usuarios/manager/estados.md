---
rol: manager
tipo: estados
audiencia: usuario_final
fuente: ticket-management
---

# Estados y transiciones — Manager

Como Manager puedes ver el estado de todos los tickets y ejecutar la mayoría de las transiciones. Esta guía detalla qué significa cada estado y qué transiciones puedes realizar tú.

---

## Los cuatro estados

### ABIERTO (OPEN)
**Qué significa:** ticket registrado, esperando ser tomado por el equipo.
**Es estado inicial:** sí — todos los tickets nuevos comienzan aquí.
**Notifica al solicitante:** según configuración del sistema.

**Transiciones que puedes ejecutar desde ABIERTO:**
- **ABIERTO → EN PROGRESO** ✅ (sin restricción de rol — requiere permiso CHANGE_STATUS)

**Transiciones que NO puedes ejecutar desde ABIERTO:**
- **ABIERTO → CERRADO** ❌ — esta transición requiere rol ADMIN

---

### EN PROGRESO (IN_PROGRESS)
**Qué significa:** alguien del equipo está trabajando activamente en este ticket.

**Transiciones que puedes ejecutar desde EN PROGRESO:**
- **EN PROGRESO → EN REVISIÓN** ✅ — cuando el trabajo está listo para revisión
- **EN PROGRESO → ABIERTO** ✅ — para regresar el ticket si necesitas reasignarlo o hay un bloqueo

**Transiciones que NO puedes ejecutar:**
- **EN PROGRESO → CERRADO** ❌ — no es una transición válida en el sistema. El ticket debe pasar por EN REVISIÓN antes de cerrarse.

---

### EN REVISIÓN (IN_REVIEW)
**Qué significa:** el trabajo está terminado y se está verificando antes del cierre. Requiere que tú u otro Manager confirme que está correcto antes de cerrar.

**Transiciones que puedes ejecutar desde EN REVISIÓN:**
- **EN REVISIÓN → CERRADO** ✅ — esta es tu transición más importante. Solo Manager y Admin pueden cerrar un ticket.
- **EN REVISIÓN → EN PROGRESO** ✅ — si la revisión detecta problemas y el trabajo debe continuar.

**Transiciones que NO puedes ejecutar:**
- **EN REVISIÓN → ABIERTO** ❌ — no es una transición válida.

---

### CERRADO (CLOSED)
**Qué significa:** ticket resuelto. Estado terminal.
**Sin salida posible:** un ticket cerrado no puede cambiar de estado bajo ninguna circunstancia.
**Registra:** fecha de cierre (`resolvedAt`) y quién lo cerró (`resolvedBy`).

---

## Mapa completo de transiciones para Manager

```
                    ┌─────────────────────────────┐
                    ▼                             │
[ABIERTO] ──────▶ [EN PROGRESO] ──────▶ [EN REVISIÓN] ──────▶ [CERRADO]
                    │                             │
                    └─────────────────────────────┘
                    (regreso a ABIERTO si hay bloqueo)
```

| Desde | Hacia | ¿Puedes? | Notas |
|---|---|---|---|
| ABIERTO | EN PROGRESO | ✅ | Transición normal |
| ABIERTO | EN REVISIÓN | ❌ | No permitida en el sistema |
| ABIERTO | CERRADO | ❌ | Solo ADMIN |
| EN PROGRESO | EN REVISIÓN | ✅ | Transición normal |
| EN PROGRESO | ABIERTO | ✅ | Regresión permitida |
| EN PROGRESO | CERRADO | ❌ | No permitida, usar EN REVISIÓN primero |
| EN REVISIÓN | CERRADO | ✅ | Solo Manager o Admin pueden cerrar |
| EN REVISIÓN | EN PROGRESO | ✅ | Si la revisión detecta problemas |
| EN REVISIÓN | ABIERTO | ❌ | No permitida |
| CERRADO | cualquiera | ❌ | Estado terminal, sin salida |

---

## Estados especiales

### BORRADOR (Draft)
Los borradores no tienen estado visible como Abierto o Cerrado. Son tickets que llegaron al sistema (normalmente por correo electrónico) y que aún no fueron activados. Su campo `status` es nulo en la base de datos.

**Como Manager:**
- Puedes ver todos los borradores en la sección **"Revisión de Borradores"**
- Puedes editar: prioridad, clasificación, categoría, asignado a
- Puedes **aprobar** el borrador → se activa con estado ABIERTO y entra al flujo normal
- **No puedes** rechazar/eliminar un borrador — eso es función del Admin

**Cuándo llega un borrador:** cuando el sistema de ingesta (ticket-ingestion-light) procesa un correo y no puede clasificarlo automáticamente con certeza, lo crea como borrador para revisión humana.

---

## Preguntas frecuentes sobre transiciones

**¿Por qué no puedo cerrar un ticket que está en ABIERTO directamente?**
La transición ABIERTO → CERRADO está reservada solo para Admin. El flujo correcto es: ABIERTO → EN PROGRESO → EN REVISIÓN → CERRADO.

**¿Puedo deshacer un cierre?**
No. Una vez cerrado, el ticket no puede cambiar de estado. Si el problema continúa, el solicitante debe crear un ticket nuevo.

**¿Qué pasa si cierro un ticket por error?**
Contacta a un Admin. Solo Admin puede gestionar situaciones de cierre incorrecto (aunque técnicamente tampoco pueden reabrir — tendrán que crear un ticket nuevo y referenciarlo).

**¿Puedo cambiar el estado de tickets que no me están asignados?**
Sí, siempre que tengas el permiso CHANGE_STATUS activo. Como Manager, este permiso está habilitado por defecto.

**¿Qué registra el sistema cuando cambio un estado?**
Registra en `ticket_status_history`: el estado anterior, el estado nuevo, tu usuario (changed_by) y la fecha/hora exacta (changed_at).
