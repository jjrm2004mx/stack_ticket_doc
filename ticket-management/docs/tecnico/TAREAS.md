# TAREAS — ticket-management

Control de actividades por fase. Marcar `[x]` al completar cada tarea.
Última actualización: 2026-05-29

---

## FASE 1 — Integración ticket-agent (RAG + Diagnóstico)

**Objetivo:** Exponer las capacidades de ticket-agent al frontend de ticket-management
según el rol del usuario autenticado.

**Estado:** En progreso — rama `feature/agent-integration`

### Contexto

- ticket-agent corre en `http://ticket-agent:8002` dentro de `ticket-management-network`
- Endpoints consumidos:
  - `POST /api/v1/query` → `{query, rol}` → respuesta RAG filtrada por rol
  - `POST /api/v1/diagnose` → `{errorCode, service, severity, message, stackTrace}` → diagnóstico + plan de remediación
- El rol se propaga desde el `SecurityContext` (atributo `role` inyectado por `JwtAuthFilter`)

### Backend — ticket-management

#### DTOs (`infrastructure/web/dto/`)

- [ ] Crear `AgentQueryRequest.java` — record `{ String query }`
- [ ] Crear `AgentQueryResponse.java` — record `{ String respuesta, List<String> fuentes, List<String> pantallas, String rol, double confianza, long latenciaMs }`
- [ ] Crear `DiagnoseRequest.java` — record `{ String errorCode, String service, String severity, String message, String stackTrace }`
- [ ] Crear `DiagnoseResponse.java` — record `{ String causa, String remediacion, String runbook, List<String> fuentes, double confianza, long latenciaMs, String servicio, String severidad }`

#### Cliente HTTP (`infrastructure/agent/`)

- [ ] Crear `AgentClient.java` — `@Component` con `RestTemplate`
  - `queryAgent(String query, String rol) → AgentQueryResponse`
  - `diagnoseTicket(DiagnoseRequest req) → DiagnoseResponse`
  - Timeout: 30s (llamadas LLM pueden tardar hasta 12s)
  - Error handling: `AgentUnavailableException` si ticket-agent no responde

#### Configuración

- [ ] Agregar `@Bean RestTemplate` en clase de configuración existente
- [ ] Agregar `agent.base-url=http://ticket-agent:8002` en `application.properties`
- [ ] Agregar `AGENT_BASE_URL` como variable de entorno en `docker-compose.yml`

#### Controller (`infrastructure/web/controller/`)

- [ ] Crear `AgentController.java`
  - `POST /api/v1/agent/query` — todos los roles autenticados
    - Toma `query` del body, `rol` del `request.getAttribute("role")`
    - Delega a `AgentClient.queryAgent()`
  - `POST /api/v1/tickets/{id}/diagnose` — solo `soporte` y `admin`
    - Valida rol antes de proceder (403 si no autorizado)
    - Delega a `AgentClient.diagnoseTicket()`

#### Excepción

- [ ] Crear `AgentUnavailableException.java` en `shared/exception/`
- [ ] Registrar handler en `GlobalExceptionHandler` → 503 con mensaje estándar

### Frontend — ticket-management

#### Widget flotante (todos los roles)

- [ ] Crear componente widget RAG (`agent-widget.js` o equivalente en el stack frontend)
  - Badge circular en esquina inferior derecha
  - Ventana de chat `380×600px` con animación slide-up
  - Efecto streaming (carácter por carácter)
  - Muestra fuentes debajo de cada respuesta del agente
  - Llama a `POST /api/v1/agent/query` con el JWT del usuario
- [ ] Integrar widget en el layout principal (visible en todas las pantallas autenticadas)

#### Consola de soporte — integrada en ticket detail

- [ ] Agregar botón "Diagnosticar" en `TicketDetail` — visible solo para `soporte`/`admin`
  - Condición adicional: ticket tiene `errorCode` o `status == ERROR`
- [ ] Al hacer clic: abrir panel/modal con formulario pre-cargado
  - Campos: `errorCode`, `service`, `severity`, `message`, `stackTrace` (pre-llenados desde el ticket)
  - Botón "Ejecutar diagnóstico" → llama a `POST /api/v1/tickets/{id}/diagnose`
  - Muestra `causa`, `remediacion`, `runbook` en el mismo panel
  - Indicador de latencia

---

## FASE 2 — (Futura) Mejoras de la integración

- [ ] Historial de consultas RAG por usuario (persistencia en BD)
- [ ] Caché de respuestas frecuentes (Redis)
- [ ] Panel de diagnósticos pasados en la consola de soporte

---

## NOTAS Y DECISIONES

| Fecha | Decisión | Razón |
|---|---|---|
| 2026-05-29 | Widget flotante para todos los roles, consola dedicada solo para soporte/admin | Roles no técnicos necesitan ayuda contextual; soporte necesita diagnóstico estructurado |
| 2026-05-29 | Consola standalone (emergencia) vive en ticket-agent, no aquí | Si ticket-management no levanta, la consola no puede depender de él — ver TAREAS.md de ticket-agent, Fase 5 |
| 2026-05-29 | Timeout de AgentClient: 30s | LLM puede tardar hasta 12s; 30s da margen sin bloquear indefinidamente |
| 2026-05-29 | Diagnose endpoint restringido a soporte/admin con validación manual de rol | Mismo patrón que el resto de los controllers — `request.getAttribute("role")` |
