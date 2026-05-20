# 01 — Ticket Management for Dummies
## ticket-management · Guía conceptual del stack
**Stack: Spring Boot · React · PostgreSQL · Redis · MinIO**
**Mayo 2026**

---

## La analogía: la oficina de soporte

Imagina una oficina de soporte técnico bien organizada.
ticket-management es esa oficina.

| Pieza de la oficina | Pieza del ecosistema |
|---|---|
| El tablero de tickets visible para todos | React — interfaz web para operadores |
| Los empleados que procesan solicitudes | Spring Boot — lógica de negocio |
| El archivador de todos los tickets | PostgreSQL — base de datos principal |
| El megáfono que anuncia novedades | Redis Streams — publica eventos a otros servicios |
| El archivero de adjuntos | MinIO — almacenamiento de archivos adjuntos |
| El guardia de seguridad en la puerta | Spring Security + JWT — autenticación |

---

## Rol en el ecosistema

ticket-management es el **núcleo** del ecosistema. Todos los demás repos
giran a su alrededor.

```
ticket-ingestion-light
     │ Reenvía correos Gmail al clasificador
     ▼
ticket-classification
     │ Clasifica y crea el ticket en:
     ▼
ticket-management  ◀──  aquí vive este repo
     │ Publica evento en Redis cuando algo cambia
     ▼
notification-service
     └─ Envía correo al solicitante del ticket

infra-monitoring  ◀──  métricas y logs
```

---

## Arquitectura hexagonal

El código sigue una regla estricta de capas que no pueden mezclarse:

```
infrastructure  →  application  →  domain
   (adapters)      (use cases)    (pure logic)
```

| Capa | Qué contiene | Puede importar |
|---|---|---|
| `domain` | Entidades, lógica pura de negocio | Nada externo |
| `application` | Casos de uso, servicios | Solo dominio |
| `infrastructure` | Controllers, JPA, Redis, MinIO | Dominio + aplicación |

Romper estas reglas introduce dependencias ocultas que hacen el sistema frágil.

---

## El ciclo de vida de un ticket

```
1. ticket-classification crea el ticket via API REST
2. Backend valida y persiste en PostgreSQL
3. Backend publica evento "ticket_created" en Redis Streams
4. notification-service consume el evento y notifica al solicitante
5. Operador revisa el ticket en la interfaz web
6. Operador actualiza el estado: ABIERTO → EN_PROCESO → CERRADO
7. Cada cambio de estado dispara un evento nuevo en Redis
8. notification-service notifica al solicitante de cada cambio
```

---

## Redis Streams — el megáfono

Cuando ticket-management cambia algo relevante, publica un evento en
el stream `ticket.events`. notification-service escucha ese stream
y envía el correo correspondiente.

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

> Si Redis no está disponible, el ticket **se guarda igual** en la base de
> datos. El fallo de Redis nunca bloquea la operación principal.

---

## MinIO — el archivero

MinIO es un almacenamiento de objetos compatible con la API de S3.
Se usa para guardar archivos adjuntos de tickets.

| Acceso | URL |
|---|---|
| Consola web | http://localhost:9003 |
| API S3 | http://localhost:9002 |
| Credenciales (dev) | minioadmin / minioadmin_password_123 |

---

## Workflows y transiciones de estado

El sistema tiene un motor de workflows configurable que define qué
transiciones de estado son válidas para cada ticket.

| Estado | Transiciones posibles |
|---|---|
| `ABIERTO` | → `EN_PROCESO`, → `CERRADO` |
| `EN_PROCESO` | → `RESUELTO`, → `EN_ESPERA` |
| `EN_ESPERA` | → `EN_PROCESO`, → `CERRADO` |
| `RESUELTO` | → `CERRADO`, → `REABIERTO` |
| `CERRADO` | — (estado final) |

---

## Servicios del stack

| Contenedor | Puerto host | Rol |
|---|---|---|
| `ticket-system-backend` | 8080 | Spring Boot API |
| `ticket-system-frontend` | 8090 | Frontend React |
| `ticket-db` | 5433 | PostgreSQL `tickets_db` |
| `ticket-system-redis` | 6380 | Redis (caché + streams) |
| `ticket-system-minio` | 9002 (API) / 9003 (consola) | Almacenamiento de adjuntos |
| `ticket-flyway` | — | Migraciones Flyway (one-shot) |

> PostgreSQL en **5433** (no 5432) y Redis en **6380** (no 6379) para evitar colisiones
> con otros stacks. MinIO API en **9002** y consola en **9003**.

---

*ticket-management · Ticket Management for Dummies · Mayo 2026*
