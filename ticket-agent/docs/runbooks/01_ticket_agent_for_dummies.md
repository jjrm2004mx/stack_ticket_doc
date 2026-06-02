# 01 — ticket-agent for Dummies

Guía de introducción para entender qué es el agente, cómo funciona y cómo usarlo.
Última actualización: 2026-05-29 (Fase 4 completa — Oracle 23ai + REST API + LangGraph)

---

## ¿Qué es ticket-agent?

Es un asistente inteligente sobre el ecosistema de tickets. Le haces una pregunta en español,
él busca en la base de conocimiento y genera una respuesta basada en los documentos reales.

No inventa — si la información no está en la base de conocimiento, lo dice.
Si la confianza local es baja, puede buscar en la web para complementar la respuesta (CRAG).

**Hacia dónde va:** el agente está diseñado para recibir eventos de error del sistema,
diagnosticar la causa y proponer remediación. Las respuestas futuras incluirán:
causa · remediación · logs relevantes · runbook aplicable · fuentes.

---

## ¿Quién lo usa y para qué?

| Rol | Preguntas típicas |
|---|---|
| **usuario** | ¿Cómo creo un ticket? ¿Qué estados puede tener? ¿Cómo adjunto un archivo? |
| **manager** | ¿Cómo apruebo un borrador? ¿Qué permisos tengo? |
| **admin** | ¿Cómo configuro el workflow de clasificación? ¿Cómo gestiono usuarios? |
| **soporte** | ¿Cómo está configurado n8n? ¿Qué hace langchain-agent? ¿Cómo reinicio el stack? |

El rol determina qué documentación consulta — un `usuario` nunca verá runbooks técnicos.

---

## ¿De dónde saca la información?

El agente consulta cinco fuentes de conocimiento almacenadas en Oracle 23ai:

1. **Documentación por rol** — manuales escritos para cada tipo de usuario
   (`ticket-management/docs/usuarios/`)

2. **Ayuda contextual de pantallas** — el mismo texto que aparece en la interfaz web
   del sistema de tickets (`helpContent.ts`)

3. **Runbooks técnicos** — documentación operacional de todos los repos del ecosistema.
   Solo accesible con rol `soporte`

4. **Grafo de arquitectura** — relaciones entre servicios del ecosistema
   (qué llama a qué, dependencias, impacto de caídas). Solo para rol `soporte`
   con preguntas sobre arquitectura

5. **Web — CRAG (condicional)** — si la confianza sobre las fuentes locales es baja,
   el agente busca en DuckDuckGo, comprime los resultados y los fusiona al contexto
   antes de regenerar la respuesta. Se activa con `CRAG_ENABLED=true` en `.env`

---

## ¿Cómo se usa?

### Vía REST API (modo producción)

El agente expone una API REST en el puerto `8002`. Es el modo de uso en el stack completo.

```bash
# Health check
curl http://localhost:8002/api/v1/health
# → {"status":"UP","application":"ticket-agent"}

# Consulta
curl -s -X POST http://localhost:8002/api/v1/query \
  -H "Content-Type: application/json" \
  -d '{"query": "¿Cómo creo un ticket?", "rol": "usuario"}' | jq .
```

Swagger disponible en: `http://localhost:8002/docs`

La respuesta incluye:
```json
{
  "respuesta": "Para crear un ticket...",
  "fuentes": ["ticket-management/docs/usuarios/usuario/flujos.md"],
  "pantallas": ["newTicket"],
  "rol": "usuario",
  "confianza": 0.8,
  "latenciaMs": 7458
}
```

### Vía CLI (desarrollo / ingest)

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate

# Pregunta como usuario final
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario

# Pregunta como manager
python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager

