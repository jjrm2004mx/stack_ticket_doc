# TAREAS — ticket-agent

Control de actividades por fase. Marcar `[x]` al completar cada tarea.
Última actualización: 2026-06-01 (KB_RUNBOOKS routing completado + re-ingesta 1,638 chunks + verificación confianza 0.8)

---

## AVANCE VS ARQUITECTURA OBJETIVO

> Resumen del estado actual frente al diagrama de arquitectura completa.
> Actualizado: 2026-05-29

### Componentes implementados

| Componente del diagrama | Estado | Implementación |
|---|---|---|
| ONNX Embedding · 1024 dim · FLOAT32 | ✅ | `ONNXEmbedderAdapter` |
| Parent-Child Contextual Chunking | ✅ | `ParentChildChunker` (hijo 512 / padre 2048) |
| Oracle 23ai — Knowledge Repository | ✅ | `KB_CHUNKS` + `KB_MANUALS` + `KB_INCIDENTS` + `KB_RUNBOOKS` + `KB_ERROR_CATALOG` |
| Hybrid Search (Vector + Keyword) | ✅ | `retrieve_hybrid()` — RRF α=0.7 |
| Property Graph — Mermaid Parser | ✅ | `KB_GRAPH_NODES` / `KB_GRAPH_EDGES` |
| Graph Traversal PGQL / SQL PGQ | ✅ | Nodo `graph_traverse` en LangGraph |
| HyDE · Multi-Query RRF · CRAG logic | ✅ | Nodos en LangGraph |
| Web Tool (CRAG condicional) | ✅ | DuckDuckGo node — falla silenciosa |
| REST API | ✅ | FastAPI puerto 8002 — `/query`, `/health`, `/info`, `/diagnose` |
| Containerización | ✅ | Dockerfile + podman-compose + ticket-management-network |
| Docling — loader multi-formato | ✅ | `docling_loader.py` — PDF/DOCX/PPTX/YAML/URLs · lazy import |
| Routing multi-tabla | ✅ | `_TIPO_TO_TABLE` en `OracleVSAdapter` · soporte: RRF 4 tablas (KB_CHUNKS + KB_MANUALS + KB_INCIDENTS + KB_RUNBOOKS) |
| Error Event + `/diagnose` | ✅ | `ErrorEventRequest` · `DiagnosisResponse` · `POST /api/v1/diagnose` |

### Gaps respecto a la arquitectura objetivo

| Gap | Descripción | Bloquea |
|---|---|---|
| **KB_ERROR_CATALOG poblada** | La tabla existe (V3) pero no hay pipeline de ingesta para errores conocidos. Sin datos, `lookup_catalog` siempre pasa vacío. | Calidad del Diagnostic Agent |
| **KB_MANUALS / KB_INCIDENTS pobladas** | Tablas creadas; requieren documentos PDF/DOCX reales via `--source manuales/incidentes`. | Retrieval multi-tabla efectivo |
| ~~**KB_RUNBOOKS sin routing**~~ | ✅ Resuelto 2026-06-01 — runbooks → KB_RUNBOOKS, 4 tablas en `_SOPORTE_TABLES`, re-ingesta 1,352 chunks. Query "¿cómo actualizo la ingesta?" → confianza 0.8. | — |
| **Web URLs en Docling** | `load_from_paths()` no detecta URLs — usa `Path.exists()` y falla silenciosamente. `load_url()` existe pero no se invoca desde el pipeline CSV. Fix: detectar `http` antes de crear `Path`. | Ingesta desde wikis, docs online |
| **CORSMiddleware** | Orígenes permitidos hardcodeados o ausentes. Baja urgencia: APIs backend-to-backend. | — |
| **Contenedores sin root** | Dockerfiles sin usuario `appuser`. Riesgo mitigado por Podman rootless. | — |

### Orden de implementación — estado actual

