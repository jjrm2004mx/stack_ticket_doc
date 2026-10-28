# 10 — Redes y Acceso LAN
## infra-monitoring · Acceso a los servicios de Rocky (LAN y túnel SSH)
**Mayo 2026** · §7 actualizada en octubre de 2026

> Las secciones 1–6 (WSL2, entorno de desarrollo que ya no se usa) se movieron a
> `docs/historico/10_redes_wsl.md`. Este runbook conserva la numeración: lo vigente es la §7.

---

## 7. Rocky (TIJDCAGENTIA) — política de acceso y túnel SSH del administrador

Servidor: `TIJDCAGENTIA` · `10.187.21.30` · usuario `Agente` · firewalld activo.

### Por qué los runbooks dicen `localhost` y desde Windows no funciona

Todos los contenedores publican **solo en `127.0.0.1`** de Rocky, salvo el portal
(`8090`), que publica en todas las interfaces (cambio `aislar-redes-y-puertos`).
Dentro de Rocky `curl http://localhost:PORT` responde. Desde otra máquina,
`localhost` es esa misma máquina, y `http://10.187.21.30:PORT` no responde (el
servicio no escucha en esa interfaz y además firewalld lo bloquea). El túnel SSH
entra por el puerto 22 y reenvía cada puerto local al `localhost` de Rocky; por eso
con el túnel las URLs `http://localhost:PORT` de los runbooks funcionan tal cual.

### Política vigente

Criterio: desde la LAN solo se ofrece lo que tiene login propio y es para usuarios;
todo lo demás escucha en `127.0.0.1` y se usa por túnel.

| Puerto | Servicio | ¿Login propio? | Acceso |
|---|---|---|---|
| 8090 | Portal (ticket-system-frontend) | Sí | **LAN**: `http://10.187.21.30:8090` |
| 22 | SSH | Sí | LAN |
| 9090 | Prometheus | No | Túnel (hasta 2026-10-17 respondía en la LAN) |
| 3000 | Grafana | Sí — por defecto `admin/admin` | Túnel |
| 5679 | n8n | No (contiene credenciales de Microsoft Graph) | Túnel |
| 3001 | support-console | No — el rol se elige en un desplegable | Túnel |
| 8080, 8081, 8000, 8001, 8002 | APIs internas | No | Túnel |
| 9000/9001, 9002/9003 | MinIO API / consola | Consola sí | Túnel |
| 5433, 5434, 5435 | PostgreSQL | Sí (usuario/password de BD) | Túnel — nunca abrir |
| 6380, 6381 | Redis | Contraseña | Túnel — nunca abrir |
| 3100, 9100, 11434 | Loki, node-exporter, Ollama | No | Túnel |
| 9011, 9012 | Servidores MCP | No | Túnel |

Para revisar las reglas de firewalld: `sudo firewall-cmd --list-ports`. La regla
de 9090 que exista ya no tiene efecto (Prometheus no escucha en la LAN); quitarla
es opcional: `sudo firewall-cmd --permanent --remove-port=9090/tcp && sudo firewall-cmd --reload`.

### ¿Cómo entro a Grafana, Prometheus, n8n o la consola desde otra máquina?

Sin la computadora del administrador también se puede, desde cualquier máquina de
la red de la oficina que tenga `ssh` (Windows 10/11, Linux y macOS ya lo traen).
La llave SSH solo está en la máquina del administrador: en otra máquina ssh pide la
**contraseña del usuario `Agente`**.

Túnel con lo más usado (PowerShell, en una sola línea):

```powershell
ssh -N -L 9090:localhost:9090 -L 3000:localhost:3000 -L 5679:localhost:5679 -L 3001:localhost:3001 -L 8002:localhost:8002 Agente@10.187.21.30
```

Mientras esa ventana siga abierta, en el navegador de **esa** máquina:
`http://localhost:9090` (Prometheus), `http://localhost:3000` (Grafana),
`http://localhost:5679` (n8n), `http://localhost:3001` (consola de soporte; usa también
el 8002 del asistente). Al cerrar la ventana (`Ctrl+C`) el acceso se corta.

