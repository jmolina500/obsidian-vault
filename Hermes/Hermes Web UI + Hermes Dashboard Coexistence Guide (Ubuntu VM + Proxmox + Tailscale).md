# Hermes Web UI + Hermes Dashboard Coexistence Guide (Ubuntu VM + Proxmox + Tailscale)

## Objective

Run both Hermes interfaces simultaneously:

* **Hermes Web UI** (browser access over Tailscale MagicDNS)
* **Hermes Dashboard** (browser access over SSH tunnel)

This setup avoids exposing the dashboard insecurely while keeping both interfaces available at the same time.

---

# Architecture

```text
Hermes Web UI
Windows Browser
   ↓
http://jimmy.tailfbd95.ts.net
   ↓
Tailscale MagicDNS
   ↓
Tailscale Serve
   ↓
127.0.0.1:8787
   ↓
Hermes Web UI


Hermes Dashboard
Windows Browser
   ↓
http://localhost:9119
   ↓
SSH Tunnel
   ↓
100.125.61.17
   ↓
127.0.0.1:9119
   ↓
Hermes Dashboard
```

---

# Environment Assumptions

Server:

* Ubuntu VM in Proxmox
* Root access
* Hermes Agent installed
* Hermes Web UI installed
* Tailscale installed and authenticated

Windows Client:

* Tailscale installed
* MagicDNS enabled
* SSH client available (Windows PowerShell)

Server details used in this guide:

```text
Tailscale hostname: jimmy.tailfbd95.ts.net
Tailscale IP: 100.125.61.17
Hermes Web UI port: 8787
Hermes Dashboard port: 9119
```

---

# Part 1 — Hermes Web UI (Persistent)

## Verify Hermes Web UI service

On Ubuntu:

```bash
systemctl status hermes-webui
```

Expected:

```text
active (running)
```

---

## Verify Hermes Web UI locally

```bash
ss -tulnp | grep 8787
curl http://127.0.0.1:8787
```

Expected:

* port listening
* HTML response

---

## Verify Tailscale Serve

```bash
tailscale serve status
```

Expected:

```text
http://jimmy.tailfbd95.ts.net
|-- / proxy http://127.0.0.1:8787
```

---

## Browser access

Open:

```text
http://jimmy.tailfbd95.ts.net
```

This is your persistent Hermes Web UI.

---

# Part 2 — Hermes Dashboard (Local Secure Access)

## Why dashboard is handled differently

Hermes Dashboard intentionally blocks direct remote binding:

```text
Refusing to bind to 100.125.61.17
Refusing to bind to jimmy.tailfbd95.ts.net
```

Reason:
the dashboard exposes:

* API keys
* configuration
* agent controls

and requires stronger protection.

Because of that, the safest method is SSH tunneling.

---

# Start Hermes Dashboard

On Ubuntu VM:

Activate Hermes environment:

```bash
source /root/.hermes/hermes-agent/venv/bin/activate
```

Start dashboard:

```bash
nohup hermes dashboard --host 127.0.0.1 --port 9119 --no-open > /root/.hermes/dashboard.log 2>&1 &
```

---

## Verify dashboard

```bash
ss -tulnp | grep 9119
curl http://127.0.0.1:9119
```

Expected:

```text
127.0.0.1:9119 LISTEN
```

and HTML containing:

```html
<title>Hermes Agent - Dashboard</title>
```

---

# Create SSH Tunnel from Windows

Open Windows PowerShell.

Run:

```powershell
ssh -L 9119:127.0.0.1:9119 root@100.125.61.17
```

Enter root password.

Keep this PowerShell window open.

---

# Browser Access for Dashboard

Open:

```text
http://localhost:9119
```

This securely forwards your local browser to the dashboard.

---

# Final Concurrent Access

With both running:

Hermes Web UI:

```text
http://jimmy.tailfbd95.ts.net
```

Hermes Dashboard:

```text
http://localhost:9119
```

Both interfaces can remain open simultaneously.

---

# Restart Commands

## Restart Hermes Web UI

Ubuntu:

```bash
systemctl restart hermes-webui
```

---

## Restart Dashboard

Ubuntu:

```bash
pkill -f "hermes dashboard"
source /root/.hermes/hermes-agent/venv/bin/activate
nohup hermes dashboard --host 127.0.0.1 --port 9119 --no-open > /root/.hermes/dashboard.log 2>&1 &
```

---

## Recreate SSH Tunnel

Windows:

```powershell
ssh -L 9119:127.0.0.1:9119 root@100.125.61.17
```

---

# Troubleshooting

## Web UI not loading

Ubuntu:

```bash
systemctl status hermes-webui
tailscale serve status
ss -tulnp | grep 8787
curl http://127.0.0.1:8787
```

---

## Dashboard not loading

Ubuntu:

```bash
ss -tulnp | grep 9119
curl http://127.0.0.1:9119
cat /root/.hermes/dashboard.log
```

---

## SSH tunnel failed

Windows:

Check connectivity:

```powershell
tailscale status
ping 100.125.61.17
```

Retry:

```powershell
ssh -L 9119:127.0.0.1:9119 root@100.125.61.17
```

---

## MagicDNS failed

Windows:

Verify:

```powershell
tailscale debug prefs
```

Expected:

```json
"CorpDNS": true
```

If false:

```powershell
tailscale set --accept-dns=true
```

---

# Usage Guidance

## Hermes Web UI

Best for:

* conversational agent interaction
* quick prompts
* mobile access
* lightweight workflows

---

## Hermes Dashboard

Best for:

* monitoring
* sessions
* logs
* model configuration
* cron tasks
* agent administration
* troubleshooting

---

# Security Notes

Web UI:

* safely exposed through private Tailscale network

Dashboard:

* intentionally kept local-only
* exposed only through encrypted SSH tunnel
* avoids unsafe public or tailnet-wide exposure

---

# Recommended Daily Workflow

1. Open browser:

```text
http://jimmy.tailfbd95.ts.net
```

2. Open PowerShell:

```powershell
ssh -L 9119:127.0.0.1:9119 root@100.125.61.17
```

3. Open dashboard:

```text
http://localhost:9119
```

---

# Final Result

You now have:

* persistent Hermes Web UI
* secure Hermes Dashboard access
* simultaneous usage
* no insecure dashboard exposure
* clean separation of operator UI vs admin dashboard
