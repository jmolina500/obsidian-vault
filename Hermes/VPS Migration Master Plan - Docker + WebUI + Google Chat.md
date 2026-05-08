# VPS Migration Master Plan
## Docker + WebUI + Google Chat Integration

Created: 2025-05-08  
Goal: Complete Hermes migration to Hostinger VPS with modern containerized architecture

---

## Executive Summary

This plan covers the complete migration of Jimmy (Hermes Agent) from your local Proxmox lab to your Hostinger VPS, with three major additions:

1. **Containerized Hermes** — Docker-based deployment for portability
2. **Hermes WebUI** — Web interface served via your existing Traefik setup
3. **Google Chat Integration** — Two-way chat via Google Chat bot

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        HOSTINGER VPS                            │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   Traefik    │◄───│   WebUI      │◄───│   Hermes     │      │
│  │   (reverse   │    │   Container  │    │   Container  │      │
│  │    proxy)    │    │   Port 8080  │    │   Port 3000  │      │
│  └──────┬───────┘    └──────────────┘    └──────────────┘      │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐                                              │
│  │   hermes.    │  ← subdomain you set                        │
│  │   yourdomain │     (HTTPS via Traefik)                      │
│  │   .com       │                                              │
│  └──────────────┘                                              │
│                                                                 │
│  ┌──────────────┐    ┌──────────────────────────────┐          │
│  │  Google Chat │◄───│  Google Cloud Webhook        │          │
│  │   Space      │    │  Endpoint → Hermes           │          │
│  └──────────────┘    └──────────────────────────────┘          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: VPS Preparation (Prerequisites)

### 1.1 Verify Current VPS State

SSH to your Hostinger VPS and confirm:

```bash
# Check Traefik is running
docker ps | grep traefik

# Check you have Docker Compose
docker compose version

# Verify subdomain DNS points to VPS
dig hermes.yourdomain.com +short
# Should return your VPS IP
```

### 1.2 Create Hermes Directory Structure

```bash
# SSH to VPS
ssh root@your-vps-ip

# Create project directory
mkdir -p /opt/hermes/{data,config,certs}
cd /opt/hermes

# This will hold:
# /opt/hermes/data/     → Persistent Hermes data (memory, sessions, skills)
# /opt/hermes/config/   → config.yaml, .env
# /opt/hermes/certs/    → SSL certs (if needed outside Traefik)
# /opt/hermes/docker-compose.yml
```

---

## Phase 2: Hermes Docker Setup

### 2.1 Create Dockerfile for Hermes

```bash
cd /opt/hermes
micro Dockerfile
```

**Dockerfile content:**

```dockerfile
FROM ubuntu:24.04

# Prevent interactive prompts during build
ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies
RUN apt-get update && apt-get install -y \
    curl \
    git \
    python3 \
    python3-pip \
    python3-venv \
    openssh-client \
    ca-certificates \
    nano \
    micro \
    && rm -rf /var/lib/apt/lists/*

# Install Hermes
RUN curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# Add Hermes to PATH
ENV PATH="/root/.local/bin:${PATH}"

# Create Hermes home directory
RUN mkdir -p /root/.hermes

# Set working directory
WORKDIR /root

# Expose gateway port (optional, for API access)
EXPOSE 3000

# Default command
CMD ["hermes", "gateway", "run"]
```

### 2.2 Create Docker Compose Configuration

```bash
cd /opt/hermes
micro docker-compose.yml
```

**docker-compose.yml content:**

```yaml
version: "3.8"

services:
  # Hermes Core Agent
  hermes:
    build: .
    container_name: hermes-agent
    restart: unless-stopped
    volumes:
      # Persist Hermes data
      - ./data/config:/root/.hermes
      - ./data/memory:/root/.hermes/memory
      - ./data/sessions:/root/.hermes/sessions
      - ./data/skills:/root/.hermes/skills
      - ./data/logs:/root/.hermes/logs
      # Mount host Docker socket (for terminal tool)
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - HERMES_HOME=/root/.hermes
      - TERM=xterm-256color
    networks:
      - hermes-network
    # Gateway runs inside container
    ports:
      - "127.0.0.1:3000:3000"  # Local only, Traefik will proxy
    healthcheck:
      test: ["CMD", "hermes", "doctor"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Hermes WebUI
  webui:
    build:
      context: https://github.com/nesquena/hermes-webui.git#main
      dockerfile: Dockerfile
    container_name: hermes-webui
    restart: unless-stopped
    environment:
      - HERMES_WEBUI_HOST=0.0.0.0
      - HERMES_WEBUI_PORT=8080
      - HERMES_WORKSPACE=/workspace
      # Point WebUI to Hermes API (internal Docker network)
      - HERMES_API_URL=http://hermes:3000
    volumes:
      - ./data/workspace:/workspace
    networks:
      - hermes-network
    depends_on:
      - hermes
    labels:
      # Traefik labels for routing
      - "traefik.enable=true"
      - "traefik.http.routers.hermes-webui.rule=Host(`hermes.yourdomain.com`)"
      - "traefik.http.routers.hermes-webui.entrypoints=websecure"
      - "traefik.http.routers.hermes-webui.tls.certresolver=letsencrypt"
      - "traefik.http.services.hermes-webui.loadbalancer.server.port=8080"
      # Optional: Basic auth middleware
      # - "traefik.http.routers.hermes-webui.middlewares=hermes-auth"
      # - "traefik.http.middlewares.hermes-auth.basicauth.users=admin:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"

networks:
  hermes-network:
    driver: bridge
```