Túnel completo con **todos** los puertos del ecosistema (una sola línea):

```powershell
ssh -N -L 3000:localhost:3000 -L 3001:localhost:3001 -L 3100:localhost:3100 -L 5433:localhost:5433 -L 5434:localhost:5434 -L 5435:localhost:5435 -L 5679:localhost:5679 -L 6380:localhost:6380 -L 6381:localhost:6381 -L 8000:localhost:8000 -L 8001:localhost:8001 -L 8002:localhost:8002 -L 8080:localhost:8080 -L 8081:localhost:8081 -L 8090:localhost:8090 -L 9000:localhost:9000 -L 9001:localhost:9001 -L 9002:localhost:9002 -L 9003:localhost:9003 -L 9011:localhost:9011 -L 9012:localhost:9012 -L 9090:localhost:9090 -L 9100:localhost:9100 -L 11434:localhost:11434 Agente@10.187.21.30
```

Si un puerto ya está ocupado en esa máquina, se cambia solo el lado local:
`-L 19090:localhost:9090` y se abre `http://localhost:19090`.

Fuera de la red de la oficina (por ejemplo desde casa) no se llega a `10.187.21.30`
(IP privada) ni siquiera al portal: hace falta la VPN de la empresa.

### Túnel del administrador (PowerShell en Windows)

Un solo comando con todos los puertos. Dejar la ventana abierta; `Ctrl+C` lo cierra.

```powershell
ssh -N `
  -L 5679:localhost:5679 `
  -L 9090:localhost:9090 `
  -L 3000:localhost:3000 `
  -L 3001:localhost:3001 `
  -L 8090:localhost:8090 `
  -L 8080:localhost:8080 `
  -L 8001:localhost:8001 `
  -L 8002:localhost:8002 `
  -L 8000:localhost:8000 `
  -L 8081:localhost:8081 `
  -L 9001:localhost:9001 `
  -L 9003:localhost:9003 `
  -L 5433:localhost:5433 `
  -L 5434:localhost:5434 `
  -L 5435:localhost:5435 `
  Agente@10.187.21.30
```

Con el túnel activo, desde el navegador de Windows:

| URL | Servicio |
|---|---|
| http://localhost:5679 | n8n |
| http://localhost:9090 | Prometheus |
| http://localhost:3000 | Grafana |
| http://localhost:3001 | support-console |
| http://localhost:8090 | Frontend de tickets |
| http://localhost:8080/api/v1/actuator/health | ticket-system-backend |
| http://localhost:8001/health | langchain-agent |
| http://localhost:8002/api/v1/health | ticket-agent |
| http://localhost:8000/health | langchain-api (gateway LLM) |
| http://localhost:8081/actuator/health | notification-service |
| http://localhost:9001 | MinIO consola — adjuntos de correos (ticket-classification) |
| http://localhost:9003 | MinIO consola — adjuntos de tickets (ticket-management) |

Bases de datos (DBeaver / pgAdmin desde Windows, host `localhost`):

| Puerto | Contenedor | Base de datos |
|---|---|---|
| 5433 | `ticket-db` | `tickets_db` |
| 5434 | `classifier-db` | `classifier_db` |
| 5435 | `notification-db` | `notifications_db` |

Este túnel diario deja fuera los que casi no se usan desde Windows (3100 Loki,
9100 node-exporter, 11434 Ollama, 9000/9002 MinIO API S3, 6380/6381 Redis,
9011/9012 MCP); el túnel completo de la sección anterior los incluye todos.
Lista de puertos: [11_puertos_y_redes.md](11_puertos_y_redes.md).

### Alternativa: dejarlo fijo en `~/.ssh/config`

En Windows, `C:\Users\<usuario>\.ssh\config`:

