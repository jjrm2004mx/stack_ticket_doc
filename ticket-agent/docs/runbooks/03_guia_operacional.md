# 03 — Guía Operacional

Operación día a día del agente RAG: arranque, re-ingesta, mantenimiento y diagnóstico.

---

## Arranque del contenedor (modo producción)

```bash
# Opción recomendada — levanta todo el ecosistema en orden
bash ~/stack_ticket/infra-monitoring/startup.sh up

# Solo este servicio
cd ~/stack_ticket/ticket-agent
podman-compose up -d

# Verificar arranque (esperar ~30s si es primera vez)
podman logs -f ticket-agent

# Health check
curl http://localhost:8002/api/v1/health
# Respuesta esperada: {"status":"UP","application":"ticket-agent"}
```

Secuencia normal en los logs (con `VECTOR_STORE=oracle`):
```
[entrypoint] VECTOR_STORE=oracle — arrancando API...
Iniciando ticket-agent API...
Cargando embeddings: mixedbread-ai/mxbai-embed-large-v1
Oracle: conectado a ticketagent_tp
Oracle: 1251 chunks disponibles en KB_CHUNKS
Application startup complete.
```

### Apagar

```bash
cd ~/stack_ticket/ticket-agent
podman-compose down
```

---

## Arranque rápido (modo desarrollo — CLI directo)

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario
```

El entorno virtual debe activarse en cada nueva sesión de terminal.

---

## Checklist antes de usar el agente

**Con `VECTOR_STORE=oracle` (configuración actual):**
- [ ] Entorno virtual activado: `which python` debe apuntar a `.venv/bin/python`
- [ ] Wallet Oracle accesible: `ls ~/.oci/wallet_ticketagent/tnsnames.ora`
- [ ] KB_CHUNKS poblado: `python3 -c "from src.config import settings; import oracledb; ..."`
  (ver sección "Diagnóstico Oracle" más abajo)
- [ ] `.env` configurado: `grep -E "VECTOR_STORE|PROVIDER|HYBRID" .env`

**Con `VECTOR_STORE=faiss` (modo desarrollo ligero):**
- [ ] Ollama corriendo: `curl http://localhost:11434/api/tags`
- [ ] Índice FAISS existe: `ls data/vector_db/faiss_index/`

---

## Re-ingesta — Documentos (KB vectorial)

Ejecutar cuando se agreguen o modifiquen documentos en los repos hermanos.

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate

# Re-ingestar todos los documentos (usuarios + runbooks + helpContent)
python -m src.main ingest --source all
```

> Con `CHUNKING_STRATEGY=parent_child` (configuración actual) tarda ~35-45 min.
> El proceso hace DELETE + INSERT en una sola transacción — si muere antes del COMMIT,
> KB_CHUNKS queda intacta (rollback automático).

Logs esperados al finalizar:
```
Oracle: 1557 chunks persistidos en KB_CHUNKS
[ingest] Listo. 1557 chunks persistidos en ...
```

---

## Re-ingesta — Property Graph (grafo de arquitectura)

Ejecutar cuando cambie `docs/arquitectura/ecosistema.mmd` o se agreguen nuevos servicios.

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate

python -m src.main ingest --source graph
```

Logs esperados:
```
[ingest graph] Parseando ./docs/arquitectura/ecosistema.mmd...
[ingest graph] 18 nodos, 27 aristas encontrados.
Oracle: 18 nodos + 27 aristas persistidos en KB_GRAPH
[ingest graph] Listo.
```

> Requiere que la migración `db/migrations/oracle/V2__add_graph_tables.sql` ya haya sido
> aplicada en Database Actions (una sola vez). Ver `ORACLE_MIGRACION.md` sección 9.11.

---

## Cambiar proveedor de LLM

### Ollama → OCI GenAI

```bash
# Editar .env
PROVIDER=oci
OCI_COMPARTMENT_ID=<compartment-id>
OCI_SERVICE_ENDPOINT=https://inference.generativeai.us-chicago-1.oci.oraclecloud.com
OCI_MODEL=cohere.command-r-plus-08-2024
```

Verificar que `~/.oci/config` exista con credenciales válidas.

### OCI → Ollama

```bash
# Editar .env
PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_LLM_MODEL=llama3.2:3b
```

El índice Oracle no cambia — solo cambia quién genera la respuesta.

---

## Queries de diagnóstico

La salida incluye respuesta, fuentes, rol, confianza (0.0–1.0) y latencia:
```
Rol: usuario | Confianza: 0.8 | Latencia: 12.1s
```

```bash
# Verificar las tres fuentes y cuatro roles
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario
python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager
python -m src.main query --query "¿Cómo configuro el workflow?" --rol admin
python -m src.main query --query "¿Cómo está configurado n8n?" --rol soporte

# Verificar aislamiento de rol (usuario NO debe ver runbooks técnicos)
python -m src.main query --query "¿Cómo funciona langchain-agent?" --rol usuario
# Respuesta esperada: "No tengo esa información disponible"

# Verificar Property Graph (keywords de arquitectura activan graph_traverse)
python -m src.main query --query "¿Qué servicios usa ticket_agent?" --rol soporte
python -m src.main query --query "¿Qué impacto tiene si cae classifier_redis?" --rol soporte
```

