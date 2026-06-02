# 01 — AI Stack for Dummies
## ticket-classification · Guía conceptual del ecosistema
**Stack: LangChain Agent · LangGraph · Podman · Redis · MinIO · LangSmith**
**Mayo 2026**

---

## La analogía: tu stack es una empresa de clasificación postal

Imagina una oficina que recibe correos de distintos departamentos de la empresa
y los clasifica automáticamente para que lleguen al lugar correcto.

| Pieza de la empresa postal | Pieza del ecosistema |
|---|---|
| La ventanilla que recibe correos | LangChain Agent — clasificador principal |
| El catálogo de reglas de clasificación | LangGraph — el cerebro que decide |
| El experto que lee y clasifica | Ollama (local) / OpenAI / Anthropic / Gemini / OCI GenAI |
| El archivero que guarda las fichas | classifier-db — PostgreSQL de este repo |
| El depósito de adjuntos originales | MinIO — almacena PDFs, Excel, imágenes |
| La memoria rápida de correos frecuentes | classifier-redis — caché de respuestas LLM |
| El registro de empleados de otra oficina | ticket-management — backend de tickets externos |
| El edificio que contiene todo esto | Podman — contenedores de este repo |
| El auditor de calidad | Pydantic — valida que la clasificación sea correcta |
| El supervisor externo de IA | LangSmith — trazabilidad del agente |
| El inspector de operaciones | Prometheus + Grafana (en `infra-monitoring`) |

---

## El ecosistema: 5 repositorios coordinados

Este repo **no vive solo**. Es una pieza de un ecosistema de 5 servicios que
colaboran a través de redes compartidas de Podman.

```
~/stack_ticket/
├── ticket-classification/     ← ESTE REPO — clasificador IA
├── ticket-management/         ← Backend de tickets + Redis + MinIO propios
├── notification-service/      ← Notificaciones por email/SMS
├── ticket-ingestion-light/    ← Disparador Gmail via n8n
└── infra-monitoring/          ← Prometheus + Grafana + scripts de arranque
```

> **Regla de oro del ecosistema:** Cada repo tiene su propio `docker-compose.yml`.
> El stack completo se levanta con el `startup.sh` de `infra-monitoring`.
> Este repo tiene su propio `start.sh` para arranque independiente o integrado.

---

## Componentes de este repo

Todos los servicios siguientes viven en el `docker-compose.yml` de este repo:

| Contenedor | Imagen | Puerto | Rol |
|---|---|---|---|
| `langchain-agent` | build local | 8001 | Núcleo — orquesta la clasificación |
| `langchain-api` | build local | 8000 | Gateway de IA — abstrae los providers |
| `ollama` | ollama/ollama | 11434 | IA local sin internet |
| `classifier-db` | postgres:15-alpine | 5432 | Base de datos propia del clasificador |
| `classifier-redis` | redis:7-alpine | 6379 | Caché de respuestas LLM + estado de jobs |
| `minio` | minio/minio | 9000/9001 | Almacenamiento de adjuntos de correos |
| `classifier-flyway` | flyway:10 | — | Migraciones automáticas al arrancar |

### Redes externas requeridas

| Red | Quién la usa |
|---|---|
| `ticket-classification-network` | Comunicación interna de este stack |
| `ticket-management-network` | Acceso a `ticket-system-backend` y n8n |

---

## Providers de IA disponibles

| Provider | Modelo default | Tipo | Configuración |
|---|---|---|---|
| Ollama | `llama3.2:3b` | Local, sin costo, sin internet | `OLLAMA_BASE_URL=http://ollama:11434` |
| OpenAI | `gpt-4o-mini` | Cloud | `OPENAI_API_KEY` en `.env` |
| Anthropic | `claude-3-5-haiku-20241022` | Cloud | `ANTHROPIC_API_KEY` en `.env` |
| Gemini | `gemini-1.5-flash` | Cloud | `GEMINI_API_KEY` en `.env` |
| OCI GenAI | `cohere.command-r-plus-08-2024` | Cloud Oracle | `OCI_COMPARTMENT_ID` + `~/.oci/config` |

