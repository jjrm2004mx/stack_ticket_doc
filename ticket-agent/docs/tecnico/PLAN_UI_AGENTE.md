# PLAN UI — Ventanas del Agente RAG
**ticket-agent · Documento de seguimiento**
Última actualización: 2026-05-21

---

## Resumen ejecutivo

El agente RAG tendrá **dos interfaces de usuario** con audiencias y tecnologías distintas:

| Interfaz | Audiencia | Repo | Puerto | Fase |
|---|---|---|---|---|
| Panel de chat embebido | Usuario, Manager, Admin | `ticket-management` frontend | 8090 (existente) | Fase 4 |
| UI independiente de soporte | Soporte técnico | `ticket-agent` FastAPI | 8002 | Fase 4 |

Ambas dependen de que `ticket-agent` exponga una **REST API** (FastAPI) como backend común.

---

## Arquitectura de comunicación

```
┌─────────────────────────────────────────────────────────┐
│  ticket-management frontend (React/Vite · :8090)        │
│                                                         │
│  ┌─────────────────────────┐                            │
│  │  ChatPanel component    │──► POST /api/query         │
│  │  rol: usuario/manager/  │    ticket-agent REST API   │
│  │        admin            │    http://localhost:8002   │
│  └─────────────────────────┘                            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  ticket-agent UI (FastAPI + HTML/JS · :8002)            │
│                                                         │
│  ┌─────────────────────────┐                            │
│  │  Consola de soporte     │──► POST /api/query         │
│  │  rol: soporte (fijo)    │    mismo backend FastAPI   │
│  │  acceso total           │    sin autenticación UI    │
│  └─────────────────────────┘                            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  ticket-agent REST API (FastAPI · :8002)                │
│                                                         │
│  POST /api/query                                        │
│    body: { query: str, rol: str }                       │
│    response: { answer: str, sources: [], latency: float}│
│                                                         │
│  GET  /api/health                                       │
│  GET  /api/info  → modelo, dims, chunks en índice       │
└─────────────────────────────────────────────────────────┘
```

---

## UI 1 — Panel de chat en ticket-management

### Descripción

Widget de chat flotante o panel lateral embebido en el portal de tickets.
Aparece en todas las pantallas y adapta su contexto según la pantalla activa
(ya que el agente conoce `helpContent.ts`).

### Audiencia

Usuarios finales del portal: `usuario`, `manager`, `admin`.
El rol se toma automáticamente de la sesión activa — el usuario no lo selecciona.

### Ubicación en el repo

```
ticket-management/
└── frontend/
    └── src/
        └── components/
            └── agent/
                ├── ChatPanel.tsx          # Componente principal
                ├── ChatMessage.tsx         # Burbuja de mensaje
                ├── ChatInput.tsx           # Input + botón enviar
                ├── SourceList.tsx          # Lista de fuentes citadas
                └── useAgentChat.ts         # Hook: llamadas a REST API
```

### Comportamiento esperado

- **Widget flotante** (botón ?) en esquina inferior derecha de todas las pantallas
- Al abrir: muestra historial de la sesión y campo de texto
- El rol se toma de la sesión activa de Spring Security — no configurable por el usuario
- La pantalla activa se envía como contexto (`screen: "ticketDetail"`) para que el agente priorice ayuda contextual
- Si `ticket-agent` no está disponible: ocultar el botón o mostrar "Asistente no disponible"
- Respuesta en streaming (si el LLM lo soporta) para mejor UX

### API call desde el frontend

```typescript
const response = await fetch(`${TICKET_AGENT_URL}/api/query`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    query: userMessage,
    rol: currentUser.role,        // "usuario" | "manager" | "admin"
    screen: currentScreen,        // "ticketDetail" | "newTicket" | etc.
  }),
});
const { answer, sources, latency } = await response.json();
```

### Variables de entorno requeridas en ticket-management

```bash
# frontend/.env o backend application.properties
VITE_TICKET_AGENT_URL=http://localhost:8002
```

### Tareas de implementación

- [ ] Crear componente `ChatPanel.tsx` con estado de mensajes
- [ ] Crear hook `useAgentChat.ts` con llamada a REST API y manejo de errores
- [ ] Crear componente `ChatMessage.tsx` (usuario vs agente, con avatar)
- [ ] Crear componente `SourceList.tsx` — muestra archivos citados como links colapsables
- [ ] Agregar botón flotante en layout principal (`App.tsx` o `Layout.tsx`)
- [ ] Integrar rol del usuario desde contexto de autenticación
- [ ] Integrar pantalla activa desde React Router
- [ ] Manejar estado "agente no disponible" (health check al montar)
- [ ] Agregar `VITE_TICKET_AGENT_URL` a `.env.example` de ticket-management
- [ ] Test: respuesta llega y se muestra correctamente
- [ ] Test: widget se oculta si el agente no responde

---

## UI 2 — Consola de soporte en ticket-agent

### Descripción

Interfaz web independiente servida por el mismo proceso FastAPI de `ticket-agent`.
Diseño funcional, sin framework pesado — HTML + JS vanilla o Jinja2 templates.
Acceso directo en `http://localhost:8002/ui`.

### Audiencia

Personal de soporte técnico. El rol `soporte` está fijo — no se autentica,
no se selecciona. Acceso total a todas las fuentes (runbooks de los 5 repos +
docs de usuarios + helpContent).

