# Architecture

## Overview

Gorilla MCP is a Rust workspace with two crates that bridge Claude AI to strength training and biometric data:

```
Claude Code / Claude AI
        |
        | MCP Protocol (stdio or HTTP/SSE)
        |
  gorilla_mcp (MCP Server)
        |
        +--- CoachClient ----> gorilla_coach (/api/v2/*)
        |                      Training plans, logs, reports,
        |                      5/3/1 auto-regulator, files
        |
        +--- GarminClient ---> garmin_api (/api/v1/*)
                               Daily biometrics, vitals,
                               baselines, connection status
```

The chatbot adds a web layer:

```
Browser -----> gorilla_chatbot (Axum)
                    |
                    +-- Spawns Claude CLI --mcp-config --> gorilla_mcp
                    |
                    +-- POST /api/gateway (for gorilla_coach to call Claude)
```

## MCP Server (`gorilla_mcp`)

### Core Struct

`GorillaMcp` in `src/server.rs` holds all state:

```rust
pub struct GorillaMcp {
    coach: CoachClient,     // gorilla_coach HTTP client
    garmin: GarminClient,   // garmin_api HTTP client
    cache: ResponseCache,   // moka cache (5-min TTL, 100 entries)
    peer: Option<Peer<RoleServer>>, // MCP peer connection
}
```

The `#[rmcp::tool(tool_box)]` macro on the impl block generates MCP tool definitions from method signatures and `#[tool(description = "...")]` attributes.

### Transport Modes

**Stdio (default):** The MCP server reads JSON-RPC from stdin and writes to stdout. This is the standard mode for Claude Code local development — Claude Code spawns the binary as a child process.

**HTTP/SSE (`--http` flag or `TRANSPORT=http`):** The server listens on `PORT` (default 3000) and accepts SSE connections at `/sse`. Each connection gets its own MCP session. Used in Docker deployments where the server runs as a persistent service behind Tailscale.

```rust
// main.rs — transport selection
match config.transport.as_str() {
    "http" => {
        // rmcp SSE server on 0.0.0.0:PORT
    }
    _ => {
        // rmcp stdio server on stdin/stdout
    }
}
```

### Data Flow

Every tool follows the same pattern:

1. Claude calls a tool (e.g., `get_sitrep`)
2. `GorillaMcp` method is invoked by rmcp
3. Method calls the appropriate client (`coach.get_sitrep()`)
4. Client makes HTTP GET/POST to the upstream API with `X-API-Key` auth
5. Response is deserialized into a typed struct
6. `format::format_*()` converts the struct to Markdown
7. Markdown string is returned to Claude

Write tools (`log_training_sets`, `log_auto_reg`, `trigger_garmin_sync`) additionally invalidate relevant cache entries after a successful write.

## HTTP Clients

### Common Infrastructure (`src/clients/mod.rs`)

```rust
pub fn http_client() -> reqwest::Client {
    // 5-second connect timeout
    // 30-second request timeout
}

pub async fn check_response<T: DeserializeOwned>(resp: Response) -> Result<T, McpServerError> {
    // Check HTTP status
    // Deserialize JSON body
    // On error: log body, return McpServerError::Upstream
}
```

### CoachClient (`src/clients/coach.rs`)

Calls gorilla_coach's v2 API. All requests include `X-API-Key` and `Accept: application/json` headers.

| Method | Endpoint | Used By |
|--------|----------|---------|
| `get_training(file?)` | `GET /api/v2/training` | `get_training_plan` tool |
| `get_files()` | `GET /api/v2/files` | `gorilla://files` resource |
| `get_e1rm_history()` | `GET /api/v2/training/e1rm-history` | `get_e1rm_history` tool |
| `get_exercise_history(exercise)` | `GET /api/v2/training/exercise-history` | `get_exercise_history` tool |
| `get_dashboard(date?)` | `GET /api/v2/dashboard` | `get_dashboard` tool |
| `get_sitrep()` | `GET /api/v2/reports/sitrep` | `get_sitrep` tool |
| `get_aar()` | `GET /api/v2/reports/aar` | `get_aar` tool |
| `get_debrief()` | `GET /api/v2/reports/debrief` | `get_debrief` tool |
| `get_auto_reg()` | `GET /api/v2/auto-reg` | `get_auto_regulator` tool |
| `get_settings()` | `GET /api/v2/settings` | `gorilla://settings` resource |
| `log_training(body)` | `POST /api/v2/training/log` | `log_training_sets` tool |
| `log_auto_reg(body)` | `POST /api/v2/auto-reg/log` | `log_auto_reg` tool |
| `trigger_sync()` | `POST /api/v2/settings/sync` | `trigger_garmin_sync` tool |

### GarminClient (`src/clients/garmin.rs`)

Calls garmin_api's v1 API. Routes are scoped to a user ID.

