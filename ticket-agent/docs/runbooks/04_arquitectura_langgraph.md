# 04 — Arquitectura LangGraph

Documentación del grafo de agente implementado en `src/agent/`.

---

## ¿Por qué LangGraph?

La chain LCEL de Fase 1 era lineal: recuperar → generar → responder.
LangGraph permite lógica condicional: si la respuesta tiene baja confianza,
el agente puede enriquecer el contexto y reintentar antes de responder.

---

## Estructura del grafo

```
src/agent/
├── state.py    → AgentState — estado compartido entre nodos
├── nodes.py    → funciones de nodo + constantes
└── graph.py    → StateGraph compilado (build_graph)
```

---

## AgentState

```python
class AgentState(TypedDict):
    query      : str            # pregunta original del usuario
    rol        : str            # usuario | manager | admin | soporte
    docs       : List[LangDoc]  # chunks recuperados (post RRF)
    contexto   : str            # texto concatenado de chunks (+ grafo si aplica)
    respuesta  : str            # respuesta generada por el LLM
    confianza  : float          # score 0.0 – 1.0
    ciclos     : int            # número de veces que pasó por web_fetch
    hyde_query : Optional[str]  # doc hipotético generado por HyDE (None si desactivado)
```

---

## Nodos

| Nodo | Factory / Función | Descripción |
|---|---|---|
| `hyde` | `make_hyde_node(llm)` | Genera doc hipotético con el LLM para mejorar el embedding de retrieval. Solo activo si `HYDE_ENABLED=true`. |
| `retrieve` | `make_multi_query_node(retriever, llm, n)` | Genera N variantes de la query → N búsquedas Hybrid Search → RRF reranking |
| `graph_traverse` | `make_graph_traverse_node(oracle_store)` | Enriquece el contexto con relaciones del Property Graph Oracle 23ai. Solo activo si `VECTOR_STORE=oracle` y la query contiene keywords de arquitectura. |
| `generate` | `make_generate_node(llm)` | Formatea el prompt y llama al LLM |
| `validate` | `validate(state)` | Calcula el score de confianza |
| `web_fetch` | `make_web_fetch_node(enabled, max_results)` | CRAG: búsqueda DuckDuckGo cuando confianza < 0.5. Fusiona fragmentos web al contexto local. Solo activo si `CRAG_ENABLED=true`. Falla silenciosa si no hay red. |

> Con `MULTI_QUERY_N=1` el grafo usa `make_retrieve_node` (búsqueda simple sin variantes).
> Con `MULTI_QUERY_N>1` (default: 3) usa `make_multi_query_node`.

---

## Diagrama del grafo (configuración actual)

```
             ┌─────────────────────────────┐
    inicio → │ hyde (opt-in HYDE_ENABLED)  │
             │ LLM genera doc hipotético   │
             │ → hyde_query en el estado   │
             └─────────────┬───────────────┘
                           │
             ┌─────────────▼───────────────┐
             │ retrieve (Multi-Query RRF)  │
             │ usa hyde_query si existe    │
             │ N variantes → Hybrid Search │
             │ → RRF → top-K chunks        │
             └─────────────┬───────────────┘
                           │
             ┌─────────────▼───────────────┐
             │ graph_traverse (opcional)   │  ← solo si VECTOR_STORE=oracle
             │ keywords detectados en query│    y keywords de arquitectura
             │ → SQL/PGQ → KB_GRAPH        │
             │ → añade relaciones al ctx   │
             └─────────────┬───────────────┘
                           │
                    ┌──────▼──────┐
                    │  generate   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  validate   │
                    └──────┬──────┘
                           │
               ┌───────────┴────────────┐
               │                        │
         conf ≥ 0.5               conf < 0.5
         ciclos ≥ 1               ciclos < 1
               │                        │
              END               ┌───────▼───────┐
                                │   web_fetch   │
                                │  (placeholder)│
                                └───────┬───────┘
                                        │
                                 (vuelve a generate)
```

---

## HyDE — Hypothetical Document Embedding

### ¿Qué hace?

Antes de embedear la query del usuario, el LLM genera un párrafo hipotético que
respondería esa pregunta. Ese párrafo (semánticamente más cercano a los documentos
reales del KB) se usa como query de embedding en lugar del texto original.

### Ejemplo

```
Query usuario: "¿Cómo creo un ticket?"

Doc hipotético generado:
"Para crear un ticket, accede a la pantalla principal y haz clic en 'Nuevo Ticket'.
Completa el formulario con título, descripción, tipo y prioridad. Al guardar,
el ticket queda en estado ABIERTO y el equipo de soporte lo recibe de inmediato."

→ embed(doc_hipotético) → búsqueda en KB_CHUNKS
→ más similitud con documentos reales que embed("¿Cómo creo un ticket?")
```

### Trade-off

| | Sin HyDE | Con HyDE |
|---|---|---|
| Latencia | ~23s | ~44s (+1 llamada LLM) |
| Calidad | Buena (Multi-Query cubre bien) | Mejor en queries cortas/ambiguas |

Default: `HYDE_ENABLED=false`. Activar si el retrieval trae documentos poco relevantes.