El provider activo se controla con `AGENT_PROVIDER` en el `.env`.
Se puede cambiar por request sin reiniciar el stack.

---

## Podman — El edificio

Podman empaqueta cada componente en un **contenedor** aislado.
La diferencia clave con Docker: Podman es **rootless** — sin daemon privilegiado,
más seguro y compatible con Kubernetes nativamente.

```bash
# Los comandos son equivalentes a Docker
docker compose up -d    →    podman-compose up -d
docker compose down     →    podman-compose down
docker ps               →    podman ps
docker logs nombre      →    podman logs --tail 50 nombre
docker exec             →    podman exec
```

> **Regla de oro de comunicación entre contenedores:**
> Dentro de Podman los servicios se hablan por **nombre del servicio**, nunca
> por `localhost`.
>
> ✅ Correcto: `http://langchain-api:8000`
> ✅ Correcto: `http://ticket-system-backend:8080`
> ❌ Incorrecto: `http://localhost:8000`

---

## LangChain Agent — El clasificador inteligente

Este es el núcleo del repo. Recibe correos (asunto + cuerpo) y los clasifica
de forma autónoma usando un **loop dinámico con decisión propia**.

El endpoint es **asíncrono** — responde inmediatamente con un `job_id`
y procesa en background. El cliente hace polling para obtener el resultado.

### ¿Qué entradas acepta?

| Origen | Cómo llega |
|---|---|
| Power Automate / Outlook | `POST /process` con asunto, cuerpo, adjuntos |
| n8n (ticket-ingestion-light) | Trigger Gmail → `POST /process` |
| Manual / pruebas | curl directo al endpoint |

### Flujo en dos pasos (asíncrono)

```
POST /process  →  responde job_id inmediatamente (202 Accepted)
                       │
                       ▼ (background)
               [classify] → [validate] → [save]
                   ↑____________❌ retry (≤ AGENT_MAX_ITERATIONS)

GET /status/{job_id}  →  polling hasta obtener status=completado|error
```

### ¿Qué hace paso a paso?

```
Correo entra (asunto + cuerpo + adjuntos opcionales)
     │
     ▼ ── Deduplicación por conversation_id
     │     Si ya existe un ticket para este hilo → status=ignorado
     ▼
[classify] ── Llama a langchain-api con asunto+cuerpo
     │         El LLM responde con JSON:
     │         { dominio, categoria, prioridad, confianza }
     ▼
[validate] ── ¿El JSON tiene formato correcto?
     │         ¿Los valores son válidos según Pydantic + fuzzy matching?
     │    ❌ No → vuelve a [classify] con instrucciones ajustadas
     │    ✅ Sí → continúa
     ▼
[save] ─────── Persiste en classifier-db (ticket + agent_run + attachments)
     │         Sube adjuntos a MinIO
     │         Llama a ticket-system-backend para registrar ticket externo
     │         Guarda resultado en Redis (job:UUID, TTL 24h)
     ▼
GET /status/{job_id} devuelve status=completado con todos los campos
```

### ¿Por qué es mejor que un flujo n8n estático?

| n8n anterior | LangChain Agent |
|---|---|
| JSON malformado del LLM → fallo silencioso | Falla → reintenta con instrucciones corregidas |
| Categorías fijas hardcodeadas | Dominios y categorías configurables vía `.env` |
| Lógica en nodos visuales JSON | Código Python puro, 100% Git-friendly |
| Flujo estático A → B → C | Loop dinámico: el agente decide cuándo avanzar |
| Sin validación de esquema | Validación Pydantic + fuzzy matching obligatorio |
| Sin deduplicación de hilos | Deduplicación automática por `conversation_id` |

---

## LangGraph — El cerebro que orquesta

LangGraph define el **grafo de decisión** del agente — un diagrama de flujo
con memoria y lógica dinámica entre nodos.

