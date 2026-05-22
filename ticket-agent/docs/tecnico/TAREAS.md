# TAREAS — ticket-agent

Control de actividades por fase. Marcar `[x]` al completar cada tarea.
Última actualización: 2026-05-21 (Fase 1 validada end-to-end — Ollama + OCI GenAI)

---

## DECISIÓN CRÍTICA: Embeddings → Oracle 23ai

> Esta sección documenta una restricción de diseño que afecta todas las fases.

### El problema

El modelo de embeddings determina las dimensiones del vector (`EMBED_DIMS`).
Si cambia el modelo, el índice entero queda inválido y hay que re-ingestar todos los documentos.

### La solución adoptada

Usar desde el Día 1 el modelo que es compatible con Oracle 23ai:

| Parámetro | Valor decidido |
|---|---|
| Modelo | `mixedbread-ai/mxbai-embed-large-v1` |
| Dims | **1024** |
| Tipo | FLOAT32 |
| Proveedor | ONNX local (sentence-transformers) |

### Plan de migración FAISS → Oracle 23ai

```
FASE 1 (actual)
  Vector store: FAISS-cpu (local, en disco)
  Embedding: mxbai-embed-large-v1, 1024 dims  ← FIJO
  Chunking: flat (RecursiveCharacterTextSplitter, 512 tok)
  Costo migración: CERO — mismo modelo ONNX

         ↓  migración Fase 2

FASE 2
  Vector store: Oracle 23ai Free (contenedor Podman)
  Embedding: mismo modelo ONNX  ← sin re-ingesta
  Chunking: Parent-Child (256-512 hijo / 2048 padre)  ← re-ingesta por estrategia
  Cambios: solo swap del adapter (FAISSVectorStore → OracleVSAdapter)
           + re-ingesta por cambio de chunking (no por embeddings)
```

**Regla:** si en algún momento se propone cambiar `EMBED_MODEL_NAME`,
evaluar primero si Oracle 23ai lo soporta a 1024 dims antes de aceptar.

---

## FASE 1 — MVP Local (Ollama + FAISS)

**Objetivo:** CLI funcional que responda preguntas sobre el ecosistema de tickets según el rol
del usuario (usuario, manager, admin, soporte), usando Ollama como LLM y FAISS como vector store.

### Setup del proyecto

- [x] Crear estructura de carpetas completa
- [x] Crear `requirements.txt` con dependencias validadas
- [x] Crear `.env.example` con todas las variables documentadas
- [x] Crear `CLAUDE.md` con contexto del proyecto para Claude Code
- [x] Crear `.gitignore` adecuado (excluir `.env`, `data/`, `*.pkl`)

### Configuración (`src/config.py`)

- [x] Implementar `Settings` con Pydantic BaseSettings
- [x] Validar que campos OCI sean `Optional[str] = None`
- [x] Agregar validación condicional: si `PROVIDER=oci`, campos OCI requeridos
- [x] Testear carga desde `.env` con `python -c "from src.config import Settings; print(Settings())"`

### Adapters base (`src/adapters/`)

- [x] Implementar `LLMAdapter` (ABC) con `invoke()` y `can_cache()`
- [x] Implementar `EmbedderAdapter` (ABC) con `embed_documents()` y `embed_query()`
- [x] Implementar `ONNXEmbedderAdapter` — modelo `mxbai-embed-large-v1`, 1024 dims
- [x] Verificar que `ONNXEmbedderAdapter.embed_query()` devuelve lista de 1024 floats
- [x] Implementar `OllamaLLMAdapter`
- [x] Test unitario: dimensión del vector = 1024 (`tests/test_adapters.py::test_onnx_embedder_dims`)

### Document processing (`src/document_processing/`)

- [x] Implementar `Document` y `Chunk` dataclasses en `schema.py`
  - Metadata obligatoria por chunk: `rol` (list), `tipo`, `fuente`, `pantalla` (opcional)
- [x] Implementar `loader.py`: loader multi-fuente
  - `load_usuarios(base_path)` → carga MD de `docs/usuarios/{rol}/`, asigna metadata `rol`
  - `load_runbooks(paths)` → carga MD de todos los `docs/runbooks/`, asigna `rol: [soporte]`
  - `load_all(source)` → orquesta las tres fuentes según `--source`
