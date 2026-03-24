# screenpipe private — MCP server

Wraps the screenpipe private sync server REST API so Claude Desktop and Claude Code can search, browse, and query your screen capture data across all synced machines.

## Tools

| Tool | Description |
|------|-------------|
| `search` | Full-text search across OCR and audio data |
| `recent_activity` | Pull recent rows from any sync table |
| `status` | Sync health — machines, last sync times, row counts |
| `timeline` | What happened in the last N hours (OCR + audio combined) |

## Requirements

- Python 3.10+
- The `mcp` package (`pip install mcp`)

```sh
pip install -r requirements.txt
```

## Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `SYNC_SERVER_URL` | Yes | Base URL of the sync server — either `http://localhost:8765` (SSH tunnel) or `https://xxxx.trycloudflare.com` (cloudflared) |
| `SYNC_TOKEN` | Yes | Shared secret token — must match the server's `SYNC_TOKEN` |

## Adding to Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "screenpipe-private": {
      "command": "python3",
      "args": ["/path/to/private-overlay/mcp/sync-server-mcp.py"],
      "env": {
        "SYNC_SERVER_URL": "https://xxxx.trycloudflare.com",
        "SYNC_TOKEN": "your-token-here"
      }
    }
  }
}
```

Replace the path, URL, and token with your actual values. Restart Claude Desktop after saving.

## Adding to Claude Code

```sh
claude mcp add screenpipe-private \
  --command python3 \
  --args "/path/to/private-overlay/mcp/sync-server-mcp.py" \
  --env SYNC_SERVER_URL=https://xxxx.trycloudflare.com \
  --env SYNC_TOKEN=your-token-here
```

Or use the interactive prompt:

```sh
claude mcp add
```

## SSH tunnel mode

If using SSH tunnel, start the tunnel first, then point `SYNC_SERVER_URL` at `localhost`:

```sh
./local/tunnel-manager.sh &
export SYNC_SERVER_URL=http://localhost:8765
export SYNC_TOKEN=your-token-here
python3 mcp/sync-server-mcp.py
```
