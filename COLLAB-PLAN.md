# AppBoards Collaborative Service — Implementation Plan

*Written May 20, 2026. Ready for execution.*

---

## Overview

Three independently deployable pieces:

1. **Hosted API** — Go HTTP + WebSocket server on dfn01
2. **Sync agent** — Go binary that runs locally, watches files, exposes local API to AI tools
3. **Viewer app** — Plain HTML/JS served as a static file by the hosted API

CLI wraps the sync agent binary. No separate build.

---

## Repository Structure

```
appboards-service/
├── cmd/
│   ├── server/         # Hosted API binary
│   │   └── main.go
│   └── agent/          # Sync agent + CLI binary
│       └── main.go
├── internal/
│   ├── api/            # Hosted API handlers
│   ├── db/             # Postgres queries (sqlc or hand-rolled)
│   ├── hub/            # WebSocket hub
│   ├── agent/          # File watcher, local HTTP server
│   └── keygen/         # Project key generation
├── viewer/
│   └── index.html      # Viewer app (embedded into server binary)
├── migrations/
│   ├── 001_init.sql
│   └── 002_...sql
├── go.mod
└── Makefile
```

---

## 1. Postgres Schema

```sql
-- migrations/001_init.sql

CREATE TABLE projects (
  key         TEXT PRIMARY KEY,            -- "fox-river-41"
  created_at  TIMESTAMPTZ DEFAULT now(),
  last_active TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE variants (
  id          SERIAL PRIMARY KEY,
  project_key TEXT NOT NULL REFERENCES projects(key) ON DELETE CASCADE,
  variant_id  TEXT NOT NULL,              -- "p1vA", "p1vB", etc.
  page_id     TEXT NOT NULL DEFAULT 'p1', -- "p1", "p2", etc.
  status      TEXT NOT NULL DEFAULT 'claimed',  -- 'claimed' | 'ready'
  label       TEXT,                       -- optional human label
  html        TEXT,                       -- NULL until first sync
  claimed_at  TIMESTAMPTZ DEFAULT now(),
  updated_at  TIMESTAMPTZ DEFAULT now(),
  UNIQUE(project_key, variant_id)
);

CREATE INDEX idx_variants_project ON variants(project_key);

-- For fast next-letter lookup
CREATE TABLE variant_counters (
  project_key TEXT NOT NULL REFERENCES projects(key) ON DELETE CASCADE,
  page_id     TEXT NOT NULL,
  next_index  INTEGER NOT NULL DEFAULT 0,  -- 0=A, 1=B, ..., 25=Z, 26=AA, etc.
  PRIMARY KEY (project_key, page_id)
);
```

**Variant ID generation** — convert `next_index` to spreadsheet column letters:

```
0→A, 1→B, ..., 25→Z, 26→AA, 27→AB, ..., 51→AZ, 52→BA, ...
```

Function lives in `internal/keygen/variant.go`. Increment `variant_counters` atomically in a transaction with the INSERT.

---

## 2. Hosted API

**Binary:** `cmd/server/main.go`  
**Port:** 8741 (internal, nginx proxies)  
**Language:** Go with `net/http` + `gorilla/websocket`

### Endpoints

```
POST   /projects                       Create new project, return key
GET    /projects/:key                  Get project + all variants (for late joiners)
POST   /projects/:key/variants/claim   Claim next variant slot (deli ticket)
PUT    /projects/:key/variants/:id     Push HTML content (sync agent calls this)
GET    /projects/:key/ws               WebSocket upgrade (viewers connect here)
GET    /                               Serve viewer/index.html (catch-all)
GET    /:key                           Serve viewer/index.html (key in URL)
```

### Request / Response Shapes

**POST /projects**
```json
// Request: (no body)
// Response:
{
  "key": "fox-river-41"
}
```

**GET /projects/:key**
```json
{
  "key": "fox-river-41",
  "variants": [
    { "variant_id": "p1vA", "page_id": "p1", "status": "claimed", "label": null, "html": null },
    { "variant_id": "p1vB", "page_id": "p1", "status": "ready",   "label": "Chip nav", "html": "<html>..." }
  ]
}
```

