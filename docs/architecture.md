# Architecture — Design Decisions & Rationale

> This document explains *why* each component was chosen, not just *what* it is.

---

## Core Design Principles

1. **Local-first inference** — no data leaves the machine by default
2. **ARM64-native** — avoid x86 emulation overhead where possible
3. **Layered fallback** — NPU → CPU → cloud, never the reverse
4. **Separation of concerns** — inference, embedding, frontend, orchestration are independent
5. **Reproducible** — every component has a documented install path and config

---

## Inference Tier Architecture

```
Tier 0 (NPU)  ─── GenieAPIService ─── QNN / Hexagon v73
                   Port 8912
                   ~3.9 TPS, 1.4 GB RAM, 3-5% CPU
                   ↓ fallback on unavailable
Tier 1 (CPU)  ─── Ollama ──────────── llama.cpp / ARM NEON SIMD
                   Port 11434
                   ~2.4 TPS, 5 GB RAM, 50-80% CPU
                   ↓ fallback on overload
Tier 2 (API)  ─── LiteLLM Proxy ───── Groq / Gemini / OpenRouter
                   Port 4000
                   300+ TPS (cloud), usage-limited free tiers
```

**Decision rationale — why not GPU?**
The Snapdragon X Elite has an Adreno 830 GPU but no stable GGUF/ONNX GPU backend exists
for Windows ARM64 as of early 2026. The NPU (Hexagon v73) has a mature QNN runtime with
pre-compiled models available. NPU was the correct choice.

**Decision rationale — why Ollama on CPU?**
Ollama ships ARM64 Windows binaries with NEON SIMD optimization. It handles model lifecycle
(pull/load/unload) cleanly and provides a stable OpenAI-compatible API. Alternative: llama.cpp
direct — but Ollama's daemon model management and API are worth the wrapper overhead.

---

## Frontend: Open WebUI vs Alternatives

| Option | ARM64 Windows | RAG Support | Docker-free | Verdict |
|--------|--------------|-------------|-------------|---------|
| **Open WebUI (uvx)** | ✅ Native | ✅ Full | ✅ Yes | **Chosen** |
| Open WebUI (Docker) | ✅ Via WSL2 | ✅ Full | ❌ No | Viable alternative |
| AnythingLLM | ✅ Native | ✅ Full | ✅ Yes | Good for doc-only RAG |
| Msty | ✅ Native | ⚠️ Basic | ✅ Yes | Limited RAG |
| LM Studio | ✅ Native | ❌ No | ✅ Yes | Inference-only |

**Open WebUI via `uvx`** avoids the WSL2/Docker layer for the frontend while keeping full
RAG, tool-use, and model management features. Critical ARM64 Windows issue: `pip install
open-webui` on Windows ARM64 fails due to missing binary wheels for some C-extension
dependencies. The `uvx` isolated environment resolves this transparently.

---

## RAG Pipeline Design

```
Documents (MD, PDF, TXT)
       ↓
Open WebUI ingestion
       ↓
Chunking: 500 tokens, 50 overlap, Markdown Header Text Splitter
       ↓
Embedding: nomic-embed-text (Ollama, 768-dim)
       ↓
Vector Store: ChromaDB (chroma.sqlite3, persistent)
       ↓
Retrieval: Hybrid Search (BM25 65% + vector 35%), Top-K=6
       ↓
Context injection → LLM (GenieAPIService NPU)
```

**Why nomic-embed-text over default SentenceTransformers?**
- Runs via Ollama (already in stack, no extra process)
- ARM64-optimized via llama.cpp backend
- 768-dim vs 384-dim default → better semantic resolution
- Consistent with inference stack (single model server for both chat + embeddings)

**Known limitation: "Lost in the Middle" with 8B models**
LLMs in the 7–8B parameter range struggle to extract specific targeted facts when the
answer is not at the beginning or end of the context window. This is an architectural
ceiling of the generation layer, not a retrieval problem. Retrieval correctness can be
verified independently (top-K documents contain the answer) while generation fails to
surface it. Resolution: Qwen3-Reranker as a pre-generation reranking step.

---

## WSL2 Role

WSL2 hosts Docker-based services that either:
1. Have no stable Windows ARM64 native binary (n8n, SearXNG)
2. Benefit from Linux-native networking (ChromaDB, LiteLLM)
3. Require systemd service management for reliability

**WSL2 memory allocation rationale:**

```ini
memory=24GB   # 75% of physical RAM
swap=8GB      # overflow buffer for model loading
processors=8  # 2/3 of cores, leaves 4 for Windows + NPU scheduler
```

The remaining 8 GB physical RAM is reserved for Windows + GenieAPIService NPU model
(1.4 GB) + Open WebUI + Ollama hot model (~5 GB).

---

## LiteLLM Proxy Role

LiteLLM sits between Open WebUI and the inference backends, providing:

1. **Model aliasing** — `fast`, `smart`, `local-npu` instead of raw model strings
2. **Automatic failover** — rate limit → next provider in 60s cooldown
3. **Free tier aggregation** — ~16,000+ req/day across Groq, Gemini, OpenRouter
4. **Single endpoint** — `http://localhost:4000/v1` regardless of actual backend

This decouples the frontend from backend specifics, enabling backend swaps without
reconfiguring Open WebUI.

---

## Security Architecture

```
Attack surface reduction:
  All services: bind 127.0.0.1 only (not 0.0.0.0)
  Windows Firewall: explicit block rules for AI ports
  No auth on Ollama/GenieAPIService — localhost-only, trusted
  Open WebUI: session-based auth for UI access
  LiteLLM: bearer token (sk-local-vivo2) for API access
  n8n: HTTP basic auth
  SSH to infrastructure: key-only, no passwords

Secret management:
  API keys in LiteLLM config.yaml (not env vars, not code)
  config.yaml in .gitignore — never committed
  Template provided: config.yaml.example (placeholders only)
```

---

## Autostart Sequence

See [windows-ai-autostart](https://github.com/gaiagent0/windows-ai-autostart) repo.

Critical ordering constraint:
```
1. Ollama must be running before Open WebUI starts
   (Open WebUI calls /api/tags at startup for model discovery)
2. GenieAPIService has no dependency on Ollama
3. LiteLLM proxy (WSL2) should start after WSL2 systemd is ready
   → handled by systemd .service unit, not Task Scheduler
```

---

## Closed Paths (What Was Tried and Rejected)

### Qwen2.5-7B NPU
- Blocker: `aimet-onnx` wheel does not exist for `aarch64`
- ONNX export from ARM64 Windows/WSL2 is not possible
- AI Hub cloud compile pipeline requires local ONNX export as first step
- **Status: Blocked upstream**

### Foundry Local (Microsoft) NPU
- `QnnHtp.dll v2.37.1` bundled vs models compiled for `v2.44.x`
- Error 14001 on model load — QNN version incompatibility
- **Status: Pending vendor fix** (`winget upgrade Microsoft.FoundryLocal`)

### Qwen2.0-7B-SSD NPU
- Error 1002: Hexagon v73 HTP Protected Domain limit exceeded
- SSD-Q1 config loads multiple graph segments simultaneously → multiple PD allocations
- **Status: Hardware limitation on 32 GB config**
