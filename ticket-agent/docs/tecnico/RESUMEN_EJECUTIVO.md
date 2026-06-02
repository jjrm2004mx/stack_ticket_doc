# ticket-agent — Resumen Ejecutivo Técnico

## ¿Qué es?

Agente RAG (*Retrieval-Augmented Generation*) del ecosistema de tickets. Responde preguntas en español sobre cómo usar el sistema (usuarios finales) y sobre la operación técnica del stack completo (personal de soporte). No inventa — fundamenta cada respuesta en documentación real.

---

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| **Vector store** | Oracle 23ai — Autonomous AI Database (OCI Always Free) |
| **Búsqueda** | Hybrid Search: HNSW vectorial + BM25 Oracle Text con RRF |
| **Embeddings** | ONNX local `mxbai-embed-large-v1` — 1024 dims, FLOAT32 |
| **Orquestación** | LangGraph StateGraph — pipeline condicional de 6 nodos |
| **LLM** | OCI GenAI (`cohere.command-r-plus-08-2024`) |
| **API** | FastAPI REST — puerto 8002 |
| **Runtime** | Podman rootless (WSL2) |

---

## Pipeline de inteligencia

```
Pregunta + rol
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

| Tabla | Contenido | Chunks |
|---|---|---|
| `KB_CHUNKS` | Docs por rol + ayuda contextual UI | 286 |
| `KB_RUNBOOKS` | Runbooks operacionales de 6 repos | 1,352 |
| `KB_MANUALS` | Manuales técnicos PDF/DOCX (Docling) | pendiente |
| `KB_INCIDENTS` | Incidentes históricos | pendiente |
| `KB_ERROR_CATALOG` | Catálogo errores → Diagnostic Agent | pendiente |
| `KB_GRAPH_NODES/EDGES` | Property Graph del ecosistema | 25 nodos / 30 aristas |

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

## Fases completadas

| Fase | Logro |
|---|---|
| ✅ Fase 1 | FAISS + ONNX + OCI GenAI — CLI funcional |
| ✅ Fase 2 | Oracle 23ai + Parent-Child chunking + Hybrid Search |
| ✅ Fase 3 | LangGraph: HyDE, Multi-Query RRF, Property Graph, CRAG real |
| ✅ Fase 4 | FastAPI REST + contenedor Podman + Prometheus + rate limiting |
| ✅ Fase 5 (parcial) | KB_RUNBOOKS routing — 1,352 runbooks en tabla dedicada, retrieval soporte activo |

---

## Diferenciadores técnicos

- **Hybrid Search real** — no solo embeddings: combina similitud vectorial HNSW + BM25 keyword con Reciprocal Rank Fusion
- **Parent-Child chunking** — recupera fragmentos pequeños (512 tok) pero entrega contexto completo al LLM (2048 tok)
- **Property Graph Oracle 23ai** — consultas SQL/PGQ sobre relaciones entre servicios del ecosistema
- **CRAG implementado** — si la confianza local es baja, busca en DuckDuckGo y fusiona al contexto
- **Filtrado por rol en Oracle** — la seguridad vive en la query SQL, no en lógica de aplicación
- **KB segmentada por tipo** — tablas dedicadas por naturaleza del conocimiento (runbooks, manuales, incidentes)

---

*ticket-agent · Junio 2026*
