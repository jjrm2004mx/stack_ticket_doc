# 15 — Preguntas sobre los datos

Función contratada por cliente: un usuario con permiso pregunta por los tickets de su empresa y recibe un texto con
la tabla de datos. Fuente de verdad: `openspec/changes/preguntas-sobre-los-datos/` (hasta su archivado) y, en el
motor, `text-to-sql/docs/runbooks/02_arquitectura_tecnica.md`.

## Piezas

| Pieza | Dónde | Qué hace |
|---|---|---|
| Motor `text-to-sql` | otro repo; contenedor `text-to-sql:8003` en `ticket-management-network` | Escribe y valida el SQL, redacta la respuesta. Nunca entra a la base |
| `POST /api/v1/insights/query` | backend, paquete `com.ticketmanagement.insights` | Compuertas, valores del cliente, llamada al motor, ejecución, registro |
| Rol `ticket_insights` | base, V73 + `afterMigrate.sql` | Solo lectura; ve únicamente las ocho vistas de `insights` |
| `insights_questions` | base | Una fila por pregunta (sin filas de resultado ni respuestas) |
| `tenants.ai_data_questions` | base | Interruptor por cliente; **nace apagado** |

## Variables (`.env` del stack de TM)

| Variable | Valor |
|---|---|
| `INSIGHTS_DB_PASSWORD` | contraseña de `ticket_insights` (sin comillas simples) |
| `ENGINE_BASE_URL` | `http://text-to-sql:8003` |
| `ENGINE_SERVICE_KEY` | igual a `APP_KEY_TICKET_MANAGEMENT` del `.env` del motor. Opcional: sin ella el portal arranca y las preguntas dan 503 |
| `ENGINE_SQL_TIMEOUT_MS`, `ENGINE_ANSWER_TIMEOUT_MS`, `INSIGHTS_TOTAL_TIMEOUT_MS` | 30000, 15000, 50000 (el total, menor que los 90 s del nginx). Medido en la compuerta 2: una pregunta completa tarda ≤ 1.4 s. El motor corta antes que TM (`SQL_BUDGET_S=26`, `ANSWER_BUDGET_S=13`, `MODEL_TIMEOUT_S=12`); si se suben aquí, se suben allá |

## Encender o apagar para un cliente

Lo hace el operador, con motivo, en la consola: detalle del cliente › pestaña «IA» › «Funciones de IA» › fila
«Preguntas sobre los datos» (Encendida / Apagada). Debajo del consumo del clasificador está la tabla de consumo de la
función. Sin la consola, por la API con un token de plataforma:

```bash
curl -s -X PUT http://localhost:8090/api/v1/platform/tenants/<id>/ai/functions/data-questions \
  -H "Authorization: Bearer <token de operador>" -H "Content-Type: application/json" \
  -d '{"enabled": true, "reason": "Contratado en <mes>"}'
```

Queda en la auditoría (`TENANT_AI_FUNCTION_CHANGED`) y en el historial de IA del cliente. Encender o apagar no
cambia permisos ni borra el registro. Encendida, el admin la tiene siempre (`ASK_DATA` fijo) y puede dársela a los
managers (que además necesitan `VIEW_ALL_TICKETS`; borradores y archivados piden `REVIEW_DRAFTS`).
El admin del cliente ve la fila en solo lectura en «IA» › «Funciones de IA», y los usuarios con el permiso ven el modo
«Mis datos» en «¿Necesitas ayuda?».

Consumo por mes (seis meses): `GET /api/v1/platform/tenants/<id>/ai/data-questions/usage`.

## Comprobar que funciona

```bash
# El motor responde y tiene el catálogo válido (la imagen del backend trae wget, no curl)
podman exec ticket-system-backend wget -qO- http://text-to-sql:8003/health
# Últimas preguntas de un cliente (como dueño de la base)
podman exec ticket-db psql -U ticket_admin -d tickets_db -c \
  "SELECT asked_at, outcome, row_count, duration_ms, sql_ms, db_ms, answer_ms, model_calls, error
     FROM insights_questions WHERE tenant_id = <id> ORDER BY asked_at DESC LIMIT 10"
```

## Si el motor no responde

El portal muestra «⚠ Las preguntas sobre los datos no están disponibles en este momento. Intenta más tarde.» (503,
sin reintentar solo) y la fila queda con
`ENGINE_UNAVAILABLE`. El resto del portal no se afecta.

| Señal en el log del backend | Causa | Qué hacer |
|---|---|---|
| `Preguntas sobre los datos sin configurar: falta ENGINE_SERVICE_KEY` (al arrancar) | Falta la llave | Ponerla en `.env` y recrear el backend (`--force-recreate`) |
| `El motor de preguntas rechazó la configuración ... (401/403)` | Llave distinta a la del motor | Igualar `ENGINE_SERVICE_KEY` y `APP_KEY_TICKET_MANAGEMENT` |
| `... (409 CATALOG_VERSION_MISMATCH)` | El motor tiene otra versión MAYOR del catálogo | Desplegar juntos el motor y TM de la misma versión |
| `error` = `motor: ResourceAccessException` | Contenedor caído o fuera de la red | `podman ps`, `podman logs text-to-sql`; levantarlo con el orquestador |
| fila `RATE_LIMITED`, portal 429 | Cupo del motor por minuto | Esperar; subir `RATE_LIMIT_USAGE_PER_MINUTE` en el motor si es habitual |
| fila `ERROR` con `sin conexión del rol de solo lectura` | Contraseña de `ticket_insights` o pool agotado (3) | Ver la rotación de abajo |
| fila `ERROR` con `SECURITY:` | El cliente de la transacción cambió durante la consulta: no se devolvió nada | Incidente de seguridad: apagar la función para todos y revisar la base |

Nada se reintenta solo tras un tiempo agotado (ni el backend ni el portal).

## Rotar la contraseña del rol

1. Cambiar `INSIGHTS_DB_PASSWORD` en `.env` (sin comillas simples).
2. Migrar (el `afterMigrate.sql` fija la contraseña nueva) y recrear el backend:
   `podman-compose up -d --force-recreate flyway backend`.
3. Hacer una pregunta en un cliente con la función encendida y revisar que su fila no sea `ERROR`.

Una base restaurada de un respaldo queda cerrada otra vez con el siguiente migrate (`TEMPORARY` y los ajustes del rol
no viajan con `pg_dump`; runbook 03).

## Quitar la función

1. Apagarla para todos los clientes (no rompe nada; los permisos y el registro se conservan).
2. Para quitarla del todo: revertir el código y aplicar una migración que borre el esquema `insights`, los roles
   `ticket_insights` y `ticket_insights_owner`, la columna `tenants.ai_data_questions`, la tabla
   `insights_questions` y las filas `ASK_DATA`, y devuelva a `PUBLIC` el `EXECUTE` de `set_config` y de
   `lo_creat`/`lo_create`/`lo_from_bytea`, el `USAGE` de `public` y `TEMPORARY` (quitando sus líneas de
   `afterMigrate.sql`). Detener el contenedor `text-to-sql`.
