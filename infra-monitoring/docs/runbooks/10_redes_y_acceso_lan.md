# 10 — Redes y Acceso LAN
## infra-monitoring · Exposición de puertos en WSL2 vs Linux nativo
**Mayo 2026**

---

## Índice

1. [El problema: solo un puerto accesible desde LAN en WSL2](#1-el-problema-solo-un-puerto-accesible-desde-lan-en-wsl2)
2. [Arquitectura de red WSL2](#2-arquitectura-de-red-wsl2)
3. [Solución A — netsh portproxy por puerto](#3-solución-a--netsh-portproxy-por-puerto)
4. [Solución B — WSL2 Mirrored Networking](#4-solución-b--wsl2-mirrored-networking)
5. [Linux nativo — cómo funciona y por qué es más simple](#5-linux-nativo--cómo-funciona-y-por-qué-es-más-simple)
6. [Tabla comparativa WSL2 vs Linux nativo](#6-tabla-comparativa-wsl2-vs-linux-nativo)

---

## 1. El problema: solo un puerto accesible desde LAN en WSL2

En el setup de desarrollo en WSL2, todos los compose files del ecosistema
usan bindings de puerto sin IP explícita (forma corta `"PORT:PORT"`), lo
que debería exponer los servicios en todas las interfaces. Sin embargo,
al intentar acceder desde otra máquina en la red local solo funciona
`http://192.168.1.220:8090` (el frontend), mientras que los demás
servicios solo responden en `localhost`.

| URL | Accesible desde LAN | Accesible en localhost |
|---|---|---|
| `:8090` — ticket-system-frontend | Sí | Sí |
| `:3000` — Grafana | No | Sí |
| `:9090` — Prometheus | No | Sí |
| `:5679` — n8n | No | Sí |
| `:8080` — ticket-system-backend | No | Sí |
| `:8001` — langchain-agent | No | Sí |
| `:8081` — notification-service | No | Sí |
| `:8002` — ticket-agent RAG API | No | Sí |

**Causa raíz:** la configuración en los compose files es correcta. El
problema es la capa de red que interpone WSL2 entre los contenedores y
la red LAN. El puerto 8090 funciona desde LAN porque tiene una regla
`netsh portproxy` y una excepción de Firewall de Windows configuradas
manualmente en algún momento; los demás no las tienen.

---

## 2. Arquitectura de red WSL2

WSL2 corre como una máquina virtual ligera (Hyper-V) con su propia
interfaz de red interna. Los contenedores bindan puertos dentro de esa VM,
no en el NIC físico de Windows.

```
Otra máquina (Windows/móvil)
        │
        │  192.168.1.220:PORT  ← IP física de la máquina Windows
        ▼
┌─────────────────────────────────────┐
│           Windows Host              │
│   NIC físico: 192.168.1.220         │
│                                     │
│   Relay automático localhost:PORT   │  ← Solo para conexiones locales
│           │                         │
│           ▼                         │
│   ┌─────────────────────┐           │
│   │   WSL2 VM (Linux)   │           │
│   │   IP interna: 172.x │           │
│   │                     │           │
│   │   Podman containers │           │
│   │   0.0.0.0:PORT      │           │
│   └─────────────────────┘           │
└─────────────────────────────────────┘
```

### Por qué `localhost` funciona

Windows mantiene un **relay automático** que intercepta conexiones a
`localhost:PORT` y las reenvía a la VM WSL2. Por eso todos los servicios
son accesibles desde el mismo equipo Windows.

### Por qué la IP LAN no funciona (por defecto)

El tráfico que llega a `192.168.1.220` (la IP del NIC físico) no pasa
por ese relay. Para que alcance a los contenedores en WSL2 hace falta
una regla explícita de reenvío de puertos.

### Por qué la IP de WSL2 cambia en cada reinicio

La VM WSL2 recibe una IP del rango `172.x` asignada dinámicamente por
Hyper-V. Cualquier regla que apunte a esa IP queda obsoleta tras un
`wsl --shutdown`. Los scripts de portproxy deben detectar la IP en
runtime.

---

## 3. Solución A — netsh portproxy por puerto

Requiere abrir PowerShell como Administrador en Windows.

### Script de configuración

```powershell
# Detectar IP actual de WSL2
$wsl_ip = (wsl hostname -I).Trim().Split(" ")[0]

# Puertos del ecosistema a exponer en LAN
$ports = @(3000, 5679, 8001, 8002, 8080, 8081, 8090, 9090)
# 3000  → Grafana
# 5679  → n8n (ticket-ingestion-light)
# 8001  → langchain-agent (ticket-classification)
# 8002  → ticket-agent RAG API
# 8080  → ticket-system-backend
# 8081  → notification-service
# 8090  → ticket-system-frontend
# 9090  → Prometheus

foreach ($port in $ports) {
    # Redirigir tráfico LAN → WSL2
    netsh interface portproxy add v4tov4 `
        listenaddress=0.0.0.0 `
        listenport=$port `
        connectaddress=$wsl_ip `
        connectport=$port

    # Abrir el puerto en el Firewall de Windows
    netsh advfirewall firewall add rule `
        name="WSL2 stack $port" `
        dir=in action=allow protocol=TCP localport=$port
}

Write-Host "Puertos expuestos en LAN para WSL2 IP: $wsl_ip"
netsh interface portproxy show all
```

### Verificar reglas activas

```powershell
netsh interface portproxy show all
```

### Eliminar todas las reglas (limpiar)

```powershell
netsh interface portproxy reset

$ports = @(3000, 5679, 8001, 8002, 8080, 8081, 8090, 9090)
foreach ($port in $ports) {
    netsh advfirewall firewall delete rule name="WSL2 stack $port"
}
```

### Cuándo ejecutarlo

Solo aplica en Windows con WSL2. Ejecutar manualmente en PowerShell como
Administrador después de cada reinicio de Windows, antes de acceder al
stack desde otra máquina en la red.

```powershell
# PowerShell como Administrador
.\wsl2-portproxy.ps1
```

En Mac o Linux nativo este script no es necesario — los puertos quedan
expuestos en LAN automáticamente al levantar el stack.

### Limitación

La IP interna de WSL2 cambia con cada reinicio de Windows. El script
la detecta en runtime, por eso hay que re-ejecutarlo tras cada reinicio.

---

## 4. Solución B — WSL2 Mirrored Networking

> **Requiere Windows 11 22H2 (build 22621) o posterior.**
> En Windows 10 la opción `networkingMode=mirrored` es silenciosamente
> ignorada por WSL2 — `ip addr show` no mostrará la IP LAN aunque el
> `.wslconfig` esté correctamente guardado. Verificado en Windows 10 Pro
> build 19045.

Disponible desde Windows 11 22H2 con WSL versión 2.0+. Con este modo
WSL2 comparte directamente las interfaces de red de Windows — todos los
puertos bindeados en WSL2 quedan automáticamente accesibles en la IP LAN
sin reglas manuales.

### Activar mirrored networking

Editar (o crear) el archivo `C:\Users\<usuario>\.wslconfig`:

```ini
[wsl2]
networkingMode=mirrored
```

Aplicar el cambio:

```powershell
wsl --shutdown
# Abrir de nuevo la terminal WSL2
```

### Verificar que está activo

```bash
# En WSL2 — debe mostrar la misma IP que el NIC físico de Windows
ip addr show eth0
```

### Ventajas sobre portproxy

| Aspecto | portproxy | mirrored |
|---|---|---|
| Configuración inicial | Script manual por puerto | Una línea en `.wslconfig` |
| Tras reiniciar | Re-ejecutar script | Automático |
| Puertos nuevos | Agregar al script | Automático |
| Compatibilidad | Windows 10+ | Windows 11 22H2+ |

### Consideración de red interna Podman

Con mirrored networking las subnets de Podman (`10.89.x.0/24`) pueden
coincidir con rangos de la red corporativa. Verificar que no hay
conflicto antes de activarlo:

```bash
podman network ls
# Comparar subnets con ip route en el host
```

---

## 5. Linux nativo — cómo funciona y por qué es más simple

En un servidor Linux nativo (física o VM), no existe la capa de
virtualización de WSL2. Podman y Docker crean reglas `iptables`/`nftables`
directamente en el kernel del host.

```
Otra máquina (Windows/móvil)
        │
        │  192.168.1.220:PORT
        ▼
┌─────────────────────────────────────┐
│        Servidor Linux               │
│   NIC: 192.168.1.220                │
│                                     │
│   iptables / nftables               │  ← regla creada por Podman al
│       │                             │    declarar "PORT:PORT"
│       ▼                             │
│   Contenedor: 0.0.0.0:PORT         │
└─────────────────────────────────────┘
```

Cuando el compose declara `"8090:80"`, Podman inserta una regla DNAT
que redirige el tráfico del puerto 8090 del host al contenedor. Eso
aplica a **todas las interfaces de red**, incluyendo el NIC físico
conectado a la LAN.

No hay portproxy, no hay relay, no hay configuración extra. Cualquier
máquina en la misma red puede acceder a `http://IP_SERVIDOR:8090`
inmediatamente después de levantar el stack.

### El único bloqueador: firewall del host

Si el servidor tiene firewall activo, hay que abrir los puertos
explícitamente.

**Ubuntu / Debian (`ufw`):**

```bash
sudo ufw status

sudo ufw allow 8090/tcp   # ticket-system-frontend
sudo ufw allow 8080/tcp   # ticket-system-backend
sudo ufw allow 8081/tcp   # notification-service
sudo ufw allow 8001/tcp   # langchain-agent
sudo ufw allow 8002/tcp   # ticket-agent RAG API
sudo ufw allow 5679/tcp   # n8n
sudo ufw allow 3000/tcp   # Grafana
sudo ufw allow 9090/tcp   # Prometheus

sudo ufw reload
```

**RHEL / Fedora / Rocky (`firewalld`):**

```bash
sudo firewall-cmd --list-ports

for PORT in 8090 8080 8081 8001 8002 5679 3000 9090; do
    sudo firewall-cmd --permanent --add-port=${PORT}/tcp
done

sudo firewall-cmd --reload
```

### Sin firewall activo (frecuente en entornos de desarrollo)

En muchas instalaciones de servidor Linux orientadas a desarrollo, `ufw`
y `firewalld` están desactivados. En ese caso todos los puertos del
compose quedan accesibles en LAN desde el primer `podman-compose up`.

```bash
# Verificar si ufw está activo
sudo ufw status
# Salida: "Status: inactive" → no bloquea nada

# Verificar si firewalld está activo
sudo systemctl is-active firewalld
# Salida: "inactive" → no bloquea nada
```

---

## 6. Tabla comparativa WSL2 vs Linux nativo

| Aspecto | WSL2 (desarrollo Windows) | Linux nativo (servidor) |
|---|---|---|
| Capa de red extra | Sí — VM Hyper-V con NAT | No |
| `localhost:PORT` | Funciona (relay automático) | Funciona |
| `IP_LAN:PORT` | Solo con portproxy o mirrored | Funciona automáticamente |
| IP del host cambia al reiniciar | Sí (IP interna de la VM) | No (IP fija del NIC) |
| Configuración extra para LAN | Sí (`netsh portproxy` o `.wslconfig`) | Solo si firewall está activo |
| Compose files a modificar | Ninguno | Ninguno |
| Reglas iptables creadas por Podman | En la VM WSL2 | En el host directamente |
| Acceso desde otra máquina sin config | No | Sí |

### Implicación para este stack

Los compose files del ecosistema ya están correctamente configurados
con bindings `"PORT:PORT"` en todos los servicios. En Linux nativo
funcionan sin cambios. La diferencia de acceso LAN es exclusivamente
una consecuencia de la arquitectura de red de WSL2, no de la
configuración de los contenedores.

---

*infra-monitoring · Redes y Acceso LAN · Mayo 2026*