```
✅ 1. Docling          → loader multi-formato (PDF/DOCX/YAML/URLs)
✅ 2. Tablas Oracle    → V3: KB_MANUALS, KB_RUNBOOKS, KB_INCIDENTS, KB_ERROR_CATALOG
✅ 3. Routing          → add_chunks() y retrieve() con tabla por tipo · soporte multi-tabla
✅ 4. Error Event      → POST /api/v1/diagnose · ErrorEventRequest · DiagnosisResponse
✅ 5. JSON mode        → PydanticOutputParser — campos tipados, sin parsing heurístico
✅ 6. Diagnostic Agent → DiagnosticState + 5 nodos LangGraph · build_query/retrieve_docs/lookup_catalog/assemble_context/generate_diagnosis
✅ 7. Remediation Agent→ RemediationOutput (pasos/verificaciones/estimacion_minutos) · nodo generate_remediation · DiagnosisResponse extendido
```

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
**Estado:** ✅ COMPLETADO (2026-05-23) — ingesta + retrieval + filtro de rol validados end-to-end.

### Oracle 26ai (infraestructura)

- [x] Crear instancia Oracle 26ai — Autonomous AI Database Serverless (OCI Always Free)
      Display name: ticket-agent-kb · DB name: TICKETAGENT · Workload: Transaction Processing
      mTLS: OFF (conexión TLS simple, sin wallet) · región: us-chicago-1
- [x] Crear usuario AGENTE con roles CONNECT + RESOURCE + acceso web habilitado
- [x] Crear tabla `KB_CHUNKS` con columna `VECTOR(1024, FLOAT32)` e índice HNSW
- [x] Crear tabla `KB_INGEST_LOG` para trazabilidad de ingestas
- [x] Crear índice Oracle Text (`idx_kb_content_text`) para Hybrid Search BM25
- [x] Crear vista `V_KB_LEAF_CHUNKS` (chunks hoja — usada por el adapter)
- [x] Descargar wallet (mTLS requerido en Always Free — sin ACL no se puede desactivar)
      Wallet en WSL: ~/.oci/wallet_ticketagent/
- [x] Verificar conectividad desde Python con `oracledb` desde WSL
      Resultado: "Versión Oracle: 23.26.2.2.0" ✅
      Parámetros: config_dir + wallet_location + wallet_password

### Oracle Vector Store adapter + Factory pattern

> Factory pattern se implementa aquí, no antes — el acoplamiento actual (FAISS en main.py)
> es mínimo y no bloquea Fase 3.

- [x] Implementar `OracleVSAdapter` en `src/vector_store/oracle_store.py`
      `add_chunks()` (DELETE + INSERT + KB_INGEST_LOG) · `load()` (count check) · `retrieve()` (VECTOR_DISTANCE + JSON_EXISTS)
      Filtro de rol pre-retrieval en SQL — sin over-fetch de Python
- [x] Implementar `VectorStoreFactory` en `src/vector_store/factory.py`
      `create(settings, embedder)` → FAISSVectorStore | OracleVSAdapter según VECTOR_STORE
- [x] Actualizar `main.py` para usar `VectorStoreFactory` — eliminado import directo de FAISS
- [x] Agregar `oracledb>=2.0,<3.0` a `requirements.txt`
- [x] Agregar variables Oracle a `config.py` y `.env.example`
- [x] Test: `python -m src.main ingest --source all` con VECTOR_STORE=oracle → 1233 chunks en KB_CHUNKS ✅
- [x] Test: `python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario` → respuesta paso a paso correcta ✅
- [x] Test: `python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager` → filtro JSON_EXISTS activo ✅
- [x] Test: `python -m src.main query --query "¿Dónde se configura el modelo LLM?" --rol soporte` → sin filtro de rol ✅
- [x] Fix: `ORA-29861` — `IDX_KB_CONTENT_TEXT` en estado FAILED por ingesta interrumpida → `DROP INDEX`
- [x] Fix: `TypeError` en `retrieve()` — Oracle retorna JSON nativo ya como `list`, no como `str` → `isinstance` check
- [x] Fix: `ValidationError` en `retrieve()` — Oracle retorna CLOB como `LOB object` → `oracledb.defaults.fetch_lobs = False`

### OCI GenAI adapter

