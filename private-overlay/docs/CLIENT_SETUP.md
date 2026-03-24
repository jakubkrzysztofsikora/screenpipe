# Client Machine Setup Guide

Complete guide to setting up a client machine to sync screen capture data to the central server.

## Overview

Each client machine runs:
1. **screenpipe** — captures screen content (OCR), audio transcriptions, and metadata
2. **sync-daemon** — pushes captured data to the central sync server every 60 seconds

The sync daemon connects to the server via SSH tunnel or cloudflared HTTPS (for corporate VPN environments).

**Time required:** ~10 minutes

**Supported platforms:** macOS (Apple Silicon & Intel), Linux x64

## Prerequisites

| Requirement | Details |
|---|---|
| Server | Sync server deployed and running (see [SERVER_SETUP.md](SERVER_SETUP.md)) |
| SYNC_TOKEN | 64-char hex token from the server's `/opt/screenpipe-private/.env` |
| Python 3.9+ | Pre-installed on macOS; `apt install python3` on Linux |
| Node.js 18+ | For installing screenpipe via npx (or download binary directly) |
| Server URL or SSH access | cloudflared URL **or** SSH key access to the VM |

## Step 1 — Install screenpipe

### Option A — npx (easiest, no Rust needed)

```sh
# First run downloads the platform binary into the npx cache
npx screenpipe@latest record --help

# Symlink to PATH
# macOS Apple Silicon:
ln -sf ~/.npm/_npx/*/node_modules/@screenpipe/cli-darwin-arm64/bin/screenpipe /opt/homebrew/bin/screenpipe

# macOS Intel:
ln -sf ~/.npm/_npx/*/node_modules/@screenpipe/cli-darwin-x64/bin/screenpipe /usr/local/bin/screenpipe

# Linux x64:
ln -sf ~/.npm/_npx/*/node_modules/@screenpipe/cli-linux-x64/bin/screenpipe /usr/local/bin/screenpipe

# Verify
screenpipe --version
```

### Option B — Download release binary

