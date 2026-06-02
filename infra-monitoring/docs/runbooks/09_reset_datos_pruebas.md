# Reset de datos para pruebas

Procedimiento para limpiar los datos de tickets en `classifier_db`, `tickets_db`,
Redis y MinIO antes de iniciar una nueva ronda de pruebas. Conserva toda la
configuración del sistema (usuarios, workflows, clasificaciones, permisos).

---

## 1. Qué se borra y qué se conserva

### ticket-classification

| Recurso | Componente | Acción | Contenido |
|---|---|---|---|
| `classifier_db` | `tickets` | TRUNCATE | Tickets clasificados |
| `classifier_db` | `agent_runs` | TRUNCATE | Historial de ejecuciones del agente |
| `classifier_db` | `attachments` | TRUNCATE | Adjuntos asociados a tickets |
| `classifier_db` | `enrichments` | TRUNCATE | Enriquecimientos de hilo de correo |
| `classifier-redis` | todas las keys | FLUSHALL | Caché de respuestas LLM |
| `minio` | bucket `email-attachments` | vaciar | Archivos de adjuntos de correo |

### ticket-management

| Recurso | Componente | Acción | Contenido |
|---|---|---|---|
| `tickets_db` | `tickets` | TRUNCATE CASCADE | Tickets del sistema |
| `tickets_db` | `ticket_comments` | (cascada) | Comentarios de tickets |
| `tickets_db` | `ticket_attachments` | (cascada) | Archivos adjuntos |
| `tickets_db` | `ticket_label_mapping` | (cascada) | Etiquetas asignadas |
| `tickets_db` | `ticket_status_history` | (cascada) | Historial de estados |
| `tickets_db` | `ticket_events` | (cascada) | Eventos de auditoría de ticket |
| `tickets_db` | `ticket_classification_review` | (cascada) | Revisiones de clasificación |
| `tickets_db` | `notifications` | (cascada) | Notificaciones del sistema |
| `tickets_db` | `ticket_email_content` | (cascada) | Contenido de correo original |
| `tickets_db` | `audit_logs` | TRUNCATE | Logs de auditoría general |
| `tickets_db` | `ticket_labels` | TRUNCATE | Etiquetas definidas |
| `ticket-system-redis` | todas las keys | FLUSHALL | Caché de sesiones y API |
| `ticket-system-minio` | bucket `ticket-attachments` | vaciar | Archivos adjuntos de tickets |
| `tickets_db` | `tenants` | **CONSERVAR** | Configuración de tenant |
| `tickets_db` | `users` | **CONSERVAR** | Usuarios y cuentas de sistema |
| `tickets_db` | `workflows` | **CONSERVAR** | Definición del flujo |
| `tickets_db` | `workflow_states` | **CONSERVAR** | Estados OPEN/IN_PROGRESS/etc. |
| `tickets_db` | `workflow_transitions` | **CONSERVAR** | Transiciones permitidas |
| `tickets_db` | `ticket_classifications` | **CONSERVAR** | Dominios IT/cliente/operaciones |
| `tickets_db` | `ticket_categories` | **CONSERVAR** | Categorías por dominio |
| `tickets_db` | `workflow_assignments` | **CONSERVAR** | Asignación de flujo por dominio |
| `tickets_db` | `role_permissions` | **CONSERVAR** | Permisos por rol |

---

## 2. Reset ticket-classification

### 2a. Base de datos

```bash
podman exec -it classifier-db psql -U admin -d classifier_db
```

```sql
TRUNCATE TABLE enrichments, attachments, agent_runs, tickets
  RESTART IDENTITY CASCADE;
```

### 2b. Redis (caché LLM)

```bash
podman exec -it classifier-redis redis-cli FLUSHALL
```

### 2c. MinIO (adjuntos de correo)

```bash
podman exec -it minio mc alias set local http://localhost:9000 minioadmin minioadmin123
podman exec -it minio mc rm --recursive --force local/email-attachments
```

---

## 3. Reset ticket-management

### 3a. Base de datos

```bash
podman exec -it ticket-db psql -U ticket_admin -d tickets_db
```

```sql
-- Limpia tickets y todas las tablas que dependen de ellos en cascada
TRUNCATE TABLE tickets CASCADE;

-- Tablas sin FK a tickets — se limpian por separado
TRUNCATE TABLE audit_logs RESTART IDENTITY;
TRUNCATE TABLE ticket_labels RESTART IDENTITY CASCADE;
```

`tickets.id` es UUID (no SERIAL), por lo que `RESTART IDENTITY` no aplica sobre él,
pero sí reinicia los contadores de `audit_logs` y `ticket_labels`.

### 3b. Redis (caché de sesiones)

```bash
podman exec -it ticket-system-redis redis-cli FLUSHALL
```

### 3c. MinIO (adjuntos de tickets)

```bash
podman exec -it ticket-system-minio mc alias set local http://localhost:9000 minioadmin minioadmin_password_123
podman exec -it ticket-system-minio mc rm --recursive --force local/ticket-attachments
```

### 3d. Reiniciar notification-service

El `FLUSHALL` elimina el stream y el consumer group de Redis. El `notification-service`
los recrea solo al arrancar (`@PostConstruct`), por lo que debe reiniciarse:

```bash
podman restart notification-service
```

Verifica que levantó sin errores:

```bash
podman logs --tail 20 notification-service
# Debe aparecer: [stream] Consumer group creado en stream 'ticket-events'
```

---

## 4. Verificación post-reset

```bash
# classifier_db — todas deben retornar 0
podman exec -it classifier-db psql -U admin -d classifier_db -c \
  "SELECT 'tickets' AS tabla, COUNT(*) FROM tickets
   UNION ALL SELECT 'agent_runs', COUNT(*) FROM agent_runs
   UNION ALL SELECT 'enrichments', COUNT(*) FROM enrichments;"

# tickets_db — todas deben retornar 0
podman exec -it ticket-db psql -U ticket_admin -d tickets_db -c \
  "SELECT 'tickets' AS tabla, COUNT(*) FROM tickets
   UNION ALL SELECT 'audit_logs', COUNT(*) FROM audit_logs
   UNION ALL SELECT 'notifications', COUNT(*) FROM notifications;"

# Redis classifier — debe retornar 0
podman exec -it classifier-redis redis-cli DBSIZE

# Redis ticket-management — debe retornar 0
podman exec -it ticket-system-redis redis-cli DBSIZE

# MinIO classifier — debe listar sin objetos
podman exec -it minio mc ls local/email-attachments

# MinIO ticket-management — debe listar sin objetos
podman exec -it ticket-system-minio mc ls local/ticket-attachments
```

---

## 5. Notas

- El stack **no necesita reiniciarse** — todos los contenedores siguen corriendo durante el reset.
- Orden recomendado: classifier primero, ticket-management después, para mantener
  consistencia si hay un correo en tránsito entre ambos.
- El volumen `n8n_data` (ticket-ingestion-light) **no se toca** — las credenciales
  OAuth de Gmail y el workflow viven ahí.