- [x] Implementar `OCILLMAdapter` (implementado y validado en Fase 1 — incluye temperature, max_tokens)
- [x] Validar autenticación con `~/.oci/config` (fix WSL: key_file `/home/jjrm` en lugar de `/root`)
- [x] Test: query contra `cohere.command-r-plus-08-2024` — latencia ~10-12s ✓

### Parent-Child chunking

- [x] Implementar `ParentChildChunker` en `chunking.py`
  - Hijo: 256-512 tokens (para retrieval)
  - Padre: ~2048 tokens (para contexto full)
- [x] Re-ingestar documentos con nueva estrategia
      1557 chunks (306 padres + 1251 hijos) en Oracle 23ai ✅
- [x] Test: recuperar chunk hijo + parent completo
      `tests/test_chunking.py::test_parent_child_*` — 6 casos unitarios ✅
      Query "¿Qué flujo sigue un ticket...?" --rol usuario → respuesta con 4 estados completos ✅

### Hybrid Search

- [x] Agregar BM25 reranking sobre resultados vector search
      `retrieve_hybrid()` en `oracle_store.py`: vector over-fetch → Oracle Text CONTAINS → RRF merge
- [x] Combinar scores con RRF (k=60): α/(k+vec_rank+1) + (1-α)/(k+bm25_rank+1)
      `HYBRID_ALPHA=0.7` (configurable en .env) — fallback a vector puro si Oracle Text no disponible
- [x] Test: hybrid search mejora relevancia vs solo vector
      `Oracle hybrid RRF: 3 docs (vec=20, bm25=20, α=0.7, rol=usuario)` ✅
      Query "¿Qué estados tiene un ticket?" → 4 estados completos (ABIERTO, EN PROGRESO, EN REVISIÓN, CERRADO) ✅
      Fix aplicado: query Oracle Text con OR explícito (`" OR ".join(words)`) — AND implícito daba bm25=0
      Fix adicional: `RETRIEVAL_K=6` necesario con parent-child para cubrir documentos largos partidos en 2 padres

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

- [x] Implementar HyDE (Hypothetical Document Embedding)
  - Nodo `hyde` en LangGraph: LLM genera doc hipotético → se usa como query de embedding
  - `HYDE_ENABLED=false` en `.env` (opt-in); compatible con Multi-Query y Property Graph
  - Validado en WSL: respuesta correcta (4 pasos detallados), bm25=24, vec=24 ✅
  - Trade-off: +~20s de latencia (1 llamada LLM extra); desactivar con `HYDE_ENABLED=false` si prioriza velocidad
- [x] Implementar Multi-Query con RRF reranking
  - `make_multi_query_node(retriever, llm, n)` en `nodes.py`
  - Genera N variantes con el LLM → N búsquedas FAISS → RRF (k=60) → top-K chunks
  - Configurable: `MULTI_QUERY_N=3` en `.env` (default 3; valor 1 desactiva y usa búsqueda simple)
  - Prompt de variantes en `src/rag/prompts.py` (`format_multi_query_prompt`)
- [x] Validar mejora de calidad: query manager antes devolvía respuesta incompleta; con Multi-Query devuelve paso a paso completo (latencia +4s, aceptable)

### CRAG (Corrective RAG)

- [x] Implementar nodo `web_fetch` real — búsqueda web cuando confianza baja
      `make_web_fetch_node(enabled, max_results)` con DuckDuckGo (sin API key)
      `CRAG_ENABLED=false` en `.env` (opt-in); `CRAG_MAX_RESULTS=3`
      Context Assembly: contexto local + fragmentos web comprimidos (600 chars/resultado)
      Fallo silencioso si red no disponible o `duckduckgo-search` no instalado
- [x] Combinar contexto local + web antes de `generate`
      Bloque "--- Resultados web (CRAG) ---" appendeado al contexto existente
      `web_sources: List[str]` en `AgentState` — expuesto en CLI y API (`fuentesWeb`)
- [x] Test: query fuera del dominio activa web_fetch; query normal no lo activa
      `tests/test_crag.py` — 11 casos: routing condicional (5) + nodo desactivado (3) + nodo activo (6)

