# 03 — Guía Operacional
## notification-service · Arranque, verificación y troubleshooting
**Mayo 2026**

---

## Índice

1. [Arranque del stack](#1-arranque-del-stack)
2. [Verificación de salud](#2-verificación-de-salud)
3. [Operaciones del día a día](#3-operaciones-del-día-a-día)
4. [Gestión de plantillas](#4-gestión-de-plantillas)
5. [Troubleshooting](#5-troubleshooting)
6. [Referencia rápida](#6-referencia-rápida)

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
cd ~/stack_ticket/notification-service
./start.sh
```

`start.sh` hace:
1. Lee `APP_PUBLIC_URL` de `~/stack_ticket/.env.deploy` (creado por startup.sh)
2. Crea `.env` desde `.env.example` si no existe
3. Inyecta `NOTIFICATION_PORTAL_URL` en `.env` con la URL detectada
4. `podman-compose down` → `podman-compose up -d`

> **Prerequisito:** `ticket-management-network` debe existir (creada por
> ticket-management o startup.sh) porque el servicio necesita alcanzar
> `ticket-system-redis`.

### Parar el stack

```bash
cd ~/stack_ticket/notification-service
podman-compose down
```

---

## 2. Verificación de salud

### Estado de contenedores

```bash
podman ps --filter name=notification
```

Deben aparecer: `notification-service` y `notification-db` con estado **Up**.

### Health checks

```bash
# Servicio principal
curl -s http://localhost:8081/actuator/health | python3 -m json.tool

# Base de datos
podman exec notification-db pg_isready -U notif_admin -d notifications_db
```

### Verificar conexión a Redis Streams

```bash
# El servicio debe logear al arrancar que conectó al stream
podman logs --tail 30 notification-service | grep -i redis
# Esperado: "Connected to Redis Stream ticket.events"
```

---

## 3. Operaciones del día a día

### Ver logs en tiempo real

```bash
podman logs -f notification-service
```

### Ver logs de las últimas notificaciones enviadas

```bash
podman logs --tail 100 notification-service | grep -i "notification\|sent\|error"
```

### Consultar historial de notificaciones en BD

```bash
podman exec -it notification-db psql -U notif_admin -d notifications_db

# Dentro de psql:
SELECT * FROM notification_logs ORDER BY created_at DESC LIMIT 20;
SELECT tipo, COUNT(*) FROM notification_logs GROUP BY tipo;
SELECT * FROM notification_logs WHERE status = 'ERROR' LIMIT 10;
\q
```

### Probar una notificación manualmente

No hay endpoint REST de prueba directo — el servicio solo consume Redis Streams.
Para simular un evento, publicar directamente en Redis:

```bash
# Publicar evento de prueba en el stream
podman exec ticket-system-redis redis-cli XADD ticket.events '*' \
  tipo ticket_created \
  ticket_id "999" \
  titulo "Ticket de prueba" \
  estado "ABIERTO" \
  solicitante_email "tu@email.com" \
  solicitante_nombre "Test User" \
  dominio "IT" \
  categoria "acceso" \
  prioridad "MEDIA"
```

El servicio debe detectar el evento en segundos y logear el intento de envío.

### Configurar proveedor de email por tenant

El servicio resuelve el proveedor en este orden:
1. Config específica del tenant en `notification_provider_configs`
2. Config global (`tenant_id IS NULL`) en `notification_provider_configs`
3. Adaptador por defecto vía env var `NOTIFICATION_WEBHOOK_URL`

**Registrar un tenant con webhook (caso actual — tenant 1):**

```bash
podman exec -it notification-db psql -U notif_admin -d notifications_db -c \
  "INSERT INTO notification_provider_configs (tenant_id, provider, webhook_url, is_active)
   VALUES (1, 'WEBHOOK', 'http://til-n8n:5678/webhook/notification-webhook', true);"
```

**Registrar un tenant con Resend o SendGrid:**

```bash
# Resend
podman exec -it notification-db psql -U notif_admin -d notifications_db -c \
  "INSERT INTO notification_provider_configs (tenant_id, provider, api_key, is_active)
   VALUES (1, 'RESEND', 'tu_api_key_resend', true);"

# SendGrid
podman exec -it notification-db psql -U notif_admin -d notifications_db -c \
  "INSERT INTO notification_provider_configs (tenant_id, provider, api_key, is_active)
   VALUES (1, 'SENDGRID', 'tu_api_key_sendgrid', true);"
```

**Verificar configuración activa:**

```bash
podman exec -it notification-db psql -U notif_admin -d notifications_db \
  -c "SELECT id, tenant_id, provider, webhook_url, is_active FROM notification_provider_configs;"
```

> Cuando el tenant está configurado en BD el log cambia de
> `sin config en BD, usando adaptador por defecto` a
> `config específica encontrada para tenant=X`.

### Cambiar el provider de email (vía env var — Fase 1 sin BD)

```bash
# Editar .env
NOTIFICATION_EMAIL_PROVIDER=resend   # o sendgrid

# Reiniciar el servicio
podman restart notification-service
```

### Cambiar la URL del portal (botón CTA en correos)

```bash
# Opción 1: editar directamente .env
NOTIFICATION_PORTAL_URL=http://192.168.137.32:8090

# Opción 2: actualizar ~/stack_ticket/.env.deploy y relanzar
echo "APP_PUBLIC_URL=http://192.168.137.32:8090" > ~/stack_ticket/.env.deploy
cd ~/stack_ticket/notification-service && ./start.sh
```

### Reiniciar el servicio sin afectar la BD

```bash
podman restart notification-service
```

---

## 4. Gestión de plantillas

Las plantillas viven en:
```
src/main/resources/templates/
```

Para modificar una plantilla:
1. Editar el archivo `.html` en el repo
2. Reconstruir la imagen del contenedor
3. Reiniciar

```bash
cd ~/stack_ticket/notification-service
podman-compose build notification-service
podman-compose up -d notification-service
```

> Las plantillas se compilan en la imagen Docker — no se recargan en caliente.

### Ver plantillas actuales (dentro del contenedor)

```bash
podman exec notification-service find /app -name "*.html" -path "*/templates/*"
```

---

## 5. Troubleshooting

### El servicio no arranca — error de BD

```bash
podman logs notification-service | tail -30

# Causa frecuente: notification-db no está healthy aún
podman exec notification-db pg_isready -U notif_admin -d notifications_db
# Si falla: esperar y reintentar, o revisar logs de notification-db
```

### No llegan notificaciones

**1. Verificar que ticket-management publica eventos:**

```bash
# Listar mensajes en el stream
podman exec ticket-system-redis redis-cli XLEN ticket.events
# Debe ser > 0

# Ver los últimos eventos
podman exec ticket-system-redis redis-cli XREVRANGE ticket.events + - COUNT 5
```

**2. Verificar que el consumer group existe:**

```bash
podman exec ticket-system-redis redis-cli XINFO GROUPS ticket.events
# Debe aparecer: notification-service-group
```

**3. Ver si hay lag en el consumer group:**

```bash
podman exec ticket-system-redis redis-cli XINFO GROUPS ticket.events
# "pel-count" > 0 significa mensajes pendientes de procesar
```

**4. Ver logs del servicio durante el procesamiento:**

```bash
podman logs -f notification-service
# Publicar un evento de prueba y observar la respuesta
```

### El webhook falla (correo no llega)

```bash
podman logs --tail 50 notification-service | grep -i "webhook\|error\|failed"
```

Causas comunes:
- `til-n8n` no está corriendo
- El workflow de n8n no tiene el nodo de webhook configurado
- URL del webhook incorrecta en `NOTIFICATION_WEBHOOK_URL`

```bash
# Verificar que til-n8n responde
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:5679/healthz
# Esperado: 200
```

### Flyway falla en la BD

```bash
podman logs notification-service | grep -i flyway

# Causa: checksum de migración cambiado
# NUNCA modificar V__*.sql ya aplicados
# Si fue accidental:
podman exec -it notification-db psql -U notif_admin -d notifications_db \
  -c "DELETE FROM flyway_schema_history WHERE version='X';"
podman restart notification-service
```

### Error de conexión a Redis (ticket-system-redis)

```bash
# Verificar que ticket-management está corriendo
podman ps | grep ticket-system-redis

# Verificar que ticket-management-network existe
podman network ls | grep ticket-management-network

# El REDIS_HOST debe ser ticket-system-redis (no classifier-redis)
podman exec notification-service env | grep REDIS_HOST
```

---

## 6. Referencia rápida

| Acción | Comando / URL |
|---|---|
| Arrancar stack | `./start.sh` |
| Parar stack | `podman-compose down` |
| Health check | `curl http://localhost:8081/actuator/health` |
| Logs en vivo | `podman logs -f notification-service` |
| Reiniciar servicio | `podman restart notification-service` |
| Abrir BD | `podman exec -it notification-db psql -U notif_admin -d notifications_db` |
| Publicar evento prueba | `podman exec ticket-system-redis redis-cli XADD ticket.events '*' tipo ticket_created ...` |
| Ver consumer lag | `podman exec ticket-system-redis redis-cli XINFO GROUPS ticket.events` |
| Prometheus metrics | http://localhost:8081/actuator/prometheus |

---

*notification-service · Guía Operacional · Mayo 2026*