```
     ┌──────────────────────────────────────────┐
     │           LangGraph Agent                │
     │                                          │
     │  ┌──────────┐   ┌──────────┐             │
     │  │ classify │──▶│ validate │             │
     │  └──────────┘   └────┬─────┘             │
     │       ▲              │                   │
     │       │         ┌────▼──────┐            │
     │       │    ❌    │  ¿válido? │            │
     │       └─────────│   No      │            │
     │      retry      └────┬──────┘            │
     │  (máx AGENT_MAX_     │ ✅ Sí              │
     │   ITERATIONS)   ┌────▼──────┐            │
     │                 │   save    │            │
     │                 └───────────┘            │
     └──────────────────────────────────────────┘
```

Cada vez que el agente regresa a `classify`, ajusta el prompt para
corregir el error anterior — comportamiento autónomo.

---

## LangChain API — Gateway de IA

FastAPI expone los modelos de IA como API REST. El patrón
**Adapter + Strategy** permite intercambiar el provider con un solo campo
en el JSON, sin tocar código ni reiniciar el stack.

```
LangChain Agent  →  langchain-api:8000/ask  →  Provider elegido
                                                 ├── Ollama (local)
                                                 ├── OpenAI
                                                 ├── Anthropic
                                                 ├── Gemini
                                                 └── OCI GenAI
```

**Caché Redis en la API:** cada llamada a `/ask` genera una clave
`MD5(prompt + system + provider)`. Si existe en Redis, responde en <100ms
sin llamar al LLM. TTL: 1 hora.

---

## Ollama — IA local

Corre modelos de inteligencia artificial **dentro del servidor**, sin internet,
sin costo por uso y con privacidad total.

**Modelo default:** `llama3.2:3b` (~2 GB, ideal para clasificación de texto)

| Ventaja | Desventaja |
|---|---|
| Privacidad total — datos no salen del servidor | Lento en CPU: 15–40 segundos por clasificación |
| Sin costo por token | Menos capaz que GPT-4 en tareas complejas |
| Funciona sin internet (tras la descarga) | Requiere ~3 GB de RAM durante la inferencia |

> Primera vez requiere descargar el modelo:
> `podman exec -it ollama ollama pull llama3.2:3b`

---

## MinIO — Almacenamiento de adjuntos

MinIO es un servidor de objetos compatible con S3 que almacena los archivos
adjuntos de los correos (PDFs, Excel, imágenes) sin guardarlos en la BD.

| Parámetro | Valor |
|---|---|
| API | http://localhost:9000 |
| Consola web | http://localhost:9001 |
| Credenciales default | minioadmin / minioadmin123 |
| Bucket | `email-attachments` |

El agente sube los adjuntos a MinIO y guarda solo `filename` + `content_type`
en la tabla `attachments` de classifier-db.

---

## classifier-redis — Caché y estado de jobs

Redis cumple dos roles en este repo:

**Rol 1 — Caché de respuestas LLM:**
```
Primera clasificación:   texto → LLM → 25 segundos → guarda en Redis
Segunda clasificación:   texto → Redis → 80 milisegundos  (cached: true)
```

**Rol 2 — Estado de jobs asíncronos:**
```
POST /process  →  Redis guarda: job:{uuid} = { status: "en_proceso" }
                                                          ↓ (background)
GET /status    →  Redis lee:    job:{uuid} = { status: "completado", ... }
```
Los jobs tienen TTL de 24 horas.

---

## Dominios y categorías configurables

El clasificador no tiene categorías fijas en el código — todo se configura en `.env`:

```bash
# Dominios disponibles
AGENT_DOMAINS=IT,cliente,operaciones,otro

# Categorías por dominio (fuzzy matching con umbral FUZZY_THRESHOLD=80)
CATEGORIES_IT=hardware,software,red,acceso,correo,impresora,vpn,servidor,base_de_datos,seguridad
CATEGORIES_CLIENTE=facturacion,reclamo,consulta,devolucion,garantia,soporte,pedido,envio
CATEGORIES_OPERACIONES=logistica,compras,inventario,mantenimiento,produccion,calidad,proveedores
CATEGORIES_OTRO=general,sin_clasificar
```

