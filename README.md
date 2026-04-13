# Nova Platform

> The plugin architecture powering Nova — a modular system where every capability is an isolated addon.

Nova is not a monolith. Every feature runs as an independent addon with its own backend, frontend, database and lifecycle. The platform manages them all: starts, stops, proxies, secures and monitors. Addons can be added or removed without touching the core.

---

## The Core Idea

```
Nova Core (Express.js)
      ↓
Addon Manager         ← starts/stops addon backends
      ↓
Addon Proxy           ← routes HTTP requests to the right addon
      ↓
Addon API             ← what addons can call (push, brain, files)
      ↓
Files Module          ← secure file system access
```

Each addon is a self-contained unit:
```
addons/
  autonomi/
    addon.json          ← manifest (name, label, backend_start)
    backend/
      index.js          ← addon backend (any language/runtime)
    frontend/
      component.js      ← UI component loaded by the panel
```

---

## Addon Manager

Handles the full lifecycle of addon backends:

```
Start addon:
  1. Read addon.json
  2. Check if package.json changed (hash) → npm install if needed
  3. Spawn backend process with RAM limit (256MB)
  4. Generate unique auth token → write to registry
  5. Monitor process → restart on crash (with backoff)

Crash backoff:
  > 3 crashes in 60 seconds → mark as failed, stop restarting
  → prevents a broken addon from hammering the system
```

Each addon backend gets:
- Its own Unix socket `/tmp/nova-addon-{name}.sock`
- A unique auth token (rotated on each start)
- An isolated process with memory limits

---

## Addon Proxy

Routes incoming HTTP requests from the panel frontend to the correct addon backend:

```
Browser → GET /addon/autonomi/flows
              ↓
        Addon Proxy
              ↓
        Unix Socket → autonomi backend
              ↓
        Response proxied back
```

Short-lived **context tokens** (30s JWT) are injected into proxied requests so addon backends can verify the calling user without storing credentials.

---

## Addon API

What addons can do by calling back into the Nova core:

```javascript
// Send a push notification to Nico
POST /addon-api/push
{ title, body, type }

// Read/write to GenesisBrain (via Doorwatcher)
POST /addon-api/brain/search
POST /addon-api/brain/create
GET  /addon-api/brain/:id

// Access the file system (sandboxed)
GET  /addon-api/files/list
POST /addon-api/files/read
POST /addon-api/files/write
```

Every addon API call is authenticated via the addon's unique token. An addon can only call what it's allowed to.

---

## Files Module

Secure file system access with section-based sandboxing:

```javascript
const SECTIONS = {
  vserver:  { root: "/opt/nova-files",      writable: true },
  webapp:   { root: "/opt/claude-panel",    writable: true },
  memory:   { root: "/memory/",             writable: true },
  projects: { root: "/data/project-files",  writable: true },
}
```

Every file operation goes through `resolveSafe()` — path traversal attempts (`../../etc/passwd`) are rejected before hitting the filesystem.

**Download tokens** — temporary signed tokens (2 min TTL) for file downloads. No direct file paths exposed to the browser.

```javascript
// Request download → get token
POST /files/download-token { section, path }
→ { token: "abc123", expires: ... }

// Use token once to download
GET /files/download?token=abc123
→ file stream
```

---

## Registry

A single JSON file tracks the state of all addons at runtime:

```json
{
  "autonomi": {
    "enabled": true,
    "pid": 12345,
    "token": "a7f3...",
    "started_at": "2026-04-13T10:00:00Z",
    "crash_times": [],
    "pkg_hash": "d41d8c..."
  },
  "spotify": {
    "enabled": false,
    "pid": null,
    "token": null
  }
}
```

---

## Current Addons

| Addon | Purpose |
|---|---|
| autonomi | Goals, flows, mood, inbox, daemon |
| bridge-connector | LLM integration, tool use, STM |
| com-hub | Communication hub |
| log-watcher | Real-time log monitoring |
| notiz | Note-taking for Nova |
| nova-leads | Lead capture system |
| nova-status | System health monitoring |
| security-monitor | Security event tracking |
| space | Personal space/journal |
| spotify | Spotify integration |
| agi-tracker | AGI development tracker |

---

## Tech Stack

- Node.js + Express (core platform)
- Unix Sockets (addon IPC)
- JWT (context tokens, 30s TTL)
- SQLite (panel database)
- Better-SQLite3 (sync, WAL mode)
- Multer (file uploads)

---

## Part of Nova

Nova Platform is the foundation of **Nova** — an autonomous AI assistant system.

🌐 [neural-werk.de](https://neural-werk.de) · [GenesisBrain](https://github.com/nico-gordon-gilles/nova-genesisbrain) · [Autonomi](https://github.com/nico-gordon-gilles/nova-autonomi)

