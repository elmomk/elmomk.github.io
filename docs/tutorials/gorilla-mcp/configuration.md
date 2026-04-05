# Configuration

All configuration is via environment variables. No config files are read at runtime (except `.mcp.json` by Claude Code).

## MCP Server (`gorilla_mcp`)

### Required

| Variable | Description | Example |
|----------|-------------|---------|
| `COACH_BASE_URL` | Base URL of gorilla_coach API | `https://gorilla-coach.example.ts.net` |
| `COACH_API_KEY` | API key for gorilla_coach | `gc_abc123...` |
| `GARMIN_BASE_URL` | Base URL of garmin_api service | `https://garmin-api.example.ts.net` |
| `GARMIN_API_KEY` | API key for garmin_api | `ga_xyz789...` |
| `GARMIN_USER_ID` | UUID of the Garmin user | `65ef1758-63e0-4a0c-89bd-17f785de0391` |

### Optional

| Variable | Default | Description |
|----------|---------|-------------|
| `TRANSPORT` | `stdio` | Transport mode: `stdio` or `http` |
| `PORT` | `3000` | HTTP server port (only used when `TRANSPORT=http`) |
| `RUST_LOG` | `info` | Log level: `error`, `warn`, `info`, `debug`, `trace` |

## Chatbot (`gorilla_chatbot`)

### Required

Same upstream variables as the MCP server (`COACH_BASE_URL`, `COACH_API_KEY`, `GARMIN_BASE_URL`, `GARMIN_API_KEY`, `GARMIN_USER_ID`) — these are passed to the MCP server it spawns.

### Optional

| Variable | Default | Description |
|----------|---------|-------------|
| `CLAUDE_MODEL` | `sonnet` | Claude model: `opus`, `sonnet`, `haiku` |
| `PORT` | `8080` | HTTP server port |
| `MCP_BINARY` | Auto-detected | Path to `gorilla_mcp` binary |
| `HISTORY_PATH` | `/tmp/gorilla-chat-history.json` | Conversation history file |
| `GATEWAY_API_KEY` | *(unset)* | Shared secret for gateway mode. If set and non-empty, enables `POST /api/gateway`. |
| `RUST_LOG` | `info` | Log level for chatbot |
| `MCP_LOG` | `warn` | Log level for MCP subprocess (passed as `RUST_LOG` to child) |

## Docker / Tailscale

| Variable | Description |
|----------|-------------|
| `TS_AUTHKEY` | Tailscale auth key for node registration (`tskey-auth-...`) |

## Claude Code MCP Config (`.mcp.json`)

This file tells Claude Code how to launch the MCP server locally:

```json
{
  "mcpServers": {
    "gorilla": {
      "command": "/path/to/target/release/gorilla_mcp",
      "env": {
        "COACH_BASE_URL": "https://gorilla-coach.example.ts.net",
        "COACH_API_KEY": "gc_abc123...",
        "GARMIN_BASE_URL": "https://garmin-api.example.ts.net",
        "GARMIN_API_KEY": "ga_xyz789...",
        "GARMIN_USER_ID": "65ef1758-63e0-4a0c-89bd-17f785de0391",
        "RUST_LOG": "warn"
      }
    }
  }
}
```

Place this in the project root. Claude Code reads it automatically and spawns the MCP server in stdio mode.

## `.env.example`

```bash
# gorilla_coach API
COACH_BASE_URL=https://gorilla-coach.your-tailnet.ts.net
COACH_API_KEY=your_api_key_here

# garmin_api service
GARMIN_BASE_URL=https://garmin-api.your-tailnet.ts.net
GARMIN_API_KEY=your_api_key_here
GARMIN_USER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Tailscale (for Docker Compose)
TS_AUTHKEY=tskey-auth-...

# Chatbot
CLAUDE_MODEL=opus

# Gateway API (allows gorilla_coach to use chatbot as LLM backend)
# GATEWAY_API_KEY=your_shared_secret_here

# Optional
# PORT=8080
# RUST_LOG=info
```

Copy to `.env` and fill in your values. The `.env` file is gitignored.

## MCP Binary Auto-Detection

The chatbot locates the MCP binary in this order:

1. `MCP_BINARY` environment variable (if set)
2. Sibling of the chatbot binary (e.g., `/app/gorilla_mcp` when chatbot is at `/app/gorilla_chatbot`)
3. Fails with an error if not found

In Docker, both binaries are copied to `/app/`, so sibling detection works automatically.
