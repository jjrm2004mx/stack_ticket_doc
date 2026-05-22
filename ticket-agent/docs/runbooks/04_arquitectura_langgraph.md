# 04 — Arquitectura LangGraph (Fase 3)

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
    query     : str           # pregunta original
    rol       : str           # usuario | manager | admin | soporte
    docs      : List[LangDoc] # chunks recuperados (post RRF)
    contexto  : str           # texto concatenado de los chunks
    respuesta : str           # respuesta generada por el LLM
    confianza : float         # score 0.0 – 1.0
    ciclos    : int           # número de veces que pasó por web_fetch
```

---

## Nodos

| Nodo | Función | Descripción |
|---|---|---|
| `retrieve` | `make_multi_query_node(retriever, llm, n)` | Genera N variantes → N búsquedas → RRF reranking |
| `generate` | `make_generate_node(llm)` | Formatea el prompt y llama al LLM |
| `validate` | `validate(state)` | Calcula el score de confianza |
| `web_fetch` | `web_fetch(state)` | Enriquece el contexto (placeholder CRAG) |

`retrieve` y `generate` usan factories (closures) porque necesitan acceso
al `retriever` y al `llm`. `validate` y `web_fetch` son funciones puras del estado.

> Cuando `MULTI_QUERY_N=1` el grafo usa `make_retrieve_node` (búsqueda simple).
> Con `MULTI_QUERY_N>1` (default: 3) usa `make_multi_query_node`.

---

## Multi-Query con RRF reranking

### ¿Qué hace?

En lugar de una sola búsqueda, el nodo `retrieve` genera N variantes de la
pregunta original con el LLM y hace una búsqueda por cada una.
Los resultados se fusionan eliminando duplicados y reordenando por RRF.

### Ejemplo

```
Pregunta original: "¿Cómo apruebo un borrador?"

Variante 1: "¿Cuál es el proceso para aprobar un ticket en borrador?"
Variante 2: "¿Qué pasos sigue un manager para revisar borradores?"
Variante 3: "¿Dónde apruebo tickets pendientes de revisión?"

→ 4 búsquedas en FAISS (original + 3 variantes)
→ RRF fusiona y reordena
→ Top-K chunks al nodo generate
```

### RRF (Reciprocal Rank Fusion)

Cada chunk recibe un score basado en su posición en cada lista de resultados:

```
score(chunk) = Σ  1 / (rank_en_lista_i + 60)
```

Un chunk que aparece en el top-3 de múltiples listas acumula más score
que uno que aparece solo una vez en posición 1. Esto favorece documentos
relevantes para varias formulaciones de la misma pregunta.

### Variables de configuración

| Variable | Default | Descripción |
|---|---|---|
| `MULTI_QUERY_N` | `3` | Número de variantes generadas por el LLM |
| `RETRIEVAL_K` | `3` | Chunks finales que pasan al nodo generate |

---

## Lógica de confianza (validate)

| Situación | Confianza |
|---|---|
| Sin documentos recuperados | 0.0 |
| Respuesta contiene frases de baja confianza (*) | 0.3 |
| Respuesta normal | 0.8 |

(*) Frases detectadas: "no tengo esa información", "no dispongo de",
"no está disponible", "no encuentro información", "no puedo responder",
"no hay información".

---

## Routing condicional

```python
UMBRAL_CONFIANZA = 0.5
MAX_CICLOS = 1

def _route(state) -> str:
    if state["confianza"] >= UMBRAL_CONFIANZA:
        return "end"          # respuesta aceptable → END
    if state["ciclos"] >= MAX_CICLOS:
        return "end"          # ya reintentamos → END de todas formas
    return "web_fetch"        # baja confianza → enriquecer y reintentar
```

---

## Diagrama del grafo

```
             ┌──────────────────────────┐
    inicio → │ retrieve (Multi-Query)   │
             │ LLM genera N variantes   │
             │ N búsquedas FAISS + RRF  │
             └────────────┬─────────────┘
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

## web_fetch — estado actual (placeholder)

El nodo `web_fetch` en Fase 3 **no hace búsqueda web real**.
Agrega un aviso al contexto y vuelve al nodo `generate` para que el LLM
intente responder con lo que tiene.

La implementación real (CRAG) está planificada para la siguiente iteración
de Fase 3 e incluirá:
- Búsqueda con `langchain WebBaseLoader` o similar
- Fusión de contexto local + web
- Umbral de confianza configurable por rol

---

## Cómo se construye el grafo (uso en main.py)

```python
from src.agent.graph import build_graph

# Construir grafo pasando retriever, llm y número de variantes
graph = build_graph(retriever, llm, multi_query_n=settings.MULTI_QUERY_N)

# Invocar con estado inicial
result = graph.invoke({
    "query": "¿Cómo apruebo un borrador?",
    "rol": "manager",
    "docs": [],
    "contexto": "",
    "respuesta": "",
    "confianza": 0.0,
    "ciclos": 0,
})

# Extraer resultados
respuesta  = result["respuesta"]
docs       = result["docs"]
confianza  = result["confianza"]   # 0.0 | 0.3 | 0.8
```

---

## Swap FAISS → Oracle (Fase 2)

El grafo **no conoce el vector store**. El nodo `retrieve` solo llama a
`retriever.retrieve()`. Cuando llegue Oracle 23ai, el cambio es una línea en `main.py`:

```python
# Hoy
store = FAISSVectorStore(embedder, settings.VECTOR_DB_PATH)

# Cuando llegue Fase 2
store = OracleVSAdapter(embedder, settings.ORACLE_DSN)
```

El grafo, los nodos, el estado y el RRF no se modifican.

---

## Próximas iteraciones (Fase 3)

- [x] Multi-Query con RRF reranking
- [ ] HyDE — generar documento hipotético antes de buscar en FAISS
- [ ] CRAG real — `web_fetch` con búsqueda en internet
- [ ] Graph Traversal — traversal sobre Property Graph Oracle 23ai (requiere Fase 2)
