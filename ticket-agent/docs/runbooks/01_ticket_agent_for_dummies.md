# 01 — ticket-agent for Dummies

Guía de introducción para entender qué es el agente, cómo funciona y cómo usarlo.

---

## ¿Qué es ticket-agent?

Es un asistente de preguntas y respuestas sobre el ecosistema de tickets. Le haces una
pregunta en español, él busca en la documentación y te da una respuesta basada en lo que
está documentado.

No inventa — si la información no está en los documentos, lo dice.

---

## ¿Quién lo usa y para qué?

| Rol | Preguntas típicas |
|---|---|
| **usuario** | ¿Cómo creo un ticket? ¿Qué estados puede tener? ¿Cómo adjunto un archivo? |
| **manager** | ¿Cómo apruebo un borrador? ¿Qué permisos tengo? |
| **admin** | ¿Cómo configuro el workflow de clasificación? ¿Cómo gestiono usuarios? |
| **soporte** | ¿Cómo está configurado n8n? ¿Qué hace langchain-agent? ¿Cómo reinicio el stack? |

El rol determina qué documentación consulta — un `usuario` nunca verá runbooks técnicos.

---

## ¿De dónde saca la información?

El agente tiene tres fuentes de conocimiento:

1. **Documentación por rol** — manuales escritos específicamente para cada tipo de usuario
   (`ticket-management/docs/usuarios/`)

2. **Ayuda contextual de pantallas** — el mismo texto de ayuda que aparece en la interfaz
   web del sistema de tickets (`helpContent.ts`)

3. **Runbooks técnicos** — documentación operacional de todos los repos del ecosistema,
   solo accesible con rol `soporte`

---

## ¿Cómo se usa?

Desde la terminal (WSL o Linux):

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate

# Pregunta como usuario final
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario

# Pregunta como manager
python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager

# Pregunta como soporte técnico
python -m src.main query --query "¿Cómo está configurado n8n?" --rol soporte
```

La respuesta incluye el texto generado, las fuentes consultadas y el tiempo de respuesta.

---

## ¿Qué necesita para funcionar?

- **Ollama corriendo** con el modelo `llama3.2:3b` (o OCI GenAI configurado)
- **El índice FAISS** generado previamente con `ingest --source all`
- **El entorno virtual** activado (`source .venv/bin/activate`)

Si el índice no existe o está desactualizado, ver `00_setup_fase1.md`.

---

## ¿Por qué no responde bien?

| Síntoma | Causa probable |
|---|---|
| "No tengo esa información disponible" | El documento no está ingestado o el rol no tiene acceso |
| Respuesta muy genérica | La pregunta es ambigua — ser más específico |
| Error de vector store | El índice no existe — ejecutar `ingest --source all` |
| Respuesta lenta (>30s) | Ollama tardando — normal en primera carga del modelo |

---

## ¿Qué NO hace (todavía)?

- No tiene interfaz web (Fase 4)
- No accede a internet para buscar información externa
- No modifica tickets ni interactúa con el sistema
- No aprende de las preguntas — la base de conocimiento se actualiza solo re-ingestando
