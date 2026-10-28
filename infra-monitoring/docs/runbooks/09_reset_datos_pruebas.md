# 09 — Reset de datos de prueba
## infra-monitoring · Borrar tickets, correos y avisos conservando la configuración
**Octubre 2026** — tablas verificadas en Rocky el 2026-10-17

Borra los **datos de operación** (tickets, correos recibidos, avisos enviados) de las tres
bases, los Redis y, opcionalmente, MinIO. **Conserva la configuración**: clientes (tenants),
canales, usuarios, operadores de plataforma, permisos, workflows y estados, clasificaciones
y categorías, proveedores de envío y todo lo de n8n.

Todo se corre en Rocky. El stack sigue arriba durante el reset. **Respaldar antes**
(`03_guia_operacional.md` §11): no hay vuelta atrás.

---

## 1. Qué se borra y qué se conserva

### classifier_db (ticket-classification)

| Tabla | Acción | Contenido |
|---|---|---|
| `inbox_messages` | TRUNCATE CASCADE | Correos recibidos en la bandeja |
| `inbox_attachments` | (cascada) | Adjuntos de esos correos |
| `agent_runs` | (cascada) | Corridas del agente por correo |
| `channel_errors` | TRUNCATE | Errores reportados por los canales |
| `sender_image_fingerprints` | Conservar (opcional borrar) | Huellas aprendidas de imágenes de firma |
| `flyway_schema_history` | **Conservar** | Migraciones aplicadas |

### tickets_db (ticket-management)

| Tabla | Acción | Contenido |
|---|---|---|
| `tickets` | TRUNCATE CASCADE | Tickets |
| `ticket_comments`, `ticket_attachments`, `ticket_status_history`, `ticket_events`, `ticket_classification_review`, `ticket_email_content`, `ticket_channel_replies` | (cascada) | Todo lo que cuelga de un ticket |
| `event_outbox` | TRUNCATE | Eventos pendientes de publicar a notification-service |
| `tenants`, `tenant_channels`, `tenant_deletion_steps` | **Conservar** | Clientes y canales |
| `users`, `user_invitations`, `password_reset_tokens` | **Conservar** | Cuentas |
| `platform_operators`, `platform_audit_log`, `support_sessions` | **Conservar** | Plataforma y su auditoría |
| `role_permissions` | **Conservar** | Permisos por rol |
| `workflows`, `workflow_states`, `workflow_transitions`, `workflow_template_states`, `workflow_template_transitions` | **Conservar** | Estados y transiciones |
| `ticket_classifications`, `ticket_categories` | **Conservar** | Catálogo de clasificación |
| `flyway_schema_history` | **Conservar** | Migraciones aplicadas |

### notifications_db (notification-service)

| Tabla | Acción | Contenido |
|---|---|---|
| `notification_logs` | TRUNCATE | Bitácora de avisos enviados |
| `notification_provider_configs` | **Conservar** | Por dónde se envía (por canal / tenant) |

### Otros

| Recurso | Acción | Nota |
|---|---|---|
| `classifier-redis` | FLUSHALL | Caché LLM y límites por canal |
| `ticket-system-redis` | FLUSHALL | Caché y stream `ticket-events`; obliga a reiniciar notification-service (§4) y cierra las sesiones abiertas del portal |
| MinIO `minio` / bucket `classifier-inbox` | Opcional (§5) | Archivos de los correos; si se quedan, solo ocupan espacio |
| MinIO `ticket-system-minio` / bucket `ticket-attachments` | Opcional (§5) | Archivos de los tickets |
| n8n (`n8n_data`) | **No se toca** | Workflows, credenciales, cursores |

---

## 2. Antes de empezar

1. Respaldo de las tres bases (`03_guia_operacional.md` §11).
2. Que no entren correos durante el reset: comprobar que las copias de ingesta de los
   canales están inactivas (`podman exec til-n8n n8n list:workflow --active=true`); si alguna
   está activa, desactivarla y reactivarla al final.

