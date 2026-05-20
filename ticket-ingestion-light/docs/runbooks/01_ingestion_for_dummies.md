# 01 — Ingestion for Dummies
## ticket-ingestion-light · Guía conceptual del stack
**Stack: n8n · Gmail Trigger · Podman**
**Mayo 2026**

---

## La analogía: el buzón automático del ecosistema

Imagina una oficina de correos que tiene un empleado cuyo único trabajo es
vigilar el buzón de Gmail, abrir cada correo que llegue y entregarlo
automáticamente al clasificador.

| Pieza del correo | Pieza del ecosistema |
|---|---|
| El empleado que vigila el buzón | n8n — motor de automatización visual |
| La bandeja de entrada de Gmail | Gmail Trigger — escucha correos nuevos |
| El sobre con la carta dentro | Payload JSON: asunto + cuerpo + adjuntos |
| La ventanilla de entrega | HTTP POST a langchain-agent:8001/process |
| El registro de entregas | n8n_data (volumen SQLite interno de n8n) |

---

## Rol en el ecosistema

Este repo es la **puerta de entrada** del ecosistema para correos Gmail.
No clasifica, no guarda tickets, no notifica — solo detecta correos y los
reenvía al clasificador.

```
Gmail (bandeja)
     │
     │ Gmail Trigger (cada N minutos)
     ▼
  til-n8n :5679
  ┌──────────────────────────────────┐
  │  Workflow n8n                    │
  │  1. Gmail Trigger                │
  │     └─ Lee correos nuevos        │
  │  2. HTTP Request                 │
  │     └─ POST /process al agente   │
  └──────────────────────────────────┘
     │
     │ POST http://host.containers.internal:8001/process
     ▼
  langchain-agent (ticket-classification)
  └─ Clasifica, guarda, notifica
```

---

## El ecosistema completo

Este repo forma parte de un ecosistema de 5 servicios. Su posición es la primera
en la cadena de procesamiento de correos.

```
ticket-ingestion-light  ──▶  ticket-classification  ──▶  ticket-management
(detección Gmail)              (clasificación IA)           (registro tickets)
                                                                   │
                                                                   ▼
                                                        notification-service
                                                        (avisos por correo)

infra-monitoring  ◀──  métricas de todos los repos
```

---

## n8n — El motor de automatización

n8n es una herramienta de automatización visual con interfaz web. Permite
conectar servicios sin escribir código, usando nodos que se encadenan en un
flujo visual llamado **workflow**.

| Característica | Detalle |
|---|---|
| Interfaz | http://localhost:5679 (desde Windows) |
| Contenedor | `til-n8n` |
| Puerto host | 5679 → 5678 interno |
| Persistencia | Volumen `n8n_data` (SQLite interno) |
| Sin BD externa | No requiere PostgreSQL — usa SQLite embebido |

> **Primera vez:** n8n arranca sin workflows. El workflow de Gmail debe
> importarse o configurarse manualmente desde la interfaz web.

---

## Workflow: Gmail Trigger → HTTP Request

El workflow tiene exactamente dos nodos:

### Nodo 1 — Gmail Trigger

- Monitorea la bandeja de entrada de Gmail con la cuenta configurada
- Se activa cuando llega un correo nuevo
- Extrae: asunto, cuerpo (HTML y texto plano), remitente, adjuntos
- Frecuencia: configurable (por defecto cada 1-5 minutos)

### Nodo 2 — HTTP Request

- Hace `POST` al agente clasificador con el contenido del correo
- URL: `http://host.containers.internal:8001/process`
- Body JSON con los campos que el agente espera

> **Importante:** La URL del agente está **hardcodeada en el nodo HTTP de n8n**,
> no se inyecta desde el `.env`. La variable `POC2_PROCESS_URL` en `.env` es
> solo referencia documental.

---

## Redes — cómo alcanza al agente

n8n vive en un contenedor Podman. Para llegar al `langchain-agent` que corre
en otro stack (ticket-classification), usa la dirección especial:

```
http://host.containers.internal:8001/process
```

Esta dirección apunta al host de Podman (la máquina WSL), desde donde el
puerto 8001 del agente es accesible.

| Red | Rol |
|---|---|
| `stack` | Red interna del repo (solo til-n8n) |
| `ticket-management-network` | Red externa compartida del ecosistema |

---

## Cuándo usar este repo

| Escenario | Acción |
|---|---|
| El ecosistema procesa correos de Gmail | Este repo debe estar levantado |
| Solo se prueba el clasificador (curl manual) | Este repo puede estar detenido |
| Se prueba con Power Automate / Outlook | Este repo no interviene |
| El agente no recibe correos de Gmail | Verificar logs de til-n8n |

---

*ticket-ingestion-light · Ingestion for Dummies · Mayo 2026*