```
# Túnel del administrador a Rocky (TIJDCAGENTIA)
# Ver infra-monitoring/docs/runbooks/10_redes_y_acceso_lan.md §7
# Uso: ssh -N rocky-tunel
Host rocky-tunel
    HostName 10.187.21.30
    User Agente
    LocalForward 5679 localhost:5679
    LocalForward 9090 localhost:9090
    LocalForward 3000 localhost:3000
    LocalForward 3001 localhost:3001
    LocalForward 8090 localhost:8090
    LocalForward 8080 localhost:8080
    LocalForward 8001 localhost:8001
    LocalForward 8002 localhost:8002
    LocalForward 8000 localhost:8000
    LocalForward 8081 localhost:8081
    LocalForward 9001 localhost:9001
    LocalForward 9003 localhost:9003
    LocalForward 5433 localhost:5433
    LocalForward 5434 localhost:5434
    LocalForward 5435 localhost:5435
    ExitOnForwardFailure yes
    ServerAliveInterval 60
```

Y se abre con:

```powershell
ssh -N rocky-tunel
```

`ExitOnForwardFailure yes` hace que el túnel falle de inmediato si algún
puerto local ya está ocupado, en vez de arrancar a medias.

> ssh pide la contraseña de `Agente` aunque se use `-N`. Los puertos no se
> abren hasta autenticarse. Una vez aceptada, la ventana queda "colgada" sin
> mostrar nada: ese es el estado correcto.

### Verificar que el túnel está activo

En otra ventana de PowerShell:

```powershell
Test-NetConnection localhost -Port 5679
# TcpTestSucceeded : True  → el túnel escucha en ese puerto
```

Si falla, lanzar el túnel en modo verbose para ver en qué paso se queda:

```powershell
ssh -v -N rocky-tunel
# Esperado:
#   Authenticated to 10.187.21.30 ...
#   Local forwarding listening on 127.0.0.1 port 5679.
#   (una línea por cada puerto)
```

### Troubleshooting del túnel

| Síntoma | Causa | Solución |
|---|---|---|
| **Todas** las URLs `localhost` dan "conexión rechazada", pero `http://10.187.21.30:8090` sí carga | El túnel no está escuchando en Windows: sigue esperando la contraseña, falló al arrancar o se cerró | Revisar la ventana del túnel; `Test-NetConnection localhost -Port 5679`; relanzar con `ssh -v -N rocky-tunel` |
| `bind [127.0.0.1]:PORT: Address already in use` | Ese puerto ya lo usa algo en Windows (otro túnel abierto, WSL, un servicio local) | Cerrar el otro proceso, o mapear a otro puerto local: `-L 15679:localhost:5679` y abrir `http://localhost:15679` |
| `channel N: open failed: connect failed: Connection refused` al abrir la URL | El túnel está bien pero el contenedor no está arriba en Rocky | En Rocky: `podman ps` y `curl -I http://localhost:PORT` |
| La ventana del túnel se cierra sola tras un rato | Timeout de inactividad | Agregar `-o ServerAliveInterval=60` (ya incluido en `~/.ssh/config`) |

### Cuando se abra a usuarios (pendiente)

Abrir un puerto para todos:

```bash
sudo firewall-cmd --permanent --add-port=8090/tcp
sudo firewall-cmd --reload
```

Abrir un puerto solo para una subred o IPs concretas (recomendado para
servicios sin login, como support-console):

```bash
sudo firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="<SUBRED>/24" port port="3001" protocol="tcp" accept'
sudo firewall-cmd --reload
sudo firewall-cmd --list-rich-rules
```

Antes de abrir Grafana (3000), cambiar la contraseña de `admin`.
`GRAFANA_PASSWORD` solo se aplica en el primer arranque (cuando el volumen
de Grafana está vacío); con el volumen ya creado, cambiarla así:

```bash
podman exec -it grafana grafana cli admin reset-admin-password '<NUEVA_PASSWORD>'
```

y actualizar `GRAFANA_PASSWORD` en el `.env` para que quede consistente.

---

*infra-monitoring · Redes y Acceso LAN · Mayo 2026*
