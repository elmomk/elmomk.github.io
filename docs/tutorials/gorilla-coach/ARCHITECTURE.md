# Gorilla Coach Ecosystem -- Architecture Document

This document describes the complete architecture of the Gorilla Coach ecosystem: all services, how they communicate, internal design decisions, data flows, and deployment topology.

---

## Table of Contents

- [System Overview](#system-overview)
- [Component Diagram](#component-diagram)
- [Service Inventory](#service-inventory)
- [Tailscale Network Topology](#tailscale-network-topology)
- [Gorilla Coach Internal Architecture](#gorilla-coach-internal-architecture)
- [Authentication and Authorization](#authentication-and-authorization)
- [LLM Integration](#llm-integration)
- [Event-Driven Architecture](#event-driven-architecture)
- [Data Architecture](#data-architecture)
- [Deployment Architecture](#deployment-architecture)
- [Security Architecture](#security-architecture)

---

## System Overview

The Gorilla ecosystem consists of four independently deployed services, all connected over a private Tailscale mesh VPN:

| Service | Role | Stack | Database |
|---------|------|-------|----------|
| **gorilla_coach** | Main app: training tracker, dashboard, AI chat UI | Rust/Axum + Dioxus Wasm PWA | PostgreSQL 16 (TimescaleDB) |
| **garmin_api** | Garmin Connect data service with webhook events | Rust/Axum | SQLite (WAL mode) |
| **gorilla_mcp** | MCP server + chatbot gateway for Claude AI | Rust (rmcp + Axum) | None (stateless + cache) |
| **life_manager** | Household PWA (to-dos, groceries, watchlist) | Rust/Dioxus 0.7 fullstack | SQLite |

These are separate git repositories, each with independent Docker Compose deployments. They communicate exclusively over HTTP/JSON APIs on the Tailscale private network.

---

## Component Diagram

```mermaid
graph TB
    subgraph VPN["Tailscale Mesh VPN (WireGuard)"]
        GC["gorilla-coach<br/>(gorilla_coach)<br/>Axum :3000<br/>Dioxus Wasm PWA<br/>PostgreSQL/TS"]
        GA["garmin-api<br/>(garmin_api)<br/>Axum :3000<br/>SQLite<br/>Webhook dispatch"]
        GM["gorilla-chatbot<br/>(gorilla_mcp)<br/>Axum :8080<br/>Claude CLI spawn<br/>MCP tools :3000"]
        LM["life_manager<br/>(life_manager)<br/>Dioxus :8080<br/>SQLite"]
    end

    GC -->|"HTTP (X-API-Key)"| GA
    GA -->|"webhooks"| GC
    GC -->|"HTTP (SSE)"| GM
    GA -->|"HTTP"| LM

    TS["Tailscale Serve<br/>(auto TLS, :443)"]
    VPN --- TS
    TS --> B["Browser / Phone<br/>(any network)"]
```

### Data Flow Summary

1. **Browser** connects to gorilla_coach (training, dashboard, chat) and life_manager (household tasks) over Tailscale HTTPS
2. **gorilla_coach** fetches Garmin biometric data from **garmin_api** via REST (X-API-Key auth)
3. **gorilla_coach** delegates AI chat to **gorilla_mcp** chatbot gateway via SSE streaming
4. **gorilla_mcp** (MCP server mode) calls back into **gorilla_coach** and **garmin_api** to fetch data for Claude's tool calls
5. **garmin_api** pushes webhook events (daily_data_synced, sync_completed) to registered consumers
6. **life_manager** reads from both gorilla_coach and garmin_api for cross-service features

---

## Service Inventory

### gorilla_coach (This Repository)

**Purpose:** Primary fitness coaching platform -- training tracking, mesocycle management, Garmin dashboard, AI chat UI, Google Calendar/Sheets sync.

**Tech:** Rust, Axum 0.8, Dioxus 0.7 Wasm PWA, PostgreSQL 16 (TimescaleDB), sqlx 0.8

**Deployment:** 3 containers (tailscale + app + db)

**Key files:**
- `gorilla_server/src/main.rs` -- Entry point, router, middleware stack
- `gorilla_server/src/config.rs` -- All env var parsing (`AppConfig`)
- `gorilla_server/src/handlers/` -- HTTP handlers (v2 API, auth, chat, sync)
- `gorilla_client/src/` -- Dioxus Wasm PWA components
- `gorilla_shared/src/` -- Domain models shared between server and client

### garmin_api

**Repository:** `/home/mo/data/Documents/git/garmin_api`

**Purpose:** Standalone Garmin Connect data service. Handles OAuth authentication (SSO + MFA), background data sync across 12 health endpoints in parallel, encrypted credential storage, and webhook-based event dispatch. Extracted from gorilla_coach to serve as a shared data source.

**Tech:** Rust, Axum 0.8, SQLite (rusqlite + r2d2, WAL mode), ChaCha20Poly1305

**Deployment:** 2 containers (tailscale + app), no database container needed (SQLite)

**API:** REST with X-API-Key auth. Key endpoints:
- `POST /api/v1/users/{id}/credentials` -- Register Garmin credentials
- `POST /api/v1/users/{id}/sync` -- Trigger on-demand sync
- `GET /api/v1/users/{id}/daily?date=...` -- Query daily health data
- `GET /api/v1/users/{id}/vitals` -- Today + 7-day baseline
- `POST /api/v1/webhooks` -- Register webhook consumer

**Events:** HMAC-SHA256 signed webhooks with 3-retry exponential backoff:
- `daily_data_synced` -- Each day's data saved (full payload)
- `sync_completed` -- Full sync finishes (days_synced, errors, duration)
- `sync_failed` -- Auth or rate limit failure

### gorilla_mcp

**Repository:** `/home/mo/data/Documents/git/gorilla_mcp`

**Purpose:** Two-crate workspace providing (1) an MCP server that bridges Claude AI to training data and Garmin biometrics, and (2) a web-based chatbot with a gateway API for gorilla_coach.

**Tech:** Rust, rmcp (MCP protocol), Axum, Claude CLI

**Deployment:** 4 containers total:
- `tailscale` + `app` (MCP SSE server on :3000) -- for external MCP consumers
- `tailscale-chatbot` + `gorilla-chatbot` (Chat UI + gateway on :8080) -- for browser and gorilla_coach

**MCP Server (gorilla_mcp):**
- 10 tools: get_training_plan, get_e1rm_history, get_exercise_history, get_garmin_daily, get_garmin_vitals, get_garmin_baseline, get_auto_regulator, log_training_sets, log_auto_reg, trigger_garmin_sync
- 3 resources: gorilla://settings, gorilla://files, gorilla://garmin/status
- 4 prompts: readiness_check, weekly_review, program_evaluation, recovery_analysis
- Supports stdio and HTTP/SSE transports
- Calls gorilla_coach v2 API and garmin_api via HTTP clients with X-API-Key auth
- ResponseCache (moka, 5-min TTL) for vitals and baseline data

**Chatbot Gateway (gorilla_chatbot):**
- `POST /api/gateway` -- For gorilla_coach. Spawns Claude CLI without MCP tools (gorilla_coach bakes data into the prompt). Authenticated via X-API-Key header.
- `POST /api/chat` -- Browser-facing. Spawns Claude CLI with MCP config pointing to gorilla_mcp. Manages conversation history (20 entries, file-backed).
- Core function: `stream_claude(mcp_config, model, message, system_prompt, tx, ct)` -- spawns CLI process, streams SSE

### life_manager

**Repository:** `/home/mo/data/Documents/git/life_manager`

**Purpose:** Household management PWA with five modules: To-Dos, Groceries, Shopee Pick-ups, Watchlist, and Cycle Tracker.

**Tech:** Rust, Dioxus 0.7 (fullstack -- both server and Wasm client), Tailwind CSS v4, SQLite

**Deployment:** 2 containers (tailscale + app)

**Cross-service connections:**
- `GORILLA_COACH_URL` + `GORILLA_COACH_API_KEY` -- reads training data
- `GARMIN_API_URL` + `GARMIN_API_KEY` -- reads health metrics
- `GOOGLE_SA_KEY_FILE` -- Google Calendar integration
- `TMDB_API_KEY` -- Movie/TV metadata for watchlist

---

## Tailscale Network Topology

All services run on the same private Tailscale mesh VPN (tailnet). Each service gets its own Tailscale hostname and automatic TLS certificate via Tailscale Serve.

```mermaid
graph LR
    subgraph Tailnet
        GC["gorilla-coach.*.ts.net :443<br/>→ tailscale sidecar → app :3000<br/>+ db :5432 (shared network ns)"]
        GA["garmin-api.*.ts.net :443<br/>→ tailscale sidecar → app :3000"]
        GM["gorilla-mcp.*.ts.net :443<br/>→ tailscale sidecar → mcp :3000"]
        CB["gorilla-chatbot.*.ts.net :443<br/>→ tailscale sidecar → chatbot :8080"]
        LM["lifemanager.*.ts.net :443<br/>→ tailscale sidecar → app :8080"]
    end
```

**Key properties:**
- No port forwarding, no public internet exposure
- Tailscale ACLs control which devices/users can access each service
- Each sidecar has its own TS_AUTHKEY and `ts-serve.json` config
- Services reference each other by Tailscale FQDN (e.g., `https://garmin-api.<tailnet>.ts.net`)
- DNS resolution via Tailscale MagicDNS (`100.100.100.100`)

---

## Gorilla Coach Internal Architecture

### Three-Crate Workspace

```mermaid
graph BT
    GS["gorilla_server"] --> Shared["gorilla_shared<br/>(domain.rs, api.rs)"]
    GC["gorilla_client"] --> Shared
```

**gorilla_shared** (`gorilla_shared/src/`):
- `domain.rs` -- Core types: `User`, `GarminDailyData`, `GarminActivity`, `PrimaryLift`, `normalize_lift_name()`
- `api.rs` -- Request/response types with `#[serde(deny_unknown_fields)]` on critical types
- Optional `server` feature enables sqlx derives
- Used by both server and client via `[workspace.dependencies]`

**gorilla_server** (`gorilla_server/src/`):
- Axum 0.8 HTTP server, binary name `gorilla_coach`
- Handlers, repository layer, LLM subsystem, Garmin integration
- Background sync worker (hourly, tokio::spawn)
- Middleware stack: CSRF, rate limiting, security headers, HSTS, trace ID, compression

**gorilla_client** (`gorilla_client/src/`):
- Dioxus 0.7 Wasm SPA served at `/` via `fallback_service`
- Offline-first: IndexedDB cache with persistent handle, Service Worker
- Cache-first loading: show IndexedDB data instantly, refresh from network in background
- Chart.js integration via typed Wasm interop (Reflect::get + Function::call1, no eval())
- Self-hosted fonts (Orbitron, Roboto Mono, Material Icons)

### Request Flow

```mermaid
graph TB
    Browser["Browser (Dioxus Wasm)"] -->|"1. HTTP request to /api/v2/*"| Router["Axum Router (:3000)"]
    Router -->|"2. Middleware pipeline"| Handler["Handler function"]
    Handler -->|"3. AuthUser extractor<br/>(session cookie or API key)"| Logic["Business logic"]
    Logic -->|"4. Repository method (raw SQL via sqlx)"| DB["PostgreSQL (sqlx query macros)"]
    DB -->|"5. Response serialized as JSON"| Browser2["Browser (updates UI + IndexedDB cache)"]

    subgraph Middleware["Middleware pipeline (order matters)"]
        direction TB
        M1["TraceLayer (request logging)"]
        M2["trace_id_middleware (X-Trace-Id)"]
        M3["security_headers_middleware (CSP, HSTS)"]
        M4["csrf_middleware (Origin/Referer check)"]
        M5["RateLimiter (200/min general, 20/min chat)"]
        M6["CompressionLayer (gzip/br)"]
        M1 --> M2 --> M3 --> M4 --> M5 --> M6
    end
```

### Module Dependency Graph

```mermaid
graph TB
    Main["main.rs"] --> Config["config.rs<br/>(AppConfig::from_env)"]
    Main --> State["state.rs<br/>(AppState: repo, vault, config,<br/>cookie_key, llm, cache, metrics)"]
    Main --> MW["middleware/<br/>(csrf.rs, rate_limit.rs)"]
    Main --> Handlers["handlers/"]
    Main --> LLM["llm/<br/>(adapter.rs, chatbot_gateway.rs,<br/>noop.rs, analyst.rs)"]
    Main --> Reports["reports/<br/>(sitrep.rs, aar.rs, debrief.rs,<br/>gatekeeper.rs)"]
    Main --> Repo["repository/<br/>(mod.rs, users.rs, garmin.rs,<br/>training.rs, chat.rs, auto_reg.rs,<br/>mesocycle.rs)"]
    Main --> Vault["vault.rs<br/>(ChaCha20Poly1305 encryption)"]
    Main --> Util["util.rs<br/>(capitalize_first, build_day_label_info)"]

    Handlers --> HMod["mod.rs<br/>(AuthUser, get_session,<br/>sanitize_filename)"]
    Handlers --> Auth["auth.rs<br/>(login, logout, Google OAuth,<br/>dev-login)"]
    Handlers --> V2["v2/"]
    Handlers --> Chat["chat.rs<br/>(SSE streaming, tool-calling loop)"]
    Handlers --> ChatPrompt["chat_prompt.rs<br/>(system prompt construction)"]
    Handlers --> ChatTools["chat_tools.rs<br/>(tool definitions + ToolExecutor)"]
    Handlers --> Sync["sync.rs<br/>(background Garmin sync,<br/>cache invalidation)"]
    Handlers --> Cal["calendar.rs<br/>(Google Calendar event sync)"]
    Handlers --> Sheets["sheets.rs<br/>(Google Sheets read/write)"]
    Handlers --> Admin["admin.rs<br/>(admin-only endpoints)"]
    Handlers --> Negotiate["negotiate.rs<br/>(JSON/postcard content negotiation)"]

    V2 --> V2Mod["mod.rs<br/>(dashboard with moka cache + ETag)"]
    V2 --> Training["training.rs<br/>(plans, sets, schedule, e1RM)"]
    V2 --> Files["files.rs<br/>(uploads, Google Sheets import)"]
    V2 --> Settings["settings.rs<br/>(user settings, credentials)"]
    V2 --> AutoReg["auto_reg.rs<br/>(auto-regulation)"]
    V2 --> Meso["mesocycle/<br/>(templates.rs, generator.rs,<br/>macrocycle.rs)"]
```

### Caching Strategy

**Server-side (moka):**
- Dashboard cache in `AppState.dashboard_cache` with 5-minute TTL
- Keyed by `(user_id, view_type, date)` producing ETag-based conditional responses
- Invalidated by the background sync worker when new Garmin data arrives

**Client-side (IndexedDB):**
- Persistent IndexedDB handle opened once at app startup (`api.rs`)
- Training and dashboard data cached on every successful fetch
- Pages render cached data immediately, then refresh from network in background
- `storage.rs` provides `open_db()`, `put()`, `get()` helpers

**Service Worker (`public/sw.js`):**
- Versioned cache (`gorilla-coach-v{N}`), auto-bumped by `scripts/build.sh`
- Static assets: cache-first (stale-while-revalidate)
- API calls: network-first with 3-second timeout fallback to cache
- Navigation: SPA fallback (serve index.html for all routes)
- Push notification listener for training reminders

---

## Authentication and Authorization

### Session-Based Auth (Browser)

```mermaid
graph TB
    User["User"] -->|"/auth/login (email) or<br/>/auth/google (OAuth)"| Server["Server creates user in DB (if new)<br/>+ generates session UUID"]
    Server -->|"Set-Cookie: session=uuid|provider<br/>(signed with COOKIE_SECRET, HttpOnly, Secure)"| Cookie["Cookie set in browser"]
    Cookie --> Check["Every request: AuthUser extractor<br/>checks cookie via get_session()"]
    Check --> Parse["Parses signed cookie"]
    Parse --> Lookup["Looks up session in DB"]
    Lookup --> Return["Returns AuthUser { user_id, email }<br/>or 401 if invalid/missing"]
```

**Google OAuth flow:**
1. Browser redirects to `/auth/google` -> Google consent screen
2. Google redirects back to `/auth/callback` with auth code
3. Server exchanges code for tokens, extracts email
4. Creates/finds user, sets session cookie

**Dev-login:** When `DEV_LOGIN_ENABLED=true`, the `/auth/dev-login` route creates a session for `DEV_LOGIN_EMAIL` without OAuth. For local development only.

### API Key Auth (Service-to-Service)

```mermaid
graph TB
    Ext["External service"] -->|"/api/v2/* with X-API-Key header"| Auth["AuthUser extractor"]
    Auth --> S1["1. Check session cookie (browser auth)"]
    S1 -->|"No cookie"| S2["2. Check X-API-Key header"]
    S2 --> S3["3. Hash the key, look up in api_keys table"]
    S3 --> S4["4. Return AuthUser { user_id, email }<br/>from associated user"]
```

API keys are stored as SHA-256 hashes in the `api_keys` table. This allows gorilla_mcp and life_manager to call the gorilla_coach v2 API programmatically.

### Admin Authorization

Admin endpoints (`/api/admin/*`) require both a valid session AND the user's email in the `ADMIN_EMAILS` environment variable (comma-separated list).

---

## LLM Integration

### Chatbot Gateway Pattern

Gorilla Coach does not call LLM APIs directly. Instead, it delegates to the gorilla_mcp chatbot gateway:

```mermaid
graph LR
    subgraph GC["gorilla_coach"]
        Chat["chat.rs<br/>Build system prompt with<br/>biometric context + chat history"]
        Parse["Parse SSE events:<br/>- status (UI)<br/>- result (tokens)"]
    end

    subgraph GM["gorilla_mcp chatbot"]
        GW["gateway handler<br/>Receive prompt +<br/>system instruction"]
        CLI["Spawn Claude CLI<br/>(no MCP tools)<br/>Stream response back as SSE"]
    end

    Chat -->|"HTTP"| GW
    GW --> CLI
    CLI -->|"SSE stream"| Parse
```

**Why this pattern?**
- gorilla_coach bakes all relevant data (biometrics, chat history, training context) into the system prompt
- The gateway just proxies to Claude CLI -- no tool calling needed at this layer
- MCP tools are used separately when Claude Desktop or other MCP clients connect to gorilla_mcp directly
- Keeps LLM API credentials (Anthropic key) out of gorilla_coach

### Adapter Pattern

The LLM subsystem uses a trait-based adapter pattern (`llm/adapter.rs`):

```rust
pub trait LlmAdapter: Send + Sync {
    async fn generate(&self, prompt: &str, system: &str) -> Result<String, AppError>;
    async fn generate_stream(&self, prompt: &str, system: &str, tx: Sender<...>);
    async fn generate_with_tools_stream(&self, prompt: &str, system: &str, tools: &[ToolDef], executor: &dyn ToolExecutor, tx: Sender<...>);
    fn model_name(&self) -> String;
    async fn health_check(&self) -> bool;
}
```

Two implementations:
- **ChatbotGatewayAdapter** (`chatbot_gateway.rs`) -- Calls gorilla_mcp gateway, parses SSE stream with status/result events, 120s request timeout
- **NoopAdapter** (`noop.rs`) -- Returns friendly error when no gateway is configured; deterministic reports (SITREP/AAR/DEBRIEF) still work

### Chat Flow

1. User sends message (or clicks SITREP/AAR/DEBRIEF button)
2. `chat.rs` builds system prompt via `chat_prompt.rs` (includes current date, last 7 days biometrics, chat history)
3. Defines tool definitions via `chat_tools.rs` (get_biometric_history, get_last_activity, analyze_metric, etc.)
4. Calls `llm.generate_with_tools_stream()` which:
   - Sends prompt to gateway
   - If gateway returns tool call -> executes via `ToolExecutor` -> sends result back -> repeats (up to 8 turns)
   - Streams final response tokens via SSE to browser
5. Browser renders markdown via marked.js

### AI Analyst

The analyst subsystem (`llm/analyst.rs`) handles natural-language data queries:
1. `detect_intent()` classifies the user's question into a metric + time range
2. `get_metric_stats()` generates safe SQL against `ALLOWED_METRICS` whitelist
3. Returns computed statistics without requiring LLM involvement

---

## Event-Driven Architecture

### Honest Assessment

The Gorilla ecosystem uses **HTTP-based communication**, not message queues or event buses. The "event-driven" aspects are limited to:

1. **Webhooks from garmin_api** -- HMAC-signed HTTP callbacks to registered consumers
2. **Background polling** -- Hourly sync workers that pull data on a schedule
3. **Cache invalidation** -- Server-side moka cache cleared when new data arrives
4. **Client-side optimistic updates** -- IndexedDB cache-first loading with background refresh

There is no shared message broker (Kafka, RabbitMQ, etc.). Each service owns its data and exposes it via REST APIs. This is appropriate for the scale (single user, handful of services).

### garmin_api Webhook Events

```mermaid
graph TB
    GA["garmin_api (hourly sync worker)"] -->|"1. Syncs 12 Garmin endpoints in parallel<br/>2. Stores data in SQLite"| WD["Webhook Dispatcher"]
    WD -->|"POST callback_url<br/>X-Webhook-Signature (HMAC-SHA256)<br/>Retries: 3x exponential backoff"| GC["gorilla_coach<br/>(registered consumer)"]
    WD -->|"POST callback_url<br/>X-Webhook-Signature (HMAC-SHA256)<br/>Retries: 3x exponential backoff"| LM["life_manager<br/>(registered consumer)"]
```

Events: `daily_data_synced`, `sync_completed`, `sync_failed`, `credentials_updated`

### Background Sync (gorilla_coach)

```mermaid
graph TB
    Task["Hourly tokio::spawn task<br/>(handlers/sync.rs)"] --> S1["1. Iterate all users with Garmin API configured"]
    S1 --> S2["2. Skip users synced within rate limit window"]
    S2 --> S3["3. Call garmin_api:<br/>GET /api/v1/users/{id}/daily?start=...&end=..."]
    S3 --> S4["4. Upsert data into garmin_daily_data<br/>(COALESCE to preserve non-nulls)"]
    S4 --> S5["5. Invalidate dashboard_cache for affected users"]
    S5 --> Dash["Dashboard shows fresh data on next request"]
```

### Client-Side Cache-First Loading

```mermaid
graph TB
    Mount["Page mount (e.g., Training page)"] -->|"1. Read from IndexedDB (instant render)"| Cached["Show cached data in UI"]
    Cached -->|"2. Fetch from /api/v2/training (network request)"| Fresh["Update UI with fresh data + write to IndexedDB"]
```

This pattern is used for training, dashboard, and settings pages. It eliminates loading spinners on repeat visits and provides offline capability.

### Schedule Change -> Calendar Sync

```mermaid
graph TB
    User["User changes training schedule"] -->|"POST /api/v2/training/schedule/init or /shift"| Server["Server computes 10-day microcycle projection"]
    Server -->|"User clicks 'Sync to Calendar'<br/>POST /api/v2/training/schedule/sync-calendar"| SA["Server uses Google Service Account (JWT/RS256)"]
    SA -->|"Google Calendar API v3"| Cal["All-day events created/updated<br/>on user's calendar"]
```

---

## Data Architecture

### PostgreSQL Schema Overview

The database uses PostgreSQL 16 with the TimescaleDB extension. Migrations live in `gorilla_server/migrations/` and auto-run on startup via `Repository::migrate()`.

**Core tables:**

```mermaid
erDiagram
    users {
        UUID id PK
        TEXT email UK
        TIMESTAMPTZ created_at
    }

    user_settings {
        UUID user_id PK,FK
        TEXT garmin_username
        TEXT encrypted_garmin_password
        TEXT nonce
        TIMESTAMPTZ last_sync_at
        REAL body_fat_pct
    }

    garmin_daily_data {
        UUID user_id FK
        DATE date
        TEXT activities_json
        TIMESTAMPTZ synced_at
    }

    daily_logs {
        UUID user_id FK
        TIMESTAMPTZ time
        REAL hrv
        INT rhr
        REAL sleep_score
    }

    training_set_logs {
        UUID user_id FK
        TEXT plan_file
        TEXT day_label
        TEXT exercise_name
        INT set_number
        REAL actual_weight
        INT actual_reps
    }

    training_day_done {
        UUID user_id FK
        TEXT plan_file
        TEXT day_label
        BOOLEAN done
    }

    mesocycle_templates {
        BIGSERIAL id PK
        UUID user_id FK
        TEXT name
        TEXT template_type
        INT num_cycles
    }

    macrocycles {
        BIGSERIAL id PK
        UUID user_id FK
        TEXT name
    }

    macrocycle_slots {
        BIGSERIAL id PK
        BIGINT macrocycle_id FK
        INT slot_order
        BIGINT template_id FK
        TEXT status
    }

    users ||--o| user_settings : "has"
    users ||--o{ garmin_daily_data : "has"
    users ||--o{ daily_logs : "has"
    users ||--o{ training_set_logs : "logs"
    users ||--o{ training_day_done : "tracks"
    users ||--o{ mesocycle_templates : "creates"
    users ||--o{ macrocycles : "creates"
    macrocycles ||--o{ macrocycle_slots : "contains"
    mesocycle_templates ||--o{ macrocycle_slots : "referenced by"
```

### TimescaleDB Usage

The `daily_logs` table is created as a TimescaleDB hypertable (partitioned by time) when the extension is available. This is done with graceful fallback:

```sql
DO $$ BEGIN
    PERFORM create_hypertable('daily_logs', 'time', if_not_exists => TRUE);
EXCEPTION WHEN OTHERS THEN
    RAISE NOTICE 'TimescaleDB not available, using plain Postgres table';
END $$;
```

The `garmin_daily_data` table is a regular PostgreSQL table (not a hypertable) since it is keyed by `(user_id, date)` and accessed by date range queries that don't benefit significantly from time-series partitioning at single-user scale.

### Migration Strategy

- Migrations are SQL files in `gorilla_server/migrations/`
- Named with timestamp prefix: `YYYYMMDD[NN]_description.sql`
- Auto-run on server startup via `Repository::migrate()` (sqlx built-in migrator)
- All use `IF NOT EXISTS` / `IF NOT EXISTS` guards for idempotency
- Foreign keys use `ON DELETE CASCADE` (users -> all child tables)
- `ON DELETE RESTRICT` for template references from macrocycle slots (prevent accidental template deletion)
- CHECK constraints enforce valid enum values and ranges

### Encryption at Rest

Sensitive data is encrypted using ChaCha20Poly1305 (AEAD cipher) via `vault.rs`:

```mermaid
graph TB
    MK["MASTER_KEY (env var, ≥ 32 bytes)"] --> Key["First 32 bytes → ChaCha20Poly1305 key"]
    Key --> Enc["encrypt(plaintext) → (ciphertext_b64, nonce_b64)<br/>12-byte random nonce per encryption"]
    Key --> Dec["decrypt(ciphertext_b64, nonce_b64) → plaintext"]
```

Encrypted fields:
- Garmin passwords (`user_settings.encrypted_garmin_password` + `nonce`)
- Garmin OAuth tokens
- Google refresh tokens

Changing the `MASTER_KEY` invalidates all stored encrypted data.

---

## Deployment Architecture

### Docker Compose with Tailscale Sidecar Pattern

Every service follows the same deployment pattern:

```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    hostname: <service-name>
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve.json
    volumes:
      - ts-state:/var/lib/tailscale
      - ./ts-serve.json:/config/serve.json:ro
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    healthcheck:
      test: ["CMD", "tailscale", "status", "--json"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  app:
    build: .
    network_mode: service:tailscale  # <-- share tailscale's network namespace
    depends_on:
      tailscale:
        condition: service_healthy
```

### The `network_mode: service:tailscale` Pattern

This is the key deployment trick. By setting `network_mode: service:tailscale`, the app container shares the Tailscale sidecar's network namespace. This means:

- The app binds to `0.0.0.0:3000` but it's only reachable via the Tailscale network
- `localhost` inside the app container resolves to the tailscale container's loopback
- The tailscale `ts-serve.json` config proxies HTTPS :443 -> localhost:3000
- No Docker port mapping needed (`ports:` is not used)
- If the app needs to reach the database, the DB must also use `network_mode: service:tailscale` (gorilla_coach does this)

**Quirks:**
- The app container cannot resolve its own Tailscale hostname (circular dependency)
- DNS must be explicitly set to `100.100.100.100` (Tailscale MagicDNS) if the app needs to resolve other Tailscale hostnames
- `TS_USERSPACE=true` is needed when the app container needs internet access (e.g., gorilla_mcp chatbot calling api.anthropic.com)

### gorilla_coach Specific Deployment

```mermaid
graph TB
    subgraph Docker["gorilla_coach Docker Compose"]
        TS["tailscale (sidecar, HTTPS :443)<br/>hostname: gorilla-coach<br/>ts-serve.json: proxy :443 → :3000"]
        App["app (gorilla_coach binary, :3000)<br/>network_mode: service:tailscale<br/>volumes: app-data, uploads, sheets key"]
        DB["db (timescaledb:latest-pg16, :5432)<br/>network_mode: service:tailscale<br/>volumes: gorilla-db-data"]
    end

    TS -->|":443 → :3000"| App
    App -->|"localhost:5432<br/>(same network ns)"| DB
```

### Build Pipeline

gorilla_coach uses a local-build + thin-runtime-image pattern:

```mermaid
graph LR
    Build["1. ./scripts/build.sh<br/>Sync CSS, bump SW version<br/>cargo build --release<br/>dx build --release"] --> Docker["2. Dockerfile (debian:trixie-slim)<br/>COPY binary + static assets<br/>Non-root user (gorilla, uid 1000)<br/>~7s to build image"]
    Docker --> Deploy["3. ./scripts/deploy.sh up<br/>docker compose up -d --build"]
```

This avoids a multi-stage Docker build (which would require Rust + dioxus-cli in the image) and keeps the runtime image minimal.

### Startup Ordering

```mermaid
graph TB
    TS["tailscale<br/>(starts first, healthcheck:<br/>tailscale status --json)"] -->|"condition: service_healthy"| DB["db (starts after tailscale)"]
    DB -->|"condition: service_started"| App["app (starts last)"]
    App --> S1["1. AppConfig::from_env()"]
    S1 --> S2["2. Repository::new() — connect to PostgreSQL"]
    S2 --> S3["3. Repository::migrate() — run pending SQL migrations"]
    S3 --> S4["4. Build AppState (vault, cache, LLM adapter, metrics)"]
    S4 --> S5["5. Spawn background sync worker"]
    S5 --> S6["6. Bind to 0.0.0.0:3000"]
```

---

## Security Architecture

### Defense in Depth

```mermaid
graph TB
    Internet["Internet"] -->|"✗ No public access (Tailscale-only)"| VPN["Tailscale mesh VPN<br/>(WireGuard encryption)"]
    VPN -->|"ACLs control device/user access"| Serve["Tailscale Serve<br/>(auto TLS, HSTS)"]
    Serve --> MW["Axum middleware stack"]
    MW --> Auth["Auth layer"]
    Auth --> Data["Data layer"]

    subgraph MWStack["Middleware"]
        direction TB
        C1["CSRF (Origin/Referer validation)"]
        C2["Rate limiting (200/min, 20/min chat)"]
        C3["Security headers (CSP, X-Frame-Options)"]
        C4["HSTS (max-age=63072000)"]
        C5["Trace ID (X-Trace-Id)"]
    end

    subgraph AuthStack["Authentication"]
        direction TB
        A1["Signed session cookies (HMAC-SHA256)"]
        A2["API key auth (SHA-256, constant-time)"]
        A3["AuthUser extractor on every endpoint"]
    end

    subgraph DataStack["Data Protection"]
        direction TB
        D1["ChaCha20Poly1305 encryption at rest"]
        D2["SQL parameterization (sqlx)"]
        D3["File upload sanitization"]
        D4["20MB upload / 1MB JSON limit"]
    end
```

### Content Security Policy

```
default-src 'self';
script-src 'self';
style-src 'self' 'unsafe-inline';  (Dioxus injects styles at runtime)
connect-src 'self';
img-src 'self' data:;
font-src 'self'
```

No `unsafe-eval` -- all JS interop uses `Reflect::get` + `Function::call1`.

### Network Isolation

- All services are only accessible on the Tailscale tailnet
- No ports are exposed to the public internet
- Inter-service communication uses Tailscale DNS names with X-API-Key authentication
- Database (PostgreSQL) is only reachable within the Docker network namespace (no external port mapping)
- garmin_api webhook signatures use HMAC-SHA256 to verify payload authenticity

### Cookie Security

- Cookies signed with HMAC-SHA256 key derived from `COOKIE_SECRET`
- `HttpOnly` flag prevents JavaScript access
- `Secure` flag (when `COOKIE_SECURE=true`) ensures HTTPS-only transmission
- `SameSite=Lax` prevents CSRF on cross-origin requests
- CSRF middleware additionally validates Origin/Referer headers on all mutation requests (POST/PUT/DELETE)

### API Key Security

- Keys stored as SHA-256 hashes in the database (plaintext never stored)
- Constant-time comparison in gorilla_mcp gateway (via `subtle` crate)
- Each key is associated with a user account (inherits that user's permissions)
- `last_used_at` tracking for audit purposes

---

*Last updated: April 2026*
