# TAREAS — ticket-agent

Control de actividades por fase. Marcar `[x]` al completar cada tarea.
Última actualización: 2026-05-21 (Fase 1 implementada por Claude Code)

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
- [ ] Testear carga desde `.env` con `python -c "from src.config import Settings; print(Settings())"`

### Adapters base (`src/adapters/`)

- [x] Implementar `LLMAdapter` (ABC) con `invoke()` y `can_cache()`
- [x] Implementar `EmbedderAdapter` (ABC) con `embed_documents()` y `embed_query()`
- [x] Implementar `ONNXEmbedderAdapter` — modelo `mxbai-embed-large-v1`, 1024 dims
- [ ] Verificar que `ONNXEmbedderAdapter.embed_query()` devuelve lista de 1024 floats
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
- [ ] Test: guardar índice y volver a cargarlo sin pérdida de datos

### RAG (`src/rag/`)

- [x] Implementar `Retriever` con `retrieve(query) → list[LangDoc]`
- [x] Implementar `prompts.py` con `SYSTEM_PROMPT` y `RAG_PROMPT`
- [x] Formatear prompt final: system + contexto + pregunta (`format_prompt()`)

### CLI (`src/main.py`)

- [x] Implementar comando `ingest --source <all|usuarios|runbooks|help>`
  - `all`: ingesta las tres fuentes
  - `usuarios`: solo `ticket-management/docs/usuarios/`
  - `runbooks`: todos los `docs/runbooks/` de los 5 repos
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

- [ ] Ejecutar `python -m src.main ingest --source all`
- [ ] Query rol usuario: `python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario`
  - Respuesta basada en `ticket-management/docs/usuarios/usuario/flujos.md`
- [ ] Query rol manager: `python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager`
  - Respuesta basada en `docs/usuarios/manager/flujos.md` y helpContent draftReview
- [ ] Query rol soporte: `python -m src.main query --query "¿Cómo está configurado n8n en ticket-ingestion-light?" --rol soporte`
  - Respuesta basada en runbooks de ticket-ingestion-light
- [ ] Verificar que un query con `--rol usuario` NO retorna contenido de runbooks técnicos
- [ ] Latencia < 10s por query (Ollama local)

---

## FASE 2 — Multi-provider + Oracle 23ai

**Objetivo:** Reemplazar FAISS por Oracle 23ai Free y agregar OCI GenAI como LLM.

### Oracle 23ai (infraestructura)

- [ ] Agregar `oracle/database-free:23.6-slim` a `podman-compose.yml`
- [ ] Crear script SQL inicial: tablas vectoriales `KB_MANUALS`, `KB_RUNBOOKS`
- [ ] Configurar Hybrid Vector Index (HNSW + Oracle Text) en cada tabla
- [ ] Verificar conectividad desde Python con `oracledb`

### Oracle Vector Store adapter

- [ ] Implementar `OracleVSAdapter` con la misma interfaz que `FAISSVectorStore`
  - `add_chunks()`, `load()`, `retrieve()`
- [ ] Usar `langchain-community` `OracleVS` o implementación directa con `oracledb`
- [ ] **Verificar que EMBED_DIMS=1024 y tipo FLOAT32** en la columna vectorial
- [ ] Test: migrar índice FAISS existente a Oracle sin cambiar el modelo ONNX

### Parent-Child chunking

- [ ] Implementar `ParentChildChunker` en `chunking.py`
  - Hijo: 256-512 tokens (para retrieval)
  - Padre: ~2048 tokens (para contexto full)
- [ ] Re-ingestar documentos con nueva estrategia
- [ ] Test: recuperar chunk hijo + parent completo

### OCI GenAI adapter

- [ ] Implementar `OCILLMAdapter` (ya en Fase 1 como placeholder)
- [ ] Validar autenticación con `~/.oci/config`
- [ ] Test: query contra `meta.llama-3.3-70b-instruct`

### Factory pattern

- [ ] Implementar `LLMAdapterFactory.create(provider, config)`
- [ ] Implementar `EmbedderAdapterFactory.create(provider, config)`
- [ ] Implementar `VectorStoreFactory.create(store_type, embeddings, config)`
- [ ] Actualizar `main.py` para usar factories (eliminar imports directos)

### Hybrid Search

- [ ] Agregar BM25 reranking sobre resultados vector search
- [ ] Combinar scores (RRF o weighted sum)
- [ ] Test: hybrid search mejora relevancia vs solo vector

---

## FASE 3 — Agentic RAG (LangGraph)

**Objetivo:** Reemplazar LCEL chain por LangGraph con lógica condicional.

- [ ] Implementar `KnowledgeAgent` con LangGraph `StateGraph`
  - Nodo: `retrieve`
  - Nodo: `generate`
  - Nodo: `validate` (score de confianza)
  - Condicional: si confianza baja → `web_fetch`
- [ ] Implementar HyDE (Hypothetical Document Embedding)
- [ ] Implementar Multi-Query con RRF reranking
- [ ] Implementar CRAG (Corrective RAG con web fetch condicional)
- [ ] Implementar Graph Traversal (PGQL/SQL PGQ) sobre Property Graph

---

## FASE 4 — Docling + Mermaid + REST API

**Objetivo:** Ingestar documentos complejos y exponer API REST.

- [ ] Reemplazar loader.py por Docling (PDF, DOCX, YAML, Web URLs)
- [ ] Implementar Mermaid parser → Property Graph en Oracle 23ai
- [ ] Implementar `Diagnostic Agent` y `Remediation Agent`
- [ ] Exponer CLI como REST API con FastAPI
- [ ] Agregar LangSmith para observabilidad

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
