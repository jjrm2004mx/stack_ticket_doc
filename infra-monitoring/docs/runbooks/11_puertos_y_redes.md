# 11 — Puertos, Redes y Bases de Datos del Ecosistema
## infra-monitoring · Referencia completa de contenedores, puertos, redes, bases de datos y flujos
**Octubre 2026** — verificado en Rocky (TIJDCAGENTIA, `10.187.21.30`) el 2026-10-17

---

## Índice

1. [Tabla maestra de puertos](#1-tabla-maestra-de-puertos)
2. [Puertos que deben estar libres antes de levantar el stack](#2-puertos-que-deben-estar-libres-antes-de-levantar-el-stack)
3. [Redes del ecosistema](#3-redes-del-ecosistema)
4. [Pertenencia de contenedores a redes](#4-pertenencia-de-contenedores-a-redes)
5. [Bases de datos y almacenamiento](#5-bases-de-datos-y-almacenamiento)
6. [Comunicaciones entre servicios](#6-comunicaciones-entre-servicios)
7. [Notas y conflictos conocidos](#7-notas-y-conflictos-conocidos)

---

## 1. Tabla maestra de puertos

Puerto en el host (Rocky) → puerto dentro del contenedor. Todos publican **solo en
`127.0.0.1`** salvo el portal (8090), que publica en todas las interfaces: desde otra
máquina solo responde `http://10.187.21.30:8090`; el resto se usa con el túnel SSH,
también desde cualquier otra máquina de la oficina (`10_redes_y_acceso_lan.md` §7).

### ticket-management

| Contenedor | Puerto host | Puerto interno | URL en Rocky | Descripción |
|---|---|---|---|---|
| `ticket-system-frontend` | 8090 | 80 | http://localhost:8090 | Portal web (Nginx + Vite) |
| `ticket-system-backend` | 8080 | 8080 | http://localhost:8080/api/v1 | API REST Spring Boot |
| `ticket-db` | 5433 | 5432 | localhost:5433 | PostgreSQL — `tickets_db` |
| `ticket-system-redis` | 6380 | 6379 | localhost:6380 | Redis — caché y stream de eventos `ticket-events` |
| `ticket-system-minio` | 9002 / 9003 | 9000 / 9001 | http://localhost:9002 | MinIO — adjuntos de tickets (API / consola) |
| `ticket-flyway` | — | — | — | Migraciones one-shot |

### ticket-classification

| Contenedor | Puerto host | Puerto interno | URL en Rocky | Descripción |
|---|---|---|---|---|
| `langchain-agent` | 8001 | 8001 | http://localhost:8001 | Clasificador: bandeja `/inbox/*` y agente LangGraph |
| `langchain-api` | 8000 | 8000 | http://localhost:8000 | Gateway LLM `/ask` (proveedor intercambiable) |
| `ollama` | 11434 | 11434 | http://localhost:11434 | LLM local (`llama3.2:3b`); en Rocky no es el proveedor activo |
| `classifier-db` | 5434 | 5432 | localhost:5434 | PostgreSQL — `classifier_db` |
| `classifier-redis` | 6381 | 6379 | localhost:6381 | Redis — caché LLM y límites por canal |
| `minio` | 9000 / 9001 | 9000 / 9001 | http://localhost:9000 | MinIO — adjuntos de la bandeja (bucket `classifier-inbox`) |
| `classifier-flyway` | — | — | — | Migraciones one-shot |

### notification-service

| Contenedor | Puerto host | Puerto interno | URL en Rocky | Descripción |
|---|---|---|---|---|
| `notification-service` | 8081 | 8081 | http://localhost:8081 | API Spring Boot — avisos por correo |
| `notification-db` | 5435 | 5432 | localhost:5435 | PostgreSQL — `notifications_db` |

### ticket-agent (en `~/stack_agent`)

| Contenedor | Puerto host | Puerto interno | URL en Rocky | Descripción |
|---|---|---|---|---|
| `ticket-agent` | 8002 | 8002 | http://localhost:8002 | Asistente RAG — `POST /api/v1/query` |
| `loki-mcp-server` | 9011 (solo `127.0.0.1`) | 9011 | http://127.0.0.1:9011 | MCP de logs (Loki) para el asistente |
| `prometheus-mcp-server` | 9012 (solo `127.0.0.1`) | 9012 | http://127.0.0.1:9012 | MCP de métricas (Prometheus) para el asistente |

### support-console (en `~/stack_agent`)

| Contenedor | Puerto host | Puerto interno | URL en Rocky | Descripción |
|---|---|---|---|---|
| `sc-nginx` | 3001 (`SC_PORT`) | 80 | http://localhost:3001 | Consola de soporte — UI del asistente |

### ticket-ingestion-light

| Contenedor | Puerto host | Puerto interno | URL en Rocky | Descripción |
|---|---|---|---|---|
| `til-n8n` | 5679 | 5678 | http://localhost:5679 | n8n — ingesta por canal y webhooks de envío |

### infra-monitoring

| Contenedor | Puerto host | Puerto interno | URL en Rocky | Descripción |
|---|---|---|---|---|
| `prometheus` | 9090 | 9090 | http://localhost:9090 | Métricas |
| `grafana` | 3000 | 3000 | http://localhost:3000 | Dashboards |
| `loki` | 3100 | 3100 | http://localhost:3100 | Logs |
| `node-exporter` | 9100 | 9100 | http://localhost:9100 | Métricas del host |
| `promtail` | — | — | — | Recolector journald → Loki |

---

## 2. Puertos que deben estar libres antes de levantar el stack

Lista completa de puertos publicados en el host:

```
3000 3001 3100 5433 5434 5435 5679 6380 6381 8000 8001 8002 8080 8081 8090
9000 9001 9002 9003 9011 9012 9090 9100 11434
```

Comprobar en Rocky, con el stack abajo, que ninguno esté ocupado:

```bash
for p in 3000 3001 3100 5433 5434 5435 5679 6380 6381 8000 8001 8002 8080 8081 8090 \
         9000 9001 9002 9003 9011 9012 9090 9100 11434; do
  ss -ltn "sport = :$p" | grep -q LISTEN && echo "OCUPADO: $p"
done
```

Con el stack arriba, `podman ps --format "{{.Names}} {{.Ports}}"` muestra quién tiene cada
puerto (todos `127.0.0.1:` salvo `0.0.0.0:8090`). Único puerto que debe estar abierto en
firewalld para la LAN, además del 22 de SSH: 8090 (ver `10_redes_y_acceso_lan.md` §7).

---

## 3. Redes del ecosistema

Ninguna red tiene subred fija: Podman (netavark, rootless) asigna la primera libre de
`10.89.0.0/16` al crearla, y nada depende de la subred. Las de abajo son las que tiene
Rocky hoy.

### Redes externas (compartidas, las crea `startup.sh` si faltan)

| Red | Subnet en Rocky | Propósito |
|---|---|---|
| `ticket-management-network` | 10.89.0.0/24 | Bus entre stacks: solo servicios que se hablan de un stack a otro |
| `ticket-classification-network` | 10.89.2.0/24 | Clasificación: LLM, BD, Redis y MinIO del clasificador |
| `observability-network` | 10.89.3.0/24 | Monitoreo, asistente y servidores MCP |

### Redes internas (las crea el compose de cada stack)

| Red | Subnet en Rocky | Contenedores |
|---|---|---|
| `ticket-management_ticket-network` | 10.89.1.0/24 | frontend, backend, ticket-db, ticket-system-redis, ticket-system-minio, flyway |
| `support-console_default` | 10.89.4.0/24 | sc-nginx |
| `notification-service_notification-network` | 10.89.5.0/24 | notification-service, notification-db |
| `ticket-ingestion-light_stack` | 10.89.6.0/24 | til-n8n |

---

## 4. Pertenencia de contenedores a redes

| Contenedor | Redes |
|---|---|
| `ticket-system-frontend` | ticket-management_ticket-network |
| `ticket-system-backend` | ticket-management-network, ticket-management_ticket-network |
| `ticket-db` | ticket-management_ticket-network |
| `ticket-system-redis` | ticket-management-network, ticket-management_ticket-network |
| `ticket-system-minio` | ticket-management_ticket-network |
| `langchain-agent` | ticket-classification-network, ticket-management-network |
| `langchain-api`, `ollama`, `classifier-redis`, `minio` | ticket-classification-network |
| `classifier-db` | ticket-classification-network |
| `notification-service` | notification-service_notification-network, ticket-management-network |
| `notification-db` | notification-service_notification-network |
| `til-n8n` | ticket-ingestion-light_stack, ticket-management-network |
| `ticket-agent` | observability-network, ticket-management-network |
| `loki-mcp-server`, `prometheus-mcp-server` | observability-network |
| `sc-nginx` | support-console_default |
| `prometheus` | observability-network, ticket-classification-network, ticket-management-network |
| `grafana`, `loki`, `promtail`, `node-exporter` | observability-network |

Las bases de datos y los MCP no están en `ticket-management-network`: solo los usa su propio
stack. Para comprobarlo: `podman inspect <contenedor> --format "{{range \$k,\$v := .NetworkSettings.Networks}}{{\$k}} {{end}}"`.

---

## 5. Bases de datos y almacenamiento

Cada servicio guarda y recupera lo suyo; no hay base compartida.

| Almacén | Tecnología | Contenedor / ubicación | Dueño | Qué guarda |
|---|---|---|---|---|
| `tickets_db` | PostgreSQL 15 | `ticket-db` (5433) | ticket-management | Tenants, canales, usuarios, operadores de plataforma, permisos, workflows y estados, tickets y su historial, auditoría, outbox de eventos. Migraciones Flyway (`ticket-flyway`) |
| `classifier_db` | PostgreSQL 15 | `classifier-db` (5434) | ticket-classification | Bandeja persistente de correos (`inbox_messages`, `inbox_attachments`), corridas del agente, errores de canal, huellas de imágenes de firma. Flyway (`classifier-flyway`) |
| `notifications_db` | PostgreSQL 15 | `notification-db` (5435) | notification-service | Bitácora de envíos (`notification_logs`) y proveedor de envío por canal/tenant (`notification_provider_configs`) |
| Redis de management | Redis 7.2 | `ticket-system-redis` (6380) | ticket-management | Caché y stream `ticket-events` que consume notification-service |
| Redis del clasificador | Redis 7.2 | `classifier-redis` (6381) | ticket-classification | Caché LLM y límites por canal |
| MinIO de management | MinIO | `ticket-system-minio` (9002) | ticket-management | Bucket `ticket-attachments` |
| MinIO del clasificador | MinIO | `minio` (9000) | ticket-classification | Bucket `classifier-inbox` (`t<tenant>/<canal>/<mensaje>/…`) |
| Base de conocimiento del asistente | **Oracle Autonomous Database** (OCI, en la nube) | Fuera de Rocky; wallet en el host | ticket-agent | Fragmentos de documentos (`KB_CHUNKS`) y grafo de arquitectura. En Rocky `VECTOR_STORE=oracle`; la alternativa local es FAISS |
| n8n | SQLite | Volumen `ticket-ingestion-light_n8n_data` | ticket-ingestion-light | Workflows, credenciales de Microsoft Graph, llaves de canal y cursores de ingesta |
| Grafana / Prometheus / Loki | Volúmenes propios | infra-monitoring | infra-monitoring | Dashboards, series de métricas y logs |

Usuarios de las bases PostgreSQL: `ticket_admin` (tickets_db), `admin` (classifier_db),
`notif_admin` (notifications_db). Las contraseñas están en el `.env` de cada repo.

Consola rápida: `podman exec -it ticket-db psql -U ticket_admin -d tickets_db` (análogo
para las otras dos). Respaldo y restauración: `03_guia_operacional.md` §11.

---

## 6. Comunicaciones entre servicios

URLs internas (se resuelven por nombre de contenedor dentro de las redes Podman).

### Correo → ticket

```
Buzón Outlook del canal (Microsoft Graph, internet)
  ◄── til-n8n: copia "Canal … — ingesta" (cada 2 min, solo lectura)
        └─► GET/POST http://langchain-agent:8001/inbox/channel | /inbox/email   (X-Channel-Key)
              (ticket-management-network)
              langchain-agent
              ├─► http://ticket-system-backend:8080/api/v1/...   (resolver canal, crear ticket)
              ├─► http://langchain-api:8000/ask ─► OCI Generative AI (openai.gpt-oss-20b, internet)
              ├─► classifier-db:5432, classifier-redis:6379, minio:9000
```

### Ticket → aviso por correo

```
ticket-system-backend ─► event_outbox (tickets_db) ─► stream "ticket-events" en ticket-system-redis
  notification-service (consume el stream)
    ├─► aviso de un ticket con canal: POST http://til-n8n:5678/webhook/canal-<id>/send (X-Send-Secret)
    └─► sin canal (cuenta, contraseña):  POST http://til-n8n:5678/webhook/send-email
          til-n8n ─► Microsoft Graph sendMail (internet)
```

Si el canal no tiene fila activa en `notification_provider_configs`, el aviso **no se
envía** (falla cerrado). Detalle: ticket-ingestion-light `02_arquitectura_tecnica.md`.

### Asistente

```
Navegador ─► sc-nginx:3001 (consola de soporte) ─► http://localhost:8002 (ticket-agent)
Portal ─► ticket-system-backend /agent/query ─► http://ticket-agent:8002/api/v1/query
ticket-agent ─► Oracle ADB y OCI GenAI (internet)
             ─► loki-mcp-server:9011, prometheus-mcp-server:9012
```

### Métricas (Prometheus, `prometheus.yml`)

| Job | Destino |
|---|---|
| `ticket-management` | `ticket-system-backend:8080/api/v1/actuator/prometheus` |
| `notification-service` | `notification-service:8081/actuator/prometheus` |
| `langchain-agent` | `langchain-agent:8001/metrics` |
| `langchain-api` | `langchain-api:8000/metrics` |
| `ticket-agent` | `ticket-agent:8002/metrics` |
| `node-exporter` | `node-exporter:9100` |
| `prometheus` | `localhost:9090` |

Logs: promtail lee journald del host y etiqueta `container` y `service`
(Grafana → Explore → Loki → `{container="<nombre>"}`).

---

## 7. Notas y conflictos conocidos

### Subredes asignadas por Podman

`startup.sh` (y los `start.sh` de cada repo) crean una red compartida solo si falta y sin
`--subnet`; nunca borran redes ni contenedores. Por eso las subredes de Rocky no siguen un
orden (la compartida quedó en `10.89.0.0/24` y la interna de ticket-management en
`10.89.1.0/24`) y no importa. **No crear redes a mano con `--subnet`**: una subred fija dentro
de `10.89.0.0/16` puede chocar con las que Podman asigna solo.

El portal (nginx) toma su DNS del `resolv.conf` del contenedor al arrancar, así que tampoco
depende de ninguna IP.

### Dos MinIO y dos Redis

Son instancias independientes con credenciales distintas: `minio` (9000/9001) y
`ticket-system-minio` (9002/9003); `classifier-redis` (6381) y `ticket-system-redis`
(6380). Ambos Redis piden contraseña (`REDIS_PASSWORD` dentro del contenedor):
`podman exec ticket-system-redis sh -c 'redis-cli -a "$REDIS_PASSWORD" --no-auth-warning DBSIZE'`.

### n8n: puerto externo vs interno

`til-n8n` escucha en 5678; en el host es 5679. Los contenedores usan `til-n8n:5678`.

### Todo en loopback salvo el portal

Desde el cambio `aislar-redes-y-puertos` todos los `ports` de los compose llevan
`127.0.0.1:` excepto `ticket-system-frontend` (`8090:80`). Abrir un puerto en firewalld no
lo expone: habría que quitar además ese prefijo.

### URL pública del portal en los correos

`NOTIFICATION_PORTAL_URL` (notification-service) es la URL que abre el usuario final desde
el correo; en Rocky `http://10.187.21.30:8090`.

---

*infra-monitoring · Puertos, Redes y Bases de Datos · Octubre 2026*
