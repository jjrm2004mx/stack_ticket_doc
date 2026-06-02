---
rol: manager
tipo: permisos
audiencia: usuario_final
fuente: ticket-management
---

# Permisos del Manager

Este documento detalla exactamente qué puede y no puede hacer un Manager en el sistema, y cómo funciona el sistema de permisos.

---

## Matriz completa de permisos — Manager

| Permiso | ¿Activo? | Descripción |
|---|---|---|
| VIEW_TICKETS | ✅ Siempre activo | Ver todos los tickets del sistema |
| CREATE_TICKET | ✅ Activo | Crear nuevos tickets |
| EDIT_TICKET | ✅ Activo | Editar campos de tickets existentes |
| CHANGE_STATUS | ✅ Activo | Cambiar el estado de un ticket |
| ADD_COMMENT | ✅ Activo | Agregar comentarios en tickets |
| VIEW_ATTACHMENTS | ✅ Activo | Ver y descargar adjuntos |
| UPLOAD_ATTACHMENT | ✅ Activo | Subir archivos adjuntos |
| DELETE_ATTACHMENT | ✅ Activo | Eliminar adjuntos existentes |
| VIEW_TICKET_EVENTS | ✅ Activo | Ver historial completo de eventos |
| VIEW_DASHBOARD | ✅ Activo | Ver métricas y estadísticas |
| REVIEW_DRAFTS | ✅ Activo | Revisar y aprobar tickets borrador |
| ARCHIVE_TICKET | ❌ No disponible | Solo Admin puede archivar |
| MANAGE_USERS | ❌ No disponible | Solo Admin puede gestionar usuarios |
| MANAGE_CLASSIFICATIONS | ❌ No disponible | Solo Admin puede gestionar clasificaciones |
| MANAGE_WORKFLOW | ❌ No disponible | Solo Admin puede configurar el workflow |

**Nota importante:** VIEW_TICKETS está marcado como "siempre activo" en el sistema — no puede ser desactivado por ningún Admin para ningún rol.

---

## ¿Puede un Admin cambiar mis permisos?

El Admin puede ajustar algunos permisos de Manager, pero hay restricciones:

**Permisos que el Admin SÍ puede cambiar para el rol Manager:**
- CREATE_TICKET
- EDIT_TICKET
- CHANGE_STATUS
- ADD_COMMENT
- VIEW_ATTACHMENTS
- UPLOAD_ATTACHMENT
- DELETE_ATTACHMENT
- VIEW_TICKET_EVENTS
- VIEW_DASHBOARD
- REVIEW_DRAFTS

**Permisos que el Admin NO puede darte como Manager:**
- ARCHIVE_TICKET — exclusivo de Admin, no transferible
- MANAGE_USERS — exclusivo de Admin, no transferible
- MANAGE_CLASSIFICATIONS — exclusivo de Admin, no transferible
- MANAGE_WORKFLOW — exclusivo de Admin, no transferible

**Permisos que el Admin no puede quitarte:**
- VIEW_TICKETS — siempre activo para todos los roles

---

## Restricciones específicas de transiciones de estado

Tener el permiso CHANGE_STATUS no significa que puedas hacer cualquier cambio de estado. El sistema valida adicionalmente las reglas del workflow:

| Transición | ¿Manager puede? | Razón |
|---|---|---|
| ABIERTO → EN PROGRESO | ✅ | Sin restricción de rol |
| EN PROGRESO → EN REVISIÓN | ✅ | Sin restricción de rol |
| EN PROGRESO → ABIERTO | ✅ | Sin restricción de rol |
| EN REVISIÓN → CERRADO | ✅ | Manager tiene este permiso |
| EN REVISIÓN → EN PROGRESO | ✅ | Sin restricción de rol |
| ABIERTO → CERRADO | ❌ | Requiere rol ADMIN |
| Cualquier estado → ABIERTO desde CERRADO | ❌ | CERRADO es terminal |

---

## Permisos sobre borradores

Con REVIEW_DRAFTS activo puedes:
- Ver la lista completa de borradores pendientes
- Abrir y revisar el contenido de cada borrador (incluyendo el email original HTML si aplica)
- Editar: prioridad, clasificación, categoría, asignado a
- Aprobar el borrador (lo activa como ticket ABIERTO)

Lo que **no puedes hacer** con borradores aunque seas Manager:
- Eliminar un borrador
- Rechazar formalmente un borrador (solo aprobar o dejar pendiente)

---

## Preguntas frecuentes sobre permisos

**¿Por qué no puedo archivar tickets?**
Archivar tickets es una acción que puede ocultar información de auditoría. El sistema reserva esta capacidad únicamente para el Admin para garantizar la trazabilidad.

**¿Puedo operar en tickets de otros Managers?**
Sí. Los permisos de Manager aplican a todos los tickets del sistema, no solo a los que te están asignados o creados por ti.

**¿Cómo sé qué permisos tengo activos actualmente?**
Tu Admin puede mostrarte la configuración actual desde la pantalla de Gestión de Permisos. Los permisos se cargan cuando inicias sesión y reflejan la configuración más reciente.

**¿Mis permisos cambian en tiempo real si el Admin los modifica?**
El cambio se aplica después de que recargas la aplicación o inicias sesión nuevamente. Si el Admin acaba de cambiar algo, cierra sesión y vuelve a entrar para ver los permisos actualizados.
