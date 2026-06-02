---
rol: admin
tipo: permisos
audiencia: usuario_final
fuente: ticket-management
---

# Permisos — Admin

El Admin tiene acceso total al sistema. Este documento detalla la matriz completa de permisos, qué puede modificar y qué está bloqueado por el propio sistema.

---

## Matriz completa de permisos — Admin

| Permiso | Admin | Manager | Usuario | Modificable por Admin |
|---|---|---|---|---|
| VIEW_TICKETS | ✅ Fijo | ✅ Fijo | ✅ Fijo | ❌ Siempre activo para todos |
| CREATE_TICKET | ✅ | ✅ (default) | ✅ (default) | ✅ Para Manager y Usuario |
| EDIT_TICKET | ✅ | ✅ (default) | ❌ (default) | ✅ Para Manager y Usuario |
| CHANGE_STATUS | ✅ | ✅ (default) | ❌ (default) | ✅ Para Manager y Usuario |
| ARCHIVE_TICKET | ✅ Fijo | ❌ No disponible | ❌ No disponible | ❌ Exclusivo Admin |
| ADD_COMMENT | ✅ | ✅ (default) | ✅ (default) | ✅ Para Manager y Usuario |
| VIEW_ATTACHMENTS | ✅ | ✅ (default) | ✅ (default) | ✅ Para Manager y Usuario |
| UPLOAD_ATTACHMENT | ✅ | ✅ (default) | ❌ (default) | ✅ Para Manager y Usuario |
| DELETE_ATTACHMENT | ✅ | ✅ (default) | ❌ (default) | ✅ Para Manager y Usuario |
| VIEW_TICKET_EVENTS | ✅ | ✅ (default) | ❌ (default) | ✅ Para Manager y Usuario |
| VIEW_DASHBOARD | ✅ | ✅ (default) | ❌ (default) | ✅ Para Manager y Usuario |
| MANAGE_USERS | ✅ Fijo | ❌ No disponible | ❌ No disponible | ❌ Exclusivo Admin |
| REVIEW_DRAFTS | ✅ | ✅ (default) | ❌ (default) | ✅ Para Manager y Usuario |
| MANAGE_CLASSIFICATIONS | ✅ Fijo | ❌ No disponible | ❌ No disponible | ❌ Exclusivo Admin |
| MANAGE_WORKFLOW | ✅ Fijo | ❌ No disponible | ❌ No disponible | ❌ Exclusivo Admin |

---

## Reglas fijas del sistema (no modificables)

Estas reglas están hardcodeadas en la lógica del sistema y ningún Admin puede cambiarlas:

### Siempre TRUE para todos los roles
- **VIEW_TICKETS** — todos los usuarios pueden ver tickets, sin excepción

### Exclusivos de Admin (no transferibles)
- **ARCHIVE_TICKET** — ningún rol diferente a Admin puede archivar
- **MANAGE_USERS** — la gestión de usuarios es exclusiva de Admin
- **MANAGE_CLASSIFICATIONS** — solo Admin gestiona el catálogo de clasificaciones
- **MANAGE_WORKFLOW** — solo Admin configura estados y transiciones

### Opcionales para Admin (Admin puede activarlos/desactivarlos sobre sí mismo)
- REVIEW_DRAFTS — activo por defecto para Admin, pero configurable
- otros permisos del Admin que no sean los 4 exclusivos arriba

---

## Cómo modificar permisos de otros roles

**Desde la pantalla Gestión de Permisos:**

1. Accede a **Gestión de Permisos** desde el menú
2. Verás la matriz completa con las filas: `(rol, acción, permitido)`
3. Cambia el valor de `allowed` para el par (rol, acción) que necesitas
4. Guarda — el cambio queda registrado con tu usuario y la fecha

**Ejemplo:** para permitir que los Usuarios puedan subir adjuntos:
- Busca la fila `(MANAGER=false)` de la acción UPLOAD_ATTACHMENT para el rol USER
- Cambia a `true`
- Guarda

**Impacto:** el cambio aplica a TODOS los usuarios de ese rol desde su próxima sesión.

---

## Permisos por rol — Configuración por defecto del sistema