### Ubicación en el repo

```
ticket-agent/
└── src/
    └── api/
        ├── main_api.py            # FastAPI app — expone /api/query, /api/health, /ui
        ├── routes/
        │   ├── query.py           # POST /api/query
        │   └── health.py          # GET /api/health, GET /api/info
        └── ui/
            ├── templates/
            │   └── index.html     # Consola de soporte (Jinja2)
            └── static/
                ├── chat.js        # Lógica de chat (vanilla JS)
                └── style.css      # Estilos minimalistas
```

### Comportamiento esperado

- Pantalla completa: sidebar izquierdo con historial de consultas + área principal de chat
- Campo de texto grande (soporte hace preguntas técnicas complejas)
- Respuesta muestra: texto generado + **fuentes expandibles** (path del archivo, pantalla si aplica)
- Indicador de latencia visible
- Botón "Limpiar conversación"
- Selector de modelo LLM (Ollama / OCI) para comparar respuestas — opcional

### Tareas de implementación

- [ ] Crear `src/api/main_api.py` con FastAPI app
- [ ] Implementar `POST /api/query` — llama al Retriever y LLM existentes
- [ ] Implementar `GET /api/health` — verifica FAISS cargado + LLM disponible
- [ ] Implementar `GET /api/info` — modelo, dims, total chunks en índice
- [ ] Crear template `index.html` con layout sidebar + chat
- [ ] Crear `chat.js` — fetch a `/api/query`, render de mensajes y fuentes
- [ ] Servir estáticos desde FastAPI (`StaticFiles`)
- [ ] Agregar `fastapi` y `uvicorn` a `requirements.txt`
- [ ] Agregar comando de arranque a `CLAUDE.md` y runbook operacional
- [ ] Test: endpoint `/api/query` responde con estructura correcta
- [ ] Test: endpoint `/api/health` detecta índice no inicializado

---

## REST API compartida (ticket-agent FastAPI)

Contrato de la API que consumen ambas UIs.

### `POST /api/query`

```json
// Request
{
  "query": "¿Cómo apruebo un borrador?",
  "rol": "manager",
  "screen": "draftReview"          // opcional — solo desde ticket-management UI
}

// Response
{
  "answer": "Para aprobar un borrador debes...",
  "sources": [
    {
      "fuente": "ticket-management/docs/usuarios/manager/flujos.md",
      "pantalla": null,
      "tipo": "usuarios"
    },
    {
      "fuente": "helpContent.ts",
      "pantalla": "draftReview",
      "tipo": "helpContent"
    }
  ],
  "rol": "manager",
  "latency_seconds": 2.34
}
```

### `GET /api/health`

```json
{
  "status": "ok",
  "faiss_loaded": true,
  "llm_provider": "ollama",
  "llm_model": "llama3.2:3b"
}
```

### `GET /api/info`

```json
{
  "embed_model": "mixedbread-ai/mxbai-embed-large-v1",
  "embed_dims": 1024,
  "total_chunks": 1119,
  "chunks_by_type": {
    "runbooks": 700,
    "usuarios": 212,
    "helpContent": 207
  },
  "vector_store": "faiss"
}
```

---

## Dependencias nuevas requeridas

### ticket-agent `requirements.txt`

```
fastapi>=0.111.0
uvicorn[standard]>=0.29.0
jinja2>=3.1.0          # templates HTML para consola soporte
python-multipart       # form data
```

### ticket-management frontend

```
# No hay nuevas dependencias — solo fetch nativo
# Variable de entorno: VITE_TICKET_AGENT_URL
```

---

## Configuración de red (Fase 4 contenedorizado)

| Contenedor | Puerto host | Red |
|---|---|---|
| `ticket-agent` | 8002 | `ticket-classification-network` (para Ollama) |

Actualizar `infra-monitoring/docs/runbooks/11_puertos_y_redes.md` al implementar.

---

## Orden de implementación sugerido

```
1. REST API FastAPI en ticket-agent          ← base de todo
     └── POST /api/query
     └── GET  /api/health
     └── GET  /api/info

2. Consola de soporte (UI 2)                 ← más simple, sin autenticación
     └── HTML + JS vanilla
     └── Jinja2 templates

3. Panel de chat en ticket-management (UI 1) ← más complejo, requiere integración con auth
     └── React component
     └── Integración con rol y pantalla activa

4. Contenedorizar ticket-agent               ← Fase 4 completa
     └── Dockerfile
     └── podman-compose.yml
     └── Registrar en 11_puertos_y_redes.md
```

---

## Decisiones abiertas

| # | Pregunta | Opciones | Estado |
|---|---|---|---|
| 1 | ¿Streaming en la respuesta? | SSE / WebSocket / polling | Pendiente |
| 2 | ¿Historial de chat persistido? | Solo sesión / SQLite / sin historial | Pendiente |
| 3 | ¿Autenticación en consola soporte? | Sin auth / token estático / integrar con stack | Pendiente |
| 4 | ¿Selector de LLM en consola soporte? | Sí (comparar Ollama vs OCI) / No | Pendiente |
| 5 | ¿UI 1 como widget flotante o panel lateral? | Flotante / Sidebar / Tab en pantalla | Pendiente |