**Note:** Replace `hermes.yourdomain.com` with your actual subdomain.

---

## Phase 3: Transfer Existing Data

### 3.1 Transfer Config from Local Machine

From your local Proxmox lab:

```bash
# Transfer config files
scp ~/.hermes/config.yaml root@your-vps:/opt/hermes/data/config/
scp ~/.hermes/.env root@your-vps:/opt/hermes/data/config/
scp ~/.hermes/auth.json root@your-vps:/opt/hermes/data/config/ 2>/dev/null || echo "No auth.json"

# Transfer memory (what Jimmy knows about you)
scp -r ~/.hermes/memory root@your-vps:/opt/hermes/data/

# Transfer sessions (conversation history)
scp -r ~/.hermes/sessions root@your-vps:/opt/hermes/data/

# Transfer skills
scp -r ~/.hermes/skills root@your-vps:/opt/hermes/data/
```

### 3.2 Verify Data on VPS

```bash
ssh root@your-vps
ls -la /opt/hermes/data/
# Should show: config, memory, sessions, skills, workspace
```

---

## Phase 4: Build and Deploy

### 4.1 Build Containers

```bash
cd /opt/hermes
docker compose build
```

### 4.2 Start Services

```bash
docker compose up -d

# Check logs
docker compose logs -f

# Verify containers are running
docker ps
```

### 4.3 Test Hermes Inside Container

```bash
# Enter the Hermes container
docker exec -it hermes-agent bash

# Test Hermes CLI
hermes doctor

# Start interactive session
hermes

# Ask: "What do you know about me?"
# Should remember Jesus, timezone, etc.
```

### 4.4 Verify WebUI via Traefik

```bash
# Check Traefik registered the route
curl -H "Host: hermes.yourdomain.com" http://localhost:8080/api/http/routers

# Test HTTPS access (from your local machine)
curl https://hermes.yourdomain.com
# Should return the WebUI
```

---

## Phase 5: Google Chat Integration

### 5.1 Create Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create new project: "Hermes Chat Bot" or similar
3. Enable the **Google Chat API**:
   - APIs & Services → Library
   - Search "Google Chat API" → Enable

### 5.2 Configure OAuth Consent Screen

1. APIs & Services → OAuth consent screen
2. Select **External** (for personal use) or **Internal** (if you have Google Workspace)
3. Fill in required fields:
   - App name: "Jimmy"
   - User support email: your email
   - Developer contact: your email
4. Scopes: Add `chat.bot` scope
5. Save and continue

### 5.3 Create Chat App/Bot

1. Go to [Google Chat API Configuration](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat)
2. Under **Configuration**:
   - App name: "Jimmy"
   - Avatar URL: (optional, can use any image URL)
   - Description: "AI assistant for Jesus"
