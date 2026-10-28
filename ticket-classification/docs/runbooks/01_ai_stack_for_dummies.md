# 01 — AI Stack for Dummies
## ticket-classification · Guía conceptual del clasificador
**Octubre 2026**

---

## La analogía: la oficina de recepción

Cada correo de un cliente llega a una **bandeja de recepción** (la bandeja del
clasificador). Ahí lo sellan y lo guardan antes de decir "recibido", para que nunca se pierda.
Después un **clasificador** lo lee, decide de qué área y categoría es y qué prioridad tiene, y
un **mensajero** lo lleva a ticket-management, que crea el ticket (o lo agrega como respuesta a
un ticket que ya existe). Si el mensajero no puede entregarlo, lo vuelve a intentar sin límite;
si ticket-management lo rechaza por algo que solo una persona puede arreglar, lo aparta
("retenido", `HELD`) para que un operador decida.

| Pieza de la analogía | Pieza del ecosistema |
|---|---|
| La bandeja de recepción | `POST /inbox/email` de `langchain-agent` + tablas `inbox_messages` / `inbox_attachments` (PostgreSQL `classifier_db`) y bucket MinIO `classifier-inbox` |
| El clasificador | El agente LangGraph de `langchain-agent`, que pregunta al LLM por medio de `langchain-api` (`/ask`) |
| El LLM | En Rocky: **OCI Generative AI, modelo `openai.gpt-oss-20b`**. Ollama corre pero no se usa |
| El mensajero | Los workers de `langchain-agent`, que entregan a ticket-management (`/internal/channel-tickets`) |
| La lista de pendientes | El stream Redis `inbox-work` (y el barrido cada minuto, que no depende de Redis) |

---

## Rol en el ecosistema

```
Buzón Outlook del canal
  └─ n8n (ticket-ingestion-light, copia "— ingesta" del canal)
        └─ POST /inbox/email  (X-Channel-Key)
              langchain-agent :8001  ── guarda ──▶ classifier-db + MinIO classifier-inbox
                 │  clasifica con el catálogo del cliente
                 ├─▶ langchain-api :8000 /ask ──▶ OCI GenAI (gpt-oss-20b)
                 └─▶ ticket-management /internal/channel-tickets  (crea el ticket)
                                       /internal/channel-tickets/{id}/replies  (respuesta)
```

- **No lee buzones**: eso lo hace n8n.
- **No guarda los tickets**: los guarda ticket-management. El clasificador guarda el correo
  hasta entregarlo (y su cuerpo y archivos 30 días más).
- **No decide a quién avisar**: eso es ticket-management y notification-service.

---

## Lo que pasa con cada correo

1. **Recepción** (`POST /inbox/email`, responde en milisegundos):
   - Con la llave del canal (`X-Channel-Key`) pregunta a ticket-management de qué cliente y
     canal es, y su catálogo de áreas y categorías.
   - `202 RECEIVED`: guardado. `200 DUPLICATE`: ya lo tenía. `200 BEFORE_KICKOFF`: llegó antes
     de la fecha "Cargar correos desde" del canal; no se guarda.
   - `401` llave desconocida, `423` canal o cliente pausado, `429` demasiados correos por
     minuto, `413` muy grande, `400` inválido, `503` ticket-management no respondió.
2. **¿Nuevo o respuesta?** Es respuesta si pertenece a la conversación de un ticket ya
   entregado, o si el asunto trae el folio `#a1b2c3d4` de un ticket del mismo canal.
3. **Firmas**: las imágenes pegadas pequeñas (< 10 KB) o repetidas del mismo remitente se
   descartan; las capturas de pantalla se conservan.
4. **Clasificación** (estado `CLASSIFYING` → `READY`): área, categoría, prioridad, título y
   descripción, con el catálogo **actual** del cliente. Si el LLM falla, reintenta a los 1, 5 y
   15 minutos; después entrega el ticket como borrador sin área (`fallback`).
5. **Respuestas**: el LLM evalúa si aportan algo (relevante o no) y se agregan al ticket.
6. **Entrega** (`DELIVERED`): ticket-management crea el ticket. Si no responde, se reintenta a
   los 1, 2, 5, 10 y 30 minutos y luego cada 30, sin límite. Si lo rechaza por algo que no se
   arregla reintentando (canal deshabilitado, demasiado grande…) queda **retenido** (`HELD`):
   ver [09_bandeja_mensajes_retenidos.md](09_bandeja_mensajes_retenidos.md).
7. **Salud del canal**: cada 5 minutos (y al momento de un error) reporta a ticket-management
   pendientes, fallidos y último error; se ve en "📨 Canales".

---

## Las áreas y categorías

Cada cliente tiene **su propio catálogo**, que el admin del cliente administra en el portal
(⚙️ Administración → Gestionar áreas y categorías). El clasificador lo recibe de
ticket-management con cada correo; **no se configura en este repo**. Las variables
`AGENT_DOMAINS` y `CATEGORIES_*` del `.env` solo se usan en modo mock (pruebas sin LLM).

---

## Contenedores

| Contenedor | Puerto (solo `127.0.0.1`) | Rol |
|---|---|---|
| `langchain-agent` | 8001 | Bandeja, clasificación, entrega |
| `langchain-api` | 8000 | Gateway LLM (`/ask`, requiere `X-Api-Key`) |
| `ollama` | 11434 | LLM local (`llama3.2:3b`); en Rocky no se usa |
| `classifier-db` | 5434 | PostgreSQL `classifier_db` |
| `classifier-redis` | 6381 | Cola de trabajo `inbox-work` y caché del LLM (1 h) |
| `minio` | 9000 / 9001 | Archivos de los correos (bucket `classifier-inbox`) |
| `classifier-flyway` | — | Migraciones (corre y termina) |

Desde otra máquina: túnel SSH (infra-monitoring `10_redes_y_acceso_lan.md` §7).

---

## Providers de IA disponibles

> **Vigente en Rocky (producción):** `AGENT_PROVIDER=oci`, modelo **`openai.gpt-oss-20b`**
> en OCI Generative AI (región Chicago). Comprobar: `curl -s localhost:8000/health` →
> `"default_provider":"oci"`.

| Provider | Modelo | Configuración |
|---|---|---|
| OCI GenAI | `openai.gpt-oss-20b` (`OCI_MODEL`) | `OCI_COMPARTMENT_ID`, `OCI_SERVICE_ENDPOINT` + `~/.oci/config` |
| Ollama | `llama3.2:3b` | `OLLAMA_BASE_URL=http://ollama:11434` |
| OpenAI | `gpt-4o-mini` | `OPENAI_API_KEY` |
| Anthropic | `claude-3-5-haiku-20241022` | `ANTHROPIC_API_KEY` |
| Gemini | `gemini-1.5-flash` | `GEMINI_API_KEY` |

Una sola variable elige el provider: `AGENT_PROVIDER` en el `.env` (compose la pasa a
`langchain-api` como `MODEL_PROVIDER`). Cambiarla: `03_guia_operacional.md` §7.

---

## Historia (para no confundirse)

Hasta el 2026-10-16 el clasificador recibía `POST /process` (sin llave), contestaba un
`job_id` y guardaba "tickets" propios en las tablas `tickets`, `attachments` y `enrichments`.
Todo eso **ya no existe**: lo reemplazó la bandeja persistente por canal
(`bandeja-persistente-del-clasificador`). Si un documento menciona `/process`, `/status/{job_id}`
o el bucket `email-attachments`, está desactualizado.

---

*ticket-classification · AI Stack for Dummies · Octubre 2026*
