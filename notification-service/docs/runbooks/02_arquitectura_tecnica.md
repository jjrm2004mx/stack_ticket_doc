# 02 — Arquitectura Técnica
## notification-service · Referencia de servicios, redes y configuración
**Mayo 2026**

---

## Índice

1. [Servicios del stack](#1-servicios-del-stack)
2. [Redes](#2-redes)
3. [Variables de entorno](#3-variables-de-entorno)
4. [Redis Streams — contratos de eventos](#4-redis-streams--contratos-de-eventos)
5. [Providers de email](#5-providers-de-email)
6. [Plantillas Thymeleaf](#6-plantillas-thymeleaf)
7. [Base de datos](#7-base-de-datos)
8. [Arquitectura hexagonal](#8-arquitectura-hexagonal)
9. [Métricas — Prometheus](#9-métricas--prometheus)

---

## 1. Servicios del stack

| Contenedor | Imagen | Puerto host | Puerto interno | Rol |
|---|---|---|---|---|
| `notification-service` | build local (Spring Boot) | 8081 | 8081 | Servicio principal |
| `notification-db` | postgres:15-alpine | 5435 | 5432 | Historial de notificaciones |

### Dependencia externa (no en este repo)

| Servicio | Ubicación | Rol |
|---|---|---|
| `ticket-system-redis` | ticket-management | Redis Streams — fuente de eventos |
| `til-n8n` | ticket-ingestion-light | Relay de correos (provider webhook) |

---

## 2. Redes

```yaml
networks:
  notification-network:
    driver: bridge              # Red interna del repo
  ticket-management-network:
    external: true              # Red compartida del ecosistema
```

| Red | Tipo | Propósito |
|---|---|---|
| `notification-network` | Interna | Entre notification-service y notification-db |
| `ticket-management-network` | Externa | Alcanzar ticket-system-redis y til-n8n |

`notification-service` está en ambas redes para poder:
- Comunicarse con `notification-db` (notification-network)
- Leer de `ticket-system-redis` y llamar a `til-n8n` (ticket-management-network)

---

## 3. Variables de entorno

### `.env.example`

```bash
# ── Provider de email ──────────────────────────────────────────────
# Opciones: webhook | resend | sendgrid
NOTIFICATION_EMAIL_PROVIDER=webhook

# URL del webhook que recibe el correo generado (til-n8n)
NOTIFICATION_WEBHOOK_URL=http://til-n8n:5678/webhook/notification-webhook

# URL del portal del solicitante (botón CTA en correos)
NOTIFICATION_PORTAL_URL=http://localhost:8090

# Remitente
NOTIFICATION_EMAIL_FROM=no-reply@ticket-manager.com
NOTIFICATION_EMAIL_FROM_NAME=TICKET-MANAGER

# Nombre de la aplicación en asunto/cabecera
NOTIFICATION_APP_NAME=Shared Service

# ── PostgreSQL ──────────────────────────────────────────────────────
POSTGRES_USER=notif_admin
POSTGRES_PASSWORD=notif_password_123
POSTGRES_HOST=notification-db
POSTGRES_PORT=5432
POSTGRES_DB=notifications_db

# ── Redis (ticket-system-redis — del stack de ticket-management) ────
REDIS_HOST=ticket-system-redis
REDIS_PORT=6379
REDIS_PASSWORD=
```

### Variables críticas

| Variable | Descripción |
|---|---|
| `NOTIFICATION_EMAIL_PROVIDER` | Controla qué backend de email usar |
| `NOTIFICATION_WEBHOOK_URL` | URL de til-n8n para relay de correos (provider webhook) |
| `NOTIFICATION_PORTAL_URL` | URL del botón "Ver ticket" en el correo — ajustar por entorno |
| `REDIS_HOST` | Debe ser `ticket-system-redis` — no `classifier-redis` |

---

## 4. Redis Streams — contratos de eventos

### Stream consumido

```
Stream: ticket.events
Consumer Group: notification-service-group
```

### Estructura de mensajes esperada

```json
{
  "tipo": "ticket_created",
  "ticket_id": "123",
  "titulo": "No puedo acceder al sistema",
  "estado": "ABIERTO",
  "solicitante_email": "usuario@empresa.com",
  "solicitante_nombre": "Juan García",
  "dominio": "IT",
  "categoria": "acceso",
  "prioridad": "ALTA"
}
```

### Tipos de evento soportados

| Tipo | Cuándo se dispara | Plantilla usada |
|---|---|---|
| `ticket_created` | Ticket nuevo creado | `ticket-created.html` |
| `ticket_updated` | Estado o datos cambiados | `ticket-updated.html` |
| `ticket_closed` | Ticket cerrado | `ticket-closed.html` |
| `comment_added` | Comentario agregado | `comment-added.html` |

---

## 5. Providers de email

### Provider `webhook` (activo en desarrollo)

```
notification-service
     │ POST NOTIFICATION_WEBHOOK_URL
     │ Body: { "to": "...", "subject": "...", "html": "..." }
     ▼
til-n8n :5678/webhook/notification-webhook
     └─ Nodo Gmail → envía correo real
```

### Provider `resend` / `sendgrid`

```
notification-service
     │ Llama API de Resend/SendGrid directamente
     └─ Requiere API key del provider en .env
```

Cambiar provider:
```bash
# En .env
NOTIFICATION_EMAIL_PROVIDER=resend
# Reiniciar el servicio
podman restart notification-service
```

---

## 6. Plantillas Thymeleaf

Las plantillas viven en `src/main/resources/templates/`.
Son archivos HTML con expresiones Thymeleaf `${variable}`.

```
templates/
├── layout.html              # Layout base con cabecera/pie comunes
├── ticket-created.html      # Correo: ticket nuevo
├── ticket-updated.html      # Correo: cambio de estado
├── ticket-closed.html       # Correo: ticket cerrado
└── comment-added.html       # Correo: comentario nuevo
```

Variables disponibles en todas las plantillas:

| Variable | Valor |
|---|---|
| `${appName}` | `NOTIFICATION_APP_NAME` |
| `${portalUrl}` | `NOTIFICATION_PORTAL_URL` |
| `${emailFrom}` | `NOTIFICATION_EMAIL_FROM_NAME` |
| `${ticket.id}` | ID del ticket |
| `${ticket.titulo}` | Título del ticket |
| `${ticket.estado}` | Estado actual |
| `${ticket.categoria}` | Categoría clasificada |

---

## 7. Base de datos

**notification-db** — PostgreSQL 15, puerto 5435 desde el host.

Tablas principales (Flyway en `src/main/resources/db/migration/`):

| Tabla | Contenido |
|---|---|
| `notification_log` | Historial de notificaciones enviadas |
| `notification_templates` | Registro de plantillas activas |

### Conexión directa

```
Host:     localhost
Port:     5435
Database: notifications_db
User:     notif_admin
Password: notif_password_123 (dev)
```

### Flyway

Las migraciones se aplican automáticamente al arrancar el servicio.
No modificar archivos `V__*.sql` ya aplicados.

---

## 8. Arquitectura hexagonal

El servicio sigue **Arquitectura Hexagonal** (Ports & Adapters):

```
┌──────────────────────────────────────────────────┐
│  Infraestructura                                  │
│  ┌────────────┐  ┌──────────────┐  ┌──────────┐  │
│  │Redis Stream│  │ Webhook/SMTP │  │  JPA/DB  │  │
│  │ Consumer   │  │  Adapter     │  │ Repos    │  │
│  └─────┬──────┘  └──────┬───────┘  └────┬─────┘  │
│        │ Port (in)       │ Port (out)    │ Port    │
├────────┼─────────────────┼───────────────┼─────────┤
│  Aplicación              │               │         │
│  NotificationUseCase ────┤               │         │
│  (orquesta el envío)                     │         │
├──────────────────────────────────────────┼─────────┤
│  Dominio                                 │         │
│  Notification, NotificationEvent, Status │         │
│  (sin dependencias Spring/JPA)           │         │
└──────────────────────────────────────────────────────┘
```

**Regla:** El dominio nunca importa clases de infraestructura.
Los adapters de infraestructura implementan las interfaces (ports) del dominio.

---

## 9. Métricas — Prometheus

El servicio expone métricas en:
```
http://localhost:8081/actuator/prometheus
```

Prometheus (infra-monitoring) scrapeó este endpoint cada 15 segundos.
Las métricas incluyen:
- `notifications_sent_total` — total de notificaciones enviadas por tipo
- `notifications_failed_total` — fallos por tipo de evento
- `redis_stream_consumer_lag` — retraso en consumo del stream
- Métricas estándar de Spring Boot (JVM, HTTP, DB pool)

---

*notification-service · Arquitectura Técnica · Mayo 2026*