**POST /projects/:key/variants/claim**
```json
// Request:
{ "page_id": "p1", "label": "My experiment" }

// Response:
{ "variant_id": "p1vC", "page_id": "p1", "status": "claimed" }
```

**PUT /projects/:key/variants/:id**
```json
// Request:
{ "html": "<html>..." }

// Response:
{ "variant_id": "p1vC", "status": "ready", "updated_at": "2026-05-20T12:31:00Z" }
```

### WebSocket Protocol

Connect: `wss://appboards.co/projects/:key/ws`

All messages are JSON. Server → client only (broadcast). Client never sends.

**Message types:**

```jsonc
// variant_claimed: new slot assigned, show placeholder
{
  "type": "variant_claimed",
  "variant_id": "p1vC",
  "page_id": "p1",
  "label": null
}

// variant_updated: HTML ready, render it
{
  "type": "variant_updated",
  "variant_id": "p1vC",
  "page_id": "p1",
  "status": "ready",
  "html": "<html>..."
}

// variant_label_updated: label changed (future)
{
  "type": "variant_label_updated",
  "variant_id": "p1vC",
  "label": "New label"
}

// heartbeat: keep-alive, clients ignore
{
  "type": "heartbeat"
}
```

### WebSocket Hub

`internal/hub/hub.go`

```go
type Hub struct {
    // map[projectKey]map[*websocket.Conn]bool
    rooms   map[string]map[*Client]bool
    mu      sync.RWMutex
    broadcast chan Message
}
```

- One goroutine per connected client reads pings
- Hub goroutine fans out broadcast messages to all clients in a room
- Client disconnect cleans up the map
- Heartbeat tick every 30s via `time.Ticker`

### Project Key Generation

`internal/keygen/project.go`

Word lists: ~200 adjectives + ~200 nouns + 2-digit number.  
Format: `{adj}-{noun}-{nn}` (e.g. `fox-river-41`).  
Generate, check DB for collision, retry up to 5 times (collision probability is negligible).

---

## 3. Sync Agent (Local)

**Binary:** `cmd/agent/main.go`  
Single binary. Doubles as the CLI (`appboard new`, `appboard join`).

### What it does

1. Reads/writes `.appboard` config file in project dir
2. Watches project dir for `*.html` files matching `p{n}v{letters}.html`
3. Exposes local HTTP API on a random port (saved to `.appboard`)
4. Pushes new/changed HTML to hosted API

### `.appboard` config file

```json
{
  "key": "fox-river-41",
  "agent_port": 52341,
  "api_url": "https://appboards.co",
  "viewer_url": "https://appboards.co/fox-river-41"
}
```

