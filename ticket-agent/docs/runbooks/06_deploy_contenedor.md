# 06 — Deploy en contenedor (Podman)

Guía de primer deploy y deploys subsecuentes del servicio `ticket-agent` en Podman.
Aplica tanto a entorno local WSL como a servidor Unix nativo.

---

## Requisitos previos

| Requisito | Verificación |
|---|---|
| Podman + podman-compose instalados | `podman --version && podman-compose --version` |
| Red del ecosistema creada | `podman network ls \| grep ticket-management-network` |
| Índice ingestado en el host | `ls data/vector_db/faiss_index/index.faiss` (solo si `VECTOR_STORE=faiss`) |
| `.env` configurado | `cp .env.example .env && nano .env` |
| Wallet OCI disponible | `ls ~/.oci/wallet_ticketagent/` (solo si `VECTOR_STORE=oracle`) |

---

## Primer deploy — checklist

### 1. Red del ecosistema

```bash
# Verificar si existe
podman network ls | grep ticket-management-network

# Crear si no existe (la crea infra-monitoring/startup.sh en el stack completo)
podman network create --subnet 10.89.1.0/24 ticket-management-network
```

### 2. Configurar `.env`

```bash
cp .env.example .env
```

Variables críticas a editar:

| Variable | Valor | Nota |
|---|---|---|
| `PROVIDER` | `oci` o `ollama` | |
| `VECTOR_STORE` | `oracle` o `faiss` | |
| `OCI_CONFIG_FILE` | `/root/.oci/config.container` | Ruta **dentro del contenedor** — ver sección OCI abajo |
| `ORACLE_CONFIG_DIR` | `/root/.oci/wallet_ticketagent` | Ruta **dentro del contenedor** |
| `ORACLE_WALLET_DIR` | `/root/.oci/wallet_ticketagent` | Ruta **dentro del contenedor** |
| `ORACLE_PASSWORD` | valor real | |
| `ORACLE_WALLET_PASSWORD` | valor real | |

> **Por qué `/root/.oci/...`:** el `docker-compose.yml` monta `${HOME}/.oci` del host
> en `/root/.oci` dentro del contenedor. Las variables de entorno son leídas por el proceso
> Python dentro del contenedor, por lo que deben usar la ruta interna, no la del host.

### 3. Config OCI para el contenedor

El archivo `~/.oci/config` del host tiene `key_file` apuntando a la ruta del usuario del host
(ej. `/home/ubuntu/...`). Dentro del contenedor la ruta es `/root/.oci/...`.
Se necesita una copia con la ruta corregida:

```bash
# Crear config específico para el contenedor (no modifica el original)
sed 's|key_file=/home/<usuario>/.oci/|key_file=/root/.oci/|g' \
  ~/.oci/config > ~/.oci/config.container

# Verificar
grep key_file ~/.oci/config.container
# Debe mostrar: key_file=/root/.oci/oci_api_key.pem

# Asegurarse de que .env apunta a este archivo
grep OCI_CONFIG_FILE .env
# Debe mostrar: OCI_CONFIG_FILE=/root/.oci/config.container
```

### 4. Build de la imagen

```bash
# Bajar el contenedor si está corriendo
podman-compose down

# Reconstruir la imagen sin caché
podman build --no-cache -t ticket-agent .

# Levantar con la nueva imagen
podman-compose up -d
```

> Usar siempre `podman build --no-cache` directo seguido de `podman-compose up -d`.
> Evitar `podman-compose build --no-cache` — puede usar caché interna de compose
> e ignorar cambios recientes en `src/`.

### 5. Poblar volúmenes (solo primera vez con FAISS)

Si `VECTOR_STORE=faiss`, el volumen `faiss_data` nace vacío. Poblar antes del primer `up`:

```bash
# Ingestar primero en el host (fuera del contenedor)
python -m src.main ingest --source all

# Copiar al volumen named
podman run --rm \
  -v faiss_data:/target \
  -v "$(pwd)/data/vector_db/faiss_index":/source:ro \
  alpine sh -c "cp -r /source/. /target/ && echo 'OK'"

podman run --rm \
  -v agent_processed:/target \
  -v "$(pwd)/data/processed":/source:ro \
  alpine sh -c "cp -r /source/. /target/ && echo 'OK'"
```

Si `VECTOR_STORE=oracle`, este paso no aplica — los datos están en Oracle.

### 6. Verificar arranque

```bash
podman logs -f ticket-agent
```

Secuencia esperada en los logs:

```
[entrypoint] VECTOR_STORE=oracle — arrancando API...
Iniciando ticket-agent API...
Cargando embeddings: mixedbread-ai/mxbai-embed-large-v1
Oracle: conectado a ticketagent_tp
Oracle: 1233 chunks disponibles en KB_CHUNKS
LLM: OCI GenAI (cohere.command-r-plus-08-2024)
Application startup complete.
```

### 7. Health check

```bash
curl http://localhost:8002/api/v1/health
# {"status":"UP","application":"ticket-agent"}

curl http://localhost:8002/api/v1/info
```

---

## Deploys subsecuentes (tras cambios de código)

```bash
cd ~/stack_ticket/ticket-agent
bash sync-repos.sh
podman-compose down
podman build --no-cache -t ticket-agent .
podman-compose up -d
podman logs -f ticket-agent
```

---

## Consideraciones para servidor Unix nativo

Al migrar de WSL a un servidor Linux real, ajustar:

| Aspecto | WSL | Servidor nativo |
|---|---|---|
| Usuario del host | `jjrm` | `ubuntu`, `ec2-user`, etc. |
| Path `key_file` en `config.container` | `/home/jjrm/.oci/...` → genera `/root/.oci/...` | `/home/<usuario>/.oci/...` → genera `/root/.oci/...` |
| `ORACLE_CONFIG_DIR` en `.env` | `/root/.oci/wallet_ticketagent` (ruta contenedor) | igual — el mount siempre llega a `/root/.oci/` |
| Ollama | `host.containers.internal:11434` | mismo (si Ollama corre en el host) |
| Red | crear manual o via `infra-monitoring/startup.sh` | igual |
| `N8N_SECURE_COOKIE` | `false` (HTTP local) | `true` si hay HTTPS |

### Checklist servidor

- [ ] Ajustar paths de usuario en `.env` y `~/.oci/config.container`
- [ ] Verificar que el wallet Oracle está disponible en el servidor
- [ ] Crear red `ticket-management-network` antes del primer `up`
- [ ] Confirmar que Ollama (si aplica) está corriendo y accesible
- [ ] Primer arranque con `podman-compose up -d --build`
- [ ] Verificar `curl http://localhost:8002/api/v1/health`

---

## Diagnóstico rápido

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Índice FAISS no encontrado` | `VECTOR_STORE=faiss` y volumen vacío | Ingestar y poblar volumen |
| `Oracle: error de conexión` o `tnsnames.ora is missing` | `ORACLE_CONFIG_DIR`/`ORACLE_WALLET_DIR` con ruta del host en lugar de la del contenedor | En `.env` usar `/root/.oci/wallet_ticketagent` (no `/home/jjrm/...`) |
| `key_file invalid path` | `config.container` no creado o con paths del host | Crear `~/.oci/config.container` con rutas `/root/.oci/` |
| `Application startup failed` en bucle | Ver logs completos — error en lifespan | `podman logs ticket-agent 2>&1 \| head -60` |
| Contenedor no aparece en `podman ps` | La red `ticket-management-network` no existe | `podman network create --subnet 10.89.1.0/24 ticket-management-network` |
| `torch` descarga paquetes CUDA en el build | `sentence-transformers` arrastra torch+CUDA | El `Dockerfile` ya pre-instala torch CPU-only — no modificar el orden |