- [x] Implementar `help_serializer.py`: serializa `helpContent.ts` → chunks con metadata
  - Lee el archivo `.ts` y extrae `HELP_CONTENT` (parseando el objeto JS/TS con regex)
  - Por cada `screen`: genera chunks con `{fuente: helpContent, pantalla: screen, rol: [rol_correspondiente]}`
  - Mapeo de pantalla → rol: newTicket/ticketDetail/attachments/ticketEvents → todos los roles;
    draftReview → [manager, admin]; classificationManagement/userManagement/permissionManagement → [admin]
- [x] Implementar `chunking.py`: `DocumentChunker` con `RecursiveCharacterTextSplitter`
  - `chunk_size=512`, `chunk_overlap=100`
  - Separadores: `["\n## ", "\n### ", "\n", " "]`
  - Preservar metadata del Document en cada Chunk resultante
- [x] Test: chunks tienen tamaño correcto y metadata `rol` propagada (`tests/test_chunking.py`)

### Vector store (`src/vector_store/faiss_store.py`)

- [x] Implementar `FAISSVectorStore.add_chunks()` con persistencia
- [x] Implementar `FAISSVectorStore.load()` con `allow_dangerous_deserialization=True`
- [x] Implementar `FAISSVectorStore.retrieve(query, k)` — filtrado por rol vía metadata post-retrieval
- [x] Test: guardar índice y volver a cargarlo sin pérdida de datos (validado en ciclo ingest→query)

### RAG (`src/rag/`)

- [x] Implementar `Retriever` con `retrieve(query) → list[LangDoc]`
- [x] Implementar `prompts.py` con `SYSTEM_PROMPT` y `RAG_PROMPT`
- [x] Formatear prompt final: system + contexto + pregunta (`format_prompt()`)

### CLI (`src/main.py`)

- [x] Implementar comando `ingest --source <all|usuarios|runbooks|help>`
  - `all`: ingesta las tres fuentes
  - `usuarios`: solo `ticket-management/docs/usuarios/`
  - `runbooks`: todos los `docs/runbooks/` de los 6 repos (incluye `./docs/runbooks` del propio agente)
  - `help`: solo `helpContent.ts` serializado
- [x] Implementar comando `query --query "<pregunta>" --rol <usuario|manager|admin|soporte>`
  - Filtra chunks por metadata `rol` antes de buscar en FAISS
  - `soporte` no aplica filtro — accede a todas las fuentes
- [x] Output de `query` incluye: respuesta, fuentes (archivos + pantalla si aplica), rol usado, latencia

### Tests

- [x] `tests/test_chunking.py`: chunks tienen tamaño esperado, metadata correcta
- [x] `tests/test_retriever.py`: top-K retorna chunks relevantes, filtrado por rol verificado
- [x] `tests/test_adapters.py`: ONNX devuelve 1024 dims, Ollama responde (mocked), ABCs correctos
- [x] `tests/conftest.py`: fixtures compartidos (Document/Chunk de ejemplo, roles mixtos)

### Validación end-to-end

- [x] Ejecutar `python -m src.main ingest --source all` (1119 chunks generados)
- [x] Query rol usuario: `python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario`
  - Respuesta basada en `ticket-management/docs/usuarios/usuario/flujos.md` ✓
  - Latencia OCI: 11.91s | Latencia Ollama: ~85s (llama3.2:3b en CPU, sin GPU)
- [x] Query rol manager: `python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager`
  - Fuentes correctas: `manager/estados.md`, `manager/glosario.md` ✓ — Confianza 0.8, 7.95s
  - Nota: respuesta incompleta (falta paso a paso) — gap de documentación en ticket-management, no del agente
- [x] Query rol soporte: `python -m src.main query --query "¿Dónde se configura el modelo LLM?" --rol soporte`
  - Devuelve respuesta basada en runbooks de ticket-classification ✓
  - Gap detectado: agente no conocía su propia config (PROVIDER, OCI_*) — corregido agregando `./docs/runbooks`
- [x] Verificar que un query con `--rol usuario` NO retorna contenido de runbooks técnicos
  - Query "¿Cómo funciona langchain-agent?" --rol usuario → "No se encontraron chunks relevantes" ✓
