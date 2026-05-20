# 03 — Guía Operacional
## ticket-ingestion-light · Arranque, verificación y troubleshooting
**Mayo 2026**

---

## Índice

1. [Arranque del stack](#1-arranque-del-stack)
2. [Verificación de salud](#2-verificación-de-salud)
3. [Configurar n8n por primera vez](#3-configurar-n8n-por-primera-vez)
4. [Operaciones del día a día](#4-operaciones-del-día-a-día)
5. [Troubleshooting](#5-troubleshooting)
6. [Referencia rápida](#6-referencia-rápida)

---

## 1. Arranque del stack

### Arranque integrado (recomendado)

```bash
# Desde infra-monitoring — levanta todo el ecosistema en orden
cd ~/stack_ticket/infra-monitoring
./startup.sh up
# o equivalente:
make up
```

### Arranque independiente

```bash
cd ~/stack_ticket/ticket-ingestion-light
./start.sh
```

`start.sh` hace:
1. Crea `.env` desde `.env.example` si no existe
2. Actualiza `POC2_PROCESS_URL` en `.env` (referencia documental)
3. `podman-compose down` → `podman-compose up -d`
4. Muestra estado de contenedores

### Parar el stack

```bash
cd ~/stack_ticket/ticket-ingestion-light
podman-compose down
```

---

## 2. Verificación de salud

```bash
# Ver estado del contenedor
podman ps --filter name=til-n8n

# Ver logs en tiempo real
podman logs --tail 50 til-n8n

# Acceder a la interfaz web de n8n
# Abrir en Windows: http://localhost:5679
```

### Verificar que n8n responde

```bash
# Desde WSL
curl -s http://localhost:5679/healthz
# Respuesta esperada: {"status":"ok"}
```

### Verificar que el workflow está activo

En la interfaz web `http://localhost:5679`:
- Menu izquierdo → **Workflows**
- El workflow "Gmail → Agente" debe aparecer con estado **Active** (toggle verde)

---

## 3. Configurar n8n por primera vez

Si el volumen `n8n_data` está vacío o es una instalación nueva:

### Paso 1 — Acceder a n8n

```
http://localhost:5679
```

Primera vez: n8n pide crear una cuenta de administrador local (usuario + password).
Esta cuenta solo existe localmente — no es una cuenta en la nube de n8n.

### Paso 2 — Configurar credencial Gmail

1. Menu izquierdo → **Credentials**
2. **Add Credential** → buscar "Gmail"
3. Seguir el flujo OAuth:
   - Necesitas un proyecto en Google Cloud Console con la API de Gmail habilitada
   - Crear credenciales OAuth 2.0 (tipo: Web Application)
   - Redirect URI: `http://localhost:5679/rest/oauth2-credential/callback`
4. Autorizar la cuenta Gmail que recibirá los correos

### Paso 3 — Importar o crear el workflow

**Opción A — Importar desde archivo (si existe backup):**
1. Menu → **Workflows** → **Import from file**
2. Seleccionar el archivo `.json` del workflow exportado

**Opción B — Crear desde cero:**
1. **New Workflow**
2. Agregar nodo **Gmail Trigger**:
   - Credencial: la configurada en Paso 2
   - Evento: `Message Received`
   - Poll every: `1 minute` (o el intervalo deseado)
3. Agregar nodo **HTTP Request**:
   - Method: `POST`
   - URL: `http://host.containers.internal:8001/process`
   - Body: JSON con los campos del correo (ver sección de payload en arquitectura técnica)
4. Conectar nodos → **Save** → **Activate** (toggle)

### Paso 4 — Verificar el flujo

```bash
# Enviar un correo de prueba a la cuenta Gmail configurada
# Esperar el intervalo de polling
# Verificar en n8n: Workflows → Executions (historial de ejecuciones)
```

---

## 4. Operaciones del día a día

### Ver historial de ejecuciones del workflow

En n8n web `http://localhost:5679`:
- Menu → **Executions**
- Cada fila es una ejecución del workflow
- Click en una ejecución para ver los datos de entrada/salida de cada nodo

### Ver si llegó un correo específico

```bash
# Logs del contenedor muestran la actividad del poller
podman logs --tail 100 til-n8n | grep -i gmail
```

### Cambiar la URL del agente

La URL `http://host.containers.internal:8001/process` está hardcodeada en el
nodo HTTP del workflow dentro de n8n.

Para cambiarla:
1. `http://localhost:5679` → Workflows → abrir el workflow
2. Click en el nodo HTTP Request
3. Cambiar el campo **URL**
4. Save → el cambio tiene efecto inmediato (sin reiniciar)

### Exportar el workflow (backup)

1. `http://localhost:5679` → Workflows → abrir el workflow
2. Menu (3 puntos) → **Download** → guarda un `.json`
3. Guardar el `.json` en un lugar seguro (el volumen puede perderse)

### Pausar el workflow temporalmente

En n8n web: Workflows → toggle del workflow → **Inactive**
El workflow deja de hacer polling pero el contenedor sigue corriendo.

---

## 5. Troubleshooting

### til-n8n no arranca

```bash
# Ver por qué falló
podman logs til-n8n

# Causa frecuente: ticket-management-network no existe
podman network ls | grep ticket-management-network
# Si no existe:
podman network create --subnet 10.89.1.0/24 ticket-management-network
# O levantar con startup.sh que la crea automáticamente
```

### El workflow no procesa correos

**1. Verificar que el workflow está activo:**
- `http://localhost:5679` → Workflows → estado debe ser **Active**

**2. Verificar credencial Gmail:**
- Credentials → Gmail → comprobar que no aparece error de OAuth expirado
- Si expiró: re-autenticar

**3. Verificar la URL del agente:**
```bash
# Desde dentro del contenedor, probar si el agente responde
podman exec til-n8n wget -qO- http://host.containers.internal:8001/health
# Debe responder JSON con status ok
```

**4. Revisar ejecuciones fallidas:**
- n8n web → Executions → buscar ejecuciones con estado Error
- Click en la ejecución roja para ver el error exacto

### Error de red al llegar al agente

```bash
# Verificar que langchain-agent está corriendo
podman ps | grep langchain-agent

# Verificar puertos
podman port langchain-agent
# Debe mostrar: 8001/tcp -> 0.0.0.0:8001
```

### Workflow perdido (volumen recreado)

Si el volumen `n8n_data` fue eliminado, n8n arranca sin workflows ni credenciales.
Volver al [Paso 3 — Importar o crear el workflow](#paso-3--importar-o-crear-el-workflow).

Lección: exportar el workflow como `.json` periódicamente como backup.

---

## 6. Referencia rápida

| Acción | Comando / URL |
|---|---|
| Arrancar stack | `./start.sh` |
| Parar stack | `podman-compose down` |
| Logs en vivo | `podman logs -f til-n8n` |
| Interfaz n8n | http://localhost:5679 |
| Health check | `curl http://localhost:5679/healthz` |
| Ver ejecuciones | http://localhost:5679 → Executions |
| Reiniciar contenedor | `podman restart til-n8n` |

---

*ticket-ingestion-light · Guía Operacional · Mayo 2026*
