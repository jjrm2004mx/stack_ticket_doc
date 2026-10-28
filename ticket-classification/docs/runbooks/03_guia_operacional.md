# 03 — Guía Operacional
## ticket-classification · Operaciones del día a día
**Ruta raíz: `~/stack_ticket/ticket-classification` (Rocky)**
**Octubre 2026**

Arquitectura y respuestas de cada endpoint: [02_arquitectura_tecnica.md](02_arquitectura_tecnica.md).
Mensajes retenidos: [09_bandeja_mensajes_retenidos.md](09_bandeja_mensajes_retenidos.md).
Fallas: [troubleshooting.md](troubleshooting.md). Todos los puertos escuchan solo en
`127.0.0.1`; desde otra máquina, túnel SSH (infra-monitoring `10_redes_y_acceso_lan.md` §7).

---

## Índice

1. [Regla de oro](#1-regla-de-oro)
2. [Ciclo de vida del stack](#2-ciclo-de-vida-del-stack)
3. [Verificación de salud](#3-verificación-de-salud)
4. [Probar el clasificador](#4-probar-el-clasificador)
5. [Logs](#5-logs)
6. [Base de datos — consultas frecuentes](#6-base-de-datos--consultas-frecuentes)
7. [Cambiar provider de IA](#7-cambiar-provider-de-ia)
8. [Áreas y categorías](#8-áreas-y-categorías)
9. [Modo mock — pruebas sin LLM](#9-modo-mock--pruebas-sin-llm)
10. [Desplegar cambios de código](#10-desplegar-cambios-de-código)
11. [Ollama](#11-ollama)
12. [Redis — cola de trabajo y caché](#12-redis--cola-de-trabajo-y-caché)
13. [MinIO — archivos de los correos](#13-minio--archivos-de-los-correos)
14. [Migraciones Flyway](#14-migraciones-flyway)
15. [LangSmith — trazas del agente](#15-langsmith--trazas-del-agente)
16. [Checklist diario](#16-checklist-diario)

---

## 1. Regla de oro

> Ejecutar desde la raíz del repo: `cd ~/stack_ticket/ticket-classification`.
> Para que `langchain-agent` tome un cambio del `.env` hay que correr **`bash start.sh`**:
> `podman-compose restart` no relee el `.env` (el agente se crea con `podman run --env-file`).

---

## 2. Ciclo de vida del stack

```bash
bash ~/stack_ticket/infra-monitoring/startup.sh up       # todo el ecosistema, en orden
bash ~/stack_ticket/ticket-classification/start.sh       # solo este repo (recrea todo)
cd ~/stack_ticket/ticket-classification && podman-compose down   # apagar (datos se conservan)
podman ps --format "table {{.Names}}\t{{.Status}}"
podman stats --no-stream
```

**Nunca** `podman-compose down -v` sin respaldo: borra la bandeja (correos aún no entregados
y los retenidos), Redis y MinIO.

`start.sh` se detiene si `CLASSIFIER_SERVICE_KEY` o `LANGCHAIN_INTERNAL_API_KEY` están vacías o
con el valor de ejemplo.

---

## 3. Verificación de salud

```bash
curl -s http://localhost:8001/health | python3 -m json.tool
# "status":"ok", "agent_provider":"oci", "ticket_management":"ok" ("failing" = llave de servicio rechazada)
curl -s http://localhost:8000/health | python3 -m json.tool
# "default_provider":"oci", "redis":"ok"
podman exec classifier-db pg_isready -U admin -d classifier_db
podman exec classifier-redis sh -c 'redis-cli --no-auth-warning -a "$REDIS_PASSWORD" ping'
curl -sf http://localhost:9000/minio/health/live && echo "MinIO OK"

# ¿Hay correos atorados? (todo lo que no está entregado)
podman exec classifier-db psql -U admin -d classifier_db -c \
  "SELECT state, count(*), max(last_error) FROM inbox_messages WHERE state <> 'DELIVERED' GROUP BY 1;"
```

---

## 4. Probar el clasificador

Ya no existe un endpoint para mandar un correo "a mano": la bandeja solo acepta correos con la
llave de un canal (`X-Channel-Key`), que ticket-management muestra una sola vez al crear el
canal. Formas de probar:

**a) De punta a punta (correo real)** — con la ingesta del canal activa, mandar un correo al
buzón del canal y seguirlo (infra-monitoring `08_smoke_test_end_to_end.md`):

```bash
podman exec classifier-db psql -U admin -d classifier_db -c \
  "SELECT id, kind, state, ticket_id, fallback, last_error, held_reason, received_at
     FROM inbox_messages ORDER BY created_at DESC LIMIT 5;"
```

**b) Solo el LLM** — `/ask` de `langchain-api` con la llave interna:

```bash
KEY=$(grep ^LANGCHAIN_INTERNAL_API_KEY= .env | cut -d= -f2)
curl -s -X POST http://localhost:8000/ask -H "Content-Type: application/json" -H "X-Api-Key: $KEY" \
  -d '{"prompt": "Clasifica: el servidor de producción no responde desde las 9am"}' | python3 -m json.tool
```

**c) Ver qué decidió para un correo** — clasificación y corridas del agente:

```bash
podman exec classifier-db psql -U admin -d classifier_db -c \
  "SELECT m.subject, m.classification->>'area' AS area, m.classification->>'category' AS category,
          m.classification->>'priority' AS priority, m.fallback, r.provider, r.duration_ms
     FROM inbox_messages m LEFT JOIN agent_runs r ON r.message_id = m.id
    ORDER BY m.created_at DESC LIMIT 5;"
```

---

## 5. Logs

```bash
podman logs --tail 100 langchain-agent
podman logs -f langchain-api
# Recorrido de los correos (eventos JSON):
podman logs --since 1h langchain-agent 2>&1 | grep -E '"(inbox_received|inbox_classified|inbox_classify_retry|inbox_classify_fallback|inbox_delivered|inbox_delivery_retry|inbox_held)"'
# Errores reales (sin el ruido de inbox_worker_error, ver troubleshooting)
podman logs --since 1h langchain-agent 2>&1 | grep -i error | grep -v inbox_worker_error
```

Grafana → Explore → Loki: `{container="langchain-agent"}`.

---

## 6. Base de datos — consultas frecuentes

```bash
podman exec -it classifier-db psql -U admin -d classifier_db      # \dt, \q
```

```sql
-- Correos por estado y canal
SELECT channel_id, state, count(*) FROM inbox_messages GROUP BY 1, 2 ORDER BY 1, 2;

-- Retenidos que esperan a una persona (ver 09)
SELECT id, tenant_id, subject, held_reason, updated_at FROM inbox_messages
 WHERE state = 'HELD' AND body_purged_at IS NULL ORDER BY updated_at;

-- Entregados como borrador sin área (el LLM falló 4 veces)
SELECT id, subject, ticket_id, delivered_at FROM inbox_messages WHERE fallback ORDER BY delivered_at DESC LIMIT 20;

-- Últimos errores por canal
SELECT channel_id, at, error FROM channel_errors ORDER BY at DESC LIMIT 20;

-- Rendimiento del LLM
SELECT provider, count(*), avg(duration_ms)::int AS avg_ms FROM agent_runs
 WHERE created_at > now() - interval '7 days' GROUP BY 1;
```

Respaldo: `podman exec classifier-db pg_dump -Fc -U admin classifier_db > ~/respaldos/classifier_db_$(date +%F).dump`
(respaldo de todo el ecosistema: infra-monitoring `03_guia_operacional.md` §11).

---

## 7. Cambiar provider de IA

### Qué modelo clasifica hoy

En Rocky: **OCI Generative AI, modelo `openai.gpt-oss-20b`** (`AGENT_PROVIDER=oci`,
`OCI_MODEL=openai.gpt-oss-20b`). Ollama (`llama3.2:3b`) corre pero no se usa.

```bash
curl -s http://localhost:8000/health    # "default_provider":"oci"
curl -s http://localhost:8001/health    # "agent_provider":"oci"
grep -E "^(AGENT_PROVIDER|OCI_MODEL)=" ~/stack_ticket/ticket-classification/.env
```

### Cambiarlo (permanente)

Una sola variable: `AGENT_PROVIDER` en el `.env` (compose la pasa a `langchain-api` como
`MODEL_PROVIDER`). Para cambiar de modelo dentro de OCI, `OCI_MODEL`.

```bash
cd ~/stack_ticket/ticket-classification
cp .env .env.bak-$(date +%F)
nano .env                      # AGENT_PROVIDER=oci|ollama|openai|anthropic|gemini (y OCI_MODEL si aplica)
bash start.sh                  # recrea langchain-api y langchain-agent con el .env nuevo
curl -s http://localhost:8000/health   # "default_provider" = el nuevo
curl -s http://localhost:8001/health   # "agent_provider" = el nuevo
```

`podman-compose restart` **no sirve**: un contenedor reiniciado conserva las variables con
las que se creó (y `langchain-agent` se crea con `podman run --env-file .env` en `start.sh`).

`langchain-api` guarda en caché (Redis, 1 h) las respuestas por prompt; tras cambiar de
modelo, un correo idéntico a uno reciente puede devolver la respuesta anterior.

### Probar un provider sin cambiar el de por defecto

`/ask` acepta `"provider"` en el cuerpo y exige la llave interna (`X-Api-Key` =
`LANGCHAIN_INTERNAL_API_KEY` del `.env`):

```bash
KEY=$(grep ^LANGCHAIN_INTERNAL_API_KEY= ~/stack_ticket/ticket-classification/.env | cut -d= -f2)
curl -s -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" -H "X-Api-Key: $KEY" \
  -d '{"prompt": "Responde solo: ok", "provider": "ollama"}' | python3 -m json.tool
```

### Verificar que una key de provider cloud funciona

```bash
# KEY como en el bloque anterior
for P in oci openai anthropic gemini; do
  echo "== $P"
  curl -s -X POST http://localhost:8000/ask \
    -H "Content-Type: application/json" -H "X-Api-Key: $KEY" \
    -d "{\"prompt\": \"Responde solo: ok\", \"provider\": \"$P\"}" | python3 -m json.tool
done
```

### Requisitos previos para OCI GenAI

OCI usa autenticación por API Key local — no requiere variable de entorno con el secreto,
sino un archivo de configuración en el host:

```bash
# Verificar que ~/.oci/config existe y tiene el perfil [DEFAULT]
cat ~/.oci/config
# Debe mostrar: [DEFAULT], user, fingerprint, tenancy, region, key_file

# El archivo se monta automáticamente como volumen en langchain-api (solo lectura)
# Configurado en docker-compose.yml: ${HOME}/.oci:/root/.oci:ro,z
# (z es obligatorio con SELinux Enforcing en Rocky; ver 08_switch_to_oci_genai.md)
```

Variables relevantes en `.env`:

```ini
AGENT_PROVIDER=oci
OCI_MODEL=openai.gpt-oss-20b
OCI_SERVICE_ENDPOINT=https://inference.generativeai.us-chicago-1.oci.oraclecloud.com
OCI_COMPARTMENT_ID=ocid1.compartment.oc1..tu_ocid_real
OCI_TEMPERATURE=0
OCI_MAX_TOKENS=1400
```

---

## 8. Áreas y categorías

Cada cliente tiene su propio catálogo y lo administra **su admin en el portal** (⚙️
Administración → Gestionar áreas y categorías). El clasificador lo recibe de
ticket-management en cada resolución del canal (caché ≤ 60 s): un cambio en el portal se usa
en los siguientes correos sin tocar este repo. Si el clasificador se reinició y el canal no ha
mandado correos desde entonces, usa el catálogo guardado con el mensaje al recibirlo.

`AGENT_DOMAINS` y `CATEGORIES_*` del `.env` **solo** se usan en modo mock (§9).

**Fuzzy matching:** si el LLM devuelve una categoría parecida (`FUZZY_THRESHOLD=80`) se guarda
la canónica y la propuesta queda en `categoriaPropuesta`.

---

## 9. Modo mock — pruebas sin LLM

Clasifica sin llamar al LLM, con valores fijos (útil en desarrollo; **no** en Rocky):

```bash
# .env
AGENT_MOCK_CLASSIFY=true
MOCK_DOMINIO=IT
MOCK_CATEGORIA=acceso
MOCK_PRIORIDAD=BAJA
MOCK_CONFIANZA=1.0
# el catálogo sale de AGENT_DOMAINS y CATEGORIES_*
bash start.sh
```

---

## 10. Desplegar cambios de código

```bash
cd ~/stack_ticket/ticket-classification
git pull                                   # Rocky sigue origin/develop
podman-compose build langchain-agent       # y/o langchain-api
bash start.sh                              # recrea todo; aplica el parche de langchain-agent
curl -s http://localhost:8001/health
```

Si hay migraciones nuevas, respaldar `classifier_db` antes (§6) y revisar
`podman logs classifier-flyway`.

---

## 11. Ollama

En Rocky no se usa (provider `oci`), pero el contenedor corre.

```bash
podman exec ollama ollama list
podman exec -it ollama ollama pull llama3.2:3b
podman exec -it ollama ollama run llama3.2:3b "Clasifica: servidor caído en producción"
podman stats ollama --no-stream
```

---

## 12. Redis — cola de trabajo y caché

`classifier-redis` guarda la cola `inbox-work` (grupo `classifier-workers`) y la caché de
respuestas del LLM (1 h). **No** es la fuente de verdad: si se pierde, el barrido de cada minuto
reconstruye el trabajo desde `inbox_messages`.

```bash
R='redis-cli --no-auth-warning -a "$REDIS_PASSWORD"'
podman exec classifier-redis sh -c "$R ping"
podman exec classifier-redis sh -c "$R XINFO GROUPS inbox-work"     # pendientes y lag
podman exec classifier-redis sh -c "$R dbsize"
```

Vaciar Redis (`FLUSHALL`) no pierde correos, pero borra la caché del LLM.

---

## 13. MinIO — archivos de los correos

Bucket privado `classifier-inbox`, ruta `t<tenant>/<canal>/<mensaje>/<uuid>`. Los archivos de un
correo se borran a los `CLASSIFIER_RETENTION_DAYS` (30) días de entregado; los de un retenido,
al descartarlo.

```bash
curl -sf http://localhost:9000/minio/health/live && echo "MinIO OK"
# Archivos de un correo (desde la base; la imagen de MinIO no trae el cliente mc)
podman exec classifier-db psql -U admin -d classifier_db -c \
  "SELECT filename, content_type, size, inline, storage_key FROM inbox_attachments WHERE message_id = '<id>';"
```

Consola web: `http://localhost:9001` (por túnel), usuario y contraseña del `.env`.

---

## 14. Migraciones Flyway

```bash
podman logs classifier-flyway        # "Successfully applied N migration(s)" o "Schema ... is up to date"
podman-compose up classifier-flyway  # re-ejecutar
podman exec classifier-db psql -U admin -d classifier_db -c \
  "SELECT version, description, installed_on, success FROM flyway_schema_history ORDER BY installed_rank;"
```

Migraciones vigentes: V1, V2 y V3 (`V3__inbox_schema.sql`, la bandeja).

---

## 15. LangSmith — trazas del agente

```bash
# .env
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=tu-key-de-langsmith
LANGCHAIN_PROJECT=ticket-classification-dev
bash start.sh
```

En https://smith.langchain.com → proyecto: duración, latencia del LLM, errores de validación,
iteraciones y confianza de cada clasificación.

---

## 16. Checklist diario

```bash
cd ~/stack_ticket/ticket-classification
podman ps --format "table {{.Names}}\t{{.Status}}"          # classifier-flyway "Exited (0)" es normal
curl -s http://localhost:8001/health | grep -q '"ticket_management":"ok"' && echo "Agente OK" || echo "Agente ❌"
curl -s http://localhost:8000/health | python3 -m json.tool
podman exec classifier-db psql -U admin -d classifier_db -At -c \
  "SELECT state, count(*) FROM inbox_messages WHERE state <> 'DELIVERED' GROUP BY 1;"   # vacío = nada pendiente
df -h ~ ; free -h | grep Mem
```

---

*ticket-classification · Guía Operacional · Octubre 2026*
