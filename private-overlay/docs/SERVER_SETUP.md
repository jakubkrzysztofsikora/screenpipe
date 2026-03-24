# Server Setup Guide

Complete guide to deploying the screenpipe private overlay sync server on a VM.

## Overview

The server runs a FastAPI sync service inside Docker that receives screen capture data from client machines, stores it in a LUKS-encrypted SQLite database, and provides cross-machine search. All data stays on your infrastructure — nothing touches third-party servers.

**Time required:** ~15 minutes

## Prerequisites

| Requirement | Details |
|---|---|
| VM provider | Scaleway, Hetzner, DigitalOcean, or any VPS with root access |
| Instance size | Minimum 2 vCPU / 4 GB RAM (e.g. Scaleway DEV1-M) |
| Block volume | Separate block storage, ≥30 GB (for LUKS-encrypted data) |
| OS | Ubuntu 22.04 or 24.04 LTS |
| Docker | Docker Engine + Docker Compose v2 (plugin) |
| Access | SSH or serial console access to the VM |

## Step 1 — Provision the VM

### Scaleway (CLI example)

```sh
# Create security group (deny all inbound except SSH)
scw instance security-group create \
  name=screenpipe-overlay-sg \
  inbound-default-policy=drop \
  outbound-default-policy=accept \
  stateful=true

# Add SSH inbound rule
scw instance security-group create-rule \
  security-group-id=<sg-id> \
  direction=inbound action=accept protocol=TCP \
  dest-port-from=22

# Create block volume (50 GB, SBS 5K IOPS)
scw block volume create \
  name=screenpipe-data \
  from-empty.size=50GB \
  perf-iops=5000

# Create instance
scw instance server create \
  type=DEV1-M \
  image=ubuntu_noble \
  name=screenpipe-overlay \
  security-group-id=<sg-id>

# Attach block volume
scw instance server attach-volume \
  server-id=<server-id> \
  volume-id=<volume-id> \
  volume-type=sbs_volume
```

### Other providers

Create a VM with Ubuntu 22.04/24.04, attach a block volume, and ensure only port 22 is open in the firewall.

## Step 2 — Install Docker

SSH into the VM and install Docker if not already present:

```sh
ssh ubuntu@<vm-ip>

# Install Docker (official method)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu

# Verify
docker compose version
# Should show: Docker Compose version v2.x.x
```

Log out and back in for the group change to take effect.

## Step 3 — Clone the repo and run setup

```sh
git clone https://github.com/<your-fork>/screenpipe.git
cd screenpipe/private-overlay

# Run the setup script as root
sudo ./vm/setup.sh
```

### What `setup.sh` does

1. **LUKS encryption** — auto-detects the block volume device (`/dev/sda` or `/dev/sdb`), formats it with LUKS, creates a keyfile at `/root/.luks-screenpipe.key`, and mounts to `/data`
2. **Data directories** — creates `/data/screenpipe-central` (owned by UID 1000 for the container)
3. **Docker stack** — copies `docker-compose.yml` and sync-server to `/opt/screenpipe-private/`, builds and starts containers
4. **SYNC_TOKEN** — generates a 64-character hex token (or accepts yours), saves to `/opt/screenpipe-private/.env`
5. **Firewall** — configures UFW to deny all inbound except SSH
6. **Health check** — verifies the sync server responds at `http://localhost:8765/health`

### Override block device detection

If auto-detection fails, specify the device manually:

```sh
sudo DATA_BLOCK_DEV=/dev/sdb ./vm/setup.sh
```

### Expected output

```
╔══════════════════════════════════════════════════════════════╗
║  Screenpipe Private Overlay — VM Setup Complete             ║
╠══════════════════════════════════════════════════════════════╣
║  VM IP: 51.158.64.55                                       ║
║  SYNC_TOKEN: stored in /opt/screenpipe-private/.env        ║
╚══════════════════════════════════════════════════════════════╝
```

## Step 4 — Note your SYNC_TOKEN

```sh
sudo cat /opt/screenpipe-private/.env
# SYNC_TOKEN=<your-64-char-hex-token>
```

Save this token — every client machine needs it.

## Step 5 — Set up transport (SSH or cloudflared)

Clients need a transport to reach the sync server on `localhost:8765`. Choose one:

### Option A — SSH tunnel (default)

No additional server setup needed. Clients connect via `ssh -L 8765:localhost:8765`.

Ensure the client's SSH public key is in `~/.ssh/authorized_keys` on the VM with tunnel-only restrictions:

```
command="echo tunnel-only",no-pty,no-X11-forwarding,no-agent-forwarding,permitopen="localhost:8765",permitopen="localhost:11434" ssh-ed25519 AAAA... screenpipe-macbook-20260322
```

### Option B — cloudflared (when SSH is blocked)

If a corporate VPN (e.g. Cato Networks) blocks SSH with DPI, use cloudflared to expose the sync server via HTTPS:

```sh
sudo bash vm/setup-cloudflared.sh
```

This installs cloudflared, creates a `cloudflared-sync` systemd service, and prints a tunnel URL:

```
SYNC_SERVER_URL=https://xxxx.trycloudflare.com
```

Give this URL to each client machine.

