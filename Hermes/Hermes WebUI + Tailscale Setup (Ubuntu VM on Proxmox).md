# Hermes Web UI + Tailscale + MagicDNS Setup Guide (Ubuntu VM on Proxmox)

## Objective

This guide walks through installing Hermes Web UI on an Ubuntu Server VM running in Proxmox and exposing it securely through Tailscale using MagicDNS.

Final result:

* Hermes Web UI runs automatically after reboot
* Accessible securely via browser using:

  ```text
  http://jimmy.tailfbd95.ts.net
  ```
* No manual startup required
* Windows client properly resolves MagicDNS

---

# Architecture Overview

```text
Windows Browser
   ↓
MagicDNS (Tailscale DNS)
   ↓
http://jimmy.tailfbd95.ts.net
   ↓
Tailscale Serve
   ↓
127.0.0.1:8787
   ↓
Hermes Web UI
```

---

# Prerequisites

Server:

* Ubuntu Server 24.04+ VM in Proxmox
* Root access
* Internet connectivity

Client:

* Windows machine with Tailscale installed
* Logged into same Tailscale account

---

# Step 1 — Update Ubuntu Server

Run as root:

```bash
apt update && apt upgrade -y
```

---

# Step 2 — Install Dependencies

Run as root:

```bash
apt install -y git python3 python3-pip curl micro
```

Optional:

```bash
apt install -y python3-venv dnsutils
```

---

# Step 3 — Install Tailscale on Ubuntu Server

Run:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Bring Tailscale online:

```bash
tailscale up
```

Follow the authentication URL shown in terminal.

Verify:

```bash
tailscale status
```

Expected:

```text
100.x.x.x   jimmy   jesus.molina@...
```

---

# Step 4 — Install Hermes Web UI

Change to root home:

```bash
cd /root
```

Clone repository:

```bash
git clone https://github.com/nesquena/hermes-webui.git
```

Enter directory:

```bash
cd /root/hermes-webui
```

---

# Step 5 — Initial Hermes Test

Run:

```bash
python3 bootstrap.py
```

Expected:

```text
Hermes Web UI listening on http://127.0.0.1:8787
```

Test locally:

```bash
curl http://127.0.0.1:8787
```

You should see HTML output.

Stop test:

```text
CTRL + C
```

---

# Important Behavior

Hermes bootstrap behaves differently than normal services.

`bootstrap.py` starts the web server, then exits.

This means:

```bash
systemctl status hermes-webui
```

may show:

```text
active (exited)
```

That can be normal depending on bootstrap behavior.

In your final setup, Hermes runs via:

```text
/root/.hermes/hermes-agent/venv/bin/python /root/hermes-webui/server.py
```

---

# Step 6 — Create systemd Service

Create file:

```bash
micro /etc/systemd/system/hermes-webui.service
```

Paste:

```ini
[Unit]
Description=Hermes Web UI
After=network.target tailscaled.service
Requires=tailscaled.service

[Service]
Type=simple
WorkingDirectory=/root/hermes-webui
ExecStart=/usr/bin/python3 /root/hermes-webui/bootstrap.py
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

Save and exit.

---

# Step 7 — Enable Hermes Service

Run:

```bash
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable hermes-webui
systemctl start hermes-webui
```

Verify:

```bash
systemctl status hermes-webui --no-pager
```

Expected:

```text
Active: active (running)
```

and:

```text
Hermes Web UI listening on http://127.0.0.1:8787
```

---

# Step 8 — Verify Hermes Locally

Check listening port:

```bash
ss -tulnp | grep 8787
```

Expected:

```text
127.0.0.1:8787 LISTEN
```

Test:

```bash
curl http://127.0.0.1:8787
```

Expected:
HTML output.

---

# Step 9 — Configure Tailscale Serve

Clear any previous config:

```bash
tailscale serve --https=443 off
tailscale serve --http=80 off
```

If you see:

```text
handler does not exist
```

it can be ignored.

Configure background proxy:

```bash
tailscale serve --bg --http=80 8787
```

Verify:

```bash
tailscale serve status
```

Expected:

```text
http://jimmy.tailfbd95.ts.net
|-- / proxy http://127.0.0.1:8787
```

---

# Step 10 — Ensure Tailscale Starts Automatically

Run:

```bash
systemctl enable tailscaled
```

Verify:

```bash
systemctl status tailscaled --no-pager
```

Expected:

```text
Active: active (running)
```

---

# Step 11 — Windows Client MagicDNS Configuration

This step is critical.

Without this, browser access will fail with DNS resolution errors.

Open Windows PowerShell as your normal user.

Run:

```powershell
tailscale set --accept-dns=true
```

Verify:

```powershell
tailscale debug prefs
```

Expected:

```json
"CorpDNS": true
```

This means Tailscale DNS integration is active.

---

# Step 12 — Verify Windows DNS

Run:

```powershell
Get-DnsClientServerAddress
```

Expected to see Tailscale DNS:

```text
Tailscale
100.100.100.100
fd7a:115c:a1e0::53
```

Test DNS:

```powershell
nslookup jimmy.tailfbd95.ts.net
```

Expected:

```text
Address: 100.125.61.17
```

---

# Step 13 — Browser Access

Open:

```text
http://jimmy.tailfbd95.ts.net
```

The Hermes Web UI should load.

---

# Troubleshooting

## Hermes service failed

Check:

```bash
systemctl status hermes-webui --no-pager
journalctl -u hermes-webui -f
```

Restart:

```bash
systemctl restart hermes-webui
```

---

## Hermes not listening

Check:

```bash
ss -tulnp | grep 8787
```

If empty:

```bash
systemctl restart hermes-webui
```

---

## Tailscale proxy missing

Check:

```bash
tailscale serve status
```

If empty:

```bash
tailscale serve --bg --http=80 8787
```

---

## MagicDNS not resolving

Check Windows:

```powershell
tailscale debug prefs
```

If:

```json
"CorpDNS": false
```

Fix:

```powershell
tailscale set --accept-dns=true
```

---

## Direct emergency access

If MagicDNS fails temporarily:

```powershell
ssh -L 8787:127.0.0.1:8787 root@100.125.61.17
```

Then browser:

```text
http://localhost:8787
```

---

# Permanent Verification Checklist

Server:

```bash
systemctl status hermes-webui
systemctl status tailscaled
tailscale serve status
ss -tulnp | grep 8787
curl http://127.0.0.1:8787
```

Windows:

```powershell
tailscale status
tailscale debug prefs
nslookup jimmy.tailfbd95.ts.net
```

Browser:

```text
http://jimmy.tailfbd95.ts.net
```

---

# Final Result

Persistent Hermes Web UI deployment with:

* automatic startup
* secure Tailscale access
* MagicDNS browser access
* fallback SSH tunnel access
* stable production-style setup
