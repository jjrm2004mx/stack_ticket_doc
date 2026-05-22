# PROMPT FINAL: RAG Manuales MVP — Agnóstico con migración a Oracle 23ai

## Para Claude Code en VS Code

---

## CONTEXTO DEL PROYECTO

**Proyecto:** `ticket-agent`
**Objetivo:** Agente RAG de conocimiento para el ecosistema de tickets. Responde preguntas
de usuarios finales (Usuario, Manager, Admin) sobre cómo usar ticket-management,
y preguntas técnicas del personal de soporte sobre todos los repos del ecosistema.
Usa búsqueda semántica + generación con LLMs locales y cloud.

**Arquitectura objetivo (largo plazo):**
- Oracle Database 23ai como Knowledge Repository (Vector Tables + Property Graph)
- Docling para parsing de documentos complejos (PDF, DOCX, YAML)
- Parent-Child contextual chunking
- Hybrid Search (Vector + Oracle Text HNSW)
- LangGraph Agentic RAG con HyDE, Multi-Query RRF, CRAG
- Graph Traversal con PGQL/SQL PGQ

**MVP (Fase 1):** FAISS local + ONNX embeddings + Ollama/OCI
**Restricción crítica:** el modelo de embeddings elegido en Fase 1 es definitivo —
no cambia en la migración a Oracle 23ai para evitar re-ingesta total.

---

## ARQUITECTURA DE FUENTES DE CONOCIMIENTO

El agente ingesta documentos de tres fuentes distintas. Cada fuente tiene audiencia y ruta definida:

### Fuente 1 — Documentación por rol (usuarios finales)
```
ticket-management/docs/usuarios/usuario/    → rol: usuario
ticket-management/docs/usuarios/manager/    → rol: manager
ticket-management/docs/usuarios/admin/      → rol: admin
```
Archivos MD creados a partir del codebase real. Cubren flujos, estados, permisos,
configuración y glosario específicos para cada rol.

### Fuente 2 — Ayuda contextual de pantallas (helpContent.ts)
```
ticket-management/frontend/src/components/help/helpContent.ts
```
Se serializa al vuelo en el momento de ingesta — NO se crea un MD intermedio.
Una función `build_help_context(role)` filtra las secciones por rol y genera texto plano
que se ingesta como chunks con metadata `{source: helpContent, screen: X, rol: Y}`.

Pantallas disponibles con ayuda: newTicket, ticketDetail, attachments, ticketEvents,
draftReview, classificationManagement, userManagement, permissionManagement.

### Fuente 3 — Runbooks técnicos (soporte)
```
ticket-ingestion-light/docs/runbooks/       → soporte
ticket-classification/docs/runbooks/        → soporte
notification-service/docs/runbooks/         → soporte
infra-monitoring/docs/runbooks/             → soporte
ticket-management/docs/runbooks/            → soporte
```

### Filtrado por audiencia al hacer query

| Rol del usuario | Fuentes que usa el agente |
|---|---|
| usuario | docs/usuarios/usuario/ + helpContent.ts filtrado por rol |
| manager | docs/usuarios/manager/ + helpContent.ts filtrado por rol |
| admin | docs/usuarios/admin/ + helpContent.ts filtrado por rol |
| soporte | todas las fuentes anteriores + todos los docs/runbooks/ |

El filtrado se implementa via metadata en los chunks: `{rol: [...], tipo: ..., fuente: ...}`.

---

## DECISIÓN DE EMBEDDINGS — NO NEGOCIABLE

| Parámetro | Valor |
|---|---|
| Modelo ONNX | `mxbai-embed-large-v1` |
| Dimensiones | **1024 dims** |
| Tipo | FLOAT32 |
| Razón | Compatible con Oracle 23ai Hybrid Vector Index (FLOAT32, 1024 dims) desde Día 1 |

**No usar** `all-MiniLM-L6-v2` (384 dims) ni `nomic-embed-text` (768 dims).
Si el modelo ONNX cambia, se requiere re-ingestar todo el vector store.

---

## STACK MVP (Fase 1)

| Componente | Tecnología |
|---|---|
| LLM local | Ollama (`llama3.2`, `mistral`, etc.) |
| LLM cloud | OCI GenAI (`meta.llama-3.3-70b-instruct`) |
| Embeddings | ONNX local (`mxbai-embed-large-v1`, 1024 dims) |
| Vector Store | FAISS-cpu (persistido en disco) |
| Chunking | RecursiveCharacterTextSplitter (flat, 512 tok) |
| Parser | Carga directa de `.md` via LangChain |
| Orquestación | LangChain LCEL (sin LangGraph en MVP) |
| CLI | `python -m src.main ingest / query --rol <rol>` |
| Filtrado | Metadata por rol en cada chunk (`rol`, `tipo`, `fuente`, `pantalla`) |
| Serialización | helpContent.ts → texto plano en ingesta (sin MD intermedio) |

---

## ESTRUCTURA DE CARPETAS

