# 02 — Arquitectura Técnica
## ticket-classification · Referencia completa
**Mayo 2026**

---

## Índice

1. [Visión general del stack](#1-visión-general-del-stack)
2. [docker-compose.yml — servicios y dependencias](#2-docker-composeyml--servicios-y-dependencias)
3. [LangChain Agent — núcleo de orquestación](#3-langchain-agent--núcleo-de-orquestación)
4. [LangChain API — gateway de IA](#4-langchain-api--gateway-de-ia)
5. [Modelo de datos](#5-modelo-de-datos)
6. [Migraciones — Flyway](#6-migraciones--flyway)
7. [MinIO — almacenamiento de adjuntos](#7-minio--almacenamiento-de-adjuntos)
8. [Redes Podman](#8-redes-podman)
9. [Variables de entorno — referencia completa](#9-variables-de-entorno--referencia-completa)
10. [Dependencias Python](#10-dependencias-python)
11. [Integración con ticket-management](#11-integración-con-ticket-management)
12. [Observabilidad](#12-observabilidad)

---

## 1. Visión general del stack

### Diagrama de arquitectura

```
┌──────────────────────────────────────────────────────────────────────┐
│                       ticket-classification                           │
│  (ticket-classification-network + ticket-management-network)          │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  CAPA DE ENTRADA — langchain-agent :8001                       │  │
│  │  POST /process → 202 + job_id                                  │  │
│  │  GET  /status/{job_id} → polling desde Redis                   │  │
│  └──────────────────────────────┬─────────────────────────────────┘  │
│                                 │                                     │
│  ┌──────────────────────────────▼─────────────────────────────────┐  │
│  │  CAPA DE AGENTE — LangGraph (background task)                  │  │
│  │                                                                │  │
│  │  ┌────────────┐   ┌────────────┐   ┌──────────────────────┐   │  │
│  │  │  classify  │──▶│  validate  │──▶│        save          │   │  │
│  │  └────────────┘   └─────┬──────┘   └──────────────────────┘   │  │
│  │        ▲                │ ❌ retry (≤ AGENT_MAX_ITERATIONS)    │  │
│  │        └────────────────┘                                      │  │
│  └──────────────────────────────┬─────────────────────────────────┘  │
│                                 │                                     │
│  ┌──────────────────────────────▼─────────────────────────────────┐  │
│  │  CAPA DE IA — langchain-api :8000                              │  │
│  │  Patrón Adapter + Strategy                                     │  │
│  │  ┌──────────┬──────────┬───────────┬──────────┐               │  │
│  │  │  Ollama  │  OpenAI  │ Anthropic │  Gemini  │               │  │
│  │  │  :11434  │  cloud   │   cloud   │  cloud   │               │  │
│  │  └──────────┴──────────┴───────────┴──────────┘               │  │
│  └─────────────────────────────────────────────────────────────── ┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  CAPA DE DATOS                                                │   │
│  │  classifier-db  :5434   DB: classifier_db                     │   │
│  │  classifier-redis :6381  caché LLM + estado jobs              │   │
│  │  minio          :9000   adjuntos de correos (S3-compatible)   │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  classifier-flyway ← migraciones automáticas (one-shot al arrancar) │
└──────────────────────────────────────────────────────────────────────┘
                    │ ticket-management-network
         ┌──────────┴──────────┐
         ▼                     ▼
ticket-system-backend    observability (infra-monitoring)
     :8080                 prometheus :9090 / grafana :3000
```

### Resumen de servicios

| Servicio | Imagen | Puerto | Rol |
|---|---|---|---|
| `langchain-agent` | build local | 8001 | Orquestador — API + LangGraph |
| `langchain-api` | build local | 8000 | Gateway de IA |
| `ollama` | ollama/ollama:0.1.32 | 11434 | IA local |
| `classifier-db` | postgres:15-alpine | **5434** (host) | Base de datos |
| `classifier-redis` | redis:7.2-alpine | **6381** (host) | Caché + estado de jobs |
| `minio` | minio/minio:RELEASE.2024-03-21T23-13-43Z | 9000/9001 | Storage de adjuntos |
| `classifier-flyway` | flyway:10.10.0 | — | Migraciones (one-shot) |

---

## 2. docker-compose.yml — servicios y dependencias

### Orden de arranque (dependencias)

```
classifier-db (healthcheck) ──┐
classifier-redis (healthcheck) ┼──▶ langchain-api ──▶ langchain-agent
minio (healthcheck) ───────────┘
ollama (service_started) ──────────▶ langchain-api
```

### Servicio: `langchain-agent`

```yaml
langchain-agent:
  build: ./langchain-agent
  container_name: langchain-agent
  ports: ["8001:8001"]
  volumes:
    - ./langchain-agent:/app    # Hot reload en desarrollo
  depends_on:
    classifier-db:    { condition: service_healthy }
    classifier-redis: { condition: service_healthy }
    langchain-api:    { condition: service_started }
    minio:            { condition: service_healthy }
  networks:
    - ticket-classification-network
    - ticket-management-network   # Acceso a ticket-system-backend
```

### Servicio: `langchain-api`

```yaml
langchain-api:
  build: ./langchain-api
  container_name: langchain-api
  ports: ["8000:8000"]
  depends_on:
    classifier-redis: { condition: service_healthy }
    ollama:           { condition: service_started }
  networks:
    - ticket-classification-network
```

### Servicio: `ollama`

```yaml
ollama:
  image: docker.io/ollama/ollama:0.1.32
  container_name: ollama
  ports: ["11434:11434"]
  volumes:
    - ./ollama_data:/root/.ollama
  dns: [8.8.8.8, 8.8.4.4]   # Necesario en WSL2 para resolver registry.ollama.ai
  networks:
    - ticket-classification-network
```

> El bloque `dns` solo es necesario en Podman rootless + WSL2 para el `pull`
> inicial del modelo. En Linux nativo no es necesario.

### Servicio: `classifier-db`

```yaml
classifier-db:
  image: docker.io/postgres:15-alpine
  container_name: classifier-db
  ports: ["5434:5432"]
  environment:
    POSTGRES_USER: admin
    POSTGRES_PASSWORD: admin
    POSTGRES_DB: classifier_db
  volumes:
    - classifier_postgres_data:/var/lib/postgresql/data
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U admin -d classifier_db"]
    interval: 10s  timeout: 5s  retries: 5  start_period: 20s
  networks:
    - ticket-classification-network
    - ticket-management-network   # Accesible desde otros stacks
```

### Servicio: `classifier-redis`

```yaml
classifier-redis:
  image: docker.io/redis:7.2-alpine
  container_name: classifier-redis
  ports: ["6381:6379"]
  command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
  volumes:
    - ./redis_data:/data
  healthcheck:
    test: ["CMD-SHELL", "redis-cli --no-auth-warning -a $REDIS_PASSWORD ping"]
  networks:
    - ticket-classification-network
```

> En desarrollo `REDIS_PASSWORD` puede quedar vacío. En producción debe
> tener un secreto fuerte.

### Servicio: `minio`

```yaml
minio:
  image: docker.io/minio/minio:RELEASE.2024-03-21T23-13-43Z
  container_name: minio
  ports: ["9000:9000", "9001:9001"]
  environment:
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin123
  command: server /data --console-address ":9001"
  volumes:
    - ./minio_data:/data
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
  networks:
    - ticket-classification-network
```

### Servicio: `classifier-flyway`

```yaml
flyway:
  image: docker.io/flyway/flyway:10.10.0
  container_name: classifier-flyway
  command: migrate
  environment:
    FLYWAY_URL: jdbc:postgresql://classifier-db:5432/classifier_db
    FLYWAY_USER: admin
    FLYWAY_PASSWORD: admin
    FLYWAY_LOCATIONS: filesystem:/flyway/sql
    FLYWAY_CLEAN_DISABLED: "true"
  volumes:
    - ./db/migrations/postgresql:/flyway/sql:ro
  depends_on:
    classifier-db: { condition: service_healthy }
  restart: "no"   # One-shot: corre y termina
```

---

## 3. LangChain Agent — núcleo de orquestación

### API REST — endpoints

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/health` | Estado del agente, provider activo, dominios |
| `POST` | `/process` | Recibe correo → devuelve `job_id` (202 Accepted) |
| `GET` | `/status/{job_id}` | Consulta resultado del job (polling) |
| `GET` | `/metrics` | Métricas Prometheus |
| `GET` | `/docs` | Swagger UI |

### Flujo asíncrono completo

```
1. POST /process
   ├── Guarda job:{uuid} = { status: "en_proceso" } en Redis (TTL 24h)
   └── Lanza background_task y responde 202 + job_id inmediatamente

2. Background task: process_email_job
   ├── Deduplicación: ¿ya existe ticket con este conversation_id?
   │   └── Sí → job:{uuid} = { status: "ignorado" } → termina
   ├── Crea AgentState y lanza graph.ainvoke()
   │   ├── [classify] → langchain-api POST /ask
   │   ├── [validate] → Pydantic + fuzzy matching
   │   └── [save]     → INSERT en classifier-db
   │                    → PUT en ticket-system-backend
   │                    → INSERT attachments en MinIO
   └── Actualiza job:{uuid} = { status: "completado", ... }

3. GET /status/{job_id}
   └── Lee job:{uuid} de Redis → devuelve JobStatusResponse
```

### Request completo — `POST /process`

```json
{
  "asunto": "Error en sistema de nómina",
  "cuerpo": "Desde esta mañana el sistema no permite procesar pagos.",
  "remitente": "juan.perez@empresa.com",
  "nombre_remitente": "Juan Pérez",
  "conversation_id": "AAMkAGI2TI5OGEtZWMxIT0001",
  "email_received_at": "2026-05-07T09:30:00Z",
  "adjuntos": [
    { "nombre": "captura_error.png", "tipo": "image/png", "contenido_b64": "..." },
    { "nombre": "reporte.xlsx", "tipo": "application/vnd.ms-excel", "contenido_b64": "..." }
  ],
  "provider": "openai",
  "max_iterations": 5
}
```

### Response `GET /status/{job_id}` — completado

```json
{
  "job_id": "abc-123-uuid",
  "status": "completado",
  "asunto": "Error en sistema de nómina",
  "remitente": "juan.perez@empresa.com",
  "conversation_id": "AAMkAGI2TI5OGEtZWMxIT0001",
  "ticket_id": 42,
  "external_ticket_id": "uuid-en-ticket-management",
  "dominio": "IT",
  "categoria": "base_de_datos",
  "categoria_propuesta": null,
  "requiere_revision": false,
  "prioridad": "alta",
  "confianza": 0.95,
  "alerta": "URGENTE: ticket de base_de_datos con prioridad alta",
  "iterations_used": 1,
  "validated": true,
  "cached": false,
  "provider": "openai",
  "duracion_ms": 3240
}
```

Estados posibles de `status`:
- `en_proceso` — el agente está trabajando, reintentar en unos segundos
- `completado` — clasificación exitosa
- `error` — ver campo `error` con el detalle
- `ignorado` — `conversation_id` ya tenía un ticket → ver `ticket_id_existente`

### Grafo LangGraph — nodos

```python
workflow = StateGraph(AgentState)
workflow.add_node("classify", classify_node)   # Llama a langchain-api
workflow.add_node("validate", validate_node)   # Pydantic + fuzzy matching
workflow.add_node("save",     save_node)        # Persiste en BD + MinIO + backend

workflow.set_entry_point("classify")
workflow.add_edge("classify", "validate")
workflow.add_conditional_edges(
    "validate",
    should_retry,       # → "save" | "classify" (retry) | END (max iteraciones)
    {"save": "save", "classify": "classify", END: END}
)
workflow.add_edge("save", END)
```

---

## 4. LangChain API — gateway de IA

### Endpoints

| Método | Endpoint | Descripción |
|---|---|---|
| `POST` | `/ask` | Consulta directa al LLM con provider elegido |
| `GET` | `/health` | Estado del servicio y providers configurados |
| `GET` | `/metrics` | Métricas Prometheus |
| `GET` | `/docs` | Swagger UI |

### Patrón Adapter + Strategy

```python
def get_llm(provider: str):
    if provider == "ollama":
        return ChatOllama(model=..., base_url="http://ollama:11434")
    elif provider == "openai":
        return ChatOpenAI(model=..., api_key=os.getenv("OPENAI_API_KEY"))
    elif provider == "anthropic":
        return ChatAnthropic(model=..., api_key=os.getenv("ANTHROPIC_API_KEY"))
    elif provider == "gemini":
        return ChatGoogleGenerativeAI(model=..., google_api_key=os.getenv("GEMINI_API_KEY"))
```

### Caché Redis en `/ask`

```python
cache_key = hashlib.md5(f"{prompt}{system}{provider}".encode()).hexdigest()
cached = redis_client.get(cache_key)
if cached:
    return AskResponse(..., cached=True)

result = await llm.ainvoke(messages)
redis_client.setex(cache_key, 3600, result.content)   # TTL 1 hora
```

### Logging de errores

Todos los errores en la llamada al LLM se loguean con traceback completo:

```python
except Exception as e:
    logger.error("Error llamando al provider '%s': %s", provider, e, exc_info=True)
    raise HTTPException(status_code=502, detail=f"Error: {str(e)}")
```

---

## 5. Modelo de datos

### Tabla `tickets`

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | SERIAL PK | ID interno |
| `body` | TEXT | Cuerpo del correo |
| `subject` | VARCHAR(500) | Asunto del correo |
| `domain` | VARCHAR(50) | IT / cliente / operaciones / otro |
| `category` | VARCHAR(255) | Categoría canónica (corregida por fuzzy) |
| `suggested_category` | VARCHAR(255) | Sugerencia original del LLM (si hubo corrección) |
| `requires_review` | BOOLEAN | True si la categoría no está en la lista |
| `priority` | VARCHAR(10) | alta / media / baja |
| `confidence` | FLOAT | 0.0 a 1.0 |
| `source` | VARCHAR(50) | webhook / gmail / manual |
| `sender` | VARCHAR(255) | Email del remitente |
| `sender_name` | VARCHAR(255) | Nombre del remitente |
| `alert` | TEXT | Mensaje generado post-clasificación |
| `external_ticket_id` | VARCHAR(36) | UUID en ticket-management-backend |
| `conversation_id` | VARCHAR(200) | ID del hilo Outlook — deduplicación |
| `email_received_at` | TIMESTAMP | Cuándo llegó el correo original |
| `created_at` | TIMESTAMP | Cuándo fue clasificado |

### Tabla `agent_runs`

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | SERIAL PK | ID interno |
| `run_id` | VARCHAR(36) | UUID = job_id devuelto al cliente |
| `ticket_id` | INTEGER FK | Referencia a `tickets.id` |
| `iterations_used` | INTEGER | Ciclos classify→validate usados |
| `is_validated` | BOOLEAN | Si terminó con clasificación válida |
| `provider` | VARCHAR(50) | ollama / openai / anthropic / gemini |
| `result` | JSON | Clasificación completa |
| `duration_ms` | INTEGER | Tiempo total de la ejecución |
| `created_at` | TIMESTAMP | Fecha de ejecución |

### Tabla `attachments`

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | SERIAL PK | ID interno |
| `ticket_id` | INTEGER FK | Referencia a `tickets.id` (CASCADE DELETE) |
| `filename` | VARCHAR(255) | Nombre del archivo adjunto |
| `content_type` | VARCHAR(100) | MIME type |
| `storage_key` | VARCHAR(500) | Ruta en MinIO |
| `created_at` | TIMESTAMP | Fecha de carga |

### Consultas útiles de operación

```sql
-- Tickets del día por dominio y prioridad
SELECT domain, priority, COUNT(*) AS total
FROM tickets
WHERE created_at >= CURRENT_DATE
GROUP BY domain, priority
ORDER BY domain, priority;

-- Tickets que requieren revisión manual
SELECT id, subject, domain, category, suggested_category, priority, created_at
FROM tickets
WHERE requires_review = true
ORDER BY created_at DESC;

-- Tickets de alta prioridad (últimas 24h)
SELECT id, subject, domain, category, sender, created_at
FROM tickets
WHERE priority = 'alta'
  AND created_at >= NOW() - INTERVAL '24 hours'
ORDER BY created_at DESC;

-- Tasa de éxito del agente
SELECT iterations_used, COUNT(*) AS total,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct
FROM agent_runs
GROUP BY iterations_used
ORDER BY iterations_used;

-- Performance por provider
SELECT provider, AVG(duration_ms) AS avg_ms, COUNT(*) AS total
FROM agent_runs
WHERE is_validated = true
GROUP BY provider;
```

---

## 6. Migraciones — Flyway

Flyway gestiona el esquema de la base de datos automáticamente.
El contenedor `classifier-flyway` corre una vez al arrancar y termina.

```
db/migrations/postgresql/
└── V1__create_schema_inicial.sql   ← Crea tickets, agent_runs, attachments
```

**Convención de naming:**
```
V{N}__{descripcion_snake_case}.sql   (doble guión bajo obligatorio)
```

**Verificar estado de migraciones:**

```bash
podman logs classifier-flyway
# Resultado esperado:
# Successfully validated 1 migration
# Current version of schema "public": 1
# Migrating schema "public" to version "1 - create schema inicial"
# Successfully applied 1 migration
```

**Agregar una nueva migración:**

```bash
# Crear el archivo con el siguiente número de versión
touch ~/stack_ticket/ticket-classification/db/migrations/postgresql/V2__descripcion.sql

# Al reiniciar el stack, Flyway aplica automáticamente las migraciones pendientes
podman-compose restart
```

---

## 7. MinIO — almacenamiento de adjuntos

MinIO almacena los archivos adjuntos de correos (PDFs, Excel, imágenes)
de forma separada a la base de datos relacional.

**Flujo de un adjunto:**

```
POST /process (adjunto en base64)
     │
     ▼
langchain-agent → sube archivo a MinIO
     │             storage_key = "tickets/{ticket_id}/{filename}"
     │
     ▼
INSERT INTO attachments (ticket_id, filename, content_type, storage_key)
     │
     ▼
GET  /status/{job_id} ← no incluye el contenido, solo metadata
```

**Acceso a MinIO:**

```bash
# Verificar salud
curl -s http://localhost:9000/minio/health/live && echo "MinIO OK"

# Consola web (navegador Windows)
# http://localhost:9001  →  minioadmin / minioadmin123

# Listar archivos del bucket desde WSL
podman exec minio mc ls local/email-attachments/
```

---

## 8. Redes Podman

### Redes del stack

| Red | Subnet | Tipo |
|---|---|---|
| `ticket-classification-network` | 10.89.2.0/24 | Externa (creada por startup.sh) |
| `ticket-management-network` | 10.89.1.0/24 | Externa (compartida con otros repos) |

### Tabla de comunicación entre servicios

| Desde | Hacia | Hostname |
|---|---|---|
| `langchain-agent` | `langchain-api` | `langchain-api:8000` |
| `langchain-agent` | `classifier-db` | `classifier-db:5432` |
| `langchain-agent` | `classifier-redis` | `classifier-redis:6379` |
| `langchain-agent` | `minio` | `minio:9000` |
| `langchain-agent` | `ticket-system-backend` | `ticket-system-backend:8080` |
| `langchain-api` | `ollama` | `ollama:11434` |
| `langchain-api` | `classifier-redis` | `classifier-redis:6379` |
| `prometheus` | `langchain-agent` | `langchain-agent:8001` |
| `prometheus` | `langchain-api` | `langchain-api:8000` |

### Diferencia Docker vs Podman rootless

```bash
# Docker
Socket: /var/run/docker.sock

# Podman rootless (WSL2)
Socket: /run/user/$UID/podman/podman.sock
export DOCKER_HOST=unix:///run/user/$UID/podman/podman.sock
```

---

## 9. Variables de entorno — referencia completa

Ver `.env.example` para la lista completa. Variables críticas:

| Variable | Default | Descripción |
|---|---|---|
| `AGENT_PROVIDER` | `ollama` | Provider activo: ollama / openai / anthropic / gemini |
| `AGENT_MAX_ITERATIONS` | `5` | Máximo de ciclos classify→validate |
| `AGENT_DOMAINS` | `IT,cliente,operaciones,otro` | Dominios del clasificador |
| `CATEGORIES_IT` | lista de categorías | Categorías válidas para dominio IT |
| `FUZZY_THRESHOLD` | `80` | Similitud mínima para fuzzy matching (0-100) |
| `MIN_CONFIDENCE` | `0.7` | Confianza mínima — por debajo reintenta |
| `TICKET_MGMT_API_KEY` | — | Key de integración con ticket-management |
| `TICKET_MGMT_CATALOGO_ENABLED` | `true` | Sincronizar catálogo de categorías |
| `AGENT_MOCK_CLASSIFY` | `false` | Modo mock — salta el LLM |
| `LANGCHAIN_TRACING_V2` | `false` | Activar trazas LangSmith |

---

## 10. Dependencias Python

### `langchain-agent/requirements.txt` (principales)

```
fastapi / uvicorn          # API REST
langgraph / langchain      # Orquestación del agente
pydantic                   # Validación de schema
asyncpg / sqlalchemy       # Acceso async a PostgreSQL
redis[asyncio]             # Caché + estado de jobs
httpx                      # Llamadas HTTP async entre servicios
minio                      # Cliente MinIO para adjuntos
prometheus-fastapi-instrumentator  # Métricas en /metrics
```

### `langchain-api/requirements.txt` (principales)

```
fastapi / uvicorn
langchain-core
langchain-ollama / langchain-openai / langchain-anthropic / langchain-google-genai
redis                      # Caché de respuestas LLM
prometheus-fastapi-instrumentator
```

---

## 11. Integración con ticket-management

El agente llama a `ticket-system-backend` para crear el ticket en el sistema
central de gestión, después de clasificarlo en `classifier-db`.

**Configuración:**

```bash
# .env
TICKET_MGMT_API_URL=http://ticket-system-backend:8080/api/v1
TICKET_MGMT_API_KEY=reemplazar_con_secreto_real    # Debe coincidir con INTERNAL_API_KEY de ticket-management
TICKET_MGMT_CATALOGO_ENABLED=true                  # Sincroniza catálogo de categorías al arrancar
```

**`start.sh` detecta automáticamente el contexto:**

```bash
# Si ticket-system-backend está en la red compartida:
TICKET_MGMT_API_URL=http://ticket-system-backend:8080/api/v1

# Si ticket-management corre fuera de WSL (modo standalone):
TICKET_MGMT_API_URL=http://<IP-Windows>:8080/api/v1
```

---

## 12. Observabilidad

La observabilidad está centralizada en `infra-monitoring` — Prometheus y Grafana
viven en ese repo, no en este.

Ambos servicios de este repo exponen `/metrics` con la instrumentación automática
de `prometheus-fastapi-instrumentator`:

| Endpoint | Qué expone |
|---|---|
| `langchain-agent:8001/metrics` | Requests, latencia de `/process` y `/status` |
| `langchain-api:8000/metrics` | Requests, latencia de `/ask` por provider |

El `prometheus.yml` en la raíz de este repo define los scrape configs usados
por la instancia de Prometheus en `infra-monitoring`.

---

*ticket-classification · Arquitectura Técnica · Mayo 2026*
