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

```
                           Tailscale Mesh VPN (WireGuard)
    +====================================================================+
    |                                                                    |
    |   +---------------------+         +---------------------+          |
    |   |  gorilla-coach      |         |  garmin-api         |          |
    |   |  (gorilla_coach)    |  HTTP   |  (garmin_api)       |          |
    |   |                     +-------->|                     |          |
    |   |  Axum :3000         | X-API-  |  Axum :3000         |          |
    |   |  Dioxus Wasm PWA    |  Key    |  SQLite             |          |
    |   |  PostgreSQL/TS      |<--------+  Webhook dispatch   |          |
    |   |                     | webhooks|                     |          |
    |   +--------+------------+         +----------+----------+          |
    |            |                                 |                     |
    |            | HTTP (SSE)                      | HTTP                |
    |            v                                 v                     |
    |   +---------------------+         +---------------------+          |
    |   |  gorilla-chatbot    |         |  life_manager       |          |
    |   |  (gorilla_mcp)      |         |  (life_manager)     |          |
    |   |                     |         |                     |          |
    |   |  Axum :8080         |         |  Dioxus :8080       |          |
    |   |  Claude CLI spawn   |         |  SQLite             |          |
    |   |  MCP tools :3000    |         |                     |          |
    |   +---------------------+         +---------------------+          |
    |                                                                    |
    +====================================================================+
                                    |
                            Tailscale Serve
                            (auto TLS, :443)
                                    |
                                    v
                        +---------------------+
                        |  Browser / Phone    |
                        |  (any network)      |
                        +---------------------+
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

```
                     Tailnet
    +------------------------------------------+
    |                                          |
    |  gorilla-coach.<tailnet>.ts.net :443     |
    |    -> tailscale sidecar -> app :3000     |
    |       + db :5432 (shared network ns)     |
    |                                          |
    |  garmin-api.<tailnet>.ts.net :443        |
    |    -> tailscale sidecar -> app :3000     |
    |                                          |
    |  gorilla-mcp.<tailnet>.ts.net :443       |
    |    -> tailscale sidecar -> mcp :3000     |
    |                                          |
    |  gorilla-chatbot.<tailnet>.ts.net :443   |
    |    -> tailscale sidecar -> chatbot :8080 |
    |                                          |
    |  lifemanager.<tailnet>.ts.net :443       |
    |    -> tailscale sidecar -> app :8080     |
    |                                          |
    +------------------------------------------+
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