**Notes:**
- The URL is saved to `/run/cloudflared-sync-url.txt` and persists until the service restarts
- Quick-tunnel URLs change on service restart — for a stable URL, set up a [named Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- The sync-server auth token provides security, but prefer named tunnels with Cloudflare Access for production

To check the URL later:

```sh
cat /run/cloudflared-sync-url.txt
```

## Verify the server is working

```sh
# Health check
curl http://localhost:8765/health
# → {"status":"ok","version":"1.0.0"}

# Check Docker containers
docker compose -f /opt/screenpipe-private/docker-compose.yml ps
# Should show sync-server as "Up (healthy)"

# View sync server logs
docker compose -f /opt/screenpipe-private/docker-compose.yml logs -f sync-server
```

## Server architecture

```
/opt/screenpipe-private/
├── .env                    # SYNC_TOKEN, DB_PATH, LOG_LEVEL (mode 600)
├── docker-compose.yml      # sync-server service definition
└── sync-server/
    ├── Dockerfile          # Python 3.12-slim, runs as UID 1000 (syncuser)
    ├── main.py             # FastAPI sync server
    ├── schema.sql          # Central DB schema (FTS5, triggers)
    └── requirements.txt    # fastapi, uvicorn, pydantic

/data/                      # LUKS-encrypted mount point
└── screenpipe-central/     # Owned by UID 1000
    └── db.sqlite           # Central merged database

/root/.luks-screenpipe.key  # LUKS keyfile (mode 400)
/etc/crypttab               # Auto-unlock on boot
/etc/fstab                  # Auto-mount /data
```

### Docker service details

| Service | Image | Port | Notes |
|---|---|---|---|
| sync-server | Built from `sync-server/Dockerfile` | `127.0.0.1:8765` | FastAPI, 2 uvicorn workers, runs as UID 1000 |
| ollama (opt-in) | `ollama/ollama:latest` | `127.0.0.1:11434` | Commented out by default — needs ≥8 GB RAM |

### API endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/health` | No | Server health check |
| POST | `/sync/push` | Yes | Push rows from a client machine |
| GET | `/sync/pull/{table}` | Yes | Pull rows (by machine, time range) |
| GET | `/sync/search` | Yes | Full-text search across all machines |
| GET | `/sync/status` | Yes | Sync state per machine |

Auth = `X-Sync-Token` header with the SYNC_TOKEN value.

## Enabling Ollama (optional)

Ollama is commented out in `docker-compose.yml` by default because it requires ≥8 GB RAM. To enable:

1. Edit `/opt/screenpipe-private/docker-compose.yml` — uncomment the `ollama` service block
2. Adjust `cpus` to match your VM's vCPU count
3. Restart: `docker compose up -d`
4. Pull a model: `docker compose exec ollama ollama pull llama3.2`

## Maintenance

### Reboot recovery

LUKS auto-unlocks on boot via `/etc/crypttab` + keyfile. Docker containers auto-restart. No manual intervention needed.

If LUKS doesn't auto-unlock:

```sh
sudo cryptsetup open --key-file /root/.luks-screenpipe.key /dev/sda screenpipe-data
sudo mount /dev/mapper/screenpipe-data /data
cd /opt/screenpipe-private && docker compose up -d
```

### Rotate SYNC_TOKEN

```sh
NEW_TOKEN=$(openssl rand -hex 32)
sudo sed -i "s/SYNC_TOKEN=.*/SYNC_TOKEN=$NEW_TOKEN/" /opt/screenpipe-private/.env
cd /opt/screenpipe-private && docker compose restart sync-server
echo "New SYNC_TOKEN: $NEW_TOKEN"
```

Update the token on every client machine's `.env` and reload the sync daemon.

### Database maintenance

```sh
# Check DB size
ls -lh /data/screenpipe-central/db.sqlite

# Optimize FTS indexes (run weekly)
docker compose exec sync-server sqlite3 /data/db.sqlite \
  "INSERT INTO ocr_fts(ocr_fts) VALUES('optimize'); INSERT INTO audio_fts(audio_fts) VALUES('optimize');"

# Check sync status
curl -H "X-Sync-Token: $(sudo grep SYNC_TOKEN /opt/screenpipe-private/.env | cut -d= -f2)" \
  http://localhost:8765/sync/status
```

### Backup

```sh
# Stop sync-server to ensure DB consistency
cd /opt/screenpipe-private && docker compose stop sync-server

# Copy the DB
cp /data/screenpipe-central/db.sqlite /data/screenpipe-central/db.sqlite.bak.$(date +%Y%m%d)

# Restart
docker compose start sync-server
```

## Troubleshooting

### sync-server won't start

```sh
docker compose logs sync-server
# Common issues:
# - "SYNC_TOKEN is not set" → check .env exists and has SYNC_TOKEN
# - "unable to open database file" → check /data is mounted and owned by UID 1000
```

### LUKS volume not detected

```sh
# List all block devices
lsblk

# Check which device is the data volume (not the root disk)
df /    # shows root disk
# The OTHER block device is your data volume
```

### Permission denied on /data

```sh
sudo chown -R 1000:1000 /data/screenpipe-central
docker compose restart sync-server
```

### cloudflared URL changed after reboot

```sh
cat /run/cloudflared-sync-url.txt
# If empty, check the service:
journalctl -u cloudflared-sync -n 50
sudo systemctl restart cloudflared-sync
# Wait 10s, then check again
cat /run/cloudflared-sync-url.txt
```
