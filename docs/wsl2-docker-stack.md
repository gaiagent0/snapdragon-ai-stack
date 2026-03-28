# WSL2 Docker Stack — Setup Guide

> **Runtime:** Ubuntu 24.04 ARM64 in WSL2  
> **Services:** LiteLLM, ChromaDB, n8n, SearXNG  
> **Prerequisite:** Windows 11 24H2, WSL2 enabled

---

## WSL2 Installation

```powershell
# Admin PowerShell
wsl --install -d Ubuntu-24.04
# Reboot required

# Post-reboot: verify WSL2 (not WSL1)
wsl --list --verbose
# Expected: Ubuntu-24.04   Running   2
```

### Memory Configuration

```powershell
notepad "$env:USERPROFILE\.wslconfig"
```

```ini
[wsl2]
memory=24GB
processors=8
swap=8GB
localhostForwarding=true
```

```powershell
wsl --shutdown
# Wait 8 seconds, then:
wsl -d Ubuntu-24.04
```

### Enable systemd

```bash
sudo tee /etc/wsl.conf << 'EOF'
[boot]
systemd=true

[automount]
options = "metadata"
EOF
```

```powershell
# From Windows:
wsl --shutdown
# Wait, then reconnect
wsl -d Ubuntu-24.04
```

```bash
# Verify
systemctl --version
# systemd 255+
```

---

## Docker Engine

```bash
# Install Docker (ARM64 auto-detected)
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify
docker run --rm hello-world
```

---

## Shared Network

All containers use a shared bridge network:

```bash
docker network create ai-network
```

---

## ChromaDB

```bash
mkdir -p ~/ai-stack/chromadb && cd ~/ai-stack/chromadb

cat > docker-compose.yml << 'EOF'
services:
  chromadb:
    image: chromadb/chroma:latest
    container_name: chromadb
    restart: unless-stopped
    ports:
      - "127.0.0.1:8001:8000"
    volumes:
      - /mnt/c/AI/chromadb:/chroma/chroma
    environment:
      - IS_PERSISTENT=TRUE
      - ANONYMIZED_TELEMETRY=FALSE
    networks:
      - ai-network

networks:
  ai-network:
    external: true
EOF

docker compose up -d
```

Vector data is persisted to `C:\AI\chromadb\` on the Windows side for portability.

---

## SearXNG (Local Web Search)

```bash
mkdir -p ~/ai-stack/searxng/searxng-config
cat > ~/ai-stack/searxng/searxng-config/settings.yml << 'EOF'
use_default_settings: true
server:
  secret_key: "CHANGE_THIS_RANDOM_KEY_32_CHARS"
  limiter: false
search:
  safe_search: 0
  formats:
    - html
    - json
EOF

cat > ~/ai-stack/searxng/docker-compose.yml << 'EOF'
services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    ports:
      - "127.0.0.1:8081:8080"
    volumes:
      - ./searxng-config:/etc/searxng:rw
    environment:
      - SEARXNG_BASE_URL=http://localhost:8081/
    networks:
      - ai-network

networks:
  ai-network:
    external: true
EOF

cd ~/ai-stack/searxng && docker compose up -d
# Test: http://localhost:8081
```

---

## n8n + PostgreSQL

```bash
mkdir -p ~/ai-stack/n8n && cd ~/ai-stack/n8n
mkdir -p /mnt/c/AI/n8n/local-files

N8N_KEY=$(openssl rand -hex 32)
DB_PASS=$(openssl rand -hex 16)

cat > .env << EOF
GENERIC_TIMEZONE=Europe/Budapest
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678/
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=CHANGE_THIS_PASSWORD
N8N_ENCRYPTION_KEY=${N8N_KEY}
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=${DB_PASS}
EOF

cat > docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:16-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=${DB_POSTGRESDB_DATABASE}
      - POSTGRES_USER=${DB_POSTGRESDB_USER}
      - POSTGRES_PASSWORD=${DB_POSTGRESDB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - ai-network

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${DB_POSTGRESDB_DATABASE}
      - DB_POSTGRESDB_USER=${DB_POSTGRESDB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}
      - N8N_BASIC_AUTH_ACTIVE=${N8N_BASIC_AUTH_ACTIVE}
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=${N8N_PORT}
      - WEBHOOK_URL=${WEBHOOK_URL}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - OLLAMA_HOST=http://host.docker.internal:11434
    volumes:
      - n8n_data:/home/node/.n8n
      - /mnt/c/AI/n8n/local-files:/home/node/local-files
    depends_on:
      postgres:
        condition: service_healthy
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      - ai-network

volumes:
  postgres_data:
  n8n_data:

networks:
  ai-network:
    external: true
EOF

docker compose --env-file .env up -d
```

---

## Service Health Check

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Expected output:
```
NAMES        STATUS         PORTS
chromadb     Up X minutes   0.0.0.0:8001->8000/tcp
searxng      Up X minutes   0.0.0.0:8081->8080/tcp
n8n          Up X minutes   0.0.0.0:5678->5678/tcp
n8n-postgres Up X minutes
```

---

## Port Reference

| Service | Host Port | Container Port | Auth |
|---------|-----------|---------------|------|
| ChromaDB | 8001 | 8000 | None (localhost-only) |
| SearXNG | 8081 | 8080 | None (localhost-only) |
| n8n | 5678 | 5678 | HTTP Basic |

All ports bound to `127.0.0.1` — not accessible from external network.
