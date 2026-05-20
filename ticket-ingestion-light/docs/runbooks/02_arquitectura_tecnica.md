# 02 — Arquitectura Técnica
## ticket-ingestion-light · Referencia de servicios, redes y configuración
**Mayo 2026**

---

## Índice

1. [Servicios del stack](#1-servicios-del-stack)
2. [Redes](#2-redes)
3. [Variables de entorno](#3-variables-de-entorno)
4. [Workflow n8n — estructura](#4-workflow-n8n--estructura)
5. [Payload enviado al agente](#5-payload-enviado-al-agente)
6. [Dependencias externas](#6-dependencias-externas)

---

## 1. Servicios del stack

Este repo tiene un único contenedor:

| Contenedor | Imagen | Puerto host | Puerto interno | Rol |
|---|---|---|---|---|
| `til-n8n` | `docker.io/n8nio/n8n` | 5679 | 5678 | Motor de automatización |

### Persistencia

| Volumen | Contenido |
|---|---|
| `n8n_data` | Base de datos SQLite de n8n + workflows + credenciales configuradas |

> El volumen `n8n_data` contiene credenciales de Gmail (OAuth token) y el
> workflow completo. **No borrar** sin antes exportar el workflow.

---

## 2. Redes

```yaml
networks:
  stack:
    driver: bridge                   # Red interna del repo
  ticket-management-network:
    external: true                   # Red compartida del ecosistema (creada por startup.sh)
```

| Red | Tipo | Propósito |
|---|---|---|
| `stack` | Interna | Comunicación local (solo un contenedor, sin uso real) |
| `ticket-management-network` | Externa | Alcanzar servicios del ecosistema |

### Alcance al langchain-agent

`til-n8n` alcanza al `langchain-agent` (en ticket-classification) a través
del host Podman, no por red interna:

```
til-n8n → http://host.containers.internal:8001/process
```

`host.containers.internal` es la IP del host desde dentro del contenedor.
En WSL2, apunta a la IP del gateway de la red Podman.

---

## 3. Variables de entorno

### `.env.example`

```bash
# IP o dominio donde corre este stack
N8N_HOST=localhost
N8N_WEBHOOK_URL=http://localhost:5679
N8N_EDITOR_BASE_URL=http://localhost:5679

# URL del agente clasificador — referencia documental
# La URL real está hardcodeada en el nodo HTTP del workflow de n8n
POC2_PROCESS_URL=http://host.containers.internal:8001/process
```

### Variables inyectadas en el contenedor

| Variable | Valor | Descripción |
|---|---|---|
| `N8N_HOST` | `localhost` | Hostname que n8n reporta |
| `N8N_PORT` | `5678` | Puerto interno (fijo) |
| `N8N_PROTOCOL` | `http` | Protocolo |
| `WEBHOOK_URL` | `${N8N_WEBHOOK_URL}` | URL base para webhooks |
| `N8N_EDITOR_BASE_URL` | `${N8N_EDITOR_BASE_URL}` | URL de la interfaz web |
| `N8N_SECURE_COOKIE` | `false` | Deshabilita cookies seguras (desarrollo HTTP) |

### Variables que NO afectan al agente

`POC2_PROCESS_URL` en `.env` es **solo referencia documental**. n8n no lee
el `.env` del host — la URL real está configurada directamente en el nodo
HTTP del workflow dentro de `n8n_data`.

---

## 4. Workflow n8n — estructura

```
┌─────────────────────────────────────────────────────────┐
│  Workflow: Gmail → Agente Clasificador                   │
│                                                         │
│  ┌──────────────────┐      ┌──────────────────────────┐ │
│  │  Gmail Trigger   │─────▶│  HTTP Request            │ │
│  │                  │      │                          │ │
│  │  • Cuenta Gmail  │      │  POST /process           │ │
│  │  • Poll interval │      │  host.containers.        │ │
│  │  • Filtros       │      │  internal:8001           │ │
│  └──────────────────┘      └──────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Gmail Trigger — campos extraídos

| Campo n8n | Descripción |
|---|---|
| `subject` | Asunto del correo |
| `text` | Cuerpo en texto plano |
| `html` | Cuerpo en HTML |
| `from.value[0].address` | Email del remitente |
| `from.value[0].name` | Nombre del remitente |
| `attachments` | Array de adjuntos |
| `threadId` | ID del hilo Gmail (para deduplicación en el agente) |

---

## 5. Payload enviado al agente

El nodo HTTP Request construye el body JSON enviado a `POST /process`:

```json
{
  "asunto": "{{ $json.subject }}",
  "cuerpo": "{{ $json.text }}",
  "remitente": "{{ $json.from.value[0].address }}",
  "nombre_remitente": "{{ $json.from.value[0].name }}",
  "conversation_id": "{{ $json.threadId }}",
  "adjuntos": []
}
```

El `conversation_id` es el `threadId` de Gmail — permite al agente deduplicar
correos del mismo hilo.

### Respuesta esperada del agente

```json
{
  "job_id": "uuid-del-job",
  "status": "en_proceso"
}
```

n8n no hace polling — envía el correo y no espera el resultado final
de la clasificación.

---

## 6. Dependencias externas

| Dependencia | Rol | Requerida para arrancar |
|---|---|---|
| `ticket-management-network` | Red externa compartida | Sí — creada por startup.sh |
| `langchain-agent:8001` | Destino del POST | No — n8n arranca sin él, falla al procesar |
| Cuenta Gmail OAuth | Autenticación del trigger | Sí — configurar en n8n la primera vez |

### Orden de arranque recomendado

```
1. ticket-classification (langchain-agent debe estar disponible)
2. ticket-ingestion-light (til-n8n puede ahora alcanzar al agente)
```

El `startup.sh` de infra-monitoring gestiona este orden automáticamente.

---

*ticket-ingestion-light · Arquitectura Técnica · Mayo 2026*