**Agregar un dominio nuevo (ejemplo: RRHH):**
```bash
# 1. Editar .env
AGENT_DOMAINS=IT,cliente,operaciones,RRHH,otro
CATEGORIES_RRHH=vacaciones,permisos,nomina,capacitacion,onboarding

# 2. Reiniciar solo el agente
cd ~/stack_ticket/ticket-classification
podman-compose restart langchain-agent
# Sin tocar código. Sin modificar la base de datos.
```

---

## Modo mock — desarrollo sin LLM

Para desarrollo y pruebas rápidas, el agente puede saltarse el LLM
y devolver una clasificación fija configurada en `.env`:

```bash
# En .env
AGENT_MOCK_CLASSIFY=true
MOCK_DOMINIO=IT
MOCK_CATEGORIA=acceso
MOCK_PRIORIDAD=BAJA
MOCK_CONFIANZA=1.0
```

Con mock activado, `POST /process` responde en milisegundos sin llamar
a ningún provider de IA.

---

## LangSmith — El supervisor de IA

Mientras Grafana monitorea la infraestructura, LangSmith monitorea
la **inteligencia**: cada llamada al LLM queda registrada con input,
output, latencia, número de iteraciones y errores de validación.

**Configuración en `.env`:**
```bash
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=tu_key_de_langsmith
LANGCHAIN_PROJECT=shared-services-classifier-dev
```

---

## Observabilidad — Prometheus y Grafana

La observabilidad **no vive en este repo** — está centralizada en `infra-monitoring`.

| Qué se monitorea | Dónde verlo |
|---|---|
| Métricas del agente (`/metrics`) | Prometheus → `http://localhost:9090` |
| Dashboards del stack | Grafana → `http://localhost:3000` |
| Logs centralizados | Grafana → Explore → Loki |

Prometheus scrapeó `langchain-agent:8001/metrics` y `langchain-api:8000/metrics`
al levantar `infra-monitoring`.

---

## Flujo completo del ecosistema

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ticket-classification                           │
│                                                                     │
│  Power Automate ─┐                                                  │
│  n8n (Gmail)    ─┤──▶  langchain-agent :8001                        │
│  Manual         ─┘     ┌──────────────────────────────────────┐     │
│                         │  LangGraph (asíncrono)               │     │
│                         │  classify → validate → save          │     │
│                         │      ↑_______❌ retry (≤N)           │     │
│                         └──────────────────────────────────────┘     │
│                              │         │         │                  │
│                              ▼         ▼         ▼                  │
│                   langchain-api   classifier-db  minio              │
│                      :8000           :5432       :9000              │
│                   ┌──────────┐                                      │
│                   │ ollama   │                                      │
│                   │ openai   │  classifier-redis :6379              │
│                   │anthropic │  (caché LLM + estado de jobs)       │
│                   │ gemini   │                                      │
│                   └──────────┘                                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ ticket-management-network
                    ┌──────────┴──────────┐
                    ▼                     ▼
         ticket-system-backend     prometheus (infra-monitoring)
              :8080                      :9090
         (registra ticket externo)   (métricas)

LangSmith (cloud) ← trazas de cada ejecución del agente
```

---

## Referencia rápida de URLs

| Servicio | URL (desde Windows) | Credenciales |
|---|---|---|
| LangChain Agent | http://localhost:8001 | sin auth |
| Agent Swagger | http://localhost:8001/docs | sin auth |
| LangChain API | http://localhost:8000 | sin auth |
| API Swagger | http://localhost:8000/docs | sin auth |
| Ollama | http://localhost:11434 | sin auth |
| classifier-db (PostgreSQL) | localhost:5432 | admin / admin / db:classifier_db |
| MinIO API | http://localhost:9000 | minioadmin / minioadmin123 |
| MinIO Consola | http://localhost:9001 | minioadmin / minioadmin123 |
| Prometheus | http://localhost:9090 | sin auth |
| Grafana | http://localhost:3000 | admin / admin |
| LangSmith | https://smith.langchain.com | cuenta Google |

---

*ticket-classification · AI Stack for Dummies · Mayo 2026*
