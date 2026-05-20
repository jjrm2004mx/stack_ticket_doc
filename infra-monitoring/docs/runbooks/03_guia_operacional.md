# 03 — Guía Operacional
## infra-monitoring · Arranque, observabilidad y troubleshooting
**Mayo 2026**

---

## Índice

1. [Arranque del ecosistema completo](#1-arranque-del-ecosistema-completo)
2. [Arranque solo el stack de monitoreo](#2-arranque-solo-el-stack-de-monitoreo)
3. [Comandos del orquestador](#3-comandos-del-orquestador)
4. [Diagnóstico del ecosistema](#4-diagnóstico-del-ecosistema)
5. [Grafana — uso básico](#5-grafana--uso-básico)
6. [Prometheus — consultas básicas](#6-prometheus--consultas-básicas)
7. [Loki — buscar logs](#7-loki--buscar-logs)
8. [Sincronizar repos](#8-sincronizar-repos)
9. [Troubleshooting](#9-troubleshooting)
10. [Referencia rápida](#10-referencia-rápida)

---

## 1. Arranque del ecosistema completo

```bash
cd ~/stack_ticket/infra-monitoring

# Opción A — Make (recomendado)
make up

# Opción B — Script directo
./startup.sh up
```

El proceso tarda 2-5 minutos según los health checks de cada servicio.
La salida muestra el estado de cada repo a medida que arranca.

### Ver estado después del arranque

```bash
make status
# o
./startup.sh status
```

---

## 2. Arranque solo el stack de monitoreo

Si los otros repos ya están levantados y solo necesitas Prometheus + Grafana + Loki:

```bash
cd ~/stack_ticket/infra-monitoring
./start.sh
```

O directamente con podman-compose:
```bash
podman-compose up -d
```

---

## 3. Comandos del orquestador

| Comando Make | Equivalente | Acción |
|---|---|---|
| `make up` | `./startup.sh up` | Levanta todo el ecosistema |
| `make down` | `./startup.sh down` | Para todo el ecosistema |
| `make restart` | `./startup.sh restart` | Reinicia todo |
| `make status` | `./startup.sh status` | Estado de todos los contenedores |
| `make build` | `./startup.sh build` | Rebuild de imágenes locales + up |
| `make deploy` | `./startup.sh deploy` | sync-repos + build + up |
| `make sync` | `./sync-repos.sh` | Sincroniza repos desde GitHub |

### Parar solo el monitoreo (sin afectar los demás)

```bash
cd ~/stack_ticket/infra-monitoring
podman-compose down
```

### Parar un repo específico

```bash
cd ~/stack_ticket/ticket-classification
podman-compose down

# Levantarlo de nuevo
./start.sh
```

---

## 4. Diagnóstico del ecosistema

```bash
cd ~/stack_ticket/infra-monitoring
./diagnostico-stack.sh
```

El script verifica:
- Directorios de los 5 repos
- Archivos `.env` en cada repo
- Estado de contenedores
- Conectividad entre servicios
- Estado de providers de IA (en ticket-classification)
- Redes Podman

### Interpretar la salida

```
[OK]    langchain-agent responde             # Servicio activo
[WARN]  .env contiene localhost en WSL       # Advertencia no bloqueante
[ERROR] ticket-system-backend NO RESPONDE    # Servicio caído o no levantado
```

Los `[WARN]` son informativos. Los `[ERROR]` requieren acción.

---

## 5. Grafana — uso básico

**URL:** http://localhost:3000
**Usuario:** admin
**Password:** valor de `GRAFANA_PASSWORD` en `.env` (default: `admin`)

### Cambiar la password de Grafana

```bash
# Editar .env
GRAFANA_PASSWORD=nueva_password_fuerte

# Reiniciar Grafana
podman restart grafana
```

### Ver métricas de un servicio

1. Menu izquierdo → **Dashboards**
2. Seleccionar el dashboard del servicio (ej: "Ticket Classification")
3. Usar los filtros de tiempo (esquina superior derecha)

### Crear un alerta

1. Abrir un panel → Edit
2. Tab **Alert** → **New alert rule**
3. Definir condición (ej: error rate > 5%)
4. Configurar canal de notificación

---

## 6. Prometheus — consultas básicas

**URL:** http://localhost:9090

### Verificar targets activos

```
http://localhost:9090/targets
```
Todos los targets deben aparecer en estado **UP**.

### Consultas PromQL útiles

```promql
# Tasa de requests HTTP al backend de ticket-management (últimos 5 min)
rate(http_server_requests_seconds_count{job="ticket-management"}[5m])

# Latencia P99 del langchain-agent
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket{job="langchain-agent"}[5m]))

# Notificaciones enviadas por tipo
notifications_sent_total

# Uso de CPU del host
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# RAM disponible en MB
node_memory_MemAvailable_bytes / 1024 / 1024
```

### Verificar que un servicio está siendo scrapeado

```promql
up{job="langchain-agent"}
# Devuelve 1 si está UP, 0 si DOWN
```

---

## 7. Loki — buscar logs

En Grafana: **Explore** → Datasource: **Loki**

### Consultas LogQL básicas

```logql
# Logs de langchain-agent
{container_name="langchain-agent"}

# Logs del backend con errores
{container_name="ticket-system-backend"} |= "ERROR"

# Logs de clasificación completada
{container_name="langchain-agent"} |= "completado"

# Logs de notification-service en los últimos 30 minutos
{container_name="notification-service"} | json
```

### Resolver "Promtail no recolecta logs"

```bash
# En WSL — dar permisos a journald (ejecutar una vez)
sudo chmod o+r /var/log/journal/$(cat /etc/machine-id)/*.journal

# Reiniciar Promtail
podman restart promtail

# Verificar que Promtail puede leer
podman logs --tail 20 promtail | grep -i "error\|warn"
```

---

## 8. Sincronizar repos

```bash
# Actualizar todos los repos desde GitHub (rama actual)
make sync
# o
./sync-repos.sh

# Forzar actualización descartando cambios locales (⚠️)
./sync-repos.sh --force
```

La salida muestra para cada repo: `updated` / `skipped` / `failed`.

### Sincronizar un repo específico

```bash
cd ~/stack_ticket/ticket-classification
git pull
```

---

## 9. Troubleshooting

### Las redes Podman no existen

```bash
# Verificar redes
podman network ls | grep -E "ticket|observability"

# Crear manualmente si faltan
podman network create --subnet 10.89.1.0/24 ticket-management-network
podman network create --subnet 10.89.2.0/24 ticket-classification-network
podman network create --subnet 10.89.3.0/24 observability-network

# O usar startup.sh que las crea automáticamente
./startup.sh up
```

### Conflicto de subred

```bash
# Ver todas las redes y sus subnets
podman network ls --format "{{.Name}}" | xargs -I{} podman network inspect {} \
  --format "{{.Name}}: {{range .Subnets}}{{.Subnet}}{{end}}"

# Si hay conflicto — eliminar la red conflictiva y recrear
podman network rm nombre-conflictivo
podman network create --subnet 10.89.X.0/24 nombre-red
```

### Prometheus no alcanza un servicio

```bash
# Verificar que el servicio está en la red correcta
podman inspect langchain-agent --format "{{json .NetworkSettings.Networks}}" \
  | python3 -m json.tool | grep -i "ticket-classification-network"

# Si no está en la red:
podman network connect ticket-classification-network langchain-agent
```

### Grafana en blanco / sin datos

```bash
# 1. Verificar que Prometheus tiene datos
# http://localhost:9090/targets — verificar que targets están UP

# 2. Verificar que Grafana puede alcanzar Prometheus
podman exec grafana wget -qO- http://prometheus:9090/-/healthy
# Respuesta esperada: Prometheus Server is Healthy.

# 3. Reiniciar Grafana
podman restart grafana
```

### El startup.sh falla en un repo

```bash
# Ver los logs del repo que falló
cd ~/stack_ticket/ticket-classification
podman-compose logs --tail 30

# Intentar arrancar ese repo manualmente para ver el error
./start.sh
```

---

## 10. Referencia rápida

| Acción | Comando |
|---|---|
| Arrancar todo | `make up` |
| Parar todo | `make down` |
| Estado del ecosistema | `make status` |
| Diagnóstico completo | `./diagnostico-stack.sh` |
| Sincronizar repos | `make sync` |
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| Prometheus targets | http://localhost:9090/targets |
| Logs centralizados | Grafana → Explore → Loki |
| Logs de Prometheus | `podman logs --tail 50 prometheus` |
| Logs de Grafana | `podman logs --tail 50 grafana` |
| Reiniciar Grafana | `podman restart grafana` |
| Reiniciar Prometheus | `podman restart prometheus` |

---

*infra-monitoring · Guía Operacional · Mayo 2026*