3. **Connection settings**:
   - Select **HTTP endpoint**
   - URL: `https://hermes.yourdomain.com/webhooks/google-chat`
     (we'll create this endpoint in Hermes)
4. **Visibility**:
   - Add your Google account email under "Specific people and groups"
5. Save

### 5.4 Create Service Account (For API Calls)

1. IAM & Admin → Service Accounts
2. Create Service Account:
   - Name: "hermes-chat-sa"
   - Role: **Chat Bot Viewer** (or custom)
3. Create key → JSON → Download
4. Save this file as `/opt/hermes/data/google-chat-credentials.json`

### 5.5 Create Webhook Handler in Hermes

We need to add a webhook handler that receives Google Chat events. Options:

**Option A: Use Hermes Built-in Webhook System**

Hermes has a webhook subscription system we can use:

```bash
# SSH to VPS and enter container
docker exec -it hermes-agent bash

# Create webhook subscription
hermes webhook subscribe google-chat

# This creates endpoint at /webhooks/google-chat
```

**Option B: Custom FastAPI Endpoint (Recommended)**

Create a separate small service that translates Google Chat → Hermes API:

```bash
cd /opt/hermes
micro google-chat-bridge.py
```

**Sample bridge code:**

```python
#!/usr/bin/env python3
"""
Google Chat → Hermes Bridge
Receives Google Chat events, forwards to Hermes, returns responses
"""

import os
import json
import requests
from flask import Flask, request, jsonify
from google.oauth2 import service_account
from googleapiclient.discovery import build

app = Flask(__name__)

# Config
HERMES_API_URL = "http://hermes:3000/api/chat"  # Internal Docker network
GOOGLE_CHAT_CREDENTIALS = "/app/credentials.json"

@app.route('/webhooks/google-chat', methods=['POST'])
def handle_google_chat():
    """Receive messages from Google Chat"""
    event = request.json
    
    # Extract message info
    message_text = event.get('message', {}).get('text', '')
    space_name = event.get('space', {}).get('name')
    user_name = event.get('user', {}).get('displayName', 'User')
    
    # Skip bot's own messages
    if event.get('message', {}).get('sender', {}).get('type') == 'BOT':
        return jsonify({})
    
    # Forward to Hermes
    try:
        response = requests.post(HERMES_API_URL, json={
            'message': message_text,
            'user': user_name,
            'platform': 'google-chat',
            'space': space_name
        }, timeout=60)
        
        hermes_reply = response.json().get('response', 'Sorry, I had trouble processing that.')
        
        # Return in Google Chat format
        return jsonify({'text': hermes_reply})
        
    except Exception as e:
        print(f"Error: {e}")
        return jsonify({'text': 'Sorry, something went wrong.'})

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'ok'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Add to docker-compose.yml:

```yaml
  google-chat-bridge:
    image: python:3.11-slim
    container_name: google-chat-bridge
    restart: unless-stopped
    working_dir: /app
    volumes:
      - ./google-chat-bridge.py:/app/app.py
      - ./data/google-chat-credentials.json:/app/credentials.json:ro
    command: >
      bash -c "pip install flask requests google-auth google-api-python-client && python app.py"
    networks:
      - hermes-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.google-chat.rule=Host(`hermes.yourdomain.com`) && PathPrefix(`/webhooks/google-chat`)"
      - "traefik.http.routers.google-chat.entrypoints=websecure"
      - "traefik.http.routers.google-chat.tls.certresolver=letsencrypt"
      - "traefik.http.services.google-chat.loadbalancer.server.port=5000"
```

---

## Phase 6: Create Google Chat Space

### 6.1 Create Space in Google Chat

1. Open [Google Chat](https://chat.google.com) (web or app)
2. Click **+** → **Create Space**
3. Name: "Jimmy" or "AI Assistant"
4. Add the bot:
   - Click the space name → **Apps & integrations**
   - Find your "Jimmy" app
   - Add to space

### 6.2 Test Two-Way Communication

1. Send a message in the space
2. Should see response from Jimmy
3. Try: "What do you know about me?"
4. Jimmy should remember everything from memory

---

## Phase 7: Final Configuration

### 7.1 Update Hermes Config for New Environment

```bash
# SSH to VPS
docker exec -it hermes-agent bash

# Edit config
hermes config edit
```

**Key settings to update:**

```yaml
# Set gateway to listen on all interfaces (for Docker)
gateway:
  host: 0.0.0.0
  port: 3000

# Enable webhook server
webhooks:
  enabled: true
  host: 0.0.0.0
  port: 3001  # or same as gateway
```

### 7.2 Set Up Health Checks & Monitoring

```bash
# Add to crontab on host (not container)
crontab -e

# Add:
*/5 * * * * /opt/hermes/health-check.sh >> /var/log/hermes-health.log 2>&1
```

**health-check.sh:**

```bash
#!/bin/bash
if ! docker ps | grep -q hermes-agent; then
    echo "$(date): Hermes down, restarting..."
    cd /opt/hermes && docker compose restart
    echo "$(date): Restarted" | mail -s "Hermes Restarted" your-email@example.com
fi
```

### 7.3 Set Up Automated Backups

```bash
# Backup script
micro /opt/hermes/backup.sh
```

```bash
#!/bin/bash
BACKUP_DIR="/backups/hermes"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p "$BACKUP_DIR"

# Backup data
tar czf "$BACKUP_DIR/hermes_backup_$DATE.tar.gz" /opt/hermes/data/

# Keep only last 7 backups
ls -t "$BACKUP_DIR"/*.tar.gz | tail -n +8 | xargs -r rm

echo "Backup completed: $BACKUP_DIR/hermes_backup_$DATE.tar.gz"
```

```bash
# Daily backup at 2 AM
crontab -e
0 2 * * * /opt/hermes/backup.sh
```

---

## Phase 8: Testing & Validation

### 8.1 Test Matrix

| Component | Test | Expected Result |
|-----------|------|-----------------|
| Docker | `docker ps` | hermes-agent, webui, bridge running |
| WebUI | Visit https://hermes.yourdomain.com | Web interface loads |
| Traefik | `curl https://hermes.yourdomain.com/api/health` | 200 OK |
| Hermes CLI | `docker exec hermes-agent hermes doctor` | All checks pass |
| Memory | Ask "What's my name?" | Responds "Jesus" |
| Google Chat | Message in space | Receives reply |
| WebUI Chat | Send message in browser | Streaming response |

### 8.2 Common Issues & Fixes

**Issue: WebUI shows "Cannot connect to Hermes"**

- Check HERMES_API_URL in docker-compose.yml
- Verify containers on same network: `docker network inspect hermes-network`

**Issue: Google Chat not receiving responses**

- Check webhook URL in Google Cloud Console matches Traefik route
- Verify SSL certificate is valid (Google requires HTTPS)
- Check bridge logs: `docker logs google-chat-bridge`

**Issue: Traefik not routing**

- Check Traefik has access to Docker socket
- Verify labels are correct in docker-compose.yml
- Check Traefik dashboard: `https://traefik.yourdomain.com`

---

## Phase 9: Post-Migration Tasks

### 9.1 Update Telegram Gateway (Optional)

If keeping Telegram as backup:

```bash
docker exec -it hermes-agent bash
hermes gateway setup telegram
# Update webhook to point to VPS instead of local
```

### 9.2 Document New Endpoints

Update your internal docs with:
- WebUI URL: `https://hermes.yourdomain.com`
- API endpoint: `https://hermes.yourdomain.com/api`
- Google Chat space: (link to space)

### 9.3 Clean Up Local Proxmox

After 1 week of stable operation:

```bash
# On local Proxmox (backup first!)
hermes gateway stop  # Stop local gateway
# Archive local data if needed
tar czf ~/hermes-local-archive.tar.gz ~/.hermes/
```

---

## Appendix A: Full docker-compose.yml Reference

```yaml
version: "3.8"

services:
  hermes:
    build: .
    container_name: hermes-agent
    restart: unless-stopped
    volumes:
      - ./data/config:/root/.hermes
      - ./data/memory:/root/.hermes/memory
      - ./data/sessions:/root/.hermes/sessions
      - ./data/skills:/root/.hermes/skills
      - ./data/logs:/root/.hermes/logs
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - HERMES_HOME=/root/.hermes
      - TERM=xterm-256color
    networks:
      - hermes-network
    ports:
      - "127.0.0.1:3000:3000"

  webui:
    build:
      context: https://github.com/nesquena/hermes-webui.git#main
    container_name: hermes-webui
    restart: unless-stopped
    environment:
      - HERMES_WEBUI_HOST=0.0.0.0
      - HERMES_WEBUI_PORT=8080
      - HERMES_WORKSPACE=/workspace
      - HERMES_API_URL=http://hermes:3000
    volumes:
      - ./data/workspace:/workspace
    networks:
      - hermes-network
    depends_on:
      - hermes
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.hermes-webui.rule=Host(`hermes.yourdomain.com`)"
      - "traefik.http.routers.hermes-webui.entrypoints=websecure"
      - "traefik.http.routers.hermes-webui.tls.certresolver=letsencrypt"
      - "traefik.http.services.hermes-webui.loadbalancer.server.port=8080"

  google-chat-bridge:
    image: python:3.11-slim
    container_name: google-chat-bridge
    restart: unless-stopped
    working_dir: /app
    volumes:
      - ./google-chat-bridge.py:/app/app.py
      - ./data/google-chat-credentials.json:/app/credentials.json:ro
    command: >
      bash -c "pip install flask requests google-auth google-api-python-client && python app.py"
    networks:
      - hermes-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.google-chat.rule=Host(`hermes.yourdomain.com`) && PathPrefix(`/webhooks/google-chat`)"
      - "traefik.http.routers.google-chat.entrypoints=websecure"
      - "traefik.http.routers.google-chat.tls.certresolver=letsencrypt"
      - "traefik.http.services.google-chat.loadbalancer.server.port=5000"

networks:
  hermes-network:
    driver: bridge
```

---

## Appendix B: Related Documents

- [[Transfer Hermes to New Machine via SSH]] — Original transfer guide
- [[Hermes WebUI + Tailscale Setup]] — Previous WebUI exploration
- [[Hermes WebUI TTS Speaker Icon Fix]] — Known issue fix

---

## Timeline Estimate

| Phase | Time |
|-------|------|
| Phase 1: VPS Prep | 30 min |
| Phase 2: Docker Setup | 1 hour |
| Phase 3: Data Transfer | 30 min |
| Phase 4: Build & Deploy | 1 hour |
| Phase 5: Google Chat Setup | 1.5 hours |
| Phase 6-7: Testing & Config | 1 hour |
| **Total** | **~6 hours** |

---

*Plan created: 2025-05-08*  
*Status: Ready for implementation*
