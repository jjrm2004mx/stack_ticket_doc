# 01 — Ingestion for Dummies
## ticket-ingestion-light · Guía conceptual del stack
**Stack: n8n · Microsoft Graph (Outlook) · Podman**
**Octubre 2026**

---

## La analogía: el cartero de cada buzón

Cada cliente tiene uno o más buzones de Outlook (por ejemplo `soporte@cliente.com`).
Para cada buzón hay un cartero que pasa cada 2 minutos, **fotocopia** los correos
nuevos (no los toca: el buzón lo atiende una persona) y los deja en la bandeja del
clasificador. El mismo cartero también entrega los avisos de los tickets de ese
buzón, firmados con la dirección del buzón.

| Pieza de la analogía | Pieza del ecosistema |
|---|---|
| El buzón del cliente | Buzón de Outlook / Microsoft 365, leído con Microsoft Graph |
| El cartero de un buzón | Las 3 copias de workflow del **canal** en n8n (ingesta, errores, envío) |
| La libreta donde anota hasta dónde leyó | El **cursor** de la copia de ingesta (vive en n8n) |
| La bandeja donde deja las fotocopias | `POST /inbox/email` del clasificador (`langchain-agent`, ticket-classification) |
| La oficina central | `til-n8n` (contenedor), con sus datos en el volumen `n8n_data` |

---

## Rol en el ecosistema

Este repo es la **puerta de entrada y salida de correo** del ecosistema:

- **Entrada**: lee los buzones de los clientes y entrega cada correo a la bandeja del
  clasificador. No clasifica ni crea tickets; eso lo hace ticket-classification.
- **Salida**: envía por Microsoft Graph los correos que le pide notification-service
  (avisos de tickets y correos de cuenta).

```
Buzón Outlook del canal                         notification-service
     │ (Graph, solo lectura, cada 2 min)                │
     ▼                                                  │ POST webhook
  til-n8n :5679 (5678 interno)  ◀────────────────────────┘
  ├─ Canal <cliente> / <canal> — ingesta  ──▶ POST langchain-agent:8001/inbox/email
  ├─ Canal <cliente> / <canal> — errores  ──▶ POST langchain-agent:8001/inbox/channel-errors
  ├─ Canal <cliente> / <canal> — envío    ──▶ Graph sendMail desde el buzón del canal
  └─ Clasificador de tickets notification ──▶ Graph sendMail desde el buzón de la plataforma
       (webhook /webhook/send-email)
```

---

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Canal** | Un buzón de un cliente, dado de alta por el operador en la consola de plataforma de ticket-management (`/plataforma` → Clientes → cliente → "Canales de correo"). Tiene una **llave** (`X-Channel-Key`). |
| **Plantillas** | `n8n/workflows/templates/` (ingesta, errores, envío). Son la fuente del comportamiento; **no se editan** para un cliente. |
| **Copias del canal** | Por cada canal se importan las 3 plantillas y se llenan los datos del canal. Las copias no se versionan. Procedimiento: [05_alta_y_baja_de_buzon.md](05_alta_y_baja_de_buzon.md). |
| **Kickoff ("Cargar correos desde")** | Fecha desde la que se cargan correos; la fija el operador en la consola. Nada anterior se carga. |
| **Cursor** | Hasta qué correo ya se entregó. Si n8n estuvo apagado días, al volver carga todo lo pendiente (25 por corrida, los más viejos primero). |
| **Workflow de notificación de plataforma** | `Clasificador de tickets notification` (webhook `send-email`). Envía los correos que no pertenecen a un canal (por ejemplo los de cuenta y contraseña). |

---

## Proveedores de buzón: Outlook hoy, Gmail como opción

| Proveedor | Estado |
|---|---|
| **Microsoft 365 / Outlook** (Microsoft Graph) | **Implementado.** Es el único que se usa hoy: plantillas `ingesta`, `errores` y `envío` |
| **Gmail / Google Workspace** | **Opción prevista, no implementada.** ticket-management ya reserva el proveedor `GOOGLE` en el canal |

Un cliente cuyo buzón está en Gmail se podría atender sin cambiar el resto del ecosistema: la
bandeja del clasificador (`POST /inbox/email`) no depende del proveedor, solo recibe el correo
con la llave del canal. Haría falta un cambio en este repo con:

- plantillas propias de Gmail (ingesta con la API de Gmail o IMAP, errores y envío), que armen el
  mismo cuerpo de `/inbox/email` (`externalId` = id del mensaje, `threadKey` = `threadId`);
- credencial OAuth de Google por cliente (proyecto en Google Cloud con la API de Gmail y
  consentimiento del administrador de su Workspace);
- el alta del canal con `provider = GOOGLE` y su runbook de alta y baja.

No confundir con la versión vieja del repo (hasta mayo de 2026), que leía **una** cuenta de
Gmail con un Gmail Trigger y posteaba a `/process`: ya no existe y no sirve como base (no tenía
canales, llave ni cursor).

## Qué NO hace este repo

- No marca como leído, no mueve ni borra correos del buzón.
- No clasifica ni decide si un correo es ticket nuevo, respuesta o duplicado (eso es la
  bandeja del clasificador, ticket-classification).
- No decide **a quién** ni **cuándo** avisar (eso es ticket-management y
  notification-service). Solo entrega el correo que le piden.

---

## Datos rápidos

| Dato | Valor |
|---|---|
| Contenedor | `til-n8n` (imagen `docker.io/n8nio/n8n`) |
| Puerto | 5679 en el host (solo `127.0.0.1`) → 5678 interno |
| Persistencia | Volumen `ticket-ingestion-light_n8n_data` (SQLite de n8n: workflows, credenciales, cursores) |
| Base de datos externa | Ninguna |
| Redes | `stack` (propia) y `ticket-management-network` (compartida, externa) |
| Ruta en Rocky | `~/stack_ticket/ticket-ingestion-light` |
| Interfaz web | `http://localhost:5679` con el túnel SSH, desde cualquier máquina de la oficina (infra-monitoring `10_redes_y_acceso_lan.md` §7) |

---

## Historia (para no confundirse)

- Hasta mayo de 2026 el repo leía **Gmail** con un Gmail Trigger y posteaba a
  `langchain-agent:8001/process`. Eso ya no existe.
- Hasta el 2026-10-16 un solo workflow leía el buzón compartido
  `sharedservicesmx@hitss.com` sin cursor. Se retiró: ese buzón es hoy el canal
  "Shared Services" del cliente inicial.
- En n8n de Rocky aún quedan workflows viejos (`SS-`, `SSS-`, `POC2 -`, todos
  **inactivos**). No deben activarse: postean a `/process`, que ya no existe.

---

*ticket-ingestion-light · Ingestion for Dummies · Octubre 2026*
