# Transfer Hermes to New Machine via SSH

Created: 2025-04-20

One-command script to transfer Jimmy (Hermes) configuration, memory, and sessions to a new machine via SSH.

---

## Prerequisites

- SSH access from current machine to new machine
- Hermes installed on new machine (script will check and install if missing)
- SSH key or password authentication configured

---

## The Script

Save this as `transfer-hermes.sh`:

```bash
#!/bin/bash

# Transfer Hermes configuration to a new machine via SSH
# Usage: ./transfer-hermes.sh user@new-machine

set -e

NEW_MACHINE="$1"

if [ -z "$NEW_MACHINE" ]; then
    echo "Usage: $0 user@hostname"
    echo "Example: $0 jesus@192.168.1.100"
    exit 1
fi

echo "=== Transferring Hermes to $NEW_MACHINE ==="

# 1. Check if Hermes is installed on new machine
echo "[1/6] Checking Hermes installation on new machine..."
if ! ssh "$NEW_MACHINE" "command -v hermes &> /dev/null"; then
    echo "    Hermes not found. Installing..."
    ssh "$NEW_MACHINE" "curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash"
else
    echo "    Hermes already installed"
fi

# 2. Create .hermes directory on new machine
echo "[2/6] Creating .hermes directory..."
ssh "$NEW_MACHINE" "mkdir -p ~/.hermes"

# 3. Transfer critical configuration files
echo "[3/6] Transferring configuration files..."
scp ~/.hermes/config.yaml "$NEW_MACHINE:~/.hermes/" 2>/dev/null || echo "    config.yaml not found, skipping"
scp ~/.hermes/.env "$NEW_MACHINE:~/.hermes/" 2>/dev/null || echo "    .env not found, skipping"
scp ~/.hermes/auth.json "$NEW_MACHINE:~/.hermes/" 2>/dev/null || echo "    auth.json not found, skipping"

# 4. Transfer memory (what Jimmy knows about you)
echo "[4/6] Transferring memory..."
if [ -d ~/.hermes/memory ]; then
    scp -r ~/.hermes/memory "$NEW_MACHINE:~/.hermes/"
    echo "    Memory transferred"
else
    echo "    No memory directory found, skipping"
fi

# 5. Transfer session history (optional)
echo "[5/6] Transferring session history..."
if [ -d ~/.hermes/sessions ]; then
    scp -r ~/.hermes/sessions "$NEW_MACHINE:~/.hermes/"
    echo "    Sessions transferred"
else
    echo "    No sessions directory found, skipping"
fi

# 6. Transfer custom skills (optional)
echo "[6/6] Transferring custom skills..."
if [ -d ~/.hermes/skills ]; then
    scp -r ~/.hermes/skills "$NEW_MACHINE:~/.hermes/"
    echo "    Skills transferred"
else
    echo "    No skills directory found, skipping"
fi

# 7. Verify transfer
echo ""
echo "=== Verifying transfer ==="
ssh "$NEW_MACHINE" "hermes doctor && hermes config"

echo ""
echo "=== Transfer complete ==="
echo "Hermes is ready on $NEW_MACHINE"
echo ""
echo "Optional: Clone your Obsidian vault:"
echo "  ssh $NEW_MACHINE \"git clone https://github.com/jmolina500/obsidian-vault.git \\"\"
echo ""
```

---

## Usage

### 1. Save the script

```bash
cd ~/Documents/Obsidian\ Vault/Hermes/
micro transfer-hermes.sh
# Paste the script above, save with Ctrl+S, quit with Ctrl+Q
```

### 2. Make it executable

```bash
chmod +x transfer-hermes.sh
```

### 3. Run it

```bash
./transfer-hermes.sh user@new-machine

# Example:
./transfer-hermes.sh jesus@192.168.1.100
```

---

## What Gets Transferred

| Component | Path | Required? |
|-----------|------|-----------|
| Config | `~/.hermes/config.yaml` | Yes - all settings |
| Secrets | `~/.hermes/.env` | Yes - API keys |
| Auth | `~/.hermes/auth.json` | Yes - OAuth tokens |
| Memory | `~/.hermes/memory/` | Yes - what Jimmy knows |
| Sessions | `~/.hermes/sessions/` | Optional - history |
| Skills | `~/.hermes/skills/` | Optional - custom skills |

---

## Manual Transfer (Alternative)

If you prefer to run commands manually:

```bash
# Set target
NEW_MACHINE="user@hostname"

# 1. Install Hermes on new machine
ssh "$NEW_MACHINE" "curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash"

# 2. Create directory
ssh "$NEW_MACHINE" "mkdir -p ~/.hermes"

# 3. Copy files
scp ~/.hermes/config.yaml "$NEW_MACHINE:~/.hermes/"
scp ~/.hermes/.env "$NEW_MACHINE:~/.hermes/"
scp ~/.hermes/auth.json "$NEW_MACHINE:~/.hermes/"

# 4. Copy directories
scp -r ~/.hermes/memory "$NEW_MACHINE:~/.hermes/"
scp -r ~/.hermes/sessions "$NEW_MACHINE:~/.hermes/"
scp -r ~/.hermes/skills "$NEW_MACHINE:~/.hermes/"

# 5. Verify
ssh "$NEW_MACHINE" "hermes doctor"
```

---

## Post-Transfer Steps

### 1. Clone your vault (if needed)

```bash
ssh user@new-machine "git clone https://github.com/jmolina500/obsidian-vault.git ~/Documents/Obsidian\ Vault"
```

### 2. Install micro editor (optional)

```bash
ssh user@new-machine "apt install micro"  # Debian/Ubuntu
# or
ssh user@new-machine "brew install micro"  # macOS
```

### 3. Configure timezone (optional)

```bash
ssh user@new-machine "timedatectl set-timezone America/Puerto_Rico"
```

---

## Troubleshooting

### Permission denied
```bash
# Ensure SSH key is added
ssh-add ~/.ssh/id_rsa
# Or use password authentication
ssh-copy-id user@new-machine
```

### Large transfers fail
```bash
# Use rsync instead (resumable)
rsync -avz --progress ~/.hermes/ user@new-machine:~/.hermes/
```

### Hermes not found after transfer
```bash
# Reload shell or source profile
ssh user@new-machine "source ~/.bashrc && hermes doctor"
```

---

## Verification Checklist

After transfer, run on new machine:

```bash
hermes doctor           # Check all dependencies
hermes config           # Verify settings
hermes status --all     # Check components
hermes memory status    # Verify memory transferred
```

Then test:
```bash
hermes                  # Start a session, ask "what do you know about me?"
```

Jimmy should remember your name, preferences, timezone, and ActionLayer project.

---

## Notes

- The script is idempotent - safe to run multiple times
- Existing files on new machine will be overwritten
- Does NOT transfer running processes or cron jobs
- Does NOT transfer SSH keys or system-wide settings
- Memory transfer is critical - this is how Jimmy knows you