```
ticket-agent/
├── CLAUDE.md
├── .env.example
├── requirements.txt
├── requirements-dev.txt
│
├── docs/
│   └── tecnico/
│       ├── PROMPT_FINAL.md           # Este archivo
│       └── TAREAS.md                 # Control de tareas
│
├── src/
│   ├── __init__.py
│   ├── main.py                       # CLI entry point
│   ├── config.py                     # Pydantic BaseSettings
│   │
│   ├── adapters/
│   │   ├── __init__.py
│   │   ├── base.py                   # ABC: LLMAdapter, EmbedderAdapter
│   │   ├── onnx_adapter.py           # ONNX local (mxbai-embed-large-v1)
│   │   ├── ollama_adapter.py         # ChatOllama + OllamaEmbeddings
│   │   └── oci_adapter.py            # ChatOCIGenAI + OCIGenAIEmbeddings
│   │
│   ├── document_processing/
│   │   ├── __init__.py
│   │   ├── loader.py                 # Loader multi-fuente: MD por ruta + helpContent.ts
│   │   ├── help_serializer.py        # Serializa helpContent.ts → chunks con metadata de rol
│   │   ├── chunking.py               # RecursiveCharacterTextSplitter
│   │   └── schema.py                 # Dataclasses: Document, Chunk (incluye metadata rol)
│   │
│   ├── vector_store/
│   │   ├── __init__.py
│   │   └── faiss_store.py            # FAISS wrapper (add, load, retrieve)
│   │
│   ├── rag/
│   │   ├── __init__.py
│   │   ├── retriever.py              # Vector search simple (top-K)
│   │   └── prompts.py                # Templates agnósticos
│   │
│   └── utils/
│       ├── __init__.py
│       ├── logger.py
│       └── validators.py
│
├── data/
│   ├── raw/manuales/                 # Archivos .md fuente
│   ├── processed/chunks.jsonl        # Cache de chunks
│   └── vector_db/faiss_index/        # FAISS index persistido
│
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_chunking.py
    ├── test_retriever.py
    └── test_adapters.py
```

---

## VARIABLES DE ENTORNO (.env.example)

```bash
# Provider LLM: ollama | oci
PROVIDER=ollama

# Embeddings — NO cambiar el modelo sin re-ingestar todo
EMBED_MODEL_TYPE=onnx
EMBED_MODEL_NAME=mxbai-embed-large-v1
EMBED_DIMS=1024

# Ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_LLM_MODEL=llama3.2

# OCI (opcional en Fase 1 — todos los campos son Optional)
OCI_CONFIG_FILE=~/.oci/config
OCI_PROFILE=DEFAULT
OCI_COMPARTMENT_ID=
OCI_SERVICE_ENDPOINT=https://inference.generativeai.us-chicago-1.oci.oraclecloud.com
OCI_LLM_MODEL=meta.llama-3.3-70b-instruct

# Vector Store
VECTOR_DB_PATH=./data/vector_db/faiss_index/
VECTOR_DB_CHUNKS_PATH=./data/processed/chunks.jsonl

# Fuentes de conocimiento (rutas absolutas o relativas al repo)
SOURCE_USUARIOS=../ticket-management/docs/usuarios
SOURCE_RUNBOOKS=../ticket-management/docs/runbooks,../ticket-classification/docs/runbooks,../notification-service/docs/runbooks,../infra-monitoring/docs/runbooks,../ticket-ingestion-light/docs/runbooks
SOURCE_HELP_TS=../ticket-management/frontend/src/components/help/helpContent.ts

# RAG
CHUNK_SIZE=512
CHUNK_OVERLAP=100
RETRIEVAL_K=3

# Logging
LOG_LEVEL=INFO
```

---

## ARCHIVOS CLAVE

### `src/config.py`

```python
from typing import Optional
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # LLM Provider
    PROVIDER: str = "ollama"

    # Embeddings — modelo fijo para toda la vida del índice
    EMBED_MODEL_TYPE: str = "onnx"
    EMBED_MODEL_NAME: str = "mxbai-embed-large-v1"
    EMBED_DIMS: int = 1024

    # Ollama
    OLLAMA_BASE_URL: str = "http://localhost:11434"
    OLLAMA_LLM_MODEL: str = "llama3.2"

    # OCI — Optional: solo se validan si PROVIDER=oci
    OCI_CONFIG_FILE: Optional[str] = None
    OCI_PROFILE: Optional[str] = "DEFAULT"
    OCI_COMPARTMENT_ID: Optional[str] = None
    OCI_SERVICE_ENDPOINT: Optional[str] = None
    OCI_LLM_MODEL: Optional[str] = "meta.llama-3.3-70b-instruct"

    # Vector Store
    VECTOR_DB_PATH: str = "./data/vector_db/faiss_index/"
    VECTOR_DB_CHUNKS_PATH: str = "./data/processed/chunks.jsonl"

    # RAG
    CHUNK_SIZE: int = 512
    CHUNK_OVERLAP: int = 100
    RETRIEVAL_K: int = 3

    LOG_LEVEL: str = "INFO"

    class Config:
        env_file = ".env"
```

### `src/adapters/base.py`