- [x] Latencia aceptable: OCI GenAI ~8-12s; Ollama CPU-only >60s (esperado sin GPU)

---

## ORDEN DE EJECUCIÓN REVISADO

> Decisión 2026-05-21: Fase 3 no depende de Oracle. El swap FAISS → Oracle es una sola
> línea en `main.py`. Factory pattern se implementa junto con `OracleVSAdapter`, no antes.

```
HOY          → Fase 3 (LangGraph sobre FAISS) + Fase 4 REST API/Docling (independientes)
MAÑANA       → Fase 2 cuando lleguen tablas Oracle (incluye Factory pattern + OracleVSAdapter)
DESPUÉS      → Fase 4 Mermaid → Property Graph (requiere Oracle operativo)
```

---

## FASE 2 — Oracle 23ai + Factory pattern

**Objetivo:** Reemplazar FAISS por Oracle 23ai Free como vector store.
**Estado:** En espera — tablas Oracle a crear el 2026-05-22.

### Oracle 23ai (infraestructura)

- [ ] Agregar `oracle/database-free:23.6-slim` a `podman-compose.yml`
- [ ] Crear script SQL inicial: tablas vectoriales `KB_MANUALS`, `KB_RUNBOOKS`
- [ ] Configurar Hybrid Vector Index (HNSW + Oracle Text) en cada tabla
- [ ] Verificar conectividad desde Python con `oracledb`

### Oracle Vector Store adapter + Factory pattern

> Factory pattern se implementa aquí, no antes — el acoplamiento actual (FAISS en main.py)
> es mínimo y no bloquea Fase 3.

- [ ] Implementar `OracleVSAdapter` con la misma interfaz que `FAISSVectorStore`
  - `add_chunks()`, `load()`, `retrieve()`
- [ ] Usar `langchain-community` `OracleVS` o implementación directa con `oracledb`
- [ ] **Verificar que EMBED_DIMS=1024 y tipo FLOAT32** en la columna vectorial
- [ ] Test: migrar índice FAISS existente a Oracle sin cambiar el modelo ONNX
- [ ] Implementar `VectorStoreFactory.create(store_type, config)` — swap FAISS/Oracle por `.env`
- [ ] Implementar `LLMAdapterFactory.create(provider, config)`
- [ ] Actualizar `main.py` para usar factories

### OCI GenAI adapter

- [x] Implementar `OCILLMAdapter` (implementado y validado en Fase 1 — incluye temperature, max_tokens)
- [x] Validar autenticación con `~/.oci/config` (fix WSL: key_file `/home/jjrm` en lugar de `/root`)
- [x] Test: query contra `cohere.command-r-plus-08-2024` — latencia ~10-12s ✓

### Parent-Child chunking

- [ ] Implementar `ParentChildChunker` en `chunking.py`
  - Hijo: 256-512 tokens (para retrieval)
  - Padre: ~2048 tokens (para contexto full)
- [ ] Re-ingestar documentos con nueva estrategia
- [ ] Test: recuperar chunk hijo + parent completo

### Hybrid Search

- [ ] Agregar BM25 reranking sobre resultados vector search
- [ ] Combinar scores (RRF o weighted sum)
- [ ] Test: hybrid search mejora relevancia vs solo vector

---

## FASE 3 — Agentic RAG (LangGraph)

**Objetivo:** Reemplazar LCEL chain por LangGraph con lógica condicional.
**Nota:** Se implementa sobre FAISS. Cuando Fase 2 esté lista, el swap es una línea en `main.py`.

### LangGraph core (`src/agent/`)

- [x] Implementar `AgentState` (TypedDict: query, rol, docs, contexto, respuesta, confianza, ciclos)
- [x] Implementar nodo `retrieve` — factory closure sobre `Retriever.retrieve()`
- [x] Implementar nodo `generate` — factory closure sobre `LLMAdapter.invoke()`
- [x] Implementar nodo `validate` — heurística de confianza (0.0 / 0.3 / 0.8)
- [x] Implementar nodo `web_fetch` — placeholder CRAG (enriquece contexto, máx 1 ciclo)
- [x] Implementar routing condicional: confianza ≥ 0.5 → END; si no → web_fetch (MAX_CICLOS=1)
- [x] Conectar nodos con `StateGraph` y compilar (`build_graph(retriever, llm)`)
- [x] Actualizar `main.py query` para invocar el grafo en lugar de la chain LCEL
- [x] Output extendido: `Rol | Confianza | Latencia`
- [x] Agregar `langgraph>=0.2,<2.0` a `requirements.txt`

