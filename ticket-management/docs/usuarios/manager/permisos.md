---
rol: manager
tipo: permisos
audiencia: usuario_final
fuente: ticket-management
---

# Permisos del Manager

Este documento detalla qué puede y no puede hacer un Manager con la configuración por defecto, y cómo funciona el sistema de permisos. El Admin de tu organización puede cambiar la mayoría de estos valores en **🔐 Gestión de Permisos**.

---

## Permisos por grupo — Manager

Las acciones están agrupadas por la parte del portal que controlan.

### 🎫 Acceso a tickets

| Acción | ¿Activo? | Qué permite |
|---|---|---|
| Ver tickets | ✅ Siempre activo | Abrir el tablero, la lista y el detalle de los tickets que te corresponden |
| Ver tickets sin asignar | ✅ Activo | Ver también los tickets que aún no tienen responsable |
| Ver todos los tickets | ✅ Activo | Ver todos los tickets de la organización |
| Crear tickets | ✅ Activo | Botón «+ Nuevo ticket» |

### 🧾 Trabajo en el ticket

| Acción | ¿Activo? | Qué permite |
|---|---|---|
| Editar tickets | ✅ Activo | Cambiar título, descripción, prioridad, área y categoría, fecha límite y datos del solicitante |
| Asignar tickets | ✅ Activo | Elegir o quitar al responsable (incluye «Asignar a» al crear y al aprobar un borrador) |
| Cambiar estado | ✅ Activo | Mover el ticket entre estados, desde el detalle o arrastrando en el tablero |
| Agregar comentarios | ✅ Activo | Escribir y editar comentarios |
| Archivar tickets | ❌ No disponible | Solo Admin puede archivar |

### 📎 Adjuntos e historial

| Acción | ¿Activo? | Qué permite |
|---|---|---|
| Ver adjuntos | ✅ Activo | Ver y descargar archivos |
| Subir adjuntos | ✅ Activo | Agregar archivos a un ticket |
| Eliminar adjuntos | ✅ Activo | Borrar archivos de un ticket |
| Ver historial de eventos | ✅ Activo | Ver quién hizo qué y cuándo en el ticket |

### 📋 Bandeja de revisión

| Acción | ¿Activo? | Qué permite |
|---|---|---|
| Revisar borradores | ✅ Activo | Abrir la Bandeja de revisión y aprobar borradores |

### 📊 Métricas y asistente

| Acción | ¿Activo? | Qué permite |
|---|---|---|
| Ver dashboard de métricas | ✅ Activo | Vista «Métricas» con KPIs, gráficas y rendimiento por agente |
| Usar asistente | ✅ Activo | Widget del asistente con respuestas de la documentación |

### ⚙️ Administración

| Acción | ¿Activo? | Qué permite |
|---|---|---|
| Gestionar usuarios | ❌ No disponible | Solo Admin |
| Gestionar áreas y categorías | ❌ No disponible | Solo Admin |
| Gestionar workflow | ❌ No disponible | Solo Admin |
| Gestionar permisos | ❌ No disponible | Solo Admin |

---

## ¿Puede un Admin cambiar mis permisos?

Sí, con estas restricciones:

**Lo que el Admin puede activar o desactivar para Manager:** todas las acciones de los grupos Acceso a tickets (excepto Ver tickets), Trabajo en el ticket (excepto Archivar), Adjuntos e historial, Bandeja de revisión y Métricas y asistente.

**Lo que el Admin no puede darte:** Archivar tickets, Gestionar usuarios, Gestionar áreas y categorías, Gestionar workflow y Gestionar permisos — son exclusivos de Admin.

**Lo que el Admin no puede quitarte:** Ver tickets — siempre activo para todos los roles.

---

## Asignar y editar son permisos distintos

"Asignar tickets" y "Editar tickets" se configuran por separado:

- Con **Asignar tickets** ves el selector **"Asignado a"** en el detalle y **"Asignar a"** en Nuevo ticket y en la Bandeja de revisión. Solo aparecen usuarios activos de tu organización.
- Sin **Asignar tickets**, el detalle muestra el responsable sin selector y los borradores se aprueban sin responsable.
- Puedes tener uno sin el otro; por ejemplo, asignar tickets sin poder cambiar su prioridad.

---

## Restricciones específicas de transiciones de estado

Tener el permiso Cambiar estado no significa que puedas hacer cualquier cambio de estado: solo puedes mover un ticket por las transiciones que tu organización habilitó en el workflow (el detalle muestra un botón por cada una). Las transiciones no dependen del rol. Un ticket en un estado Cerrado solo se reabre si existe una transición que salga de ese estado.

---

## Permisos sobre borradores

Con "Revisar borradores" activo puedes:
- Ver la lista completa de borradores pendientes
- Revisar el contenido de cada borrador
- Editar prioridad, área y categoría, y el responsable si además tienes "Asignar tickets"
- Aprobar el borrador (entra al flujo en el estado inicial)

Lo que **no puedes hacer** con borradores:
- Eliminar un borrador
- Rechazar formalmente un borrador (solo aprobar o dejar pendiente)

---

## Preguntas frecuentes sobre permisos

**¿Por qué no puedo archivar tickets?**
Archivar oculta el ticket de todas las vistas, así que el sistema reserva esta capacidad para el Admin.

**¿Puedo operar en tickets de otros Managers?**
Sí. Con "Ver todos los tickets" ves todos los tickets de la organización, no solo los asignados a ti o creados por ti.

**¿Cómo sé qué permisos tengo activos actualmente?**
Tu Admin puede mostrarte la configuración actual desde la pantalla de Gestión de Permisos. Los permisos se cargan cuando inicias sesión o recargas el portal.

**¿Mis permisos cambian en tiempo real si el Admin los modifica?**
El servidor aplica el cambio en cuanto el Admin guarda: si intentas una acción que ya no tienes verás "No tienes permiso para esta acción". Los botones y pantallas se actualizan cuando recargas el portal o vuelves a iniciar sesión.
