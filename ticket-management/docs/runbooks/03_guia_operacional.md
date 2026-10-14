# 03 — Guía Operacional
## ticket-management · Arranque, verificación y troubleshooting
**Octubre 2026**

---

## Índice

1. [Arranque del stack](#1-arranque-del-stack)
2. [Verificación de salud](#2-verificación-de-salud)
3. [Operaciones del día a día](#3-operaciones-del-día-a-día)
4. [Gestión de la base de datos](#4-gestión-de-la-base-de-datos)
5. [MinIO — adjuntos](#5-minio--adjuntos)
6. [Registry de imágenes — REGISTRY_BASE](#6-registry-de-imágenes--registry_base)
7. [Troubleshooting](#7-troubleshooting)
8. [Referencia rápida](#8-referencia-rápida)

> Los comandos con `localhost` se ejecutan **dentro de Rocky**. Desde Windows,
> las URLs `localhost` funcionan solo con el túnel del administrador
> (`ssh -N rocky-tunel`, ver `infra-monitoring/docs/runbooks/10_redes_y_acceso_lan.md` §7).

---

## 1. Arranque del stack

### Arranque integrado (recomendado)

```bash
cd ~/stack_ticket/infra-monitoring
./startup.sh up
# o:
make up
```

### Arranque independiente

```bash
cd ~/stack_ticket/ticket-management
./start.sh
```

`start.sh` hace:
1. Crea `.env` desde `.env.example` si no existe
2. Crea la red `ticket-management-network` si no existe
3. `podman-compose up -d` en el orden correcto (healthchecks)
4. Muestra estado y puertos

### Parar el stack

```bash
cd ~/stack_ticket/ticket-management
podman-compose down
# Para eliminar volúmenes (borra datos):
podman-compose down -v   # ⚠️ Irreversible — confirmar antes
```

---

## 2. Verificación de salud

### Estado de contenedores

```bash
podman ps --filter name=ticket
```

Deben aparecer: `ticket-system-backend`, `ticket-system-frontend`, `ticket-db`,
`ticket-system-redis`, `ticket-system-minio` con estado **Up**.
(`ticket-flyway` es one-shot: termina en `Exited (0)` tras aplicar migraciones.)

### Health checks

```bash
# Backend — API REST (context-path /api/v1)
curl -s http://localhost:8080/api/v1/actuator/health | python3 -m json.tool

# Frontend — interfaz web
curl -s -o /dev/null -w "%{http_code}" http://localhost:8090

# Base de datos
podman exec ticket-db pg_isready -U ticket_admin -d tickets_db

# Redis (autentica solo si REDIS_PASSWORD está definido en .env)
podman exec ticket-system-redis sh -c 'redis-cli ${REDIS_PASSWORD:+--no-auth-warning -a "$REDIS_PASSWORD"} ping'
# Respuesta esperada: PONG
```

### URLs de acceso

| Servicio | URL |
|---|---|
| Frontend | http://localhost:8090 |
| Backend API | http://localhost:8080/api/v1/ |
| API Swagger | http://localhost:8080/api/v1/swagger-ui.html |
| Prometheus metrics | http://localhost:8080/api/v1/actuator/prometheus |
| MinIO Consola | http://localhost:9003 |

---

## 3. Operaciones del día a día

### Login en el portal

```
http://localhost:8090
Usuario: email del usuario (p. ej. el admin creado en el setup)
Password: el definido al crear el usuario
```

### Crear un ticket manualmente via API

```bash
# Obtener JWT (el login es por email, no por username)
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@empresa.com","password":"TU_PASSWORD"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# Crear ticket
curl -s -X POST http://localhost:8080/api/v1/tickets \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Ticket de prueba",
    "description": "Descripción del problema",
    "priority": "MEDIA"
  }'
```

Campos de `CreateTicketRequest`: `title` y `priority` son obligatorios;
`description`, `draft`, `requesterEmail`, `classificationId` (dominio) y
`categoryId` son opcionales. Dominio y categoría se envían como **IDs**
(tablas `ticket_classifications` y `ticket_categories`), no como texto.

### Crear ticket desde el clasificador (API Key interna)

```bash
curl -s -X POST http://localhost:8080/api/v1/tickets \
  -H "X-Api-Key: el_valor_de_INTERNAL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "...",
    "priority": "ALTA",
    "requesterEmail": "remitente@empresa.com"
  }'
```

### Ver logs del backend

```bash
podman logs --tail 100 ticket-system-backend
podman logs -f ticket-system-backend   # en tiempo real
```

### Reiniciar solo el backend (sin afectar BD ni Redis)

```bash
podman restart ticket-system-backend
```

### Cambiar la INTERNAL_API_KEY

```bash
# 1. Editar .env en ticket-management
# INTERNAL_API_KEY=nuevo_valor

# 2. Editar .env en ticket-classification
# TICKET_MGMT_API_KEY=nuevo_valor (mismo valor)

# 3. Recrear los contenedores afectados (restart no relee el .env)
cd ~/stack_ticket/ticket-management
podman-compose up -d backend          # servicio "backend" → contenedor ticket-system-backend
cd ~/stack_ticket/ticket-classification
podman-compose up -d langchain-agent
```

---

## 4. Gestión de la base de datos

### Conexión directa con DBeaver u otra herramienta

```
Host:     localhost (en Rocky, o desde Windows con el túnel rocky-tunel)
Port:     5433
Database: tickets_db
User:     ticket_admin
Password: POSTGRES_PASSWORD del .env (dev: ticket_password_123)
```

### Consultas útiles

```bash
# Abrir psql
podman exec -it ticket-db psql -U ticket_admin -d tickets_db
```

```sql
\dt                                   -- listar tablas

-- Últimos tickets con su estado (el estado vive en workflow_states)
SELECT t.id, t.title, ws.name AS status, t.priority, t.created_at
FROM tickets t
LEFT JOIN workflow_states ws ON ws.id = t.status_id
ORDER BY t.created_at DESC
LIMIT 20;

-- Tickets abiertos (estados: OPEN, IN_PROGRESS, IN_REVIEW, CLOSED)
SELECT COUNT(*)
FROM tickets t
JOIN workflow_states ws ON ws.id = t.status_id
WHERE ws.name = 'OPEN';

\q                                    -- salir
```

### Estado de migraciones Flyway

Hay dos ejecuciones de Flyway: el contenedor one-shot `ticket-flyway` y el
propio backend (`spring.flyway.enabled=true`). Ambas registran en la misma
tabla:

```bash
podman exec ticket-db psql -U ticket_admin -d tickets_db -c \
  "SELECT version, description, success, installed_on FROM flyway_schema_history ORDER BY installed_rank;"
```

En desarrollo, con Maven instalado, también: `mvn flyway:info` / `mvn flyway:validate`.

### Backup de la BD

```bash
podman exec ticket-db pg_dump -U ticket_admin tickets_db > backup_$(date +%Y%m%d).sql
```

---

## 5. MinIO — adjuntos

Contenedor `ticket-system-minio`, bucket `ticket-attachments` (`MINIO_BUCKET`).
No confundir con `minio` (9000/9001), que es el MinIO de ticket-classification.

### Acceder a la consola web

```
http://localhost:9003
User:     MINIO_ACCESS_KEY del .env (dev: minioadmin)
Password: MINIO_SECRET_KEY del .env (dev: minioadmin_password_123)
```

Desde la consola: **Object Browser → ticket-attachments** para listar,
descargar o eliminar adjuntos.

### Ver adjuntos via CLI

La imagen `localhost/minio:local` se compila desde fuente y **no incluye `mc`**.
Para usar CLI hace falta `mc` instalado en el host, con un alias hacia la API (9002):

```bash
mc alias set tm http://localhost:9002 minioadmin 'MINIO_SECRET_KEY'
mc ls tm/ticket-attachments/
```

### Limpiar adjuntos huérfanos

Si un ticket fue eliminado pero sus adjuntos siguen en MinIO:

```bash
mc ls tm/ticket-attachments/
mc rm tm/ticket-attachments/nombre-del-archivo
```

(O desde la consola web: seleccionar el objeto → Delete.)

---

## 6. Registry de imágenes — REGISTRY_BASE

### Concepto

Las imágenes del frontend y backend se nombran así en `docker-compose.yml`:

```
${REGISTRY_BASE:-docker.io}/ticket/ticket-management-frontend:${IMAGE_TAG:-1.0.0}
${REGISTRY_BASE:-docker.io}/ticket/ticket-management-backend:${IMAGE_TAG:-1.0.0}
```

`REGISTRY_BASE` no cambia **dónde** se construye (siempre es local con
`podman-compose build`): define **cómo se nombra** la imagen y, por tanto,
**a qué registry** va un `podman-compose push`.

| Entorno | `REGISTRY_BASE` | Resultado |
|---|---|---|
| Rocky (valor activo en `.env.example`) | `mty.ocir.io/axalfzdz0tik/rnum/noprod/tmp-02` | Imágenes listas para push a OCIR |
| WSL2 (desarrollo) | `docker.io` | Nombre local; no se publica |
| Sin definir | `docker.io` (default del compose) | Igual que WSL2 |

### Verificar el valor actual

```bash
cd ~/stack_ticket/ticket-management
grep '^REGISTRY_BASE\|^IMAGE_TAG' .env
```

### Construir y publicar en OCIR

```bash
cd ~/stack_ticket/ticket-management
podman login mty.ocir.io          # una vez por sesión
podman-compose build
podman-compose push
```

Para construir solo en local sin riesgo de publicar, poner
`REGISTRY_BASE=docker.io` en el `.env` (y no ejecutar `push`).

---

## 7. Troubleshooting

### El backend no arranca

```bash
podman logs ticket-system-backend | tail -50

# Causa frecuente: ticket-db no está healthy aún
podman exec ticket-db pg_isready -U ticket_admin -d tickets_db
# Si falla: esperar ~30 segundos y reintentar
```

### Flyway falla al arrancar

```bash
podman logs ticket-flyway 2>&1 | tail -30
podman logs ticket-system-backend 2>&1 | grep -i flyway | tail -20

# Causa: checksum cambiado en migración ya aplicada
# NUNCA modificar V__*.sql ya aplicados
# Si fue accidental: restaurar el archivo original o limpiar flyway_schema_history
podman exec -it ticket-db psql -U ticket_admin -d tickets_db \
  -c "DELETE FROM flyway_schema_history WHERE version='X';"
# Luego reiniciar el backend para que Flyway (spring.flyway.enabled=true) reaplique
```

### Redis no responde

```bash
podman logs ticket-system-redis | tail -20

# Probar conexión (autentica solo si REDIS_PASSWORD está definido)
podman exec ticket-system-redis sh -c 'redis-cli ${REDIS_PASSWORD:+--no-auth-warning -a "$REDIS_PASSWORD"} ping'
# NOAUTH → REDIS_PASSWORD definido en el servidor pero no se pasó al cliente

# Reiniciar Redis (los streams persisten en el volumen)
podman restart ticket-system-redis
```

### El clasificador no puede registrar tickets (401/403)

```bash
# Verificar que la API key coincide en ambos repos
# En ticket-management:
podman exec ticket-system-backend env | grep INTERNAL_API_KEY

# En ticket-classification:
podman exec langchain-agent env | grep TICKET_MGMT_API_KEY

# Los valores deben ser idénticos. El header que valida el backend es X-Api-Key.
```

### Frontend en blanco o error de CORS

```bash
# Verificar CORS_ALLOWED_ORIGINS en el backend
podman exec ticket-system-backend env | grep CORS

# El origen desde el que se abre el frontend debe estar incluido
# (p. ej. http://localhost:8090 o http://10.187.21.30:8090)
```

---

## 8. Referencia rápida

| Acción | Comando / URL |
|---|---|
| Arrancar stack | `./start.sh` |
| Parar stack | `podman-compose down` |
| Frontend | http://localhost:8090 |
| Backend health | http://localhost:8080/api/v1/actuator/health |
| Backend Swagger | http://localhost:8080/api/v1/swagger-ui.html |
| MinIO Consola | http://localhost:9003 |
| Logs backend | `podman logs --tail 100 ticket-system-backend` |
| Logs frontend | `podman logs --tail 50 ticket-system-frontend` |
| Reiniciar backend | `podman restart ticket-system-backend` |
| Abrir psql | `podman exec -it ticket-db psql -U ticket_admin -d tickets_db` |
| Redis ping | `podman exec ticket-system-redis sh -c 'redis-cli ${REDIS_PASSWORD:+--no-auth-warning -a "$REDIS_PASSWORD"} ping'` |

---

*ticket-management · Guía Operacional · Octubre 2026*
