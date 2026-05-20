# 02 — Arquitectura Técnica
## infra-monitoring · Referencia de servicios, redes y configuración
**Mayo 2026**

---

## Índice

1. [Servicios del stack](#1-servicios-del-stack)
2. [Redes](#2-redes)
3. [Variables de entorno](#3-variables-de-entorno)
4. [Prometheus — configuración de scraping](#4-prometheus--configuración-de-scraping)
5. [Promtail — recolección de logs](#5-promtail--recolección-de-logs)
6. [Grafana — provisioning](#6-grafana--provisioning)
7. [startup.sh — flujo de orquestación](#7-startupsh--flujo-de-orquestación)
8. [sync-repos.sh](#8-sync-repossh)

---

## 1. Servicios del stack

| Contenedor | Imagen | Puerto host | Rol |
|---|---|---|---|
| `prometheus` | prom/prometheus:v2.51.2 | 9090 | Recolección y almacenamiento de métricas |
| `grafana` | grafana/grafana:10.4.3 | 3000 | Dashboards y visualización |
| `loki` | grafana/loki:2.9.0 | 3100 | Almacenamiento de logs |
| `promtail` | grafana/promtail:2.9.0 | — | Recolección de logs desde journald |
| `node-exporter` | prom/node-exporter:v1.7.0 | 9100 | Métricas del host (CPU, RAM, disco) |

### Versiones fijadas

Todas las imágenes están fijadas a versiones específicas para garantizar reproducibilidad.
Loki y Promtail están fijados en `2.9.0` por compatibilidad de API — no actualizar una sin la otra.

### Retención de datos

- **Prometheus:** 15 días (`--storage.tsdb.retention.time=15d`)
- **Loki:** configurado por defecto (sin límite explícito — monitorear volumen)

---

## 2. Redes

### Redes propias del stack

```yaml
# En docker-compose.yml — todas externas, creadas por startup.sh
networks:
  observability-network:
    external: true    # 10.89.3.0/24
  ticket-management-network:
    external: true    # 10.89.1.0/24
  ticket-classification-network:
    external: true    # 10.89.2.0/24
```

Todos los servicios de este stack están en `observability-network`.
Prometheus también está en `ticket-management-network` y
`ticket-classification-network` para alcanzar los endpoints de métricas
de los otros repos.

### Subnets fijas

| Red | Subred | Por qué fija |
|---|---|---|
| `ticket-management-network` | 10.89.1.0/24 | Evita conflictos con otras redes Podman |
| `ticket-classification-network` | 10.89.2.0/24 | Idem |
| `observability-network` | 10.89.3.0/24 | Idem |

Si Podman asigna subnets automáticas pueden colisionar con la red del host WSL.
Por eso startup.sh las crea con subnets explícitas.

---

## 3. Variables de entorno

### `.env.example`

```bash
GRAFANA_PASSWORD=admin
```

### `.env.wsl` (overrides para WSL2)

```bash
PORTAL_PORT=8090
# WINDOWS_HOST_IP — se auto-detecta en runtime
```

### `.env.produccion` (overrides para Linux nativo)

```bash
TICKET_MGMT_API_URL=http://ticket-system-backend:8080/api/v1
NOTIFICATION_WEBHOOK_URL=http://til-n8n:5678/webhook/notification-webhook
APP_PUBLIC_URL=http://localhost:8090
```

### `~/stack_ticket/.env.deploy` (generado por startup.sh)

```bash
APP_PUBLIC_URL=http://192.168.137.1:8090   # URL pública detectada
```

Este archivo es leído por `notification-service/start.sh` y
`ticket-management/start.sh` para configurar URLs públicas correctas
según el entorno.

---

## 4. Prometheus — configuración de scraping

Archivo: `prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'ticket-management'
    metrics_path: '/api/v1/actuator/prometheus'
    static_configs:
      - targets: ['ticket-system-backend:8080']

  - job_name: 'notification-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['notification-service:8081']

  - job_name: 'langchain-agent'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['langchain-agent:8001']

  - job_name: 'langchain-api'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['langchain-api:8000']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

Prometheus alcanza los servicios por nombre de contenedor a través de las
redes externas. Si un servicio no está corriendo, Prometheus marca ese
target como `DOWN` pero continúa funcionando.

---

## 5. Promtail — recolección de logs

Archivo: `promtail.yml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: journald
    journal:
      max_age: 12h
      labels:
        job: systemd-journal
    relabel_configs:
      - source_labels: [__journal__systemd_unit]
        target_label: unit
      - source_labels: [__journal_container_name]
        target_label: container
      - source_labels: [__journal_container_name]
        target_label: service
```

### Prerequisito WSL2 — permisos de journald

```bash
# Ejecutar una vez en WSL para dar acceso a journald
sudo chmod o+r /var/log/journal/$(cat /etc/machine-id)/*.journal
```

Sin este permiso, Promtail no puede leer los logs del sistema.

---

## 6. Grafana — provisioning

Los datasources y dashboards se configuran automáticamente al arrancar
Grafana desde `grafana-provisioning/`:

```
grafana-provisioning/
├── datasources/
│   └── datasources.yml    # Prometheus y Loki como datasources
└── dashboards/
    ├── dashboards.yml     # Directorio de dashboards
    └── *.json             # Archivos de dashboard importados
```

Grafana lee estos archivos al iniciar — no requiere configuración manual.

---

## 7. startup.sh — flujo de orquestación

`startup.sh` es el orquestador maestro. Subcomandos disponibles:

| Subcomando | Acción |
|---|---|
| `up` | Levanta todo el ecosistema en orden |
| `down` | Para todo el ecosistema |
| `restart` | Para y levanta todo |
| `status` | Estado de todos los contenedores |
| `build` | Reconstruye imágenes locales antes de levantar |
| `deploy` | Actualiza repos + rebuild + up |

### Flujo interno de `startup.sh up`

```
1. detect-environment.sh → detecta WSL vs Linux, IP del host
2. Escribe ~/stack_ticket/.env.deploy con APP_PUBLIC_URL
3. Crea redes Podman (si no existen):
   - podman network create --subnet 10.89.1.0/24 ticket-management-network
   - podman network create --subnet 10.89.2.0/24 ticket-classification-network
   - podman network create --subnet 10.89.3.0/24 observability-network
4. cd ticket-classification  → ./start.sh
5. cd ticket-management      → ./start.sh
6. cd notification-service   → ./start.sh
7. cd ticket-ingestion-light → ./start.sh
8. cd infra-monitoring       → ./start.sh (Prometheus + Grafana + Loki)
```

### Makefile — atajos

```makefile
up:      ./startup.sh up
down:    ./startup.sh down
restart: ./startup.sh restart
status:  ./startup.sh status
build:   ./startup.sh build
deploy:  ./startup.sh deploy
sync:    ./sync-repos.sh
```

---

## 8. sync-repos.sh

Sincroniza los 5 repos desde sus remotos en GitHub:

```bash
./sync-repos.sh           # Sincroniza si no hay cambios locales
./sync-repos.sh --force   # Descarta cambios locales y sincroniza (⚠️)
```

Comportamiento:
1. Por cada repo: `git fetch` + `git pull` en la rama actual
2. Si hay cambios locales sin commitear → falla (safe) a menos que `--force`
3. Configura `core.fileMode=false` para ignorar diferencias de permisos Windows↔WSL
4. Muestra resumen: repos actualizados / omitidos / fallidos

---

*infra-monitoring · Arquitectura Técnica · Mayo 2026*
