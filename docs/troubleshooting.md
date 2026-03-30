# Troubleshooting Guide

---

## Quick Diagnostic Checklist

```powershell
# Run from PowerShell — checks all services in one shot
$services = @(
    @{Name="Ollama";      URL="http://127.0.0.1:11434/api/tags"},
    @{Name="GenieAPI";    URL="http://127.0.0.1:8912/v1/models"},
    @{Name="Open WebUI";  URL="http://127.0.0.1:8080"},
    @{Name="LiteLLM";     URL="http://127.0.0.1:4000/health"}
)

foreach ($svc in $services) {
    try {
        $r = Invoke-WebRequest $svc.URL -TimeoutSec 3 -UseBasicParsing -EA Stop
        Write-Host "OK  $($svc.Name)" -ForegroundColor Green
    } catch {
        Write-Host "FAIL $($svc.Name) — $_" -ForegroundColor Red
    }
}
```

---

## NPU Issues

### NPU not detected

```
Symptom: foundry service status → NPU Detected: false
         GenieAPIService → QnnHtp.dll load error
```

**Fix:**
```powershell
# 1. Check device manager
Get-PnpDevice | Where-Object {$_.FriendlyName -match "neural|NPU|hexagon|qualcomm"}

# 2. If not found — install driver
# Settings → Windows Update → Advanced Options → Optional Updates
# Look for: Qualcomm Neural Processing Unit driver

# 3. Verify OS version (24H2 required)
[System.Environment]::OSVersion.Version
# Major=10, Build must be 26100+
```

### Error 1002 — HTP Protected Domain limit

```
Symptom: GenieAPIService crashes on load with "Error 1002"
Root cause: HTP Protected Domain memory exhausted (hardware limit)
```

**Fix:** Only one QNN model may be loaded at a time on Snapdragon X Elite.
```powershell
# Kill any existing GenieAPIService instances
Stop-Process -Name GenieAPIService -Force -ErrorAction SilentlyContinue
Start-Sleep 3
# Then restart with single model
cd C:\AI\GenieAPIService_cpp
.\GenieAPIService.exe -c models\llama3.1-8b-8380-qnn2.38\config.json -l -d 3 -p 8912
```

### Slow first load (60+ seconds)

Normal — cold NPU initialization. The Hexagon v73 compiles graph shards on first load.
Subsequent loads use the compiled cache: typically 4–6 seconds.

---

## Ollama Issues

### Port 11434 not responding

```powershell
# Check if process is running
Get-Process | Where-Object {$_.Name -match "ollama"}

# Check system tray — Ollama has a tray icon
# If missing: Start Menu → Ollama → click to start

# Verify OLLAMA_MODELS env var (must be Machine-level, not User-level)
[System.Environment]::GetEnvironmentVariable("OLLAMA_MODELS", "Machine")
# If null → set it and reboot
[System.Environment]::SetEnvironmentVariable(
  "OLLAMA_MODELS", "C:\AI\models\ollama",
  [System.EnvironmentVariableTarget]::Machine)
```

### Open WebUI cannot see Ollama models

```
Root cause: CORS origin mismatch between Open WebUI and Ollama
```

```powershell
# Check current OLLAMA_ORIGINS (Machine level)
[System.Environment]::GetEnvironmentVariable("OLLAMA_ORIGINS", "Machine")

# Fix: add Open WebUI origin
[System.Environment]::SetEnvironmentVariable(
  "OLLAMA_ORIGINS",
  "http://localhost:8080,http://127.0.0.1:8080",
  [System.EnvironmentVariableTarget]::Machine)

Restart-Computer
```

---

## Open WebUI Issues

### `pip install open-webui` fails on Windows ARM64

```
Error: No matching distribution found for [some-c-extension-package]
```

**Fix:** Use `uvx` isolated environment (bypasses ARM64 wheel gaps):

```powershell
pip install uv
$env:DATA_DIR = "C:\AI\openwebui"
uvx --python 3.11 open-webui serve
```

### WebUI starts but shows no models

