# Migration Guide: Ubuntu → macOS

> Moving the multi-agent fleet from Ubuntu Desktop to Mac Mini (24/7 host)

---

## Prerequisites on macOS

Before starting, ensure these are installed on the Mac Mini:

```bash
# 1. Homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. uv (Python package manager — used instead of pip/venv directly)
brew install uv

# 3. Node.js (for WhatsApp bridge)
brew install node
# OR use nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install --lts

# 4. Git
brew install git
```

---

## Step 1: Transfer the Fleet Directory

The fleet lives in `~/.hermes/` (the real directory). Transfer the full directory tree
including all agent homes.

**From Ubuntu:**
```bash
# Option A: rsync over SSH (recommended — handles symlinks)
rsync -avz -L ~/.hermes/ deeog@MAC_MINI_IP:~/.hermes-transfer/

# Option B: tar archive (preserves symlinks)
tar czhf hermes-fleet.tar.gz -C ~ .hermes .oracle .eureka .aurus
scp hermes-fleet.tar.gz deeog@MAC_MINI_IP:~/
```

**On macOS:**
```bash
# If using tar:
cd ~
tar xzf hermes-fleet.tar.gz

# If using rsync:
mv ~/.hermes-transfer ~/.hermes
```

**IMPORTANT:** Do NOT copy these directories (platform-specific, will be rebuilt):
- `~/.hermes/hermes-agent/venv/` — Linux Python binary, must be recreated
- `~/.hermes/hermes-agent/scripts/whatsapp-bridge/node_modules/` — Linux native modules
- `~/.hermes/bin/tirith` — Linux ELF binary

```bash
# Clean these before or after transfer
rm -rf ~/.hermes/hermes-agent/venv/
rm -rf ~/.hermes/hermes-agent/scripts/whatsapp-bridge/node_modules/
rm -f ~/.hermes/bin/tirith
```

---

## Step 2: Handle the Directory Restructuring

The directory layout is being restructured. The old layout had `~/.hermes` as a symlink
pointing to `~/.arch` (the real directory). The new layout reverses this:

- **`~/.hermes`** is now the REAL directory (primary)
- **`~/.arch`** is a backward-compatibility symlink → `~/.hermes`
- **`~/.oracle`**, **`~/.eureka`**, **`~/.aurus`** are proper CLI entry points, each with
  their own `SOUL.md` and `config.yaml`

### If migrating BEFORE the rename on Ubuntu:

The transferred data will still have the old layout (`~/.arch` is real, `~/.hermes` is a
symlink). Perform the rename on macOS:

```bash
# 1. Remove the old symlink (if it exists)
rm -f ~/.hermes

# 2. Rename the real directory
mv ~/.arch ~/.hermes

# 3. Create a backward-compat symlink for anything referencing ~/.arch
ln -sfn ~/.hermes ~/.arch

# 4. Verify
ls -la ~/.arch
# Should show: ~/.arch -> /Users/deeog/.hermes
ls -la ~/.hermes
# Should show a real directory (no arrow)
```

### If migrating AFTER the rename on Ubuntu:

The rename already happened before transfer. `~/.hermes` is already the real directory.
Just create the backward-compat symlink:

```bash
# Create backward-compat symlink
ln -sfn ~/.hermes ~/.arch

# Verify
ls -la ~/.arch
# Should show: ~/.arch -> /Users/deeog/.hermes
```

**Cross-platform note:** All symlinks inside agent homes use relative paths
(e.g., `../.hermes/.env`) specifically for portability between Ubuntu and macOS.
This means home directory differences (`/home/deeog/` vs `/Users/deeog/`) are
handled automatically.

---

## Step 3: Recreate the Python Virtual Environment

The venv contains a Linux x86_64 Python binary. It must be recreated on macOS.

```bash
cd ~/.hermes/hermes-agent

# Create a fresh venv with macOS Python
uv venv venv

# Install hermes and all dependencies
uv pip install -e .
```

**Verify:**
```bash
./venv/bin/python --version
# Should show Python 3.11+ (macOS native)

which hermes
# Should point to a binary using the new venv

# If hermes CLI is not found or points to old venv:
uv pip install -e .
# Or manually:
./venv/bin/pip install -e .
```