### ROL: USER (Usuario)
```
VIEW_TICKETS:         TRUE  (fijo)
CREATE_TICKET:        TRUE
EDIT_TICKET:          FALSE
CHANGE_STATUS:        FALSE
ARCHIVE_TICKET:       FALSE (no disponible)
ADD_COMMENT:          TRUE
VIEW_ATTACHMENTS:     TRUE
UPLOAD_ATTACHMENT:    FALSE
DELETE_ATTACHMENT:    FALSE
MANAGE_USERS:         FALSE (no disponible)
VIEW_TICKET_EVENTS:   FALSE
VIEW_DASHBOARD:       FALSE
REVIEW_DRAFTS:        FALSE
MANAGE_CLASSIFICATIONS: FALSE (no disponible)
MANAGE_WORKFLOW:      FALSE (no disponible)
```

### ROL: MANAGER
```
VIEW_TICKETS:         TRUE  (fijo)
CREATE_TICKET:        TRUE
EDIT_TICKET:          TRUE
CHANGE_STATUS:        TRUE
ARCHIVE_TICKET:       FALSE (no disponible)
ADD_COMMENT:          TRUE
VIEW_ATTACHMENTS:     TRUE
UPLOAD_ATTACHMENT:    TRUE
DELETE_ATTACHMENT:    TRUE
MANAGE_USERS:         FALSE (no disponible)
VIEW_TICKET_EVENTS:   TRUE
VIEW_DASHBOARD:       TRUE
REVIEW_DRAFTS:        TRUE
MANAGE_CLASSIFICATIONS: FALSE (no disponible)
MANAGE_WORKFLOW:      FALSE (no disponible)
```

### ROL: ADMIN
```
Todos los permisos: TRUE
```

---

## Restricciones de transición de estado por rol

El permiso CHANGE_STATUS es necesario pero no suficiente. El sistema valida adicionalmente las reglas del workflow para cada transición:

| Transición | Usuario | Manager | Admin |
|---|---|---|---|
| ABIERTO → EN PROGRESO | ❌ sin CHANGE_STATUS | ✅ | ✅ |
| ABIERTO → CERRADO | ❌ | ❌ restricción workflow | ✅ exclusivo Admin |
| EN PROGRESO → EN REVISIÓN | ❌ | ✅ | ✅ |
| EN PROGRESO → ABIERTO | ❌ | ✅ | ✅ |
| EN REVISIÓN → CERRADO | ❌ | ✅ | ✅ |
| EN REVISIÓN → EN PROGRESO | ❌ | ✅ | ✅ |

---

## Auditoría de cambios en permisos

Cada modificación a la matriz de permisos queda registrada:
- **updatedBy**: usuario Admin que realizó el cambio
- **timestamp**: fecha y hora del cambio
- **qué cambió**: acción, rol, valor anterior, valor nuevo

Esto garantiza trazabilidad completa de quién modificó los permisos del sistema y cuándo.

---

## Preguntas frecuentes sobre permisos

**¿Puedo darle a un Manager el permiso de archivar tickets?**
No. ARCHIVE_TICKET es exclusivo de Admin por diseño del sistema y no puede asignarse a otro rol.

**¿Puedo quitarle a un Manager el permiso de cambiar estados?**
Sí. CHANGE_STATUS es configurable por Admin para el rol Manager. Si lo desactivas, los Managers no podrán ejecutar ninguna transición de estado.

**¿Qué pasa si desactivo CREATE_TICKET para el rol Usuario?**
Los usuarios no podrán crear nuevos tickets desde la interfaz. Seguirán pudiendo ver y comentar en tickets existentes.

**¿Los permisos aplican al usuario específico o al rol?**
Al **rol**. No hay permisos individuales por usuario — todos los usuarios del mismo rol tienen exactamente los mismos permisos. Para dar capacidades especiales a una persona, debes cambiar su rol.

**¿Cuándo se reflejan los cambios de permisos en los usuarios?**
En la próxima sesión del usuario (después de cerrar sesión y volver a entrar). Los cambios no se aplican a sesiones activas en curso.

**¿Puedo tener dos Admins?**
Sí. Puede haber múltiples usuarios con rol ADMIN. Todos tienen las mismas capacidades.
