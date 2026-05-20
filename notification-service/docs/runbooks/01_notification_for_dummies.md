# 01 — Notification for Dummies
## notification-service · Guía conceptual del stack
**Stack: Spring Boot · PostgreSQL · Redis Streams · Thymeleaf · Webhook**
**Mayo 2026**

---

## La analogía: la secretaria de avisos

Imagina una secretaria cuyo único trabajo es estar atenta a lo que pasa
en la oficina y mandar correos de aviso a los solicitantes cuando algo
cambia en su ticket.

| Pieza de la secretaría | Pieza del ecosistema |
|---|---|
| La secretaria que vigila | notification-service — escucha eventos de Redis |
| La lista de eventos pendientes | Redis Streams (de ticket-system-redis) |
| La plantilla de correo oficial | Thymeleaf — plantillas HTML de los correos |
| El mensajero que entrega el correo | Provider de email (webhook/resend/sendgrid) |
| El buzón de envíos entregados | notification-db — historial de notificaciones |
| El mensajero externo (Fase 1) | til-n8n — recibe el webhook y envía el correo real |

---

## Rol en el ecosistema

Este repo es el **sistema de avisos** del ecosistema. No crea tickets,
no clasifica correos — solo escucha lo que pasa en ticket-management
y notifica a los solicitantes.

```
ticket-management (ticket-system-backend)
     │ Publica evento en Redis Streams
     │ "ticket_created", "ticket_updated", "ticket_closed"
     ▼
ticket-system-redis :6380
     │ Stream: ticket.events
     ▼
notification-service :8081  ◀── aquí vive este repo
     ├─ Consume el evento
     ├─ Genera email HTML con Thymeleaf
     ├─ POST webhook → til-n8n :5678
     └─ Guarda registro en notification-db :5435
          │
          ▼
     til-n8n (ticket-ingestion-light)
     └─ Webhook recibido → envía el correo real por Gmail
```

---

## El flujo de una notificación

```
1. Operador cambia estado de un ticket en ticket-management
2. Backend publica evento en Redis Stream "ticket.events"
3. notification-service detecta el evento (consumer group)
4. Selecciona la plantilla Thymeleaf según tipo de evento
5. Genera HTML del correo con datos del ticket
6. POST al webhook URL (til-n8n) con el HTML generado
7. n8n recibe el webhook y lo envía por Gmail al solicitante
8. notification-service registra el envío en notification-db
```

---

## Proveedores de email

El servicio soporta múltiples proveedores, configurable sin cambiar código:

| Provider | Descripción | Estado |
|---|---|---|
| `webhook` | Delega a un servicio externo via HTTP POST (til-n8n) | Activo en dev/prod |
| `resend` | API de email transaccional (resend.com) | Disponible |
| `sendgrid` | API de email transaccional (sendgrid.com) | Disponible |

```bash
# Cambiar provider en .env
NOTIFICATION_EMAIL_PROVIDER=webhook   # til-n8n como relay
NOTIFICATION_EMAIL_PROVIDER=resend    # Resend directo
NOTIFICATION_EMAIL_PROVIDER=sendgrid  # SendGrid directo
```

---

## Redis Streams — cómo escucha eventos

Redis Streams es una estructura de datos de Redis similar a una cola de
mensajes persistente. ticket-management **publica** eventos y
notification-service los **consume** como parte de un consumer group.

```
ticket-management                notification-service
      │                                │
      │  XADD ticket.events *          │
      │  { tipo: "ticket_created",     │
      │    ticket_id: 123,             │
      │    solicitante: "j@j.com" }   │
      │                                │
      ▼                                │
  Redis Stream                         │
  "ticket.events"  ────────────────▶  XREADGROUP
                                       │
                                       ▼
                                  Procesa evento
```

> notification-service usa el **ticket-system-redis** de ticket-management
> (puerto 6380), no el `classifier-redis` de ticket-classification.

---

## APP_PUBLIC_URL — la URL del portal en los correos

Los correos enviados incluyen un botón CTA ("Ver ticket") que enlaza
al portal frontend de ticket-management.

```bash
# En .env — ajustar según el entorno
NOTIFICATION_PORTAL_URL=http://192.168.137.32:8090  # IP del host Windows en WSL
```

El `start.sh` actualiza esta URL automáticamente leyendo `~/stack_ticket/.env.deploy`.

---

## Plantillas de correo (Thymeleaf)

Cada tipo de evento tiene una plantilla HTML separada en
`src/main/resources/templates/`. Las plantillas incluyen:
- Cabecera con el nombre del sistema (`NOTIFICATION_APP_NAME`)
- Cuerpo con los datos del ticket (ID, título, estado, categoría)
- Botón CTA que apunta a `NOTIFICATION_PORTAL_URL`
- Pie con el remitente (`NOTIFICATION_EMAIL_FROM_NAME`)

---

## Servicios del stack

| Contenedor | Puerto host | Rol |
|---|---|---|
| `notification-service` | 8081 | Servicio principal (Spring Boot) |
| `notification-db` | 5435 | PostgreSQL — historial de notificaciones |

---

*notification-service · Notification for Dummies · Mayo 2026*
