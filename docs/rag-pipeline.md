# RAG Pipeline — Architecture & Configuration

> **Stack:** Open WebUI + ChromaDB + nomic-embed-text (Ollama)  
> **Use case:** Local document retrieval over homelab/infrastructure docs  
> **Tested corpus:** 35+ Markdown files, ~40 MB, README/runbook format

---

## Pipeline Architecture

```
┌─────────────────── INGESTION ───────────────────┐
│                                                   │
│  Documents (MD, PDF, TXT)                        │
│       ↓                                           │
│  Markdown Header Text Splitter                   │
│  chunk_size=500, overlap=50                      │
│       ↓                                           │
│  nomic-embed-text (Ollama, 768-dim)              │
│       ↓                                           │
│  ChromaDB (chroma.sqlite3, persistent)           │
│                                                   │
└───────────────────────────────────────────────────┘

┌─────────────────── RETRIEVAL ───────────────────┐
│                                                   │
│  User Query                                       │
│       ↓                                           │
│  Hybrid Search:                                   │
│    BM25 (lexical)  ~65% weight                   │
│    Vector cosine   ~35% weight                   │
│       ↓                                           │
│  Top-K = 6 chunks                                │
│  Enrich Hybrid Search Text: enabled              │
│       ↓                                           │
│  Context injection → LLM                         │
│                                                   │
└───────────────────────────────────────────────────┘
```

---

## Open WebUI RAG Settings

Navigate to: **Admin Panel → Settings → Documents**

| Setting | Value | Rationale |
|---------|-------|-----------|
| Embedding model | `nomic-embed-text` (Ollama) | ARM64-native, 768-dim, no Python deps |
| Vector database | ChromaDB | Persistent SQLite backend, simple ops |
| Chunk size | 500 | Optimal for technical docs with code blocks |
| Chunk overlap | 50 | 10% overlap preserves context at boundaries |
| Text splitter | Markdown Header Text Splitter | Respects document structure |
| Retrieval method | Hybrid Search | BM25 + vector, best for technical terminology |
| Top-K | 6 | Sufficient context without overwhelming 8K window |
| Enrich Hybrid Search | Enabled | Query expansion improves recall on terse queries |

**RAG system prompt (strict mode — no prior knowledge fallback):**

```
You are a documentation assistant. Answer ONLY using the provided context.
If the answer is not in the context, say "Not found in documentation."
Do not use any prior knowledge outside of the retrieved context.
```

---

## Embedding Model: nomic-embed-text

```bash
# Pull via Ollama
ollama pull nomic-embed-text

# Verify
curl -s http://localhost:11434/api/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model": "nomic-embed-text", "prompt": "test query"}' | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print('Dims:', len(d['embedding']))"
# Expected: Dims: 768
```

### Why nomic-embed-text over alternatives?

| Model | Dims | ARM64 | Install | Performance |
|-------|------|-------|---------|-------------|
| **nomic-embed-text** | 768 | ✅ via Ollama | Simple | Good for English tech docs |
| all-MiniLM-L6-v2 | 384 | ⚠️ Python wheel | Complex | Baseline |
| text-embedding-3-small | 1536 | ✅ (cloud) | API key | Best quality, not local |
| mxbai-embed-large | 1024 | ✅ via Ollama | Simple | Slightly better MTEB score |

---

## ChromaDB Configuration

```bash
# Data persisted to Windows-side path via WSL2 mount
# C:\AI\chromadb\ ↔ /mnt/c/AI/chromadb/

# Health check
curl http://localhost:8001/api/v1/heartbeat
# {"nanosecond heartbeat": <timestamp>}

# List collections
curl http://localhost:8001/api/v1/collections

# Index completeness check (PowerShell)
$vdb = "C:\AI\openwebui"
Get-ChildItem $vdb -Recurse -Filter "chroma.sqlite3" | 
  Select-Object Name, Length, LastWriteTime
```

> **Authoritative completeness indicator:** `chroma.sqlite3` file size.  
> After re-indexing 35 docs (~40 MB), expect ~15–25 MB SQLite file.

---

## Document Ingestion

### Via Open WebUI UI

1. Open WebUI → **+** (new chat) → **Documents** icon
2. Upload files or paste folder path
3. Wait for embedding progress bar
4. Verify in Admin Panel → Documents list

### Via API (bulk ingest)

```python
import requests
import os
from pathlib import Path

WEBUI_URL = "http://localhost:8080"
AUTH_TOKEN = "your-session-token"  # from /api/v1/auths/signin

def get_token():
    resp = requests.post(f"{WEBUI_URL}/api/v1/auths/signin", json={
        "email": "your@email.com",
        "password": "your-password"
    })
    return resp.json()["token"]

def upload_document(filepath: Path, token: str):
    with open(filepath, "rb") as f:
        resp = requests.post(
            f"{WEBUI_URL}/api/v1/files/",
            headers={"Authorization": f"Bearer {token}"},
            files={"file": (filepath.name, f, "text/markdown")}
        )
    return resp.json()

token = get_token()
docs_dir = Path("C:/AI/homelab-docs")

for md_file in docs_dir.glob("*.md"):
    result = upload_document(md_file, token)
    print(f"Uploaded: {md_file.name} → ID: {result.get('id')}")
```

---

## Known Limitations

### "Lost in the Middle" — Generation Failure with 8B Models

**Symptom:** Retrieval returns correct chunks (verifiable), but the model generates
summary-level or generic content instead of the specific targeted section.

**Root cause:** LLMs in the 7–8B parameter range exhibit a well-documented "lost in the
middle" degradation — attention scores drop for context positioned in the middle of a long
prompt. With Top-K=6 chunks, the relevant chunk may occupy the 3rd or 4th position.

**Diagnosis:** This is a **generation layer ceiling**, not a retrieval failure.
Retrieval can be verified independently:

```python
# Direct ChromaDB query to verify retrieval quality
import chromadb

client = chromadb.PersistentClient(path="C:/AI/openwebui")
collection = client.get_collection("your-collection-name")

results = collection.query(
    query_texts=["your test query"],
    n_results=6
)
for i, doc in enumerate(results["documents"][0]):
    print(f"Chunk {i+1}: {doc[:200]}")
# If the answer exists in any chunk → retrieval is correct, generation is the bottleneck
```

**Resolution path:** Qwen3-Reranker as pre-generation reranking step.
Download: `https://www.aidevhome.com/data/adh2/models/suggested/qwen3-reranker-8380-2.38.zip`

---

## Regression Test Queries

Use these to baseline RAG behavior after config changes:

| Query | Expected answer source | Tests |
|-------|----------------------|-------|
| "What is the Ollama embedding model used?" | `README-ai-stack-vivo2.md` | Direct fact retrieval |
| "How to check if Docker is running in WSL2?" | `ai-stack-install-guide.md` | Command retrieval |
| "What is the LiteLLM API key?" | `README-litellm-multi-provider.md` | Sensitive fact (should NOT be in public docs) |

> Re-run after any chunking, embedding, or retrieval parameter change.