# Pregunta como soporte técnico
python -m src.main query --query "¿Cómo está configurado n8n?" --rol soporte
```

---

## ¿Qué necesita para funcionar?

| Requisito | Detalle |
|---|---|
| Oracle 23ai accesible | Wallet en `~/.oci/wallet_ticketagent/` y tabla `KB_CHUNKS` poblada |
| OCI GenAI configurado | `~/.oci/config` válido; o Ollama si `PROVIDER=ollama` |
| Contenedor levantado | `bash start.sh` desde WSL |

Si `KB_CHUNKS` está vacío o desactualizado, ver `03_guia_operacional.md`.

Para re-ingestar:
```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate
python -m src.main ingest --source all
```

---

## ¿Por qué no responde bien?

| Síntoma | Causa probable |
|---|---|
| "No tengo esa información disponible" | El documento no está ingestado o el rol no tiene acceso |
| Respuesta muy genérica | La pregunta es ambigua — ser más específico |
| Error de conexión Oracle | Wallet no accesible o rutas incorrectas en `.env` — ver nota de rutas más abajo |
| `KB_CHUNKS` vacío | Ejecutar `ingest --source all` |
| Respuesta lenta (~8-12s) | Normal con OCI GenAI (latencia de red a OCI Chicago) |
| Respuesta lenta (~23s) | Multi-Query activo (3 llamadas al embedder + RRF) — reducir `MULTI_QUERY_N` |
| Respuesta lenta (~44s) | `HYDE_ENABLED=true` — agrega 1 llamada LLM extra |
| Fuentes web en el output | CRAG activo (`CRAG_ENABLED=true`) y la confianza local fue baja |
| CRAG activo pero sin fuentes web | DuckDuckGo sin resultados o sin red — el agente respondió solo con la base local |

> **Nota rutas Oracle:** las variables `ORACLE_CONFIG_DIR` y `ORACLE_WALLET_DIR` en `.env`
> deben tener rutas del **contenedor** (`/root/.oci/...`), no del host WSL.
> El compose monta `${HOME}/.oci → /root/.oci:ro` automáticamente.

---

## ¿Qué hace solo si se activa en `.env`?

| Variable | Default | Efecto |
|---|---|---|
| `CRAG_ENABLED=true` | `false` | Búsqueda web (DuckDuckGo) cuando la confianza local es baja |
| `HYDE_ENABLED=true` | `false` | Genera doc hipotético antes de embedear — mejora retrieval semántico, +~20s |
| `HYBRID_SEARCH=true` | `false` | Combina búsqueda vectorial + BM25 (requiere Oracle 23ai con índice Oracle Text activo) |
| `MULTI_QUERY_N=1` | `3` | Desactiva Multi-Query — más rápido, menos cobertura |

---

## ¿Qué NO hace todavía?

- No ingesta PDF, DOCX ni páginas web como fuente de conocimiento (Docling pendiente)
- No recibe eventos de error estructurados — la entrada es siempre texto libre `{query, rol}`
- No modifica tickets ni interactúa con el sistema de gestión
- No aprende de las conversaciones — la base de conocimiento se actualiza solo re-ingestando
- No tiene Diagnostic Agent ni Remediation Agent (próxima fase)

---

## ¿Cómo está organizado internamente?

```
ticket-agent/
├── src/
│   ├── main.py              → CLI (ingest + query)
│   ├── config.py            → configuración desde .env
│   ├── adapters/            → LLM (Ollama / OCI GenAI) + Embeddings (ONNX)
│   ├── document_processing/ → loader, chunking, mermaid_parser, help_serializer
│   ├── vector_store/        → OracleVSAdapter, FAISSVectorStore, VectorStoreFactory
│   ├── rag/                 → Retriever + prompts
│   ├── agent/               → LangGraph: state, nodes, graph
│   └── api/                 → FastAPI: health, info, query
├── docker-compose.yml       → contenedor ticket-agent (puerto 8002)
├── start.sh                 → arranque del stack
├── entrypoint.sh            → verificación pre-arranque dentro del contenedor
└── docs/
    ├── runbooks/            → esta guía y documentación operacional
    ├── tecnico/             → TAREAS.md, ORACLE_MIGRACION.md, PLAN_UI_AGENTE.md
    └── arquitectura/        → ecosistema.mmd (fuente del Property Graph)
```

---

## Pipeline interno (resumen)

```
Pregunta + rol
      │
      ▼
 [HyDE opt-in]  →  genera doc hipotético para mejorar el embedding
      │
      ▼
 [retrieve]     →  Multi-Query RRF: N variantes → Hybrid Search → top-K chunks
      │
      ▼
 [graph_traverse]  →  si hay keywords de arquitectura: SQL/PGQ sobre Property Graph
      │
      ▼
 [generate]     →  prompt con contexto → LLM → respuesta
      │
      ▼
 [validate]     →  confianza 0.0 / 0.3 / 0.8
      │
    ≥ 0.5 → respuesta final
    < 0.5 → [web_fetch] (DuckDuckGo) → vuelve a generate (máx 1 vez)
```
