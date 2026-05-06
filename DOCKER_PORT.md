# Docker Port Configuration

This document tracks the changes made to allow Open Design's web UI and daemon to be accessible from outside a Docker container.

## Problem

By default, both the daemon and the web server bind to `127.0.0.1` (localhost). Inside a Docker container, this means they are only reachable from within the container itself. Port mappings like `3000:3000` or `7456:7456` do not work because the host's connection arrives with a non-loopback source IP, which the Node.js servers reject.

## Changes Made

### 1. `apps/web/sidecar/server.ts`

**Before:**
```ts
const HOST = process.env.OD_HOST || "127.0.0.1";
```

**After:**
```ts
const HOST = process.env.OD_HOST || process.env.OD_BIND_HOST || "127.0.0.1";
```

The web sidecar now falls back to `OD_BIND_HOST` if `OD_HOST` is not set. This allows a single environment variable (`OD_BIND_HOST=0.0.0.0`) to configure both the daemon and the web server.

### 2. `apps/web/next.config.ts`

**Before:**
```ts
const DAEMON_ORIGIN = `http://127.0.0.1:${DAEMON_PORT}`;
// ...
allowedDevOrigins: ['127.0.0.1'],
```

**After:**
```ts
const DAEMON_HOST = process.env.OD_BIND_HOST || process.env.OD_HOST || '127.0.0.1';
const DAEMON_ORIGIN = `http://${DAEMON_HOST}:${DAEMON_PORT}`;
// ...
allowedDevOrigins: [DAEMON_HOST],
```

Dev-mode rewrites and origin validation now respect the same `OD_BIND_HOST` / `OD_HOST` variables instead of hardcoding `127.0.0.1`.

## Usage in Docker

Set `OD_BIND_HOST=0.0.0.0` in your container environment:

```yaml
environment:
  - OD_BIND_HOST=0.0.0.0
  - OD_DATA_DIR=/app/.od
```

Both services will then listen on all interfaces and become reachable through Docker port mappings:

- Daemon API: `http://host:7456`
- Web UI: `http://host:3000`

## Environment Variables Reference

| Variable | Used By | Default | Purpose |
|---|---|---|---|
| `OD_BIND_HOST` | Daemon + Web | `127.0.0.1` | Shared bind host for both services |
| `OD_HOST` | Web only | `127.0.0.1` | Legacy web-specific host override |
| `OD_DATA_DIR` | Daemon | — | Directory for SQLite and project files |
