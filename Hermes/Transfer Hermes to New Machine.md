# Transfer Hermes to New Machine via SSH

Created: 2025-04-20

Complete guide to transfer Jimmy (Hermes) from your local environment to a remote VPS/server.

---

## Overview

**Source:** Local Proxmox lab environment (where Jimmy currently runs)  
**Destination:** Remote VPS/server (where you want Jimmy to run)

This guide covers transferring all configuration, memory, and history to the new machine.

---

## Prerequisites (Do These First)

### Step 1: Get Your VPS IP Address

On your **destination machine (VPS)**, run:

```bash
# Method 1: Check network interface
ip addr show
# Look for "inet" followed by a public IP (not 127.0.0.1 or 192.168.x.x)
# Example: inet 203.0.113.45/24

# Method 2: External check
curl ifconfig.me
# Returns your public IP: 203.0.113.45
```

**Write down this IP address.** You'll need it for the transfer.

---

### Step 2: Ensure SSH is Enabled on VPS

On your **destination machine (VPS)**:

```bash
# Check if SSH is running
sudo systemctl status sshd

# If not running, install and start:
# For Debian/Ubuntu:
sudo apt update
sudo apt install openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd

# For CentOS/RHEL/Rocky:
sudo dnf install openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
```

---

### Step 3: Open Firewall Port 22

On your **destination machine (VPS)**:

```bash
# Check if port 22 is open
sudo ss -tlnp | grep :22

# If using UFW (Ubuntu/Debian):
sudo ufw allow 22/tcp
sudo ufw enable

# If using firewalld (CentOS/RHEL):
sudo firewall-cmd --permanent --add-port=22/tcp
sudo firewall-cmd --reload

# If using iptables:
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables-save
```

**Note:** Some VPS providers have a web-based firewall (security groups). You may also need to allow port 22 there through your provider's control panel.

---

### Step 4: Generate SSH Key (If You Don't Have One)

On your **source machine (local Proxmox lab)**:

```bash
# Check if you already have SSH keys
ls ~/.ssh/id_*.pub

# If no files found, generate a new key:
ssh-keygen -t ed25519 -C "your-email@example.com"

# When prompted:
# Enter file in which to save the key: (press Enter for default)
# Enter passphrase: (press Enter for no passphrase, or set one)
# Enter same passphrase again: (press Enter again)

# You should see: "Your identification has been saved"
```

---

### Step 5: Copy SSH Key to VPS

On your **source machine (local Proxmox lab)**:

```bash
# Copy your public key to the VPS (replace with your actual IP)
ssh-copy-id root@203.0.113.45
# Or if using a different user:
ssh-copy-id username@203.0.113.45

# You'll be asked for the VPS password once
# After this, you can SSH without password

# Test the connection:
ssh root@203.0.113.45
# Should log in without asking for password
exit
```

**Troubleshooting:** If `ssh-copy-id` doesn't work, manually copy the key:
```bash
# On source machine:
cat ~/.ssh/id_ed25519.pub
# Copy the output (starts with ssh-ed25519)

# SSH to VPS and add it:
ssh root@203.0.113.45
echo "ssh-ed25519 AAAA... your-email@example.com" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
exit
```

---

## The Transfer Script

### Step 6: Create the Script File

On your **source machine (local Proxmox lab)**:

```bash
# Navigate to your vault folder
cd "/root/Documents/Obsidian Vault/Hermes/"

# Create the script file
micro transfer-hermes.sh
```

In the micro editor:
1. Paste the script content below
2. Press `Ctrl+S` to save
3. Press `Ctrl+Q` to quit

---

### The Script Content

Copy this entire block into `transfer-hermes.sh`:

