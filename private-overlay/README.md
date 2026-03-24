# Screenpipe Private Overlay

A security-hardened, multi-machine overlay for [screenpipe](https://github.com/screenpipe/screenpipe). Syncs screen capture data across machines via a central server — all traffic either over SSH tunnels or cloudflared HTTPS, zero plain-internet exposure.

**This overlay never modifies screenpipe core files.** It sits alongside screenpipe and can be updated independently.

## Quick links

- **[Server Setup Guide](docs/SERVER_SETUP.md)** — full guide to deploying the VM and sync server
- **[Client Setup Guide](docs/CLIENT_SETUP.md)** — full guide to setting up each client machine

## Architecture

```
┌──────────────────────┐      ┌──────────────────────┐
│  Machine A (local)   │      │  Machine B (local)   │
│                      │      │                      │
│  screenpipe record   │      │  screenpipe record   │
│  --disable-telemetry │      │  --disable-telemetry │
│  --sync-machine-id A │      │  --sync-machine-id B │
│                      │      │                      │
│  ~/.screenpipe/      │      │  ~/.screenpipe/      │
│  db.sqlite (local)   │      │  db.sqlite (local)   │
│                      │      │                      │
│  sync-daemon.py      │      │  sync-daemon.py      │
│  (every 60s)         │      │  (every 60s)         │
└────────┬─────────────┘      └────────┬─────────────┘
         │                             │
         │  SSH tunnel OR cloudflared  │
         │  (see Transport section)    │
         │                             │
         └──────────────┬──────────────┘
                        │
         ┌──────────────▼──────────────────────────┐
         │          Scaleway VM (server)            │
         │                                          │
         │  Docker Compose:                         │
         │    sync-server (FastAPI :8765, localhost) │
         │    ollama (:11434, localhost, opt-in)     │
         │                                          │
         │  /data/screenpipe-central/               │
         │    db.sqlite (merged, all machines)      │
         │                                          │
         │  LUKS encrypted /data volume             │
         │  Firewall: port 22 only (SSH)            │
         └──────────────────────────────────────────┘
```

Each machine captures its own screen data (`machine_id` = hostname). Rows are unique per machine — no conflicts, no CRDTs needed. The sync daemon pushes new rows to the central server every 60 seconds.

## Prerequisites

- **screenpipe** installed on each client machine (see [Installing screenpipe](#installing-screenpipe))
- **Python 3.9+** on client machines (stdlib only — no pip installs)
- **Docker + Docker Compose v2** on the VM
- macOS or Linux on client machines

## Installing screenpipe

The simplest install — no Rust toolchain required:

```sh
# First run downloads the binary into the npx cache
npx screenpipe@latest record --help

# Symlink the cached binary to PATH (adjust path for your platform/arch)
# macOS Apple Silicon:
ln -sf ~/.npm/_npx/*/node_modules/@screenpipe/cli-darwin-arm64/bin/screenpipe /opt/homebrew/bin/screenpipe
# macOS Intel:
ln -sf ~/.npm/_npx/*/node_modules/@screenpipe/cli-darwin-x64/bin/screenpipe /opt/homebrew/bin/screenpipe
# Linux x64:
ln -sf ~/.npm/_npx/*/node_modules/@screenpipe/cli-linux-x64/bin/screenpipe /usr/local/bin/screenpipe

screenpipe --version   # verify
```

Or download a release binary directly from [screenpipe releases](https://github.com/mediar-ai/screenpipe/releases) and place it in your PATH.

## VM Setup

1. SSH to your VM (or access via VS Code tunnel if SSH is blocked — see [Transport](#transport-ssh-vs-cloudflared)):

   ```sh
   ssh ubuntu@your-vm-ip
   ```

2. Clone this repo and run the setup script:

   ```sh
   cd screenpipe/private-overlay
   sudo ./vm/setup.sh
   ```

   This will:
   - Auto-detect the block device (`/dev/sda` or `/dev/sdb`) and set up LUKS encryption
   - Deploy the Docker Compose stack (sync-server; Ollama is opt-in)
   - Generate a `SYNC_TOKEN`
   - Configure UFW firewall (SSH only)

3. Note the `SYNC_TOKEN` from `/opt/screenpipe-private/.env` — you'll need it on each client.

## Client Machine Setup

Run these steps **on each machine** you want to sync:

1. Clone the repo and run setup:

   ```sh
   cd screenpipe/private-overlay
   ./local/setup-machine.sh
   ```

2. Edit the config file:

   ```sh
   nano ~/.config/screenpipe-private/.env
   ```

   Required fields:

   | Variable | Description |
   |----------|-------------|
   | `SYNC_TOKEN` | From VM `/opt/screenpipe-private/.env` |
   | `MACHINE_NAME` | Unique name for this machine (no spaces) |
   | `SCREENPIPE_DATA_DIR` | Path to screenpipe data dir (default `~/.screenpipe`) |
   | `VM_HOST` | VM IP — required for SSH tunnel mode |
   | `SYNC_SERVER_URL` | cloudflared URL — set instead of `VM_HOST` if SSH is blocked |

3. **SSH tunnel mode only** — add your SSH public key to the VM with tunnel restrictions:

   The setup script prints the exact `authorized_keys` entry:
   ```
   command="echo tunnel-only",no-pty,no-X11-forwarding,no-agent-forwarding,permitopen="localhost:8765",permitopen="localhost:11434" ssh-ed25519 AAAA... screenpipe-macbook-20260322
   ```
   Paste this into `~/.ssh/authorized_keys` on the VM.

4. Load the LaunchAgent (macOS):

   ```sh
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.screenpipe.private.sync.plist
   ```

   Or enable the systemd service (Linux):

   ```sh
   systemctl --user enable --now screenpipe-sync
   ```

5. Start screenpipe:

   ```sh
   ./local/start-screenpipe.sh
   ```

   This auto-detects the transport: starts `tunnel-manager.sh` for SSH mode, or skips it for cloudflared mode.

## Transport: SSH vs cloudflared

The sync daemon supports two transports. Set one in `~/.config/screenpipe-private/.env`:

### SSH tunnel (default)

Standard path — requires SSH access to the VM.

```sh
# In .env:
VM_HOST=51.158.64.55
VM_USER=ubuntu
VM_SSH_KEY=~/.ssh/screenpipe_vm_ed25519
# SYNC_SERVER_URL is not set (or empty)
```

`start-screenpipe.sh` launches `tunnel-manager.sh` which keeps `localhost:8765 → VM:8765` alive. The sync daemon connects to `http://localhost:8765`.

### cloudflared quick tunnel (Cato VPN / SSH blocked)

For environments where SSH is blocked by corporate DPI (e.g. Cato Networks SASE). cloudflared makes an **outbound** HTTPS connection from the VM to Cloudflare's network — no SSH needed from the client.

**Step 1 — On the VM** (via VS Code tunnel terminal or serial console):

```sh
cd screenpipe/private-overlay
git pull
sudo bash vm/setup-cloudflared.sh
```

This installs cloudflared, creates a `cloudflared-sync` systemd service, and prints the tunnel URL:

```
SYNC_SERVER_URL=https://xxxx.trycloudflare.com
```

The URL is saved to `/run/cloudflared-sync-url.txt` and persists until the service restarts.

**Step 2 — On each client machine:**

```sh
# In ~/.config/screenpipe-private/.env:
SYNC_SERVER_URL=https://xxxx.trycloudflare.com

# In ~/Library/LaunchAgents/com.screenpipe.private.sync.plist,
# uncomment the SYNC_SERVER_URL key and set the same URL.

# Reload the daemon:
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.screenpipe.private.sync.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.screenpipe.private.sync.plist
```

`start-screenpipe.sh` detects `SYNC_SERVER_URL` and skips `tunnel-manager.sh` automatically.

**Note:** `trycloudflare.com` quick-tunnel URLs change on every service restart. For a stable URL, set up a [named Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) with a free Cloudflare account and domain.

## Verifying Everything Works

```sh
# Check sync daemon logs
tail -f ~/.screenpipe-private/logs/sync-daemon.log

# Check sync status across all machines
python3 private-overlay/client/query.py status

# Search across all machines
python3 private-overlay/client/query.py search "meeting notes"

# Show recent data
python3 private-overlay/client/query.py tail
```

## Security Notes

### What IS protected
- All sync traffic goes through SSH tunnels or cloudflared HTTPS — nothing exposed directly
- VM firewall allows only SSH (port 22); sync-server bound to `127.0.0.1` only
- Sync server uses timing-safe HMAC-SHA256 token authentication
- All SQL queries use parameterized statements; column names validated by allowlist
- SSH keys are restricted to tunnel-only (no shell access)
- Telemetry endpoints are blocked via `/etc/hosts`
- Central DB lives on a LUKS-encrypted volume (auto-unlock on reboot via keyfile)

### What is NOT protected
- **Local screenpipe database is unencrypted.** The `~/.screenpipe/db.sqlite` file is plain SQLite. Enable FileVault (macOS) or LUKS (Linux) for full-disk encryption.
- **cloudflared quick tunnels** expose the sync-server to any internet client who discovers the random `trycloudflare.com` URL. The SYNC_TOKEN provides auth, but prefer a named tunnel with Cloudflare Access for production use.
- **Screenpipe captures everything on screen** — this includes passwords, private messages, banking info. Be aware of what's being recorded.
- **Pipes (plugins)** have full access to screenpipe data. Never load a pipe you haven't vetted.

### Auto-updates blocked
`block-telemetry.sh` blocks `screenpi.pe`, which also prevents auto-updates. To update screenpipe:

1. Remove the `screenpi.pe` line from `/etc/hosts`
2. Re-run the npx install or download the new binary
3. Re-run `sudo ./local/block-telemetry.sh`

## Troubleshooting

### Sync daemon not pushing data
- Check daemon logs: `tail -f ~/.screenpipe-private/logs/sync-daemon.log`
- **SSH mode:** verify tunnel is up: `nc -z localhost 8765`
- **cloudflared mode:** `curl https://xxxx.trycloudflare.com/health` should return `{"status":"ok",...}`
- Verify token matches: compare `SYNC_TOKEN` in client `.env` and server `.env`
- Check screenpipe is running: `curl http://localhost:3030/health`

### SSH tunnel won't connect
- Verify your SSH key is in the VM's `authorized_keys`
- Check VM firewall: `sudo ufw status`
- Test SSH: `ssh -i ~/.ssh/screenpipe_vm_ed25519 ubuntu@your-vm`
- Check tunnel logs: `tail -f ~/.screenpipe-private/logs/tunnel.log`
- If SSH is blocked by corporate VPN → switch to cloudflared (see [Transport](#transport-ssh-vs-cloudflared))

### cloudflared tunnel URL changed
- On VM: `cat /run/cloudflared-sync-url.txt` for current URL
- Or: `journalctl -u cloudflared-sync -n 50 | grep trycloudflare`
- Update `SYNC_SERVER_URL` in `.env` and LaunchAgent plist, then reload daemon

### Sync server returning 500
- Check server logs: `docker compose logs sync-server` (on VM)
- Verify DB permissions: `ls -la /data/screenpipe-central/` (should be owned by UID 1000)
- Fix permissions: `sudo chown -R 1000:1000 /data/screenpipe-central`
- Restart: `docker compose restart sync-server`

### LUKS volume not mounted after VM reboot
- The keyfile at `/root/.luks-screenpipe.key` auto-unlocks via `/etc/crypttab`
- Manually unlock: `sudo cryptsetup open --key-file /root/.luks-screenpipe.key /dev/sda screenpipe-data && sudo mount /dev/mapper/screenpipe-data /data`

## File Structure

```
private-overlay/
├── README.md                          # This file
├── .env.example                       # Template for all env vars
├── .gitignore                         # Ignores secrets, DBs, logs
│
├── docs/
│   ├── SERVER_SETUP.md                # Full server installation guide
│   └── CLIENT_SETUP.md               # Full client installation guide
│
├── vm/                                # VM-side components
│   ├── docker-compose.yml             # sync-server stack (Ollama opt-in)
│   ├── setup.sh                       # One-shot VM provisioning (LUKS + Docker)
│   ├── setup-cloudflared.sh           # Install cloudflared quick tunnel on VM
│   └── sync-server/
│       ├── Dockerfile
│       ├── requirements.txt
│       ├── main.py                    # FastAPI sync server
│       └── schema.sql                 # Central DB schema
│
├── local/                             # Client machine components
│   ├── start-screenpipe.sh            # Launcher — auto-selects SSH or cloudflared
│   ├── sync-daemon.py                 # Push rows to central server
│   ├── tunnel-manager.sh              # SSH tunnel keepalive (SSH mode only)
│   ├── block-telemetry.sh             # Block telemetry via /etc/hosts
│   ├── setup-machine.sh               # One-shot machine setup
│   └── com.screenpipe.private.plist   # macOS LaunchAgent template
│
├── client/
│   └── query.py                       # CLI search across all machines
│
└── scripts/
    ├── generate-keys.sh               # Generate SSH keys
    └── health-check.sh                # Full stack health check
```