---

## Step 4: Fix Symlinks in Agent Homes

The agent home directories (`~/.oracle`, `~/.eureka`, `~/.aurus`) may have symlinks with
absolute Ubuntu paths (`/home/deeog/.hermes/...`). On macOS, home is `/Users/deeog/...`.

Each agent home is a proper CLI entry point with its own identity files. Recreate all
symlinks with relative paths and ensure per-agent files exist:

```bash
# Oracle
cd ~/.oracle
ln -sf ../.hermes/.env .env
ln -sf ../.hermes/auth.json auth.json
ln -sf ../.hermes/agent-templates/SHARED_FOUNDATION.md SHARED_FOUNDATION.md
# Per-agent identity files (create if missing):
touch SOUL.md config.yaml

# Eureka
cd ~/.eureka
ln -sf ../.hermes/.env .env
ln -sf ../.hermes/auth.json auth.json
ln -sf ../.hermes/agent-templates/SHARED_FOUNDATION.md SHARED_FOUNDATION.md
# Per-agent identity files (create if missing):
touch SOUL.md config.yaml

# Aurus
cd ~/.aurus
ln -sf ../.hermes/.env .env
ln -sf ../.hermes/auth.json auth.json
ln -sf ../.hermes/agent-templates/SHARED_FOUNDATION.md SHARED_FOUNDATION.md
# Per-agent identity files (create if missing):
touch SOUL.md config.yaml
```

**Per-agent files explained:**
- `SOUL.md` — Each agent's personality/identity definition. This is unique per agent
  and defines how it presents itself. Populate with agent-specific content.
- `config.yaml` — Per-agent configuration overrides. Agents read their own config
  first, falling back to `~/.hermes/config.yaml` for shared settings.

**Verify:**
```bash
# Symlinks should show relative targets, not /home/deeog/...
ls -la ~/.oracle/.env ~/.eureka/.env ~/.aurus/.env
ls -la ~/.oracle/auth.json ~/.eureka/auth.json ~/.aurus/auth.json
ls -la ~/.oracle/SHARED_FOUNDATION.md ~/.eureka/SHARED_FOUNDATION.md ~/.aurus/SHARED_FOUNDATION.md

# Per-agent files should exist as real files
ls -la ~/.oracle/SOUL.md ~/.eureka/SOUL.md ~/.aurus/SOUL.md
ls -la ~/.oracle/config.yaml ~/.eureka/config.yaml ~/.aurus/config.yaml
```

---

## Step 5: Rebuild WhatsApp Bridge Dependencies

```bash
cd ~/.hermes/hermes-agent/scripts/whatsapp-bridge

# Clean install (native modules like sharp need macOS binaries)
rm -rf node_modules
npm install
```

---

## Step 6: Install Gateway as macOS Service (launchd)

Hermes includes built-in launchd support. Do NOT try to use systemd on macOS.

```bash
cd ~/.hermes/hermes-agent

# Install as a launchd agent
./scripts/hermes-gateway install

# Start the gateway
./scripts/hermes-gateway start
```

**What this does:**
- Creates `~/Library/LaunchAgents/com.hermes.gateway.plist`
- Sets HERMES_HOME, PATH, VIRTUAL_ENV environment variables
- Configures auto-restart on crash
- Starts the gateway process

**Verify:**
```bash
# Check service status
./scripts/hermes-gateway status

# Check logs
./scripts/hermes-gateway logs

# Or manually:
launchctl list | grep hermes
```

---

## Step 7: Fix tirith Security Binary

tirith is a Linux-only ELF binary. Two options:

**Option A: Disable tirith on macOS (safe — it's configured to fail open)**
```yaml
# In ~/.hermes/config.yaml
security:
  tirith_enabled: false
```

**Option B: Obtain macOS binary** (if available from tirith project)

---

## Step 8: Verify the Fleet

Run through each channel and verify the agent responds:

```bash
# 1. Check gateway is running
ps aux | grep hermes

# 2. Test Discord connectivity
#    Send "hi" in each Discord channel:
#    - #oracle (or #assistant)
#    - #arch (or #code)
#    - #eureka (or #content)
#    - #aurus (or #money)

# 3. Test WhatsApp (Oracle only)
#    Send a message via WhatsApp self-chat

# 4. Check sessions are being created
ls -lt ~/.hermes/sessions/ | head -10

# 5. Check memories are accessible
ls -la ~/.hermes/memories/

# 6. Verify CLI access per agent (each agent home should resolve correctly)
HERMES_HOME=~/.oracle hermes --help
HERMES_HOME=~/.eureka hermes --help
HERMES_HOME=~/.aurus hermes --help

# 7. Verify backward-compat access via ~/.arch
HERMES_HOME=~/.arch hermes --help
```

If any `HERMES_HOME` test fails, check that the agent home directory exists and contains
the expected symlinks (Step 4).

---

## Step 9: Enable Auto-Start on Boot

launchd handles this automatically if installed correctly. Verify:

```bash
# The plist should be in LaunchAgents
ls ~/Library/LaunchAgents/com.hermes.gateway.plist

# Check it's loaded
launchctl list | grep hermes
```

If not auto-starting:
```bash
launchctl load ~/Library/LaunchAgents/com.hermes.gateway.plist
```

---

## Portability Audit Summary

| # | Area | Severity | Fix |
|---|------|----------|-----|
| 1 | systemd service | BLOCKER | Use `scripts/hermes-gateway install` (launchd support built-in) |
| 2 | Python venv (Linux binary) | BLOCKER | `uv venv venv && uv pip install -e .` |
| 3 | Hermes CLI shebang | BLOCKER | Reinstalled automatically with venv recreation |
| 4 | Directory restructuring (arch → hermes) | BLOCKER | Rename `~/.arch` → `~/.hermes`, create compat symlink (Step 2) |
| 5 | Agent home symlinks (absolute paths) | WARNING | Recreate with relative paths (Step 4) |
| 6 | Per-agent identity files | WARNING | Create `SOUL.md` and `config.yaml` in each agent home (Step 4) |
| 7 | tirith binary (Linux ELF) | WARNING | Disable or obtain macOS binary (Step 7) |
| 8 | WhatsApp bridge node_modules | WARNING | `npm install` fresh on macOS (Step 5) |

Items that are already portable (no changes needed):
- Node bootstrap script (already has macOS support)
- Config files and `.env` (no hardcoded Linux paths)
- Network setup (standard HTTP/WS)
- File watchers (uses portable mtime checks)
- Relative symlinks (portable across Ubuntu ↔ macOS)

---

## Optional: Make the Ubuntu Setup Portable Too

To prevent drift between the two machines, consider applying the directory restructuring
on Ubuntu as well:

```bash
# On Ubuntu — rename ~/.arch to ~/.hermes (if not already done)
mv ~/.arch ~/.hermes
ln -sfn ~/.hermes ~/.arch

# On Ubuntu — make symlinks relative in agent homes
for agent_home in ~/.oracle ~/.eureka ~/.aurus; do
    cd "$agent_home"
    ln -sf ../.hermes/.env .env
    ln -sf ../.hermes/auth.json auth.json
    ln -sf ../.hermes/agent-templates/SHARED_FOUNDATION.md SHARED_FOUNDATION.md
    # Ensure per-agent files exist
    touch SOUL.md config.yaml
done
```

This way both machines use the same relative symlink structure and the same primary
directory (`~/.hermes`), making future transfers seamless.

---

## Rollback Plan

If the macOS setup has issues and you need to fall back to Ubuntu:

1. Ubuntu setup is untouched — just start the gateway there
2. Be aware: sessions/memories created on macOS won't be on Ubuntu unless synced
3. Consider rsyncing `~/.hermes/sessions/` and `~/.hermes/memories/` between machines if running both

---

## Future: Dual-Machine Operation

Once the macOS setup is stable, you may want to run agents on both machines:
- Mac Mini: 24/7 gateway (Oracle, WhatsApp, scheduled tasks)
- Ubuntu Desktop: Development agent (Arch, heavy builds)

This requires either:
- Separate Discord bot tokens for each machine (Option C architecture)
- Or: keep one gateway on Mac Mini and access Arch via CLI on Ubuntu when needed