Committed to the project folder. OpenCode/Claude Code picks it up automatically via `LLMCONTEXT.md` (we'll add a section).

### File Watcher

Uses `fsnotify`. On `CREATE` or `WRITE` event for a file matching `^p\d+v[A-Z]+\.html$`:

1. Read file contents
2. Extract variant_id from filename (e.g. `p1vC.html` → `p1vC`)
3. `PUT /projects/{key}/variants/{id}` with HTML body
4. Log result

Debounce: 500ms (AI writes large files incrementally; wait for the write to settle).

### Local HTTP API

Listens on `localhost:{agent_port}`.

```
POST /variants/claim        Claim next variant slot
GET  /status                Agent health + project info
GET  /variants              List all variants (proxied from hosted API)
```

**POST /variants/claim**
```json
// Request:
{ "page_id": "p1", "label": "optional label" }

// Response:
{
  "variant_id": "p1vC",
  "page_id": "p1",
  "filename": "p1vC.html",
  "filepath": "/Users/mark/projects/my-app/p1vC.html",
  "viewer_url": "https://appboards.co/fox-river-41"
}
```

The agent forwards the claim to hosted API, gets back the variant_id, constructs the local filepath, returns both to the AI. AI writes `p1vC.html`. Watcher picks it up and pushes.

**GET /status**
```json
{
  "key": "fox-river-41",
  "project_dir": "/Users/mark/projects/my-app",
  "viewer_url": "https://appboards.co/fox-river-41",
  "connected": true,
  "variants_synced": 3
}
```

### CLI Commands

```bash
appboard new              # creates .appboard, registers project, starts agent, prints viewer URL
appboard join fox-river-41  # writes .appboard with existing key, starts agent
appboard status           # shows current project info
```

**`appboard new` flow:**
```
$ appboard new
Creating new AppBoards session...

  Project key:  fox-river-41
  Viewer URL:   https://appboards.co/fox-river-41
  Agent port:   52341

Watching /Users/mark/projects/my-app for variants...
Share the viewer URL with collaborators. They need nothing installed.
```

**`appboard join` flow:**
```
$ appboard join fox-river-41
Joining session fox-river-41...

  Viewer URL:  https://appboards.co/fox-river-41
  Agent port:  52342

Watching /Users/mark/projects/my-app for variants...
```

After `new` or `join`, process stays running (foreground). Ctrl+C stops it.  
For background: `appboard new --detach` (writes PID to `.appboard`).

### LLMCONTEXT.md Addition

Add a section to `LLMCONTEXT.md` that tells AI how to use the agent:

```markdown
## Collaborative mode

If `.appboard` exists in this project folder, a sync agent is running.

To claim a new variant:
  POST http://localhost:{agent_port}/variants/claim
  Body: {"page_id": "p1", "label": "optional description"}
  Returns: variant_id, filename, filepath

Write your variant HTML to the returned filepath.
The agent will detect it and sync it to all viewers automatically.
The agent_port is in .appboard.
```

---

## 4. Viewer Web App

**File:** `viewer/index.html`  
Embedded into the server binary via `//go:embed viewer/index.html`.  
Served at `/` and `/:key`.

### Design Requirements

Match AppBoard's aesthetic: Outfit font, same design tokens, clean minimal. Not a new visual language.

### Layout

```
┌─────────────────────────────────────────┐
│ appboards     fox-river-41    [+ New]    │  ← header bar
├─────────────────────────────────────────┤
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐  │  ← variant grid
│  │  p1vA    │  │  p1vB    │  │  ...  │  │
│  │ [phone]  │  │ [phone]  │  │       │  │
│  └──────────┘  └──────────┘  └───────┘  │
│                                         │
└─────────────────────────────────────────┘
```

- Each variant renders in an `<iframe>` (blob URL from the HTML)
- Claimed-but-not-ready variants show a placeholder (pulsing skeleton)
- Ready variants show a scaled-down phone frame (CSS transform scale)
- Clicking a variant opens it full-screen in a modal overlay

### Key Entry Screen

If no key in URL:
```
┌──────────────────────────────┐
│         appboards            │
│                              │
│  Enter a session key         │
│  ┌──────────────────────┐    │
│  │ fox-river-41         │    │
│  └──────────────────────┘    │
│         [Open →]             │
└──────────────────────────────┘
```

### JavaScript Architecture

```javascript
// State
const state = {
  key: null,           // from URL
  variants: [],        // [{variant_id, page_id, status, html}]
  ws: null,
};

// On load
const key = location.pathname.slice(1);  // e.g. "fox-river-41"
if (key) {
  fetchInitialState(key);   // GET /projects/:key
  connectWS(key);           // ws://host/projects/:key/ws
}

// WebSocket handler
ws.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  if (msg.type === 'variant_claimed') addPlaceholder(msg);
  if (msg.type === 'variant_updated') renderVariant(msg);
};

// Render variant in iframe
function renderVariant({ variant_id, html }) {
  const blob = new Blob([html], { type: 'text/html' });
  const url = URL.createObjectURL(blob);
  document.querySelector(`#frame-${variant_id}`).src = url;
}
```

### Placeholder

While `status === 'claimed'`:
```html
<div class="variant-card placeholder">
  <div class="phone-skeleton">
    <div class="skeleton-bar"></div>
    <div class="skeleton-block"></div>
    <div class="skeleton-block short"></div>
  </div>
  <div class="variant-label">Working...</div>
