# 01 — Ticket Management for Dummies
## ticket-management · Guía conceptual del stack
**Stack: Spring Boot · React · PostgreSQL · Redis · MinIO · Flyway**
**Mayo 2026**

---

## La analogía: el Jira propio del ecosistema

Imagina una oficina que gestiona solicitudes de soporte. Cada correo que
llega del clasificador se convierte en un ticket, los agentes lo atienden,
y el sistema lleva el registro completo.

| Pieza de la oficina | Pieza del ecosistema |
|---|---|
| El mostrador de atención | Frontend React — interfaz web del sistema |
| El motor de gestión interno | Backend Spring Boot — toda la lógica de negocio |
| El archivero de expedientes | ticket-db — PostgreSQL con todos los tickets |
| La bandeja de mensajes urgentes | ticket-system-redis — caché y sesiones |
| El repositorio de documentos adjuntos | MinIO — almacena archivos subidos |
| El encargado de la estructura de archivos | Flyway — migraciones automáticas de BD |

---

## Rol en el ecosistema

Este repo es el **núcleo del ecosistema** — el destino final de los tickets
clasificados y la interfaz central para los operadores humanos.

```
ticket-ingestion-light
     │ Gmail correo
     ▼
ticket-classification (agente IA)
     │ ticket clasificado
     ▼
ticket-management ◀── aquí vive este repo
     ├─ ticket-system-backend :8080  (API REST + Spring Boot)
     ├─ ticket-system-frontend :8090  (React)
     ├─ ticket-db :5433              (PostgreSQL)
     ├─ ticket-system-redis :6380    (Redis)
     └─ MinIO :9002/:9003            (almacenamiento)
          │
          ▼
notification-service
(escucha Redis Streams → envía correos de aviso)
```

---

## Qué hace este repo

| Función | Detalle |
|---|---|
| Recibir tickets del clasificador | API REST protegida por `INTERNAL_API_KEY` |
| Gestionar tickets | Kanban board, vistas lista y dashboard |
| Autenticación de usuarios | JWT — login, roles, permisos |
| Gestión de workflows | Estados y transiciones configurables |
| Catálogo de categorías | Dominios y categorías que el clasificador consulta |
| Comentarios en tickets | Historial de interacciones por ticket |
| Adjuntos | Subida y descarga de archivos via MinIO |
| Publicar eventos | Redis Streams → notification-service escucha |

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

## Frontend — La interfaz

El frontend es una aplicación React con tres modos de vista:

| Vista | Descripción |
|---|---|
| **Kanban** | Columnas por estado: Abierto / En progreso / En revisión / Cerrado |
| **Lista** | Tabla filtrable de todos los tickets |
| **Dashboard** | Estadísticas y conteos rápidos |

El panel de administración incluye:
- Gestión de usuarios y permisos
- Configuración de workflows y transiciones de estado
- Revisión de clasificaciones (borradores del agente IA)
- Catálogo de categorías sincronizables con ticket-classification

---

## Backend — La API

Spring Boot expone una API REST bajo `/api/v1/` con autenticación JWT.

Grupos de endpoints principales:

| Grupo | Propósito |
|---|---|
| `/api/v1/tickets` | CRUD de tickets |
| `/api/v1/comments` | Comentarios por ticket |
| `/api/v1/users` | Gestión de usuarios |
| `/api/v1/workflows` | Estados y transiciones |
| `/api/v1/classifications` | Catálogo de categorías para el agente |

### Comunicación interna con ticket-classification

El agente de ticket-classification llama al backend para registrar tickets:

```
POST /api/v1/tickets
Header: X-Internal-Api-Key: <INTERNAL_API_KEY>
```

La key debe coincidir con `TICKET_MGMT_API_KEY` del lado del clasificador.

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

## Redis — Caché y sesiones

`ticket-system-redis` cumple dos roles:

| Rol | Detalle |
|---|---|
| Caché de sesiones | JWT sessions y datos frecuentes |
| Redis Streams | Publica eventos de tickets para notification-service |

> Este Redis es **distinto** del `classifier-redis` de ticket-classification.
> Ambos corren en paralelo con puertos diferentes (6380 vs 6379).

Ejemplo de evento publicado en el stream `ticket.events`:

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

## MinIO — Adjuntos

MinIO almacena los archivos adjuntos subidos a tickets. El backend
guarda solo la referencia (nombre + tipo) en PostgreSQL.

| Recurso | URL |
|---|---|
| API S3 | http://localhost:9002 |
| Consola web | http://localhost:9003 |
| Credenciales dev | minioadmin / minioadmin123 |

---

## Flyway — Migraciones

Flyway aplica las migraciones de BD automáticamente al arrancar el backend.
Las migraciones viven en `backend/src/main/resources/db/migration/`.

```
Backend arranca
    └─ Flyway verifica versión actual de ticket-db
    └─ Aplica migraciones pendientes (V1__, V2__, ...)
    └─ Backend inicia normalmente
```

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

## Puertos del stack

| Servicio | Puerto host | Rol |
|---|---|---|
| Frontend React | 8090 | Interfaz web principal |
| Backend Spring Boot | 8080 | API REST |
| ticket-db (PostgreSQL) | 5433 | Base de datos |
| ticket-system-redis | 6380 | Caché y streams |
| MinIO API | 9002 | Almacenamiento S3 |
| MinIO Consola | 9003 | Administración MinIO |

---

*ticket-management · Ticket Management for Dummies · Mayo 2026*