### Graph Traversal *(requiere Fase 2 — Oracle 23ai)*

- [x] Implementar traversal sobre Property Graph con SQL/PGQ
- [x] Conectar como nodo adicional del StateGraph

---

## FASE 4 — Docling + REST API + Mermaid

**Objetivo:** Ingestar documentos complejos y exponer API REST.

### Independiente de Oracle — puede empezar en paralelo con Fase 3

- [x] Integrar Docling como loader multi-formato (PDF, DOCX, PPTX, YAML, Web URLs)
      `src/document_processing/docling_loader.py` — loader con importación lazy de Docling
      `load_manuales()` (rol: soporte+admin) y `load_incidentes()` (rol: soporte)
      `SOURCE_MANUALES` y `SOURCE_INCIDENTES` en config/env — vacíos por defecto
      CLI: `--source manuales` y `--source incidentes` agregados al comando ingest
      Backward compatible: `--source all` sigue funcionando igual; .md se lee directo sin overhead Docling
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
- [x] Fix: verificación FAISS en `entrypoint.sh` y `start.sh` condicional a `VECTOR_STORE` — soporta modo Oracle sin índice local
- [x] Fix: `docker-compose.yml` expone `VECTOR_STORE` y variables Oracle al contenedor; monta `~/.oci` para OCI/Oracle
- [x] Fix: `docker-compose.yml` agrega `image: ticket-agent` — evita conflicto entre `podman build` y `podman-compose` (imágenes distintas)
- [x] Fix: `Dockerfile` pre-instala torch CPU-only antes de `requirements.txt` — evita descarga de ~3 GB de paquetes CUDA innecesarios
- [x] Fix: `src/api/main.py` usaba `FAISSVectorStore` directamente ignorando `VectorStoreFactory` — corregido para usar la factory
- [x] Fix: `OCILLMAdapter` no pasaba `auth_file_location` a `ChatOCIGenAI` — `OCI_CONFIG_FILE` del settings ahora se propaga
- [x] Fix: `~/.oci/config.container` con rutas `/root/.oci/...` para el contenedor — evita error `InvalidKeyFilePath`
- [x] Fix: `ORACLE_CONFIG_DIR` y `ORACLE_WALLET_DIR` en `.env` deben usar rutas del contenedor (`/root/.oci/wallet_ticketagent`), no rutas del host WSL — el compose monta `~/.oci → /root/.oci:ro`; el proceso Python dentro del contenedor solo ve `/root/`, nunca `/home/jjrm/`
- [x] Test: `GET /api/v1/health → 200 OK` validado con contenedor + Oracle 23ai + OCI GenAI (2026-05-24) ✅
- [x] Crear runbook `docs/runbooks/06_deploy_contenedor.md` — checklist de primer deploy y diagnóstico rápido

### Requiere Oracle (Fase 2 completa)

- [x] Implementar Mermaid parser → Property Graph en Oracle 23ai
      `docs/arquitectura/ecosistema.mmd` → `mermaid_parser.py` → `KB_GRAPH_NODES` / `KB_GRAPH_EDGES`
      `CREATE PROPERTY GRAPH ticket_sys_graph` con SQL/PGQ en Oracle 23ai ✅
      Nodo `graph_traverse` en LangGraph — activo con keywords de arquitectura/dependencia ✅
      Migración Flyway: `db/migrations/oracle/V2__add_graph_tables.sql` ✅
      Validado en WSL: `ingest --source graph` → 25 nodos + 30 aristas persistidos ✅
      Query "¿Qué servicios usa ticket_agent?" → Graph traverse: 5 relaciones ✅
      Query "¿Qué impacto si cae classifier_redis?" → Graph traverse: 2 relaciones + análisis de dependencias ✅