```bash
#!/bin/bash

# Transfer Hermes configuration from local environment to remote VPS
# Usage: ./transfer-hermes.sh user@vps-ip-address

set -e

NEW_MACHINE="$1"

if [ -z "$NEW_MACHINE" ]; then
    echo ""
    echo "ERROR: No destination provided"
    echo ""
    echo "Usage: $0 user@vps-ip-address"
    echo ""
    echo "Examples:"
    echo "  $0 root@203.0.113.45"
    echo "  $0 ubuntu@198.51.100.20"
    echo ""
    exit 1
fi

echo ""
echo "=================================="
echo "Transferring Hermes to $NEW_MACHINE"
echo "=================================="
echo ""

# 1. Test SSH connection first
echo "[1/7] Testing SSH connection..."
if ! ssh -o ConnectTimeout=10 "$NEW_MACHINE" "echo 'SSH OK'" 2>/dev/null; then
    echo ""
    echo "ERROR: Cannot connect to $NEW_MACHINE"
    echo ""
    echo "Please check:"
    echo "  - Is the IP address correct?"
    echo "  - Is SSH running on the destination?"
    echo "  - Is port 22 open in the firewall?"
    echo "  - Have you added your SSH key? (run: ssh-copy-id $NEW_MACHINE)"
    echo ""
    exit 1
fi
echo "    SSH connection successful"

# 2. Check if Hermes is installed on destination
echo "[2/7] Checking Hermes installation..."
if ! ssh "$NEW_MACHINE" "command -v hermes &> /dev/null"; then
    echo "    Hermes not found. Installing..."
    echo "    This may take a few minutes..."
    ssh "$NEW_MACHINE" "curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash"
    echo "    Hermes installed successfully"
else
    echo "    Hermes already installed"
fi

# 3. Create .hermes directory
echo "[3/7] Creating Hermes directory..."
ssh "$NEW_MACHINE" "mkdir -p ~/.hermes"

# 4. Transfer core configuration files
echo "[4/7] Transferring configuration files..."
TRANSFERRED=0

if [ -f ~/.hermes/config.yaml ]; then
    scp ~/.hermes/config.yaml "$NEW_MACHINE:~/.hermes/"
    echo "    ✓ config.yaml"
    TRANSFERRED=$((TRANSFERRED + 1))
else
    echo "    ⚠ config.yaml not found (skipped)"
fi

if [ -f ~/.hermes/.env ]; then
    scp ~/.hermes/.env "$NEW_MACHINE:~/.hermes/"
    echo "    ✓ .env (API keys)"
    TRANSFERRED=$((TRANSFERRED + 1))
else
    echo "    ⚠ .env not found (skipped)"
fi

if [ -f ~/.hermes/auth.json ]; then
    scp ~/.hermes/auth.json "$NEW_MACHINE:~/.hermes/"
    echo "    ✓ auth.json"
    TRANSFERRED=$((TRANSFERRED + 1))
else
    echo "    ⚠ auth.json not found (skipped)"
fi

if [ $TRANSFERRED -eq 0 ]; then
    echo ""
    echo "WARNING: No configuration files found to transfer!"
    echo "Checked: ~/.hermes/config.yaml, ~/.hermes/.env, ~/.hermes/auth.json"
    echo ""
fi

# 5. Transfer memory (CRITICAL - what Jimmy knows about you)
echo "[5/7] Transferring memory..."
if [ -d ~/.hermes/memory ] && [ "$(ls -A ~/.hermes/memory 2>/dev/null)" ]; then
    scp -r ~/.hermes/memory "$NEW_MACHINE:~/.hermes/"
    echo "    ✓ Memory transferred ($(ls ~/.hermes/memory | wc -l) files)"
else
    echo "    ⚠ No memory directory found or empty (Jimmy will start fresh)"
fi

# 6. Transfer session history (optional)
echo "[6/7] Transferring session history..."
if [ -d ~/.hermes/sessions ] && [ "$(ls -A ~/.hermes/sessions 2>/dev/null)" ]; then
    scp -r ~/.hermes/sessions "$NEW_MACHINE:~/.hermes/"
    echo "    ✓ Sessions transferred ($(ls ~/.hermes/sessions | wc -l) files)"
else
    echo "    ℹ No session history found (optional)"
fi

# 7. Transfer custom skills (optional)
echo "[7/7] Transferring custom skills..."
if [ -d ~/.hermes/skills ] && [ "$(ls -A ~/.hermes/skills 2>/dev/null)" ]; then
    scp -r ~/.hermes/skills "$NEW_MACHINE:~/.hermes/"
    echo "    ✓ Skills transferred"
else
    echo "    ℹ No custom skills found (optional)"
fi

# Verify the transfer
echo ""
echo "=================================="
echo "Verifying transfer..."
echo "=================================="
echo ""

if ssh "$NEW_MACHINE" "command -v hermes &> /dev/null"; then
    echo "✓ Hermes is installed"
else
    echo "✗ Hermes installation check failed"
fi

if ssh "$NEW_MACHINE" "[ -f ~/.hermes/config.yaml ]"; then
    echo "✓ Configuration file present"
else
    echo "✗ Configuration file missing"
fi

if ssh "$NEW_MACHINE" "[ -f ~/.hermes/.env ]"; then
    echo "✓ Environment file (API keys) present"
else
    echo "✗ Environment file missing"
fi

if ssh "$NEW_MACHINE" "[ -d ~/.hermes/memory ] && [ \"\$(ls -A ~/.hermes/memory 2>/dev/null)\" ]"; then
    echo "✓ Memory transferred"
else
    echo "⚠ Memory not found (Jimmy will start fresh)"
fi

echo ""
echo "=================================="
echo "Transfer Complete!"
echo "=================================="
echo ""
echo "Next steps:"
echo ""
echo "1. Test Jimmy on the new machine:"
echo "   ssh $NEW_MACHINE"
echo "   hermes"
echo ""
echo "2. Ask: 'What do you know about me?'"
echo "   Jimmy should remember your name and preferences."
echo ""
echo "3. (Optional) Clone your Obsidian vault:"
echo "   ssh $NEW_MACHINE"
echo "   git clone https://github.com/jmolina500/obsidian-vault.git \"\$HOME/Documents/Obsidian Vault\""
echo ""
echo "4. (Optional) Set your timezone:"
echo "   ssh $NEW_MACHINE"
echo "   sudo timedatectl set-timezone America/Puerto_Rico"
echo ""
```

