# ticket-agent — Resumen Ejecutivo Técnico
*Corte: Junio 2026*

## ¿Qué es?

Agente RAG (*Retrieval-Augmented Generation*) del ecosistema de tickets. Responde preguntas en español sobre cómo usar el sistema (usuarios finales) y sobre la operación técnica del stack completo (personal de soporte). No inventa — fundamenta cada respuesta en documentación real.

---

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| **Vector store** | Oracle 23ai — Autonomous AI Database (OCI Always Free) |
| **Búsqueda** | Hybrid Search: HNSW vectorial + BM25 Oracle Text con RRF |
| **Embeddings** | ONNX local `mxbai-embed-large-v1` — 1024 dims, FLOAT32, sin API externa |
| **Orquestación** | LangGraph StateGraph — pipeline condicional de 6 nodos |
| **LLM** | OCI GenAI (`cohere.command-r-plus-08-2024`) |
| **API** | FastAPI REST — puerto 8002 |
| **Runtime** | Podman rootless (WSL2) |

---

## Pipeline de inteligencia

```
Pregunta / Error Event + rol
    │
    ▼
[HyDE]          → LLM genera documento hipotético para mejorar el embedding
    │
    ▼
[Multi-Query]   → 3 variantes de la pregunta → Hybrid Search → RRF reranking
    │
    ▼
[Graph Traverse]→ Si pregunta sobre arquitectura: SQL/PGQ sobre Property Graph Oracle
    │
    ▼
[Generate]      → Prompt + contexto → LLM → respuesta
    │
    ▼
[Validate]      → Score de confianza 0.0 / 0.3 / 0.8
    │
  ≥ 0.5 → respuesta final
  < 0.5 → [CRAG] búsqueda web DuckDuckGo → regenerar (máx 1 vez)
```

---

## Base de conocimiento — Oracle 23ai

| Tabla | Contenido | Estado | Chunks |
|---|---|---|---|
| `KB_CHUNKS` | Docs por rol + ayuda contextual UI | ✅ Activa | 286 |
| `KB_RUNBOOKS` | Runbooks operacionales de 6 repos | ✅ Activa | 1,352 |
| `KB_MANUALS` | Manuales técnicos PDF/DOCX (Docling) | ⏳ Sin datos | — |
| `KB_INCIDENTS` | Incidentes históricos PDF/DOCX | ⏳ Sin datos | — |
| `KB_ERROR_CATALOG` | Catálogo errores → Diagnostic Agent | ⏳ Sin datos | — |
| `KB_GRAPH_NODES/EDGES` | Property Graph del ecosistema | ✅ Activa | 25 nodos / 30 aristas |

---

## Control de acceso por rol

| Rol | Accede a |
|---|---|
| `usuario` | Documentación de usuario de ticket-management |
| `manager` | Documentación manager + aprobaciones |
| `admin` | Todo lo anterior + gestión del sistema |
| `soporte` | Todo — incluyendo runbooks técnicos de los 6 repos y Property Graph |

Filtrado **pre-retrieval en SQL** con `JSON_EXISTS` — sin over-fetch en Python.

---

## Estado vs arquitectura objetivo

### ✅ Implementado y funcional (80%)

| Componente | Detalle |
|---|---|
| Parent-Child Chunking | 512 tok hijo / 2048 tok padre |
| ONNX Embedding local | 1024 dims, FLOAT32, sin API externa |
| Oracle 23ai Vector Tables | KB_CHUNKS + KB_RUNBOOKS con HNSW + Oracle Text |
| Property Graph | Mermaid → KB_GRAPH Oracle SQL/PGQ |
| LangGraph Agentic RAG | HyDE · Multi-Query RRF · CRAG real (DuckDuckGo) |
| Hybrid Search | HNSW vectorial + BM25 Oracle Text + RRF |
| Graph Traversal | SQL/PGQ — impacto en cascada entre servicios |
| Error Event | `POST /api/v1/diagnose` — `ErrorEventRequest` |
| Diagnostic Agent | DiagnosticState + 5 nodos LangGraph |
| Remediation Agent | `RemediationOutput` — pasos, verificaciones, estimación |

### ⏳ Pendiente (20%)

| Componente | Motivo |
|---|---|
| **KB_MANUALS** | Requiere documentos PDF/DOCX reales — `SOURCE_MANUALES` vacío en `.env` |
| **KB_INCIDENTS** | Requiere documentos PDF/DOCX reales — `SOURCE_INCIDENTES` vacío en `.env` |
| **KB_ERROR_CATALOG** | Sin pipeline de ingesta — `lookup_catalog` retorna vacío, afecta calidad del Diagnostic Agent |
| **Docling en producción** | El loader existe y está validado — bloqueado por falta de documentos fuente |
| **Context Assembly < 8K tokens** | Fusión de contexto implementada — límite explícito de compresión pendiente |

---

## Diferenciadores técnicos

- **Hybrid Search real** — combina similitud vectorial HNSW + BM25 keyword con Reciprocal Rank Fusion
- **Parent-Child chunking** — recupera fragmentos pequeños (512 tok) pero entrega contexto completo al LLM (2048 tok)
- **Property Graph Oracle 23ai** — consultas SQL/PGQ sobre relaciones entre servicios del ecosistema
- **CRAG implementado** — si la confianza local es baja, busca en DuckDuckGo y fusiona al contexto
- **Filtrado por rol en Oracle** — la seguridad vive en la query SQL, no en lógica de aplicación
- **KB segmentada por tipo** — tablas dedicadas por naturaleza del conocimiento (runbooks, manuales, incidentes)

---

*ticket-agent · Junio 2026*