---

## Multi-Query con RRF reranking

### ¿Qué hace?

Genera N variantes de la pregunta con el LLM y hace una búsqueda por cada una.
Los resultados se fusionan eliminando duplicados y reordenando por RRF.

### Ejemplo

```
Query: "¿Cómo apruebo un borrador?"

Variante 1: "¿Cuál es el proceso para aprobar un ticket en borrador?"
Variante 2: "¿Qué pasos sigue un manager para revisar borradores?"
Variante 3: "¿Dónde apruebo tickets pendientes de revisión?"

→ 4 búsquedas Hybrid Search (query original + 3 variantes)
→ RRF fusiona y reordena
→ Top-K chunks al nodo generate
```

Nota: si `HYDE_ENABLED=true`, la primera búsqueda usa el `hyde_query` en lugar
de la query original. Las variantes siempre se generan desde la query original.

### RRF (Reciprocal Rank Fusion)

```
score(chunk) = Σ  1 / (rank_en_lista_i + 60)
```

Un chunk que aparece en el top-3 de múltiples listas acumula más score.

### Variables de configuración

| Variable | Default | Descripción |
|---|---|---|
| `MULTI_QUERY_N` | `3` | Número de variantes generadas |
| `RETRIEVAL_K` | `6` | Chunks finales que pasan a generate |
| `HYDE_ENABLED` | `false` | Activa el nodo hyde |

---

## graph_traverse — Property Graph

### Activación

El nodo solo ejecuta si:
1. `VECTOR_STORE=oracle` (el store tiene `graph_traverse_query`)
2. La query contiene al menos una keyword del set:
   `{"depende", "dependencia", "usa", "utiliza", "llama", "conecta", "cae", "falla", "impacto", "afecta", "cascada", "arquitectura", "servicios", "infraestructura", "almacena", "guarda"}`

### ¿Qué hace?

1. Carga todos los nodos de `KB_GRAPH_NODES`
2. Detecta qué nodos están mencionados en la query (por label)
3. Para cada nodo mencionado: ejecuta SQL/PGQ en ambas direcciones (outgoing + incoming)
4. Añade las relaciones al campo `contexto` del estado

### Falla silenciosa

Si `KB_GRAPH_NODES` no existe o está vacía, el nodo retorna `{}` sin propagar el error.
Esto permite que el grafo funcione sin el Property Graph (modo FAISS o pre-migración V2).

---

## Lógica de confianza (validate)

| Situación | Confianza |
|---|---|
| Sin documentos recuperados | 0.0 |
| Respuesta contiene frases de baja confianza | 0.3 |
| Respuesta normal | 0.8 |

Frases detectadas: "no tengo esa información", "no dispongo de",
"no está disponible", "no encuentro información", "no puedo responder",
"no hay información".

---

## Routing condicional

```python
UMBRAL_CONFIANZA = 0.5
MAX_CICLOS = 1

def _route(state) -> str:
    if state["confianza"] >= UMBRAL_CONFIANZA:
        return "end"
    if state["ciclos"] >= MAX_CICLOS:
        return "end"
    return "web_fetch"
```

---

## Cómo se construye el grafo

```python
from src.agent.graph import build_graph

graph = build_graph(
    retriever,
    llm,
    multi_query_n=settings.MULTI_QUERY_N,   # default 3
    graph_store=oracle_store,               # None si VECTOR_STORE=faiss
    hyde=settings.HYDE_ENABLED,             # default False
)

result = graph.invoke({
    "query": "¿Cómo apruebo un borrador?",
    "rol": "manager",
    "docs": [],
    "contexto": "",
    "respuesta": "",
    "confianza": 0.0,
    "ciclos": 0,
    "hyde_query": None,
})
```

---

## web_fetch — CRAG implementado (DuckDuckGo)

El nodo `web_fetch` realiza búsqueda web real cuando la confianza local es baja.

```
make_web_fetch_node(enabled=CRAG_ENABLED, max_results=CRAG_MAX_RESULTS)

Flujo:
  1. Si CRAG_ENABLED=false → retorna {} sin hacer nada
  2. Búsqueda DuckDuckGo con la query original (sin API key)
  3. Comprime cada resultado a 600 chars
  4. Appende bloque "--- Resultados web (CRAG) ---" al contexto existente
  5. Expone URLs en AgentState.web_sources → visible en CLI y en fuentesWeb del API
  6. Si falla (sin red, paquete no instalado) → falla silenciosa, sigue con contexto local
```

Variables de configuración:

| Variable | Default | Descripción |
|---|---|---|
| `CRAG_ENABLED` | `false` | Activa búsqueda web |
| `CRAG_MAX_RESULTS` | `3` | Máx. resultados DuckDuckGo a fusionar |

---

## Estado de implementación

- [x] LangGraph StateGraph core
- [x] Multi-Query con RRF reranking
- [x] HyDE — Hypothetical Document Embedding
- [x] graph_traverse — Property Graph Oracle 23ai SQL/PGQ
- [x] CRAG real — `web_fetch` con DuckDuckGo + Context Assembly + 11 tests