- [x] En containerización: eliminar volumen `faiss_data` del compose — Oracle 23ai es el store activo; `ORACLE_DSN` ya expuesto como variable de entorno
- [x] Crear tablas Oracle especializadas — migración Flyway V3
      `db/migrations/oracle/V3__add_specialized_tables.sql`
      KB_MANUALS (DOC_FORMAT, DOC_VERSION) · KB_RUNBOOKS (SERVICE, RUNBOOK_TYPE)
      KB_INCIDENTS (INCIDENT_DATE, SEVERITY, STATUS, SERVICE)
      KB_ERROR_CATALOG (ERROR_CODE, SERVICE, SEVERITY, REMEDIATION) ← alimenta Diagnostic Agent
      HNSW + Oracle Text + vistas leaf en cada tabla
      KB_CHUNKS.chk_tipo ampliado para aceptar 'manuales' e 'incidentes' (temporalmente)
      KB_INGEST_LOG: columna TARGET_TABLE agregada para trazabilidad por tabla destino
      Aplicado en Oracle 23ai (2026-05-29) ✅
- [x] Routing de tablas en OracleVSAdapter
      `_TIPO_TO_TABLE`: manuales→KB_MANUALS, incidentes→KB_INCIDENTS, resto→KB_CHUNKS
      `add_chunks()`: agrupa por tipo y hace DELETE+INSERT en la tabla correspondiente
      `_do_insert()`: SQL específico por tabla (KB_CHUNKS con TIPO/PANTALLA; KB_MANUALS con DOC_FORMAT; KB_INCIDENTS)
      `retrieve()` soporte: multi-tabla RRF Python — KB_CHUNKS + KB_MANUALS + KB_INCIDENTS + KB_RUNBOOKS (2026-06-01)
      `_vec_search()`: helper de búsqueda vectorial por tabla con parent expansion
      `load()`: cuenta chunks en todas las tablas activas (_SOPORTE_TABLES)
      BM25 (_bm25_rank): opera sobre KB_CHUNKS — chunks especializados contribuyen via path vectorial
- [x] Error Event schema + endpoint POST /api/v1/diagnose
      `ErrorEventRequest` (errorCode, service, severity, message, stackTrace)
      `DiagnosisResponse` (causa, remediacion, runbook, fuentes, confianza, latenciaMs, servicio, severidad)
      `DIAGNOSE_PROMPT` + `format_diagnose_prompt()` + `_parse_diagnose_response()` en prompts.py
      `src/api/routes/diagnose.py` — usa retriever directo (rol soporte) + LLM con prompt estructurado
      `app.state.retriever` y `app.state.llm` expuestos en lifespan (antes solo estaba `graph`)
      Output: secciones CAUSA / REMEDIACION / RUNBOOK parseadas desde texto LLM
      Nota: Step 5 reemplazará el parsing heurístico por JSON mode del LLM

---

## ✅ COMPLETADO — Routing KB_RUNBOOKS (2026-06-01, rama: feature/kb-runbooks-routing)

**Problema resuelto:** los runbooks operacionales iban a KB_CHUNKS mezclados con docs de usuario.
El agente respondía con confianza 0.0 a preguntas sobre re-ingesta y operación del stack.

**Evidencia antes:** query `"¿Cambié la documentación, cómo actualizo la ingesta?"` con `rol=soporte`
→ `"Lo siento, no tengo información disponible sobre este procedimiento"` (confianza 0.0).

**Evidencia después:** misma query → respuesta con fuente `03_guia_operacional.md` (confianza 0.8).

**Cambios implementados en `src/vector_store/oracle_store.py`:**

- [x] `_TIPO_TO_TABLE`: `"runbooks": "KB_CHUNKS"` → `"runbooks": "KB_RUNBOOKS"`
- [x] `_SOPORTE_TABLES`: agregado `"KB_RUNBOOKS"` — ahora 4 tablas en retrieval soporte
- [x] `_do_insert()`: branch `elif table == "KB_RUNBOOKS"` con `_service_from_path()` y `_runbook_type_from_path()`
- [x] `_vec_search()`: branch `elif table == "KB_RUNBOOKS"` con query vectorial + parent expansion
- [x] Re-ingesta: `python -m src.main ingest --source all` → 286 KB_CHUNKS + 1,352 KB_RUNBOOKS
- [x] Verificación: confianza 0.8 confirmada con fuente `docs/runbooks/03_guia_operacional.md`