Download from [screenpipe releases](https://github.com/mediar-ai/screenpipe/releases), extract, and place in your PATH.

### Option C — Desktop app

Download the `.dmg` (macOS) or `.exe` (Windows) from [screenpi.pe/onboarding](https://screenpi.pe/onboarding). The desktop app includes the CLI binary.

## Step 2 — Run the setup script

```sh
cd screenpipe/private-overlay
./local/setup-machine.sh
```

### What `setup-machine.sh` does

1. Verifies Python 3.9+
2. Creates `~/.screenpipe-private/logs/` and `~/.config/screenpipe-private/`
3. Copies `.env.example` to `~/.config/screenpipe-private/.env`
4. Generates an SSH keypair at `~/.ssh/screenpipe_vm_ed25519`
5. Prints the `authorized_keys` entry for the VM (SSH tunnel mode)
6. Blocks telemetry domains in `/etc/hosts` (requires sudo)
7. Installs the sync daemon as a macOS LaunchAgent or Linux systemd service

## Step 3 — Configure the `.env` file

```sh
nano ~/.config/screenpipe-private/.env
```

### Required fields

```sh
# ── VM Connection (SSH tunnel mode) ───────────────────────────
VM_HOST=51.158.64.55          # VM IP address
VM_USER=ubuntu                 # SSH username on VM
VM_SSH_KEY=~/.ssh/screenpipe_vm_ed25519

# ── Sync Authentication ──────────────────────────────────────
SYNC_TOKEN=<paste-64-char-hex-token-from-server>

# ── Local Machine Identity ───────────────────────────────────
MACHINE_NAME=macbook-pro-jakub  # Unique per machine, no spaces

# ── Screenpipe Settings ──────────────────────────────────────
SCREENPIPE_DATA_DIR=~/.screenpipe

# ── Sync Daemon ──────────────────────────────────────────────
SYNC_INTERVAL_SECONDS=60
SYNC_TUNNEL_LOCAL_PORT=8765
OLLAMA_TUNNEL_LOCAL_PORT=11434
```

### cloudflared mode (when SSH is blocked)

If your corporate VPN blocks SSH (e.g. Cato Networks), add the cloudflared URL instead:

```sh
# Replace SSH tunnel with cloudflared HTTPS
SYNC_SERVER_URL=https://xxxx.trycloudflare.com
# VM_HOST is not needed in this mode
```

Get the URL from the server admin (see [SERVER_SETUP.md — cloudflared](SERVER_SETUP.md#option-b--cloudflared-when-ssh-is-blocked)).

## Step 4 — Add SSH key to VM (SSH tunnel mode only)

Skip this step if using cloudflared mode.

The setup script printed an `authorized_keys` entry. Add it to the VM:

```sh
# On the VM:
echo 'command="echo tunnel-only",no-pty,no-X11-forwarding,no-agent-forwarding,permitopen="localhost:8765",permitopen="localhost:11434" ssh-ed25519 AAAA... screenpipe-macbook-20260322' >> ~/.ssh/authorized_keys
```

This restricts the key to tunnel-only — no shell access.

## Step 5 — Update the LaunchAgent plist (macOS)

The setup script installs the plist, but you need to fill in the environment variables:

```sh
nano ~/Library/LaunchAgents/com.screenpipe.private.sync.plist
```

Set these values inside `<dict>` under `EnvironmentVariables`:

```xml
<key>SCREENPIPE_DATA_DIR</key><string>/Users/yourname/.screenpipe</string>
<key>SYNC_TOKEN</key><string>your-64-char-hex-token</string>
<key>MACHINE_NAME</key><string>macbook-pro-jakub</string>
<key>SYNC_TUNNEL_LOCAL_PORT</key><string>8765</string>
<key>SYNC_INTERVAL_SECONDS</key><string>60</string>
```

For cloudflared mode, also add:

```xml
<key>SYNC_SERVER_URL</key><string>https://xxxx.trycloudflare.com</string>
```

### Linux: systemd service

The setup script creates `~/.config/systemd/user/screenpipe-sync.service` which reads from the `.env` file directly. No additional editing needed.

## Step 6 — Load the sync daemon

### macOS

```sh
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.screenpipe.private.sync.plist
```

### Linux

```sh
systemctl --user enable --now screenpipe-sync
```

## Step 7 — Grant macOS permissions

Before screenpipe can capture, macOS requires permissions:

1. **Screen Recording** (required): System Settings → Privacy & Security → Screen Recording → enable your terminal app
2. **Accessibility** (optional): enables input capture — same path under Accessibility

## Step 8 — Start screenpipe

```sh
cd screenpipe/private-overlay
./local/start-screenpipe.sh
```

The script auto-detects the transport:
- If `SYNC_SERVER_URL` is set → cloudflared mode (no SSH tunnel started)
- If `VM_HOST` is set → SSH tunnel mode (starts `tunnel-manager.sh` in background)

Expected output:

```
Transport: cloudflared (SYNC_SERVER_URL=https://xxxx.trycloudflare.com)
  → sync-daemon connects via HTTPS, no local SSH tunnel required
Starting screenpipe: machine=macbook-pro-jakub data=/Users/yourname/.screenpipe
```

Screenpipe runs in the foreground. To run in the background:

```sh
nohup ./local/start-screenpipe.sh > /tmp/screenpipe.log 2>&1 &
```

## Verify sync is working

### Check sync daemon logs

```sh
tail -f ~/.screenpipe-private/logs/sync-daemon.log
```

Expected output after the first 60-second cycle:

```
INFO: Sync daemon started: machine=macbook-pro-jakub db=/Users/you/.screenpipe/db.sqlite interval=60s server=https://xxxx.trycloudflare.com
INFO: Synced: table=ocr_text rows=8 inserted=8 skipped=0
INFO: Synced: table=frames rows=53 inserted=53 skipped=0
```

### Check sync state

```sh
cat ~/.screenpipe-private/.sync_state.json
```

Shows `last_rowid` and `rows_pushed` per table:

```json
{
  "ocr_text": { "last_rowid": 42, "rows_pushed": 42 },
  "frames": { "last_rowid": 58, "rows_pushed": 58 }
}
```

### Check screenpipe is recording

```sh
curl http://localhost:3030/health
# → {"status":"ok",...}
```

### Test server connectivity

```sh
# SSH tunnel mode:
nc -z localhost 8765 && echo "tunnel up" || echo "tunnel down"

# cloudflared mode:
curl https://xxxx.trycloudflare.com/health
# → {"status":"ok","version":"1.0.0"}
```

## File locations

| Path | Purpose |
|---|---|
| `~/.config/screenpipe-private/.env` | All configuration (mode 600) |
| `~/.screenpipe/db.sqlite` | Local screenpipe database |
| `~/.screenpipe-private/.sync_state.json` | Sync cursor state (rowid per table) |
| `~/.screenpipe-private/logs/sync-daemon.log` | Sync daemon rotating log (10 MB, 3 backups) |
| `~/.screenpipe-private/logs/sync-daemon.stdout.log` | LaunchAgent stdout |
| `~/.screenpipe-private/logs/sync-daemon.stderr.log` | LaunchAgent stderr |
| `~/.ssh/screenpipe_vm_ed25519` | SSH key for VM tunnel |
| `~/Library/LaunchAgents/com.screenpipe.private.sync.plist` | macOS LaunchAgent (sync daemon) |

## Adding another machine

Repeat steps 1–8 on each machine, using a **unique `MACHINE_NAME`** (e.g. `macstudio-jakub`, `macbook-air-office`). All machines share the same `SYNC_TOKEN` and server URL.

## Updating screenpipe

```sh
# Re-run npx to get the latest version
npx screenpipe@latest record --help

# Update the symlink (macOS Apple Silicon example)
ln -sf ~/.npm/_npx/*/node_modules/@screenpipe/cli-darwin-arm64/bin/screenpipe /opt/homebrew/bin/screenpipe

screenpipe --version
```

If telemetry blocking is enabled (`/etc/hosts` blocks `screenpi.pe`), auto-updates are disabled. Update manually as above.

## Troubleshooting

### "Local DB not found" in sync logs

screenpipe hasn't started yet or hasn't created the DB. Start screenpipe first, then the sync daemon will pick it up on the next cycle.

### "Connection refused" or "Connection error"

- **SSH mode:** tunnel isn't running. Check `tunnel-manager.sh` or run `nc -z localhost 8765`
- **cloudflared mode:** check the URL is correct and the VM's `cloudflared-sync` service is running

### Sync daemon not pushing data

Check the log for errors:

```sh
tail -50 ~/.screenpipe-private/logs/sync-daemon.log
```

Common issues:
- `HTTP 401` → SYNC_TOKEN mismatch between client and server
- `HTTP 400` → schema mismatch; ensure both client and server are on the same version
- `HTTP 500` → server-side error; check `docker compose logs sync-server` on the VM

### macOS permissions not granted

If screenpipe says "screen recording: missing", go to System Settings → Privacy & Security → Screen Recording and enable the terminal app you're using (Terminal, iTerm2, etc.). Restart screenpipe after granting.

### LaunchAgent not starting

```sh
# Check if loaded
launchctl list | grep screenpipe

# Reload
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.screenpipe.private.sync.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.screenpipe.private.sync.plist

# Check stderr for Python errors
cat ~/.screenpipe-private/logs/sync-daemon.stderr.log
```
