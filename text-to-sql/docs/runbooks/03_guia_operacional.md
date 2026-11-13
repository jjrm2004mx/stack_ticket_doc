# 03 — Guía operacional

Servidor: `~/stack_ticket/text-to-sql`. Contenedor: `text-to-sql`. Puerto: `127.0.0.1:8003`.
En operación normal lo arranca y detiene `infra-monitoring/startup.sh` (sexto proyecto, opcional).

## Arrancar y detener

```bash
cd ~/stack_ticket/text-to-sql
bash start.sh                      # crea .env la primera vez, crea la red si falta, down + up -d
podman-compose down                # detener
podman-compose up -d --build       # reconstruir tras un cambio de código o de catálogo
curl -s http://localhost:8003/health
journalctl -t text-to-sql -f       # logs (JSON, sin preguntas ni filas)
```

Antes de construir en Rocky, revisar el disco (`df -h /home`): las imágenes viejas ya lo llenaron una
vez. `podman image prune` limpia las colgadas; **no** borrar volúmenes de otros proyectos.

## Primera vez

1. `git clone` en `~/stack_ticket/text-to-sql` y `cp .env.example .env`.
2. En `.env`: `PROVIDER=oci`, `OCI_COMPARTMENT_ID`, `OCI_SERVICE_ENDPOINT` (los mismos que usa
   `ticket-agent`), y `APP_KEY_TICKET_MANAGEMENT` = `openssl rand -hex 32`.
3. La misma llave va en `ticket-management` como `ENGINE_SERVICE_KEY`, y `ENGINE_BASE_URL=http://text-to-sql:8003`.
4. `bash start.sh` y `curl -s http://localhost:8003/health`: debe listar `ticket-management` con
   `"valid": true`.

## Comprobar sin gastar en el modelo

```bash
KEY=$(grep ^APP_KEY_TICKET_MANAGEMENT .env | cut -d= -f2)
curl -s -X POST http://localhost:8003/v1/validate -H "X-Service-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"application":"ticket-management","sql":"SELECT count(*) FROM tickets WHERE is_open","paramStyle":"qmark"}'
curl -s http://localhost:8003/v1/catalogs/ticket-management -H "X-Service-Key: $KEY" | head -c 300
```

`/v1/validate` y `/v1/catalogs` no llaman al modelo ni cuentan para el cupo.

## Rotar una llave

1. Generar la nueva: `openssl rand -hex 32`.
2. Cambiar `APP_KEY_TICKET_MANAGEMENT` en `.env` del motor y `ENGINE_SERVICE_KEY` en TM.
3. Reiniciar los dos (`podman-compose up -d --force-recreate` en cada uno). La llave vieja recibe `401`.
   Entre los dos reinicios las preguntas responden `503` en el portal; el resto de TM sigue igual.

## Cambiar el proveedor o el modelo

- `PROVIDER=oci` u `ollama` en `.env`; para Ollama, `OLLAMA_BASE_URL` y `OLLAMA_LLM_MODEL`, y el
  contenedor debe alcanzar a Ollama (hoy vive en `ticket-classification-network`).
- `OCI_MODEL` cambia el modelo de OCI; `OCI_REASONING_EFFORT=LOW` evita que los modelos de
  razonamiento agoten `OCI_MAX_TOKENS` (lección de `ticket-agent`).
- Un `PROVIDER` desconocido o sin sus variables impide arrancar; el log lo dice.
- `/health` muestra el proveedor y el modelo vigentes; cada respuesta los informa.

## Agregar o cambiar un catálogo

1. Carpeta `catalogs/<aplicación>/` con `catalog.yaml`, `rules.md`, `examples.yaml` (ver `docs/catalogo.md`).
2. `.venv\Scripts\python -m src.catalog.contract` genera `contract.json`.
3. `py -m pytest` (los ejemplos deben validar).
4. Llave `APP_KEY_<APLICACIÓN>` en `.env`, reconstruir y reiniciar.
5. Si cambió la versión MAYOR, la aplicación actualiza sus vistas y su `catalogVersion` en el mismo
   despliegue; mientras tanto recibe `409`.

Si un catálogo no carga, el log de arranque dice por qué (`catalog refused`, campo `reason`) y esa
aplicación recibe `503 CATALOG_INVALID`; las demás siguen.

## Qué hacer si el modelo no responde

Síntomas: `/v1/sql` o `/v1/answer` con `503 MODEL_UNAVAILABLE`; en el portal, "La función de preguntas
no está disponible en este momento" (o la tabla sin texto, si falló solo la redacción).

1. `curl -s http://localhost:8003/metrics | grep tts_model_calls_total` — ver `outcome="unavailable"`.
2. `journalctl -t text-to-sql --since "10 min ago" | grep MODEL_UNAVAILABLE`.
3. Revisar OCI con el mismo diagnóstico de `ticket-agent` (credenciales en `~/.oci`, compartimento,
   cuota). Si `ticket-agent` tampoco responde, el problema es de OCI y no del motor.
4. Tiempos: `tts_model_call_seconds` muestra si las llamadas rozan `MODEL_TIMEOUT_S` (20 s). Subirlo
   exige que siga cabiendo en `SQL_BUDGET_S` y en lo que espera TM (45 s / 20 s).

El motor no reintenta una llamada fallida: así una caída de OCI no duplica el gasto.

## Otros códigos

| Código | Qué revisar |
|---|---|
| `401` | La llave de TM no coincide con la del motor |
| `403` | La llave es de otra aplicación |
| `409` | `catalogVersion` de TM y el catálogo del motor tienen MAYOR distinto |
| `429` | Cupo por minuto: `tts_rate_limited_total{scope}` dice si es de la aplicación o de un cliente |
| `503 CATALOG_INVALID` | El catálogo no cargó: log de arranque |

## Quitar el motor

`podman-compose down` en su carpeta y borrar la carpeta. No deja volúmenes ni datos. En TM la función
queda respondiendo `503`; se apaga por cliente con el interruptor del operador.