---

### Step 7: Make Script Executable

On your **source machine**:

```bash
cd "/root/Documents/Obsidian Vault/Hermes/"
chmod +x transfer-hermes.sh
```

**What this does:** Makes the script runnable. Without this step, you'd get "Permission denied" when trying to run it.

---

### Step 8: Run the Transfer

On your **source machine**:

```bash
./transfer-hermes.sh root@203.0.113.45
# Replace with your actual VPS IP

# Or if your VPS uses a different username:
./transfer-hermes.sh ubuntu@203.0.113.45
```

**What you'll see:**
```
==================================
Transferring Hermes to root@203.0.113.45
==================================

[1/7] Testing SSH connection...
    SSH connection successful
[2/7] Checking Hermes installation...
    Hermes not found. Installing...
    This may take a few minutes...
    Hermes installed successfully
[3/7] Creating Hermes directory...
[4/7] Transferring configuration files...
    ✓ config.yaml
    ✓ .env (API keys)
    ✓ auth.json
[5/7] Transferring memory...
    ✓ Memory transferred (8 files)
[6/7] Transferring session history...
    ✓ Sessions transferred (156 files)
[7/7] Transferring custom skills...
    ℹ No custom skills found (optional)

==================================
Verifying transfer...
==================================

✓ Hermes is installed
✓ Configuration file present
✓ Environment file (API keys) present
✓ Memory transferred

==================================
Transfer Complete!
==================================
```

---

## Post-Transfer Verification

### Step 9: Test on New Machine

On your **source machine**:

```bash
# SSH to the VPS
ssh root@203.0.113.45

# Check Hermes
hermes doctor

# Start a conversation
hermes
```

**Test questions to ask Jimmy:**
- "What do you know about me?"
- "What's my timezone?"
- "What projects am I working on?"

If Jimmy remembers your details, the transfer was successful!

---

## Optional: Set Up Obsidian Vault on VPS

If you want to continue using your vault on the new machine:

```bash
# SSH to VPS
ssh root@203.0.113.45

# Create Documents directory
mkdir -p ~/Documents

# Clone your vault
git clone https://github.com/jmolina500/obsidian-vault.git "$HOME/Documents/Obsidian Vault"

# Verify it cloned
ls -la "$HOME/Documents/Obsidian Vault"
```

---

## Troubleshooting

### "Connection refused" or "Connection timed out"

**Problem:** Can't reach the VPS

**Solutions:**
1. Check the IP address is correct
2. Verify SSH is running: `sudo systemctl status sshd`
3. Check firewall allows port 22
4. Check VPS provider's security groups/web firewall

### "Permission denied (publickey)"

**Problem:** SSH key not accepted

**Solutions:**
1. Run `ssh-copy-id` again: `ssh-copy-id root@203.0.113.45`
2. Check key permissions: `chmod 600 ~/.ssh/id_ed25519`
3. On VPS, check authorized_keys: `cat ~/.ssh/authorized_keys`

### "Hermes: command not found" after transfer

**Problem:** Hermes not in PATH

**Solutions:**
1. Reload shell: `source ~/.bashrc` or `source ~/.zshrc`
2. Log out and back in
3. Check installation: `ls -la ~/.local/bin/hermes`

### Memory not transferred

**Problem:** Jimmy doesn't remember you

**Check:**
```bash
# On VPS
ls -la ~/.hermes/memory/
# Should show files

# If empty, re-run transfer focusing on memory:
scp -r ~/.hermes/memory root@203.0.113.45:~/.hermes/
```

---

## What Gets Transferred

| Component | Required? | Description |
|-----------|-----------|-------------|
| `config.yaml` | **Yes** | All Hermes settings |
| `.env` | **Yes** | API keys and secrets |
| `auth.json` | **Yes** | OAuth tokens |
| `memory/` | **Yes** | What Jimmy knows about you |
| `sessions/` | Optional | Past conversation history |
| `skills/` | Optional | Custom skills |

---

## Security Notes

- **Never share your `.env` file** - it contains API keys
- **Use SSH keys, not passwords** for VPS access
- **The transfer happens over SSH** (encrypted)
- **Delete the script after use** if it contains sensitive info
- **Rotate API keys** if you suspect they were compromised

---

## Quick Reference

```bash
# 1. Get VPS IP
curl ifconfig.me

# 2. Copy SSH key
ssh-copy-id root@203.0.113.45

# 3. Run transfer
./transfer-hermes.sh root@203.0.113.45

# 4. Test
ssh root@203.0.113.45
hermes
```