Con Hybrid Search activo, los logs muestran:
```
Oracle hybrid RRF: 3 docs (vec=20, bm25=20, α=0.7, rol=usuario)
```

Con Property Graph activo, los logs muestran:
```
Graph traverse: N relaciones añadidas al contexto
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

## Diagnóstico Oracle

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate

python3 - << 'EOF'
from src.config import settings
import oracledb

conn = oracledb.connect(
    user=settings.ORACLE_USER, password=settings.ORACLE_PASSWORD,
    dsn=settings.ORACLE_DSN, config_dir=settings.ORACLE_CONFIG_DIR,
    wallet_location=settings.ORACLE_WALLET_DIR,
    wallet_password=settings.ORACLE_WALLET_PASSWORD,
)
cur = conn.cursor()

cur.execute("SELECT COUNT(*) FROM KB_CHUNKS WHERE IS_PARENT = 0")
print(f"KB_CHUNKS (hijos): {cur.fetchone()[0]}")

cur.execute("SELECT COUNT(*) FROM KB_CHUNKS WHERE IS_PARENT = 1")
print(f"KB_CHUNKS (padres): {cur.fetchone()[0]}")

cur.execute("SELECT SOURCE_FILE, CHUNK_COUNT, STATUS FROM KB_INGEST_LOG ORDER BY INGEST_AT DESC FETCH FIRST 3 ROWS ONLY")
for r in cur.fetchall():
    print(f"INGEST_LOG: {r[0]} | {r[1]} chunks | {r[2]}")

try:
    cur.execute("SELECT COUNT(*) FROM KB_GRAPH_NODES")
    print(f"KB_GRAPH_NODES: {cur.fetchone()[0]} nodos")
    cur.execute("SELECT COUNT(*) FROM KB_GRAPH_EDGES")
    print(f"KB_GRAPH_EDGES: {cur.fetchone()[0]} aristas")
except Exception:
    print("KB_GRAPH_NODES/EDGES: no creadas aún (ejecutar V2__add_graph_tables.sql)")

cur.execute("SELECT INDEX_NAME, STATUS FROM USER_INDEXES WHERE TABLE_NAME = 'KB_CHUNKS'")
for r in cur.fetchall():
    print(f"INDEX: {r[0]} — {r[1]}")
conn.close()
EOF
```

Salida esperada:
```
KB_CHUNKS (hijos): 1251
KB_CHUNKS (padres): 306
INGEST_LOG: batch_ingest | 1557 chunks | OK
KB_GRAPH_NODES: 18 nodos
KB_GRAPH_EDGES: 27 aristas
INDEX: IDX_KB_EMBEDDING — VALID
INDEX: IDX_KB_CONTENT_TEXT — VALID
```

---

## Troubleshooting

**KB_CHUNKS vacío o desactualizado:**
```bash
python -m src.main ingest --source all
```

**Property Graph vacío (`KB_GRAPH_NODES: 0 nodos`):**
```bash
python -m src.main ingest --source graph
```

**`IDX_KB_CONTENT_TEXT` en estado FAILED (Hybrid Search falla):**
```bash
# Ejecutar en Database Actions como AGENTE:
DROP INDEX IDX_KB_CONTENT_TEXT;
CREATE INDEX IDX_KB_CONTENT_TEXT ON KB_CHUNKS(CONTENT)
  INDEXTYPE IS CTXSYS.CONTEXT PARAMETERS ('SYNC (ON COMMIT)');
```

**`Permission denied` al conectar Oracle (wallet inaccesible como usuario no-root):**
```bash
sudo chmod o+rx /root
sudo chmod o+rx /root/.oci
sudo chmod o+rx /root/.oci/wallet_ticketagent
sudo chmod o+r /root/.oci/wallet_ticketagent/*
```
Ocurre cuando el wallet está en `/root/.oci/` y el proceso corre como usuario normal (`jjrm`).
Solo se necesita ejecutar una vez por sesión WSL / reinicio.

**DNS no resuelve en WSL (al hacer git pull o pip install):**
```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```

**Ollama no responde:**
```bash
curl http://localhost:11434/api/tags
# Si falla, verificar que el contenedor ollama esté corriendo en ticket-classification
cd ~/stack_ticket/ticket-classification && podman-compose ps
```

**Respuestas incompletas (falta información):**
1. Verificar que el índice esté actualizado (`ingest --source all`)
2. Verificar que el rol sea el correcto para la pregunta
3. Con `CHUNKING_STRATEGY=parent_child`, usar `RETRIEVAL_K=6` mínimo

---

## Archivos y recursos clave

| Recurso | Descripción |
|---|---|
| `docs/arquitectura/ecosistema.mmd` | Fuente del Property Graph — editar aquí y re-ingestar con `--source graph` |
| `db/migrations/oracle/V1__init_kb.sql` | Schema base KB_CHUNKS — aplicado manualmente en Fase 2 |
| `db/migrations/oracle/V2__add_graph_tables.sql` | Tablas del Property Graph — aplicar una sola vez en Database Actions |
| `data/processed/chunks.jsonl` | Cache de chunks para diagnóstico — puede borrarse sin problema |
| `.env` | Configuración local — nunca commitear |
