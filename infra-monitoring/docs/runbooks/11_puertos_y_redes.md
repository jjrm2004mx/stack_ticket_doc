# 11 — Puertos y Redes del Ecosistema
## infra-monitoring · Referencia completa de contenedores, puertos y topología de red
**Mayo 2026**

---

## Índice

1. [Tabla maestra de puertos](#1-tabla-maestra-de-puertos)
2. [Redes del ecosistema](#2-redes-del-ecosistema)
3. [Pertenencia de contenedores a redes](#3-pertenencia-de-contenedores-a-redes)
4. [Comunicaciones internas entre servicios](#4-comunicaciones-internas-entre-servicios)
5. [Diagrama de topología](#5-diagrama-de-topología)
6. [Notas y conflictos conocidos](#6-notas-y-conflictos-conocidos)

---

## 1. Tabla maestra de puertos

Todos los contenedores del ecosistema con su puerto en el host, puerto interno y URL de acceso local.

### ticket-management

| Contenedor | Puerto host | Puerto interno | Protocolo | URL localhost | Descripción |
|---|---|---|---|---|---|
| `ticket-system-frontend` | 8090 | 80 | HTTP | http://localhost:8090 | Portal web de tickets (Nginx + Vite) |
| `ticket-system-backend` | 8080 | 8080 | HTTP | http://localhost:8080/api/v1 | API REST Spring Boot |
| `ticket-db` | 5433 | 5432 | TCP | localhost:5433 | PostgreSQL — `tickets_db` |
| `ticket-system-redis` | 6380 | 6379 | TCP | localhost:6380 | Redis — caché y sesiones |
| `ticket-system-minio` | 9002 | 9000 | HTTP | http://localhost:9002 | MinIO API — adjuntos de tickets |
| `ticket-system-minio` | 9003 | 9001 | HTTP | http://localhost:9003 | MinIO consola web |
| `ticket-flyway` | — | — | — | — | Migraciones one-shot, sin puerto |

### ticket-classification

| Contenedor | Puerto host | Puerto interno | Protocolo | URL localhost | Descripción |
|---|---|---|---|---|---|
| `langchain-agent` | 8001 | 8001 | HTTP | http://localhost:8001 | Agente LangGraph — punto de entrada `/process` |
| `langchain-api` | 8000 | 8000 | HTTP | http://localhost:8000 | Gateway LLM — `/ask` con provider intercambiable |
| `ollama` | 11434 | 11434 | HTTP | http://localhost:11434 | Modelos LLM locales (llama3.2:3b por defecto) |
| `classifier-db` | 5434 | 5432 | TCP | localhost:5434 | PostgreSQL — `classifier_db` |
| `classifier-redis` | 6381 | 6379 | TCP | localhost:6381 | Redis — caché LLM (TTL 1h) + estado agente |
| `minio` | 9000 | 9000 | HTTP | http://localhost:9000 | MinIO API — adjuntos de correos |
| `minio` | 9001 | 9001 | HTTP | http://localhost:9001 | MinIO consola web |
| `classifier-flyway` | — | — | — | — | Migraciones one-shot, sin puerto |

### notification-service

| Contenedor | Puerto host | Puerto interno | Protocolo | URL localhost | Descripción |
|---|---|---|---|---|---|
| `notification-service` | 8081 | 8081 | HTTP | http://localhost:8081 | API REST Spring Boot — envío de notificaciones |
| `notification-db` | 5435 | 5432 | TCP | localhost:5435 | PostgreSQL — `notifications_db` |

### ticket-agent

| Contenedor | Puerto host | Puerto interno | Protocolo | URL localhost | Descripción |
|---|---|---|---|---|---|
| `ticket-agent` | 8002 | 8002 | HTTP | http://localhost:8002 | RAG API — `POST /api/v1/query`, FAISS + ONNX + LangGraph |

### ticket-ingestion-light

| Contenedor | Puerto host | Puerto interno | Protocolo | URL localhost | Descripción |
|---|---|---|---|---|---|
| `til-n8n` | 5679 | 5678 | HTTP | http://localhost:5679 | n8n — editor y webhook receiver |

### infra-monitoring

| Contenedor | Puerto host | Puerto interno | Protocolo | URL localhost | Descripción |
|---|---|---|---|---|---|
| `prometheus` | 9090 | 9090 | HTTP | http://localhost:9090 | Métricas — scrape de todo el ecosistema |
| `grafana` | 3000 | 3000 | HTTP | http://localhost:3000 | Dashboards (admin / admin) |
| `loki` | 3100 | 3100 | HTTP | http://localhost:3100 | Agregación de logs |
| `node-exporter` | 9100 | 9100 | HTTP | http://localhost:9100 | Métricas del host (CPU, RAM, disco) |
| `promtail` | — | — | — | — | Recolector journald → Loki, sin puerto expuesto |

---

## 2. Redes del ecosistema

### Redes externas (compartidas entre stacks)

Creadas por `infra-monitoring/startup.sh` con subnets fijas para evitar colisiones.

| Red | Subnet | Propósito |
|---|---|---|
| `ticket-management-network` | 10.89.1.0/24 | Bus principal — conecta ticket-management, classification, notification y n8n |
| `ticket-classification-network` | 10.89.2.0/24 | Red interna de clasificación — LLM, DB, Redis, MinIO del clasificador |
| `observability-network` | 10.89.3.0/24 | Monitoreo — Prometheus, Grafana, Loki, Promtail, node-exporter |

### Redes internas (locales a cada stack)

Creadas por el propio `docker-compose.yml` de cada proyecto. Solo conectan contenedores del mismo stack.

| Red | Stack | Contenedores |
|---|---|---|
| `ticket-network` | ticket-management | ticket-system-frontend, ticket-system-backend, ticket-db, ticket-system-redis, ticket-system-minio, ticket-flyway |
| `notification-network` | notification-service | notification-service, notification-db |
| `stack` | ticket-ingestion-light | til-n8n |
| *(ninguna)* | ticket-agent | `ticket-agent` solo necesita la red compartida |

---

## 3. Pertenencia de contenedores a redes

Vista por red compartida — qué contenedores pueden comunicarse entre sí.

### ticket-management-network (10.89.1.0/24)

| Contenedor | Stack de origen |
|---|---|
| `ticket-system-backend` | ticket-management |
| `ticket-db` | ticket-management |
| `ticket-system-redis` | ticket-management |
| `langchain-agent` | ticket-classification |
| `classifier-db` | ticket-classification |
| `notification-service` | notification-service |
| `notification-db` | notification-service |
| `til-n8n` | ticket-ingestion-light |
| `ticket-agent` | ticket-agent |
| `prometheus` | infra-monitoring |

### ticket-classification-network (10.89.2.0/24)

| Contenedor | Stack de origen |
|---|---|
| `langchain-agent` | ticket-classification |
| `langchain-api` | ticket-classification |
| `ollama` | ticket-classification |
| `classifier-db` | ticket-classification |
| `classifier-redis` | ticket-classification |
| `minio` | ticket-classification |
| `classifier-flyway` | ticket-classification |
| `prometheus` | infra-monitoring |

### observability-network (10.89.3.0/24)

| Contenedor | Stack de origen |
|---|---|
| `prometheus` | infra-monitoring |
| `grafana` | infra-monitoring |
| `loki` | infra-monitoring |
| `promtail` | infra-monitoring |
| `node-exporter` | infra-monitoring |

---

## 4. Comunicaciones internas entre servicios

URLs que usan los contenedores para hablarse entre sí (dentro de las redes Podman).
Estas URLs nunca pasan por el host — se resuelven por nombre de contenedor.

### Flujo principal: correo → ticket

```
til-n8n
  └─► POST http://langchain-agent:8001/process
        (ticket-management-network)
        └─► GET  http://ticket-system-backend:8080/api/v1/catalogo
        │         (ticket-management-network)
        └─► POST http://langchain-api:8000/ask
        │         (ticket-classification-network)
        │         └─► http://ollama:11434  (ticket-classification-network)
        └─► POST http://ticket-system-backend:8080/api/v1/tickets
                  (ticket-management-network)
```

### Flujo de notificaciones: ticket → correo

```
ticket-system-backend
  └─► POST http://notification-service:8081/...
        (ticket-management-network)
        └─► POST http://til-n8n:5678/webhook/notification-webhook
                  (ticket-management-network)
                  └─► Gmail API  (internet)
```

### Bases de datos e infraestructura

| Desde | Hacia | URL interna | Red |
|---|---|---|---|
| `ticket-system-backend` | `ticket-db` | `ticket-db:5432` | ticket-network |
| `ticket-system-backend` | `ticket-system-redis` | `ticket-system-redis:6379` | ticket-network |
| `ticket-system-backend` | `ticket-system-minio` | `http://ticket-system-minio:9000` | ticket-network |
| `notification-service` | `notification-db` | `notification-db:5432` | notification-network |
| `notification-service` | `ticket-system-redis` | `ticket-system-redis:6379` | ticket-management-network |
| `langchain-agent` | `classifier-db` | `classifier-db:5432` | ticket-classification-network |
| `langchain-agent` | `classifier-redis` | `classifier-redis:6379` | ticket-classification-network |
| `langchain-agent` | `minio` | `minio:9000` | ticket-classification-network |
| `langchain-api` | `classifier-redis` | `classifier-redis:6379` | ticket-classification-network |
| `langchain-api` | `ollama` | `http://ollama:11434` | ticket-classification-network |

### Scrape de métricas (Prometheus)

| Prometheus scrape | URL interna | Red |
|---|---|---|
| Prometheus mismo | `localhost:9090` | observability-network |
| `ticket-system-backend` | `ticket-system-backend:8080` | ticket-management-network |
| `langchain-agent` | `langchain-agent:8001` | ticket-classification-network |
| `node-exporter` | `node-exporter:9100` | observability-network |

---

## 5. Diagrama de topología

```
╔══════════════════════════════════════════════════════════════════╗
║  ticket-management-network  (10.89.1.0/24)                       ║
║                                                                  ║
║  ticket-system-backend:8080  ◄──────────────────────────────┐   ║
║  ticket-system-redis:6379    ◄──── notification-service      │   ║
║  ticket-db:5432              ◄──── (notification-network)    │   ║
║  ticket-system-minio:9000    ◄────┘                          │   ║
║                                                               │   ║
║  langchain-agent:8001 ◄──── til-n8n:5678 ◄────── Gmail       │   ║
║  (+ ticket-classification-network)   │                        │   ║
║                                      └── webhook ────────────┘   ║
║  classifier-db:5432                                              ║
║  notification-service:8081 ◄─── ticket-system-backend            ║
║  notification-db:5432                                            ║
║  til-n8n:5678                                                    ║
║  ticket-agent:8002  (RAG API — consultas externas por 8002)      ║
║  prometheus:9090                                                 ║
╚══════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════╗
║  ticket-classification-network  (10.89.2.0/24)                   ║
║                                                                  ║
║  langchain-agent:8001 ──► langchain-api:8000 ──► ollama:11434   ║
║  classifier-db:5432                                              ║
║  classifier-redis:6379                                           ║
║  minio:9000                                                      ║
║  prometheus:9090 (solo scrape)                                   ║
╚══════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════╗
║  observability-network  (10.89.3.0/24)                           ║
║                                                                  ║
║  prometheus:9090 ──► grafana:3000                                ║
║  loki:3100        ──► grafana:3000                               ║
║  promtail ──────────► loki:3100                                  ║
║  node-exporter:9100 (prometheus lo scrape desde ticket-mgmt-net) ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 6. Notas y conflictos conocidos

### Dos instancias de MinIO en el mismo host

El ecosistema tiene dos MinIO independientes, cada uno en un stack distinto:

| Contenedor | Puerto host API | Puerto host consola | Uso |
|---|---|---|---|
| `minio` (ticket-classification) | **9000** | **9001** | Adjuntos de correos entrantes |
| `ticket-system-minio` (ticket-management) | **9002** | **9003** | Adjuntos de tickets creados |

No hay conflicto de puertos en el host. No confundir sus credenciales — son instancias separadas.

### Dos instancias de Redis en el mismo host

| Contenedor | Puerto host | Uso |
|---|---|---|
| `ticket-system-redis` | 6380 | Caché de ticket-management y notification-service |
| `classifier-redis` | 6381 | Caché LLM (TTL 1h) + estado del agente LangGraph |

`notification-service` usa `ticket-system-redis` (no su propio Redis) a través de `ticket-management-network`.

### n8n: puerto externo vs interno

`til-n8n` escucha internamente en el puerto **5678** (convención n8n).
En el host se expone como **5679** para evitar colisión con otras instancias n8n.
Los demás contenedores que llaman a n8n deben usar `til-n8n:5678` (puerto interno).

### NOTIFICATION_PORTAL_URL — URL pública del portal

`notification-service` necesita la URL pública accesible desde el navegador para generar
el botón "Ver el estado de mi ticket" en los correos. Esta URL **no es** una comunicación
interna — es la URL que abre el usuario final.

Se inyecta a través de `~/stack_ticket/.env.deploy` (ver runbook `10_redes_y_acceso_lan.md`).
`infra-monitoring/startup.sh` la detecta automáticamente usando PowerShell para obtener
la IP LAN real de Windows (no la IP virtual de WSL).

---

### ticket-agent — acceso a Ollama desde el contenedor

`ticket-agent` está en `ticket-management-network` pero Ollama vive en
`ticket-classification-network`. Ambas redes no están conectadas directamente.

La solución es acceder a Ollama por el host WSL:
- `OLLAMA_BASE_URL=http://host.containers.internal:11434`

`host.containers.internal` resuelve a la IP del host Windows en Podman rootless WSL.
Si PROVIDER=oci, esta variable no se usa y el problema no aplica.

---

*infra-monitoring · Puertos y Redes · Mayo 2026*
