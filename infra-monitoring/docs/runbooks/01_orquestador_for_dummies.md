# 01 — Orquestador for Dummies
## infra-monitoring · Guía conceptual del stack
**Stack: Prometheus · Grafana · Loki · Promtail · Node Exporter · startup.sh**
**Mayo 2026**

---

## La analogía: la sala de control del ecosistema

Imagina un edificio con 5 plantas (los repos del ecosistema). Este repo
es la sala de control en la planta baja: tiene los paneles que muestran
qué está pasando en cada planta, los interruptores para encender todo
en el orden correcto, y el registro de todo lo que ha pasado.

| Pieza de la sala de control | Pieza del ecosistema |
|---|---|
| El gran interruptor principal | `startup.sh` — arranca los 5 repos en orden |
| Los paneles de métricas | Grafana — dashboards del stack |
| El recolector de datos | Prometheus — scrapeó métricas de cada servicio |
| El archivo de logs | Loki — centraliza los logs de todos los contenedores |
| El recolector de logs | Promtail — lee journald y envía a Loki |
| El sensor de la sala de máquinas | Node Exporter — métricas del host (CPU, RAM, disco) |
| El manual de arranque | Makefile — comandos abreviados para el operador |

---

## Dos responsabilidades distintas

Este repo tiene **dos roles** que es importante no confundir:

### Rol 1 — Orquestador del ecosistema

`startup.sh` es el script maestro que arranca los 5 repos en el orden correcto,
crea las redes Podman compartidas y detecta el entorno (WSL vs Linux nativo).

```bash
make up       # Arranca todo el ecosistema
make down     # Para todo el ecosistema
make status   # Estado de todos los contenedores
make restart  # Reinicia todo
```

### Rol 2 — Observabilidad centralizada

Prometheus + Grafana + Loki monitorean **todos** los repos del ecosistema
desde un único punto. Los dashboards muestran métricas de ticket-classification,
ticket-management, notification-service y la infraestructura del host.

---

## El ecosistema completo — arranque en orden

`startup.sh` levanta los repos en este orden por dependencias:

```
1. ticket-classification     ← agente IA (sin dependencias externas)
2. ticket-management         ← crea ticket-management-network
3. notification-service      ← necesita ticket-management-network
4. ticket-ingestion-light    ← necesita ticket-management-network
5. infra-monitoring          ← se levanta al final (monitorea a los demás)
```

---

## Prometheus — el recolector

Prometheus hace **scraping** (solicitudes periódicas) a los endpoints de
métricas de cada servicio cada 15 segundos:

| Servicio | Endpoint de métricas |
|---|---|
| ticket-system-backend | `:8080/api/v1/actuator/prometheus` |
| notification-service | `:8081/actuator/prometheus` |
| langchain-agent | `:8001/metrics` |
| langchain-api | `:8000/metrics` |
| node-exporter | `:9100/metrics` (host) |

Las métricas se almacenan en Prometheus durante 15 días.

---

## Grafana — los paneles

Grafana lee de Prometheus y Loki y muestra dashboards visuales:

| Dashboard | Qué muestra |
|---|---|
| Stack Overview | Estado general de todos los servicios |
| ticket-classification | Clasificaciones por minuto, latencia del agente, iteraciones |
| ticket-management | Tickets creados, latencia API, conexiones BD |
| notification-service | Notificaciones enviadas, consumer lag de Redis |
| Host metrics | CPU, RAM, disco del servidor WSL |

URL: `http://localhost:3000` — User: admin / Password: `GRAFANA_PASSWORD` del `.env`

---

## Loki — los logs

Loki almacena los logs centralizados de todos los contenedores.
Promtail los recolecta desde journald (el sistema de logs de Linux/WSL).

En Grafana: **Explore** → Fuente: **Loki** → buscar por etiquetas:
```
{container_name="langchain-agent"}
{container_name="ticket-system-backend"}
{container_name="notification-service"}
```

---

## Redes creadas por startup.sh

startup.sh crea tres redes Podman antes de levantar los stacks:

| Red | Subred | Propósito |
|---|---|---|
| `ticket-management-network` | 10.89.1.0/24 | Red compartida del ecosistema |
| `ticket-classification-network` | 10.89.2.0/24 | Red del stack de clasificación |
| `observability-network` | 10.89.3.0/24 | Red del stack de monitoreo |

---

## Detección de entorno

`startup.sh` detecta automáticamente si está corriendo en WSL o Linux nativo:

| Entorno | Detección | APP_PUBLIC_URL |
|---|---|---|
| WSL2 | `/proc/version` contiene "microsoft" | IP del host Windows (ej: 192.168.137.1:8090) |
| Linux nativo | No contiene "microsoft" | http://localhost:8090 |

La URL detectada se escribe en `~/stack_ticket/.env.deploy` y los repos la
leen al arrancar para configurar sus URLs públicas.

---

## Herramientas incluidas

| Herramienta | Uso |
|---|---|
| `startup.sh` | Orquestador maestro del ecosistema |
| `start.sh` | Levanta solo el stack de infra-monitoring |
| `diagnostico-stack.sh` | Diagnóstico completo de salud del ecosistema |
| `sync-repos.sh` | Sincroniza los 5 repos desde GitHub |
| `detect-environment.sh` | Detecta WSL vs Linux nativo |
| `Makefile` | Atajos para los comandos más frecuentes |

---

*infra-monitoring · Orquestador for Dummies · Mayo 2026*
