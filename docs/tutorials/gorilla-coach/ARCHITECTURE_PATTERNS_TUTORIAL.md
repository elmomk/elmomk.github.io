# Monolith vs Microservices vs Modular Monolith

> *"All architectures become iterative — the only question is whether you
> iterate deliberately or whether the codebase iterates toward chaos."*
> — Adapted from Neal Ford, *Building Evolutionary Architectures*

This tutorial examines the three dominant architectural patterns in software
engineering — the traditional monolith, microservices, and the modular monolith —
through the lens of the Gorilla Coach codebase. We analyze where Gorilla Coach
falls today, why that's the right choice, what the alternatives would look like,
and when (if ever) you'd migrate between them.

---

## Reference Texts

| Abbreviation | Book |
|---|---|
| **BM** | Sam Newman — *Building Microservices*, 2nd Ed. (2021) |
| **DDIA** | Martin Kleppmann — *Designing Data-Intensive Applications* (2017) |
| **DDD** | Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) |
| **IDDD** | Vaughn Vernon — *Implementing Domain-Driven Design* (2013) |
| **CA** | Robert C. Martin — *Clean Architecture: A Craftsman's Guide to Software Structure and Design* (2017) |
| **PoEAA** | Martin Fowler — *Patterns of Enterprise Application Architecture* (2002) |
| **SRE** | Beyer, Jones, Petoff & Murphy — *Site Reliability Engineering* (2016) |
| **PP** | Andrew Hunt & David Thomas — *The Pragmatic Programmer*, 20th Anniversary Ed. (2019) |
| **USAH** | Nemeth, Snyder, Hein, Whaley & Mackin — *UNIX and Linux System Administration Handbook*, 5th Ed. (2017) |
| **RDD** | Neal Ford, Rebecca Parsons & Patrick Kua — *Building Evolutionary Architectures*, 2nd Ed. (2023) |
| **MfM** | Kamil Grzybek — "Modular Monolith: A Primer" (2019, blog series + reference architecture) |
| **SaP** | Vlad Khononov — *Learning Domain-Driven Design* (2021) |
| **ZtP** | Luca Palmieri — *Zero To Production in Rust* (2022) |
| **MP** | Chris Richardson — *Microservices Patterns* (2018) |
| **RiA** | Tim McNamara — *Rust in Action* (2021) |

---

## Table of Contents