</div>
```

Pulsing animation via `@keyframes pulse { 0%,100%{opacity:.4} 50%{opacity:.8} }`.

### Full-Screen Modal

Click any variant → modal opens with full-size iframe + sidebar showing variant metadata.  
Esc or click overlay to close. This is where design review happens.

---

## 5. Deployment on dfn01

**dfn01 is shared production.** Minimal footprint. Additive only.

### What Gets Added

- New Postgres database: `appboards`
- New Go binary: `/usr/local/bin/appboards-server`
- New systemd service: `appboards-server.service`
- New nginx server block: `appboards.co`
- TLS cert via certbot (existing certbot installation)

### Postgres Setup

```bash
sudo -u postgres createuser appboards
sudo -u postgres createdb appboards -O appboards
sudo -u postgres psql -c "ALTER USER appboards WITH PASSWORD 'strong-password';"

# Run migrations
psql -U appboards -d appboards -f migrations/001_init.sql
```

No changes to existing databases.

### Systemd Service

`/etc/systemd/system/appboards-server.service`:

```ini
[Unit]
Description=AppBoards Collaboration Server
After=network.target postgresql.service

[Service]
Type=simple
User=www-data
Group=www-data
ExecStart=/usr/local/bin/appboards-server
Restart=on-failure
RestartSec=5s
Environment=APP_PORT=8741
Environment=DATABASE_URL=postgres://appboards:strong-password@localhost/appboards?sslmode=disable
Environment=APP_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable appboards-server
systemctl start appboards-server
```

### Nginx Config

New file: `/etc/nginx/sites-available/appboards.co`

```nginx
server {
    listen 80;
    server_name appboards.co www.appboards.co;
    return 301 https://appboards.co$request_uri;
}

