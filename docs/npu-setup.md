# NPU Setup — GenieAPIService on Snapdragon X Elite

> **Inference backend:** Qualcomm QNN (Hexagon v73)  
> **Driver requirement:** Windows 11 24H2, NPU Driver 30.0.140.1000+

---

## Prerequisites

### 1. Verify NPU Driver

```powershell
# Check NPU device presence
Get-PnpDevice | Where-Object {$_.FriendlyName -match "neural|NPU|hexagon|qualcomm"}
# Expected: "Qualcomm(R) AI Engine / Neural Processing Unit" — Status: OK

# If not found: Settings → Windows Update → Advanced Options → Optional Updates
# Look for: "Qualcomm Neural Processing Unit" driver package
```

### 2. Verify Windows Version

```powershell
winver
# Minimum: Windows 11 24H2 (Build 26100)
# The NPU driver is included in 24H2 — earlier builds have no NPU support
```

---

## GenieAPIService Setup

### Directory Structure

```
C:\AI\GenieAPIService_cpp\
├── GenieAPIService.exe       ← Main binary
├── QnnHtp.dll                ← Hexagon v73 QNN runtime
├── QnnSystem.dll
└── models\
    ├── llama3.1-8b-8380-qnn2.38\
    │   ├── config.json
    │   ├── tokenizer.json
    │   ├── htp_backend_ext_config.json
    │   └── llama_v3_1_8b_chat_quantized_part_[1-5]_of_5.bin
    └── Llama3.2-3B\
        └── ...
```

### Download Models

Models are pre-compiled for Snapdragon 8380 (X Elite) / QNN 2.38:

```powershell
$base = "https://www.aidevhome.com/data/adh2/models/suggested"

# Llama 3.1 8B — primary model
Invoke-WebRequest "$base/llama3.1-8b-8380-qnn2.38.zip" -OutFile "C:\AI\llama31-8b.zip"
Expand-Archive "C:\AI\llama31-8b.zip" -DestinationPath "C:\AI\GenieAPIService_cpp\models\"

# Llama 3.2 3B — lighter model
Invoke-WebRequest "$base/llama3.2-3b-8380-qnn2.37.zip" -OutFile "C:\AI\llama32-3b.zip"
Expand-Archive "C:\AI\llama32-3b.zip" -DestinationPath "C:\AI\GenieAPIService_cpp\models\"
```

### Start NPU Server

```powershell
cd C:\AI\GenieAPIService_cpp

# Full flags explained:
#   -c  : model config.json path
#   -l  : enable logging
#   -d 3: debug level (0=silent, 3=info, 5=verbose)
#   -p  : API port

.\GenieAPIService.exe `
  -c models\llama3.1-8b-8380-qnn2.38\config.json `
  -l -d 3 -p 8912
```

### Verify NPU Load

Successful startup log pattern:

```
Backend library: QnnHtp.dll
QnnDevice_create done. device = 0x1
Key: Weight Sharing, Value: true
Allocated total size = 306545152   ← 292 MB NPU protected domain
load successfully! use second: 4.03
Server listening on port 8912
```

> **If you see** `Initialization Time Acceleration not possible` — this is a QNN DLL minor
> version mismatch (non-blocking). Expect ~20–30% TPS degradation vs optimal.

---

## API Usage

GenieAPIService exposes an **OpenAI-compatible** REST API.

### List Models

```powershell
Invoke-RestMethod http://localhost:8912/v1/models
```

### Chat Completion

```powershell
$body = @{
    model    = "llama3.1-8b-8380-qnn2.38"
    messages = @(@{role = "user"; content = "Explain ARM64 SIMD in one paragraph."})
    max_tokens = 200
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
    -Uri "http://localhost:8912/v1/chat/completions" `
    -Method POST `
    -Body $body `
    -ContentType "application/json"
```

### Python Client

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8912/v1",
    api_key="not-required"  # GenieAPIService needs no auth key
)

response = client.chat.completions.create(
    model="llama3.1-8b-8380-qnn2.38",
    messages=[{"role": "user", "content": "Hello from NPU!"}],
    max_tokens=100
)
print(response.choices[0].message.content)
```

---

## Known Quirks

These are GenieAPIService implementation artifacts — non-blocking, safe to ignore:

| Field | Anomaly | Root Cause |
|-------|---------|-----------|
| `response.model` | Empty string `""` | Not populated by GenieAPIService |
| `response.created` | Negative integer e.g. `-590142444` | int32 timestamp overflow |
| `response.usage.total_tokens` | Always `0` | Token counting not implemented |

---

## Performance Tuning

### TPS Optimization via Context Window

In the model `config.json`, the `size` parameter controls KV cache allocation:

```json
{
  "size": 2048
}
```

Setting `"size": 2048` (from default 4096) yields **~20–30% TPS improvement** by reducing
HTP memory bandwidth pressure. Trade-off: maximum context length halved.

Benchmark results (Llama 3.1 8B, Snapdragon X Elite X1E78100):

| Config size | TPS | Max Context |
|-------------|-----|-------------|
| 4096 (default) | ~3.9 | 4096 tokens |
| 2048 (optimized) | ~4.8–5.1 | 2048 tokens |

### NPU Protected Domain Memory

The Hexagon v73 HTP Protected Domain is limited. Loading multiple models simultaneously
will hit `Error 1002: HTP Protected Domain limit`. Load one model at a time.

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `NPU Detected: false` | Missing driver | `Windows Update → Optional Updates → Qualcomm NPU` |
| `Error 1002` | Protected domain OOM | Unload other NPU models, restart service |
| `load time > 60s` | Cold NPU init | Normal on first load — cached on subsequent starts |
| `QnnHtp.dll not found` | DLL not in working directory | Run from `GenieAPIService_cpp\` dir |
| Port 8912 already in use | Previous instance running | `Stop-Process -Name GenieAPIService` |