| Method | Endpoint | Used By |
|--------|----------|---------|
| `get_daily(date)` | `GET /api/v1/users/{id}/daily?date=` | `get_garmin_daily` tool |
| `get_daily_range(start, end)` | `GET /api/v1/users/{id}/daily?start=&end=` | `get_garmin_daily` tool |
| `get_vitals()` | `GET /api/v1/users/{id}/vitals` | `get_garmin_vitals` tool |
| `get_baseline(days?)` | `GET /api/v1/users/{id}/baseline` | `get_garmin_baseline` tool |
| `get_status()` | `GET /api/v1/users/{id}/status` | `gorilla://garmin/status` resource |

## Caching

`ResponseCache` in `src/cache.rs` wraps a [moka](https://github.com/moka-rs/moka) async cache:

- **TTL:** 5 minutes
- **Capacity:** 100 entries
- **Key:** String (e.g., `"vitals"`, `"baseline:7"`, `"baseline:14"`)
- **Value:** Pre-formatted Markdown string

**Cached tools:**
- `get_garmin_vitals` — key `"vitals"`
- `get_garmin_baseline` — key `"baseline:{days}"`

**Cache invalidation:**
All write operations invalidate the cache. The `ResponseCache` exposes granular invalidation methods:

```rust
pub async fn invalidate_training(&self)  // clears training-related entries
pub async fn invalidate_auto_reg(&self)  // clears auto-reg entries
pub async fn invalidate_sync(&self)      // clears all entries (full sync)
```

## Error Handling

`McpServerError` in `src/error.rs` has two variants:

```rust
pub enum McpServerError {
    Http(reqwest::Error),           // Network/connection errors
    Upstream { status: u16, body: String }, // HTTP error responses
}
```

The `to_tool_text()` method sanitizes errors for Claude:

| Error Type | User Sees | Logged |
|------------|-----------|--------|
| Connection timeout | "Service temporarily unavailable" | Full error |
| Connection refused | "Service temporarily unavailable" | Full error |
| HTTP 4xx/5xx | "Upstream error (HTTP {status})" | Status + body |
| JSON parse error | "Invalid response from service" | Full error |

Internal URLs, response bodies, and stack traces are never exposed to the MCP client.

## Formatting

All tool outputs are formatted as Markdown in `src/format.rs`. Each tool has a corresponding `format_*` function that converts typed response structs into human-readable Markdown with:

- Headers for sections (`# Training Plan: file.csv`)
- Tables for tabular data (e1RM history, exercise history, vitals comparison)
- Lists for structured data (settings, files, exercises)
- Status indicators (`[DONE]`, `[LOGGED]`, `[PARTIAL]`)
- Units on numeric values (`kg`, `ms`, `bpm`, `%`)

Reports from gorilla_coach (sitrep, AAR, debrief) arrive as pre-formatted Markdown and are passed through with a title header added.

## Chatbot (`gorilla_chatbot`)

### Two Operating Modes

**MCP Mode** — serves the browser chat UI:

```
Browser --POST /api/chat--> Axum
                             |
                             +-- spawn Claude CLI
                             |     --mcp-config (gorilla_mcp)
                             |     --model sonnet
                             |     --system-prompt "..."
                             |
                             +-- stream stdout as SSE events --> Browser
```

**Gateway Mode** — serves gorilla_coach as an LLM backend:

```
gorilla_coach --POST /api/gateway--> Axum
               X-API-Key: secret      |
                                      +-- spawn Claude CLI
                                      |     (no MCP tools)
                                      |     --model from request
                                      |     --system-prompt from request
                                      |
                                      +-- stream SSE events --> gorilla_coach
```

Gateway mode is enabled when `GATEWAY_API_KEY` is set. Authentication uses constant-time comparison via the `subtle` crate.

### Shared Core

Both modes use `stream_claude()`:

```rust
async fn stream_claude(
    mcp_config: Option<&str>,  // None for gateway mode
    model: &str,
    message: &str,
    system_prompt: &str,
    tx: Sender<Event>,         // SSE event sender
    ct: CancellationToken,     // For abort/stop
)
```

This function:
1. Spawns Claude CLI as a child process
2. Pipes stdin (message) and reads stdout (JSON stream)
3. Parses newline-delimited JSON events from Claude
4. Forwards status events (tool calls, init) as SSE
5. Sends the final result as a `result` SSE event
6. Handles timeout (180s) and cancellation

### Conversation History

Server-side, file-backed conversation history:

- Max 20 entries (older entries truncated)
- Stored as JSON array at `HISTORY_PATH` (default: `/tmp/gorilla-chat-history.json`)
- Appended to the system prompt to give Claude context across turns
- `GET /api/history` returns current entries
- `POST /api/stop` cancels the current request

### Frontend

Single-page app at `static/index.html` (853 lines, no build step):

- Dark theme with amber/green accents and scanline overlay
- JetBrains Mono + Oswald fonts
- Inline Markdown renderer (code blocks, tables, headers, bold, links)
- SSE streaming with thinking/status indicators
- Send/Stop/Clear controls