server {
    listen 443 ssl http2;
    server_name appboards.co www.appboards.co;

    ssl_certificate     /etc/letsencrypt/live/appboards.co/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/appboards.co/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;

    # WebSocket support
    location /projects/ {
        proxy_pass         http://127.0.0.1:8741;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;  # keep WS connections alive
    }

    # All other requests
    location / {
        proxy_pass         http://127.0.0.1:8741;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/appboards.co /etc/nginx/sites-enabled/
certbot --nginx -d appboards.co -d www.appboards.co
nginx -t && systemctl reload nginx
```

### Deployment Script

`Makefile` target:

```makefile
deploy-server:
    GOOS=linux GOARCH=amd64 go build -o bin/appboards-server-linux ./cmd/server
    scp bin/appboards-server-linux dfn01:/usr/local/bin/appboards-server
    ssh dfn01 "systemctl restart appboards-server"
    ssh dfn01 "systemctl status appboards-server --no-pager"
```

---

## 6. Build Order

Get end-to-end working as fast as possible. Each step is a shippable checkpoint.

### Step 1 — Hosted API skeleton (Day 1)

Goal: `POST /projects` and `GET /projects/:key` working against real Postgres.

- `go mod init`, add `gorilla/websocket`, `lib/pq`
- Migrations + schema
- Two handlers: create project, get project
- Deploy to dfn01, confirm it's live
- Test with `curl`

**Checkpoint:** `curl -X POST https://appboards.co/projects` returns a key.

### Step 2 — Viewer app shell (Day 1-2)

Goal: `https://appboards.co/fox-river-41` opens and shows something.

- Static `viewer/index.html` embedded in server binary
- Served at `/` and `/:key`
- Key entry screen
- Fetch initial state from API, display variant list (even if empty)
- No WebSocket yet — polling is fine at this stage

**Checkpoint:** Open URL in browser, enter a key, see "No variants yet."

### Step 3 — Variant claiming + WebSocket (Day 2-3)

Goal: Two browser windows both see new variants appear in real time.

- `POST /projects/:key/variants/claim` endpoint
- WebSocket hub implementation
- Broadcast `variant_claimed` on claim
- Viewer connects to WS, adds placeholder on message
- Test manually: claim via curl, see placeholder appear in browser

**Checkpoint:** `curl -X POST .../variants/claim` → placeholder appears in two open browsers simultaneously.

### Step 4 — Sync agent core (Day 3-4)

Goal: AI writes a file, viewers update.

- Agent binary: reads/writes `.appboard`
- File watcher (`fsnotify`)
- `PUT /projects/:key/variants/:id` hosted endpoint
- Broadcast `variant_updated` with HTML on PUT
- Viewer renders variant in iframe on `variant_updated` message

**Checkpoint:** Write `p1vA.html` to disk → all viewers render it within 1 second.

### Step 5 — Local API for AI (Day 4)

Goal: AI can `POST localhost:PORT/variants/claim` and get back a filename.

- Local HTTP server in agent
- `/variants/claim` endpoint (proxies to hosted)
- Returns filepath as part of response
- Add LLMCONTEXT.md section

**Checkpoint:** `curl http://localhost:52341/variants/claim` returns `{"variant_id":"p1vB","filepath":"/path/p1vB.html"}`. Write the file. Viewers update.

### Step 6 — CLI polish (Day 5)

Goal: `appboard new` is the entry point.

- `appboard new` and `appboard join` commands
- Nice terminal output
- `--detach` flag
- `appboard status`

**Checkpoint:** `appboard new` in a project folder → message printed → share URL → collaborator opens it, sees live updates.

### Step 7 — Viewer polish (Day 5-6)

Goal: The viewer looks like an AppBoards product.

- Design tokens match `index.html`
- Placeholder animation
- Full-screen modal on click
- Page filter (p1, p2, etc.)
- Reconnect logic on WebSocket drop

**Checkpoint:** Demo the full flow. It looks real. Hand it to someone on a call.

---

## Key Decisions & Rationale

| Decision | Choice | Why |
|---|---|---|
| Real-time transport | WebSockets | Simple, well-understood, no library needed beyond gorilla/websocket. CRDTs overkill for MVP — no conflict resolution needed when server assigns IDs. |
| Server language | Go | Single static binary for both server and sync agent. No runtime deps. Easy to deploy. |
| DB | Postgres | Already on dfn01 (or easily added). ACID guarantees for variant counter atomicity. |
| Viewer | Plain HTML/JS | Matches AppBoard's zero-dependency philosophy. Embedded in Go binary — no static file server needed. |
| Sync direction | Agent → server only | Simple. No merging needed. Server is source of truth. |
| Late joiner sync | HTTP GET on connect | Simple full-state fetch before WS connect. No replay log needed. |
| Variant ID assignment | Server-side, atomic | Deli ticket system. Zero collision risk. Spreadsheet column order matches existing AppBoard naming. |
| Key format | adj-noun-nn | Short, readable, speakable on a phone call ("fox river 41"). |

---

## Edge Cases to Handle

- **Agent restart mid-session**: reads `.appboard`, re-registers watcher, re-syncs all local `.html` files via PUT (server deduplicates on content hash or updated_at comparison)
- **Multiple agents on same project**: fine — server serializes variant counter with a transaction
- **WebSocket reconnect**: viewer retries with exponential backoff (500ms, 1s, 2s, 4s max), fetches full state again on reconnect
- **Large HTML files**: no hard limit in MVP, but nginx `client_max_body_size 10m` to be safe
- **Stale projects**: cron job (or lazy TTL) to clean up projects inactive >7 days — not needed day one
- **Same file written rapidly**: debounce in fsnotify handler (500ms)

---

## What's Deliberately Not in MVP

- User accounts / auth
- Project listing / history
- Variant deletion
- Multiple pages (API supports it, viewer filters by page, but CLI defaults to p1)
- Marketing site
- Billing
- Real-time cursor presence

---

*Plan complete. Start with Step 1.*
