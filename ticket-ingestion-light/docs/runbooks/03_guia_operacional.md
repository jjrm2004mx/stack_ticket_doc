# 03 — Guía Operacional
## ticket-ingestion-light · Arranque, verificación, puesta en marcha de n8n y operación diaria
**Octubre 2026**

Todos los comandos se corren **en Rocky** (`ssh Agente@10.187.21.30`), en
`~/stack_ticket/ticket-ingestion-light`. La interfaz de n8n se abre desde Windows en
`http://localhost:5679` con el túnel SSH (infra-monitoring `10_redes_y_acceso_lan.md` §7).
Comandos sueltos: [operacion.md](operacion.md). Fallas: [troubleshooting.md](troubleshooting.md).

---

## Índice

1. [Arranque y parada](#1-arranque-y-parada)
2. [Verificación de salud](#2-verificación-de-salud)
3. [Estado esperado de los workflows](#3-estado-esperado-de-los-workflows)
4. [n8n desde cero (volumen nuevo)](#4-n8n-desde-cero-volumen-nuevo)
5. [Operación diaria](#5-operación-diaria)
6. [Actualizar n8n o este repo](#6-actualizar-n8n-o-este-repo)
7. [Respaldo y restauración de n8n_data](#7-respaldo-y-restauración-de-n8n_data)

---

## 1. Arranque y parada

```bash
# Todo el ecosistema, en orden (recomendado tras reiniciar Rocky)
bash ~/stack_ticket/infra-monitoring/startup.sh up

# Solo este stack (la red ticket-management-network ya debe existir)
cd ~/stack_ticket/ticket-ingestion-light && ./start.sh

# Parar (el volumen n8n_data se conserva)
cd ~/stack_ticket/ticket-ingestion-light && podman-compose down
```

`start.sh`: crea `.env` desde `.env.example` si falta, hace `podman-compose down` + `up -d`
y muestra `podman ps`. n8n escucha solo en `127.0.0.1:5679` de Rocky.

**Nunca** `podman-compose down -v`: borra workflows, credenciales y cursores.

---

## 2. Verificación de salud

```bash
podman ps --filter name=til-n8n                  # STATUS: Up
curl -s http://localhost:5679/healthz            # {"status":"ok"}
podman exec til-n8n n8n list:workflow --active=true   # workflows activos (ver §3)
podman exec til-n8n wget -qO- http://langchain-agent:8001/health   # n8n alcanza al clasificador
```

`infra-monitoring/health-check.sh` incluye el chequeo de n8n junto con el resto del stack.

El estado de cada canal (última lectura, errores de autorización del buzón) se ve en
ticket-management: consola `/plataforma` → Clientes → cliente → "Canales de correo", y
el admin del cliente en "📨 Canales".

---

## 3. Estado esperado de los workflows

Estado en Rocky al 2026-10-17 (decisión del dueño: la ingesta y los avisos de canal
están **apagados a propósito** mientras se afina el core):

| Workflow | Estado esperado |
|---|---|
| `Clasificador de tickets notification` (`send-email`) | **Activo** |
| `Canal cliente-inicial / Shared Services — ingesta` | Inactivo (se activa cuando el dueño lo decida) |
| `Canal cliente-inicial / Shared Services — errores` | Inactivo |
| `Canal cliente-inicial / Shared Services — envío` | Inactivo (su fila en `notification_provider_configs` también está inactiva) |
| `SS- …`, `SSS- …`, `POC2 - …` (viejos) | Inactivos siempre |

Mientras el envío del canal esté inactivo, los avisos de tickets de ese canal **no se
mandan** (fallan cerrados en notification-service). Los correos de cuenta y contraseña sí
salen por `send-email`.

Para encender un canal: activar primero errores y envío, luego ingesta, y activar su fila
de proveedor en notification-service ([05_alta_y_baja_de_buzon.md](05_alta_y_baja_de_buzon.md)).

---

## 4. n8n desde cero (volumen nuevo)

Si `n8n_data` se perdió y no hay respaldo (si lo hay, ver §7):

1. Abrir `http://localhost:5679` y crear la cuenta de administrador local de n8n.
2. **Credencial de Graph**: Credentials → nueva credencial de tipo Microsoft Entra
   (service principal) `HITSS - Shared Services - Outlook AppOnly` con tenant, client id y
   secreto de la app de Azure de HITSS (permisos de aplicación para leer y enviar correo).
3. **Notificación de plataforma**: importar `n8n/workflows/2026-10-15-notificacion-microsoft-graph.json`,
   asignar la credencial al nodo "Send Email - Microsoft Graph" y activarlo.
4. **Cada canal**: seguir el alta de [05_alta_y_baja_de_buzon.md](05_alta_y_baja_de_buzon.md)
   (credenciales de cabecera, 3 copias desde las plantillas). La llave del canal no se puede
   recuperar: hay que **rotarla** en la consola y usar la nueva.
5. La ingesta empezará desde el kickoff de cada canal; el clasificador descarta lo que ya
   tenía (responde `200`), así que no se duplican tickets.

---

## 5. Operación diaria

| Tarea | Cómo |
|---|---|
| Ver ejecuciones | n8n → Executions (filtrar por workflow; rojo = error, clic para ver el nodo que falló) |
| Pausar un canal | Desactivar su copia "— ingesta" (el cursor se conserva; al reactivar carga lo pendiente) |
| Mover el inicio de lectura | Consola de plataforma → canal → "Cambiar fecha" (kickoff). La siguiente corrida empieza en esa fecha |
| Rotar la llave de un canal | Consola → "Rotar llave" → actualizar la credencial `Canal … — X-Channel-Key` en n8n. Mientras no se actualice, la ingesta falla con `401` y se reporta como error del canal |
| Alta / baja de buzón | [05_alta_y_baja_de_buzon.md](05_alta_y_baja_de_buzon.md) |
| Cambiar una plantilla | Editar el JSON en `n8n/workflows/templates/`, PR, y **reimportar a mano** cada copia existente (los cambios no se propagan solos) |
| Exportar un workflow | `podman exec til-n8n n8n export:workflow --id=<id> --output=/tmp/wf.json` y `podman cp til-n8n:/tmp/wf.json .` (las copias de canal no se versionan) |

---

## 6. Actualizar n8n o este repo

```bash
cd ~/stack_ticket/ticket-ingestion-light
git pull                                  # Rocky sigue origin/develop
podman pull docker.io/n8nio/n8n           # solo si se quiere una imagen nueva
podman-compose up -d --force-recreate     # sin --force-recreate no siempre se recrea
curl -s http://localhost:5679/healthz
```

La imagen no tiene versión fija: una versión nueva de n8n puede cambiar valores por
defecto de los nodos. Respaldar `n8n_data` antes (§7) y revisar Executions después.

---

## 7. Respaldo y restauración de n8n_data

```bash
# Respaldo (con n8n parado para que el SQLite quede consistente)
cd ~/stack_ticket/ticket-ingestion-light && podman-compose down
podman volume export ticket-ingestion-light_n8n_data --output ~/respaldos/n8n_data_$(date +%F).tar
./start.sh

# Restauración
podman-compose down
podman volume import ticket-ingestion-light_n8n_data ~/respaldos/n8n_data_<fecha>.tar
./start.sh
```

El respaldo contiene las credenciales de Graph y las llaves de los canales: guardarlo
solo en Rocky (`~/respaldos/`), nunca en git. El respaldo integral del ecosistema (las 3
bases PostgreSQL + `n8n_data`) se describe en infra-monitoring.

---

*ticket-ingestion-light · Guía Operacional · Octubre 2026*