1. Admin Panel → Settings → Connections
2. Verify Ollama URL: `http://localhost:11434`
3. Click **Verify Connection** — should show green checkmark
4. If red: check Ollama CORS settings (see above)

### RAG returns no results / empty context

```powershell
# 1. Verify embedding model is running
Invoke-RestMethod -Uri "http://localhost:11434/api/tags" | 
  Select-Object -ExpandProperty models | 
  Where-Object {$_.name -match "nomic"}
# Must return: nomic-embed-text:latest

# 2. Check ChromaDB health
Invoke-RestMethod "http://localhost:8001/api/v1/heartbeat"

# 3. Check index size (indicator of successful embedding)
Get-ChildItem "C:\AI\openwebui" -Filter "chroma.sqlite3" -Recurse |
  Select-Object Length, LastWriteTime
# If size is 0 or very small: re-ingest documents
```

---

## WSL2 / Docker Issues

### Docker containers not starting after boot

```powershell
# Check WSL2 is running
wsl --list --running
# If Ubuntu-24.04 not listed:
wsl -d Ubuntu-24.04
```

```bash
# Check Docker systemd service
systemctl status docker
systemctl is-enabled docker

# If not enabled:
sudo systemctl enable docker
sudo systemctl start docker

# Restart all AI stack containers
cd ~/ai-stack/chromadb  && docker compose up -d
cd ~/ai-stack/searxng   && docker compose up -d
cd ~/ai-stack/n8n       && docker compose --env-file .env up -d
```

### n8n cannot reach Ollama

```bash
# Test from within n8n container
docker exec n8n wget -q -O- http://host.docker.internal:11434/api/tags

# If fails — check /etc/hosts in WSL2
cat /etc/hosts | grep host.docker.internal
# If missing:
echo "$(ip route | grep default | awk '{print $3}') host.docker.internal" | \
  sudo tee -a /etc/hosts
```

### WSL2 Bridge IP changed after reboot

```bash
# Get current bridge IP
ip route | grep default | awk '{print $3}'

# Update any hardcoded IP references (e.g. in LiteLLM config)
NEW_IP=$(ip route | grep default | awk '{print $3}')
sed -i "s|http://[0-9.]*:8912|http://$NEW_IP:8912|g" ~/litellm/config.yaml
systemctl --user restart litellm-proxy
```

---

## LiteLLM Issues

### Health shows unhealthy providers

```bash
curl -s http://localhost:4000/health \
  -H "Authorization: Bearer sk-local-vivo2" | python3 -m json.tool

# Unhealthy cloud provider → check API key in config.yaml
# Unhealthy local provider → check if GenieAPIService/Ollama is running
```

### litellm-proxy not starting (WSL2 systemd)

```bash
# IMPORTANT: always --user, never sudo
systemctl --user status litellm-proxy
journalctl --user -u litellm-proxy -n 50

# Common causes:
# 1. Python venv missing
ls ~/litellm-env/bin/litellm || echo "MISSING — recreate venv"

# 2. config.yaml syntax error
cd ~/litellm && python3 -c "import yaml; yaml.safe_load(open('config.yaml'))"

# 3. Port 4000 already in use
ss -tlnp | grep 4000
```

---

## Performance Issues

### GenieAPIService slow after sleep/resume

The NPU context is not preserved through sleep. Full re-initialization on resume:

```powershell
# Kill and restart GenieAPIService after wake from sleep
Stop-Process -Name GenieAPIService -Force -ErrorAction SilentlyContinue
Start-Sleep 5
cd C:\AI\GenieAPIService_cpp
.\GenieAPIService.exe -c models\llama3.1-8b-8380-qnn2.38\config.json -l -d 3 -p 8912
```

The autostart script handles this automatically if configured. See
[windows-ai-autostart](https://github.com/gaiagent0/windows-ai-autostart).

### Ollama high memory after multiple queries

Ollama keeps models in memory with a default TTL. Force unload:

```powershell
# Unload all models (keep-alive=0)
Invoke-RestMethod -Uri "http://localhost:11434/api/generate" `
  -Method POST `
  -ContentType "application/json" `
  -Body '{"model":"llama3.1:8b","keep_alive":0}'
```
