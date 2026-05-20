# 02 — Arquitectura Técnica
## ticket-management · Referencia de servicios, redes, API y configuración
**Mayo 2026**

---

## Índice

1. [Servicios del stack](#1-servicios-del-stack)
2. [Redes](#2-redes)
3. [Variables de entorno](#3-variables-de-entorno)
4. [API REST — endpoints principales](#4-api-rest--endpoints-principales)
5. [Autenticación](#5-autenticación)
6. [Base de datos](#6-base-de-datos)
7. [Redis Streams — publicación de eventos](#7-redis-streams--publicación-de-eventos)
8. [MinIO](#8-minio)
9. [Arquitectura de capas — backend](#9-arquitectura-de-capas--backend)
10. [Dependencias Maven — decisiones clave](#10-dependencias-maven--decisiones-clave)

---

## 1. Servicios del stack

| Contenedor | Imagen | Puerto host | Puerto interno | Rol |
|---|---|---|---|---|
| `ticket-system-backend` | build local (Spring Boot) | 8080 | 8080 | API REST principal |
| `ticket-system-frontend` | build local (React) | 8090 | — | Interfaz web |
| `ticket-db` | postgres:15 | 5433 | 5432 | Base de datos principal |
| `ticket-system-redis` | redis:7 | 6380 | 6379 | Caché + Redis Streams |
| `minio` | minio/minio | 9002/9003 | 9000/9001 | Almacenamiento adjuntos |
| `ticket-flyway` | flyway | — | — | Migraciones BD (one-shot) |

### Orden de arranque (health checks)

```
ticket-db (healthy)
     └─ ticket-system-redis (healthy)
           └─ minio (healthy)
                 └─ ticket-flyway (completes)
                       └─ ticket-system-backend (starts)
                             └─ ticket-system-frontend (starts)
```

---

## 2. Redes

```yaml
networks:
  ticket-network:
    driver: bridge              # Red interna del repo
  ticket-management-network:
    external: true              # Red compartida del ecosistema
```

| Red | Tipo | Quién la usa |
|---|---|---|
| `ticket-network` | Interna | Comunicación entre servicios del repo |
| `ticket-management-network` | Externa | ticket-classification, notification-service, til-n8n |

`ticket-system-backend` y `ticket-system-redis` están en ambas redes — los
servicios externos del ecosistema los alcanzan por `ticket-management-network`.

---

## 3. Variables de entorno

### `.env.example` — referencias

```bash
# ── Base de datos ─────────────────────────────────────────────────
SPRING_DATASOURCE_URL=jdbc:postgresql://ticket-db:5432/tickets_db
SPRING_DATASOURCE_USERNAME=ticket_admin
SPRING_DATASOURCE_PASSWORD=ticket_password_123

# ── Redis ──────────────────────────────────────────────────────────
REDIS_HOST=ticket-system-redis
REDIS_PORT=6379
REDIS_PASSWORD=

# ── MinIO ──────────────────────────────────────────────────────────
MINIO_ENDPOINT=http://minio:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin_password_123
MINIO_BUCKET=ticket-attachments

# ── JWT ────────────────────────────────────────────────────────────
JWT_SECRET=jwt-secret-key-reemplazar
JWT_EXPIRATION=86400000

# ── CORS ───────────────────────────────────────────────────────────
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173,http://localhost:8090

# ── Integración interna ────────────────────────────────────────────
INTERNAL_API_KEY=reemplazar_con_secreto_real

# ── Email (futuro uso) ─────────────────────────────────────────────
MAIL_HOST=smtp.gmail.com
```

### Variables críticas para integración con ticket-classification

| Variable | Uso |
|---|---|
| `INTERNAL_API_KEY` | Autenticación de llamadas del agente IA (header `X-Internal-Api-Key`) |
| `JWT_SECRET` | Firma de tokens JWT — cambiar en producción |

---

## 4. API REST — endpoints principales

Base URL: `http://localhost:8080/api/v1/`

### Tickets

| Método | Endpoint | Descripción | Auth |
|---|---|---|---|
| `GET` | `/tickets` | Listar tickets (filtros opcionales) | JWT |
| `POST` | `/tickets` | Crear ticket | JWT o API Key interna |
| `GET` | `/tickets/{id}` | Obtener ticket por ID | JWT |
| `PUT` | `/tickets/{id}` | Actualizar ticket | JWT |
| `DELETE` | `/tickets/{id}` | Eliminar ticket | JWT (admin) |
| `PATCH` | `/tickets/{id}/status` | Cambiar estado | JWT |

### Comentarios

| Método | Endpoint | Descripción | Auth |
|---|---|---|---|
| `GET` | `/tickets/{id}/comments` | Listar comentarios de un ticket | JWT |
| `POST` | `/tickets/{id}/comments` | Agregar comentario | JWT |

### Usuarios y Auth

| Método | Endpoint | Descripción | Auth |
|---|---|---|---|
| `POST` | `/auth/login` | Login — devuelve JWT | Público |
| `GET` | `/users` | Listar usuarios | JWT (admin) |
| `POST` | `/users` | Crear usuario | JWT (admin) |

### Clasificaciones (catálogo para el agente)

| Método | Endpoint | Descripción | Auth |
|---|---|---|---|
| `GET` | `/classifications` | Listar dominios y categorías | API Key o JWT |

### Actuator (métricas)

| Endpoint | Descripción |
|---|---|
| `/api/v1/actuator/health` | Estado del servicio |
| `/api/v1/actuator/prometheus` | Métricas para Prometheus |

---

## 5. Autenticación

### JWT — usuarios del portal

```
POST /api/v1/auth/login
Body: { "username": "admin", "password": "..." }
Response: { "token": "eyJ..." }

# Usar en requests posteriores:
Authorization: Bearer eyJ...
```

### API Key interna — comunicación entre servicios

El agente `langchain-agent` llama a este backend para registrar tickets.
La autenticación usa un header:

```
X-Internal-Api-Key: <INTERNAL_API_KEY>
```

El valor debe coincidir en ambos repos:
- `ticket-management/.env`: `INTERNAL_API_KEY=el_secreto`
- `ticket-classification/.env`: `TICKET_MGMT_API_KEY=el_secreto`

---

## 6. Base de datos

**ticket-db** — PostgreSQL 15, puerto 5433 desde el host.

Tablas principales (gestionadas por Flyway):

| Tabla | Contenido |
|---|---|
| `tickets` | Tickets con estado, categoría, prioridad, solicitante |
| `comments` | Comentarios asociados a cada ticket |
| `users` | Usuarios del sistema con roles |
| `workflows` | Definición de estados y transiciones |
| `classifications` | Catálogo de dominios y categorías |

### Conexión directa (herramientas externas)

```
Host:     localhost
Port:     5433
Database: tickets_db
User:     ticket_admin
Password: ticket_password_123 (dev)
```

### Flyway

Las migraciones viven en `backend/src/main/resources/db/migration/`.
Flyway las aplica al arrancar el backend. Nunca modificar archivos
`V__*.sql` ya aplicados.

```bash
# Ver estado de migraciones (desde el host, con Java)
cd ~/stack_ticket/ticket-management
mvn flyway:info
```

---

## 7. Redis Streams — publicación de eventos

El backend publica eventos en Redis cuando cambia el estado de un ticket.
`notification-service` los consume.

```
Stream:  ticket.events
Eventos: ticket_created | ticket_updated | ticket_closed | comment_added
```

Formato de cada mensaje en el stream:
```json
{
  "tipo": "ticket_created",
  "ticket_id": "123",
  "titulo": "No puedo acceder al sistema",
  "estado": "ABIERTO",
  "solicitante_email": "usuario@empresa.com",
  "solicitante_nombre": "Juan García",
  "dominio": "IT",
  "categoria": "acceso"
}
```

---

## 8. MinIO

MinIO almacena archivos adjuntos a tickets. El backend guarda
en PostgreSQL solo el nombre del archivo y el tipo MIME.

| Parámetro | Valor (dev) |
|---|---|
| API S3 | http://localhost:9002 |
| Consola web | http://localhost:9003 |
| Access Key | minioadmin |
| Secret Key | minioadmin_password_123 |
| Bucket | `ticket-attachments` |

---

## 9. Arquitectura de capas — backend

El backend sigue **Clean Architecture**:

```
┌────────────────────────────────────────────┐
│  Infraestructura (Spring, JPA, Redis, S3)  │
│  Controllers, Repositories, Config         │
├────────────────────────────────────────────┤
│  Aplicación                                │
│  Use Cases, DTOs, Mappers                  │
├────────────────────────────────────────────┤
│  Dominio                                   │
│  Entities, Value Objects, Domain Services  │
│  (sin dependencias de Spring/JPA)          │
└────────────────────────────────────────────┘
```

**Regla:** Las capas internas (dominio, aplicación) nunca importan
clases de infraestructura.

---

## 10. Dependencias Maven — decisiones clave

### Gestión de versiones

La mayoría de versiones las gestiona el parent `spring-boot-starter-parent`.
Las excepciones son:

| Dependencia | Versión gestionada por |
|---|---|
| `spring-boot-*` | Spring Boot parent (3.2.0) |
| `flyway-core`, `flyway-postgresql` | Spring Boot parent |
| `postgresql` driver | Declarada explícita (42.7.1) |
| `querydsl-*` | Propiedad `${querydsl.version}` (5.0.0) |
| `jjwt-*` | Declarada explícita (0.12.3) |
| `testcontainers` | Propiedad `${testcontainers.version}` (1.19.3) |

### Dependencias de test

`spring-boot-starter-test` ya incluye JUnit 5 completo, Mockito y AssertJ —
**no declarar estas librerías por separado** para evitar conflictos de versión.
Solo se declaran explícitamente las que Spring Boot no gestiona:
`testcontainers`, `testcontainers:postgresql` y `rest-assured`.

---

*ticket-management · Arquitectura Técnica · Mayo 2026*
