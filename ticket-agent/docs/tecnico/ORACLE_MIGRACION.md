# Migración FAISS → Oracle 23ai

Documento técnico de referencia para la Fase 2 de ticket-agent.
Explica cómo funciona el vector store hoy (FAISS) y cómo funcionará con Oracle 23ai,
con el script SQL de inicialización y las tareas de implementación.

Última actualización: 2026-05-29 — V3 tablas especializadas + routing multi-tabla + `/api/v1/diagnose` validados en WSL

---

## Contenido

1. [Arquitectura actual — FAISS](#1-arquitectura-actual--faiss)
2. [Arquitectura objetivo — Oracle 23ai](#2-arquitectura-objetivo--oracle-23ai)
3. [Diferencias lado a lado](#3-diferencias-lado-a-lado)
4. [Esquema de datos](#4-esquema-de-datos)
5. [Provisionamiento en OCI — Autonomous AI Database](#5-provisionamiento-en-oci--autonomous-ai-database)
6. [Script SQL completo](#6-script-sql-completo)
7. [Variables de entorno adicionales](#7-variables-de-entorno-adicionales)
8. [Tareas de implementación](#8-tareas-de-implementación)
9. [Operaciones — Ingesta y monitoreo](#9-operaciones--ingesta-y-monitoreo)

---

## 1. Arquitectura actual — FAISS

### 1.1 Pipeline de ingesta

```
INGEST
──────
3 fuentes de documentos:
  docs/usuarios/{rol}/*.md
  helpContent.ts
  */docs/runbooks/*.md
          │
          ▼
  loader.py
  asigna metadata: rol · tipo · fuente · pantalla
          │
          ▼
  DocumentChunker
  RecursiveCharacterTextSplitter
  chunk_size=512 · chunk_overlap=100
          │
          ▼
  ONNXEmbedderAdapter                ← mxbai-embed-large-v1, corre en Python/CPU
  embed_documents()
          │
          ▼
  Vectores [1024 floats] ────────────┐
  Metadata (dict Python) ────────────┤
          │                          │
          ▼                          ▼
  FAISS.from_texts()            faiss_index/
  (índice en RAM)               ├── index.faiss   ← vectores binarios
                                └── index.pkl     ← metadata (pickle Python)
```

### 1.2 Pipeline de recuperación

```
QUERY
──────
"¿Cómo creo un ticket?" + rol="usuario"
          │
          ▼
  ONNXEmbedderAdapter
  embed_query() → query_vector [1024 floats]
          │
          ▼
  FAISS.similarity_search(query_vector, k=60)   ← busca los 60 más cercanos en RAM
  (fetch_k = RETRIEVAL_K × _FETCH_MULTIPLIER)   ← sobre-fetch porque el filtro
                                                    de rol es post-retrieval
          │
          ▼
  Python filtra en memoria:
  [d for d in 60_results if rol in d.metadata["rol"]]
          │
          ▼
  top 3 LangDocs  →  contexto  →  LLM  →  respuesta
```

### 1.3 Puntos débiles del enfoque FAISS

```
PROBLEMA                     CAUSA                          CONSECUENCIA
──────────────────────────────────────────────────────────────────────────
Filtro por rol ineficiente   Se hace en Python post-search  Fetch de 20×K candidatos
                             con metadata["rol"]            aunque el índice tenga miles
                                                            de chunks irrelevantes

Re-ingesta total             FAISS.from_texts() reemplaza   Un chunk nuevo = rebuild
                             el índice entero               completo del índice

Sin transacciones            Archivos binarios en disco     Corrupción = pérdida total
                             (index.faiss + index.pkl)      sin recuperación automática

RAM = tamaño del índice      El índice entero vive en el    Escalar a >100K chunks
                             heap del proceso Python        presiona la memoria del pod

Sin búsqueda híbrida         Solo similitud vectorial       No puede combinar
                             (coseno)                       keyword + vector
```

---

## 2. Arquitectura objetivo — Oracle 23ai

### 2.1 Pipeline de ingesta

```
INGEST
──────
3 fuentes de documentos:
  docs/usuarios/{rol}/*.md
  helpContent.ts
  */docs/runbooks/*.md
          │
          ▼
  loader.py + DocumentChunker          ← sin cambios
          │
          ▼
  ONNXEmbedderAdapter                  ← mismo modelo, sin cambios
  embed_documents() → [1024 floats]
          │
          ▼
  INSERT INTO KB_CHUNKS (               ← SQL transaccional
    chunk_id, content, embedding,
    source_file, tipo, fuente,
    pantalla, rol_json, is_parent
  )
          │
          ▼
  Oracle actualiza índice HNSW          ← incremental, sin rebuild
  Oracle actualiza índice Oracle Text   ← SYNC ON COMMIT
          │
          ▼
  INSERT INTO KB_INGEST_LOG            ← trazabilidad de cada corrida
```

### 2.2 Pipeline de recuperación

```
QUERY
──────
"¿Cómo creo un ticket?" + rol="usuario"
          │
          ▼
  ONNXEmbedderAdapter
  embed_query() → query_vector [1024 floats]   ← sin cambios
          │
          ▼
  SELECT chunk_id, content, source_file,
         tipo, fuente, pantalla,
         VECTOR_DISTANCE(embedding, :qvec, COSINE) AS score
  FROM   KB_CHUNKS
  WHERE  is_parent = 0
    AND  JSON_EXISTS(rol_json, '$[*]?(@ == "usuario")')   ← filtro DENTRO del SQL
  ORDER  BY score
  FETCH  FIRST 3 ROWS ONLY;
          │
          ▼
  Oracle aplica HNSW sobre los chunks   ← solo evalúa vectores que ya
  que cumplen el WHERE                    pasan el filtro de rol
          │
          ▼
  top 3 filas  →  contexto  →  LLM  →  respuesta
```

### 2.3 Hybrid Search (Fase 2 — activable)

```
QUERY CON HYBRID SEARCH
────────────────────────
query_vector [1024 floats]  +  "¿Cómo creo un ticket?" (texto)
          │                              │
          ▼                              ▼
  VECTOR_DISTANCE()              CONTAINS(content,
  (índice HNSW)                  'creo NEAR ticket')
  score_vector                   (índice Oracle Text · BM25)
  score_bm25
          │                              │
          └──────────────┬───────────────┘
                         ▼
               score_final = α × score_vector + (1-α) × score_bm25
                         │
                         ▼
               top 3 chunks  →  LLM  →  respuesta
```

### 2.4 Parent-Child chunking (Fase 2) ✅ VALIDADO (2026-05-24)

```
INGEST CON PARENT-CHILD
────────────────────────
Documento fuente
          │
          ▼
  ParentChildChunker
  ├── Chunk padre (~2048 tokens)   is_parent=1   ← contexto completo para el LLM
  │   parent_id = NULL
  └── Chunks hijos (256-512 tok)   is_parent=0   ← se indexan y se recuperan
      parent_id → chunk_id del padre
          │
          ▼
  KB_CHUNKS: se embeben TODOS los chunks (padres e hijos)
  El padre recibe embedding pero V_KB_LEAF_CHUNKS (IS_PARENT=0) lo excluye del retrieval
  Resultado: 1557 chunks totales (306 padres + 1251 hijos) ✅


RETRIEVAL CON PARENT-CHILD
────────────────────────────
  Búsqueda vectorial sobre HIJOS (KB_CHUNKS WHERE IS_PARENT=0)
          │
          ▼
  top K chunks hijos recuperados
          │
          ▼
  LEFT JOIN KB_CHUNKS p ON p.CHUNK_ID = c.PARENT_ID
  COALESCE(p.CONTENT, c.CONTENT)   ← padre si existe, hijo si es chunking flat
          │
          ▼
  Contexto = texto del PADRE (~2048 tok)  →  LLM  →  respuesta más completa
```

> **Nota de implementación:** se optó por embeber padres e hijos por simplicidad
> (evita hacer EMBEDDING nullable en Oracle). Los padres tienen embedding pero
> el WHERE IS_PARENT=0 los excluye de cualquier búsqueda vectorial.

---

## 3. Diferencias lado a lado

| Aspecto | FAISS (Fase 1) | Oracle 26ai (Fase 2) |
|---|---|---|
| **Almacenamiento** | `index.faiss` + `index.pkl` en disco | Tablas SQL en datafiles Oracle |
| **RAM requerida** | Índice entero en el heap Python | Solo índice HNSW en buffer Oracle |
| **Filtro por rol** | Post-retrieval en Python (fetch 20×K) | Pre-retrieval en `WHERE` SQL |
| **Ingest incremental** | No — rebuild completo | Sí — `INSERT` fila por fila |
| **Recovery ante fallo** | Corrupción del `.pkl` = pérdida total | WAL / redo log — recuperación automática |
| **Búsqueda híbrida** | No (solo similitud coseno) | Sí — vector + BM25 (Oracle Text) |
| **Concurrencia** | No seguro | ACID completo |
| **Parent-Child chunking** | No soportado nativamente | FK `parent_id → chunk_id` |
| **Trazabilidad ingest** | No | `KB_INGEST_LOG` por corrida |
| **Swap en código** | — | 1 línea en `main.py` (`VectorStoreFactory`) |
| **Lo que NO cambia** | — | Embedder ONNX · modelo · 1024 dims · chunker · grafo LangGraph |

---

## 4. Esquema de datos

### Modelo de datos actual (FAISS)

Cada chunk almacenado en el índice FAISS contiene esta metadata:

```
Chunk (Python dataclass)
├── id          : str (UUID)
├── content     : str
├── source_file : str   ← path al archivo fuente
├── rol         : List[str]   ← ["usuario", "manager", "admin", "soporte"]
├── tipo        : str   ← "usuarios" | "runbooks" | "helpContent"
├── fuente      : str   ← identificador de la fuente
└── pantalla    : Optional[str]   ← "newTicket" | "draftReview" | etc.

+ vector [1024 floats]   ← calculado por ONNXEmbedderAdapter, guardado en index.faiss
```

### Modelo de datos Oracle 23ai

La misma información se mapea a columnas SQL con tipos nativos de Oracle 23ai:

```
KB_CHUNKS (tabla)
├── CHUNK_ID    VARCHAR2(36)        ← UUID — PRIMARY KEY
├── PARENT_ID   VARCHAR2(36)        ← FK a CHUNK_ID (NULL en Fase 1)
├── CONTENT     CLOB                ← texto del chunk (indexado por Oracle Text)
├── EMBEDDING   VECTOR(1024,FLOAT32)← vector — mismo modelo ONNX
├── SOURCE_FILE VARCHAR2(512)       ← path al archivo fuente
├── TIPO        VARCHAR2(50)        ← 'usuarios' | 'runbooks' | 'helpContent'
├── FUENTE      VARCHAR2(100)
├── PANTALLA    VARCHAR2(100)       ← NULL si no es helpContent
├── ROL_JSON    JSON                ← ["usuario","manager"] — filtrable con JSON_EXISTS
├── IS_PARENT   NUMBER(1,0)         ← 0=chunk hoja  1=chunk padre (Fase 2)
└── CREATED_AT  TIMESTAMP

KB_INGEST_LOG (tabla)
├── LOG_ID      NUMBER IDENTITY     ← PRIMARY KEY autoincremental
├── SOURCE_FILE VARCHAR2(512)
├── CHUNK_COUNT NUMBER(6)
├── TIPO        VARCHAR2(50)
├── INGEST_AT   TIMESTAMP
├── STATUS      VARCHAR2(20)        ← 'OK' | 'ERROR'
└── ERROR_MSG   VARCHAR2(2000)

V_KB_LEAF_CHUNKS (vista)
└── KB_CHUNKS WHERE IS_PARENT = 0   ← usada por el adapter para retrieval
```

---

## 5. Provisionamiento en OCI — Autonomous AI Database

Esta sección documenta cómo crear la base de datos Oracle 23ai usando la cuenta
OCI Always Free. No requiere las mismas credenciales que OCI GenAI — la autenticación
de la BD usa un wallet independiente.

### 5.1 Qué servicio seleccionar en la consola OCI

```
Oracle Cloud Console → Oracle Database → Autonomous AI Database
                                         ↑
                                         ESTA opción
                                         (no Exadata, no Base Database Service,
                                          no Dedicated Infrastructure)
```

"Autonomous AI Database" es el nombre actual de Oracle Autonomous Database 23ai.
Incluye Vector Search, HNSW indexes y Oracle Text sin configuración adicional.

### 5.2 Campos del formulario de creación

```
Pantalla: Create Autonomous AI Database Serverless
──────────────────────────────────────────────────

Display name    : ticket-agent-kb
                  (nombre visible en la consola — sin restricciones)

Database name   : TICKETAGENT
                  (aparece en el DSN de conexión — solo alfanumérico, máx 14 chars)

Compartment     : ticket-sys (root)
                  (dejar el que ya viene seleccionado)

Workload type   : Transaction Processing   ← seleccionar esta
                  ┌─────────────────────────────────────────────────────────┐
                  │ Lakehouse         → analytics y data warehouse (NO)     │
                  │ Transaction Proc. → OLTP, APIs, queries cortas  (SÍ) ✅ │
                  │ JSON              → solo apps JSON/REST          (NO)   │
                  │ APEX              → desarrollo low-code           (NO)  │
                  └─────────────────────────────────────────────────────────┘
                  Razón: el RAG hace queries vectoriales cortas e inserts
                  de ingest — es un workload transaccional, no analítico.

Always Free     : ON  ← activar el toggle (viene apagado por defecto)
                  Incluye: 1 OCPU · 20 GB almacenamiento · sin costo

Database version: 26ai  ← versión disponible en OCI (2026)
                  Incluye todas las capacidades de 23ai + mejoras de Vector Search
                  VECTOR(1024, FLOAT32) y CREATE VECTOR INDEX HNSW disponibles ✅
```

### 5.3 Contraseña del usuario ADMIN

Al hacer scroll aparece el campo de contraseña del usuario `ADMIN`:

```
Requisitos Oracle:
  ├── Mínimo 12 caracteres
  ├── Al menos 1 mayúscula
  ├── Al menos 1 número
  └── Al menos 1 carácter especial (# _ -)

Ejemplo válido: TicketAgent#2026

Guarda esta contraseña — se necesita para crear el usuario del agente
y para conectar desde Python si no se crea un usuario separado.
```

### 5.4 Network access y autenticación

```
Network access  : Secure access from everywhere   ✅
mTLS            : Required                        ← Always Free no permite
                                                     desactivar mTLS sin ACL
                  "Configure an ACL or private endpoint to enable TLS auth"
                  La IP de WSL es dinámica → ACL no es práctica
                  → Solución: usar wallet (mTLS con certificados)
```

> **Nota:** Durante la creación el toggle de mTLS aparece como desactivable,
> pero en la instancia creada queda como `Required` en el tier Always Free.
> Desactivarlo requiere una ACL con IPs fijas, incompatible con la IP dinámica de WSL.

### 5.5 Descargar el wallet

```
Consola OCI → tu instancia TICKETAGENT
  → botón "Database connection"
    → sección "Download client credentials (Wallet)"
      → Wallet type: Instance Wallet
      → clic "Download wallet"
      → asignar una wallet password (guárdala — se necesita en Python)
      → descarga: Wallet_TICKETAGENT.zip
```

Descomprimir en WSL:

```bash
mkdir -p ~/.oci/wallet_ticketagent
cp /mnt/c/Users/JJRM/Downloads/Wallet_TICKETAGENT.zip ~/.oci/wallet_ticketagent/
cd ~/.oci/wallet_ticketagent && unzip Wallet_TICKETAGENT.zip
# Archivos resultantes:
# cwallet.sso  ewallet.p12  keystore.jks  ojdbc.properties
# sqlnet.ora   tnsnames.ora  truststore.jks
```

El DSN a usar está en `tnsnames.ora` — usar el perfil `ticketagent_tp`.

### 5.6 Crear usuario del agente (recomendado — no usar ADMIN)

Oracle advierte explícitamente que ADMIN no debe usarse en desarrollo normal.
El usuario `AGENTE` se crea desde la interfaz **Database Actions → Usuarios**
(accesible desde la consola OCI sin instalar nada).

```
Consola OCI → tu instancia → "Database actions" → Administration → Database Users
  → botón "Create User"
```

#### Campos del formulario "Crear usuario"

```
Pestaña: Usuario
────────────────────────────────────────────────────────────────────
Nombre de usuario     : AGENTE
Contraseña            : (contraseña fuerte — mín 12 chars, 1 mayúscula,
                         1 número, 1 especial como # _ -)
Cuota en tablespace   : Unlimited
                        (500M es suficiente hoy pero Unlimited evita
                         errores inesperados al crecer el índice HNSW)

Contraseña caducada   : OFF   (no forzar cambio en el primer login)
La cuenta bloqueada   : OFF

REST, API GraphQL y
acceso web            : ON    (necesario para acceder a Database Actions
                               como usuario AGENTE desde el navegador)

Gráfico               : OFF   (visualización de grafos — no aplica)

OML                   : OFF   (Oracle Machine Learning — no aplica)
                        El ML de este proyecto ocurre FUERA de Oracle:
                        · Embeddings  → ONNX en Python/CPU (externo)
                        · LLM         → Ollama / OCI GenAI (externo)
                        · Vector Search HNSW → feature nativa del motor,
                          no requiere OML

Espacial              : OFF   (Oracle Spatial — no aplica)


Pestaña: Roles Otorgados
────────────────────────────────────────────────────────────────────
☑ CONNECT    ← permite abrir sesiones de base de datos
☑ RESOURCE   ← permite crear tablas, índices y vistas propias
```

El usuario `AGENTE` puede crear sus propias tablas e índices sin acceso
a ningún objeto del sistema Oracle.

### 5.7 Ejecutar el script SQL para crear las tablas

El script se ejecuta en dos pasos con dos usuarios distintos, ambos desde
**Database Actions → SQL** (navegador, sin instalar nada).

```
Consola OCI → tu instancia TICKETAGENT → "Database actions" → SQL
```

#### Paso 1 — Como ADMIN: dar permiso Oracle Text a AGENTE

```
Database Actions
  ├── Usuario activo: ADMIN  (selector arriba a la izquierda)
  └── Panel SQL (Hoja de trabajo)
```

```sql
GRANT EXECUTE ON CTXSYS.CTX_DDL TO AGENTE;
```

Ejecutar con el botón **▶ verde** de la barra de herramientas.
Resultado esperado en el panel inferior: `Grant succeeded.`

> El panel izquierdo ("Tablas") solo muestra `DBTOOLS$EXECUTION_HISTORY`
> mientras estás como ADMIN — es la tabla interna de Database Actions, normal.

#### Paso 2 — Como AGENTE: crear tablas e índices

Cerrar sesión de ADMIN y volver a entrar como **AGENTE**:

```
URL directa: https://adb.us-chicago-1.oraclecloud.com/ords/agente/_sdw/
  (o desde la consola OCI → Database actions → cambiar usuario)
```

Pegar y ejecutar el script completo en la Hoja de trabajo:

```sql
-- 1. Tabla principal de chunks vectoriales
CREATE TABLE KB_CHUNKS (
    CHUNK_ID    VARCHAR2(36)         NOT NULL,
    PARENT_ID   VARCHAR2(36),
    CONTENT     CLOB                 NOT NULL,
    EMBEDDING   VECTOR(1024, FLOAT32),
    SOURCE_FILE VARCHAR2(512),
    TIPO        VARCHAR2(50),
    FUENTE      VARCHAR2(100),
    PANTALLA    VARCHAR2(100),
    ROL_JSON    JSON,
    IS_PARENT   NUMBER(1,0)  DEFAULT 0,
    CREATED_AT  TIMESTAMP    DEFAULT SYSTIMESTAMP,
    CONSTRAINT pk_kb_chunks   PRIMARY KEY (CHUNK_ID),
    CONSTRAINT chk_is_parent  CHECK (IS_PARENT IN (0, 1)),
    CONSTRAINT chk_tipo       CHECK (TIPO IN ('usuarios', 'runbooks', 'helpContent')),
    CONSTRAINT fk_kb_parent   FOREIGN KEY (PARENT_ID)
                              REFERENCES KB_CHUNKS(CHUNK_ID)
);

-- 2. Índice HNSW para búsqueda vectorial ANN
CREATE VECTOR INDEX idx_kb_embedding
    ON KB_CHUNKS(EMBEDDING)
    ORGANIZATION INMEMORY NEIGHBOR GRAPH
    DISTANCE COSINE
    WITH TARGET ACCURACY 95;

-- 3. Índice Oracle Text para BM25 / Hybrid Search
CREATE INDEX idx_kb_content_text
    ON KB_CHUNKS(CONTENT)
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS ('SYNC (ON COMMIT)');

-- 4. Índices regulares para filtros frecuentes
CREATE INDEX idx_kb_tipo      ON KB_CHUNKS(TIPO);
CREATE INDEX idx_kb_is_parent ON KB_CHUNKS(IS_PARENT);

-- 5. Log de ingestas
CREATE TABLE KB_INGEST_LOG (
    LOG_ID      NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    SOURCE_FILE VARCHAR2(512)  NOT NULL,
    CHUNK_COUNT NUMBER(6),
    TIPO        VARCHAR2(50),
    INGEST_AT   TIMESTAMP DEFAULT SYSTIMESTAMP,
    STATUS      VARCHAR2(20) DEFAULT 'OK',
    ERROR_MSG   VARCHAR2(2000),
    CONSTRAINT chk_log_status CHECK (STATUS IN ('OK', 'ERROR'))
);

-- 6. Vista de chunks hoja (usada por el OracleVSAdapter)
CREATE OR REPLACE VIEW V_KB_LEAF_CHUNKS AS
SELECT CHUNK_ID, PARENT_ID, CONTENT, EMBEDDING,
       SOURCE_FILE, TIPO, FUENTE, PANTALLA, ROL_JSON, CREATED_AT
FROM KB_CHUNKS
WHERE IS_PARENT = 0;

-- 7. Verificación final
SELECT TABLE_NAME FROM USER_TABLES
WHERE TABLE_NAME IN ('KB_CHUNKS','KB_INGEST_LOG')
ORDER BY TABLE_NAME;
```

Resultado esperado en el panel de resultados:

```
TABLE_NAME
──────────────
KB_CHUNKS
KB_INGEST_LOG
```

Si el panel izquierdo ("Tablas") se refresca, también deberías ver
`KB_CHUNKS` y `KB_INGEST_LOG` en el árbol de objetos.

### 5.9 Verificar conexión desde WSL

```bash
pip install oracledb

python3 -c "
import oracledb

conn = oracledb.connect(
    user='AGENTE',
    password='TuPasswordAGENTE',
    dsn='ticketagent_tp',
    config_dir='/home/jjrm/.oci/wallet_ticketagent',     # ubicación de tnsnames.ora
    wallet_location='/home/jjrm/.oci/wallet_ticketagent', # ubicación de los certificados
    wallet_password='TuWalletPassword'
)
print('Versión Oracle:', conn.version)
conn.close()
"
# Resultado obtenido: Versión Oracle: 23.26.2.2.0  ✅
# Nota: config_dir y wallet_location apuntan al mismo directorio del wallet
```

### 5.8 Configuración en el contenedor ticket-agent

El `docker-compose.yml` real monta el directorio `.oci` completo (no solo la subcarpeta del wallet):

```yaml
# docker-compose.yml — fragmento relevante del servicio ticket-agent
volumes:
  - ${HOME}/.oci:/root/.oci:ro   # monta TODO ~/.oci del host → /root/.oci en el contenedor
environment:
  - ORACLE_CONFIG_DIR=${ORACLE_CONFIG_DIR:-}
  - ORACLE_WALLET_DIR=${ORACLE_WALLET_DIR:-}
  - ORACLE_WALLET_PASSWORD=${ORACLE_WALLET_PASSWORD:-}
  - VECTOR_STORE=${VECTOR_STORE:-faiss}
```

Y en el `.env` del host (valores que se inyectan al contenedor):

```bash
ORACLE_CONFIG_DIR=/root/.oci/wallet_ticketagent   # ruta DENTRO del contenedor
ORACLE_WALLET_DIR=/root/.oci/wallet_ticketagent   # ruta DENTRO del contenedor
```

> **Distinción crítica**: las variables `ORACLE_CONFIG_DIR` y `ORACLE_WALLET_DIR` las lee
> el proceso Python **dentro del contenedor**. La ruta debe ser la ruta de contenedor
> (`/root/.oci/...`), no la ruta del host WSL (`/home/jjrm/.oci/...`).
>
> El montaje `${HOME}/.oci:/root/.oci:ro` hace que el wallet del host quede visible
> en el contenedor como `/root/.oci/wallet_ticketagent/tnsnames.ora`.
>
> **Para ejecutar el ingest CLI directamente en WSL** (fuera del contenedor),
> las rutas deben ser las del host: `/home/jjrm/.oci/wallet_ticketagent`.
> En ese caso sobrescribir con: `ORACLE_CONFIG_DIR=/home/jjrm/.oci/wallet_ticketagent python -m src.main ingest`
>
> **`config_dir` vs `wallet_location`**: ambos apuntan al mismo directorio del wallet.
> `config_dir` es necesario para que oracledb encuentre `tnsnames.ora` al resolver
> el DSN por nombre (`ticketagent_tp`). Sin él se obtiene `DPY-4027`.
> `wallet_location` es necesario para los certificados mTLS.

---

## 6. Script SQL completo

Guardar como `oracle/init_kb.sql` en la raíz del repo.
Ejecutar conectado al usuario del agente (no SYS).

```sql
-- =============================================================
-- ticket-agent / Fase 2 — Oracle 26ai (Autonomous AI Database, OCI Always Free)
-- Script: init_kb.sql
-- Ejecutar como: usuario AGENTE en Database Actions (navegador OCI)
-- Verificado en: Oracle 23.26.2.2.0
-- =============================================================

-- ------------------------------------------------------------
-- 1. Tabla principal de chunks vectoriales
-- ------------------------------------------------------------
CREATE TABLE KB_CHUNKS (
    CHUNK_ID    VARCHAR2(36)         NOT NULL,
    PARENT_ID   VARCHAR2(36),
    CONTENT     CLOB                 NOT NULL,
    EMBEDDING   VECTOR(1024, FLOAT32),
    SOURCE_FILE VARCHAR2(512),
    TIPO        VARCHAR2(50),
    FUENTE      VARCHAR2(100),
    PANTALLA    VARCHAR2(100),
    ROL_JSON    JSON,
    IS_PARENT   NUMBER(1,0)  DEFAULT 0,
    CREATED_AT  TIMESTAMP    DEFAULT SYSTIMESTAMP,
    CONSTRAINT pk_kb_chunks   PRIMARY KEY (CHUNK_ID),
    CONSTRAINT chk_is_parent  CHECK (IS_PARENT IN (0, 1)),
    CONSTRAINT chk_tipo       CHECK (TIPO IN ('usuarios', 'runbooks', 'helpContent')),
    CONSTRAINT fk_kb_parent   FOREIGN KEY (PARENT_ID)
                              REFERENCES KB_CHUNKS(CHUNK_ID)
);

-- ------------------------------------------------------------
-- 2. Índice HNSW para búsqueda vectorial ANN (similitud coseno)
-- ------------------------------------------------------------
CREATE VECTOR INDEX idx_kb_embedding
    ON KB_CHUNKS(EMBEDDING)
    ORGANIZATION INMEMORY NEIGHBOR GRAPH
    DISTANCE COSINE
    WITH TARGET ACCURACY 95;

-- ------------------------------------------------------------
-- 3. Índice Oracle Text para BM25 / Hybrid Search (Fase 2)
-- ------------------------------------------------------------
CREATE INDEX idx_kb_content_text
    ON KB_CHUNKS(CONTENT)
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS ('SYNC (ON COMMIT)');

-- ------------------------------------------------------------
-- 4. Índices regulares para filtros frecuentes
-- ------------------------------------------------------------
CREATE INDEX idx_kb_tipo      ON KB_CHUNKS(TIPO);
CREATE INDEX idx_kb_is_parent ON KB_CHUNKS(IS_PARENT);

-- ------------------------------------------------------------
-- 5. Log de ingestas (trazabilidad)
-- ------------------------------------------------------------
CREATE TABLE KB_INGEST_LOG (
    LOG_ID      NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    SOURCE_FILE VARCHAR2(512)  NOT NULL,
    CHUNK_COUNT NUMBER(6),
    TIPO        VARCHAR2(50),
    INGEST_AT   TIMESTAMP DEFAULT SYSTIMESTAMP,
    STATUS      VARCHAR2(20) DEFAULT 'OK',
    ERROR_MSG   VARCHAR2(2000),
    CONSTRAINT chk_log_status CHECK (STATUS IN ('OK', 'ERROR'))
);

-- ------------------------------------------------------------
-- 6. Vista de chunks hoja (sin padres) — usada por el adapter
-- ------------------------------------------------------------
CREATE OR REPLACE VIEW V_KB_LEAF_CHUNKS AS
SELECT
    CHUNK_ID,
    PARENT_ID,
    CONTENT,
    EMBEDDING,
    SOURCE_FILE,
    TIPO,
    FUENTE,
    PANTALLA,
    ROL_JSON,
    CREATED_AT
FROM KB_CHUNKS
WHERE IS_PARENT = 0;

-- ------------------------------------------------------------
-- 7. Queries de referencia para el OracleVSAdapter
-- ------------------------------------------------------------
-- Retrieval filtrado por rol (equivalente a FAISSVectorStore.retrieve con rol)
--
-- SELECT CHUNK_ID, CONTENT, SOURCE_FILE, TIPO, PANTALLA,
--        VECTOR_DISTANCE(EMBEDDING, :query_vec, COSINE) AS score
-- FROM   V_KB_LEAF_CHUNKS
-- WHERE  JSON_EXISTS(ROL_JSON, '$[*]?(@ == :rol)')
-- ORDER  BY score
-- FETCH  FIRST :k ROWS ONLY;
--
-- Retrieval sin filtro de rol (equivalente a rol=soporte)
--
-- SELECT CHUNK_ID, CONTENT, SOURCE_FILE, TIPO, PANTALLA,
--        VECTOR_DISTANCE(EMBEDDING, :query_vec, COSINE) AS score
-- FROM   V_KB_LEAF_CHUNKS
-- ORDER  BY score
-- FETCH  FIRST :k ROWS ONLY;
--
-- Parent-Child: recuperar contexto padre de un chunk hijo (Fase 2)
--
-- SELECT c_padre.CONTENT
-- FROM   KB_CHUNKS c_padre
-- JOIN   KB_CHUNKS c_hijo ON c_hijo.PARENT_ID = c_padre.CHUNK_ID
-- WHERE  c_hijo.CHUNK_ID = :hijo_chunk_id;

-- ------------------------------------------------------------
-- 8. Verificación post-creación
-- ------------------------------------------------------------
SELECT TABLE_NAME, NUM_ROWS
FROM   USER_TABLES
WHERE  TABLE_NAME IN ('KB_CHUNKS', 'KB_INGEST_LOG')
ORDER  BY TABLE_NAME;

SELECT INDEX_NAME, INDEX_TYPE, STATUS
FROM   USER_INDEXES
WHERE  TABLE_NAME = 'KB_CHUNKS'
ORDER  BY INDEX_NAME;
```

---

## 7. Variables de entorno adicionales

Agregar a `.env` y a `.env.example` cuando se implemente `OracleVSAdapter`:

```bash
# Oracle 26ai — Autonomous AI Database (OCI Always Free)
# mTLS requerido — wallet en ~/.oci/wallet_ticketagent/
VECTOR_STORE=faiss                 # faiss | oracle — faiss por defecto hasta Fase 2

ORACLE_USER=AGENTE
ORACLE_PASSWORD=tu_password
ORACLE_DSN=ticketagent_tp          # perfil en tnsnames.ora del wallet
ORACLE_CONFIG_DIR=/root/.oci/wallet_ticketagent   # ruta dentro del contenedor (compose monta ~/.oci → /root/.oci)
ORACLE_WALLET_DIR=/root/.oci/wallet_ticketagent   # ruta dentro del contenedor
# Para ingest CLI directo en WSL (fuera del contenedor) usar: /home/jjrm/.oci/wallet_ticketagent
ORACLE_WALLET_PASSWORD=tu_wallet_password
ORACLE_TABLE=KB_CHUNKS

# Hybrid Search (Fase 2 — opcional)
HYBRID_SEARCH_ENABLED=false
HYBRID_ALPHA=0.7                   # peso del score vectorial vs BM25 (0.0–1.0)
```

`VECTOR_STORE=faiss` por defecto — el sistema sigue funcionando sin Oracle hasta que
`VectorStoreFactory` se implemente en Fase 2.

> **mTLS requerido en Always Free**: el tier gratuito de OCI Autonomous Database
> mantiene mTLS como `Required`. Desactivarlo requiere una ACL con IPs fijas,
> incompatible con la IP dinámica de WSL.
> Wallet en WSL: `~/.oci/wallet_ticketagent/` — conexión verificada `23.26.2.2.0` ✅

---

## 8. Tareas de implementación

### 8.1 Infraestructura Oracle ✅ COMPLETADA (2026-05-23)

- [x] Crear instancia Oracle 26ai — Autonomous AI Database Serverless (OCI Always Free)
      `ticket-agent-kb` · DB: `TICKETAGENT` · región: US Midwest (Chicago)
- [x] Crear usuario `AGENTE` con roles CONNECT + RESOURCE + acceso web
- [x] Ejecutar `init_kb.sql` como AGENTE en Database Actions:
      `KB_CHUNKS` · `KB_INGEST_LOG` · índices HNSW + Oracle Text · `V_KB_LEAF_CHUNKS`
- [x] Descargar wallet → `~/.oci/wallet_ticketagent/`
- [x] Verificar conexión Python desde WSL:
      `oracledb.connect(dsn, config_dir, wallet_location, wallet_password)` → `23.26.2.2.0` ✅
- [x] Documentar `config_dir` (requerido para resolver TNS — evita `DPY-4027`)

### 8.2 OracleVSAdapter ✅ COMPLETADA (2026-05-23)

- [x] Crear `src/vector_store/oracle_store.py`
  - `OracleVSAdapter` con la misma interfaz que `FAISSVectorStore`
  - Métodos: `add_chunks()`, `load()`, `retrieve(query, k, rol)`
  - `add_chunks()` → DELETE + INSERT + LOG en una sola transacción con rollback
  - `retrieve()` con filtro `JSON_EXISTS` cuando `rol != "soporte"`
  - `retrieve()` sin filtro cuando `rol == "soporte"`
- [x] Usar `oracledb` directamente (no `langchain-community OracleVS`)
  - Razón: `langchain-community OracleVS` almacena metadata como JSON en un solo CLOB,
    lo que impide filtrar por rol a nivel SQL
- [x] `EMBEDDING` insertado con `array.array("f", emb)` → `VECTOR(1024, FLOAT32)` ✅
- [x] Ingesta end-to-end validada: 1233 chunks persistidos en KB_CHUNKS ✅
- [x] Retrieval validado en los 3 roles: `usuario`, `manager`, `soporte` ✅
- [x] Filtro `JSON_EXISTS` SQL confirmado (roles devuelven chunks coherentes) ✅
- [x] OCI GenAI (`cohere.command-r-plus-08-2024`) generando respuestas con fuentes ✅

### 8.3 VectorStoreFactory ✅ COMPLETADA (2026-05-23)

- [x] Crear `src/vector_store/factory.py`:
  ```python
  class VectorStoreFactory:
      @staticmethod
      def create(settings, embeddings):
          if settings.VECTOR_STORE == "oracle":
              return OracleVSAdapter(embeddings, settings)
          return FAISSVectorStore(embeddings, settings.VECTOR_DB_PATH)
  ```
- [x] Actualizar `src/main.py` para usar `VectorStoreFactory.create(settings, embedder)`
  - Comandos `ingest` y `query` usan la factory — swap transparente con 1 var de entorno
- [x] Agregar `oracledb>=2.0,<3.0` a `requirements.txt`
- [x] Agregar bloque Oracle a `.env.example`

### 8.4 Parent-Child chunking ✅ COMPLETADA (2026-05-24)

- [x] Implementar `ParentChildChunker` en `src/document_processing/chunking.py`
  - Hijo: 512 tokens, overlap 100 (se indexa con embedding)
  - Padre: ~2048 tokens, overlap 0 (`IS_PARENT=1`, proporciona contexto al LLM)
  - Genera `parent_id` (UUID) en cada chunk hijo apuntando al chunk padre
- [x] Actualizar `OracleVSAdapter.add_chunks()`:
  - INSERT incluye `IS_PARENT` y `PARENT_ID` (10 columnas nombradas)
  - Se embeben TODOS los chunks (padres e hijos) — EMBEDDING no es nullable
- [x] Actualizar `OracleVSAdapter.retrieve()` con LEFT JOIN:
  ```sql
  SELECT c.CHUNK_ID, COALESCE(p.CONTENT, c.CONTENT) AS CONTEXT_CONTENT, ...
  FROM KB_CHUNKS c
  LEFT JOIN KB_CHUNKS p ON p.CHUNK_ID = c.PARENT_ID
  WHERE c.IS_PARENT = 0 [AND JSON_EXISTS...]
  ORDER BY VECTOR_DISTANCE(c.EMBEDDING, :qvec, COSINE)
  FETCH FIRST :k ROWS ONLY
  ```
- [x] Re-ingestar con `CHUNKING_STRATEGY=parent_child`:
  **1557 chunks persistidos (306 padres + 1251 hijos)** ✅
- [x] Test: query "¿Qué flujo sigue un ticket...?" --rol usuario → respuesta completa
  con los 4 estados del ciclo de vida (ABIERTO → EN PROGRESO → EN REVISIÓN → CERRADO) ✅
- [x] Agregar `CHUNKING_STRATEGY` y `PARENT_CHUNK_SIZE` a `config.py` y `.env.example`
- [x] 6 tests unitarios en `tests/test_chunking.py::test_parent_child_*` ✅

### 8.5 Hybrid Search ✅ COMPLETADA (2026-05-24)

- [x] Implementar `retrieve_hybrid()` en `OracleVSAdapter`:
  - Vector over-fetch (4×K) → Oracle Text `CONTAINS` BM25 → RRF merge (k=60)
  - `alpha=0.7` (peso vector en RRF) configurable con `HYBRID_ALPHA` en `.env`
  - Fallback automático a vector puro si `IDX_KB_CONTENT_TEXT` no está disponible
- [x] Implementar `_bm25_rank()` — CONTAINS con filtro de rol y `SCORE(1)` ordenado
- [x] `Retriever` llama a `retrieve_hybrid()` cuando `HYBRID_SEARCH=true` y el store lo soporta
- [x] Agregar `HYBRID_SEARCH: bool = False` y `HYBRID_ALPHA: float = 0.7` a `config.py`
- [x] **Recrear `IDX_KB_CONTENT_TEXT`** (recreado tras DROP por ingesta interrumpida) ✅
- [x] **Validar end-to-end**: `Oracle hybrid RRF: 3 docs (vec=20, bm25=20, α=0.7, rol=usuario)` ✅
      Query "¿Qué estados tiene un ticket?" → 4 estados correctos (antes: 2 incompletos) ✅
      Fix clave: query Oracle Text usa OR explícito (`_to_oracle_text_query` → `" OR ".join(words)`) — ver sección 9.8
      Fix soporte: `RETRIEVAL_K=6` (default era 3) — necesario con parent-child para cubrir docs largos partidos en 2 padres

### 8.6 Property Graph ✅ IMPLEMENTADO (2026-05-27) — pendiente validación en WSL

- [x] Crear `docs/arquitectura/ecosistema.mmd` — fuente canónica del grafo
      18 nodos (9 SERVICE, 7 STORAGE, 2 EXTERNAL) · 27 aristas tipadas
      Tipos de relación: `CALLS`, `STORES_IN`, `READS`, `SCRAPES`, `PUSHES_TO`, `TRIGGERS`
- [x] Implementar `src/document_processing/mermaid_parser.py`
      Parsea nodos (`ID["..."]` y `ID[("...")]`) y aristas (`-->|"RELATION"|`)
      Extrae `tipo:`, `puerto:` y descripción de etiquetas multilinea
- [x] Crear `db/migrations/oracle/V1__init_kb.sql` — baseline del schema KB_CHUNKS
      Documenta formalmente la migración aplicada manualmente en Fase 2
- [x] Crear `db/migrations/oracle/V2__add_graph_tables.sql`
      `KB_GRAPH_NODES` · `KB_GRAPH_EDGES` · `CREATE PROPERTY GRAPH ticket_sys_graph`
      Índices de soporte en `SRC_ID`, `DST_ID`, `RELATION`, `TIPO`
- [x] Agregar `add_graph()` y `graph_traverse_query()` a `OracleVSAdapter`
      `add_graph()`: DELETE + INSERT nodos → DELETE + INSERT aristas → COMMIT
      `graph_traverse_query()`: SQL/PGQ en ambas direcciones (salientes + entrantes)
- [x] Nodo `graph_traverse` en LangGraph (`src/agent/nodes.py`)
      Se activa con keywords: `depende`, `usa`, `cae`, `impacto`, `arquitectura`, etc.
      Detecta nombres de servicios en la query → ejecuta PGQ → añade relaciones al contexto
      Falla silenciosamente si las tablas no existen o el grafo está vacío
- [x] `build_graph(graph_store=None)` — nodo opcional en `src/agent/graph.py`
      Si `graph_store` es None (FAISS), el flujo es idéntico al anterior
      Si `graph_store` es OracleVSAdapter, inserta `graph_traverse` entre `retrieve` y `generate`
- [x] Agregar `GRAPH_MMD_PATH` a `config.py` y `.env.example`
- [ ] **Validar en WSL**: `ingest --source graph` + queries de arquitectura (ver sección 9.11)

### 8.7 Tablas Oracle especializadas + Routing multi-tabla ✅ COMPLETADO (2026-05-29)

- [x] Crear `db/migrations/oracle/V3__add_specialized_tables.sql`
      Aplicar como AGENTE en Database Actions (mismo procedimiento que V1/V2)
- [x] `KB_CHUNKS.chk_tipo` ampliado: permite `'manuales'` e `'incidentes'` (DROP + recrear constraint)
- [x] **KB_MANUALS** — manuales técnicos PDF/DOCX/MD · campos: `DOC_FORMAT`, `DOC_VERSION` · HNSW + Oracle Text + `V_KB_MANUALS_LEAF`
- [x] **KB_RUNBOOKS** — runbooks operacionales · campos: `SERVICE`, `RUNBOOK_TYPE` · HNSW + Oracle Text + `V_KB_RUNBOOKS_LEAF`
- [x] **KB_INCIDENTS** — incidentes históricos · campos: `INCIDENT_DATE`, `SEVERITY`, `STATUS`, `SERVICE` · HNSW + Oracle Text + `V_KB_INCIDENTS_LEAF`
- [x] **KB_ERROR_CATALOG** — catálogo de errores · campos: `ERROR_CODE`, `SERVICE`, `SEVERITY`, `REMEDIATION` · HNSW + Oracle Text + `V_KB_ERRORS_LEAF`
- [x] `KB_INGEST_LOG`: columna `TARGET_TABLE VARCHAR2(50)` agregada para trazabilidad por tabla destino
- [x] Routing de escritura en `OracleVSAdapter.add_chunks()`:
      `_TIPO_TO_TABLE = {manuales→KB_MANUALS, incidentes→KB_INCIDENTS, resto→KB_CHUNKS}`
      `_do_insert()`: SQL específico por tabla; KB_MANUALS deriva `DOC_FORMAT` desde extensión del archivo
- [x] Routing de lectura en `OracleVSAdapter.retrieve()`:
      rol específico → KB_CHUNKS con filtro JSON (ruta rápida, sin cambio)
      rol soporte → `_SOPORTE_TABLES = [KB_CHUNKS, KB_MANUALS, KB_INCIDENTS]` con RRF Python
- [x] `_vec_search(table, query_vec, k, rol)` — helper de búsqueda vectorial por tabla con parent expansion
- [x] `load()` — cuenta chunks en las 3 tablas activas al arrancar la API
- [x] **Validado en WSL (2026-05-29)**:
      `ingest --source all` → `KB_CHUNKS=1284, KB_MANUALS=0, KB_INCIDENTS=0` ✅
      `query --rol usuario` → `tables=['KB_CHUNKS']` ✅
      `query --rol soporte` → `tables=['KB_CHUNKS', 'KB_MANUALS', 'KB_INCIDENTS']` · RRF degrada limpiamente con tablas vacías ✅

### 8.8 Docling — loader multi-formato ✅ COMPLETADO (2026-05-29)

- [x] `src/document_processing/docling_loader.py` — importación lazy de Docling
      `.md` → lectura directa (sin overhead Docling)
      `.yaml/.yml` → pyyaml
      PDF/DOCX/PPTX/URLs → `DocumentConverter` de Docling
      `load_manuales(paths_csv)` → rol soporte+admin, tipo manuales → KB_MANUALS
      `load_incidentes(paths_csv)` → rol soporte, tipo incidentes → KB_INCIDENTS
- [x] `loader.py.load_all()` extendido: `--source manuales` y `--source incidentes`
- [x] `config.py`: `SOURCE_MANUALES = ""` y `SOURCE_INCIDENTES = ""` (vacíos → desactivados)
- [x] `.env.example` documentado con las nuevas variables
- [x] `requirements.txt`: `docling>=2.0,<3.0`

### 8.9 Error Event + POST /api/v1/diagnose ✅ COMPLETADO (2026-05-29)

- [x] `ErrorEventRequest` (Pydantic): `errorCode`, `service`, `severity`, `message`, `stackTrace`
- [x] `DiagnosisResponse` (Pydantic): `causa`, `remediacion`, `runbook`, `fuentes`, `confianza`, `latenciaMs`, `servicio`, `severidad`
- [x] `DIAGNOSE_PROMPT` + `format_diagnose_prompt()` + `_parse_diagnose_response()` en `src/rag/prompts.py`
      Prompt pide secciones CAUSA / REMEDIACION / RUNBOOK al LLM
      `_parse_diagnose_response()`: busca marcadores con y sin tilde (`REMEDIACION` / `REMEDIACIÓN`)
- [x] `src/api/routes/diagnose.py` — `POST /api/v1/diagnose`
      Usa `retriever` directo (rol soporte → multi-tabla) en lugar del grafo LangGraph completo
      Construye query desde `{errorCode} + {service} + {message} + {stackTrace[:300]}`
- [x] `app.state.retriever` y `app.state.llm` expuestos en lifespan
- [x] **Validado en WSL (2026-05-29)**:
      `POST /api/v1/diagnose` con `ORA-00942` → causa + remediacion separados, runbook=null, 5 fuentes ✅
      Latencia: 13.8s (OCI GenAI cohere.command-r-plus-08-2024) ✅

### 8.10 Actualizar podman-compose.yml

- [ ] Agregar servicio `oracle23ai` antes de `ticket-agent`
- [ ] `ticket-agent` depende de `oracle23ai` (`depends_on`)
- [ ] Reemplazar variable de volumen `faiss_data` por variable `ORACLE_DSN` en el servicio
  cuando `VECTOR_STORE=oracle`
- [ ] Mantener compatibilidad: si `VECTOR_STORE=faiss`, el volumen `faiss_data` sigue activo

---

## 9. Operaciones — Ingesta y monitoreo

### 9.1 Pre-requisitos

```
✅ .venv activado
✅ .env con VECTOR_STORE=oracle y variables Oracle configuradas
✅ Wallet en ~/.oci/wallet_ticketagent/
✅ KB_CHUNKS existente en Oracle (ejecutar init_kb.sql si no existe)
```

### 9.2 Ejecutar la ingesta

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate
python -m src.main ingest --source all
```

### 9.3 Logs esperados durante la ingesta

El proceso tarda ~35-45 minutos para 1557 chunks con `CHUNKING_STRATEGY=parent_child`
en CPU (modelo ONNX `mxbai-embed-large-v1`). Con `flat` (~1233 chunks) tarda ~27 min.

**Con `CHUNKING_STRATEGY=parent_child` (configuración actual):**

```
[ingest] Fuente: all
INFO  __main__  Cargando modelo de embeddings: mixedbread-ai/mxbai-embed-large-v1
[ingest] Cargando documentos...
INFO  loader    load_all(source=all): 63 documentos en total
[ingest] 63 documentos cargados. Chunkeando...
[ingest] 1557 chunks generados (306 padres + 1251 hijos). Embebiendo y persistiendo...
INFO  oracle_store  Oracle: embebiendo 1557 chunks...
INFO  oracle_store  Oracle: embebidos 100/1557 chunks    ← ~2 min entre batches
INFO  oracle_store  Oracle: embebidos 200/1557 chunks
...
INFO  oracle_store  Oracle: embebidos 1500/1557 chunks
INFO  oracle_store  Oracle: 1557 chunks persistidos en KB_CHUNKS   ← COMMIT
[ingest] Listo. 1557 chunks persistidos en KB_CHUNKS
```

**Con `CHUNKING_STRATEGY=flat` (legacy):**

```
[ingest] 1233 chunks generados. Embebiendo y persistiendo...
INFO  oracle_store  Oracle: embebiendo 1233 chunks...
...
INFO  oracle_store  Oracle: 1233 chunks persistidos en KB_CHUNKS
```

> La transacción hace COMMIT una sola vez al final: DELETE + todos los INSERTs + LOG.
> Si el proceso muere antes del COMMIT, KB_CHUNKS queda intacta (rollback automático).

### 9.4 Monitorear en terminal paralelo

Abrir un segundo terminal y crear el script de monitoreo:

```bash
cat > /tmp/check_oracle.py << 'EOF'
import oracledb, os
from dotenv import load_dotenv

load_dotenv("/home/jjrm/stack_ticket/ticket-agent/.env")

conn = oracledb.connect(
    user=os.environ["ORACLE_USER"],
    password=os.environ["ORACLE_PASSWORD"],
    dsn=os.environ["ORACLE_DSN"],
    config_dir=os.environ["ORACLE_CONFIG_DIR"],
    wallet_location=os.environ["ORACLE_WALLET_DIR"],
    wallet_password=os.environ["ORACLE_WALLET_PASSWORD"],
)
cur = conn.cursor()
cur.execute("SELECT COUNT(*) FROM KB_CHUNKS WHERE IS_PARENT = 0")
chunks = cur.fetchone()[0]
print(f"KB_CHUNKS : {chunks} chunks")

cur.execute("SELECT SOURCE_FILE, CHUNK_COUNT, STATUS, ERROR_MSG FROM KB_INGEST_LOG ORDER BY INGEST_AT DESC FETCH FIRST 5 ROWS ONLY")
rows = cur.fetchall()
if rows:
    print(f"INGEST_LOG: {len(rows)} registros")
    for r in rows:
        msg = f"  {r[0]} | {r[1]} chunks | {r[2]}"
        if r[3]:
            msg += f" | {r[3][:80]}"
        print(msg)
else:
    print("INGEST_LOG: vacio")
conn.close()
EOF

# Ejecutar con actualización cada 30 segundos
watch -n 30 python3 /tmp/check_oracle.py
```

**Qué esperar en el monitor:**

```
Durante la ingesta (transacción en curso):
  KB_CHUNKS : 0 chunks       ← NORMAL — el COMMIT no ocurrió aún
  INGEST_LOG: vacio

Cuando termina exitosamente:
  KB_CHUNKS : 1233 chunks    ← aparece de golpe al hacer COMMIT
  INGEST_LOG: 1 registros
    batch_ingest | 1233 chunks | OK

Si falló con error:
  KB_CHUNKS : 0 chunks       ← rollback → KB_CHUNKS intacta
  INGEST_LOG: 1 registros
    batch_ingest | 1233 chunks | ERROR | <mensaje de error>
```

> Terminal 2 mostrará **0 hasta el final** — es comportamiento correcto.
> El conteo salta de 0 a 1233 en una sola actualización del `watch`.

### 9.5 Verificar que el proceso sigue vivo

Si no hay logs nuevos en Terminal 1 por varios minutos:

```bash
# Ver si el proceso existe
ps aux | grep "src.main" | grep -v grep

# Salida esperada (proceso vivo):
# jjrm  1434  193 30.9 6004512 2516100 pts/0 Rl+  07:20 27:35 python -m src.main ingest --source all
#         ↑              ↑
#       %CPU           %RAM (el modelo ONNX consume ~2.5 GB)

# Si no aparece → el proceso murió. Ver causa:
dmesg | grep -i "kill\|oom" | tail -10
```

### 9.6 Re-ejecutar si el proceso murió

Si el proceso terminó sin llegar al COMMIT (KB_CHUNKS sigue en 0):

```bash
source .venv/bin/activate
python -m src.main ingest --source all
```

La ingesta es idempotente: el `DELETE FROM KB_CHUNKS` al inicio de la transacción
asegura que no quedan chunks duplicados de corridas anteriores parciales.

### 9.7 Queries de validación post-ingesta

Una vez que KB_CHUNKS muestra 1233 chunks, validar los 3 roles:

```bash
source .venv/bin/activate

# Rol usuario — busca chunks con ["usuario", ...]
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario

# Rol manager — filtro JSON_EXISTS activo
python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager

# Rol soporte — sin filtro (ve todos los chunks)
python -m src.main query --query "¿Dónde se configura el modelo LLM?" --rol soporte
```

Resultado esperado: 3 documentos recuperados por query con `metadata.rol` coherente.

### 9.8 Errores conocidos durante la implementación

Errores reales encontrados durante la primera puesta en marcha (2026-05-23):

---

#### ORA-29861 — Domain index marked FAILED

```
oracledb.exceptions.DatabaseError: ORA-29861: Domain index AGENTE.IDX_KB_CONTENT_TEXT
is marked FAILED and currently not usable.
```

**Causa:** El índice Oracle Text (`IDX_KB_CONTENT_TEXT`) quedó en estado `FAILED`
cuando una ingesta anterior murió antes del COMMIT. Oracle marca el índice como
inusable y bloquea cualquier DML sobre la tabla (incluyendo el `DELETE` de la siguiente
corrida).

**Fix — ejecutar como AGENTE en Database Actions:**

```sql
DROP INDEX IDX_KB_CONTENT_TEXT;
```

Este índice es solo para Hybrid Search (Fase 2). No afecta el retrieval vectorial.
Se puede recrear cuando se implemente `retrieve_hybrid()`.

---

#### DPY-4026 — tnsnames.ora missing or unreadable (contenedor) (2026-05-29)

```
oracledb.exceptions.DatabaseError: DPY-4026: file '/home/jjrm/.oci/wallet_ticketagent/tnsnames.ora'
is missing or unreadable [Errno 2] No such file or directory
```

**Causa:** `ORACLE_CONFIG_DIR` (o `ORACLE_WALLET_DIR`) en `.env` apunta a la ruta del host WSL
(`/home/jjrm/.oci/...`). El proceso Python corre **dentro del contenedor**, donde esa ruta no existe.
El wallet está montado en `/root/.oci/wallet_ticketagent/`, no en `/home/jjrm/`.

**Diagnóstico rápido:**
```bash
grep "ORACLE_CONFIG_DIR\|ORACLE_WALLET_DIR" ~/stack_ticket/ticket-agent/.env
# Si dice /home/jjrm/... → fix requerido
```

**Fix:**
```bash
sed -i \
  's|ORACLE_CONFIG_DIR=/home/jjrm/.oci/wallet_ticketagent|ORACLE_CONFIG_DIR=/root/.oci/wallet_ticketagent|;
   s|ORACLE_WALLET_DIR=/home/jjrm/.oci/wallet_ticketagent|ORACLE_WALLET_DIR=/root/.oci/wallet_ticketagent|' \
  ~/stack_ticket/ticket-agent/.env
cd ~/stack_ticket/ticket-agent && podman-compose down && podman-compose up -d
```

> **Regla de oro:** las variables `ORACLE_CONFIG_DIR` / `ORACLE_WALLET_DIR` en `.env` deben ser
> **rutas de contenedor** cuando se usa podman-compose. El compose monta `~/.oci → /root/.oci:ro`.
> Para usar el ingest CLI directamente en WSL sin contenedor, los valores son los del host (`/home/jjrm/...`).

---

#### DPY-4027 — No configuration directory specified

```
oracledb.exceptions.DatabaseError: DPY-4027: no configuration directory specified
```

**Causa:** `oracledb.connect()` no sabe dónde está `tnsnames.ora` para resolver
el DSN por nombre (`ticketagent_tp`).

**Fix:** Pasar `config_dir` apuntando al directorio del wallet (mismo valor que
`wallet_location`):

```python
oracledb.connect(
    dsn="ticketagent_tp",
    config_dir="/home/jjrm/.oci/wallet_ticketagent",     # ← requerido
    wallet_location="/home/jjrm/.oci/wallet_ticketagent",
    wallet_password="...",
)
```

---

#### Ingesta completa pero fue a FAISS en lugar de Oracle

**Síntoma:** La ingesta termina exitosamente pero el log muestra:
```
"name": "src.vector_store.faiss_store"
"message": "FAISS: 1233 chunks persistidos en ./data/vector_db/faiss_index/"
```

**Causa:** `.env` tiene `VECTOR_STORE=faiss` (el default) en lugar de `oracle`.

**Fix:**
```bash
grep VECTOR_STORE .env
# Si dice faiss:
sed -i 's/VECTOR_STORE=faiss/VECTOR_STORE=oracle/' .env
```

---

#### ModuleNotFoundError: No module named 'oracledb'

```
ModuleNotFoundError: No module named 'oracledb'
```

**Causa:** El virtual environment no está activado.

**Fix:**
```bash
source .venv/bin/activate
python -m src.main ingest --source all
```

---

#### Proceso muere silenciosamente (sin mensaje de error en el terminal)

**Síntoma:** El terminal se queda detenido en `[ingest] Embebiendo y persistiendo...`
sin logs de progreso. Al ejecutar `ps aux` el proceso ya no existe.

**Causa probable:** OOM killer de Linux eliminó el proceso. El modelo ONNX
`mxbai-embed-large-v1` consume ~2.5 GB de RAM durante la ingesta.

**Diagnóstico:**
```bash
ps aux | grep "src.main" | grep -v grep   # verificar si el proceso existe
dmesg | grep -i "kill\|oom" | tail -10    # buscar OOM en el kernel log
```

**Fix:** Re-ejecutar la ingesta — es idempotente:
```bash
source .venv/bin/activate
python -m src.main ingest --source all
```

---

#### TypeError — json.loads sobre ROL_JSON ya deserializado

```
TypeError: the JSON object must be str, bytes or bytearray, not list
```

**Causa:** Oracle retorna el tipo `JSON` nativo ya como objeto Python (`list`).
El código llamaba `json.loads()` sobre algo que ya era una lista.

**Fix en `oracle_store.py` — retrieve():**

```python
# Antes:
"rol": json.loads(rol_json) if rol_json else [],

# Después:
"rol": rol_json if isinstance(rol_json, list) else (json.loads(rol_json) if rol_json else []),
```

---

#### ValidationError — CLOB devuelto como LOB object, no como string

```
pydantic_core.ValidationError: page_content — Input should be a valid string
  input_value=<oracledb.lob.LOB object at 0x...>, input_type=LOB
```

**Causa:** Oracle retorna columnas `CLOB` como objetos `oracledb.lob.LOB`.
`LangChain Document` requiere `page_content` como `str`.

**Fix en `oracle_store.py` — nivel de módulo:**

```python
import oracledb
oracledb.defaults.fetch_lobs = False  # CLOB/BLOB devueltos como str/bytes
```

Esta línea hace que todos los CLOBs del módulo se reciban como `str` directamente.

---

#### Pydantic — Extra inputs are not permitted

```
pydantic_core._pydantic_core.ValidationError: Extra inputs are not permitted
```

**Causa:** La instancia WSL tiene una versión antigua de `src/config.py`
que no incluye las variables declaradas en `.env` (`VECTOR_STORE`, `ORACLE_*`,
`CHUNKING_STRATEGY`, etc.). Pydantic v2 rechaza campos del env que no están
definidos en la clase Settings.

**Fix:**
```bash
git pull
```

> Patrón recurrente: el `.env` siempre se edita en WSL, pero el código que
> declara las variables (`config.py`) vive en el repo de Windows. Si se agrega
> una variable al `.env` antes de hacer `git pull`, se obtiene este error.

---

#### InvalidKeyFilePath — key_file apunta a ruta del host, no del contenedor (2026-05-24)

```
InvalidKeyFilePath: Config file /root/.oci/config is invalid:
the key_file's value '/home/jjrm/.oci/oci_api_key.pem' at line 6
must be a valid file path.
```

**Causa:** `~/.oci/config` del host tiene `key_file=/home/jjrm/.oci/oci_api_key.pem`.
El volumen monta `~/.oci → /root/.oci` dentro del contenedor, pero la ruta `/home/jjrm/`
no existe dentro del contenedor.

**Fix — crear config específico para el contenedor (no modifica el original):**

```bash
sed 's|key_file=/home/jjrm/.oci/|key_file=/root/.oci/|g' \
  ~/.oci/config > ~/.oci/config.container

# En .env del contenedor:
OCI_CONFIG_FILE=/root/.oci/config.container
```

---

#### DRG-50901 — Oracle Text parser syntax error (2026-05-24)

```
oracledb.exceptions.DatabaseError: ORA-29902: ... DRG-50901: text query parser syntax error
```

**Causa:** Oracle Text interpreta caracteres especiales como operadores de query:
`?`, `!`, `~`, `&`, `;`, `¿`, `á`, `é`, `ñ`, `(`, `)` → error de sintaxis en `CONTAINS()`.

**Fix en `oracle_store.py` — función `_to_oracle_text_query()`:**

```python
def _to_oracle_text_query(text: str) -> str:
    # 1. NFKD: descompone acentos (é → e + ́)
    normalized = unicodedata.normalize("NFKD", text).encode("ascii", "ignore").decode("ascii")
    # 2. Elimina todo lo que no sea letra/dígito/espacio
    clean = re.compile(r"[^a-zA-Z0-9\s]").sub(" ", normalized)
    words = [w for w in clean.split() if len(w) > 2]
    return " OR ".join(words) if words else "ticket"
```

Resultado: `"¿Qué estados tiene un ticket?"` → `"Que OR estados OR tiene OR ticket"` — sin errores DRG.

---

#### bm25=0 en todos los queries híbridos (2026-05-24)

**Síntoma:** `Oracle hybrid RRF: 3 docs (vec=20, bm25=0, α=0.7, rol=usuario)` en todos los queries.
La búsqueda cae en silencio al vector puro; el BM25 nunca aporta resultados.

**Causa:** `_to_oracle_text_query()` unía palabras con espacio: `"estados tiene ticket"`.
Oracle Text interpreta espacios como AND implícito — **todos** los términos deben aparecer
en el mismo chunk, condición casi imposible de cumplir.

**Fix:** cambiar a OR explícito:

```python
# Antes:
return " ".join(words) if words else "ticket"

# Después:
return " OR ".join(words) if words else "ticket"
```

Con OR, cualquier chunk que contenga al menos uno de los términos es candidato BM25.
Resultado: `bm25=20` en queries típicas.

---

#### RETRIEVAL_K=3 insuficiente con parent-child chunking (2026-05-24)

**Síntoma:** Respuesta incompleta — `"Los estados son ABIERTO y EN PROGRESO"` (faltan EN REVISIÓN y CERRADO).
Los logs muestran `bm25=20, vec=20` — el híbrido funciona, pero la respuesta es parcial.

**Causa:** Un documento largo (ej. `estados.md` ~4000 chars) se parte en 2 padres con
`PARENT_CHUNK_SIZE=2048`. El RRF con `k=3` selecciona solo hijos del Padre 1 (estados iniciales),
nunca llega al Padre 2 (estados finales).

**Fix:**

```bash
sed -i 's|^RETRIEVAL_K=.*|RETRIEVAL_K=6|' .env
```

Con `RETRIEVAL_K=6` el RRF tiene margen para incluir hijos de ambos padres, obteniendo
contexto completo del documento.

> Recomendación: con `CHUNKING_STRATEGY=parent_child`, usar `RETRIEVAL_K=6` como mínimo.

---

#### Parsing de /diagnose: REMEDIACION vs REMEDIACIÓN (2026-05-29)

**Síntoma:** `POST /api/v1/diagnose` devuelve `"remediacion": ""` aunque el LLM respondió con pasos de remediación.

**Causa:** `_parse_diagnose_response()` buscaba el marcador `"REMEDIACION:"` (sin tilde) en el texto del LLM. Python `str.upper()` preserva tildes — `"remediación".upper()` → `"REMEDIACIÓN"`, no `"REMEDIACION"`. La búsqueda no encontraba el marcador y dejaba el campo vacío.

**Fix en `src/rag/prompts.py`:**

```python
markers = [
    ("CAUSA:", "causa"),
    ("REMEDIACION:", "remediacion"),
    ("REMEDIACIÓN:", "remediacion"),  # variante con tilde
    ("RUNBOOK:", "runbook"),
]
seen_keys: set = set()
for marker, key in markers:
    if key in seen_keys:
        continue   # evita duplicar si ambas variantes aparecen
    idx = upper.find(marker)
    if idx != -1:
        positions.append((idx, marker, key))
        seen_keys.add(key)
```

También se corrigió el check de `runbook`: de `value.lower() in (...)` exacto a `value.lower().startswith("no disponible")` para cubrir cuando el LLM agrega texto adicional tras "No disponible.".

---

#### OCI_CONFIG_FILE ignorado por ChatOCIGenAI (2026-05-24)

**Síntoma:** `OCI_CONFIG_FILE` existe en `.env` y en `settings`, pero `ChatOCIGenAI`
sigue leyendo `~/.oci/config` por defecto.

**Causa:** `OCILLMAdapter` no pasaba `auth_file_location` al constructor de `ChatOCIGenAI`.

**Fix en `src/adapters/oci_adapter.py`:**

```python
if config_file:
    kwargs["auth_file_location"] = config_file
self.llm = ChatOCIGenAI(**kwargs)
```

Y en `src/api/main.py`, pasar `config_file=settings.OCI_CONFIG_FILE` al construir el adapter.

---

#### api/main.py hardcodeado a FAISSVectorStore, ignorando VectorStoreFactory (2026-05-24)

**Síntoma:** Con `VECTOR_STORE=oracle`, el contenedor crashea con:
```
RuntimeError: could not open /app/data/vector_db/faiss_index/index.faiss
```

**Causa:** `src/api/main.py` importaba `FAISSVectorStore` directamente.
La `VectorStoreFactory` existía en `src/main.py` (CLI) pero no en la API REST.

**Fix en `src/api/main.py`:**

```python
# Antes:
from ..vector_store.faiss_store import FAISSVectorStore
store = FAISSVectorStore(embedder, settings.VECTOR_DB_PATH)

# Después:
from ..vector_store.factory import VectorStoreFactory
store = VectorStoreFactory.create(settings, embedder)
```

---

### 9.10 Activar Hybrid Search

#### Paso 1 — Recrear el índice Oracle Text

El índice `IDX_KB_CONTENT_TEXT` fue eliminado durante la Fase 2 (ingesta interrumpida
lo dejó en estado FAILED). Debe crearse una vez, con datos ya en la tabla.

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate

python3 -c "
from src.config import settings
import oracledb
conn = oracledb.connect(
    user=settings.ORACLE_USER, password=settings.ORACLE_PASSWORD,
    dsn=settings.ORACLE_DSN, config_dir=settings.ORACLE_CONFIG_DIR,
    wallet_location=settings.ORACLE_WALLET_DIR, wallet_password=settings.ORACLE_WALLET_PASSWORD,
)
conn.cursor().execute('''
    CREATE INDEX IDX_KB_CONTENT_TEXT ON KB_CHUNKS(CONTENT)
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS (\'SYNC (ON COMMIT)\')
''')
conn.commit()
print('IDX_KB_CONTENT_TEXT creado OK')
"
```

> La creación indexa los 1557 chunks existentes. Tarda ~30-60 segundos.
> Con `SYNC (ON COMMIT)` el índice se actualiza automáticamente en cada `COMMIT`
> de una ingesta futura — no requiere mantenimiento manual.

#### Paso 2 — Activar en `.env`

```bash
# Ya configurado en .env de producción:
HYBRID_SEARCH=true
HYBRID_ALPHA=0.7
RETRIEVAL_K=6      # mínimo recomendado con parent_child — ver sección 9.8
```

`HYBRID_ALPHA` controla el balance vector/BM25 en RRF:
- `0.7` → vector tiene más peso (recomendado para preguntas semánticas)
- `0.5` → pesos iguales
- `0.3` → BM25 tiene más peso (mejor para búsqueda por términos exactos)

> Con `CHUNKING_STRATEGY=parent_child` se recomienda `RETRIEVAL_K=6` (no el default de 3).
> Documentos largos partidos en 2 padres necesitan más slots en el RRF para cubrir todo el contenido.

#### Paso 3 — Validar mejora de relevancia

Ejecuta los mismos queries con hybrid desactivado y activado para comparar:

```bash
# Desactivar (referencia)
HYBRID_SEARCH=false python -m src.main query \
  --query "¿Qué estados tiene un ticket?" --rol usuario

# Activar
python -m src.main query \
  --query "¿Qué estados tiene un ticket?" --rol usuario
```

Queries representativas para comparar:

```bash
# 1. Búsqueda semántica (vector debería dominar)
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario

# 2. Términos técnicos exactos (BM25 debería aportar)
python -m src.main query --query "PROVIDER OCI_MODEL OLLAMA_BASE_URL" --rol soporte

# 3. Pregunta contextual multi-párrafo
python -m src.main query --query "¿Qué flujo sigue un ticket desde que se crea hasta que se resuelve?" --rol usuario

# 4. Filtro de rol
python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager

# 5. Runbook técnico
python -m src.main query --query "¿Cómo está configurado n8n en ticket-ingestion-light?" --rol soporte
```

En el log aparecerá la línea de diagnóstico cuando hybrid está activo:
```
Oracle hybrid RRF: 3 docs (vec=20, bm25=8, α=0.7, rol=usuario)
```

Si `bm25=0` en todos los queries, el índice Oracle Text no está indexando correctamente
— verificar con `SELECT STATUS FROM USER_INDEXES WHERE INDEX_NAME='IDX_KB_CONTENT_TEXT'`.

---

### 9.11 Activar Property Graph

#### Paso 1 — Aplicar migración V2 en Oracle (Database Actions como AGENTE)

```sql
-- Pegar y ejecutar el contenido de db/migrations/oracle/V2__add_graph_tables.sql
-- Resultado esperado:
-- Table KB_GRAPH_NODES created.
-- Table KB_GRAPH_EDGES created.
-- Property graph TICKET_SYS_GRAPH created.
```

#### Paso 2 — Ingestar el grafo desde WSL

```bash
cd ~/stack_ticket/ticket-agent
source .venv/bin/activate

python -m src.main ingest --source graph
# Log esperado:
# [ingest graph] Parseando ./docs/arquitectura/ecosistema.mmd...
# [ingest graph] 18 nodos, 27 aristas encontrados.
# Oracle: 18 nodos + 27 aristas persistidos en KB_GRAPH
# [ingest graph] Listo. Grafo persistido en KB_GRAPH_NODES/EDGES.
```

#### Paso 3 — Validar queries de arquitectura

```bash
# Dependencias de un servicio (keywords: "usa", "utiliza")
python -m src.main query \
  --query "¿Qué servicios usa ticket_agent?" --rol soporte

# Impacto ante fallo (keywords: "cae", "impacto")
python -m src.main query \
  --query "¿Qué impacto tiene si cae classifier_redis?" --rol soporte

# Almacenamiento (keyword: "almacena")
python -m src.main query \
  --query "¿Qué almacena ticket_backend?" --rol soporte

# Vista de arquitectura (keyword: "arquitectura")
python -m src.main query \
  --query "Explica la arquitectura del ecosistema" --rol soporte
```

En los logs aparecerá cuando el grafo aportó contexto:
```
Graph traverse: N relaciones añadidas al contexto
```

---

## Notas y decisiones

| Fecha | Decisión | Razón |
|---|---|---|
| 2026-05-24 | Se embeben padres E hijos (no solo hijos) | `EMBEDDING VECTOR(1024,FLOAT32)` es NOT NULL en la tabla; hacer nullable requería ALTER TABLE. El sobrecoste es ~25% más embeddings; los padres nunca se recuperan por vector (WHERE IS_PARENT=0) |
| 2026-05-24 | `retrieve()` usa `KB_CHUNKS c LEFT JOIN KB_CHUNKS p` en lugar de `V_KB_LEAF_CHUNKS` | El LEFT JOIN necesita acceso a `PARENT_ID` del hijo — más claro y explícito que depender de que la vista exponga esa columna |
| 2026-05-24 | `_to_oracle_text_query()` usa NFKD + ASCII + OR explícito | Oracle Text trata `¿`, `á`, `ñ`, `?` como operadores → DRG-50901. NFKD normaliza acentos, ASCII strip elimina chars especiales, OR explícito evita AND implícito (que daba bm25=0) |
| 2026-05-24 | `RETRIEVAL_K=6` con parent-child (default era 3) | Con PARENT_CHUNK_SIZE=2048, docs de ~4000 chars se parten en 2 padres. K=3 seleccionaba solo hijos del primer padre → respuesta incompleta |
| 2026-05-22 | `OracleVSAdapter` usa `oracledb` directo, no `langchain-community OracleVS` | LangChain OracleVS almacena metadata en JSON CLOB — impide filtrar por rol a nivel SQL |
| 2026-05-22 | Una sola tabla `KB_CHUNKS` como base — tablas especializadas en V3 | V3 agrega KB_MANUALS/KB_RUNBOOKS/KB_INCIDENTS/KB_ERROR_CATALOG; KB_CHUNKS sigue para docs usuario + helpContent |
| 2026-05-22 | `ROL_JSON` almacenado como tipo nativo `JSON` de Oracle 23ai | Permite `JSON_EXISTS` con full predicate pushdown — más eficiente que `LIKE '%usuario%'` |
| 2026-05-22 | Índice HNSW con `TARGET ACCURACY 95` | Balance entre velocidad y precisión; ajustable si latencia es inaceptable |
| 2026-05-22 | Padres almacenados con `EMBEDDING NULL` | Solo los hijos se recuperan por similitud; el padre es contexto, no punto de búsqueda |
| 2026-05-22 | Factory pattern se implementa junto con `OracleVSAdapter`, no antes | El acoplamiento actual (FAISS en `main.py`) es mínimo y está en un solo lugar — no justifica la abstracción anticipada |
| 2026-05-27 | `graph_traverse` como nodo opcional en LangGraph (`graph_store=None`) | Con FAISS el flujo es idéntico al anterior; el nodo solo existe cuando `VECTOR_STORE=oracle`. Evita dos versiones del grafo LangGraph |
| 2026-05-27 | `graph_traverse` falla silenciosamente si KB_GRAPH_NODES no existe | El agente no debe degradarse si el grafo no se ha ingestado — devuelve contexto vacío y continúa con retrieval vectorial |
| 2026-05-27 | Keyword routing con set intersection (`_needs_graph`) — no LLM routing | El LLM routing añadiría una llamada extra por query. La intersección de keywords es O(n) y suficiente para distinguir queries estructurales de semánticas |
| 2026-05-27 | Migración Flyway `V1__init_kb.sql` documenta schema aplicado manualmente | Establece el baseline para que Flyway pueda rastrear migraciones futuras sin re-ejecutar lo que ya está en producción |
| 2026-05-27 | `docs/arquitectura/ecosistema.mmd` como fuente canónica del grafo | El diagrama es la documentación de arquitectura Y la fuente de datos del Property Graph — una sola fuente de verdad |
