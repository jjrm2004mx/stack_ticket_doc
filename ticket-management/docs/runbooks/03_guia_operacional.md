# 03 — Guía Operacional
## ticket-management · Arranque, verificación y troubleshooting
**Mayo 2026**

---

## Índice

1. [Arranque del stack](#1-arranque-del-stack)
2. [Verificación de salud](#2-verificación-de-salud)
3. [Operaciones del día a día](#3-operaciones-del-día-a-día)
4. [Gestión de la base de datos](#4-gestión-de-la-base-de-datos)
5. [MinIO — adjuntos](#5-minio--adjuntos)
6. [Troubleshooting](#6-troubleshooting)
7. [Referencia rápida](#7-referencia-rápida)

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
`ticket-system-redis`, `minio` con estado **Up**.

### Health checks

```bash
# Backend — API REST
curl -s http://localhost:8080/api/v1/actuator/health | python3 -m json.tool

# Frontend — interfaz web
curl -s -o /dev/null -w "%{http_code}" http://localhost:8090

# Base de datos
podman exec ticket-db pg_isready -U ticket_admin -d tickets_db

# Redis
podman exec ticket-system-redis redis-cli ping
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
Usuario: admin (o el configurado)
Password: el definido al crear el usuario
```

### Crear un ticket manualmente via API

```bash
# Obtener JWT
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin_password"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# Crear ticket
curl -s -X POST http://localhost:8080/api/v1/tickets \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "titulo": "Ticket de prueba",
    "descripcion": "Descripción del problema",
    "dominio": "IT",
    "categoria": "acceso",
    "prioridad": "MEDIA"
  }'
```

### Crear ticket desde el clasificador (API Key interna)

```bash
curl -s -X POST http://localhost:8080/api/v1/tickets \
  -H "X-Internal-Api-Key: el_valor_de_INTERNAL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "titulo": "...",
    "dominio": "IT",
    "categoria": "acceso",
    "prioridad": "ALTA",
    "solicitante_email": "remitente@empresa.com"
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

# 3. Reiniciar backends afectados
podman restart ticket-system-backend
cd ~/stack_ticket/ticket-classification
podman-compose restart langchain-agent
```

---

## 4. Gestión de la base de datos

### Conexión directa con DBeaver u otra herramienta

```
Host:     localhost (o IP Windows desde WSL)
Port:     5433
Database: tickets_db
User:     ticket_admin
Password: ticket_password_123 (dev)
```

### Consultas útiles

```bash
# Abrir psql
podman exec -it ticket-db psql -U ticket_admin -d tickets_db

# Dentro de psql:
\dt                           -- listar tablas
SELECT * FROM tickets LIMIT 10;
SELECT id, titulo, estado, created_at FROM tickets ORDER BY created_at DESC LIMIT 20;
SELECT COUNT(*) FROM tickets WHERE estado = 'ABIERTO';
\q                            -- salir
```

### Estado de migraciones Flyway

```bash
# Requiere Maven instalado localmente
cd ~/stack_ticket/ticket-management
mvn flyway:info

# Validar integridad de migraciones aplicadas
mvn flyway:validate
```

### Backup de la BD

```bash
podman exec ticket-db pg_dump -U ticket_admin tickets_db > backup_$(date +%Y%m%d).sql
```

---

## 5. MinIO — adjuntos

### Acceder a la consola web

```
http://localhost:9003
User: minioadmin
Password: minioadmin_password_123 (dev)
```

### Ver adjuntos de un ticket via CLI

```bash
# Listar objetos en el bucket
podman exec minio mc ls local/ticket-attachments/

# O desde wsL con mc instalado
mc ls http://localhost:9002/ticket-attachments/
```

### Limpiar adjuntos huérfanos

Si un ticket fue eliminado pero sus adjuntos siguen en MinIO:

```bash
# Listar todos los objetos
podman exec minio mc ls local/ticket-attachments/

# Eliminar un adjunto específico
podman exec minio mc rm local/ticket-attachments/nombre-del-archivo
```

---

## 6. Troubleshooting

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

# Causa: checksum cambiado en migración ya aplicada
# NUNCA modificar V__*.sql ya aplicados
# Si fue accidental: restaurar el archivo original o limpiar flyway_schema_history
podman exec -it ticket-db psql -U ticket_admin -d tickets_db \
  -c "DELETE FROM flyway_schema_history WHERE version='X';"
# Luego reiniciar el backend para que Flyway reaplique
```

### Redis no responde

```bash
podman logs ticket-system-redis | tail -20

# Probar conexión
podman exec ticket-system-redis redis-cli ping

# Reiniciar Redis (los streams persisten en el volumen)
podman restart ticket-system-redis
```

### El clasificador no puede registrar tickets (403)

```bash
# Verificar que INTERNAL_API_KEY coincide en ambos repos
# En ticket-management:
podman exec ticket-system-backend env | grep INTERNAL_API_KEY

# En ticket-classification:
podman exec langchain-agent env | grep TICKET_MGMT_API_KEY

# Los valores deben ser idénticos
```

### Frontend en blanco o error de CORS

```bash
# Verificar CORS_ALLOWED_ORIGINS en el backend
podman exec ticket-system-backend env | grep CORS

# El origen del frontend (http://localhost:8090) debe estar incluido
```

---

## 7. Referencia rápida

| Acción | Comando / URL |
|---|---|
| Arrancar stack | `./start.sh` |
| Parar stack | `podman-compose down` |
| Frontend | http://localhost:8090 |
| Backend Swagger | http://localhost:8080/api/v1/swagger-ui.html |
| MinIO Consola | http://localhost:9003 |
| Logs backend | `podman logs --tail 100 ticket-system-backend` |
| Logs frontend | `podman logs --tail 50 ticket-system-frontend` |
| Reiniciar backend | `podman restart ticket-system-backend` |
| Abrir psql | `podman exec -it ticket-db psql -U ticket_admin -d tickets_db` |
| Redis ping | `podman exec ticket-system-redis redis-cli ping` |

---

*ticket-management · Guía Operacional · Mayo 2026*