### Estrategias de retrieval avanzado

- [ ] Implementar HyDE (Hypothetical Document Embedding)
  - Generar documento hipotético con el LLM → embedear → buscar por similitud
- [x] Implementar Multi-Query con RRF reranking
  - `make_multi_query_node(retriever, llm, n)` en `nodes.py`
  - Genera N variantes con el LLM → N búsquedas FAISS → RRF (k=60) → top-K chunks
  - Configurable: `MULTI_QUERY_N=3` en `.env` (default 3; valor 1 desactiva y usa búsqueda simple)
  - Prompt de variantes en `src/rag/prompts.py` (`format_multi_query_prompt`)
- [x] Validar mejora de calidad: query manager antes devolvía respuesta incompleta; con Multi-Query devuelve paso a paso completo (latencia +4s, aceptable)

### CRAG (Corrective RAG)

- [ ] Implementar nodo `web_fetch` real — búsqueda web cuando confianza baja
- [ ] Combinar contexto local + web antes de `generate`
- [ ] Test: query fuera del dominio activa web_fetch; query normal no lo activa

### Graph Traversal *(requiere Fase 2 — Oracle 23ai)*

- [ ] Implementar traversal sobre Property Graph con PGQL/SQL PGQ
- [ ] Conectar como nodo adicional del StateGraph

---

## FASE 4 — Docling + REST API + Mermaid

**Objetivo:** Ingestar documentos complejos y exponer API REST.

### Independiente de Oracle — puede empezar en paralelo con Fase 3

- [ ] Reemplazar `loader.py` por Docling (PDF, DOCX, YAML, Web URLs)
- [x] Implementar REST API con FastAPI (`src/api/`)
  - [x] `POST /api/v1/query` — recibe `{query, rol}`, devuelve `{respuesta, fuentes, pantallas, rol, confianza, latenciaMs}`
  - [x] `GET /api/v1/health` → `{"status": "UP", "application": "ticket-agent"}`
  - [x] `GET /api/v1/info` → modelo activo, chunks ingestados, multiQueryN, retrievalK
  - [x] Error estándar: `{status, error, message, timestamp}` — mismo patrón que ticket-management
  - [x] Lifespan: embedder + FAISS + LLM + grafo cargados una vez al arrancar (no por request)
  - [x] Swagger disponible en `/docs`
  - [x] Agregar `fastapi>=0.110` y `uvicorn[standard]>=0.27` a `requirements.txt`
  - [x] Validar: `uvicorn src.api.main:app --port 8002 --reload` arranca sin errores
  - [x] Validar: `GET /api/v1/health` → `{"status":"UP","application":"ticket-agent"}` ✓
  - [x] Validar: `GET /api/v1/info` → provider, model, chunksTotal=1184, multiQueryN=3 ✓
  - [x] Validar: `POST /api/v1/query` → respuesta completa, 2 fuentes, pantalla newTicket, confianza=0.8, latenciaMs=7458 ✓
  - [x] Bonus: latencia API (7.4s) < latencia CLI (11.5s) — modelo cargado en memoria en lifespan
- [ ] Implementar `Diagnostic Agent` y `Remediation Agent`
- [x] LangSmith para observabilidad (implementado en Fase 1 — proyecto `ticket-agent-dev`)

### Containerización *(se activa junto con REST API)*

- [x] Crear `Dockerfile` para el servicio FastAPI
  - Base: `python:3.11-slim`
  - `ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1` (logs sin buffering para Loki)
  - Instala dependencias desde `requirements.txt`
  - Expone puerto `8002`
  - ENTRYPOINT: `entrypoint.sh` (verifica FAISS antes de arrancar uvicorn)
- [x] Crear `entrypoint.sh` — falla con mensaje claro si el volumen `faiss_data` está vacío
- [x] Crear `start.sh` — mismo patrón que demás repos: crea `.env`, avisa si falta índice FAISS, levanta podman-compose
- [x] Crear `podman-compose.yml` con servicio `ticket-agent`
  - Puerto: `8002:8002`
  - Volúmenes: `faiss_data` (índice FAISS), `agent_processed` (chunks.jsonl), `hf_cache` (modelo ONNX ~670 MB)
  - Red: `ticket-management-network` (red compartida del ecosistema)
  - Variables desde `.env`; Ollama por `host.containers.internal:11434`