**Nota:** BM25 (`_bm25_rank`) sigue operando sobre KB_CHUNKS — KB_RUNBOOKS contribuye
al score final via el path vectorial, mismo patrón que KB_MANUALS e KB_INCIDENTS.

---

## FASE 5 — Consola Standalone (operaciones de emergencia)

**Objetivo:** Consola de diagnóstico que funciona aunque ticket-management esté caído.
Servida directamente por ticket-agent como archivo estático en `/ui`.

**Estado:** Pendiente — se implementa después de Fase 1 en ticket-management.

### Decisiones de diseño

- Un solo archivo `static/index.html` — HTML/CSS/JS puro, sin frameworks, sin CDN externo
- Sin dependencias de red excepto `ticket-agent:8002` — funciona en red aislada
- Llama directamente a `POST /api/v1/query` y `POST /api/v1/diagnose` en el mismo host
- Auth: API key de operaciones (no JWT de ticket-management)
- Interfaz mínima: chat RAG + formulario de diagnóstico manual

### Tareas

- [ ] Montar `StaticFiles` en FastAPI: `app.mount("/ui", StaticFiles(directory="static", html=True))`
- [ ] Crear `static/index.html` — consola standalone (HTML/CSS/JS inline, sin CDN)
  - Chat RAG conectado a `POST /api/v1/query` con rol=soporte
  - Formulario de diagnóstico: errorCode, service, severity, message, stackTrace
  - Indicador de estado: `/api/v1/health` al cargar
  - Sin Font Awesome ni Google Fonts — iconos Unicode
- [ ] Agregar variable `UI_ENABLED=true` en `.env.example` (opt-in)
- [ ] Documentar acceso en `docs/runbooks/06_deploy_contenedor.md`

---

## ANÁLISIS — Redis compartido con ticket-classification

> Análisis técnico de viabilidad. No es una tarea activa — se retoma cuando se implemente rate limiting distribuido o jobs asíncronos.

### Estado actual

| Servicio | Redis | Para qué | Keys | TTL |
|---|---|---|---|---|
| `langchain-api` | `classifier-redis` (puerto 6381) | Caché de respuestas LLM | `{sha256_hash}` (sin prefijo) | 1 hora |
| `langchain-agent` | `classifier-redis` (puerto 6381) | Estado de jobs async `/process` → `/status` | `job:{UUID}` | 24 horas |
| `ticket-agent` | **ninguno** | — | — | — |

ticket-agent usa `slowapi` con almacenamiento en memoria para rate limiting. Se reinicia con el contenedor y no funciona si hubiera más de una réplica.

### Casos de uso donde Redis aportaría valor

**1. Rate limiting distribuido (valor medio)**
- Problema actual: el limitador de `slowapi` es por-instancia; 2+ réplicas tendrían contadores separados.
- Solución: mover el backend de `slowapi` a Redis con clave `agent:rate_limit:{IP}` (TTL ~60s, atómico).

**2. Jobs asíncronos (valor medio-alto)**
- Si se quisiera un patrón `POST /query/async → job_id` + `GET /results/{job_id}` (espejo de langchain-agent), Redis sería necesario.
- Key propuesta: `agent:job:{UUID}` (TTL 86400s).

### Viabilidad de compartir `classifier-redis`

**Seguro con prefijos.** Sin prefijos hay riesgo de colisión porque `langchain-api` guarda hashes sin namespace.

Asignación de namespaces propuesta:

```
classifier:cache:{sha256}     # langchain-api  (TTL 3600s)
classifier:job:{UUID}         # langchain-agent (TTL 86400s)
agent:rate_limit:{IP}         # ticket-agent    (TTL 60s)
agent:job:{UUID}              # ticket-agent async (TTL 86400s, futuro)
```

Cambios necesarios si se comparte:
- `ticket-classification`: prefijar keys en `langchain-api/main.py` y `langchain-agent/main.py` (2 archivos, ~3 líneas cada uno).
- `ticket-agent`: añadir `redis>=4.0,<6.0` a `requirements.txt`, variables `REDIS_HOST/PORT/PASSWORD` al `.env.example`.
- Memoria estimada total: < 50 MB (manejable en cualquier instancia Redis 7.x).