```
gorilla_shared (domain.rs, api.rs)
    ^                ^
    |                |
gorilla_server       gorilla_client
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

```
Browser (Dioxus Wasm)
    |
    | 1. HTTP request to /api/v2/*
    v
Axum Router (:3000)
    |
    | 2. Middleware pipeline (order matters):
    |    - TraceLayer (request logging)
    |    - trace_id_middleware (X-Trace-Id header)
    |    - security_headers_middleware (CSP, HSTS, X-Frame-Options)
    |    - csrf_middleware (Origin/Referer check on mutations)
    |    - RateLimiter (200 req/min general, 20 req/min for chat)
    |    - CompressionLayer (gzip/br)
    v
Handler function
    |
    | 3. AuthUser extractor (checks signed session cookie or API key)
    |    Pattern: AuthUser { user_id, .. }: AuthUser
    v
Business logic
    |
    | 4. Repository method (raw SQL via sqlx)
    v
PostgreSQL (sqlx query macros)
    |
    | 5. Response serialized as JSON
    v
Browser (updates UI + IndexedDB cache)
```

### Module Dependency Graph

```
main.rs
  +-- config.rs (AppConfig::from_env)
  +-- state.rs (AppState: repo, vault, config, cookie_key, llm, cache, metrics)
  +-- middleware/ (csrf.rs, rate_limit.rs)
  +-- handlers/
  |     +-- mod.rs (AuthUser, get_session, sanitize_filename)
  |     +-- auth.rs (login, logout, Google OAuth, dev-login)
  |     +-- v2/
  |     |     +-- mod.rs (dashboard with moka cache + ETag, reports dispatch)
  |     |     +-- training.rs (plans, sets, schedule, e1RM, exercise history)
  |     |     +-- files.rs (uploads, Google Sheets import/sync)
  |     |     +-- settings.rs (user settings, Garmin credentials, push subscriptions)
  |     |     +-- auto_reg.rs (auto-regulation)
  |     |     +-- mesocycle/ (templates.rs, generator.rs, macrocycle.rs)
  |     +-- chat.rs (SSE streaming, tool-calling loop)
  |     +-- chat_prompt.rs (system prompt construction)
  |     +-- chat_tools.rs (tool definitions + ToolExecutor impl)
  |     +-- sync.rs (background Garmin sync worker, cache invalidation)
  |     +-- calendar.rs (Google Calendar event sync)
  |     +-- sheets.rs (Google Sheets read/write via Service Account)
  |     +-- admin.rs (admin-only endpoints)
  |     +-- negotiate.rs (JSON/postcard content negotiation)
  +-- llm/ (adapter.rs, chatbot_gateway.rs, noop.rs, analyst.rs)
  +-- reports/ (sitrep.rs, aar.rs, debrief.rs, gatekeeper.rs)
  +-- repository/ (mod.rs, users.rs, garmin.rs, training.rs, chat.rs, auto_reg.rs, mesocycle.rs)
  +-- vault.rs (ChaCha20Poly1305 encryption)
  +-- util.rs (capitalize_first, build_day_label_info)
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

```
User -> /auth/login (email) or /auth/google (OAuth)
  |
  v
Server creates user in DB (if new) + generates session UUID
  |
  v
Set-Cookie: session={uuid}|{provider} (signed with COOKIE_SECRET, HttpOnly, Secure)
  |
  v
Every request: AuthUser extractor checks cookie via get_session()
  - Parses signed cookie
  - Looks up session in DB
  - Returns AuthUser { user_id, email }
  - Returns 401 if invalid/missing
```

**Google OAuth flow:**
1. Browser redirects to `/auth/google` -> Google consent screen
2. Google redirects back to `/auth/callback` with auth code
3. Server exchanges code for tokens, extracts email
4. Creates/finds user, sets session cookie

**Dev-login:** When `DEV_LOGIN_ENABLED=true`, the `/auth/dev-login` route creates a session for `DEV_LOGIN_EMAIL` without OAuth. For local development only.

### API Key Auth (Service-to-Service)

```
External service -> /api/v2/* with X-API-Key header
  |
  v
AuthUser extractor:
  1. Check session cookie (browser auth)
  2. If no cookie, check X-API-Key header
  3. Hash the key, look up in api_keys table
  4. Return AuthUser { user_id, email } from associated user
```

API keys are stored as SHA-256 hashes in the `api_keys` table. This allows gorilla_mcp and life_manager to call the gorilla_coach v2 API programmatically.

### Admin Authorization

Admin endpoints (`/api/admin/*`) require both a valid session AND the user's email in the `ADMIN_EMAILS` environment variable (comma-separated list).

---

## LLM Integration

### Chatbot Gateway Pattern

Gorilla Coach does not call LLM APIs directly. Instead, it delegates to the gorilla_mcp chatbot gateway:

```
gorilla_coach                    gorilla_mcp chatbot
+-------------------+           +-------------------+
| chat.rs           |           | gateway handler   |
|                   |  HTTP     |                   |
| Build system      +---------->| Receive prompt +  |
| prompt with       | SSE      | system instruction|
| biometric context | stream   |                   |
| + chat history    |<---------+ Spawn Claude CLI  |
|                   |           | (no MCP tools)    |
| Parse SSE events: |           |                   |
| - status (UI)     |           | Stream response   |
| - result (tokens) |           | back as SSE       |
+-------------------+           +-------------------+
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

```
garmin_api (hourly sync worker)
    |
    | 1. Syncs 12 Garmin endpoints in parallel
    | 2. Stores data in SQLite
    |
    v
Webhook Dispatcher
    |
    | 3. For each registered webhook:
    |    POST {callback_url}
    |    Headers: X-Webhook-Signature (HMAC-SHA256)
    |    Body: { event: "daily_data_synced", payload: { ... } }
    |    Retries: 3x with exponential backoff
    |
    +---> gorilla_coach (registered consumer)
    +---> life_manager (registered consumer)
```

Events: `daily_data_synced`, `sync_completed`, `sync_failed`, `credentials_updated`

### Background Sync (gorilla_coach)

```
Hourly tokio::spawn task (handlers/sync.rs)
    |
    | 1. Iterate all users with Garmin API configured
    | 2. Skip users synced within rate limit window
    | 3. Call garmin_api: GET /api/v1/users/{id}/daily?start=...&end=...
    | 4. Upsert data into garmin_daily_data (COALESCE to preserve non-nulls)
    | 5. Invalidate dashboard_cache for affected users
    v
Dashboard shows fresh data on next request
```

### Client-Side Cache-First Loading

```
Page mount (e.g., Training page)
    |
    | 1. Read from IndexedDB (instant render)
    v
Show cached data in UI
    |
    | 2. Fetch from /api/v2/training (network request)
    v
Update UI with fresh data + write to IndexedDB
```

This pattern is used for training, dashboard, and settings pages. It eliminates loading spinners on repeat visits and provides offline capability.

### Schedule Change -> Calendar Sync

```
User changes training schedule
    |
    | POST /api/v2/training/schedule/init or /shift
    v
Server computes 10-day microcycle projection
    |
    | User clicks "Sync to Calendar"
    | POST /api/v2/training/schedule/sync-calendar
    v
Server uses Google Service Account (JWT/RS256)
    |
    | Google Calendar API v3
    v
All-day events created/updated on user's calendar
```

---

## Data Architecture

### PostgreSQL Schema Overview

The database uses PostgreSQL 16 with the TimescaleDB extension. Migrations live in `gorilla_server/migrations/` and auto-run on startup via `Repository::migrate()`.

**Core tables:**

```
users
  +-- id (UUID PK)
  +-- email (UNIQUE)
  +-- created_at

user_settings
  +-- user_id (FK -> users, PK)
  +-- garmin_username
  +-- encrypted_garmin_password (ChaCha20)
  +-- nonce
  +-- last_sync_at
  +-- body_fat_pct, google_refresh_token, etc.

garmin_daily_data
  +-- user_id (FK -> users)
  +-- date (DATE)
  +-- 42+ metric columns (steps, HR, HRV, sleep, stress, body battery, etc.)
  +-- activities_json (TEXT, JSON array of activities)
  +-- synced_at
  +-- PK: (user_id, date)

daily_logs  [TimescaleDB hypertable]
  +-- user_id (FK -> users)
  +-- time (TIMESTAMPTZ)
  +-- hrv, rhr, sleep_score, training_load, notes

training_set_logs
  +-- user_id, plan_file, day_label, exercise_name, set_number (composite PK)
  +-- actual_weight, actual_reps, technique
  +-- logged_at

training_day_done
  +-- user_id, plan_file, day_label (composite PK)
  +-- done (BOOLEAN)
  +-- done_at

mesocycle_templates
  +-- id (BIGSERIAL PK)
  +-- user_id, name (UNIQUE together)
  +-- template_type (strength | hypertrophy | warrior)
  +-- num_cycles, days_json
  +-- cycle_percentages, cycle_reps (5/3/1 per-cycle config)

macrocycles
  +-- id (BIGSERIAL PK)
  +-- user_id, name (UNIQUE together)

macrocycle_slots
  +-- id (BIGSERIAL PK)
  +-- macrocycle_id (FK -> macrocycles, CASCADE)
  +-- slot_order, template_id (FK -> mesocycle_templates, RESTRICT)
  +-- status (pending | active | completed)
  +-- generated_file, started_at, completed_at

training_plan_exercises
  +-- Per-exercise configuration within generated plans

e1rm_history
  +-- Estimated 1-rep max tracking for primary lifts

chat_messages
  +-- user_id, role, content, time

api_keys
  +-- key_hash (SHA-256), user_id, label, last_used_at

push_subscriptions
  +-- Web push notification endpoints

auto_regulation_logs
  +-- Daily readiness tracking
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

```
MASTER_KEY (env var, >= 32 bytes)
    |
    v
First 32 bytes -> ChaCha20Poly1305 key
    |
    +-- encrypt(plaintext) -> (ciphertext_b64, nonce_b64)
    |   12-byte random nonce per encryption
    |
    +-- decrypt(ciphertext_b64, nonce_b64) -> plaintext
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

```
+-- tailscale (sidecar, HTTPS :443)
|     hostname: gorilla-coach
|     ts-serve.json: proxy :443 -> :3000
|
+-- app (gorilla_coach binary, :3000)
|     network_mode: service:tailscale
|     volumes: app-data, uploads, sheets key
|     DATABASE_URL -> localhost:5432 (same network ns as db)
|
+-- db (timescale/timescaledb:latest-pg16, :5432)
      network_mode: service:tailscale
      volumes: gorilla-db-data
```

### Build Pipeline

gorilla_coach uses a local-build + thin-runtime-image pattern:

```
1. ./scripts/build.sh
   - Syncs CSS (gorilla_client/assets/main.css)
   - Bumps Service Worker version in sw.js
   - cargo build -p gorilla_server --release
   - cd gorilla_client && dx build --release

2. Dockerfile (debian:trixie-slim)
   - COPY target/release/gorilla_coach .
   - COPY target/dx/gorilla_client/release/web/public ./target/dx/...
   - Non-root user (gorilla, uid 1000)
   - ~7 seconds to build Docker image

3. ./scripts/deploy.sh up
   - docker compose up -d --build
```

This avoids a multi-stage Docker build (which would require Rust + dioxus-cli in the image) and keeps the runtime image minimal.

### Startup Ordering

```
tailscale (starts first, healthcheck: tailscale status --json)
    |
    | condition: service_healthy (wait for Tailscale to connect)
    v
db (starts after tailscale)
    |
    | condition: service_started
    v
app (starts last)
    |
    | 1. AppConfig::from_env() -- parse all env vars
    | 2. Repository::new() -- connect to PostgreSQL
    | 3. Repository::migrate() -- run pending SQL migrations
    | 4. Build AppState (vault, cache, LLM adapter, metrics)
    | 5. Spawn background sync worker
    | 6. Bind to 0.0.0.0:3000
```

---

## Security Architecture

### Defense in Depth

```
Internet
    |
    X  No public access (Tailscale-only)
    |
Tailscale mesh VPN (WireGuard encryption)
    |
    | ACLs control device/user access
    v
Tailscale Serve (auto TLS, HSTS)
    |
    v
Axum middleware stack:
    1. CSRF (Origin/Referer validation on mutations)
    2. Rate limiting (200 req/min general, 20 req/min chat)
    3. Security headers (CSP, X-Frame-Options, X-Content-Type-Options)
    4. HSTS (max-age=63072000)
    5. Trace ID (X-Trace-Id for request tracking)
    |
    v
Auth layer:
    - Signed session cookies (HMAC-SHA256 derived key)
    - API key auth (SHA-256 hashed, constant-time comparison)
    - AuthUser extractor on every protected endpoint
    |
    v
Data layer:
    - ChaCha20Poly1305 encryption for credentials at rest
    - SQL parameterization (sqlx) prevents injection
    - File upload sanitization (whitelist: [a-zA-Z0-9._-])
    - 20MB upload limit, 1MB JSON body limit
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