- [x] Agregar `ticket-agent` al mapa de redes en `infra-monitoring/docs/runbooks/11_puertos_y_redes.md`
- [x] Integrar arranque en `infra-monitoring/startup.sh` (posición 3, después de ticket-classification)
- [x] Logs JSON estructurados con `pythonjsonlogger` (mismo estándar que `langchain-agent`)
- [x] Métricas Prometheus vía `prometheus-fastapi-instrumentator` — endpoint `/metrics`
- [x] Job `ticket-agent` agregado en `infra-monitoring/prometheus.yml`
- [x] Rate limiting en `POST /api/v1/query` — 10 req/min por IP (`slowapi`) con handler 429 en formato estándar del ecosistema
- [ ] Test: `curl http://localhost:8002/api/v1/health` desde Windows (pendiente primer build en WSL)

### Requiere Oracle (Fase 2 completa)

- [ ] Implementar Mermaid parser → Property Graph en Oracle 23ai
- [ ] En containerización: reemplazar volumen `faiss_data` por variable `ORACLE_DSN` apuntando al contenedor Oracle

---

## MEJORAS DE ECOSISTEMA (afectan todos los repos Python)

> Estas tareas no son exclusivas de ticket-agent. Se registran aquí para visibilidad.
> Implementarlas requiere cambios coordinados en `langchain-agent` y `langchain-api` también.

- [ ] **X-Request-ID / Correlation ID** — inyectar header `X-Request-ID` en cada request y propagarlo en los logs JSON. Permite trazar una transacción distribuida a través de múltiples servicios en Loki/Grafana. Impacto: bajo hoy, crítico en producción con tráfico concurrente.
- [ ] **CORSMiddleware** — configurar orígenes permitidos explícitamente. Baja urgencia: los servicios Python son APIs backend-to-backend, no las consume el browser directamente.
- [ ] **Contenedores sin root** — crear usuario `appuser` no privilegiado en los Dockerfiles. En Podman rootless el riesgo está mitigado, pero es buena práctica para migración futura a Docker o servidor compartido.

---

## NOTAS Y DECISIONES

