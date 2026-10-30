# 02 — Arquitectura Técnica
## ticket-classification · Referencia de servicios, endpoints, datos y configuración
**Octubre 2026**

Fuente de verdad verificada contra código: `openspec/specs/` (`email-inbox`,
`inbox-processing`, `ticket-mgmt-integration`, `ticket-persistence`, `email-dedup`,
`ticket-classification-agent`, `llm-gateway`, `stack-deployment`).

---

## Índice

1. [Servicios y redes](#1-servicios-y-redes)
2. [Endpoints](#2-endpoints)
3. [Ciclo de vida de un correo](#3-ciclo-de-vida-de-un-correo)
4. [Datos: PostgreSQL, MinIO y Redis](#4-datos-postgresql-minio-y-redis)
5. [Integración con ticket-management](#5-integración-con-ticket-management)
6. [Gateway LLM](#6-gateway-llm)
7. [Variables de entorno](#7-variables-de-entorno)
8. [Arranque (`start.sh`)](#8-arranque-startsh)
9. [Observabilidad](#9-observabilidad)

---

## 1. Servicios y redes

| Contenedor | Puerto host | Redes | Rol |
|---|---|---|---|
| `langchain-agent` | `127.0.0.1:8001` | ticket-classification-network, ticket-management-network | Bandeja, workers de clasificación y entrega, barrido, retención, reportes de salud |
| `langchain-api` | `127.0.0.1:8000` | ticket-classification-network | Gateway LLM `/ask` con provider intercambiable y caché |
| `ollama` | `127.0.0.1:11434` | ticket-classification-network | LLM local `llama3.2:3b` (en Rocky no se usa) |
| `classifier-db` | `127.0.0.1:5434` | ticket-classification-network | PostgreSQL 15, base `classifier_db`, usuario `admin` |
| `classifier-redis` | `127.0.0.1:6381` | ticket-classification-network | Redis 7.2 con contraseña y AOF |
| `minio` | `127.0.0.1:9000` / `9001` | ticket-classification-network | Bucket privado `classifier-inbox` |
| `classifier-flyway` | — | ticket-classification-network | Migraciones `db/migrations/postgresql/` (V1–V3) |

Solo `langchain-agent` está en la red compartida: n8n le llega (`til-n8n` → `langchain-agent:8001`)
y él llega a ticket-management (`ticket-system-backend:8080`). Prometheus también está en
`ticket-classification-network` para leer las métricas.

---

## 2. Endpoints

### langchain-agent (:8001)

| Método y ruta | Autenticación | Uso |
|---|---|---|
| `GET /health` | — | Estado; incluye `agent_provider` y `ticket_management` (`ok`/`failing`) |
| `GET /metrics` | — | Métricas Prometheus |
| `POST /inbox/email` | `X-Channel-Key` | Recibe un correo de n8n (ver §3) |
| `GET /inbox/channel` | `X-Channel-Key` | Datos del canal y su kickoff (`ingestSince`) para n8n |
| `POST /inbox/channel-errors` | `X-Channel-Key` | n8n reporta errores del buzón (`{error}`) → `204` |
| `GET /internal/inbox/held[?tenantId=]` | `X-Service-Key` | Lista mensajes retenidos |
| `POST /internal/inbox/messages/{id}/redeliver` | `X-Service-Key` | Reentrega un retenido → `204` |
| `POST /internal/inbox/messages/{id}/discard` | `X-Service-Key` | Descarta un retenido → `204` |
| `DELETE /internal/tenants/{id}/inbox` | `X-Service-Key` | Borra todo lo del cliente (lo llama ticket-management al eliminar un tenant) |

`X-Service-Key` = `CLASSIFIER_SERVICE_KEY` (la misma llave que el clasificador usa contra
ticket-management). `/process` y `/status/{job_id}` **ya no existen**.

### langchain-api (:8000)

| Método y ruta | Autenticación | Uso |
|---|---|---|
| `GET /health` | — | `default_provider`, providers configurados, Redis |
| `POST /ask` | `X-Api-Key` = `LANGCHAIN_INTERNAL_API_KEY` | `{prompt, system?, provider?}` → `{provider, model, response, cached}` |
| `GET /metrics` | — | Métricas Prometheus |

---

## 3. Ciclo de vida de un correo

```
POST /inbox/email ─▶ RECEIVED ─▶ CLASSIFYING ─▶ READY ─▶ DELIVERED
                                                   └──▶ HELD (necesita una persona)
```

**Recepción** — orden de las comprobaciones y respuesta a n8n:

| Comprobación | Respuesta |
|---|---|
| Falta `X-Channel-Key` o ticket-management responde `404` a la resolución | `401` |
| Canal o cliente no `ACTIVE` | `423` |
| ticket-management no responde y no hay resolución en caché (60 s) | `503` |
| Más de 120 correos por minuto en el canal | `429` |
| Cuerpo > 25 MB / archivo > 10 MB / más de 50 archivos | `413` |
| JSON inválido, sin `externalId` o `from.address`, base64 inválido | `400` |
| `receivedAt` anterior al kickoff del canal | `200 BEFORE_KICKOFF` (no guarda) |
| Ya existe `(channel_id, external_id)` | `200 DUPLICATE` |
| Guardado en MinIO y PostgreSQL | `202 RECEIVED` (y se encola en `inbox-work`) |
| Falla el almacenamiento | `500` (no queda nada parcial; n8n reintenta) |

**Nuevo o respuesta (`kind`)**, en este orden:
1. `REPLY` si la conversación (`thread_key`) ya tiene un ticket entregado; si el primer correo de la
   conversación aún no se entrega, la respuesta espera.
2. `REPLY` si el asunto trae `#<8 hex>` de exactamente un ticket del canal.
3. `REPLY` si `In-Reply-To` o `References` citan un correo del canal que ya es parte de un ticket
   (Outlook cambia la conversación cuando editan el asunto), **solo** si ese ticket sigue abierto
   (`GET /internal/channel-tickets/{id}/state` de ticket-management) y el correo citado tiene 30 días
   o menos (`HEADER_LINK_MAX_DAYS`); ante la duda, `NEW`.
4. Si no, `NEW`.

**Imágenes en línea**: < 10 240 bytes, o repetidas del mismo remitente (3.ª vez en 90 días) =
firma, no se guardan; las demás se guardan con su `contentId`.

**Clasificación** (`NEW`): grafo LangGraph con el catálogo actual del cliente (área =
Clasificación del tenant, categorías). Un intento falla si el LLM falla o no responde
parseable; reintentos a 1, 5 y 15 min (4 intentos) y luego `READY` con `fallback=true` (sin
área, `requiereValidacion=true`). Para `REPLY`: evaluación `{relevante, razon, resumen, intencion}` en
una sola llamada; `intencion` = `pide_informacion`, `entrega_informacion`, `confirma_solucion`,
`nueva_solicitud` o `comentario` (lo demás, null) y `resumen` viaja aunque no sea relevante. Con eso
ticket-management sugiere o hace el movimiento del ticket (gestión por correo).

**¿Es ticket?** (`NEW`, `TRIAGE_ENABLED`): antes de entregar se decide el tipo (`solicitud`,
`seguimiento`, `paso`, `para_conocimiento`, `no_requiere_accion`, `dudoso`) con reglas fijas (avisos
propios, invitaciones, recall, automáticos, masivos) y luego el LLM con el contexto del cliente; viaja
en `data.triage = {tipo, accion, motivo, confianza, regla, dueno}` (`dueno` solo en un seguimiento del
equipo: quien lo mandó). Un reintento de entrega reusa la decisión guardada; si falla, `triage` es null
y el ticket se crea como siempre. Si se aplica (archivar, Borradores, asignar al dueño) lo decide el
interruptor del cliente en ticket-management.

**Entrega** a ticket-management: `NEW` → `POST /internal/channel-tickets`; `REPLY` →
`POST /internal/channel-tickets/{ticketId}/replies` (multipart: parte `data` JSON + `files`). Con cada
correo viajan los destinatarios (`to`, `cc`), el texto propio sin la cita (`ownText`) y, en las
respuestas, `receivedAt` e `intencion`: alimentan el hilo del correo del detalle.

| Respuesta | Resultado |
|---|---|
| `200`/`201` | `DELIVERED` (guarda `ticket_id`) |
| `423`, `5xx`, red, timeout, error leyendo MinIO | Reintento 1, 2, 5, 10, 30 min y luego cada 30 min, **sin límite** |
| `401` | Reintento igual; `/health` → `"ticket_management":"failing"` |
| `404` | `HELD` "channel or ticket not found" |
| `400`, `413`, `415` | `HELD` "`<status>`: `<mensaje>`" |

ticket-management es idempotente por `(canal, externalId)`: repetir una entrega nunca duplica.

**Barrido** cada minuto: reencola `RECEIVED`/`CLASSIFYING` atascados > 10 min y los
`RECEIVED`/`READY` cuyo siguiente intento ya venció; funciona aunque Redis no esté.

**Salud del canal**: cada 5 min y al haber un error, `POST /internal/channels/{id}/status`
`{lastMessageAt, lastErrorAt, lastError, pendingCount, failedCount}` → "📨 Canales".

**Retención** (cada hora): a los `CLASSIFIER_RETENTION_DAYS` (30) días de entregado se borran
cuerpo y archivos (`body_purged_at`); la fila queda. `HELD` se conserva completo hasta que un
operador lo resuelve.

---

## 4. Datos: PostgreSQL, MinIO y Redis

| Tabla (`classifier_db`) | Contenido |
|---|---|
| `inbox_messages` | Un correo por fila: canal, tenant, `kind`, `state`, `classification`, `reply_evaluation` (con `intencion`), `triage` (decisión ¿es ticket?), `envelope` (cc, encabezados, `conversationIndex`…), texto propio (`unique_body_*`), `fallback`, `ticket_id`, intentos, `last_error`, `held_reason`, fechas. Único `(channel_id, external_id)` |
| `inbox_attachments` | Archivos guardados: nombre, tipo, tamaño, `sha256`, `inline`, `content_id`, `storage_key` |
| `agent_runs` | Un registro por intento de clasificación/evaluación (`result` JSONB, `duration_ms`, `provider`) |
| `channel_errors` | Errores por canal (buzón, clasificación, entrega) |
| `sender_image_fingerprints` | Huellas de imágenes por remitente (firmas) |

MinIO `classifier-inbox`: `t<tenantId>/<channelId>/<messageId>/<uuid>`.

Redis `classifier-redis` (con contraseña): stream `inbox-work` (grupo `classifier-workers`,
`CLASSIFIER_CONCURRENCY` workers, por defecto 2) y caché de respuestas del LLM (1 h). Si se
pierde, el barrido reconstruye el trabajo desde la base.

---

## 5. Integración con ticket-management

Base `TICKET_MGMT_API_URL` (`http://ticket-system-backend:8080/api/v1`), cabecera
`X-Service-Key: CLASSIFIER_SERVICE_KEY`, timeout 30 s:

| Llamada | Para qué |
|---|---|
| `POST /internal/channels/resolve` (con el `X-Channel-Key`) | Tenant, canal, estados, kickoff y catálogo |
| `POST /internal/channels/{id}/status` | Reporte de salud del canal |
| `POST /internal/channel-tickets` | Crear ticket (idempotente) |
| `POST /internal/channel-tickets/{ticketId}/replies` | Agregar respuesta |
| `GET /internal/channel-tickets/{ticketId}/state?channelId=` | ¿El ticket sigue abierto? (antes de ligar por encabezados) |

Si `CLASSIFIER_SERVICE_KEY` no coincide con ticket-management: todo da `401`, `/inbox/email`
responde `503` y las entregas esperan en `READY` (nada se pierde).

---

## 6. Gateway LLM

`langchain-agent` llama a `{LANGCHAIN_API_URL}/ask` (`http://langchain-api:8000`) con
`X-Api-Key`, timeouts `LLM_TIMEOUT` (clasificación) y `ENRICH_LLM_TIMEOUT` (respuestas, máx.
120 s). El provider por defecto es `MODEL_PROVIDER` (= `AGENT_PROVIDER` del `.env`); en Rocky
`oci` con `OCI_MODEL=openai.gpt-oss-20b`. Las respuestas exitosas se cachean 1 h por prompt.

---

## 7. Variables de entorno

`.env` en `~/stack_ticket/ticket-classification` (nunca en git). `langchain-agent` se crea con
`podman run --env-file .env` (§8): **solo** lo que está en `.env` le llega.

| Variable | Uso |
|---|---|
| `AGENT_PROVIDER` | Provider del LLM (Rocky: `oci`); compose la pasa a langchain-api como `MODEL_PROVIDER` |
| `OCI_MODEL`, `OCI_SERVICE_ENDPOINT`, `OCI_COMPARTMENT_ID`, `OCI_TEMPERATURE`, `OCI_MAX_TOKENS` | OCI GenAI |
| `OLLAMA_BASE_URL`, `OLLAMA_MODEL`, `OPENAI_*`, `ANTHROPIC_*`, `GEMINI_*` | Otros providers |
| `LANGCHAIN_INTERNAL_API_KEY` | Llave de `/ask` (obligatoria; sin ella no arrancan) |
| `TICKET_MGMT_API_URL` | URL de ticket-management (`start.sh` la fija) |
| `CLASSIFIER_SERVICE_KEY` | Llave mutua con ticket-management (obligatoria) |
| `CLASSIFIER_CONCURRENCY` | Workers de clasificación/entrega (2) |
| `CLASSIFIER_RETENTION_DAYS` | Días que se guardan cuerpos y archivos tras entregar (30) |
| `AGENT_MAX_ITERATIONS`, `VALIDATION_ENABLED`, `FUZZY_THRESHOLD`, `MIN_CONFIDENCE` | Comportamiento del grafo de clasificación |
| `LLM_TIMEOUT`, `ENRICH_LLM_TIMEOUT` | Timeouts de las llamadas al LLM |
| `POSTGRES_*`, `REDIS_*`, `MINIO_*` | Conexiones |
| `AGENT_MOCK_CLASSIFY`, `MOCK_*`, `AGENT_DOMAINS`, `CATEGORIES_*` | Solo modo mock (pruebas sin LLM) |
| `LANGCHAIN_TRACING_V2`, `LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT` | Trazas en LangSmith (opcional) |
| `REGISTRY_BASE`, `IMAGE_TAG` | Nombre de las imágenes |

---

## 8. Arranque (`start.sh`)

1. Crea `.env` si falta y **se detiene** si `CLASSIFIER_SERVICE_KEY` o
   `LANGCHAIN_INTERNAL_API_KEY` están vacías o con el valor de ejemplo.
2. Fija `TICKET_MGMT_API_URL`; crea las redes si faltan (sin subred fija).
3. `podman-compose down` + `up -d`.
4. Recrea `langchain-agent` con `podman run --env-file .env` (parche para podman-compose 1.6.0,
   que montaba el código encima de la imagen): por eso el bloque `environment:` del compose no
   aplica a `langchain-agent` y `podman-compose restart` no toma cambios del `.env`.

---

## 9. Observabilidad

- Logs JSON: `podman logs langchain-agent` (eventos `inbox_received`, `inbox_classified`,
  `inbox_classify_retry`, `inbox_classify_fallback`, `inbox_delivered`,
  `inbox_delivery_retry`, `inbox_held`, `held_redeliver`, `held_discard`, `retention_purged`).
  `inbox_worker_error` cada ~10 s sin correos es ruido conocido (lectura bloqueante de Redis).
- Métricas: Prometheus lee `langchain-agent:8001/metrics` y `langchain-api:8000/metrics`.
- Grafana → Explore → Loki: `{container="langchain-agent"}`.

---

*ticket-classification · Arquitectura Técnica · Octubre 2026*