### Alternativa: mantener Redis separado en producción

Si el volumen de jobs crece o Redis se convierte en SPOF, escalar a instancia dedicada es trivial: solo cambiar `REDIS_HOST` en `.env` de ticket-agent.

### Tareas (cuando se decida implementar)

- [ ] Prefijar keys en `langchain-api` (`classifier:cache:`) y `langchain-agent` (`classifier:job:`)
- [ ] Añadir `redis>=4.0,<6.0` a `requirements.txt` de ticket-agent
- [ ] Configurar backend Redis en `slowapi` para rate limiting distribuido
- [ ] Exponer `classifier-redis` a `ticket-management-network` (actualmente solo accesible dentro de `ticket-classification`)

---

## MEJORAS DE ECOSISTEMA (afectan todos los repos Python)

> Estas tareas no son exclusivas de ticket-agent. Se registran aquí para visibilidad.
> Implementarlas requiere cambios coordinados en `langchain-agent` y `langchain-api` también.

- [x] **X-Request-ID / Correlation ID** — inyectar header `X-Request-ID` en cada request y propagarlo en los logs JSON.
      `RequestIdMiddleware` en `src/api/middleware.py` (Starlette BaseHTTPMiddleware)
      `ContextVar` en `src/utils/request_id.py` — aislado por async task, sin colisión entre requests concurrentes
      `_RequestIdFilter` en `src/utils/logger.py` — agrega `request_id` a cada línea JSON automáticamente
      Header devuelto en la respuesta HTTP — cliente puede correlacionar errores con logs de Loki
- [ ] **Web URLs en Docling** — agregar detección de `http` en `load_from_paths()` antes de crear `Path`:
      ```python
      if path_str.startswith("http"):
          doc = load_url(path_str, rol=rol, tipo=tipo)
          if doc: docs.append(doc)
          continue
      ```
      Después activar con `SOURCE_MANUALES=https://...` en `.env` y correr `--source manuales`.
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
| 2026-05-21 | Redis: no se añade en Fase 4; se evalúa si se escala horizontalmente o se añaden jobs async | `slowapi` en memoria es suficiente para instancia única; compartir `classifier-redis` es viable con prefijos de namespace — ver sección "ANÁLISIS — Redis compartido" |
| 2026-05-23 | `oracledb.defaults.fetch_lobs = False` a nivel de módulo en `oracle_store.py` | Oracle retorna CLOB como objeto `LOB` — LangChain `Document.page_content` requiere `str`; configuración global evita casting manual en cada fila |
| 2026-05-23 | `ROL_JSON` retornado por Oracle como `list` nativa, no como JSON string | Oracle tipo `JSON` nativo devuelve el valor ya deserializado; `json.loads()` falla si recibe una lista — fix con `isinstance(rol_json, list)` |
| 2026-05-23 | `IDX_KB_CONTENT_TEXT` (Oracle Text) no se recrea tras el DROP en Fase 2 | El índice falló por ingesta interrumpida; se necesita solo en Fase 2 (Hybrid Search) — se recrea cuando se implemente `retrieve_hybrid()` |
| 2026-05-27 | `ORACLE_CONFIG_DIR` y `ORACLE_WALLET_DIR` usan rutas distintas según el contexto de ejecución | WSL directo (ingest CLI): `/home/jjrm/.oci/wallet_ticketagent`; contenedor: `/root/.oci/wallet_ticketagent` — el compose monta `~/.oci → /root/.oci:ro` |
| 2026-05-29 | El `.env` activo usa rutas de contenedor (`/root/.oci/...`) — no rutas WSL | El modo principal de ejecución es podman-compose; el ingest CLI desde WSL puede sobrescribir con `ORACLE_CONFIG_DIR=... python -m src.main ingest` si necesita rutas del host |
| 2026-05-29 | Consola standalone (Fase 5) vive en ticket-agent como static file | Herramienta de emergencia: si ticket-management no levanta, la consola de diagnóstico no puede depender de él; ticket-agent la sirve en `/ui` con FastAPI StaticFiles |
