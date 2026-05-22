# 05 — REST API (Fase 4)

Documentación de la API REST expuesta por `src/api/`.

---

## Arranque

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate
uvicorn src.api.main:app --port 8002 --reload
```

Al arrancar, la API carga el modelo ONNX, el índice FAISS y el LLM **una sola vez**.
Las requests posteriores reutilizan esos recursos — no hay cold start por query.

Swagger disponible en: `http://localhost:8002/docs`

---

## Endpoints

### GET /api/v1/health

Estado del servicio.

```bash
curl http://localhost:8002/api/v1/health
```

```json
{
  "status": "UP",
  "application": "ticket-agent"
}
```

---

### GET /api/v1/info

Configuración activa y métricas básicas.

```bash
curl http://localhost:8002/api/v1/info
```

```json
{
  "application": "ticket-agent",
  "provider": "oci",
  "model": "cohere.command-r-plus-08-2024",
  "chunksTotal": 1184,
  "multiQueryN": 3,
  "retrievalK": 3
}
```

---

### POST /api/v1/query

Consulta RAG con filtrado por rol.

```bash
curl -X POST http://localhost:8002/api/v1/query \
  -H "Content-Type: application/json" \
  -d '{"query": "¿Cómo creo un ticket?", "rol": "usuario"}'
```

**Request:**

```json
{
  "query": "¿Cómo creo un ticket?",
  "rol": "usuario"
}
```

| Campo | Tipo | Valores | Default |
|---|---|---|---|
| `query` | string | 1–1000 caracteres | requerido |
| `rol` | string | `usuario`, `manager`, `admin`, `soporte` | `usuario` |

**Response 200:**

```json
{
  "respuesta": "Para crear un ticket, haz clic en el botón Nuevo Ticket...",
  "fuentes": ["../ticket-management/docs/usuarios/usuario/flujos.md"],
  "pantallas": [],
  "rol": "usuario",
  "confianza": 0.8,
  "latenciaMs": 11540.5
}
```

**Response 404** — sin documentos relevantes para el rol:

```json
{
  "status": 404,
  "error": "Not Found",
  "message": "No se encontraron documentos relevantes para rol 'usuario'",
  "timestamp": "2026-05-21T21:00:00Z"
}
```

**Response 500** — error interno:

```json
{
  "status": 500,
  "error": "Internal Server Error",
  "message": "...",
  "timestamp": "2026-05-21T21:00:00Z"
}
```

---

## Estructura interna

```
src/api/
├── main.py          ← FastAPI app + lifespan
├── models/
│   ├── request.py   ← QueryRequest (Pydantic)
│   └── response.py  ← QueryResponse, HealthResponse, InfoResponse, ErrorResponse
└── routes/
    ├── health.py    ← GET /api/v1/health, GET /api/v1/info
    └── query.py     ← POST /api/v1/query
```

---

## Diferencia CLI vs API

| Aspecto | CLI (`src/main.py`) | API (`src/api/main.py`) |
|---|---|---|
| Carga del modelo | En cada ejecución | Una vez al arrancar |
| Interfaz | Terminal | HTTP REST |
| Swagger | No | `http://localhost:8002/docs` |
| Uso | Desarrollo / diagnóstico | Integración con otros servicios |

---

## Convenciones (alineadas con ticket-management)

- URLs: `/api/v1/` con kebab-case
- JSON responses: camelCase (`latenciaMs`, `chunksTotal`)
- Errores: `{status, error, message, timestamp}`
- HTTP status codes: 200, 404, 500

---

## Configuración relevante en .env

```bash
# Puerto de arranque (no leído por FastAPI — se pasa en el comando uvicorn)
# uvicorn src.api.main:app --port 8002

# El resto de variables son las mismas que usa el CLI
PROVIDER=oci
MULTI_QUERY_N=3
RETRIEVAL_K=3
```

---

## Próximo paso — Containerización

Cuando la API esté validada, se empaqueta como contenedor Podman:

```
Dockerfile         → python:3.11-slim + requirements.txt + uvicorn
podman-compose.yml → puerto 8002, red ticket-management-network, volumen faiss_data
```

Ver `TAREAS.md` sección Fase 4 → Containerización.
