# 03 — Guía Operacional
## ticket-classification · Operaciones del día a día
**Ruta raíz: `~/stack_ticket/ticket-classification`**
**Mayo 2026**

---

## Índice

1. [Regla de oro](#1-regla-de-oro)
2. [Ciclo de vida del stack](#2-ciclo-de-vida-del-stack)
3. [Verificación de salud del sistema](#3-verificación-de-salud-del-sistema)
4. [Probar el clasificador](#4-probar-el-clasificador)
5. [Gestión de logs](#5-gestión-de-logs)
6. [Base de datos — operaciones frecuentes](#6-base-de-datos--operaciones-frecuentes)
7. [Cambiar provider de IA](#7-cambiar-provider-de-ia)
8. [Agregar o modificar dominios y categorías](#8-agregar-o-modificar-dominios-y-categorías)
9. [Modo mock — desarrollo sin LLM](#9-modo-mock--desarrollo-sin-llm)
10. [Ciclo de desarrollo — cambios en código](#10-ciclo-de-desarrollo--cambios-en-código)
11. [Gestión de Ollama](#11-gestión-de-ollama)
12. [Redis — caché y estado de jobs](#12-redis--caché-y-estado-de-jobs)
13. [MinIO — adjuntos de correos](#13-minio--adjuntos-de-correos)
14. [Migraciones Flyway](#14-migraciones-flyway)
15. [LangSmith — trazas del agente](#15-langsmith--trazas-del-agente)
16. [Git — control de versiones](#16-git--control-de-versiones)
17. [Troubleshooting](#17-troubleshooting)
18. [Checklist de verificación diaria](#18-checklist-de-verificación-diaria)

---

## 1. Regla de oro

> **Siempre ejecutar desde la raíz del repo:**
> ```bash
> cd ~/stack_ticket/ticket-classification
> ```
> Los servicios se comunican por nombre de contenedor (`langchain-api:8000`),
> nunca por `localhost` dentro de los contenedores.

---

## 2. Ciclo de vida del stack

### Levantar — opción recomendada (ecosistema completo)

```bash
# Levanta todos los repos en orden con las redes correctas
bash ~/stack_ticket/infra-monitoring/startup.sh up
```

### Levantar — solo este repo

```bash
# start.sh detecta si ticket-management está corriendo y configura la URL
bash ~/stack_ticket/ticket-classification/start.sh
```

### Apagar este repo (datos se conservan)

```bash
cd ~/stack_ticket/ticket-classification
podman-compose down
```

### Apagar y eliminar volúmenes (reset completo — PRECAUCIÓN)

```bash
cd ~/stack_ticket/ticket-classification
# ⚠️ Borra datos de classifier-db, redis y minio
podman-compose down -v
```

### Reiniciar un servicio específico

```bash
cd ~/stack_ticket/ticket-classification
podman-compose restart langchain-agent   # Solo el agente
podman-compose restart langchain-api     # Solo la API
podman-compose restart classifier-db     # Solo la BD
podman-compose restart ollama            # Solo Ollama
```

### Estado de contenedores

```bash
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Networks}}"
podman stats --no-stream   # CPU y RAM en snapshot
```

---

## 3. Verificación de salud del sistema

```bash
cd ~/stack_ticket/ticket-classification

echo "════════════════════════════════════"
echo "ESTADO — $(date)"
echo "════════════════════════════════════"

echo -e "\n── Contenedores ──"
podman ps --format "table {{.Names}}\t{{.Status}}"

echo -e "\n── Agente ──"
curl -s http://localhost:8001/health | python3 -m json.tool 2>/dev/null || echo "NO RESPONDE"

echo -e "\n── API (providers activos) ──"
curl -s http://localhost:8000/health | python3 -m json.tool 2>/dev/null || echo "NO RESPONDE"

echo -e "\n── PostgreSQL ──"
podman exec classifier-db psql -U admin -d classifier_db -c "SELECT COUNT(*) FROM tickets;" -t \
  && echo "classifier_db OK" || echo "classifier_db NO DISPONIBLE"

echo -e "\n── Redis ──"
podman exec classifier-redis redis-cli ping 2>/dev/null || echo "Redis NO DISPONIBLE"

echo -e "\n── MinIO ──"
curl -sf http://localhost:9000/minio/health/live && echo "MinIO OK" || echo "MinIO NO DISPONIBLE"
```

---

## 4. Probar el clasificador

El endpoint es **asíncrono**: `POST /process` devuelve un `job_id` y la
clasificación se hace en background. Consultar el resultado con `GET /status/{job_id}`.

### Ticket de IT

```bash
# Paso 1 — enviar
JOB=$(curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{
    "asunto": "Servidor de producción no responde",
    "cuerpo": "El servidor ERP no responde desde las 9am. Usuarios sin acceso.",
    "remitente": "soporte@empresa.com",
    "nombre_remitente": "Soporte TI",
    "conversation_id": "TEST-IT-001"
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
echo "Job ID: $JOB"

# Paso 2 — esperar y consultar (~25s Ollama, <5s cloud)
sleep 30
curl -s http://localhost:8001/status/$JOB | python3 -m json.tool
```

### Ticket de cliente

```bash
JOB=$(curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{
    "asunto": "Cargo duplicado en factura de marzo",
    "cuerpo": "Mi factura tiene un cargo duplicado de $500. Por favor revisar.",
    "remitente": "cliente@externo.com",
    "nombre_remitente": "Cliente Externo",
    "conversation_id": "TEST-CLI-001"
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
sleep 30 && curl -s http://localhost:8001/status/$JOB | python3 -m json.tool
```

### Ticket con adjunto

```bash
JOB=$(curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{
    "asunto": "Error en sistema de nómina",
    "cuerpo": "El sistema no permite procesar pagos desde esta mañana.",
    "remitente": "juan.perez@empresa.com",
    "nombre_remitente": "Juan Pérez",
    "conversation_id": "TEST-ADJ-001",
    "adjuntos": [
      {"nombre": "captura_error.png", "tipo": "image/png"}
    ]
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
sleep 30 && curl -s http://localhost:8001/status/$JOB | python3 -m json.tool
```

### Probar con provider específico

```bash
JOB=$(curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{
    "asunto": "Sistema de reportes sin actualizar",
    "cuerpo": "El BI lleva 2 horas sin actualizar los dashboards.",
    "remitente": "operaciones@empresa.com",
    "nombre_remitente": "Operaciones BI",
    "conversation_id": "TEST-OAI-001",
    "provider": "openai"
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
sleep 10 && curl -s http://localhost:8001/status/$JOB | python3 -m json.tool
```

### Verificar deduplicación por conversation_id

```bash
# Enviar el mismo conversation_id dos veces
JOB1=$(curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{"asunto":"Test dedup","cuerpo":"Primer envío","conversation_id":"DEDUP-001"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
sleep 30

JOB2=$(curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{"asunto":"Test dedup","cuerpo":"Segunda respuesta al mismo hilo","conversation_id":"DEDUP-001"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
sleep 5

# El segundo debe devolver status=ignorado
curl -s http://localhost:8001/status/$JOB2 | python3 -m json.tool
```

---

## 5. Gestión de logs

```bash
# Agente — tiempo real
podman logs -f langchain-agent

# API — tiempo real
podman logs -f langchain-api

# Últimas N líneas
podman logs --tail 50 langchain-agent
podman logs --tail 50 langchain-api

# Flyway — ver resultado de migraciones
podman logs classifier-flyway

# Buscar errores en el agente
podman logs langchain-agent 2>&1 | grep -i "error\|exception"

# Buscar clasificaciones exitosas
podman logs langchain-agent 2>&1 | grep "completado"

# Todos los servicios en tiempo real
podman-compose logs -f
```

---

## 6. Base de datos — operaciones frecuentes

### Conectarse a classifier-db

```bash
podman exec -it classifier-db psql -U admin -d classifier_db
# Para salir: \q
# Ver tablas: \dt
```

### Consultas de operación

```bash
# Últimos 10 tickets clasificados
podman exec classifier-db psql -U admin -d classifier_db -c "
SELECT id, subject, domain, category, priority, confidence, created_at
FROM tickets
ORDER BY id DESC LIMIT 10;"

# Tickets del día por dominio
podman exec classifier-db psql -U admin -d classifier_db -c "
SELECT domain, priority, COUNT(*) AS total
FROM tickets
WHERE created_at >= CURRENT_DATE
GROUP BY domain, priority
ORDER BY domain, priority;"

# Tickets que requieren revisión manual
podman exec classifier-db psql -U admin -d classifier_db -c "
SELECT id, subject, domain, category, suggested_category, sender, created_at
FROM tickets
WHERE requires_review = true
ORDER BY created_at DESC;"

# Buscar ticket por job_id
podman exec classifier-db psql -U admin -d classifier_db -c "
SELECT t.id, t.subject, t.domain, t.category, t.priority,
       r.run_id AS job_id, r.iterations_used, r.is_validated, r.duration_ms
FROM tickets t
JOIN agent_runs r ON r.ticket_id = t.id
WHERE r.run_id = 'PEGA-EL-JOB-ID-AQUI';"

# Performance del agente por provider
podman exec classifier-db psql -U admin -d classifier_db -c "
SELECT provider, AVG(duration_ms) AS avg_ms, COUNT(*) AS total
FROM agent_runs
WHERE is_validated = true
GROUP BY provider;"
```

### Backup

```bash
podman exec classifier-db pg_dump -U admin classifier_db \
  > ~/stack_ticket/ticket-classification/backup_$(date +%Y%m%d_%H%M%S).sql
```

---

## 7. Cambiar provider de IA

### Por request (sin reiniciar)

```bash
# Pasar el campo "provider" en el body
curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{
    "asunto": "Mi asunto",
    "cuerpo": "Mi cuerpo",
    "provider": "anthropic"
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])"

# Providers disponibles: ollama | openai | anthropic | gemini
```

### Por defecto permanente

```bash
cd ~/stack_ticket/ticket-classification

# 1. Editar .env
nano .env
# Cambiar: AGENT_PROVIDER=openai

# 2. Reiniciar agente y API
podman-compose restart langchain-agent langchain-api

# 3. Verificar
curl -s http://localhost:8001/health | python3 -m json.tool
# "agent_provider": "openai"
```

### Verificar que una key de provider cloud funciona

```bash
# Probar OpenAI directamente contra la API
curl -s -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Responde solo: ok", "provider": "openai"}' \
  | python3 -m json.tool

# Probar Anthropic
curl -s -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Responde solo: ok", "provider": "anthropic"}' \
  | python3 -m json.tool
```

---

## 8. Agregar o modificar dominios y categorías

```bash
cd ~/stack_ticket/ticket-classification

# 1. Editar .env
nano .env
# Ejemplo — agregar dominio RRHH:
# AGENT_DOMAINS=IT,cliente,operaciones,RRHH,otro
# CATEGORIES_RRHH=vacaciones,permisos,nomina,capacitacion,onboarding

# 2. Reiniciar solo el agente
podman-compose restart langchain-agent

# 3. Verificar dominios activos
curl -s http://localhost:8001/health | python3 -m json.tool
# "agent_domains": ["IT", "cliente", "operaciones", "RRHH", "otro"]
```

**Fuzzy matching:** si el LLM devuelve `"categoría": "vacacion"` (sin 's')
y el umbral `FUZZY_THRESHOLD=80` lo acepta, guarda `"vacaciones"` como
categoría canónica y `"vacacion"` como `suggested_category`.

---

## 9. Modo mock — desarrollo sin LLM

Útil para probar el flujo completo sin consumir tokens ni esperar al LLM.

```bash
# 1. Activar en .env
nano ~/stack_ticket/ticket-classification/.env
# AGENT_MOCK_CLASSIFY=true
# MOCK_DOMINIO=IT
# MOCK_CATEGORIA=acceso
# MOCK_PRIORIDAD=BAJA
# MOCK_CONFIANZA=1.0

# 2. Reiniciar agente
cd ~/stack_ticket/ticket-classification
podman-compose restart langchain-agent

# 3. Probar — responde en <1s sin llamar al LLM
JOB=$(curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{"asunto":"Test mock","cuerpo":"Prueba rápida sin LLM"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
sleep 2
curl -s http://localhost:8001/status/$JOB | python3 -m json.tool
```

---

## 10. Ciclo de desarrollo — cambios en código

### Cambios en langchain-agent

```bash
cd ~/stack_ticket/ticket-classification

# Editar código (hot reload activo — el volumen monta ./langchain-agent:/app)
# Solo reiniciar el contenedor si cambió requirements.txt o el Dockerfile

# Si cambió solo código Python → reiniciar el proceso:
podman-compose restart langchain-agent

# Si cambió requirements.txt → reconstruir imagen:
podman-compose build langchain-agent
podman-compose up -d langchain-agent
podman logs -f langchain-agent
# Buscar: "Uvicorn running on http://0.0.0.0:8001"
```

### Cambios en langchain-api

```bash
cd ~/stack_ticket/ticket-classification
podman-compose build langchain-api
podman-compose up -d langchain-api
podman logs -f langchain-api
```

### Reconstrucción completa

```bash
cd ~/stack_ticket/ticket-classification
podman-compose down
podman-compose build --no-cache
podman-compose up -d
```

---

## 11. Gestión de Ollama

```bash
# Ver modelos instalados
podman exec ollama ollama list

# Descargar modelo (primera vez o actualizar)
podman exec -it ollama ollama pull llama3.2:3b

# Modelo más capaz (más lento, requiere más RAM)
podman exec -it ollama ollama pull llama3.1:8b

# Probar modelo directamente
podman exec -it ollama ollama run llama3.2:3b "Clasifica: servidor caído en producción"
# Ctrl+D para salir

# Eliminar modelo (liberar espacio en disco)
podman exec ollama ollama rm llama3.1:8b

# Ver RAM durante inferencia
podman stats ollama --no-stream
```

---

## 12. Redis — caché y estado de jobs

```bash
# Verificar que Redis responde
podman exec classifier-redis redis-cli ping
# PONG

# Número de keys en cache
podman exec classifier-redis redis-cli dbsize

# Ver todos los jobs activos
podman exec classifier-redis redis-cli keys "job:*"

# Ver estado de un job específico
podman exec classifier-redis redis-cli get "job:PEGA-EL-JOB-ID-AQUI"

# Limpiar caché LLM (fuerza re-clasificación en siguientes requests)
# ⚠️ También elimina el estado de jobs en proceso
podman exec classifier-redis redis-cli flushdb

# TTL restante de un job (en segundos — máximo 86400 = 24h)
podman exec classifier-redis redis-cli ttl "job:PEGA-EL-JOB-ID-AQUI"
```

---

## 13. MinIO — adjuntos de correos

```bash
# Verificar salud
curl -sf http://localhost:9000/minio/health/live && echo "MinIO OK"

# Consola web (navegador Windows)
# http://localhost:9001  →  minioadmin / minioadmin123

# Listar archivos del bucket desde WSL
podman exec minio mc alias set local http://localhost:9000 minioadmin minioadmin123
podman exec minio mc ls local/email-attachments/

# Ver adjuntos de un ticket específico
podman exec classifier-db psql -U admin -d classifier_db -c "
SELECT a.filename, a.content_type, a.storage_key, a.created_at
FROM attachments a
WHERE a.ticket_id = TU_TICKET_ID
ORDER BY a.created_at;"
```

---

## 14. Migraciones Flyway

```bash
# Ver logs de la última migración
podman logs classifier-flyway
# Resultado esperado:
# Successfully applied N migration(s) to schema "public"

# Forzar re-ejecución de migraciones (si classifier-db fue recreado)
podman-compose up classifier-flyway

# Ver estado de migraciones directamente en la BD
podman exec classifier-db psql -U admin -d classifier_db -c "
SELECT version, description, installed_on, success
FROM flyway_schema_history
ORDER BY installed_rank;"
```

---

## 15. LangSmith — trazas del agente

```bash
# Activar trazas en .env
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=tu-key-de-langsmith
LANGCHAIN_PROJECT=ticket-classification-dev

# Reiniciar agente y API para que tomen la configuración
cd ~/stack_ticket/ticket-classification
podman-compose restart langchain-agent langchain-api
```

En https://smith.langchain.com → proyecto `ticket-classification-dev`:

| Dato | Qué indica |
|---|---|
| Total duration | Tiempo total de clasificación |
| Nodo classify → latencia | Tiempo de respuesta del LLM |
| Nodo validate → error | Problemas de formato JSON o schema |
| iterations_used | Cuántos intentos necesitó el agente |
| confidence | Certeza del LLM en su respuesta |

---

## 16. Git — control de versiones

```bash
cd ~/stack_ticket/ticket-classification

git status
git diff

# Convención de commits (Conventional Commits en español imperativo)
git add langchain-agent/agent.py
git commit -m "fix(agent): corregir validación de confianza mínima"

git push origin develop   # Rama de trabajo — nunca pushear directo a main
```

**Ramas (Git Flow):**

| Prefijo | Uso |
|---|---|
| `feature/` | Nueva funcionalidad |
| `bugfix/` | Corrección de bug |
| `hotfix/` | Fix urgente en producción |
| `chore/` | Configuración, dependencias |
| `docs/` | Solo documentación |

---

## 17. Troubleshooting

### El agente devuelve 502 en `/ask`

```bash
# Ver logs de langchain-api con el error real
podman logs --tail 50 langchain-api 2>&1 | grep -i "error\|provider"
# El error se loguea con traceback completo gracias al logger configurado

# Causas comunes:
# 1. API key del provider inválida o expirada → verificar .env
podman exec langchain-api env | grep -E "MODEL_PROVIDER|OPENAI|ANTHROPIC|GEMINI|OLLAMA"

# 2. Ollama sin modelo descargado
podman exec ollama ollama list

# 3. Ollama sin recursos suficientes
free -h   # Verificar RAM disponible (>4GB recomendado)
```

### Job queda en `en_proceso` indefinidamente

```bash
# Ver logs del agente en el momento del error
podman logs --tail 100 langchain-agent 2>&1 | grep -i "error\|job"

# Verificar Redis
podman exec classifier-redis redis-cli get "job:TU-JOB-ID"

# Causas comunes:
# 1. classifier-db no disponible
podman ps | grep classifier-db

# 2. ticket-system-backend no alcanzable (en nodo save)
podman exec langchain-agent sh -c 'python3 -c "import urllib.request; urllib.request.urlopen(\"http://ticket-system-backend:8080/actuator/health\")"' 2>&1
```

### classifier-db no acepta conexiones

```bash
podman logs --tail 20 classifier-db
podman-compose restart classifier-db
sleep 15
podman exec classifier-db psql -U admin -d classifier_db -c "SELECT version();"
```

### Flyway falla al migrar

```bash
podman logs classifier-flyway
# Si el error es "duplicate key" → la migración ya fue aplicada parcialmente
# Si el error es "connection refused" → classifier-db no terminó de iniciar

# Re-ejecutar Flyway
podman-compose up classifier-flyway
```

### "network not found" al hacer podman-compose up

```bash
# Crear las redes manualmente
podman network create --subnet 10.89.1.0/24 ticket-management-network
podman network create --subnet 10.89.2.0/24 ticket-classification-network
# Luego reintentar
podman-compose up -d
```

---

## 18. Checklist de verificación diaria

```bash
cd ~/stack_ticket/ticket-classification

# ✅ 1. Contenedores corriendo
podman ps --format "table {{.Names}}\t{{.Status}}"
# classifier-flyway puede estar Exited 0 — es normal (one-shot)

# ✅ 2. Agente responde
curl -s http://localhost:8001/health | grep -q "ok" && echo "Agente: OK" || echo "Agente: ❌"

# ✅ 3. API responde y provider configurado
curl -s http://localhost:8000/health | python3 -m json.tool

# ✅ 4. Clasificación funciona end-to-end
JOB=$(curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{"asunto":"Test diario","cuerpo":"Verificación de salud del clasificador","conversation_id":"DAILY-CHECK"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
sleep 30
curl -s http://localhost:8001/status/$JOB \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('Clasificador:', d.get('status'), '| dominio:', d.get('domain','?'))"

# ✅ 5. PostgreSQL accesible
podman exec classifier-db psql -U admin -d classifier_db -c "SELECT COUNT(*) FROM tickets;" -t \
  && echo "PostgreSQL: OK" || echo "PostgreSQL: ❌"

# ✅ 6. Redis activo
podman exec classifier-redis redis-cli ping | grep -q "PONG" && echo "Redis: OK" || echo "Redis: ❌"

# ✅ 7. MinIO accesible
curl -sf http://localhost:9000/minio/health/live && echo "MinIO: OK" || echo "MinIO: ❌"

# ✅ 8. Disco disponible
df -h ~/stack_ticket/ticket-classification

# ✅ 9. RAM disponible (>4GB recomendado si usas Ollama)
free -h | grep Mem
```

---

## Referencia rápida de comandos

```bash
# ── Stack ──────────────────────────────────────────────────────────
bash ~/stack_ticket/ticket-classification/start.sh      # Levantar
cd ~/stack_ticket/ticket-classification && podman-compose down   # Apagar
podman-compose restart langchain-agent                  # Reiniciar servicio
podman ps                                               # Estado
podman stats --no-stream                                # CPU y RAM

# ── Logs ───────────────────────────────────────────────────────────
podman logs --tail 50 langchain-agent
podman logs --tail 50 langchain-api
podman logs classifier-flyway

# ── Clasificador (flujo asíncrono) ─────────────────────────────────
# Paso 1: enviar
JOB=$(curl -s -X POST http://localhost:8001/process \
  -H "Content-Type: application/json" \
  -d '{"asunto":"ASUNTO","cuerpo":"CUERPO","remitente":"user@empresa.com"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
# Paso 2: consultar
curl -s http://localhost:8001/status/$JOB | python3 -m json.tool

# ── Base de datos ──────────────────────────────────────────────────
podman exec -it classifier-db psql -U admin -d classifier_db
podman exec classifier-db psql -U admin -d classifier_db \
  -c "SELECT id, subject, domain, category, priority FROM tickets ORDER BY id DESC LIMIT 5;"

# ── Redis ──────────────────────────────────────────────────────────
podman exec classifier-redis redis-cli ping
podman exec classifier-redis redis-cli dbsize
podman exec classifier-redis redis-cli keys "job:*"

# ── Ollama ─────────────────────────────────────────────────────────
podman exec ollama ollama list
podman exec -it ollama ollama pull llama3.2:3b

# ── Desarrollo ─────────────────────────────────────────────────────
podman-compose build langchain-agent && podman-compose up -d langchain-agent
```

---

*ticket-classification · Guía Operacional · Mayo 2026*
