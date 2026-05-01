# Hermes Web UI + Tailscale Setup (Ubuntu VM on Proxmox)

## Goal

* Run Hermes Web UI persistently
* Access it securely via Tailscale
* No manual restarts
* Survives reboot

---

# 1. Prerequisites

* Ubuntu Server 22.04 or newer
* Root access (this guide assumes you are root)
* Internet access

---

# 2. Update System

```bash
apt update && apt upgrade -y
```

---

# 3. Install Dependencies

```bash
apt install -y git python3 python3-pip curl
```

Optional:

```bash
apt install -y python3-venv
```

---

# 4. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Start and authenticate:

```bash
tailscale up
```

Follow the login URL shown in the terminal.

---

# 5. Verify Tailscale

```bash
tailscale status
```

---

# 6. Install Hermes Web UI

```bash
cd ~
git clone https://github.com/nesquena/hermes-webui.git
cd hermes-webui
```

---

# 7. Test Hermes (Initial Run)

Run this inside the Hermes directory:

```bash
cd ~/hermes-webui
python3 bootstrap.py
```

Expected output:

```text
[bootstrap] Web UI is ready: http://localhost:8787
```

Test locally:

```bash
curl http://127.0.0.1:8787
```

You should see HTML output.

Stop the process:

```text
CTRL + C
```

---

# 8. Create systemd Service (Persistent Setup)

Create the service file:

```bash
micro /etc/systemd/system/hermes-webui.service
```

Paste:

```ini
[Unit]
Description=Hermes Web UI
After=network.target

[Service]
Type=simple
WorkingDirectory=/root/hermes-webui
ExecStart=/usr/bin/python3 /root/hermes-webui/bootstrap.py
Restart=always
RestartSec=5

RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

---

# 9. Enable and Start Service

```bash
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable hermes-webui
systemctl start hermes-webui
```

---

# 10. Verify Service

```bash
systemctl status hermes-webui
```

Expected:

```text
Active: active (exited)
```

This is normal behavior.

---

# 11. Confirm Hermes is Running

Check the port:

```bash
ss -tulnp | grep 8787
```

Expected:

```text
127.0.0.1:8787 LISTEN
```

Test again:

```bash
curl http://127.0.0.1:8787
```

---

# Important Behavior

Hermes works as follows:

* `bootstrap.py` starts the web server
* The script exits after launching it
* systemd shows "active (exited)"
* The actual web server continues running

This is expected and correct.

---

# 12. Configure Tailscale Serve

Reset any previous configuration:

```bash
tailscale serve --https=443 off
tailscale serve --http=80 off
```

If you see an error like "handler does not exist", it can be ignored.

Start the service in the background:

```bash
tailscale serve --bg --http=80 8787
```

---

# 13. Verify Tailscale Serve

```bash
tailscale serve status
```

Expected:

```text
http://<your-node>.tailnet.ts.net
|-- proxy http://127.0.0.1:8787
```

---

# 14. Access the Web UI

Use HTTP:

```text
http://<your-node>.tailnet.ts.net
```

Example:

```text
http://jimmy.tailfbd95.ts.net
```

Do not use HTTPS in this setup.

---

# 15. Persistence After Reboot

After reboot:

* Tailscale starts automatically
* Hermes starts via systemd
* Tailscale Serve remains active
* Web UI is accessible without manual steps

---

# 16. Troubleshooting

Check if Hermes is running:

```bash
ss -tulnp | grep 8787
curl http://127.0.0.1:8787
```

Restart Hermes:

```bash
systemctl restart hermes-webui
```

View logs:

```bash
journalctl -u hermes-webui -f
```

Check Tailscale:

```bash
tailscale serve status
tailscale status
```

---

# 17. Common Issues

Web UI not loading:

* Hermes may not be running. Restart the service.

SSL errors:

* Use HTTP instead of HTTPS.

"handler does not exist":

* This can be ignored.

Browser issues:

* Try incognito mode or clear cache.

---

# 18. Architecture Overview

```text
Browser
   ↓
http://tailnet URL
   ↓
Tailscale Serve
   ↓
127.0.0.1:8787
   ↓
Hermes Web UI
```

---

# Final Result

* Persistent service
* Remote access via Tailscale
* No manual startup required
* Clean and stable setup

---

# Optional Improvements

* Run Hermes under a non-root user
* Add reverse proxy (Caddy or Nginx)
* Serve multiple applications under one domain
* Add authentication layer

---
