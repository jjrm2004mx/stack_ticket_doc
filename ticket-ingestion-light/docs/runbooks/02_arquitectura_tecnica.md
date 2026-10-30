# 02 — Arquitectura Técnica
## ticket-ingestion-light · Referencia de servicios, redes, workflows y configuración
**Octubre 2026**

Fuente de verdad verificada contra código: `openspec/specs/` (n8n-container,
stack-networking, stack-startup, configuration, channel-mailbox-workflows,
email-notification-workflow, email-ingestion-workflow).

---

## Índice

1. [Servicios del stack](#1-servicios-del-stack)
2. [Redes](#2-redes)
3. [Variables de entorno](#3-variables-de-entorno)
4. [Workflows](#4-workflows)
5. [Contratos HTTP](#5-contratos-http)
6. [Credenciales](#6-credenciales)
7. [Dependencias y orden de arranque](#7-dependencias-y-orden-de-arranque)

---

## 1. Servicios del stack

| Contenedor | Imagen | Puerto host | Puerto interno | Rol |
|---|---|---|---|---|
| `til-n8n` | `docker.io/n8nio/n8n` (sin versión fija) | 5679 (solo `127.0.0.1`) | 5678 | Motor de workflows: ingesta y envío de correo |

| Volumen | Contenido |
|---|---|
| `ticket-ingestion-light_n8n_data` | SQLite de n8n: workflows, credenciales (Microsoft Graph, llaves de canal), cursores de ingesta |

> **No borrar `n8n_data`** sin respaldarlo: las credenciales no se pueden exportar
> en claro y los cursores de ingesta se pierden (la ingesta vuelve al kickoff; el
> clasificador evita duplicados).

`restart: always`; DNS `8.8.8.8` y `8.8.4.4` (para resolver `graph.microsoft.com` y
`login.microsoftonline.com`).

---

## 2. Redes

| Red | Tipo | Propósito |
|---|---|---|
| `stack` | Interna (bridge) | Solo `til-n8n`; sin uso real |
| `ticket-management-network` | Externa (la crea `infra-monitoring/startup.sh`) | Alcanzar `langchain-agent` y que `notification-service` alcance a n8n |

Llamadas por nombre de servicio en `ticket-management-network`:

| Desde | Hacia | URL |
|---|---|---|
| Copia de ingesta | Clasificador | `http://langchain-agent:8001/inbox/channel` (GET) y `/inbox/email` (POST) |
| Copia de errores | Clasificador | `http://langchain-agent:8001/inbox/channel-errors` (POST) |
| notification-service | Copia de envío de un canal | `http://til-n8n:5678/webhook/canal-<id del canal>/send` |
| notification-service | Notificación de plataforma | `http://til-n8n:5678/webhook/send-email` (`NOTIFICATION_WEBHOOK_URL` en Rocky) |

---

## 3. Variables de entorno

| Variable (`.env`) | Llega al contenedor como | Uso |
|---|---|---|
| `N8N_HOST` | `N8N_HOST` | Hostname que reporta n8n (default `localhost`) |
| `N8N_WEBHOOK_URL` | `WEBHOOK_URL` | Base de las URLs de webhook que muestra n8n |
| `N8N_EDITOR_BASE_URL` | `N8N_EDITOR_BASE_URL` | URL de la interfaz |
| — | `N8N_PORT=5678`, `N8N_PROTOCOL=http`, `N8N_SECURE_COOKIE=false` | Fijas en el compose |

Las URLs de los workflows están dentro de cada workflow (en `n8n_data`), no en el `.env`.

---

## 4. Workflows

### Por canal (uno por buzón de cliente)

Se crean desde `n8n/workflows/templates/` siguiendo [05_alta_y_baja_de_buzon.md](05_alta_y_baja_de_buzon.md).

| Copia | Disparador | Qué hace |
|---|---|---|
| `Canal <slug> / <canal> — ingesta` | Cada 2 min | 1) `GET /inbox/channel` (con `X-Channel-Key`) → buzón y kickoff. 2) Lee del Inbox hasta 25 correos desde el cursor (leídos o no, más viejos primero). 3) Por correo: trae adjuntos (incluye inline), arma el cuerpo y hace `POST /inbox/email`. 4) Según la respuesta: `200/202` avanza el cursor; `400/413` lo registra y avanza; `401` detiene con error (llave rotada); `423/429/503`/otro/timeout 30 s detiene sin avanzar y reintenta en la siguiente corrida. Correos > 25 MB de cuerpo, con un archivo > 10 MB o con más de 50 archivos se tratan como `413`. **Nunca modifica el buzón.** |
| `Canal <slug> / <canal> — errores` | Error de la ingesta | Reporta el error al clasificador (`POST /inbox/channel-errors`) a lo más 1 vez por hora; se ve en el estado del canal en ticket-management. |
| `Canal <slug> / <canal> — envío` | Webhook `POST /webhook/canal-<id>/send` (`X-Send-Secret`) | Envía `{to, subject, html}` desde el buzón del canal como **correo nuevo** (`sendMail`); responde `200 {"status":"sent"}` o `502 {"status":"failed","error":...}`; secreto inválido → `403`. |

El cursor vive en el *static data* de la copia de ingesta. Si se borra o reimporta la
copia, o el operador cambia el kickoff, la lectura vuelve a empezar en el kickoff.

### De plataforma

| Workflow | Id en Rocky | Qué hace |
|---|---|---|
| `Clasificador de tickets notification` | `Ur6k52xhukzGfEX9` | Webhook `POST /webhook/send-email` sin autenticación → `sendMail` desde `sharedservicesmx@hitss.com`. Lo usa notification-service para los correos que no son de un canal. Export: `n8n/workflows/2026-10-15-notificacion-microsoft-graph.json`. Se retirará cuando todo salga por canal. |

### Cómo elige notification-service por dónde enviar

1. Aviso de un ticket con canal → la fila **activa** de ese canal en
   `notification_provider_configs` (URL de la copia de envío). Si no hay fila activa,
   **falla cerrado**: no se envía desde otro buzón.
2. Sin canal → fila del tenant, luego fila global, luego `NOTIFICATION_WEBHOOK_URL`
   (en Rocky: `http://til-n8n:5678/webhook/send-email`).

### Workflows viejos

En el n8n de Rocky siguen, **inactivos**, `SS- Clasificador de tickets - Microsoft Graph`
(y sus variantes `V6`), `SSS- …` (`qm6Jfd7ZuggRfJAw`) y `POC2 - …`. Leían el buzón
compartido y posteaban a `/process`, que ya no existe. No activarlos.

---

## 5. Contratos HTTP

### `POST /inbox/email` (cuerpo que envía la ingesta)

```json
{
  "externalId": "<Graph id>",
  "threadKey": "<conversationId>",
  "internetMessageId": "<internetMessageId>",
  "from": {"address": "...", "name": "..."},
  "to": ["..."],
  "subject": "...",
  "bodyText": "...",
  "bodyHtml": "...",
  "receivedAt": "<receivedDateTime>",
  "importance": "low | normal | high",
  "cc": ["..."],
  "sender": {"address": "...", "name": "..."},
  "replyTo": ["..."],
  "sentAt": "<sentDateTime>",
  "conversationIndex": "<conversationIndex>",
  "messageType": "message | eventMessageRequest | …",
  "inferenceClassification": "focused | other",
  "categories": ["..."],
  "flagStatus": "notFlagged | flagged | complete",
  "uniqueBodyHtml": "...", "uniqueBodyText": "...",
  "headers": {"in-reply-to": "<…>", "references": "<…> <…>", "auto-submitted": "...", "…": "…"},
  "attachments": [{"name": "...", "contentType": "...", "size": 0,
                   "contentBase64": "...", "inline": false, "contentId": null}]
}
```

Cabecera `X-Channel-Key: <llave del canal>`. La ingesta **transporta, no decide**: nunca descarta ni
reescribe un correo por su contenido (decisión del 2026-10-29). El clasificador decide si es ticket,
respuesta o duplicado (ver ticket-classification, `02_arquitectura_tecnica.md` §3 y
`09_bandeja_mensajes_retenidos.md`).

De estos campos dependen funciones de la plataforma; **quitarlos de una copia del workflow las apaga
para ese buzón**:

| Campo | Lo usa |
|---|---|
| `cc`, `to`, `sender` | Destinatarios y "a quién" del hilo del correo; ¿es ticket? (¿va dirigido al buzón?, ¿lo mandó el equipo?) |
| `uniqueBodyText/Html` | Texto propio sin la cita (hilo del correo, ¿es ticket?) |
| `headers.in-reply-to`, `headers.references` | Ligar una respuesta a su ticket aunque Outlook le haya cambiado la conversación |
| `headers.auto-submitted`, `precedence`, `list-*`, `x-canal-aviso`, `messageType` | Reglas fijas de ¿es ticket? (automáticos, masivos, invitaciones, avisos propios) |
| `importance` | Prioridad del ticket |
| `threadKey`, `internetMessageId` | Ligar respuestas por conversación y por encabezados |

La lista completa y exacta está en la spec `channel-mailbox-workflows` (requisito "Full message transport").

### Webhooks de envío

| Webhook | Cuerpo | Autenticación |
|---|---|---|
| `/webhook/canal-<id>/send` | `{to, subject, html, originMessageRef}` (`originMessageRef` se ignora) | `X-Send-Secret` |
| `/webhook/send-email` | Ver spec `email-notification-workflow` | Ninguna |

---

## 6. Credenciales (en n8n → Credentials)

| Credencial | Uso |
|---|---|
| `HITSS - Shared Services - Outlook AppOnly` | Acceso de aplicación (Graph) a los buzones de HITSS: canal del cliente inicial y workflow de plataforma |
| `Microsoft Outlook OAuth2` por cliente | Buzón de un cliente externo, con la app multi-tenant de Azure (**pendiente**: la app aún no se registra) |
| `Canal <slug> / <canal> — X-Channel-Key` | Header auth de la ingesta y de errores |
| `Canal <slug> / <canal> — X-Send-Secret` | Header auth del webhook de envío |

Nunca se escriben llaves o secretos en texto dentro de los nodos.

---

## 6.1 Proveedores de buzón

Solo **Microsoft 365** (Graph) está implementado. El canal de ticket-management tiene
`provider` (`MICROSOFT_365` hoy; `GOOGLE` reservado): un proveedor nuevo como **Gmail** sería un
juego de plantillas propio (ingesta, errores, envío) con su credencial OAuth de Google, que
postee el mismo contrato de `POST /inbox/email` (§5); clasificador, ticket-management y
notification-service no cambian (este último solo necesita la URL del webhook de envío de la
copia del canal). Ver `01_ingestion_for_dummies.md`, "Proveedores de buzón".

## 7. Dependencias y orden de arranque

| Dependencia | Necesaria para |
|---|---|
| `ticket-management-network` | Arrancar el contenedor (sin ella `podman-compose up` falla) |
| `langchain-agent` (ticket-classification) | Ingesta. Si no está, la corrida se detiene sin avanzar el cursor y reintenta |
| Microsoft Graph (internet) | Leer buzones y enviar correo |

`infra-monitoring/startup.sh up` levanta todo en orden: ticket-classification y
ticket-management antes que este stack.

---

*ticket-ingestion-light · Arquitectura Técnica · Octubre 2026*