```python
from abc import ABC, abstractmethod
from typing import List

class LLMAdapter(ABC):
    @abstractmethod
    def invoke(self, prompt: str) -> str:
        pass

    @abstractmethod
    def can_cache(self) -> bool:
        pass

class EmbedderAdapter(ABC):
    @abstractmethod
    def embed_documents(self, texts: List[str]) -> List[List[float]]:
        pass

    @abstractmethod
    def embed_query(self, query: str) -> List[float]:
        pass
```

### `src/adapters/onnx_adapter.py`

```python
from sentence_transformers import SentenceTransformer
from .base import EmbedderAdapter
from typing import List

class ONNXEmbedderAdapter(EmbedderAdapter):
    """
    Embedder local ONNX con mxbai-embed-large-v1.
    Produce vectores de 1024 dims (FLOAT32) — compatibles con Oracle 23ai.
    """
    def __init__(self, model_name: str = "mixedbread-ai/mxbai-embed-large-v1"):
        self.model = SentenceTransformer(model_name)

    def embed_documents(self, texts: List[str]) -> List[List[float]]:
        return self.model.encode(texts, normalize_embeddings=True).tolist()

    def embed_query(self, query: str) -> List[float]:
        return self.model.encode(query, normalize_embeddings=True).tolist()
```

### `src/adapters/ollama_adapter.py`

```python
from langchain_ollama import ChatOllama
from .base import LLMAdapter

class OllamaLLMAdapter(LLMAdapter):
    def __init__(self, base_url: str, model: str):
        self.llm = ChatOllama(base_url=base_url, model=model)

    def invoke(self, prompt: str) -> str:
        return self.llm.invoke(prompt).content

    def can_cache(self) -> bool:
        return False
```

### `src/adapters/oci_adapter.py`

```python
from langchain_oci import ChatOCIGenAI
from .base import LLMAdapter

class OCILLMAdapter(LLMAdapter):
    def __init__(self, model_id: str, service_endpoint: str, compartment_id: str):
        self.llm = ChatOCIGenAI(
            model_id=model_id,
            service_endpoint=service_endpoint,
            compartment_id=compartment_id,
        )

    def invoke(self, prompt: str) -> str:
        return self.llm.invoke(prompt).content

    def can_cache(self) -> bool:
        return False
```

### `src/vector_store/faiss_store.py`

```python
from langchain_community.vectorstores import FAISS
from langchain.embeddings.base import Embeddings

class FAISSVectorStore:
    def __init__(self, embeddings: Embeddings, persist_path: str):
        self.embeddings = embeddings
        self.persist_path = persist_path
        self.store = None

    def add_chunks(self, chunks):
        texts = [c.content for c in chunks]
        metadatas = [{"source": c.source_file, "chunk_id": c.id} for c in chunks]
        self.store = FAISS.from_texts(texts, self.embeddings, metadatas=metadatas)
        self.store.save_local(self.persist_path)

    def load(self):
        # allow_dangerous_deserialization requerido en versiones recientes de langchain-community
        self.store = FAISS.load_local(
            self.persist_path,
            self.embeddings,
            allow_dangerous_deserialization=True
        )

    def retrieve(self, query: str, k: int = 3):
        return self.store.similarity_search(query, k=k)
```

### `requirements.txt`

```
# LangChain core
langchain==0.3.*
langchain-core==0.3.*
langchain-community==0.3.*
langchain-text-splitters==0.3.*

# LLM Providers
langchain-ollama
langchain-oci==0.2.5

# Vector Store
faiss-cpu==1.8.*

# Embeddings ONNX
sentence-transformers>=2.7.0
optimum[onnxruntime]>=1.19.0

# Config y utils
pydantic==2.*
pydantic-settings==2.*
python-dotenv==1.*

# Tests
pytest==7.*
pytest-mock
```

---

## FLUJO DE EJECUCIÓN (MVP)

```
1. Ingest:
   .md → loader.py → RecursiveCharacterTextSplitter (512 tok)
       → ONNXEmbedderAdapter (1024 dims) → FAISS.save_local()

2. Query:
   pregunta → embed_query() → FAISS.similarity_search(k=3)
            → prompts.py (system + chunks + pregunta)
            → LLMAdapter.invoke() → respuesta + sources
```

---

## COMANDOS

```bash
# Setup
pip install -r requirements.txt
cp .env.example .env

# Ingestar todas las fuentes
python -m src.main ingest --source all

# Ingestar solo docs de usuario (ticket-management/docs/usuarios/)
python -m src.main ingest --source usuarios

# Ingestar solo runbooks (todos los repos)
python -m src.main ingest --source runbooks

# Ingestar solo helpContent.ts
python -m src.main ingest --source help

# Query como usuario final (filtra por rol)
python -m src.main query --query "¿Cómo creo un ticket?" --rol usuario
python -m src.main query --query "¿Cómo apruebo un borrador?" --rol manager
python -m src.main query --query "¿Cómo configuro el workflow?" --rol admin

# Query como soporte (sin filtro de rol — acceso total)
python -m src.main query --query "¿Cómo está configurado n8n?" --rol soporte

# Tests
pytest tests/
```
