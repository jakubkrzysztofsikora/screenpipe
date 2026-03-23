# Screenpipe Private Overlay

A security-hardened, multi-machine overlay for [screenpipe](https://github.com/screenpipe/screenpipe). Syncs screen capture data across machines via a central server — all traffic over SSH tunnels, zero internet exposure.

**This overlay never modifies screenpipe core files.** It sits alongside screenpipe and can be updated independently.

## Architecture

```
┌──────────────────────┐      ┌──────────────────────┐
│  Machine A (local)   │      │  Machine B (local)   │
│                      │      │                      │
│  screenpipe          │      │  screenpipe          │
│  --disable-telemetry │      │  --disable-telemetry │
│  --device-name A     │      │  --device-name B     │
│                      │      │                      │
│  ~/.screenpipe/      │      │  ~/.screenpipe/      │
│  db.sqlite (local)   │      │  db.sqlite (local)   │
│                      │      │                      │
│  sync-daemon.py      │      │  sync-daemon.py      │
│  (every 60s)         │      │  (every 60s)         │
└────────┬─────────────┘      └────────┬─────────────┘
         │                             │
         │  SSH tunnel (port-forward)  │
         │  localhost:8765 → VM:8765   │
         │  localhost:11434 → VM:11434 │
         │                             │
         └──────────────┬──────────────┘
                        │
         ┌──────────────▼──────────────────────────┐
         │          Scaleway VM (server)            │
         │                                          │
         │  Docker Compose:                         │
         │    sync-server (FastAPI :8765)            │
         │    ollama (:11434, localhost only)        │
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

- **screenpipe** installed and working on each client machine
- **Python 3.9+** on client machines
- **SSH access** to your VM (Scaleway, Hetzner, etc.)
- **Docker + Docker Compose v2** on the VM
- macOS or Linux on client machines

## VM Setup

1. SSH to your VM:
   ```sh
   ssh ubuntu@your-vm-ip
   ```

2. Clone this repo and run the setup script:
   ```sh
   cd private-overlay
   sudo ./vm/setup.sh
   ```
   This will:
   - Set up LUKS encryption (if `/dev/sdb` exists)
   - Deploy the Docker Compose stack (sync-server + Ollama)
   - Generate a `SYNC_TOKEN`
   - Configure the firewall (SSH only)
   - Pull the llama3.2 model

3. Note the `SYNC_TOKEN` from `/opt/screenpipe-private/.env` — you'll need it on each client machine.

## Client Machine Setup

Run these steps **on each machine** you want to sync:

1. Clone the repo and run setup:
   ```sh
   cd private-overlay
   ./local/setup-machine.sh
   ```

2. Edit the config file:
   ```sh
   nano ~/.config/screenpipe-private/.env
   ```
   Set at minimum: `VM_HOST`, `SYNC_TOKEN`, `MACHINE_NAME`

3. Add your SSH public key to the VM. The setup script prints the exact `authorized_keys` entry with tunnel-only restrictions:
   ```
   command="echo tunnel-only",no-pty,no-X11-forwarding,no-agent-forwarding,permitopen="localhost:8765",permitopen="localhost:11434" ssh-ed25519 AAAA... screenpipe-macbook-20260322
   ```
   Paste this into `~/.ssh/authorized_keys` on the VM.

4. Load the LaunchAgent (macOS):
   ```sh
   launchctl load ~/Library/LaunchAgents/com.screenpipe.private.sync.plist
   ```
   Or enable the systemd service (Linux):
   ```sh
   systemctl --user enable --now screenpipe-sync
   ```

5. Start the tunnel and screenpipe:
   ```sh
   ./local/tunnel-manager.sh &
   ./local/start-screenpipe.sh
   ```

## Verifying Everything Works

```sh
# Full stack health check
./scripts/health-check.sh

# Check sync status across all machines
python3 client/query.py status

# Search across all machines
python3 client/query.py search "meeting notes"

# Show recent data
python3 client/query.py tail
```

## Security Notes

### What IS protected
- All sync traffic goes through SSH tunnels — nothing exposed to the internet
- VM firewall allows only SSH (port 22)
- Sync server uses timing-safe token authentication
- All SQL queries use parameterized statements
- SSH keys are restricted to tunnel-only (no shell access)
- Telemetry endpoints are blocked via `/etc/hosts`
- Central DB lives on a LUKS-encrypted volume

### What is NOT protected
- **Local screenpipe database is unencrypted.** The `~/.screenpipe/db.sqlite` file is plain SQLite. Enable FileVault (macOS) or LUKS (Linux) for full-disk encryption.
- **Screenpipe captures everything on screen** — this includes passwords you type, private messages, banking info. Be aware of what's being recorded.
- **Pipes (plugins)** have full access to screenpipe data. Never load a pipe you haven't vetted.

### Auto-updates blocked
The `block-telemetry.sh` script blocks `screenpi.pe`, which also prevents auto-updates. To update screenpipe manually:
1. Remove the `screenpi.pe` line from `/etc/hosts`
2. Run `screenpipe --update`
3. Re-run `./local/block-telemetry.sh`

Or build from source:
```sh
git pull && cargo build --release
```

## Troubleshooting

### Tunnel won't connect
- Verify your SSH key is in the VM's `authorized_keys`
- Check VM firewall: `sudo ufw status`
- Test SSH manually: `ssh -i ~/.ssh/screenpipe_vm_ed25519 ubuntu@your-vm`
- Check tunnel logs: `tail -f ~/.screenpipe-private/logs/tunnel.log`

### Sync daemon not pushing data
- Check daemon logs: `tail -f ~/.screenpipe-private/logs/sync-daemon.log`
- Verify tunnel is up: `nc -z localhost 8765`
- Verify token matches: compare `SYNC_TOKEN` in client `.env` and server `.env`
- Check screenpipe is running: `curl http://localhost:3030/health`

### Sync server returning 500
- Check server logs: `docker compose logs sync-server` (on VM)
- Verify DB permissions: `ls -la /data/screenpipe-central/`
- Restart: `docker compose restart sync-server`

### Ollama not responding
- Check Ollama logs: `docker compose logs ollama`
- Verify model is loaded: `curl http://localhost:11434/api/tags`
- Re-pull model: `docker compose exec ollama ollama pull llama3.2`

## Updating Screenpipe

This overlay is independent of screenpipe itself. To update screenpipe:

```sh
git pull && cargo build --release
```

The overlay requires no changes when screenpipe updates — it never modifies core files.

## File Structure

```
private-overlay/
├── README.md                          # This file
├── .env.example                       # Template for all env vars
├── .gitignore                         # Ignores secrets, DBs, logs
│
├── vm/                                # VM-side components
│   ├── docker-compose.yml             # Ollama + sync-server stack
│   ├── setup.sh                       # One-shot VM provisioning
│   └── sync-server/
│       ├── Dockerfile
│       ├── requirements.txt
│       ├── main.py                    # FastAPI sync server
│       └── schema.sql                 # Central DB schema
│
├── local/                             # Client machine components
│   ├── start-screenpipe.sh            # Hardened screenpipe launcher
│   ├── sync-daemon.py                 # Push rows to central server
│   ├── tunnel-manager.sh             # SSH tunnel keepalive
│   ├── block-telemetry.sh            # Block telemetry via /etc/hosts
│   ├── setup-machine.sh              # One-shot machine setup
│   └── com.screenpipe.private.plist  # macOS LaunchAgent
│
├── client/
│   └── query.py                       # CLI search across all machines
│
└── scripts/
    ├── generate-keys.sh               # Generate SSH keys
    └── health-check.sh                # Full stack health check
```
