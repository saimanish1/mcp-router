# Connecting Claude Code on a Remote Machine to MCP Router

This guide documents how to connect Claude Code running on a **remote machine** (e.g. a MacBook over Tailscale) to an MCP Router instance running on a **different machine** (e.g. a Windows PC).

## Architecture

```
Mac (Claude Code)
  └─ npx @mcp_router/cli connect --host <windows-ip> --port 3282
       └─ HTTP over Tailscale
            └─ Windows PC (MCP Router Electron app, port 3282)
                 └─ All configured MCP servers
```

## Prerequisites

- MCP Router running on the host machine (default port: `3282`)
- Both machines on the same network or VPN (e.g. Tailscale)
- Node.js available on the remote machine (nvm is fine)
- An auth token from MCP Router (Settings → Auth Token)

## Configuration

Add the following to `~/.claude.json` on the remote machine, under the top-level `mcpServers` key:

```json
"mcpServers": {
  "mcp-router": {
    "command": "/path/to/node/bin/npx",
    "args": [
      "-y",
      "@mcp_router/cli@latest",
      "connect",
      "--host", "<MCP_ROUTER_HOST_IP>",
      "--port", "3282"
    ],
    "env": {
      "MCPR_TOKEN": "<your-auth-token>",
      "PATH": "/path/to/node/bin:/usr/local/bin:/usr/bin:/bin"
    }
  }
}
```

### Important: use `--host` and `--port`, not `--url`

The published `@mcp_router/cli` package supports `--host` and `--port` flags. Do **not** use `--url` — it is not supported in the published build and will be silently ignored, causing the CLI to fall back to `localhost:3282`.

### PATH must include Node

Claude Code spawns the MCP server process with a minimal environment. If `npx` cannot be found, explicitly set `PATH` in the `env` block to include the directory containing your Node/npx binaries.

**nvm users**: use the full path to your nvm node version, e.g.:
```
/Users/you/.nvm/versions/node/v22.3.0/bin
```

## Verify the connection

```bash
claude mcp list
```

Expected output:
```
mcp-router: /path/to/npx -y @mcp_router/cli@latest connect --host <ip> --port 3282 - ✓ Connected
```

## How it works

The CLI (`@mcp_router/cli`) acts as a bridge:

1. Claude Code spawns the CLI as a local stdio MCP server
2. On receiving an `initialize` message from Claude Code, the CLI connects upstream to MCP Router over HTTP
3. All tool calls, resource reads, and prompts are proxied between Claude Code and MCP Router
4. MCP Router aggregates all its configured MCP servers and exposes them as a single unified server

### GET /mcp → 405 (required server behaviour)

The MCP SDK (`StreamableHTTPClientTransport`) sends a `GET /mcp` request on connect to attempt an SSE stream for server-to-client messages. MCP Router responds with **HTTP 405 Method Not Allowed**, which signals to the SDK to use POST-only mode. Without this 405 response the SDK would treat the missing handler as a hard error and fail to connect.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Failed to connect` in `claude mcp list` | CLI hitting `localhost` instead of remote host | Ensure you are using `--host` and `--port`, not `--url` |
| `fetch failed` error | CLI falling back to `localhost:3282` (see above) | Same as above |
| `command not found: npx` | PATH missing Node bin dir | Add full Node bin path to `env.PATH` in config |
| HTTP 406 on POST | Missing `Accept` header | Upgrade to latest `@mcp_router/cli` |
| Connected but no tools | MCP Router has no servers configured | Add MCP servers in the MCP Router Electron app |
