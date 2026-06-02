# Ecosistema de Gestión de Tickets — Resumen Ejecutivo

## Propósito general

El sistema automatiza el ciclo de vida completo de tickets de soporte, desde la recepción de un correo electrónico de un usuario hasta la notificación de resolución, pasando por clasificación inteligente, gestión del flujo de trabajo y auditoría.

---

## Los cinco componentes y su rol

### 1. Ticket Ingestion Light — Puerta de entrada

Monitorea una bandeja de Gmail y, en cuanto llega un correo, lo captura y reenvía automáticamente al clasificador. No toma ninguna decisión: su único trabajo es detectar el correo y disparar el proceso.

### 2. Ticket Classification — Cerebro de entrada

Recibe el correo y aplica inteligencia artificial para determinar:

- A qué dominio pertenece (IT, operaciones, cliente, otro)
- Qué categoría y prioridad corresponde
- Si es un ticket nuevo o una respuesta a uno existente

Evita duplicados, detecta si un correo de seguimiento aporta información relevante o es solo un "gracias", y crea el ticket ya clasificado en el sistema de gestión.

### 3. Ticket Management — Núcleo del sistema

Es el sistema de gestión propiamente dicho, equivalente a un Jira interno. Permite:

- Crear, editar, asignar y cerrar tickets con estado, prioridad, tipo y fechas
- Definir flujos de trabajo configurables por organización, con transiciones de estado controladas
- Registrar comentarios, adjuntos y horas trabajadas
- Mantener un historial de auditoría completo de cada cambio
- Soportar múltiples organizaciones de forma aislada (multi-tenant)
- Gestionar tickets como borradores que requieren aprobación antes de activarse

Cada cambio relevante genera un evento que alimenta al servicio de notificaciones.

### 4. Notification Service — Comunicación con el solicitante

Escucha los eventos del sistema de gestión y envía correos HTML personalizados al solicitante del ticket según el tipo de cambio (creación, cambio de estado, asignación, comentario, cierre). Solo notifica cuando la configuración del flujo lo indica, y registra cada intento de envío para auditoría y reintento.

### 5. Infra Monitoring — Observabilidad del ecosistema

Plataforma centralizada que recopila métricas y logs de todos los servicios anteriores, exponiéndolos en dashboards unificados. Permite detectar problemas de rendimiento o fallos en cualquier punto del ecosistema.

---

## Flujo de extremo a extremo

```
Correo del usuario
       ↓
Ticket Ingestion Light   (detecta y reenvía)
       ↓
Ticket Classification    (clasifica con IA, crea el ticket)
       ↓
Ticket Management        (gestión, asignación, seguimiento)
       ↓
Notification Service     (informa al solicitante de cada cambio)
       ↓
Infra Monitoring         (observa la salud de todo lo anterior)
```

---

## Capacidades clave del sistema de gestión

- Clasificación automática de solicitudes entrantes sin intervención manual
- Flujos de trabajo configurables por organización
- Soporte multi-tenant con aislamiento completo de datos
- Historial de auditoría inmutable de cada ticket
- Comunicación automática con el solicitante en cada hito relevante
- Visibilidad operativa centralizada del ecosistema completo
