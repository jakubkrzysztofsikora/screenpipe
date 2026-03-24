# screenpipe MCP — Remote Server Setup

The screenpipe sync server includes an embedded MCP server that exposes your
screen capture data to AI assistants via the Model Context Protocol (MCP).
It uses Streamable HTTP transport with OAuth 2.1 authentication.

## Architecture

The MCP server runs inside the sync server FastAPI app on the same port (8765),
exposed via cloudflared as `https://xxxx.trycloudflare.com`.

```
https://xxxx.trycloudflare.com/
├── /mcp                          MCP Streamable HTTP endpoint (POST/GET)
├── /.well-known/oauth-authorization-server
├── /.well-known/oauth-protected-resource/mcp
├── /authorize                    OAuth authorization endpoint
├── /token                        OAuth token endpoint
├── /register                     OAuth dynamic client registration
├── /login                        Sync token login form (shown to the user)
└── ... (existing sync API unchanged)
```

## Configuration

Add `MCP_BASE_URL` to your `.env` file (same file as `SYNC_TOKEN`):

```env
SYNC_TOKEN=your-secret-token-here
MCP_BASE_URL=https://xxxx.trycloudflare.com
```

`MCP_BASE_URL` must be the public HTTPS URL of your cloudflared tunnel.
Restart the container after changing it: `docker compose restart sync-server`.

## Connecting from claude.ai (Custom Connectors)

1. Open claude.ai → Settings → Integrations → Add Custom Connector
2. Paste your MCP URL: `https://xxxx.trycloudflare.com/mcp`
3. Click **Connect** — claude.ai redirects you to the login form
4. Enter your `SYNC_TOKEN` value and click **Authorise**
5. You are redirected back to claude.ai — the connector is now active

Available tools will appear automatically in new conversations.

## Connecting from Claude Code

```sh
claude mcp add screenpipe-private \
  --transport streamable-http \
  --url https://xxxx.trycloudflare.com/mcp
```

Claude Code will prompt you to authenticate via OAuth — open the URL in your
browser, enter your `SYNC_TOKEN`, and the token is stored locally.

## Available tools

### `search`

Full-text search across OCR and audio data on all synced machines.

```
search("standup meeting", content_type="audio", limit=10)
search("budget spreadsheet", machine_id="macbook-pro")
```

Parameters:
- `query` — search phrase (min 2 chars)
- `content_type` — `"all"` (default), `"ocr"`, or `"audio"`
- `machine_id` — restrict to one machine (optional)
- `limit` — max results, default 20, max 100

### `recent_activity`

Pull the most recent rows from any sync table.

```
recent_activity(table="ocr_text", limit=30)
recent_activity(table="meetings", since="2026-03-24T09:00:00")
recent_activity(table="audio_transcriptions", machine_id="macbook-pro")
```

Parameters:
- `table` — `ocr_text` (default), `audio_transcriptions`, `frames`, `memories`, `meetings`
- `machine_id` — restrict to one machine (optional)
- `limit` — max rows, default 20, max 1000
- `since` — ISO8601 datetime; defaults to 24 hours ago

### `status`

Show sync health across all machines.

```
status()
```

Returns a markdown summary with last sync time and row counts per machine and table.

### `timeline`

Merged OCR + audio timeline for the last N hours.

```
timeline(hours=2)
timeline(hours=4, machine_id="desktop-win")
```

Parameters:
- `hours` — how many hours back, default 1, max 24
- `machine_id` — restrict to one machine (optional)

Returns events sorted oldest-first with timestamps, machine names, and content snippets.

## Token security

The OAuth flow uses the `SYNC_TOKEN` env var as the credential. The token is
never stored in the browser or sent to claude.ai — the OAuth access token issued
after login is an opaque random string that expires after 24 hours.

OAuth state (tokens, auth codes, client registrations) is in-memory only.
A container restart clears all sessions; users will need to re-authenticate.