1. [Definitions: What Are We Actually Talking About?](#1-definitions)
2. [The Traditional Monolith](#2-the-traditional-monolith)
3. [Gorilla Coach: Anatomy of a Monolith](#3-gorilla-coach-anatomy-of-a-monolith)
4. [Why Gorilla Coach's Monolith Is the Right Choice](#4-why-the-monolith-is-the-right-choice)
5. [When Monoliths Go Wrong: The Big Ball of Mud](#5-when-monoliths-go-wrong)
6. [Microservices: The Distributed Alternative](#6-microservices)
7. [Gorilla Coach as Microservices: A Thought Experiment](#7-gorilla-coach-as-microservices)
8. [The Hidden Costs of Microservices](#8-the-hidden-costs-of-microservices)
9. [The Modular Monolith: Best of Both Worlds?](#9-the-modular-monolith)
10. [Gorilla Coach as a Modular Monolith](#10-gorilla-coach-as-a-modular-monolith)
11. [Bounded Contexts and Domain Boundaries](#11-bounded-contexts-and-domain-boundaries)
12. [Data Ownership and the Database Question](#12-data-ownership)
13. [The Dependency Graph: Coupling in Practice](#13-the-dependency-graph)
14. [Communication Patterns: In-Process vs Network](#14-communication-patterns)
15. [Testing Strategies Across Architectures](#15-testing-strategies)
16. [Deployment and Operations](#16-deployment-and-operations)
17. [The Migration Path: Monolith → Modular Monolith → Selective Extraction](#17-the-migration-path)
18. [Decision Framework: How to Choose](#18-decision-framework)
19. [Further Reading](#19-further-reading)

---

## 1. Definitions

Before diving in, let's be precise about what these terms mean. The industry uses
them loosely, which causes endless confusion.

### Monolith

A **monolith** is a single deployable unit that contains all the application's
functionality. One binary, one process, one deployment artifact. All code shares
the same memory space, the same database connection, and the same runtime.

> **BM**, Chapter 1: *"A monolith is a single deployable unit. All functionality
> is deployed as one entity... This simplicity is the monolith's greatest
> strength and its greatest weakness."*

### Microservices

**Microservices** are independently deployable services that are modeled around
a business domain. Each service owns its data, communicates over the network
(HTTP, gRPC, message queues), and can be developed, deployed, and scaled
independently.

> **BM**, Chapter 1: *"Microservices are independently releasable services that
> are modeled around a business domain. A service encapsulates functionality and
> makes it accessible to other services via networks."*

### Modular Monolith

A **modular monolith** is a single deployable unit (like a monolith) but with
strict internal module boundaries that enforce separation between domains. Modules
communicate through well-defined interfaces, not by reaching into each other's
internals. The key difference from a raw monolith: modules could theoretically be
extracted into services without rewriting the business logic.

> **MfM** (Grzybek): *"A modular monolith has the deployment simplicity of a
> monolith with the logical separation of microservices. Modules are the unit of
> modularity; the codebase enforces boundaries rather than relying on
> developer discipline."*

---

## 2. The Traditional Monolith

A monolith isn't inherently good or bad — it's a **deployment topology**. Within
that topology, the internal code can be well-structured or a disaster. The
distinction matters enormously.

### Strengths

1. **Simplicity of development**: One repo, one build, one debugger session. You
   can trace a request from HTTP handler to database query in your IDE without
   crossing process boundaries.

2. **Simplicity of deployment**: Build one binary, copy it to a server, restart.
   No service discovery, no API versioning, no distributed tracing needed.

3. **Simplicity of testing**: Integration tests that exercise the full stack are
   straightforward. No need to mock network boundaries or stand up dependent
   services.

4. **Transactional consistency**: All data access goes through one database
   connection pool. ACID transactions work naturally. You can atomically update a
   user's settings and their Garmin credentials in one transaction.

5. **Performance**: In-process function calls are nanoseconds. No serialization,
   no network latency, no retry logic.

> **CA**, Chapter 16 (Independence): *"A good architecture maximizes the number
> of decisions not made. A monolith defers the decision of how to distribute
> the system until the need is proven."*

### Weaknesses

1. **Coupling creep**: Without discipline, modules gradually reach into each
   other's internals. A change to the Garmin sync logic accidentally breaks
   the dashboard because they share a data structure.

2. **Scaling is all-or-nothing**: You can't scale the LLM chat handler
   independently from the static settings page. You scale the whole binary,
   even though only one path is under load.

3. **Deployment is all-or-nothing**: A change to the training page requires
   redeploying the entire application, including the Garmin sync worker.

4. **Technology lock-in**: Everything must be written in the same language and
   framework. You can't use Python for ML analysis and Rust for the web layer.

> **PoEAA**, Chapter 1 (Layering): *"The hardest part of a monolith is not
> building it — it's preventing it from becoming a ball of mud over time."*

---

## 3. Gorilla Coach: Anatomy of a Modular Monolith

Gorilla Coach is a **well-structured modular monolith** organized as a three-
crate Cargo workspace. It evolved from a single-crate monolith to enforce
stronger domain boundaries through crate-level separation.

### The Three-Crate Workspace

```
gorilla_coach/                        (workspace root)
├── gorilla_shared/                   ← domain models + API types (innermost)
│   └── src/
│       ├── domain.rs                 ← User, GarminDailyData, TrainingSetLog, etc.
│       └── api.rs                    ← DashboardResponse, TrainingPlanResponse, etc.
├── gorilla_server/                   ← Axum HTTP server (v1 HTML + v2 JSON API)
│   └── src/
│       ├── main.rs                   ← entry point, router, middleware, background worker
│       ├── config.rs                 ← environment configuration
│       ├── state.rs                  ← shared AppState struct
│       ├── error.rs                  ← unified error type
│       ├── vault.rs                  ← encryption
│       ├── garmin/                   ← Garmin Connect SSO + API
│       ├── handlers/                 ← HTTP route handlers (auth, calendar, chat, v2, etc.)
│       ├── llm/                      ← LLM adapter trait + implementations
│       ├── reports/                  ← tactical report generation (SITREP, AAR, DEBRIEF)
│       ├── repository/               ← all database access (split by domain submodule)
│       ├── ui/                       ← legacy v1 server-rendered HTML
│       └── middleware/               ← CSRF, rate limiting
└── gorilla_client/                   ← Dioxus 0.7 Wasm PWA
    └── src/
        ├── main.rs                   ← Router, App component
        ├── api.rs                    ← HTTP client (gloo_net → /api/v2/*)
        └── pages/                    ← page components (dashboard, training, chat, etc.)
```

This three-crate split provides **compile-time boundary enforcement**:
- `gorilla_shared` knows nothing about HTTP, Wasm, or databases — it's pure
  domain types. This is the Dependency Rule from Clean Architecture.
- `gorilla_server` depends on `gorilla_shared` but not on `gorilla_client`.
- `gorilla_client` depends on `gorilla_shared` but not on `gorilla_server`.
- A field rename in `gorilla_shared/src/api.rs` triggers compile errors in
  *both* server and client — impossible with separate JS/Rust codebases.

### The God Object: AppState

From `gorilla_server/src/state.rs`:

```rust
pub struct AppState {
    pub repo: Repository,
    pub cookie_key: Key,
    pub llm: Arc<dyn crate::llm::LlmAdapter>,
    pub metrics: Arc<Mutex<Metrics>>,
    pub llm_logger: LlmLogger,
    pub vault: Arc<Vault>,
    pub config: AppConfig,
    pub http_client: reqwest::Client,
    pub sa_token_cache: Arc<tokio::sync::Mutex<Option<(String, std::time::Instant)>>>,
}
```

Every handler receives the entire `AppState` via Axum's `State` extractor.
The dashboard handler has access to the LLM adapter. The chat handler has
access to the Vault. Nothing prevents any handler from using any capability.

This is characteristic of the server crate: **shared state with broad access**.
It works because developer discipline maintains the invariants, and the crate
boundary prevents the Dioxus client from directly accessing server internals.

### The Repository (Split by Domain)

From `gorilla_server/src/repository/mod.rs`:

```rust
pub struct Repository {
    pool: PgPool,
}
```

One struct, one connection pool — but queries are now split into submodules:
`users.rs`, `garmin.rs`, `training.rs`, `chat.rs`, `auto_reg.rs`. Each
submodule contains only queries for its domain. This is a step toward the
modular monolith pattern: same deployment unit, separated concerns.

The Repository has methods like:
- `get_or_create_user()` — auth domain
- `save_chat_message()`, `get_chat_history()` — chat domain
- `upsert_garmin_daily()`, `get_garmin_range()` — biometric domain
- `upsert_training_sets()`, `get_training_logs()` — training domain
- `mark_training_day_done()`, `unmark_training_day_done()`, `get_training_days_done()` — training domain
- `get_users_with_garmin()`, `save_garmin_token()` — sync domain
- `update_profile()`, `save_manual_body_fat()` — settings domain

Six domains, one Repository. This is the **Transaction Script** pattern from
Fowler's PoEAA — simple, direct, and effective at small scale.

> **PoEAA**, Chapter 9 (Transaction Script): *"A Transaction Script organizes
> business logic by procedures where each procedure handles a single request
> from the presentation... Its simplicity is its greatest asset; it works well
> for applications with a small amount of logic."*

### The Shared Domain Module

From `gorilla_shared/src/domain.rs`, all domain structures live in the shared crate:

```rust
pub struct User { ... }
pub struct ChatMessage { ... }
pub struct UserSettings { ... }
pub struct GarminDailyData { ... }
pub struct TrainingSetLog { ... }
pub struct AnalystIntent { ... }
```

These structures belong to different bounded contexts (auth, chat, biometrics,
training, analytics) but are co-located in one crate. The key advantage: both
the server and the Wasm client compile against the same types, so API contracts
are enforced at compile time.

### The Shared Error Type

From `gorilla_server/src/error.rs`:

```rust
pub enum AppError {
    Llm(String),
    Sheets(String),
    Garmin(String),
    Auth(String),
    Internal(#[from] anyhow::Error),
}
```

A single error enum encompasses all failure modes across all domains. This is
pragmatic — one error type means one `impl IntoResponse` — but it couples
unrelated domains through a shared type.

---

## 4. Why the Monolith Is the Right Choice

Given the project's current state, the monolith is not just acceptable — it's
**optimal**. Here's why:

### 1. Small Team (Likely Solo Developer)

> **BM**, Chapter 3: *"The real enemy is not the monolith; it's the coupling.
> A small team working on a monolith can move faster than a small team working
> on microservices, because they don't pay the distributed systems tax."*

Microservices are an organizational pattern as much as a technical one. Conway's
Law states that system architecture mirrors communication structures. A solo
developer communicates perfectly with themselves — there's no coordination
overhead to optimize away.

### 2. Domain Exploration Phase

Gorilla Coach is still evolving — new Garmin fields get added, the AI analyst
system uses a novel two-stage approach, chat modes (SITREP/AAR/DEBRIEF) are
being refined. In this phase, the ability to refactor freely across module
boundaries is essential.

> **DDD**, Chapter 14 (Maintaining Model Integrity): *"Context boundaries should
> emerge from the domain learning process, not from premature architectural
> decisions. Draw boundaries only where the cost of integration exceeds the cost
> of separation."*

If the analyst's `AnalystIntent` needs a new field from `GarminDailyData`, you
change two structs in one commit. In a microservices world, that's a contract
change across two service teams with API versioning.

### 3. Operational Simplicity

The entire deployment is:

```bash
cargo build --release
./gorilla_coach server
```

Plus a PostgreSQL instance and Tailscale for remote access with TLS. That's it.
No container orchestrator, no service mesh, no distributed tracing infrastructure,
no manual certificate management. Tailscale's `serve` command proxies HTTPS to
the local Axum server with automatic Let's Encrypt certificates. The
`docker-compose.yaml` runs two containers:

```yaml
services:
  gorilla-coach:
    build: .
    command: server
  gorilla-db:
    image: timescale/timescaledb:latest-pg16
```

> **USAH**, Chapter 29 (Virtualization): *"Every additional service in your
> infrastructure is a liability: it must be configured, monitored, patched,
> secured, backed up, and documented. The simplest architecture that meets
> requirements is the one that will be most reliable."*

### 4. Transactional Simplicity

When a user saves settings and triggers a Garmin login, the handler can:
1. Encrypt the password with Vault
2. Store settings in the DB
3. Attempt Garmin SSO login
4. Store the OAuth token
5. Update in-memory metrics

All in one function, with one error handling path. In microservices, steps 1-2
are the "Settings Service," step 3-4 is the "Garmin Service," and step 5 is the
"Metrics Service." Now you need eventual consistency, compensating transactions,
and saga orchestration for what was a simple function.

> **MP**, Chapter 4 (Managing Transactions with Sagas): *"With microservices,
> you can't have a simple ACID transaction that spans services... Instead, you
> must use a saga — a sequence of local transactions, each of which publishes
> an event that triggers the next step."*

---

## 5. When Monoliths Go Wrong

The monolith's strengths become weaknesses when certain conditions emerge. Let's
identify the warning signs using Gorilla Coach as context.

### The Big Ball of Mud

> **CA**, Chapter 34 (The Missing Chapter): *"The Big Ball of Mud is the most
> common software architecture in practice. It's the natural result of making
> changes without considering the broader structure."*

Warning signs that Gorilla Coach could drift toward a Big Ball of Mud:

**1. Circular Dependencies**

Currently, the dependency graph is mostly acyclic:

```
main.rs → handlers/ → repository.rs → domain.rs
               ↓           ↓
           llm/         vault.rs
               ↓
          garmin.rs
```

But `handlers/settings.rs` directly calls `garmin::garmin_login()`, and
`handlers/sync.rs` directly calls `garmin::garmin_login()` too. If `garmin.rs`
started importing from `handlers/` (to access `get_session`, for example),
you'd have a cycle. Rust's module system prevents circular `mod` declarations,
but you can create logical cycles through shared types in `domain.rs`.

**2. Shared Mutable State**

`AppState` is cloned into every handler. The `Arc<Mutex<Metrics>>` is modified by
auth handlers, sync handlers, and settings handlers. As the application grows,
more fields accumulate in `AppState`, and more handlers mutate shared state. This
is the "God Object" anti-pattern.

> **PP**, Chapter 28 (Decoupling): *"We want to design components that are
> self-contained: independent, and with a single, well-defined purpose... When
> components are connected by shared mutable state, changes to one component
> ripple unpredictably to others."*

**3. Repository Bloat**

`repository.rs` currently has methods for six domains. At 20 methods it's
manageable. At 100 methods (as the app grows), finding the right function becomes
a cognitive burden, and there's no compile-time guarantee that the chat handler
won't call a training repository method.

**4. The Blast Radius Problem**

A bug in `garmin.rs` that panics in the SSO parsing code takes down the web
server, the dashboard, the chat — everything. In a monolith, fault isolation
requires careful coding discipline. One unhandled error in any module can take
down the entire service.

---

## 6. Microservices

Now let's examine the opposite extreme.

### Core Principles

> **BM**, Chapter 1: *"Microservices embrace the concept of information hiding.
> They have an interface that can be changed independently, and the
> implementation is hidden behind that interface."*

The defining characteristics:

1. **Independent deployability**: Each service can be deployed without deploying
   any other service. This requires each service to have a stable API contract.

2. **Modeled around business domains**: Services map to business capabilities
   (user management, biometric data, training plans), not technical layers
   (data access, business logic, UI).

3. **Own their data**: Each service has its own database (or schema). No shared
   database between services. This is the hardest rule and the one most often
   violated.

4. **Communicate over the network**: Services talk via HTTP/gRPC/messaging,
   never by sharing memory or libraries.

5. **Decentralized governance**: Each service can use different languages,
   frameworks, and data stores.

### The Promise

- Scale independently: Run 10 instances of the Chat Service and 1 instance of
  the Settings Service.
- Deploy independently: Ship a fix to Garmin sync without touching the dashboard.
- Fault isolation: If the Garmin API is down, chat still works.
- Team autonomy: The "Biometric Team" owns the Garmin service end-to-end.

---

## 7. Gorilla Coach as Microservices: A Thought Experiment

Let's imagine decomposing Gorilla Coach into microservices. Where would we draw
the boundaries?

### Service Decomposition

Based on the current module structure and the domain boundaries visible in the
code:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Auth Service      │     │ Biometric Service │    │ Chat Service      │
│ ──────────────── │     │ ──────────────── │     │ ──────────────── │
│ handlers/auth.rs  │     │ garmin.rs         │    │ handlers/chat.rs  │
│ Google OAuth      │     │ handlers/sync.rs  │    │ llm/              │
│                   │     │ handlers/dash.rs  │    │ SSE streaming     │
│ DB: users         │     │                   │    │                   │
│     user_settings │     │ DB: garmin_daily   │   │ DB: chat_messages │
│     (credentials) │     │     manual_metrics │   │                   │
└──────────────────┘     └──────────────────┘     └──────────────────┘
          ↑                        ↑                        ↑
          │                        │                        │
          └────────────┬───────────┴────────────────────────┘
                       │
               ┌───────┴──────┐
               │ API Gateway   │
               │ (nginx/envoy) │
               └───────────────┘

┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Training Service  │    │ File Service      │     │ Analytics Service │
│ ──────────────── │    │ ──────────────── │     │ ──────────────── │
│ handlers/train.rs │    │ handlers/files.rs │    │ llm/analyst.rs    │
│ CSV parsing       │    │ handlers/sheets.rs│    │ SQL metric engine │
│                   │    │                   │     │                   │
│ DB: training_logs │    │ Storage: S3/MinIO │    │ DB: read-only     │
│                   │    │ DB: sheet metadata│     │     biometric view│
└──────────────────┘    └──────────────────┘     └──────────────────┘
```

### What Would Change: The Settings Flow

Currently, `handlers/settings.rs` → `update_settings()` does this in one
function:

```rust
pub async fn update_settings(
    State(state): State<AppState>,
    jar: SignedCookieJar,
    Form(input): Form<SettingsForm>,
) -> impl IntoResponse {
    let (user_id, _) = get_session(&jar)?;
    let login_result = garmin::garmin_login(&input.garmin_user, &input.garmin_pass).await;
    let (enc_pass, nonce) = vault.encrypt(&input.garmin_pass);
    state.repo.update_settings(&settings).await?;
    state.metrics.lock().unwrap().garmin_auth_valid = creds_valid;
    // ... return HTML
}
```

In a microservices world, this becomes:

```
Browser → API Gateway → Auth Service: validate session cookie
                       → Auth Service: store encrypted credentials
                       → Auth Service → Biometric Service: validate Garmin login
                       → Auth Service → Metrics Service: update garmin_auth_valid
                       → Auth Service: return response
```

That's **four network hops** for what's currently an in-process function call.
Each hop adds latency, can fail independently, and needs retry logic.

But here's the real problem: the **Vault** (encryption key) now needs to be
accessible to the Auth Service. The Garmin login function needs to be accessible
too. Either we duplicate the encryption code across services, or we create a
shared "Crypto Service" — which re-introduces coupling.

### What Would Change: The Chat Flow

Currently, `handlers/chat.rs` does this in the tool-calling loop:

```rust
// Get biometric context
let (bio_ctx, msgs) = state.repo.get_chat_context(user_id, 100).await?;

// Read user's uploaded files
let files = std::fs::read_dir(user_uploads_dir(user_id))?;

// Call LLM with tools
let response = state.llm.generate_with_tools(prompt, system, &tools, &executor).await?;

// The tool executor can:
// - Query garmin_daily_data (biometric domain)
// - Read files (file domain)
// - Parse training plans (training domain)
// - Run analyst SQL queries (analytics domain)
```

In a microservices world, each tool call crosses a network boundary:

```
Chat Service → Biometric Service: get_biometric_history(start, end)
Chat Service → File Service: list_files(user_id)
Chat Service → File Service: read_file(user_id, filename)
Chat Service → Training Service: get_training_plan(user_id, file)
Chat Service → Analytics Service: analyze_metric(user_id, metric, range)
```

With up to 8 tool-calling turns and 5 tools, that's potentially **40 network
round-trips** per chat message. At ~10ms per call, that's 400ms of pure network
overhead added to every chat interaction — on top of the LLM inference time.

> **BM**, Chapter 4 (Communication Styles): *"The move to inter-process
> communication introduces new sources of failure, latency, and complexity.
> Calls that were nanoseconds become milliseconds, calls that always succeeded
> now fail regularly, and calls that were atomic now require distributed
> coordination."*

---

## 8. The Hidden Costs of Microservices

This thought experiment reveals costs that are often underestimated:

### 1. Data Consistency

Currently, the Repository provides transactional consistency:

```rust
// This is implicitly a single DB transaction
state.repo.update_settings(&settings).await?;
state.repo.save_garmin_token(user_id, &enc, &n).await?;
```

If the second call fails, the first has already committed. In a monolith with
one database, you'd wrap these in a transaction. In microservices with separate
databases, you need a saga:

```
1. Auth Service: save credentials → emit CredentialsSaved event
2. Biometric Service: receives event → validate Garmin login
   Success → emit GarminValidated
   Failure → emit GarminValidationFailed
3. Auth Service: receives GarminValidationFailed → compensate: delete credentials
```

This is the **Saga pattern** from Chris Richardson's *Microservices Patterns*.
It replaces a 5-line function with an event-driven state machine spread across
two services, a message broker, and compensation logic.

> **MP**, Chapter 4: *"The biggest challenge with sagas is the lack of
> isolation. Intermediate states are visible to other transactions, and you
> must design compensating transactions for each step that might need to be
> undone."*

### 2. Shared Domain Knowledge

`GarminDailyData` is used by:
- `repository.rs` (database persistence)
- `handlers/dashboard.rs` (rendering charts)
- `handlers/chat.rs` (biometric context for LLM)
- `llm/analyst.rs` (SQL metric analysis)
- `handlers/sync.rs` (Garmin API → database)

In microservices, this struct would be defined in a shared library (coupling
services via a common dependency) or duplicated in each service (risking drift).
Neither option is great.

> **DDIA**, Chapter 4 (Encoding and Evolution): *"When you change a shared data
> format, all producers and consumers must be updated... Forward compatibility
> (new code reading old data) and backward compatibility (old code reading new
> data) require careful schema evolution."*

### 3. Operational Complexity

| Concern | Monolith | Microservices |
|---|---|---|
| Deployment | 1 binary | 6+ containers + message broker |
| Monitoring | 1 process | Distributed tracing required |
| Debugging | Single stack trace | Correlate traces across services |
| Testing | 37 unit tests, done | Contract tests between every service pair |
| Configuration | 1 `.env` file | 6+ config files, service discovery |
| Infrastructure | Docker Compose | Kubernetes + service mesh |

> **SRE**, Chapter 27 (Reliable Product Launches): *"The operational cost of a
> system is proportional to the number of moving parts. Each additional service
> adds monitoring, alerting, runbook pages, and on-call expertise requirements."*

### 4. The Distributed Monolith Anti-Pattern

The most common microservices failure mode: services that must be deployed
together, share a database, or can't function without each other. This gives
you all the complexity of microservices with none of the benefits.

> **BM**, Chapter 1: *"If you have a system where services are always deployed
> at the same time, or where a change to one service requires a change to
> another, you don't have microservices — you have a distributed monolith. You
> have all the downsides of both architectures."*

Gorilla Coach would be especially prone to this because the Chat Service needs
data from every other service. It would become the integration hub, tightly
coupled to everything.

---

## 9. The Modular Monolith: Best of Both Worlds?

The modular monolith recognizes that the monolith's problems are not about
deployment topology — they're about **coupling and cohesion**. You can have a
single deployment artifact with strong internal boundaries.

### Core Principles

1. **Explicit module boundaries**: Each module has a public API. Internal
   implementation details are hidden.

2. **Enforced encapsulation**: A module cannot directly access another module's
   data structures, database tables, or internal functions.

3. **Module-specific data access**: Each module owns its database tables. Other
   modules access that data only through the owning module's public API.

4. **Shared nothing between modules**: No shared mutable state, no shared
   domain types (or only shared kernel types at the boundary).

5. **Single deployment**: Despite internal boundaries, it's still one binary.

> **DDD**, Chapter 14: *"A Bounded Context defines the applicability of a
> particular model. Each Bounded Context has its own Ubiquitous Language and
> its own model, which may share names but have different meanings across
> contexts."*

> **IDDD**, Chapter 2 (Domains, Subdomains, and Bounded Contexts): *"A Bounded
> Context is explicitly scoped: it defines the range within which a particular
> model is defined and applicable. It is a concrete boundary around a set of
> domain logic."*

### In Rust: Module Visibility as Enforcement

Rust's module system is uniquely suited to the modular monolith. The `pub(crate)`
and `pub(in path)` visibility modifiers let you enforce boundaries at compile
time — not by convention, but by the type system.

```rust
// Inside biometric/repository.rs
pub(in crate::biometric) fn internal_query() { ... }  // only biometric module can call this
pub fn get_daily_summary() { ... }                     // anyone can call this
```

This is more than convention-based separation (as in Java packages or Go
packages). The Rust compiler **rejects** cross-boundary access to private items.

> **RiA**, Chapter 10 (Modules): *"Rust's privacy model ensures that internal
> implementation details are truly hidden. Unlike many languages where 'private'
> is a polite suggestion, Rust enforces it at compilation."*

---

## 10. Gorilla Coach as a Modular Monolith

Gorilla Coach has already taken key steps toward the modular monolith pattern
with its three-crate workspace split. Here's how the current architecture maps
to modular monolith principles, and where further decomposition could go.

### What's Already Modular

**1. Shared domain types in a boundary crate** — `gorilla_shared` enforces
that domain types and API contracts are defined exactly once. Neither the server
nor the client can drift from the shared schema.

**2. Repository split by domain** — Instead of one monolithic `repository.rs`,
queries are organized into submodules: `users.rs`, `garmin.rs`, `training.rs`,
`chat.rs`, `auto_reg.rs`. Each file contains only its domain's queries.

**3. Separate client crate** — The Dioxus Wasm client (`gorilla_client`) is
a completely separate compilation unit. It can only access the server through
the v2 JSON API — no backdoor imports of server internals.

**4. Reports module** — `reports/` (gatekeeper, sitrep, aar, debrief) is
cleanly separated from handlers, with its own public API.

### What Could Be Further Decomposed

The server crate still has flat module access — any handler can `use crate::*`.
A deeper restructuring would look like:

```
src/
  lib.rs                   ← declares top-level modules only
  main.rs                  ← entry point, wires modules together
  shared/
    mod.rs                 ← shared kernel: AppState, error types, config
    config.rs
    error.rs
    crypto.rs              ← Vault (used by auth and sync modules)

  auth/
    mod.rs                 ← public API: authenticate(), get_session(), create_user()
    handlers.rs            ← HTTP route handlers (login, logout, OAuth)
    domain.rs              ← User, Session types (auth-specific view)
    repository.rs          ← users table, user_settings (credentials only)

  biometric/
    mod.rs                 ← public API: get_summary(), get_range(), sync_user()
    handlers.rs            ← dashboard routes, sync endpoint
    domain.rs              ← GarminDailyData, SyncResult
    repository.rs          ← garmin_daily_data table, manual_body_metrics
    garmin.rs              ← Garmin SSO + API (internal to this module)
    sync.rs                ← background sync logic

  chat/
    mod.rs                 ← public API: get_history(), send_message(), stream()
    handlers.rs            ← chat page, SSE stream endpoint
    domain.rs              ← ChatMessage, ChatMode
    repository.rs          ← chat_messages table
    tools.rs               ← tool definitions and executor

  training/
    mod.rs                 ← public API: get_plan(), log_sets(), parse_csv()
    handlers.rs            ← training page, log endpoint
    domain.rs              ← TrainingSetLog, TrainingPlan
    repository.rs          ← training_set_logs table
    parser.rs              ← CSV parsing logic

  files/
    mod.rs                 ← public API: upload(), download(), list()
    handlers.rs            ← file page, upload/download endpoints
    storage.rs             ← file I/O abstraction

  analytics/
    mod.rs                 ← public API: analyze_metric()
    analyst.rs             ← two-stage rocket (classify + query)
    domain.rs              ← AnalystIntent, MetricStats
```

### Further Decomposition: Per-Module Repositories

The current `Repository` struct shares one `PgPool` across all domain submodules.
A deeper split would give each module its own typed data access layer:

```rust
// biometric/repository.rs
pub(in crate::biometric) struct BiometricRepo {
    pool: PgPool,
}

impl BiometricRepo {
    pub async fn upsert_daily(&self, d: &GarminDailyData) -> anyhow::Result<()> { ... }
    pub async fn get_range(&self, user_id: Uuid, start: NaiveDate, end: NaiveDate) -> ... { ... }
}
```

```rust
// chat/repository.rs
pub(in crate::chat) struct ChatRepo {
    pool: PgPool,
}

impl ChatRepo {
    pub async fn save_message(&self, user_id: Uuid, role: &str, content: &str) -> ... { ... }
    pub async fn get_history(&self, user_id: Uuid, limit: i64) -> ... { ... }
}
```

Note the `pub(in crate::biometric)` — the repo struct is only accessible within
its own module. Other modules call the public API functions, not the raw
repository.

**2. Module-specific domain types**

Instead of one `domain.rs` with everything, each module defines its own types.
The chat module doesn't need to know about `GarminDailyData`. It only needs a
`BiometricSummary` — a simplified view provided by the biometric module's public
API:

```rust
// biometric/mod.rs (public API)
pub struct BiometricSummary {
    pub date: NaiveDate,
    pub rhr: Option<i64>,
    pub hrv: Option<f64>,
    pub sleep_score: Option<i64>,
    pub body_battery_high: Option<i64>,
    // ... curated subset
}

pub async fn get_daily_summaries(
    pool: &PgPool,
    user_id: Uuid,
    start: NaiveDate,
    end: NaiveDate,
) -> anyhow::Result<Vec<BiometricSummary>> { ... }
```

This is the **Anti-Corruption Layer** pattern from DDD. The chat module works
with `BiometricSummary`, not `GarminDailyData`. If the internal Garmin data
structure changes (new fields, renamed columns), only the biometric module
needs updating.

> **DDD**, Chapter 14 (Anti-Corruption Layer): *"Create an isolating layer to
> provide clients with functionality in terms of their own domain model. The
> layer talks to the other model through its existing interface, requiring
> little or no modification to the other model."*

**3. Explicit cross-module communication**

The chat tool executor currently reaches directly into the repository:

```rust
// Current: chat handler directly queries garmin_daily_data
let rows = state.repo.get_garmin_range(user_id, start, end).await?;
```

In the modular monolith:

```rust
// Modular: chat calls biometric module's public API
let summaries = biometric::get_daily_summaries(&pool, user_id, start, end).await?;
```

The difference is significant: `get_garmin_range` returns the raw 40-field
`GarminDailyData`. `get_daily_summaries` returns a curated `BiometricSummary`.
The biometric module controls what data it exposes, and can change its internal
storage without affecting consumers.

**4. Isolated AppState**

Instead of one AppState with everything, each module receives only what it needs:

```rust
// auth module needs: cookie_key, repo (users table), config
pub struct AuthState {
    pub repo: AuthRepo,
    pub cookie_key: Key,
    pub config: AuthConfig,
}

// biometric module needs: repo (garmin tables), vault, http_client, config
pub struct BiometricState {
    pub repo: BiometricRepo,
    pub vault: Arc<Vault>,
    pub http_client: reqwest::Client,
    pub config: SyncConfig,
}

// chat module needs: repo (chat tables), llm, biometric API, training API, file API
pub struct ChatState {
    pub repo: ChatRepo,
    pub llm: Arc<dyn LlmAdapter>,
    pub biometric: BiometricApi,   // trait or function pointer
    pub training: TrainingApi,
    pub files: FileApi,
}
```

Now the compiler enforces boundaries. The auth handler **cannot** access the
LLM adapter because it's not in `AuthState`. The chat handler **cannot** write
to the Garmin table because it has `ChatRepo`, not `BiometricRepo`.

> **CA**, Chapter 22 (The Clean Architecture): *"The dependency rule states
> that source code dependencies may only point inward. Inner circles must not
> know about outer circles. This is enforced by the structure of the code, not
> by developer discipline."*

---

## 11. Bounded Contexts and Domain Boundaries

The hardest decision in any architecture is **where to draw the boundaries**.
Let's apply DDD analysis to Gorilla Coach.

### Identifying Bounded Contexts

A bounded context is a region of the codebase where a particular model is valid.
The word "set" means something completely different in the training context
(a set of reps) vs. the settings context (a configuration set).

Gorilla Coach has these natural bounded contexts:

| Bounded Context | Core Concept | Key Entities | Tables |
|---|---|---|---|
| **Identity** | Who is the user? | User, Session | `users` |
| **Biometric** | Health data from wearables | GarminDailyData, SyncResult | `garmin_daily_data`, `manual_body_metrics` |
| **Training** | Workout plans and logging | TrainingPlan, TrainingSetLog | `training_set_logs` |
| **Intelligence** | AI chat and analysis | ChatMessage, AnalystIntent | `chat_messages` |
| **Vault** | Credential management | UserSettings (encrypted) | `user_settings` |
| **Files** | User document storage | File metadata | Filesystem / S3 |

### Context Mapping

> **SaP**, Chapter 4 (Context Mapping): *"Context mapping defines the
> relationships between bounded contexts. Understanding these relationships is
> essential for making integration decisions."*

The relationships between Gorilla Coach's contexts:

```
                ┌─────────────┐
                │  Identity   │
                │  (upstream) │
                └──────┬──────┘
                       │ user_id flows downstream
          ┌────────────┼───────────────┐
          ↓            ↓               ↓
    ┌──────────┐ ┌──────────┐ ┌──────────────┐
    │Biometric │ │ Training │ │ Intelligence │
    │          │ │          │ │   (chat+AI)  │
    └────┬─────┘ └────┬─────┘ └──────┬───────┘
         │            │              │
         └────────────┼──────────────┘
                      ↓
              Intelligence consumes
              data from Biometric,
              Training, and Files
```

**Identity** is the upstream context — it provides `user_id` to every other
context. This is a **Shared Kernel** relationship.

**Intelligence** (chat/AI) is the main consumer — it needs data from Biometric
(health metrics), Training (workout logs), and Files (uploaded CSVs). This is a
**Customer-Supplier** relationship where Intelligence is the customer.

> **DDD**, Chapter 14 (Customer/Supplier Development Teams): *"When two teams
> are in an upstream-downstream relationship, the downstream team is the
> customer. The upstream team may have the power, but they have a duty to serve
> the downstream team's needs."*

### Shared Kernel: The Dangerous Middle Ground

Some types must be shared across all contexts: `user_id` (UUID), `AppError`,
and basic configuration. DDD calls this the **Shared Kernel**:

> **DDD**, Chapter 14: *"Designate some subset of the domain model that the
> two teams agree to share. This includes both code and database schema. The
> shared kernel has special status and shouldn't be changed without
> consultation with the other team."*

In Gorilla Coach, the shared kernel is small:
- `Uuid` (user identity)
- `AppError` (error handling)
- `AppConfig` (environment configuration)
- `Vault` (encryption primitives)

Keep this kernel **minimal**. The moment `GarminDailyData` enters the shared
kernel, you've lost the boundary benefits.

---

## 12. Data Ownership

### The Single Database Question

> **BM**, Chapter 4: *"The key challenge... is to keep the database from being
> a giant coupling point between services. If every service reads and writes
> to the same tables, you don't have services — you have a distributed
> monolith with extra network calls."*

In Gorilla Coach's monolith, all tables live in one PostgreSQL database. This is
fine. The question is: **within that database, who owns which tables?**

### Current State: No Ownership

Any repository method can query any table. `get_chat_context()` in the Repository
joins `chat_messages` with `garmin_daily_data`:

```rust
pub async fn get_chat_context(&self, user_id: Uuid, limit: i64)
    -> anyhow::Result<(String, Vec<ChatMessage>)>
{
    let rows = self.get_garmin_range(user_id, start, today).await?;
    // ... format biometric data
    let msgs = sqlx::query_as::<_, ChatMessage>(...)
        .fetch_all(&self.pool).await?;
    Ok((bio_ctx, msgs.into_iter().rev().collect()))
}
```

This method crosses two bounded contexts in one function: it reads from the
biometric context (`garmin_daily_data`) and the intelligence context
(`chat_messages`).

### Modular Monolith Approach: Logical Ownership

Even with one database, you can enforce ownership:

```
┌─────────────────────────────────────────────────────┐
│                   PostgreSQL                         │
│                                                     │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │ auth schema │  │ bio schema   │  │ chat schema  │ │
│  │ ────────── │  │ ──────────── │  │ ──────────── │ │
│  │ users      │  │ garmin_daily │  │ chat_messages│ │
│  │ user_creds │  │ manual_body  │  │              │ │
│  └────────────┘  └─────────────┘  └──────────────┘ │
│                                                     │
│  ┌────────────────┐  ┌────────────────┐            │
│  │ training schema │  │ analytics views│            │
│  │ ────────────── │  │ ────────────── │            │
│  │ training_logs  │  │ bio_summary_v  │            │
│  └────────────────┘  │ (read-only)    │            │
│                      └────────────────┘            │
└─────────────────────────────────────────────────────┘
```

PostgreSQL schemas enforce this at the database level. Each module's connection
role has `GRANT` permissions only on its own schema. The analytics module gets a
**read-only view** (`bio_summary_v`) of the biometric data — it can't write to
`garmin_daily_data`.

```sql
-- Create schemas
CREATE SCHEMA IF NOT EXISTS auth;
CREATE SCHEMA IF NOT EXISTS bio;
CREATE SCHEMA IF NOT EXISTS chat;
CREATE SCHEMA IF NOT EXISTS training;

-- Create roles
CREATE ROLE auth_role;
CREATE ROLE bio_role;
CREATE ROLE chat_role;

-- Grant permissions
GRANT ALL ON SCHEMA auth TO auth_role;
GRANT ALL ON SCHEMA bio TO bio_role;
GRANT ALL ON SCHEMA chat TO chat_role;

-- Read-only cross-schema view for the chat module
CREATE VIEW chat.bio_summary AS
    SELECT user_id, date, resting_heart_rate, hrv_last_night, sleep_score
    FROM bio.garmin_daily_data;
GRANT SELECT ON chat.bio_summary TO chat_role;
```

This is stronger than code-level enforcement: even if a bug in the chat module
constructs a raw SQL query against `bio.garmin_daily_data`, PostgreSQL rejects it.

> **DDIA**, Chapter 4 (Encoding and Evolution): *"Schema evolution at the
> database level can be managed independently for each bounded context, allowing
> one context to evolve its schema without coordinating with others."*

---

## 13. The Dependency Graph

### Current Dependency Graph

Let's trace the actual imports in Gorilla Coach:

```
main.rs
  ├── config::AppConfig
  ├── handlers::* (all handlers)
  ├── middleware::{csrf_middleware, RateLimiter}
  ├── repository::Repository
  ├── state::{AppState, Metrics, LlmLogger}
  ├── vault::Vault
  └── llm::{LlmAdapter, GeminiAdapter, OllamaAdapter, FallbackLlmAdapter}

handlers/chat.rs
  ├── error::AppError
  ├── llm::LlmAdapter
  ├── state::AppState
  ├── ui
  ├── handlers::{get_session, sanitize_filename, user_uploads_dir}
  └── llm::{ToolDef, ToolExecutor, detect_intent, get_metric_stats, ALLOWED_METRICS}

handlers/settings.rs
  ├── domain::UserSettings
  ├── garmin (direct call to garmin_login)
  ├── state::AppState
  ├── ui
  └── handlers::get_session

handlers/sync.rs
  ├── garmin (direct calls to login, refresh, fetch)
  ├── state::AppState
  ├── vault::Vault
  ├── domain::UserSettings
  └── handlers::perform_user_sync (called from main.rs)

handlers/dashboard.rs
  ├── state::AppState
  ├── ui
  └── handlers::get_session

llm/analyst.rs
  ├── domain::{AnalystIntent, GarminDailyData}
  ├── error::AppError
  ├── llm::LlmAdapter
  └── repository::Repository
```

### The Problem: Everything Depends on Everything

The dependency graph reveals a hub-and-spoke pattern centered on `AppState`,
`domain`, and `repository`. Almost every module depends on these three, creating a
de facto coupling layer.

```
         ┌─────────┐
     ┌───│ domain  │───┐
     │   └─────────┘   │
     ↓        ↑        ↓
┌─────────┐   │   ┌──────────┐
│handlers │───┼───│repository│
└────┬────┘   │   └──────────┘
     │        │        ↑
     ↓        │        │
┌─────────┐   │   ┌────────┐
│  garmin │   └───│  llm   │
└─────────┘       └────────┘
     ↑                ↑
     │                │
     └────────────────┘
      (indirect through
       handlers/sync)
```

> **CA**, Chapter 14 (Component Coupling): *"When you have a cluster of
> components that are mutually dependent, they are effectively one component.
> They must be released together, tested together, and changed together."*

### Modular Monolith: Acyclic Dependencies

The Acyclic Dependencies Principle (ADP) states that the dependency graph should
be a **directed acyclic graph (DAG)** — no cycles.

```
                ┌────────┐
                │ shared │  (config, error, crypto)
                └───┬────┘
          ┌─────────┼──────────┬──────────┐
          ↓         ↓          ↓          ↓
     ┌────────┐ ┌────────┐ ┌────────┐ ┌──────┐
     │  auth  │ │  bio   │ │training│ │files │
     └────┬───┘ └───┬────┘ └───┬────┘ └──┬───┘
          │         │          │          │
          └─────────┼──────────┼──────────┘
                    ↓          ↓
              ┌──────────────────┐
              │  intelligence    │  (chat + analytics)
              │  (depends on     │
              │   bio, training, │
              │   files APIs)    │
              └──────────────────┘
```

Dependencies flow strictly downward. `auth` never calls `intelligence`.
`bio` never calls `training`. The `intelligence` module depends on the public
APIs of `bio`, `training`, and `files` — but not on their internals.

> **PP**, Chapter 36 (Architecture): *"Good architecture is about drawing
> boundaries — lines between things that matter and things that don't. The art
> is deciding where those lines should go."*

---

## 14. Communication Patterns

The three architectures use fundamentally different communication mechanisms,
each with distinct trade-offs.

### Monolith: Function Calls

```rust
// Direct function call — nanoseconds, always succeeds (no network)
let data = state.repo.get_garmin_range(user_id, start, end).await?;
```

**Latency**: ~1µs (in-memory) to ~1ms (database query)
**Failure modes**: Only database errors
**Consistency**: ACID transactions across all operations

### Microservices: Network Calls

```rust
// HTTP call to another service — milliseconds, can fail ¹⁰⁰ ways
let response = http_client.get("http://biometric-service/api/range")
    .query(&[("user_id", user_id), ("start", start), ("end", end)])
    .send().await?;
let data: Vec<BiometricSummary> = response.json().await?;
```

**Latency**: ~10-50ms (network + serialization + deserialization)
**Failure modes**: Network partition, timeout, 500, 502, 503, 429, DNS failure,
serialization error, schema mismatch, circuit breaker open...
**Consistency**: Eventual (at best)

> **DDIA**, Chapter 8 (The Trouble with Distributed Systems): *"Anything that
> can happen, will happen. The network is unreliable, clocks are unreliable,
> and processes can pause at any time. Building reliable systems from
> unreliable components requires explicit handling of every failure mode."*

### Modular Monolith: Trait-Based APIs

```rust
// Call through a module's public trait — nanoseconds, type-safe
let data = biometric::get_daily_summaries(&self.bio_api, user_id, start, end).await?;
```

**Latency**: Same as monolith (in-process)
**Failure modes**: Same as monolith (no network)
**Consistency**: Same as monolith (shared database, transactions available)
**Boundary enforcement**: Compiler-enforced via `pub(in crate::module)` visibility

This is the key insight of the modular monolith: **you get the encapsulation
benefits of microservices with the performance and consistency of a monolith.**

> **BM**, Chapter 3: *"Start with a monolith, and extract services only when
> you have a good reason... A critical benefit of modular monoliths is that you
> can define module boundaries without paying the operational cost of
> distributed systems."*

### Async Communication: Events Within a Monolith

Even within a monolith, you can use events for decoupling:

```rust
// Event bus (in-process, using tokio broadcast channel)
pub enum DomainEvent {
    GarminSynced { user_id: Uuid, days_synced: i32 },
    ChatMessageSent { user_id: Uuid },
    TrainingLogged { user_id: Uuid, exercise: String },
}

// Biometric module publishes after sync
event_bus.send(DomainEvent::GarminSynced { user_id, days_synced: 5 })?;

// Analytics module listens and updates summaries
while let Ok(event) = event_rx.recv().await {
    match event {
        DomainEvent::GarminSynced { user_id, .. } => {
            analytics::refresh_cache(user_id).await;
        }
        _ => {}
    }
}
```

This is **event-driven architecture within a single process** — no Kafka, no
RabbitMQ, no network. Just tokio channels.

> **IDDD**, Chapter 8 (Domain Events): *"Domain events are a way of capturing
> something that happened in the domain that domain experts care about. They
> enable decoupled communication between bounded contexts."*

---

## 15. Testing Strategies

### Monolith Testing (Current State)

Gorilla Coach has 37 tests:

```
tests/integration_tests.rs  — 14 tests (domain, vault, metrics, date parsing)
src/ (inline tests)         — 23 tests (CSV parsers, CSRF middleware, AI analyst)
```

These run with `cargo test`, no database required. They test individual functions
and modules in isolation.

**Strength**: Dead simple. One command, no infrastructure.
**Weakness**: No tests for cross-module interactions (e.g., "does a Garmin sync
→ dashboard render correctly?"). At the monolith scale, you'd add integration
tests that spin up the full Axum app and hit it with HTTP requests.

### Microservices Testing

> **MP**, Chapter 9 (Testing Microservices): *"The test pyramid for
> microservices has an additional layer: contract tests, which verify that
> service boundaries remain compatible without requiring both services to be
> running."*

```
                   ┌─────────────┐
                   │   E2E Tests │  ← slowest, most brittle
                  ┌┴─────────────┴┐
                  │ Contract Tests │ ← verify API compatibility
                 ┌┴───────────────┴┐
                 │ Integration Tests│ ← test with real DB, mocked services
                ┌┴─────────────────┴┐
                │    Unit Tests      │ ← fastest, most isolated
                └────────────────────┘
```

For Gorilla Coach's 6 services, you'd need:
- Unit tests for each service (still fast)
- Integration tests for each service with its own DB
- **Contract tests** between every service pair: Auth↔Bio, Bio↔Chat,
  Chat↔Training, Chat↔Files, Chat↔Analytics — at least 10 contract test suites
- E2E tests that spin up all 6 services + databases + message broker

This is an order of magnitude more testing infrastructure than the monolith.

> **BM**, Chapter 8 (Testing): *"End-to-end tests across microservices
> are expensive to write, slow to run, and brittle. They should be the tip of
> the pyramid, not the base."*

### Modular Monolith Testing

The modular monolith gets the best of both:

```
                   ┌─────────────┐
                   │   E2E Tests │ ← test full HTTP stack
                  ┌┴─────────────┴┐
                  │ Module Integ.  │ ← test module interactions via public APIs
                 ┌┴───────────────┴┐
                 │  Module Unit     │ ← test each module in isolation
                ┌┴─────────────────┴┐
                │    Function Unit   │ ← test individual functions
                └────────────────────┘
```

No contract tests needed (it's all in-process). Module integration tests verify
that the biometric module's public API works correctly when called from the
chat module — but it's all in one test binary with one `cargo test`.

```rust
#[cfg(test)]
mod tests {
    use crate::biometric;
    use crate::chat;

    #[tokio::test]
    async fn chat_can_read_biometric_summary() {
        let pool = test_db_pool().await;
        // Insert test data via biometric module
        biometric::insert_test_data(&pool, user_id, date, ...).await;
        // Verify chat module can read it through the public API
        let summaries = biometric::get_daily_summaries(&pool, user_id, start, end).await;
        assert_eq!(summaries.len(), 1);
    }
}
```

> **ZtP**, Chapter 3 (Testing): *"Integration tests should test the behavior
> of your system at its boundaries. For a web application, that means sending
> HTTP requests and verifying responses — not testing internal implementation
> details."*

---

## 16. Deployment and Operations

### Monolith Deployment (Current)

```bash
# Build
cargo build --release

# Deploy (on a single server)
scp target/release/gorilla_coach server:/opt/gorilla/
ssh server 'systemctl restart gorilla-coach'

# Or with Docker
docker compose up -d --build
```

**Remote access via Tailscale**: The server is reached over a WireGuard-based
mesh VPN (tailnet). `tailscale serve` proxies HTTPS traffic to `localhost:3000`
with automatic Let's Encrypt TLS certificates — no reverse proxy or manual cert
management needed. Only devices on the tailnet can reach the server.

```
Phone / Laptop (any network)
  ↓ WireGuard tunnel (Tailscale)
https://<machine>.ts.net → tailscale serve → localhost:3000 (Axum)
```

**Artifacts**: 1 binary (~20 MB), 1 Docker image
**Infrastructure**: 1 server, 1 PostgreSQL instance, Tailscale for networking + TLS
**Monitoring**: 1 process to watch
**Rollback**: `systemctl restart gorilla-coach` with previous binary

> **USAH**, Chapter 29: *"Simplicity in deployment is not a luxury; it is a
> prerequisite for reliability. Every additional step in a deployment process is
> a potential point of failure."*

### Microservices Deployment

```yaml
# 6 services, each independently deployed
services:
  auth-service:
    image: gorilla/auth:v1.2.3
  biometric-service:
    image: gorilla/biometric:v2.0.1
  chat-service:
    image: gorilla/chat:v1.8.0
  training-service:
    image: gorilla/training:v1.1.0
  file-service:
    image: gorilla/files:v1.0.5
  analytics-service:
    image: gorilla/analytics:v1.3.2
  # Plus infrastructure
  api-gateway:
    image: envoy:latest
  message-broker:
    image: rabbitmq:3-management
  # Plus 3-6 separate databases
```

**Artifacts**: 6+ Docker images, each versioned independently
**Infrastructure**: Kubernetes cluster, container registry, CI/CD pipelines ×6
**Monitoring**: Distributed tracing (Jaeger/Tempo), per-service metrics, log
aggregation across all services
**Rollback**: Roll back one service — but verify contract compatibility with
all dependent services first

> **SRE**, Chapter 8 (Release Engineering): *"Release engineering is a
> discipline of its own. For microservices, the release process must ensure that
> every combination of service versions is compatible."*

### Modular Monolith Deployment

Identical to the monolith:

```bash
cargo build --release
docker compose up -d --build
```

Same single binary. Same simple deployment. But internally, the code is organized
so that if you ever *need* to extract a service, the module boundaries are
already defined. You're not paying for microservices complexity today, but you're
not painting yourself into a corner either.

This is the essence of **evolutionary architecture**:

> **RDD**, Chapter 1: *"An evolutionary architecture supports guided,
> incremental change across multiple dimensions... The architecture should make
> it easy to change, not just easy to build."*

---

## 17. The Migration Path

If Gorilla Coach grows to the point where a monolith becomes constraining, the
modular monolith provides a clean extraction path.

### Phase 1: Current State (Monolith)

```
[One Binary] → [One Database]
```

Works for: 1 developer, < 10K users, evolving domain.

### Phase 2: Modular Monolith (Internal Boundaries)

Refactor the codebase into modules with explicit boundaries. Still one binary,
still one database, but with logical separation:

```
[One Binary]
  ├── auth module      ──→  users, user_settings tables
  ├── biometric module ──→  garmin_daily_data tables
  ├── chat module      ──→  chat_messages table
  ├── training module  ──→  training_set_logs table
  ├── files module     ──→  filesystem / S3
  └── analytics module ──→  read-only views

→ [One Database with schema-level ownership]
```

**Effort**: Days to weeks. No infrastructure change. Just code reorganization.
**Benefit**: Clearer ownership, enforceable boundaries, easier to test, easier to
onboard new developers.

### Phase 3: Selective Extraction (The First Service)

When a specific module reaches a scale or velocity that justifies extraction,
pull it out. The most likely candidate for Gorilla Coach is the **biometric
sync worker**:

```
[Web Binary]                    [Sync Worker Binary]
  ├── auth module               ├── biometric module
  ├── chat module                   (garmin.rs, sync logic)
  ├── training module           └── writes to garmin_daily_data
  ├── files module
  └── analytics module
      (reads garmin_daily_data)

         ↕                              ↕
    [Shared Database]  ←──────→  [Shared Database]
```

This isn't even a microservice yet — it's just a second binary reading/writing
the same database. But the sync worker can now:
- Run on a separate machine with more CPU/network
- Be scaled independently (3 sync workers for 100K users)
- Be deployed independently (fix a Garmin SSO change without restarting the web)
- Crash without taking down the web server

> **BM**, Chapter 3: *"The safest way to extract a service is to first create
> a module boundary within the monolith, verify it works, and then physically
> separate the module into its own process."*

### Phase 4: True Microservice (Full Separation)

Only if needed — years down the line, with a team of developers and hundreds of
thousands of users:

```
[Auth Service] ──→ [Auth DB]
[Bio Service]  ──→ [Bio DB]   ← separate database
[Chat Service] ──→ [Chat DB]
   ↕ events via message broker ↕
```

By this point, the module boundaries defined in Phase 2 have been battle-tested
in production for months or years. The extraction is mechanical — replace
in-process function calls with HTTP/gRPC calls, replace shared-database access
with API calls, add contract tests.

> **DDD**, Chapter 14: *"Let experience guide your decisions about context
> boundaries. Premature splitting creates unnecessary complexity; delayed
> splitting creates an entangled mess. The modular monolith lets you defer the
> decision while keeping the option valuable."*

---

## 18. Decision Framework

### When to Use Each Architecture

| Factor | Monolith | Modular Monolith | Microservices |
|---|---|---|---|
| Team size | 1-5 devs | 3-15 devs | 10+ devs, multiple teams |
| Domain maturity | Exploring/evolving | Stable core, evolving edges | Well-understood, stable |
| Scale requirements | Single server sufficient | Single server or few instances | Independent scaling critical |
| Deployment frequency | Weekly-monthly | Daily-weekly per module | Hourly per service |
| Data consistency | ACID required | ACID within modules, eventual across | Eventual acceptable |
| Operational expertise | Minimal DevOps | Moderate DevOps | Dedicated platform team |
| Time to market | Fastest | Fast | Slowest (initial setup) |

### The Gorilla Coach Verdict

Gorilla Coach is correctly positioned as a **well-structured monolith trending
toward a modular monolith**. The evidence:

**Already modular**:
- Handlers are split by domain (`handlers/{auth,calendar,chat,dashboard,files,settings,sheets,sync,training}`)
- LLM is behind a trait (`LlmAdapter`) with pluggable implementations
- Middleware is isolated (`middleware/{csrf,rate_limit}`)
- Error types are domain-tagged (`AppError::Llm`, `AppError::Garmin`)

**Not yet a modular monolith**:
- All domain types in one `domain.rs`
- All data access in one `repository.rs`
- Everything receives the full `AppState`
- No enforcement of module boundaries (any handler can call any repo method)
- `garmin.rs` is called directly from `handlers/settings.rs` and `handlers/sync.rs`
  (no abstraction boundary)

### Rules of Thumb

> **BM**, Chapter 3: *"If you can't build a well-structured monolith, what
> makes you think you can build a set of well-structured microservices?"*

1. **Start with a monolith** when you're exploring a domain. Gorilla Coach did
   this correctly.

2. **Move to a modular monolith** when the codebase grows large enough that
   developers step on each other, or when different parts of the system evolve
   at different rates. This is Gorilla Coach's next natural step.

3. **Extract a service** when you have a concrete, measurable need:
   - A module needs to scale independently (the sync worker)
   - A module needs a different technology (ML model in Python)
   - A module needs a different deployment cadence
   - A separate team owns the module end-to-end

4. **Never go full microservices** unless you have the team and operational
   maturity to support it. Most applications never need it. The ones that do
   typically have hundreds of developers.

> **PP**, Chapter 36: *"Don't outrun your headlights. You can always take
> a larger step when you know more later. An incremental approach allows you
> to change direction when you learn something new."*

---

## Day-2 Operations: Living With the Monolith

The sections above describe **design decisions**. This section covers the
operational reality — what you actually do on a Tuesday afternoon when
something breaks.

### Startup Verification

After deploying a new binary, verify the architecture is healthy:

```bash
# Check the startup log for the full initialization sequence
RUST_LOG=gorilla_coach=debug cargo run -- server 2>&1 | head -30
```

Expected startup order:
1. `AppConfig::from_env()` — logs config summary, Google OAuth status, Gemini models
2. `Repository::new()` — Postgres pool connects
3. `repo.migrate()` — auto-runs pending migrations
4. LLM adapter construction — logs primary/fallback providers
5. Rate limiters created (auth: 10/60s, uploads: 30/60s)
6. Middleware stack wired (compression → trace → trace-id → security headers → CSRF)
7. Background sync worker spawned (30s initial delay, then hourly)
8. `"🦍 GORILLA COACH ONLINE"` — server is accepting connections

If the server doesn't reach step 8, check:
- **Step 2 fails**: `DATABASE_URL` is wrong or Postgres is down
- **Step 3 fails**: Migration SQL has an error (check `_sqlx_migrations` table)
- **Step 4 fails**: `GEMINI_API_KEY` missing and no Ollama running

### Module Boundary Health Checks

The monolith's internal boundaries (handlers → repository → domain) should
never leak. Signs of boundary violations in production:

- **SQL in handler logs**: If you see raw SQL in handler-level tracing, a
  repository method is logging at the wrong level
- **HTML in non-handler code**: If `ui/` functions are called from `garmin/`
  or `repository/`, the rendering layer is leaking into the data layer.
  v2 handlers return JSON only; HTML generation is limited to legacy v1 code.
- **Direct pool access**: `AppState` exposes `repo` (the Repository), never
  the raw `PgPool`. If a handler constructs its own SQL, it's bypassing the
  repository boundary

### Tracing and Debugging

Every HTTP request gets an `X-Trace-Id` header (UUID) via middleware. To
trace a specific request through the monolith:

```bash
# Find all log lines for a specific request
RUST_LOG=gorilla_coach=debug cargo run -- server 2>&1 | grep "<trace-id>"
```

Log levels by module:
- `gorilla_coach=debug` — all application logging
- `gorilla_coach::garmin=debug` — Garmin API details (response keys, parsed values)
- `gorilla_coach::llm=debug` — LLM requests, tool calls, fallback decisions
- `gorilla_coach::handlers=debug` — handler entry/exit, form parsing

### Graceful Shutdown

The server handles `SIGTERM` and `Ctrl+C`:

```
2026-02-21T03:58:03Z  INFO gorilla_coach: SIGTERM received, shutting down
```

Axum's graceful shutdown drains in-flight requests. The background sync worker
is a `tokio::spawn` task that will be dropped when the runtime shuts down. If
a sync is mid-flight, it will be interrupted — this is safe because each day's
upsert is an independent atomic operation.

### Common Operational Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| Server won't start | `DATABASE_URL` wrong, Postgres down | Check `docker ps`, verify connection string |
| 403 on all POST requests | CSRF mismatch (Host vs Origin) | Check Tailscale proxy headers, look for `CSRF:` in logs |
| 429 responses | Rate limiter triggered | Wait 60s, or check if a bot is hitting auth endpoints |
| Blank dashboard | No Garmin data synced yet | Check Settings → Garmin credentials, trigger manual sync |
| Chat not responding | LLM unreachable | Check `/status` page for LLM connection status |
| MFA prompt from Garmin | Background sync re-login attempt | Clear stale Garmin credentials in DB (see DATABASE_MAINTENANCE.md) |

---

## 19. Further Reading

### Architecture & Patterns

- **Building Microservices** — Sam Newman, 2nd Ed. (2021).
  *The definitive work on microservices. Critically, Newman spends significant
  time on when NOT to use microservices. Chapter 3 ("Splitting the Monolith")
  is the most practical guide to monolith-to-services migration. Read this
  before making any architecture decision.*

- **Microservices Patterns** — Chris Richardson (2018).
  *Where Newman explains the why, Richardson explains the how. The Saga
  pattern, API Gateway pattern, and Database per Service pattern are essential
  knowledge. The decision flowcharts at the start of each chapter are
  invaluable.*

- **Clean Architecture** — Robert C. Martin (2017).
  *The Dependency Rule and the importance of stable abstractions at the core
  of your system. Chapter 16 on "Independence" makes the case for deferred
  architectural decisions. Directly applicable to the modular monolith
  approach.*

- **Patterns of Enterprise Application Architecture** — Martin Fowler (2002).
  *The original patterns book for server-side applications. Transaction
  Script, Domain Model, Data Mapper, Repository — the vocabulary of enterprise
  architecture. Dated in some technology specifics, timeless in its pattern
  language.*

- **Building Evolutionary Architectures** — Ford, Parsons & Kua, 2nd Ed. (2023).
  *How to design systems that evolve gracefully over time. Introduces
  "fitness functions" for measuring architectural health. Directly relevant
  to the question of when to extract services.*

### Domain-Driven Design

- **Domain-Driven Design** — Eric Evans (2003).
  *The foundational text. Bounded Contexts, Ubiquitous Language, Context
  Mapping, Anti-Corruption Layers — every concept in Sections 11 and 12 of
  this tutorial comes from Evans. Dense but essential. Read Part IV
  (Strategic Design) first.*

- **Implementing Domain-Driven Design** — Vaughn Vernon (2013).
  *The practical companion to Evans. Where Evans is philosophical, Vernon is
  concrete. The chapters on Bounded Contexts and Context Mapping have
  step-by-step guidance for identifying and implementing boundaries.*

- **Learning Domain-Driven Design** — Vlad Khononov (2021).
  *The most accessible modern introduction to DDD. Clearer than Evans,
  more updated than Vernon. The chapter on Context Mapping patterns is
  excellent and directly applicable to analyzing Gorilla Coach's module
  relationships.*

### Rust-Specific

- **Zero To Production in Rust** — Luca Palmieri (2022).
  *Building a production web service in Rust. Covers the exact stack
  decisions Gorilla Coach faces: Actix vs Axum, sqlx, configuration,
  telemetry, error handling. The testing chapters are particularly
  relevant for modular monolith testing strategies.*

- **Rust in Action** — Tim McNamara (2021).
  *Systems-level Rust. The module system chapter explains how Rust's
  visibility rules can enforce architectural boundaries at compile time —
  a capability that makes Rust uniquely suited for modular monoliths.*

- **Programming Rust** — Blandy, Orendorff & Tindall, 2nd Ed. (2021).
  *Deep dive into Rust's type system and ownership model. The trait
  chapters explain why `LlmAdapter` as a trait gives you the same
  interface abstraction as a microservice boundary, without the network.*

### Systems & Operations

- **Designing Data-Intensive Applications** — Martin Kleppmann (2017).
  *Chapter 4 (Encoding and Evolution) on schema compatibility across
  services. Chapter 8 (Distributed Systems) on everything that goes wrong
  when you split a monolith. Chapter 12 on the future of data systems.
  Required reading for any architecture decision.*

- **Site Reliability Engineering** — Google (2016).
  *Operational reality of running large systems. Chapters 21-22 on
  cascading failures and overload handling are essential context for
  understanding the operational cost of microservices. Chapter 8 on
  release engineering explains why simple deployment matters.*

- **UNIX and Linux System Administration Handbook** — Nemeth et al., 5th Ed.
  (2017). *The operational bedrock. Chapter 29 on containers and
  virtualization provides context for deployment topology decisions.
  Chapter 12 on system monitoring and logging applies regardless of
  architecture. The principle "simplicity is a prerequisite for reliability"
  runs throughout.*

- **The Pragmatic Programmer** — Hunt & Thomas, 20th Anniversary Ed. (2019).
  *Chapter 28 on decoupling and Chapter 36 on architecture provide timeless
  design principles. The concepts of "Don't Repeat Yourself," "Tell, Don't
  Ask," and "Design for Reversibility" are the micro-level complement to the
  macro-level patterns in this tutorial.*

- **A Philosophy of Software Design** — John Ousterhout, 2nd Ed. (2021).
  *The case for deep modules with simple interfaces. Ousterhout argues
  that microservices often create shallow modules (thin wrappers around
  CRUD operations) rather than deep modules (significant functionality behind
  a simple interface). Directly relevant to the modular monolith's emphasis
  on substantial modules with clean public APIs.*

---

## Closing Note

Architecture is not a destination — it's a series of decisions made over time.
The best architectures are the ones that defer decisions until the last
responsible moment, make those decisions reversible where possible, and optimize
for the current reality rather than a hypothetical future.

Gorilla Coach's monolith is not a limitation to be fixed. It's a **rational
choice** for a self-hosted fitness coaching app with a small user base and an
evolving domain. The path forward — adding internal module boundaries where the
cost of coupling exceeds the cost of separation — is precisely what the
literature recommends.

> *"Architect for the system you have, not the system you imagine."*
> — Mary Poppendieck, *Lean Software Development*

> *"A complex system that works is invariably found to have evolved from a
> simple system that worked."*
> — John Gall, *Systemantics* (1975)
