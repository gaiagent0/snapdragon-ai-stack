# snapdragon-ai-stack

> **Complete local AI stack for Snapdragon X Elite (ARM64 Windows)**  
> Production-ready · Privacy-first · NPU-accelerated

[![Platform](https://img.shields.io/badge/platform-Windows%2011%20ARM64-blue)](https://www.qualcomm.com/products/mobile/snapdragon/pcs-and-tablets/snapdragon-x-series)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Status](https://img.shields.io/badge/status-production-brightgreen)](docs/architecture.md)

---

## What This Is

A fully documented, production-tested local AI stack optimized for **Snapdragon X Elite** laptops (ASUS Vivobook S15, Lenovo ThinkPad T14s Gen 6, etc.). All inference runs **locally** — no data leaves your machine.

```
┌─────────────────── WINDOWS ARM64 (Native) ───────────────────┐
│  GenieAPIService   → NPU inference  (Hexagon v73, QNN 2.38)  │
│  Ollama ARM64      → CPU inference  (GGUF, NEON SIMD)        │
│  Open WebUI        → Chat frontend  (uvx, port 8080)         │
└──────────────────────────────────────────────────────────────┘
         │  WSL2 backend (Ubuntu 24.04 ARM64)
┌─────────────────── WSL2 UBUNTU 24.04 ────────────────────────┐
│  LiteLLM Proxy     → Unified API    (port 4000)              │
│  ChromaDB          → Vector store   (RAG pipeline)           │
│  n8n               → Workflow automation                     │
│  SearXNG           → Local web search                        │
└──────────────────────────────────────────────────────────────┘
```

---

## Hardware Requirements

| Component | Minimum | Tested On |
|-----------|---------|-----------|
| CPU | Snapdragon X (any) | X Elite X1E78100, 12-core Oryon @ 3.42 GHz |
| RAM | 16 GB | 32 GB |
| NPU | Hexagon v73 | Qualcomm AI Engine, Driver 30.0.140.1000 |
| OS | Windows 11 24H2 ARM64 | Build 26100+ |
| Storage | 60 GB free | 100 GB recommended |

> **Why 24H2?** The Snapdragon X Elite NPU driver ships with 24H2. Earlier builds have no NPU support.

---

## Performance Benchmarks

Measured on Llama 3.1 8B (identical model, same prompt):

| Backend | Tokens/sec | CPU Load | RAM Usage |
|---------|-----------|----------|-----------|
| **GenieAPIService (NPU)** | **~3.9 TPS** | ~3–5% | 1.4 GB |
| Ollama (CPU/NEON) | ~2.4 TPS | ~50–80% | ~5 GB |

NPU advantage: **1.6× faster, ~90% less CPU load, ~72% less RAM**.

---

## Quick Start

### Prerequisites

```powershell
# Verify Windows version
winver
# Must be: 24H2 (Build 26100+)

# Verify NPU driver
Get-PnpDevice | Where-Object {$_.FriendlyName -match "neural|NPU|hexagon|qualcomm"}
# Expected: Qualcomm(R) AI Engine / Neural Processing Unit
```

### 1. Install Ollama (CPU inference baseline)

```powershell
winget install Ollama.Ollama

# Set model storage location
[System.Environment]::SetEnvironmentVariable(
  "OLLAMA_MODELS", "C:\AI\models\ollama",
  [System.EnvironmentVariableTarget]::Machine)

# Required restart after env var change
Restart-Computer
```

### 2. Pull essential models

```powershell
# Embedding (required for RAG)
ollama pull nomic-embed-text

# General chat (CPU fallback)
ollama pull llama3.1:8b

# Code assistant
ollama pull qwen2.5-coder:7b

# Fast autocomplete
ollama pull qwen2.5-coder:1.5b
```

### 3. Install GenieAPIService (NPU inference)

See [docs/npu-setup.md](docs/npu-setup.md) for complete NPU setup.

```powershell
# Download GenieAPIService binary + Llama 3.1 8B QNN model
# Source: https://www.aidevhome.com/data/adh2/models/suggested/

# Start NPU inference server
cd C:\AI\GenieAPIService_cpp
.\GenieAPIService.exe -c models\llama3.1-8b-8380-qnn2.38\config.json -l -d 3 -p 8912
```

### 4. Install Open WebUI

```powershell
# Windows-native install via uvx (no Docker required on Windows)
pip install uv

$env:DATA_DIR = "C:\AI\openwebui"
uvx --python 3.11 open-webui serve
# Access: http://localhost:8080
```

### 5. WSL2 + Docker stack (LiteLLM, ChromaDB, n8n)

See [docs/wsl2-docker-stack.md](docs/wsl2-docker-stack.md).

---

## Architecture Deep-Dive

| Document | Description |
|----------|-------------|
| [docs/architecture.md](docs/architecture.md) | Full stack design decisions |
| [docs/npu-setup.md](docs/npu-setup.md) | GenieAPIService + QNN model setup |
| [docs/rag-pipeline.md](docs/rag-pipeline.md) | ChromaDB + nomic-embed RAG config |
| [docs/wsl2-docker-stack.md](docs/wsl2-docker-stack.md) | WSL2, Docker, LiteLLM, n8n |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Known issues + fixes |

---

## Stack Components

### Windows-Native (ARM64)

| Component | Version | Port | Role |
|-----------|---------|------|------|
| GenieAPIService | QNN 2.38 | 8912 | NPU inference server |
| Ollama | Latest | 11434 | CPU inference (GGUF) |
| Open WebUI | Latest | 8080 | Chat UI + RAG frontend |

### WSL2 / Docker

| Component | Port | Role |
|-----------|------|------|
| LiteLLM Proxy | 4000 | Unified API, fallback routing |
| ChromaDB | 8001 | Vector database |
| n8n | 5678 | Workflow automation |
| SearXNG | 8080 | Local web search |

---

## Known Limitations

| Issue | Root Cause | Workaround |
|-------|-----------|------------|
| `"model": ""` in response | GenieAPIService quirk | Ignore, functional |
| `"created": -590142444` negative timestamp | int32 overflow in GenieAPIService | Ignore |
| `Initialization Time Acceleration not possible` | QNN DLL minor version mismatch | ~20–30% TPS loss, non-blocking |
| Ollama = CPU only on Windows ARM64 | No GPU/NPU backend in llama.cpp WinARM | Use GenieAPIService for NPU |
| Open WebUI pip install fails on ARM64 Windows | No ARM64 wheel | Use uvx method (see docs) |

---

## Supported NPU Models (aidevhome.com — Genie format)

| Model | File | Size | Status |
|-------|------|------|--------|
| Llama 3.1 8B | `llama3.1-8b-8380-qnn2.38.zip` | ~4.9 GB | ✅ Tested |
| Llama 3.2 3B | `llama3.2-3b-8380-qnn2.37.zip` | ~2.0 GB | ✅ Tested |
| Qwen2.5-VL 3B | `qwen2.5vl3b-8380-2.42.zip` | ~2.1 GB | ⬜ Vision |
| Qwen3-Reranker | `qwen3-reranker-8380-2.38.zip` | ~1.5 GB | ⬜ RAG reranker |

Base URL: `https://www.aidevhome.com/data/adh2/models/suggested/`

---

## Security Model

All services bind to `127.0.0.1` by default — no external exposure.

```powershell
# Verify nothing is externally exposed
netstat -an | findstr "LISTENING" | findstr -v "127.0.0.1"
# Should return empty for AI ports (8912, 11434, 8080, 4000)
```

Firewall rules are enforced via the autostart script. See [windows-ai-autostart](https://github.com/your-username/windows-ai-autostart).

---

## Related Repositories

| Repo | Description |
|------|-------------|
| [litellm-local-config](https://github.com/your-username/litellm-local-config) | LiteLLM multi-provider proxy templates |
| [windows-ai-autostart](https://github.com/your-username/windows-ai-autostart) | Task Scheduler autostart automation |

---

## License

MIT — see [LICENSE](LICENSE).

---

*Tested on: ASUS Vivobook S15 · Snapdragon X Elite X1E78100 · 32 GB RAM · Windows 11 24H2 ARM64*