| Fecha | Decisión | Razón |
|---|---|---|
| 2026-05-21 | Nombre del repo: `ticket-agent` | Consistencia con convención `ticket-*` del ecosistema |
| 2026-05-21 | Usar FAISS en Fase 1, no Oracle 23ai | Reduce setup inicial; migración limpia posible |
| 2026-05-21 | Modelo ONNX: `mxbai-embed-large-v1` (1024 dims) | Compatibilidad directa con Oracle 23ai FLOAT32 |
| 2026-05-21 | OCI campos como Optional en Fase 1 | Evita fallo de Pydantic cuando PROVIDER=ollama |
| 2026-05-21 | `allow_dangerous_deserialization=True` en FAISS.load_local | Requerido por langchain-community reciente |
| 2026-05-21 | Parser: RecursiveCharacterTextSplitter en Fase 1 | Docling es overkill para .md; Fase 4 lo incorpora |
| 2026-05-21 | Filtrado por rol via metadata en chunks, no índices separados | Un solo FAISS, filtrado post-retrieval por metadata `rol` |
| 2026-05-21 | helpContent.ts serializado al vuelo, sin MD intermedio | Evita doble fuente de verdad si la UI cambia |
| 2026-05-21 | 3 fuentes: docs/usuarios/ + helpContent.ts + docs/runbooks/ | Cubre usuario final, UI contextual y soporte técnico |
| 2026-05-21 | 11 MD en ticket-management/docs/usuarios/ ya creados y commiteados | Fuente 1 lista; Fuentes 2 y 3 se ingestan desde paths originales |
| 2026-05-21 | Rol `soporte` no tiene restricción de metadata — accede a todo | Personal técnico necesita cruzar info de múltiples repos |
| 2026-05-21 | Agregar `./docs/runbooks` al inicio de SOURCE_RUNBOOKS | El agente no conocía su propia configuración (PROVIDER, OCI_*) al consultarse con --rol soporte |
| 2026-05-21 | LangSmith integrado en Fase 1, proyecto separado `ticket-agent-dev` | Mismas credenciales que ticket-classification pero trazas aisladas por proyecto |
| 2026-05-21 | OCI key_file en WSL apunta a `/home/jjrm/.oci/`, no a `/root/.oci/` | Ticket-agent corre como usuario WSL, no como root — fix con sed en `~/.oci/config` |
| 2026-05-21 | Modelo OCI probado: `cohere.command-r-plus-08-2024` (Chicago endpoint) | Meta llama queda para Fase 2 cuando se elija modelo definitivo de producción |
| 2026-05-21 | Fase 3 arranca sobre FAISS sin esperar Oracle | Swap a Oracle = 1 línea en main.py; LangGraph no conoce el vector store |
| 2026-05-21 | Factory pattern se implementa junto con OracleVSAdapter en Fase 2 | No es bloqueante para Fase 3; el acoplamiento actual es mínimo y está en un solo lugar |
| 2026-05-21 | Fase 4 REST API y Docling son independientes de Oracle | Se pueden implementar en paralelo con Fase 3; solo Mermaid→Oracle espera Fase 2 |
| 2026-05-21 | Containerización va junto con REST API en Fase 4, no antes | El CLI no necesita contenedor; el servicio FastAPI sí necesita vivir en la red del stack |
| 2026-05-21 | Puerto asignado a ticket-agent: 8002 | Sigue la secuencia del ecosistema (8000 API, 8001 agent classification, 8002 ticket-agent) |
| 2026-05-21 | LangGraph core implementado sobre FAISS (Fase 3) | StateGraph con 4 nodos; swap a Oracle es 1 línea en main.py cuando llegue Fase 2 |
| 2026-05-21 | web_fetch es placeholder en Fase 3 inicial | CRAG real viene en siguiente iteración; MAX_CICLOS=1 evita loops infinitos |
| 2026-05-21 | Confianza calculada con heurística (no LLM scoring) | Evita segunda llamada al LLM; suficiente para Fase 3 — scoring real en Fase 3 CRAG |
| 2026-05-21 | Multi-Query usa RRF k=60 (constante estándar de la literatura) | Valor probado en benchmarks IR; no requiere tuning inicial |
| 2026-05-21 | MULTI_QUERY_N=1 desactiva Multi-Query y usa búsqueda simple | Permite comparar calidad con/sin Multi-Query cambiando solo el .env |
| 2026-05-21 | FastAPI sigue convenciones de ticket-management: `/api/v1/`, camelCase, error con `{status,error,message,timestamp}` | Consistencia del ecosistema — mismos patrones entre todos los servicios |
| 2026-05-21 | Lifespan carga embedder+FAISS+LLM+grafo una sola vez al arrancar la API | El modelo ONNX pesa 670MB — cargarlo por request sería inaceptable |
| 2026-05-21 | Logs JSON con `pythonjsonlogger` (mismo formato que `langchain-agent`) | Parseable en Loki; campos `levelname`, `name`, `message` filtrables en Grafana |
| 2026-05-21 | Métricas con `prometheus-fastapi-instrumentator` en `/metrics` | Mismo patrón que `langchain-agent` y `langchain-api`; scrapeado por Prometheus en `ticket-management-network` |
| 2026-05-21 | `entrypoint.sh` verifica `index.faiss` antes de arrancar uvicorn | Sin FAISS el contenedor fallaría silenciosamente en el lifespan; mejor fallo rápido con mensaje accionable |
| 2026-05-21 | `PYTHONUNBUFFERED=1` en Dockerfile | Garantiza que los logs llegan a stdout/journald sin buffering — crítico para que Promtail los capture en tiempo real |
| 2026-05-21 | `start.sh` con aviso interactivo si falta índice FAISS | Previene arrancar el contenedor sin datos; mismo patrón de arranque que el resto del ecosistema |
| 2026-05-21 | Rate limiting 10 req/min en `POST /query` con `slowapi` | `POST /query` invoca el LLM — sin límite podría saturar Ollama o generar costos en OCI GenAI; 10/min es suficiente para uso legítimo dado que cada llamada tarda 7-12s |