---

## 3. Bases de datos

Orden: clasificador, después ticket-management, después notificaciones.

```bash
podman exec -i classifier-db psql -U admin -d classifier_db <<'SQL'
TRUNCATE TABLE inbox_messages RESTART IDENTITY CASCADE;
TRUNCATE TABLE channel_errors RESTART IDENTITY;
-- opcional: TRUNCATE TABLE sender_image_fingerprints RESTART IDENTITY;
SQL

podman exec -i ticket-db psql -U ticket_admin -d tickets_db <<'SQL'
TRUNCATE TABLE tickets CASCADE;
TRUNCATE TABLE event_outbox RESTART IDENTITY;
SQL

podman exec -i notification-db psql -U notif_admin -d notifications_db <<'SQL'
TRUNCATE TABLE notification_logs RESTART IDENTITY;
SQL
```

`TRUNCATE tickets CASCADE` vacía también las 7 tablas que tienen llave foránea a `tickets`
y nada más (verificado contra las llaves foráneas de `tickets_db`).

---

## 4. Redis

Ambos Redis piden contraseña; se toma de la variable del propio contenedor:

```bash
podman exec classifier-redis    sh -c 'redis-cli -a "$REDIS_PASSWORD" --no-auth-warning FLUSHALL'
podman exec ticket-system-redis sh -c 'redis-cli -a "$REDIS_PASSWORD" --no-auth-warning FLUSHALL'

# El FLUSHALL borra el stream y el consumer group: notification-service los recrea al arrancar
podman restart notification-service
podman logs --tail 20 notification-service     # sin errores del stream "ticket-events"
```

---

## 5. MinIO (opcional)

Las imágenes de MinIO del stack no traen el cliente `mc` y la imagen `minio/mc` no se puede
descargar desde Rocky. Si se quiere liberar el espacio, borrar el contenido de la carpeta
del bucket y reiniciar MinIO (no probado; los archivos huérfanos no afectan al sistema):

```bash
podman exec minio               sh -c 'rm -rf /data/classifier-inbox/*'       && podman restart minio
podman exec ticket-system-minio sh -c 'rm -rf /minio_data/ticket-attachments/*' && podman restart ticket-system-minio
```

---

## 6. Verificación

```bash
podman exec classifier-db psql -U admin -d classifier_db -At -c \
  "SELECT 'inbox_messages', count(*) FROM inbox_messages UNION ALL
   SELECT 'agent_runs', count(*) FROM agent_runs UNION ALL
   SELECT 'channel_errors', count(*) FROM channel_errors;"
podman exec ticket-db psql -U ticket_admin -d tickets_db -At -c \
  "SELECT 'tickets', count(*) FROM tickets UNION ALL
   SELECT 'ticket_events', count(*) FROM ticket_events UNION ALL
   SELECT 'event_outbox', count(*) FROM event_outbox UNION ALL
   SELECT 'tenants (se conservan)', count(*) FROM tenants UNION ALL
   SELECT 'users (se conservan)', count(*) FROM users;"
podman exec notification-db psql -U notif_admin -d notifications_db -At -c \
  "SELECT 'notification_logs', count(*) FROM notification_logs;"
bash ~/stack_ticket/infra-monitoring/health-check.sh
```

Todo en 0 salvo lo marcado "se conservan"; health-check 10/10.

---

## 7. Después del reset

- **Volver a cargar correos ya leídos**: el cursor de cada canal en n8n sigue adelante, así
  que los correos viejos no vuelven a entrar solos. Para recargarlos, en la consola de
  plataforma → cliente → canal → "Cambiar fecha" (kickoff): la ingesta vuelve a empezar en
  esa fecha y, como la bandeja quedó vacía, los convierte otra vez en tickets.
- Reactivar las copias de ingesta que se hayan desactivado en el §2.

---

*infra-monitoring · Reset de datos de prueba · Octubre 2026*
