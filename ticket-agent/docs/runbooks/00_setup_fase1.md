# Setup Fase 1 — ticket-agent

Guía paso a paso para dejar el agente RAG funcionando desde cero.

| Entorno | SO | Ollama |
|---|---|---|
| **Local (dev)** | WSL sobre Windows 10 | Windows host |
| **Pre/Producción** | Linux nativo | Mismo host o servicio |

---

## Prerequisitos

### Local (WSL)
- WSL con Python 3.12+
- Ollama corriendo en Windows: `http://localhost:11434`
- Repos hermanos clonados en `~/stack_ticket/`
- DNS de WSL funcionando (ver Troubleshooting si falla)

### Pre/Producción (Linux nativo)
- Python 3.12+
- Ollama instalado y corriendo: `http://localhost:11434`
- Repos hermanos en el mismo directorio base (ej. `/opt/stack_ticket/`)
- Ajustar rutas `SOURCE_*` en `.env` según la ubicación real

---

## 1. Clonar el repo

```bash
# Local (WSL)
cd ~/stack_ticket
git clone https://<token>@github.com/jjrm2004mx/ticket-agent.git
cd ticket-agent

# Pre/Producción
cd /opt/stack_ticket          # o el path que corresponda
git clone https://github.com/jjrm2004mx/ticket-agent.git
cd ticket-agent
```

---

## 2. Crear entorno virtual e instalar dependencias

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

La primera vez descarga dependencias pesadas (faiss, sentence-transformers, torch).
Puede tardar 5-10 minutos.

> **WSL:** Si falla con `externally-managed-environment`, asegúrate de haber creado el venv primero.

---

## 3. Configurar variables de entorno

```bash
cp .env.example .env
```

Editar `.env` según el entorno:

```bash
# Local (WSL) — rutas relativas desde ~/stack_ticket/ticket-agent/
SOURCE_USUARIOS=../ticket-management/docs/usuarios
SOURCE_RUNBOOKS=../ticket-management/docs/runbooks,...
SOURCE_HELP_TS=../ticket-management/frontend/src/components/help/helpContent.ts
OLLAMA_BASE_URL=http://localhost:11434

# Pre/Producción — ajustar al path real de instalación
SOURCE_USUARIOS=/opt/stack_ticket/ticket-management/docs/usuarios
SOURCE_RUNBOOKS=/opt/stack_ticket/ticket-management/docs/runbooks,...
SOURCE_HELP_TS=/opt/stack_ticket/ticket-management/frontend/src/components/help/helpContent.ts
OLLAMA_BASE_URL=http://localhost:11434
```

Verificar que las rutas existan:

```bash
ls ../ticket-management/docs/usuarios/
ls ../ticket-management/frontend/src/components/help/helpContent.ts
```

---

## 4. Ingestar documentos

```bash
python -m src.main ingest --source all
```

**La primera vez descarga el modelo ONNX `mxbai-embed-large-v1` (~670 MB)** — puede tardar varios minutos.

Salida esperada:
```
[ingest] Fuente: all
[ingest] Cargando documentos...
[ingest] N documentos cargados. Chunkeando...
[ingest] M chunks generados. Embebiendo y persistiendo...
[ingest] Listo. M chunks persistidos en ./data/vector_db/faiss_index/
```

Para ingestar fuentes individuales:
```bash
python -m src.main ingest --source usuarios
python -m src.main ingest --source runbooks
python -m src.main ingest --source help
```

> **Importante:** cada `ingest` **sobreescribe** el índice FAISS completo — no acumula.
> Si ingesta parcial (`--source usuarios`), el índice solo contendrá esa fuente.
> Para tener todas las fuentes activas, siempre terminar con `--source all`.

---

## Procedimiento: re-ingestar cuando hay documentos nuevos

Ejecutar cuando se agreguen o modifiquen archivos en cualquiera de las fuentes de conocimiento.

### Paso 1 — Sincronizar repos hermanos

```bash
# Actualizar todos los repos del ecosistema
bash ~/stack_ticket/infra-monitoring/sync-repos.sh

# O manualmente repo por repo
cd ~/stack_ticket/ticket-management && git pull
cd ~/stack_ticket/ticket-classification && git pull
# ... etc
```

### Paso 2 — Activar el entorno virtual

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate
```

### Paso 3 — Re-ingestar

```bash
# Re-ingestar todo (recomendado para asegurar consistencia)
python -m src.main ingest --source all

# O solo la fuente modificada (más rápido, pero el índice quedará solo con esa fuente)
python -m src.main ingest --source usuarios   # si cambiaron docs/usuarios/
python -m src.main ingest --source runbooks   # si cambiaron docs/runbooks/
python -m src.main ingest --source help       # si cambió helpContent.ts
```

### Paso 4 — Verificar

```bash
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario
```

### ¿Cuándo re-ingestar?

| Evento | Acción |
|---|---|
| Se agregaron MD en `docs/usuarios/` de ticket-management | `ingest --source all` |
| Se modificó `helpContent.ts` en ticket-management | `ingest --source all` |
| Se agregaron runbooks en cualquier repo | `ingest --source all` |
| El índice FAISS quedó corrupto o incompleto | `ingest --source all` |
| Primera vez en una máquina nueva | `ingest --source all` |

---

## 5. Verificar con queries de prueba

```bash
# Usuario final
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario

# Manager
python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager

# Admin
python -m src.main query --query "¿Cómo configuro el workflow de clasificación?" --rol admin

# Soporte técnico (acceso total)
python -m src.main query --query "¿Cómo está configurado n8n en ticket-ingestion-light?" --rol soporte
```

Cada query muestra: respuesta, fuentes, rol usado y latencia. Latencia esperada < 10s con Ollama local.

---

## 6. Correr tests

```bash
pytest tests/
```

Los tests de embeddings reales requieren que el modelo esté descargado (paso 4). Para saltarlos:

```bash
pytest tests/ -k "not onnx_embedder_dims and not onnx_embed_documents"
```

---

## Activar el entorno en sesiones futuras

El virtualenv no persiste entre sesiones de terminal:

```bash
cd ~/stack_ticket/ticket-agent   # o el path correspondiente
source .venv/bin/activate
```

---

## Estado de runbooks por repo

El agente ingesta `docs/runbooks/` de 6 repos (incluido el propio):

| Repo | docs/runbooks/ | Archivos |
|---|---|---|
| **ticket-agent** (propio) | ✅ | 5 MD |
| ticket-ingestion-light | ✅ | 5 MD |
| ticket-management | ✅ | 6 MD |
| ticket-classification | ✅ | 8 MD |
| notification-service | ✅ | 4 MD |
| infra-monitoring | ✅ | 12 MD |

---

## Troubleshooting

**[WSL] DNS no resuelve (`Temporary failure in name resolution`):**
```bash
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf   # evita que WSL lo sobreescriba al reiniciar
```

**[WSL] Error `externally-managed-environment` en pip:**
```bash
python3 -m venv .venv && source .venv/bin/activate
```

**Vector store no inicializado al hacer query:**
```bash
python -m src.main ingest --source all
```

**Conflicto de dependencias LangChain:**
Verificar que `requirements.txt` use `>=0.3,<2.0` en lugar de `==0.3.*`.

**Modelo ONNX no descarga (sin internet en WSL):**
Verificar DNS primero. El modelo se descarga de HuggingFace Hub la primera vez.
