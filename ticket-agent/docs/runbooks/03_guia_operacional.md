# 03 — Guía Operacional

Operación día a día del agente RAG: arranque, re-ingesta, mantenimiento y diagnóstico.

---

## Arranque rápido

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario
```

El entorno virtual debe activarse en cada nueva sesión de terminal.

---

## Checklist antes de usar el agente

- [ ] Ollama corriendo: `curl http://localhost:11434/api/tags`
- [ ] Entorno virtual activado: `which python` debe apuntar a `.venv/bin/python`
- [ ] Índice FAISS existe: `ls data/vector_db/faiss_index/`
- [ ] `.env` configurado: `cat .env | grep PROVIDER`

---

## Re-ingesta (documentos actualizados)

Ejecutar cuando se agreguen o modifiquen documentos en los repos hermanos.

```bash
# 1. Sincronizar repos
bash ~/stack_ticket/infra-monitoring/sync-repos.sh

# 2. Activar entorno
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate

# 3. Re-ingestar todo
python -m src.main ingest --source all
```

> Cada `ingest` sobreescribe el índice completo. Siempre usar `--source all`
> para que el índice quede con las tres fuentes activas.

---

## Cambiar proveedor de LLM

### Ollama → OCI GenAI

```bash
# Editar .env
PROVIDER=oci
OCI_COMPARTMENT_ID=<compartment-id>
OCI_SERVICE_ENDPOINT=https://inference.generativeai.us-chicago-1.oci.oraclecloud.com
OCI_LLM_MODEL=meta.llama-3.3-70b-instruct
```

Verificar que `~/.oci/config` exista con credenciales válidas.

### OCI → Ollama

```bash
# Editar .env
PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_LLM_MODEL=llama3.2:3b
```

El índice FAISS no cambia — solo cambia quién genera la respuesta.

---

## Queries de diagnóstico

```bash
# Verificar las tres fuentes responden
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario
python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager
python -m src.main query --query "¿Cómo configuro el workflow?" --rol admin
python -m src.main query --query "¿Cómo está configurado n8n?" --rol soporte

# Verificar aislamiento de rol (usuario NO debe ver runbooks)
python -m src.main query --query "¿Cómo funciona langchain-agent?" --rol usuario
# Respuesta esperada: "No tengo esa información disponible"
```

---

## Correr tests

```bash
source .venv/bin/activate

# Suite completa (requiere modelo ONNX descargado)
pytest tests/

# Sin tests de modelo real (más rápido)
pytest tests/ -k "not onnx_embedder_dims and not onnx_embed_documents"

# Test específico
pytest tests/test_chunking.py -v
pytest tests/test_retriever.py -v
```

---

## Logs

El agente loguea a stdout con nivel configurable en `.env`:

```bash
LOG_LEVEL=DEBUG  # INFO | WARNING | DEBUG
```

Con `DEBUG` se ven las rutas de cada documento cargado y las dimensiones validadas.

---

## Diagnóstico del índice

```bash
# Ver cuántos chunks existen por fuente
cat data/processed/chunks.jsonl | python3 -c "
import sys, json
from collections import Counter
tipos = Counter(json.loads(l)['tipo'] for l in sys.stdin)
print(tipos)
"
```

Salida esperada con índice completo:
```
Counter({'runbooks': ~700, 'usuarios': ~212, 'helpContent': ~207})
```

---

## Troubleshooting

**Vector store no inicializado:**
```bash
python -m src.main ingest --source all
```

**DNS no resuelve en WSL (al hacer git pull o pip install):**
```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```

**Ollama no responde:**
```bash
curl http://localhost:11434/api/tags
# Si falla, verificar que el contenedor ollama esté corriendo en ticket-classification
cd ~/stack_ticket/ticket-classification
podman-compose ps
```

**Modelo llama3.2:3b no disponible en Ollama:**
```bash
curl -X POST http://localhost:11434/api/pull -d '{"name": "llama3.2:3b"}'
```

**Respuestas incorrectas o fuera de contexto:**
1. Verificar que el índice esté actualizado (`ingest --source all`)
2. Verificar que el rol sea el correcto para la pregunta
3. Aumentar `RETRIEVAL_K` en `.env` (default: 3)

---

## Archivos de estado

| Archivo | Descripción |
|---|---|
| `data/vector_db/faiss_index/` | Índice FAISS — borrar y re-ingestar si está corrupto |
| `data/processed/chunks.jsonl` | Cache de chunks para diagnóstico — puede borrarse sin problema |
| `.env` | Configuración local — nunca commitear |
