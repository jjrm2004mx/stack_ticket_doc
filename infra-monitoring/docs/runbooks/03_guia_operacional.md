# 03 — Guía Operacional
## infra-monitoring · Arranque, salud, despliegue, respaldo, observabilidad y troubleshooting
**Octubre 2026**

Entorno único: Rocky Linux (TIJDCAGENTIA, `ssh Agente@10.187.21.30`), Podman rootless.
Todos los servicios escuchan solo en `127.0.0.1` salvo el portal (8090).
Repos en `~/stack_ticket/` (ticket-management, ticket-classification, notification-service,
ticket-ingestion-light, infra-monitoring) y `~/stack_agent/` (ticket-agent, support-console);
todos en la rama `develop`. Desde Windows las URLs `localhost:<puerto>` se usan con el
túnel SSH (`10_redes_y_acceso_lan.md` §7). Puertos, redes y bases: `11_puertos_y_redes.md`.

---

## Índice

1. [Arranque del ecosistema completo](#1-arranque-del-ecosistema-completo)
2. [Arranque solo el stack de monitoreo](#2-arranque-solo-el-stack-de-monitoreo)
3. [Comandos del orquestador](#3-comandos-del-orquestador)
4. [Salud del ecosistema](#4-salud-del-ecosistema)
5. [Grafana — uso básico](#5-grafana--uso-básico)
6. [Prometheus — consultas básicas](#6-prometheus--consultas-básicas)
7. [Loki — buscar logs](#7-loki--buscar-logs)
8. [Sincronizar repos](#8-sincronizar-repos)
9. [Troubleshooting](#9-troubleshooting)
10. [Desplegar una versión nueva en Rocky](#10-desplegar-una-versión-nueva-en-rocky)
11. [Respaldo y restauración](#11-respaldo-y-restauración)
12. [Referencia rápida](#12-referencia-rápida)

---

## 1. Arranque del ecosistema completo

Tras reiniciar Rocky, en este orden:

```bash
# 1. Stacks de ~/stack_ticket (orden automático, ver abajo)
cd ~/stack_ticket/infra-monitoring
make up            # equivale a: bash startup.sh up

# 2. Asistente y consola de soporte (~/stack_agent), después del paso 1
bash ~/stack_agent/support-console/startup.sh up-all

# 3. Verificar
bash ~/stack_ticket/infra-monitoring/health-check.sh
```

`startup.sh up` crea las redes compartidas si faltan y levanta, esperando los health checks:

1. **ticket-management** — `ticket-db`, `ticket-system-redis`, `ticket-system-minio`, flyway, backend, frontend
2. **ticket-classification** — `classifier-db`, `classifier-redis`, `minio`, flyway, `langchain-api`, `langchain-agent`, `ollama`
3. **notification-service** — necesita `ticket-system-redis`
4. **ticket-ingestion-light** — `til-n8n`
5. **infra-monitoring** — Prometheus, Grafana, Loki, promtail, node-exporter

`up-all` de support-console levanta `ticket-agent`, los servidores MCP (`loki-mcp-server`,
`prometheus-mcp-server`) y `sc-nginx`. El primer arranque de `ticket-agent` puede tardar
hasta 3 min (descarga del modelo ONNX).

Estado: `make status` (o `bash startup.sh status`).

---

## 2. Arranque solo el stack de monitoreo

```bash
cd ~/stack_ticket/infra-monitoring
./start.sh          # o: podman-compose up -d
```

---

## 3. Comandos del orquestador

| Comando Make | Equivalente | Acción |
|---|---|---|
| `make up` | `bash startup.sh up` | Levanta los 5 stacks de `~/stack_ticket` |
| `make down` | `bash startup.sh down` | Los para |
| `make restart` | `bash startup.sh restart` | Reinicia |
| `make status` | `bash startup.sh status` | Estado de los contenedores |
| `make build` | `bash startup.sh build` | Rebuild de imágenes locales + up |
| `make deploy` | `bash startup.sh deploy` | sync-repos + build + up |
| `make sync` | `bash sync-repos.sh` | `git pull` de los 5 repos de `~/stack_ticket` |

`~/stack_agent`: `bash ~/stack_agent/support-console/startup.sh up-all | down-all | status`.

Parar un solo stack: `cd ~/stack_ticket/<repo> && podman-compose down` (nunca `down -v`:
borra los volúmenes de datos).

---

## 4. Salud del ecosistema

### health-check.sh

```bash
bash ~/stack_ticket/infra-monitoring/health-check.sh
# Al final: "10 / 10 servicios OK" (código de salida 0) o cuáles fallan
```

(Se corre con `bash`: el archivo no tiene permiso de ejecución.)

| Servicio | Endpoint que revisa | Respuesta esperada |
|---|---|---|
| ticket-management backend | `http://localhost:8080/api/v1/actuator/health` | `"status":"UP"` |
| ticket-management frontend | `http://localhost:8090` | HTTP 200 |
| langchain-agent | `http://localhost:8001/health` | `"status":"ok"` (incluye `"ticket_management":"ok"`) |
| langchain-api | `http://localhost:8000/health` | `"status":"ok"` |
| notification-service | `http://localhost:8081/actuator/health` | `"status":"UP"` |
| n8n | `http://localhost:5679/healthz` | `"status":"ok"` |
| prometheus | `http://localhost:9090/-/healthy` | `Healthy` |
| grafana | `http://localhost:3000/api/health` | `"database"` |
| loki | `http://localhost:3100/ready` | `ready` (reintenta: tarda en arrancar) |
| ollama | `http://localhost:11434/api/tags` | `"models"` |

**No cubre** `~/stack_agent` ni las bases de datos directamente. Revisarlos a mano:

```bash
curl -s http://localhost:8002/api/v1/health          # ticket-agent: {"status":"UP",...}
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3001   # consola de soporte: 200
podman ps --format "{{.Names}} {{.Status}}" | grep -E "mcp-server|healthy|unhealthy"
podman exec ticket-db pg_isready -U ticket_admin -d tickets_db
podman exec classifier-db pg_isready -U admin -d classifier_db
podman exec notification-db pg_isready -U notif_admin -d notifications_db
```

### Diagnóstico extendido

`bash ./diagnostico-stack.sh` revisa directorios, `.env`, contenedores, conectividad entre
servicios, proveedores de IA y redes. `[WARN]` es informativo; `[ERROR]` requiere acción.

---

## 5. Grafana — uso básico

**URL:** http://localhost:3000 · usuario `admin` · password = `GRAFANA_PASSWORD` del `.env`.

- Dashboards: menú → **Dashboards** → elegir el del servicio; rango de tiempo arriba a la derecha.
- Cambiar password: editar `GRAFANA_PASSWORD` en `.env` y `podman restart grafana`.
- Alertas: panel → Edit → **Alert** → New alert rule.

---

## 6. Prometheus — consultas básicas

**URL:** http://localhost:9090 · targets en http://localhost:9090/targets (todos **UP**).
Jobs: `ticket-management`, `notification-service`, `langchain-agent`, `langchain-api`,
`ticket-agent`, `node-exporter`, `prometheus`.

```promql
up                                                     # 1 = UP, 0 = DOWN, por job
rate(http_server_requests_seconds_count{job="ticket-management"}[5m])
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="langchain-agent"}[5m]))
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)   # CPU %
node_memory_MemAvailable_bytes / 1024 / 1024                                      # RAM libre MB
```

---

## 7. Loki — buscar logs

Grafana → **Explore** → datasource **Loki**. promtail lee journald y etiqueta cada línea
con `container` (nombre del contenedor) y `service`.

```logql
{container="langchain-agent"}
{container="ticket-system-backend"} |= "ERROR"
{container="notification-service"} |= "canal"
{container="til-n8n"}
```

Sin Grafana: `podman logs --tail 100 <contenedor>`.

### Promtail no recolecta logs

```bash
sudo chmod o+r /var/log/journal/$(cat /etc/machine-id)/*.journal   # una vez
podman restart promtail
podman logs --tail 20 promtail | grep -i "error\|warn"
```

---

## 8. Sincronizar repos

```bash
make sync                    # git pull de los 5 repos de ~/stack_ticket (no toca ~/stack_agent)
./sync-repos.sh --force      # ⚠️ descarta cambios locales
cd ~/stack_agent/ticket-agent && git pull     # ~/stack_agent, a mano
```

Al final avisa de ramas sin fusionar a `develop`.

---

## 9. Troubleshooting

### Las redes Podman no existen

```bash
podman network ls | grep -E "ticket|observability"
bash ./startup.sh up     # las crea
```

`startup.sh` las crea sin subred fija y nunca borra redes ni contenedores. No crearlas a
mano con `--subnet`: una subred fija puede chocar con las que Podman asigna solo
(`11_puertos_y_redes.md` §7). Si hace falta crear una a mano: `podman network create <nombre>`.

### Prometheus no alcanza un servicio

```bash
podman inspect <contenedor> --format "{{range \$k,\$v := .NetworkSettings.Networks}}{{\$k}} {{end}}"
# Prometheus está en observability, ticket-classification y ticket-management-network
```

### Grafana en blanco / sin datos

```bash
podman exec grafana wget -qO- http://prometheus:9090/-/healthy   # Prometheus Server is Healthy.
podman restart grafana
```

### startup.sh falla en un repo

```bash
cd ~/stack_ticket/<repo> && podman-compose logs --tail 30
./start.sh     # arrancarlo solo para ver el error
```

### Un contenedor no toma la versión nueva

`podman-compose up -d` no recrea un contenedor que ya corre aunque haya imagen nueva: usar
`--force-recreate` (§10).

---

## 10. Desplegar una versión nueva en Rocky

Los cambios llegan a Rocky al fusionarse en `develop`. Por servicio:

```bash
cd ~/stack_ticket/<repo>          # o ~/stack_agent/<repo>
git pull                          # Rocky sigue origin/develop

# Si el cambio trae migraciones de base de datos: respaldar antes (§11)

podman-compose build <servicio>                         # imágenes que se construyen en Rocky
podman-compose up -d --force-recreate --no-deps <servicio>
bash ~/stack_ticket/infra-monitoring/health-check.sh
```

- Sin `--force-recreate` el contenedor sigue con la imagen anterior. Comprobar con
  `podman inspect <contenedor> --format "{{.Image}}"`.
- Migraciones: ticket-management (`podman-compose up flyway`, revisar
  `podman logs ticket-flyway`) y ticket-classification (`classifier-flyway`) las aplican al
  levantar; revisar sus logs.
- Bind mounts bajo SELinux necesitan `:Z`/`:z`; si un contenedor no puede leer su volumen,
  revisar eso primero.
- Procedimientos específicos: runbooks de cada repo (por ejemplo ticket-management
  `operacion.md`, ticket-ingestion-light `03_guia_operacional.md` §6).
- Todo junto: `make deploy` (sync + build + up de `~/stack_ticket`).

---

## 11. Respaldo y restauración

Respaldos en `~/respaldos/` (en Rocky, nunca en git: contienen datos y credenciales).
Referencia estable: `~/respaldos/v0.1.0/` (tag `v0.1.0`).

### Respaldo integral

```bash
F=$(date +%F); D=~/respaldos/$F; mkdir -p $D
podman exec ticket-db       pg_dump -Fc -U ticket_admin tickets_db       > $D/tickets_db.dump
podman exec classifier-db   pg_dump -Fc -U admin        classifier_db    > $D/classifier_db.dump
podman exec notification-db pg_dump -Fc -U notif_admin  notifications_db > $D/notifications_db.dump
# n8n (workflows, credenciales Graph, llaves y cursores): con n8n parado
cd ~/stack_ticket/ticket-ingestion-light && podman-compose down
podman volume export ticket-ingestion-light_n8n_data --output $D/n8n_data.tar
./start.sh
for r in ~/stack_ticket/* ~/stack_agent/*; do echo "$(basename $r) $(git -C $r rev-parse --short HEAD)"; done > $D/commits.txt
```

Antes de una migración basta el dump de esa base, con el nombre de la versión
(por ejemplo `tickets_db_pre_v40_<fecha>.dump`).

No incluido: MinIO (adjuntos) y la base de conocimiento del asistente (Oracle ADB, en OCI;
se regenera re-ingiriendo los documentos).

### Restauración

```bash
# Una base (sobrescribe su contenido)
podman exec -i ticket-db pg_restore -c -U ticket_admin -d tickets_db < tickets_db.dump

# n8n
cd ~/stack_ticket/ticket-ingestion-light && podman-compose down
podman volume import ticket-ingestion-light_n8n_data n8n_data.tar
./start.sh
```

Después: reiniciar el servicio dueño de la base y correr `health-check.sh`.

### Borrar datos de prueba

`09_reset_datos_pruebas.md`.

---

## 12. Referencia rápida

| Acción | Comando |
|---|---|
| Arrancar todo | `make up` y `bash ~/stack_agent/support-console/startup.sh up-all` |
| Parar todo | `make down` y `... startup.sh down-all` |
| Salud | `bash health-check.sh` (+ §4 para stack_agent) |
| Diagnóstico | `bash diagnostico-stack.sh` |
| Sincronizar repos | `make sync` |
| Desplegar un servicio | `git pull` + `podman-compose up -d --force-recreate --no-deps <svc>` |
| Respaldo integral | §11 |
| Grafana / Prometheus | http://localhost:3000 · http://localhost:9090 |
| Logs | Grafana → Explore → Loki `{container="<nombre>"}` o `podman logs --tail 100 <nombre>` |

---

*infra-monitoring · Guía Operacional · Octubre 2026*
