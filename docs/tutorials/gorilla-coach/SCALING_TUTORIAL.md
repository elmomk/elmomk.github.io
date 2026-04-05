# Scaling Gorilla Coach: 100 Users to 1,000,000 Users

> *"Premature optimization is the root of all evil — but so is premature neglect of architecture."*
> — Adapted from Donald Knuth, *The Art of Computer Programming*

This tutorial is a deep-dive into what changes between running Gorilla Coach for
100 users on a single VPS and running it for 1,000,000 users across a fleet of
machines. Every section references actual code in this repository, explains **why**
the current design works at small scale, and lays out **exactly** what breaks and
how to fix it at large scale.

We reference several foundational texts throughout:

| Abbreviation | Book |
|---|---|
| **USAH** | Nemeth, Snyder, Hein, Whaley & Mackin — *UNIX and Linux System Administration Handbook*, 5th Ed. (2017) |
| **DDIA** | Martin Kleppmann — *Designing Data-Intensive Applications* (2017) |
| **SRE** | Beyer, Jones, Petoff & Murphy — *Site Reliability Engineering: How Google Runs Production Systems* (2016) |
| **HPMS** | Baron Schwartz, Peter Zaitsev & Vadim Tkachenko — *High Performance MySQL*, 4th Ed. (applies to PostgreSQL patterns too) (2021) |
| **RiA** | Tim McNamara — *Rust in Action* (2021) |
| **BPE** | Brendan Gregg — *Systems Performance: Enterprise and the Cloud*, 2nd Ed. (2020) |
| **TAoEP** | Werner Vogels (various) — *The Architecture of Every Platform* (AWS blog series) |
| **PiP** | *PostgreSQL Internals: A Deep Dive* — Egor Rogov (2024) |
| **CaP** | Christof Strauch — *NoSQL Databases* + Eric Brewer — CAP Theorem papers (2000–2012) |

---

## Table of Contents

1. [The Two Worlds: Mental Model](#1-the-two-worlds-mental-model)
2. [Database: From Single Pool to Distributed Data Tier](#2-database)
3. [Connection Management](#3-connection-management)
4. [Indexing Strategy](#4-indexing-strategy)
5. [Data Volume & Partitioning](#5-data-volume--partitioning)
6. [Read Scaling & Replication](#6-read-scaling--replication)
7. [Background Sync: From Serial Loop to Job Queue](#7-background-sync)
8. [Rate Limiting: From In-Memory to Distributed](#8-rate-limiting)
9. [Encryption at Scale](#9-encryption-at-scale)
10. [LLM Integration Under Load](#10-llm-integration-under-load)
11. [Session & Authentication Scaling](#11-session--authentication-scaling)
12. [File Storage: From Local Disk to Object Storage](#12-file-storage)
13. [Observability: Metrics, Logging, Tracing](#13-observability)
14. [Server Rendering & Caching](#14-server-rendering--caching)
15. [Horizontal Scaling & Deployment](#15-horizontal-scaling--deployment)
16. [Garmin API: The External Dependency Problem](#16-garmin-api)
17. [Security at Scale](#17-security-at-scale)
18. [The Migration Path: Phased Approach](#18-the-migration-path)
19. [Further Reading](#19-further-reading)

---

## 1. The Two Worlds: Mental Model

At **100 users**, your system looks like this:

```
[100 browsers] → [1 Axum process] → [1 PostgreSQL instance]
                       ↕                      ↕
               [local uploads/]        [20 connections]
               [in-memory state]       [~36,500 garmin rows/yr]
```

At **1,000,000 users**, it needs to look like this:

```
[1M browsers] → [CDN/LB] → [N Axum instances] → [PgBouncer cluster]
                                    ↕                     ↕
                            [Redis cluster]      [PG primary + N replicas]
                            [S3/MinIO]           [partitioned tables]
                            [Job queue]          [365M garmin rows/yr]
                            [Prometheus]
```

The fundamental shift is from **vertical** (one bigger machine) to **horizontal**
(many machines coordinating). Every piece of in-process state becomes a
distributed systems problem.

> **DDIA**, Chapter 1: *"An application has to meet various requirements... If the
> data volume, read/write/query load, or complexity grows, you need to find ways
> of splitting the load across multiple machines."*

---

## 2. Database

### Current: `gorilla_server/src/repository/mod.rs`

```rust
let pool = PgPoolOptions::new()
    .max_connections(20)     // ← hard-coded ceiling
    .connect(url)
    .await?;
```

This creates a single connection pool with 20 connections. At 100 users this is
generous — each web request holds a connection for microseconds, so 20
connections can serve hundreds of concurrent requests.

### At 1M Users: Why It Breaks

PostgreSQL has a **per-connection cost**: each connection is a Linux process
consuming ~5-10 MB of RAM. The PG default `max_connections` is 100. Even if you
raise it to 1000, connection overhead starts competing with shared buffers for
memory.

> **HPMS**, Chapter 5 (Connection Management): *"Each connection to the database
> consumes memory and CPU resources... Using a connection pooler is essential at
> scale."*

With multiple Axum instances (say 10), each running a pool of 100 connections,
you'd need 1000 simultaneous PG connections. This is **expensive**. The solution
is a **connection pooler** that sits between your app and PostgreSQL.

### The Fix: PgBouncer

PgBouncer maintains a small number of actual PostgreSQL connections and
multiplexes hundreds of application connections onto them:

```
[Axum instance 1: 100 conns] ──┐
[Axum instance 2: 100 conns] ──┤── [PgBouncer: 50 real PG conns] ── [PostgreSQL]
[Axum instance N: 100 conns] ──┘
```

Configuration (`pgbouncer.ini`):
```ini
[databases]
gorilla_hq = host=pg-primary port=5432 dbname=gorilla_hq

[pgbouncer]
pool_mode = transaction          ; release connection after each txn
max_client_conn = 10000          ; total app connections allowed
default_pool_size = 50           ; actual PG connections per database
reserve_pool_size = 10           ; emergency overflow
reserve_pool_timeout = 3         ; seconds before using reserve
server_idle_timeout = 300        ; close idle PG connections
```

**Transaction mode** is critical: it returns the PG connection to the pool after
each SQL statement/transaction completes, rather than holding it for the lifetime
of the client connection. This gives you 10,000:50 multiplexing.

> **USAH**, Chapter 24 (Databases): *"Connection pooling is not optional for
> production database deployments. The overhead of establishing and tearing down
> connections dwarfs the cost of the queries themselves at scale."*

In `docker-compose.yaml`, add:

```yaml
pgbouncer:
  image: edoburu/pgbouncer:latest
  environment:
    DATABASE_URL: postgres://gorilla:tactical_pass@localhost:5432/gorilla_hq
    POOL_MODE: transaction
    MAX_CLIENT_CONN: 10000
    DEFAULT_POOL_SIZE: 50
  ports:
    - "6432:6432"
  depends_on:
    - gorilla-db
```

Then point `DATABASE_URL` in your `.env` to PgBouncer (`port 6432`) instead of
PG directly. The `Repository::new()` code doesn't change at all — sqlx connects
to PgBouncer as if it were PostgreSQL.

---

## 3. Connection Management

### App-Side Pool Tuning

Even with PgBouncer, your sqlx pool settings matter. The pool should be large
enough to avoid application-level queuing but not so large that it overwhelms
PgBouncer:

```rust
let pool = PgPoolOptions::new()
    .max_connections(100)              // per-instance (not total)
    .min_connections(10)               // pre-warm connections
    .acquire_timeout(Duration::from_secs(5))  // fail fast, don't hang
    .idle_timeout(Duration::from_secs(300))
    .max_lifetime(Duration::from_secs(1800))  // recycle connections
    .connect(url)
    .await?;
```

Make `max_connections` configurable via environment variable:

```rust
let max_conns: u32 = std::env::var("DB_MAX_CONNECTIONS")
    .ok()
    .and_then(|v| v.parse().ok())
    .unwrap_or(20);
```

### The Math

With 10 Axum instances × 100 connections = 1000 client connections to PgBouncer.
PgBouncer holds 50 real PG connections in transaction mode. Average query time is
~1ms. So each real connection can serve ~1000 queries/second. That's 50 × 1000 =
**50,000 queries/second** — more than enough for 1M users with typical fitness-app
access patterns (most users check the app 1-5 times/day).

> **PiP**, Chapter 3 (Buffer Management): *"PostgreSQL's shared_buffers should be
> set to 25% of available RAM... Each backend process and connection has its own
> work_mem allocation, which is why limiting connections is critical."*

---

## 4. Indexing Strategy

### Current State

The migration files create tables with only primary keys:

- `users(id)` — UUID PK
- `user_settings(user_id)` — UUID PK, FK to users
- `garmin_daily_data(user_id, date)` — composite PK
- `chat_messages` — **no primary key, no indexes at all**
- `training_set_logs(user_id, plan_file, day_label, exercise_name, set_number)` — composite PK

### At 100 Users

The table sizes are tiny. PostgreSQL keeps them entirely in memory (`shared_buffers`).
Sequential scans are fast because there's nothing to scan — maybe a few thousand
rows total. The query planner often *prefers* sequential scans on small tables
because the overhead of an index lookup isn't worth it.

### At 1M Users: Sequential Scans Kill You

`chat_messages` is the most dangerous table. With 1M users averaging 200 messages
each, that's **200 million rows**. The `get_chat_history` query:

```sql
SELECT role, content FROM chat_messages
WHERE user_id = $1 ORDER BY time DESC LIMIT $2
```

Without an index, PostgreSQL must scan all 200M rows to find the ones matching a
specific `user_id`, then sort by `time`. This goes from being instant to taking
**minutes**.

### Required Indexes

Create a new migration `2026022100_scaling_indexes.sql`:

```sql
-- chat_messages: the most-queried table per request
-- Covering index: the query only needs role, content, so we INCLUDE them
-- to enable index-only scans (no heap lookup needed).
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_chat_messages_user_time
    ON chat_messages (user_id, time DESC);

-- garmin_daily_data: range queries for dashboard and biometric history
-- The PK (user_id, date) already serves this, but if we query by date
-- across users (admin analytics), we need:
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_garmin_daily_date
    ON garmin_daily_data (date);

-- training_set_logs: the get_training_logs query filters on (user_id, plan_file)
-- The composite PK starts with these columns, so PG can use it.
-- No additional index needed — the PK serves as a prefix scan.

-- user_settings: get_users_with_garmin filters on non-null credentials
-- Partial index: only index rows where credentials exist
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_user_settings_garmin_active
    ON user_settings (user_id)
    WHERE garmin_username IS NOT NULL AND encrypted_garmin_password IS NOT NULL;
```

> **HPMS**, Chapter 7 (Indexing): *"The key to indexing is understanding which
> queries run most frequently and which are most expensive. Indexes trade write
> performance for read performance — every INSERT or UPDATE must also update
> every index on the table."*

### Index-Only Scans

For `chat_messages`, we could create a **covering index**:

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_chat_messages_covering
    ON chat_messages (user_id, time DESC)
    INCLUDE (role, content);
```

This lets PostgreSQL answer the `get_chat_history` query entirely from the index
without touching the heap (table data). The downside: the index is as large as
the table because it includes the `content` column. At 200M rows of chat text,
this index could be tens of gigabytes. The trade-off depends on your read/write
ratio — for a chat app that reads far more than it writes, it's worth it.

> **PiP**, Chapter 10 (Indexes): *"An index-only scan reads data exclusively from
> the index, never accessing the heap. This is the fastest possible scan for
> queries that only need indexed columns."*

### EXPLAIN ANALYZE — Your Best Friend

Always validate indexes with `EXPLAIN ANALYZE`:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT role, content FROM chat_messages
WHERE user_id = 'some-uuid' ORDER BY time DESC LIMIT 50;
```

Look for:
- **Index Scan** or **Index Only Scan** (good) vs **Seq Scan** (bad at scale)
- **Buffers: shared hit** (data was in cache) vs **shared read** (disk I/O)
- **Rows Removed by Filter** — if this is high, your index isn't selective enough

---

## 5. Data Volume & Partitioning

### The Numbers

| Table | Rows/User/Year | At 1M Users/Year |
|---|---|---|
| `garmin_daily_data` | 365 | 365,000,000 |
| `chat_messages` | ~200 (varies) | 200,000,000 |
| `training_set_logs` | ~1,000 | 1,000,000,000 |
| `manual_body_metrics` | ~50 | 50,000,000 |

At these volumes, even indexed queries slow down because the **index itself** is
too large to fit in memory. B-tree indexes grow logarithmically, but the constant
factors matter when the index is 50 GB.

### Partitioning `garmin_daily_data`

PostgreSQL declarative partitioning lets you split a table into physically
separate sub-tables. The query planner automatically prunes partitions that
can't contain matching rows.

**Range partitioning by date** (monthly):

```sql
-- Convert existing table to partitioned
CREATE TABLE garmin_daily_data_new (
    LIKE garmin_daily_data INCLUDING ALL
) PARTITION BY RANGE (date);

-- Create partitions
CREATE TABLE garmin_daily_data_2025_01
    PARTITION OF garmin_daily_data_new
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE garmin_daily_data_2025_02
    PARTITION OF garmin_daily_data_new
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- ... automated partition creation via cron or pg_partman
```

With monthly partitions, a query for "last 7 days" only hits 1-2 partitions
instead of scanning the entire table. Each partition has its own smaller indexes.

> **DDIA**, Chapter 6 (Partitioning): *"The main reason for wanting to partition
> data is scalability. Different partitions can be placed on different nodes in a
> shared-nothing cluster, so a large dataset can be distributed across many
> disks."*

### Partitioning `chat_messages`

Chat messages are best partitioned by **hash on `user_id`**, since queries always
filter by user:

```sql
CREATE TABLE chat_messages_partitioned (
    id BIGSERIAL,
    user_id UUID NOT NULL,
    role TEXT,
    content TEXT,
    time TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY HASH (user_id);

-- Create 32 partitions (power of 2 for even distribution)
CREATE TABLE chat_messages_p00 PARTITION OF chat_messages_partitioned
    FOR VALUES WITH (MODULUS 32, REMAINDER 0);
CREATE TABLE chat_messages_p01 PARTITION OF chat_messages_partitioned
    FOR VALUES WITH (MODULUS 32, REMAINDER 1);
-- ... through p31
```

Each partition holds ~1/32 of all users' messages. Queries filtering by
`user_id` hit exactly one partition.

### Automated Partition Management: pg_partman

For time-based partitions, use the `pg_partman` extension:

```sql
CREATE EXTENSION IF NOT EXISTS pg_partman;

SELECT partman.create_parent(
    p_parent_table := 'public.garmin_daily_data',
    p_control := 'date',
    p_type := 'range',
    p_interval := '1 month',
    p_premake := 3  -- create 3 future partitions
);
```

Run `partman.run_maintenance()` daily to auto-create new partitions.

### Data Retention

Not all data needs to live forever. Define retention policies:

```sql
-- Drop garmin data older than 3 years
DROP TABLE IF EXISTS garmin_daily_data_2022_01;
-- Or archive to cold storage (pg_dump specific partition → S3)
```

For `chat_messages`, consider keeping only the last N messages per user and
archiving the rest:

```sql
-- Materialized retention: keep last 500 messages per user
DELETE FROM chat_messages
WHERE ctid NOT IN (
    SELECT ctid FROM (
        SELECT ctid, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY time DESC) as rn
        FROM chat_messages
    ) sub WHERE rn <= 500
);
```

> **SRE**, Chapter 26 (Data Integrity): *"Data that isn't actively maintained is
> data that will eventually cause an incident. Define retention policies early,
> not when you're out of disk space."*

---

## 6. Read Scaling & Replication

### The Problem

Currently, `Repository` wraps a single `PgPool`:

```rust
pub struct Repository {
    pool: PgPool,
}
```

Every query — reads and writes — goes to the same PostgreSQL instance. At 1M
users, reads vastly outnumber writes. Every page load (dashboard, chat, training)
triggers 2-5 read queries. Writes happen only on sync, settings updates, and
chat messages.

### The Fix: Read Replicas

PostgreSQL streaming replication creates one or more read-only replicas that stay
in sync with the primary:

```
[Primary] ──WAL stream──→ [Replica 1]
                      └──→ [Replica 2]
```

Modify `Repository` to use two pools:

```rust
pub struct Repository {
    write_pool: PgPool,  // → primary
    read_pool: PgPool,   // → replica(s) via PgBouncer
}

impl Repository {
    pub async fn new(write_url: &str, read_url: &str) -> anyhow::Result<Self> {
        let write_pool = PgPoolOptions::new()
            .max_connections(50)
            .connect(write_url)
            .await?;
        let read_pool = PgPoolOptions::new()
            .max_connections(200)  // reads are more frequent
            .connect(read_url)
            .await?;
        Ok(Self { write_pool, read_pool })
    }
}
```

Then route queries accordingly:

```rust
// Reads use read_pool
pub async fn get_chat_history(&self, ...) -> ... {
    sqlx::query_as(...).fetch_all(&self.read_pool).await
}

// Writes use write_pool
pub async fn save_chat_message(&self, ...) -> ... {
    sqlx::query(...).execute(&self.write_pool).await
}
```

### Replication Lag

Streaming replication is **asynchronous by default**: there's a small delay
(typically <100ms) between a write on the primary and when it appears on
replicas. This means a user who sends a chat message and immediately refreshes
might not see it. Two solutions:

1. **Read-your-own-writes**: After a write, route that user's subsequent reads to
   the primary for a brief window (e.g., 5 seconds).
2. **Synchronous replication**: Guarantees replicas are up-to-date before
   acknowledging the write. Slower, but consistent. Only use for critical paths.

> **DDIA**, Chapter 5 (Replication): *"If the application reads from an
> asynchronous follower, it may see outdated information... This is not just a
> theoretical problem; it regularly causes confusion in applications."*

---

## 7. Background Sync

### Current: Sequential Loop

From `src/main.rs`:

```rust
tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(60 * 60));
    loop {
        interval.tick().await;
        match bg_state.repo.get_users_with_garmin().await {
            Ok(users) => {
                for s in users {                          // ← sequential!
                    if let Err(e) = handlers::perform_user_sync(&bg_state, s.user_id).await {
                        tracing::warn!("...");
                    }
                }
            }
            // ...
        }
    }
});
```

This iterates over **every user with Garmin credentials** one at a time. Each
sync involves:

1. Decrypting stored credentials (fast, ~µs)
2. OAuth token refresh if expired (1 HTTP round-trip, ~200ms)
3. Fetching up to `sync_days` (default 30) days of data, skipping existing dates
4. Each day = 1 Garmin API call (~300ms)
5. Each day = 1 DB upsert (~1ms)

**Best case per user** (all dates cached): ~200ms (just the auth check).
**Worst case per user** (new user, 30 days): ~10 seconds.

At 1M users with, say, 500K having Garmin credentials:

- Best case: 500,000 × 200ms = **100,000 seconds ≈ 27.7 hours**
- That's longer than the 1-hour sync interval. **It will never finish.**

### Fix 1: Concurrent Batching

The simplest improvement — process users in parallel:

```rust
use futures::stream::{self, StreamExt};

tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(60 * 60));
    loop {
        interval.tick().await;
        match bg_state.repo.get_users_with_garmin().await {
            Ok(users) => {
                let state = &bg_state;
                stream::iter(users)
                    .map(|s| async move {
                        if let Err(e) = handlers::perform_user_sync(state, s.user_id).await {
                            tracing::warn!("Sync failed for {}: {}", s.user_id, e);
                        }
                    })
                    .buffer_unordered(50)  // 50 concurrent syncs
                    .collect::<()>()
                    .await;
            }
            Err(e) => tracing::error!("Failed to fetch users: {}", e),
        }
    }
});
```

50 concurrent syncs × 200ms = **throughput of ~250 users/second**. For 500K
users, that's ~33 minutes — fits within the hour. But this is still fragile and
runs in the web process.

### Fix 2: Dedicated Job Queue

At true scale, sync should be a **separate service** consuming jobs from a queue:

```
[Scheduler (cron)] → [Redis / PG job queue] → [N Sync Workers]
                                                     ↓
                                              [Garmin API]
                                              [PostgreSQL]
```

Options for the job queue:
- **PostgreSQL-based**: Use `pgmq` or `SKIP LOCKED` pattern
- **Redis-based**: Use Redis Streams or a library like `sidekiq`-style queues
- **Cloud-native**: AWS SQS, Google Cloud Tasks

The `SKIP LOCKED` pattern in PostgreSQL is elegant:

```sql
-- Enqueue: mark users needing sync
UPDATE user_settings SET sync_status = 'pending'
WHERE last_sync_at < NOW() - INTERVAL '1 hour'
  AND garmin_username IS NOT NULL;

-- Worker: claim a batch
BEGIN;
SELECT user_id FROM user_settings
WHERE sync_status = 'pending'
ORDER BY last_sync_at ASC NULLS FIRST
LIMIT 50
FOR UPDATE SKIP LOCKED;

-- ... perform sync ...

UPDATE user_settings SET sync_status = 'idle', last_sync_at = NOW()
WHERE user_id = ANY($1);
COMMIT;
```

`SKIP LOCKED` ensures multiple workers don't pick the same user. Each worker
claims 50 users, syncs them concurrently, and commits. This is distributed,
fault-tolerant, and uses only PostgreSQL.

> **SRE**, Chapter 22 (Cascading Failures): *"Background processes that share
> resources with serving processes are a classic cause of cascading failures.
> A batch job that consumes too many DB connections can starve the real-time
> serving path."*

### Fix 3: Staggered Scheduling

Don't sync all users at the same wall-clock time. Distribute work across the
interval:

```rust
// Instead of syncing everyone at :00, assign each user a slot
let slot_minutes = (user_id_hash % 60) as u64;
// User abc123 syncs at :17, user def456 syncs at :42, etc.
```

This turns a spike of 500K syncs into a steady stream of ~8,333 syncs/minute.

### Per-Sync HTTP Client Waste

In `src/handlers/sync.rs`, each sync creates a new `reqwest::Client`:

```rust
let client = reqwest::Client::builder()
    .cookie_store(true)
    .user_agent("com.garmin.android.apps.connectmobile")
    .build()
    .unwrap_or_else(|_| reqwest::Client::new());
```

`reqwest::Client` internally creates a connection pool and DNS resolver. Creating
one per user wastes memory and prevents TCP connection reuse. At 1M users you'd
create 1M clients.

The problem is the `cookie_store(true)` — it makes the client stateful per-user.
The fix: use the shared `state.http_client` and manage cookies manually, or create
a lightweight cookie jar per request rather than a full `Client`:

```rust
// Share the Client, isolate cookies
let jar = reqwest::cookie::Jar::default();
let client = reqwest::Client::builder()
    .cookie_provider(Arc::new(jar))
    .user_agent(USER_AGENT)
    .build()?;
```

Even better: create a small pool of clients and reuse them across syncs.

---

## 8. Rate Limiting

### Current: In-Memory HashMap

From `src/middleware/rate_limit.rs`:

```rust
pub struct RateLimiter {
    state: Arc<Mutex<HashMap<std::net::IpAddr, (u32, Instant)>>>,
    max_requests: u32,
    window: std::time::Duration,
}
```

This works perfectly for one process. But it has two problems at scale:

1. **Multiple instances don't share state**: User hits instance A 5 times (under
   limit), then instance B 5 times (under limit) — they've made 10 requests but
   neither limiter knows about the other half.

2. **Memory growth**: The cleanup only triggers when `map.len() > 1000`. At 1M
   users, this map could hold hundreds of thousands of entries. The `Mutex`
   becomes a bottleneck as every request must acquire it.

### Fix: Redis-Based Rate Limiting

Redis is the industry standard for distributed rate limiting. The sliding window
algorithm using sorted sets:

```
MULTI
ZADD rate:{ip} {now_ms} {request_id}
ZREMRANGEBYSCORE rate:{ip} 0 {now_ms - window_ms}
ZCARD rate:{ip}
EXPIRE rate:{ip} {window_secs}
EXEC
```

In Rust with `redis-rs`:

```rust
pub struct DistributedRateLimiter {
    redis: redis::Client,
    max_requests: u32,
    window: Duration,
}

impl DistributedRateLimiter {
    pub async fn check(&self, ip: IpAddr) -> bool {
        let key = format!("rate:{}", ip);
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_millis() as f64;
        let window_start = now - self.window.as_millis() as f64;

        let mut conn = self.redis.get_multiplexed_async_connection().await.unwrap();
        let (_, _, count): ((), (), u32) = redis::pipe()
            .atomic()
            .zrembyscore(&key, 0.0, window_start)
            .zadd(&key, now, now.to_string())
            .zcard(&key)
            .expire(&key, self.window.as_secs() as i64)
            .query_async(&mut conn)
            .await
            .unwrap_or(((), (), self.max_requests + 1));

        count <= self.max_requests
    }
}
```

This is:
- **Distributed**: All instances share the same Redis, so all requests count.
- **Memory-bounded**: Redis handles TTL expiry, no manual cleanup.
- **Fast**: Redis operations take ~0.1ms.

> **DDIA**, Chapter 8 (Trouble with Distributed Systems): *"In a distributed
> system, you must assume that anything that can go wrong will go wrong. Rate
> limiters must fail open or closed — decide which before you need it in
> production."*

**Fail-open policy**: If Redis is unreachable, allow the request rather than
blocking all users. Rate limiting is a safety net, not a critical path.

---

## 9. Encryption at Scale

### Current: Single-Threaded ChaCha20

From `src/vault.rs`:

```rust
pub fn encrypt(&self, data: &str) -> (String, String) {
    let mut nonce_bytes = [0u8; 12];
    rand::rng().fill_bytes(&mut nonce_bytes);
    let nonce = Nonce::from_slice(&nonce_bytes);
    let ciphertext = self.cipher
        .encrypt(nonce, data.as_bytes())
        .expect("Encryption failure");
    // ...
}
```

ChaCha20Poly1305 is already one of the fastest authenticated ciphers in
software — it runs at ~1 GB/s on modern CPUs. At 100 users, you're encrypting
maybe 100 KB of credentials. At 1M users during a mass sync, you're decrypting
1M sessions.

### The Math

1M decryptions × ~1 KB each ≈ 1 GB of data. At 1 GB/s ChaCha20 throughput,
that's **~1 second of CPU time**. Spread across concurrent sync workers, this
is negligible. **Encryption is not a bottleneck at this scale.**

However, watch for:

1. **Entropy exhaustion**: `rand::rng().fill_bytes()` uses the OS CSPRNG
   (`/dev/urandom`). On Linux, this never blocks and is fast. But if you were on
   an embedded system or VM with poor entropy, high-volume nonce generation could
   slow down. Not a concern on standard Linux.

2. **Key rotation**: With 1M users, rotating `MASTER_KEY` means re-encrypting
   every secret. Design a migration path: support two keys simultaneously during
   rotation, decrypt with old key, re-encrypt with new key.

> **RiA**, Chapter 12 (Signals, Interrupts, and Exceptions): *"Cryptographic
> operations are CPU-bound. In async Rust, long-running CPU work should be
> offloaded to a blocking thread pool to avoid starving the async reactor."*

For bulk decryption during sync, use `tokio::task::spawn_blocking`:

```rust
let decrypted = tokio::task::spawn_blocking(move || {
    vault.decrypt(&encrypted, &nonce)
}).await??;
```

---

## 10. LLM Integration Under Load

### Current Architecture

From `src/llm/adapter.rs`:

```rust
pub trait LlmAdapter: Send + Sync {
    async fn generate(&self, prompt: &str, system_instruction: &str) -> Result<String, AppError>;
    async fn generate_with_tools(&self, ...) -> Result<String, AppError>;
}
```

The `FallbackLlmAdapter` chains Gemini → Ollama (or vice versa). Chat uses a
multi-turn tool-calling loop with up to 8 turns.

### At 100 Users

Maybe 10-20 concurrent chat requests. Each takes 2-10 seconds (LLM generation
time). Perfectly manageable.

### At 1M Users

If even 1% of users chat in the same hour, that's 10,000 concurrent LLM
requests. Each holds an HTTP connection open for seconds. Problems:

1. **Gemini API rate limits**: Google's free tier allows ~60 requests/minute.
   Even paid tiers have per-project limits. You'd need a quota increase or
   multiple API keys with rotation.

2. **Ollama is single-machine**: Self-hosted Ollama serves one model on one
   GPU. It can handle maybe 5-10 concurrent inferences. At 10K concurrent
   users, you need a fleet of GPU machines behind a load balancer.

3. **Token costs**: At $0.075/1M input tokens (Gemini Flash), 10K requests ×
   ~2K tokens each = 20M tokens/hour = **$1.50/hour**. That's manageable. But
   the tool-calling loop multiplies this — 8 turns × 10K requests × 2K tokens
   = $12/hour, $288/day. Design token budgets per user.

### Fixes

**Request queuing with backpressure**:

```rust
// Semaphore to limit concurrent LLM calls
let llm_semaphore = Arc::new(tokio::sync::Semaphore::new(100));

// In handler:
let permit = llm_semaphore.acquire().await?;
let response = state.llm.generate(prompt, system).await;
drop(permit);
```

This prevents overwhelming the LLM backend. Users beyond the 100th concurrent
request wait in a queue rather than getting timeout errors.

**Per-user rate limiting on LLM calls**: Prevent a single user from consuming
disproportionate resources:

```rust
// Redis key: llm_calls:{user_id}
// Limit: 50 LLM calls per hour per user
```

**Response caching**: The AI Analyst (`src/llm/analyst.rs`) uses a two-stage
approach: classify intent → run SQL → narrate. The SQL results for common
queries (e.g., "7-day HRV trend") rarely change within an hour. Cache them:

```rust
// Cache key: analyst:{user_id}:{metric}:{range_days}:{date}
// TTL: 1 hour
```

**Model tiering**: Use smaller, faster models for simple queries (intent
classification, SITREP) and reserve large models for complex analysis (DEBRIEF,
AAR):

```rust
match chat_mode {
    ChatMode::Sitrep => llm.generate_fast(prompt, system).await,
    ChatMode::Debrief => llm.generate_full(prompt, system).await,
}
```

> **SRE**, Chapter 21 (Handling Overload): *"Load shedding allows a service to
> maintain good performance for the requests it does accept, rather than
> degrading performance for all requests equally."*

---

## 11. Session & Authentication Scaling

### Current: Signed Cookies

From `src/main.rs`:

```rust
let cookie_key = derive_cookie_key(&cfg.cookie_secret);
```

Sessions are stored in **signed cookies** — the server doesn't maintain any
session state. The cookie contains the user ID and email, signed with HMAC-SHA256.
This is **already horizontally scalable**: any instance can verify any cookie
because they all share the same `COOKIE_SECRET`.

### What Works at 1M Users

Signed cookies are one of the best session strategies for horizontal scaling:

- No server-side session store to shard or replicate
- No session affinity ("sticky sessions") needed at the load balancer
- Stateless verification: O(1) CPU, no network call

> **USAH**, Chapter 27 (Security): *"Stateless authentication tokens (signed
> cookies, JWTs) eliminate the session store as a scaling bottleneck and
> single point of failure."*

### What to Watch

1. **Cookie size**: If you add more data to the cookie, it's sent with every
   request. Keep it minimal (user_id + email).

2. **Key rotation**: When rotating `COOKIE_SECRET`, support both old and new
   keys temporarily. Axum's `SignedCookieJar` supports this via `Key::from()` —
   you can try the new key first, fall back to the old key.

3. **CSRF token validation**: The current `csrf_middleware` checks `Origin` and
   `Referer` headers. This is stateless and scales fine. No changes needed.

---

## 12. File Storage

### Current: Local Filesystem

Files are stored at `uploads/{user_id}/`:

```rust
pub fn user_uploads_dir(user_id: Uuid) -> PathBuf {
    PathBuf::from("uploads").join(user_id.to_string())
}
```

### At 100 Users

With 100 users and maybe 5 files each (training plans, CSVs), that's 500 files
in 100 directories. The filesystem handles this trivially.

### At 1M Users: Filesystem Limits

1M directories under `uploads/`, each with 5-10 files = 5-10 million files. Most
filesystems (ext4, XFS) handle this, but:

- **Directory listing becomes slow**: `readdir()` on a directory with 1M entries
  is O(n). Use hash-based subdirectories: `uploads/ab/cd/abcd1234-.../`.
- **Single disk capacity**: 10M files × average 100 KB = 1 TB. Need network
  storage or object storage.
- **Single machine failure**: Disk dies, all files lost. Need replication.
- **Multi-instance**: With multiple Axum instances, each needs access to the
  same files. Local disk doesn't work unless you use a network filesystem
  (NFS, which is slow and fragile).

### The Fix: Object Storage (S3/MinIO)

Object storage is designed for this:

```
[Axum instance N] → [S3-compatible API] → [Object storage cluster]
```

Key scheme: `s3://gorilla-uploads/{user_id}/{filename}`

```rust
use aws_sdk_s3::Client as S3Client;

pub struct FileStore {
    client: S3Client,
    bucket: String,
}

impl FileStore {
    pub async fn upload(&self, user_id: Uuid, name: &str, data: &[u8]) -> anyhow::Result<()> {
        let key = format!("{}/{}", user_id, name);
        self.client.put_object()
            .bucket(&self.bucket)
            .key(&key)
            .body(data.into())
            .send()
            .await?;
        Ok(())
    }

    pub async fn download(&self, user_id: Uuid, name: &str) -> anyhow::Result<Vec<u8>> {
        let key = format!("{}/{}", user_id, name);
        let resp = self.client.get_object()
            .bucket(&self.bucket)
            .key(&key)
            .send()
            .await?;
        let bytes = resp.body.collect().await?.into_bytes();
        Ok(bytes.to_vec())
    }
}
```

For self-hosting, use **MinIO** — S3-compatible, runs in Docker:

```yaml
minio:
  image: minio/minio:latest
  command: server /data --console-address ":9001"
  environment:
    MINIO_ROOT_USER: gorilla
    MINIO_ROOT_PASSWORD: tactical_pass
  ports:
    - "9000:9000"
    - "9001:9001"
  volumes:
    - minio_data:/data
```

> **USAH**, Chapter 21 (Storage): *"Object storage separates the data path from
> the metadata path, allowing linear scalability... Unlike block storage, it
> doesn't require a filesystem layer and scales to billions of objects."*

---

## 13. Observability

### Current: In-Memory Everything

From `src/state.rs`:

```rust
pub struct Metrics {
    pub garmin_sync_attempts: usize,
    pub garmin_sync_success: usize,
    pub garmin_auth_valid: bool,
    pub llm_requests: usize,
    pub llm_tokens_used: usize,
}

pub struct LlmLogger {
    entries: Arc<Mutex<VecDeque<LlmLogEntry>>>,
    max_capacity: usize,  // 100 entries
}
```

At 100 users: you can SSH in, check the status page, and see what's happening.
At 1M users: this tells you nothing. Each instance has its own `Metrics` and
`LlmLogger`, and you can't aggregate them.

### The Three Pillars of Observability

> **BPE**, Chapter 2 (Methodologies): *"The three pillars of observability are
> metrics, logs, and traces. Each serves a different purpose: metrics tell you
> **what** is happening, logs tell you **why**, and traces tell you **where** in
> the system it happened."*

#### Metrics → Prometheus

Replace `Arc<Mutex<Metrics>>` with Prometheus counters:

```rust
use prometheus::{IntCounter, IntGauge, Histogram, register_int_counter, register_histogram};

lazy_static! {
    static ref SYNC_ATTEMPTS: IntCounter = register_int_counter!(
        "garmin_sync_attempts_total", "Total Garmin sync attempts"
    ).unwrap();
    static ref SYNC_DURATION: Histogram = register_histogram!(
        "garmin_sync_duration_seconds", "Time spent syncing per user"
    ).unwrap();
    static ref LLM_REQUESTS: IntCounter = register_int_counter!(
        "llm_requests_total", "Total LLM API calls"
    ).unwrap();
    static ref DB_QUERY_DURATION: Histogram = register_histogram!(
        "db_query_duration_seconds", "Database query latency"
    ).unwrap();
    static ref ACTIVE_CONNECTIONS: IntGauge = register_int_gauge!(
        "active_connections", "Currently active HTTP connections"
    ).unwrap();
}
```

Expose a `/metrics` endpoint:

```rust
.route("/metrics", get(|| async {
    let encoder = prometheus::TextEncoder::new();
    let metrics = prometheus::gather();
    encoder.encode_to_string(&metrics).unwrap()
}))
```

Prometheus scrapes this every 15 seconds. Grafana dashboards visualize it.

#### Logs → Structured Logging

Replace `tracing::info!("message")` with structured fields:

```rust
tracing::info!(
    user_id = %user_id,
    sync_days = sync_days,
    duration_ms = elapsed.as_millis(),
    "garmin sync completed"
);
```

Ship logs to a centralized system (Loki, Elasticsearch) via the `tracing-loki`
or `tracing-opentelemetry` crate.

#### Traces → OpenTelemetry

Distributed tracing shows you exactly where time is spent across services:

```rust
use tracing_opentelemetry::OpenTelemetryLayer;

// Trace a full request: HTTP → handler → DB query → LLM call → response
#[tracing::instrument(skip(state))]
pub async fn chat_stream_handler(State(state): State<AppState>, ...) {
    // Each span shows up in Jaeger/Tempo
}
```

> **SRE**, Chapter 6 (Monitoring Distributed Systems): *"Your monitoring system
> needs to very quickly answer two questions: What's broken? and Why?... Metrics
> answer the first; logs and traces answer the second."*

---

## 14. Server Rendering & Caching

### Current: Format Strings in `ui.rs`

Every page is rendered via Rust `format!()` strings:

```rust
pub fn chat_page(messages: &[ChatMessage]) -> Html<String> {
    Html(format!(r#"<!DOCTYPE html>
    <html>
    <head>...</head>
    <body>
    {message_list}
    </body>
    </html>"#,
        message_list = render_messages(messages),
    ))
}
```

### At 100 Users

Rendering a page takes ~10-100µs. With 100 users, total rendering load is
negligible.

### At 1M Users

If 10% of users load a page in the same minute, that's 100K page renders/minute
or ~1,700/second. Each render is fast (µs), so a single Axum instance can handle
it. But the format strings are **not pre-compiled** — the template logic runs
every time.

### Optimizations

**1. Template pre-compilation (Askama):**

Askama compiles templates at Rust compile time to native code. Zero runtime
parsing:

```rust
#[derive(Template)]
#[template(path = "chat.html")]
struct ChatTemplate<'a> {
    messages: &'a [ChatMessage],
}
```

The speedup is 2-5× for complex templates, but your current format strings are
already efficient. The real value of Askama is **maintainability** at scale, not
raw performance.

**2. HTTP caching headers:**

Static content (the HTML shell, CSS, JS) rarely changes. Add `Cache-Control`:

```rust
async fn security_headers_middleware(...) -> Response {
    // ...
    h.insert("Cache-Control", "public, max-age=3600".parse().unwrap());
}
```

For API responses (dashboard data, biometrics), use `Cache-Control: private, max-age=60`
to let browsers cache for 60 seconds.

**3. CDN for static assets:**

Serve htmx, Chart.js, and marked.js from a CDN (you already do this via
`cdn.jsdelivr.net`). Also consider putting your own domain behind a CDN
(CloudFlare, Fastly) to cache HTML pages at the edge.

**4. Fragment caching:**

Cache rendered HTML fragments in Redis for expensive pages like the dashboard:

```rust
let cache_key = format!("dashboard:{}:{}", user_id, today);
if let Some(cached) = redis.get(&cache_key).await? {
    return Html(cached);
}
let html = render_dashboard(data);
redis.set_ex(&cache_key, &html, 300).await?;  // 5 min TTL
Html(html)
```

> **BPE**, Chapter 5 (Applications): *"The fastest code is code that doesn't
> run. Caching at every layer — browser, CDN, application, database — is the
> single most effective performance optimization."*

---

## 15. Horizontal Scaling & Deployment

### Current: Single Container

`docker-compose.yaml` runs one `gorilla-coach` container and one PostgreSQL.

### At 1M Users: Multi-Instance Architecture

```yaml
version: '3.8'

services:
  # Load balancer
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - gorilla-coach-1
      - gorilla-coach-2
      - gorilla-coach-3

  gorilla-coach-1:
    build: .
    env_file: .env
    command: server

  gorilla-coach-2:
    build: .
    env_file: .env
    command: server

  gorilla-coach-3:
    build: .
    env_file: .env
    command: server

  # Sync workers (separate from web serving)
  sync-worker-1:
    build: .
    env_file: .env
    command: sync-worker  # new CLI subcommand

  sync-worker-2:
    build: .
    env_file: .env
    command: sync-worker

  # Infrastructure
  gorilla-db:
    image: timescale/timescaledb:latest-pg16
    # ...

  gorilla-db-replica:
    image: timescale/timescaledb:latest-pg16
    # streaming replication from gorilla-db

  pgbouncer:
    image: edoburu/pgbouncer:latest
    # ...

  redis:
    image: redis:7-alpine
    # ...

  minio:
    image: minio/minio:latest
    # ...
```

### Load Balancer Configuration

Nginx upstream with health checks:

```nginx
upstream gorilla_backend {
    least_conn;  # send to least-busy instance
    server gorilla-coach-1:3000;
    server gorilla-coach-2:3000;
    server gorilla-coach-3:3000;
}

server {
    listen 443 ssl http2;

    # SSE streams need long timeouts
    location /api/chat/stream {
        proxy_pass http://gorilla_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_buffering off;
        proxy_read_timeout 300s;
    }

    location / {
        proxy_pass http://gorilla_backend;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Statelessness Checklist

For horizontal scaling, every request must be handleable by any instance. Your
current state:

| Component | Stateless? | Fix Needed? |
|---|---|---|
| Session (signed cookies) | ✅ Yes | No |
| Rate limiter (in-memory) | ❌ No | → Redis |
| Metrics (in-memory) | ❌ No | → Prometheus |
| LLM logger (in-memory) | ❌ No | → Structured logging |
| File storage (local disk) | ❌ No | → S3/MinIO |
| Background sync (tokio::spawn) | ❌ No | → Job queue |
| SA token cache (in-memory) | ❌ No | → Redis |

> **DDIA**, Chapter 1: *"A service is stateless if it does not store any data
> that must survive restarts. Stateless services are trivially scalable — just
> add more instances."*

---

## 16. Garmin API

### The External Dependency Problem

Your app scrapes Garmin Connect via SSO authentication (there is no official
public API). This is the most fragile part of the system and the hardest to
scale.

### At 100 Users

100 hourly syncs × ~5 API calls each = 500 Garmin API calls/hour. Well
within their tolerance.

### At 1M Users

500,000 Garmin-connected users × 5 calls each = **2.5 million API calls/hour**.
Garmin will absolutely rate-limit or block your IP.

From `src/garmin.rs`, the `GarminApiError::RateLimited` variant exists but the
sync loop doesn't have a global backoff strategy:

```rust
pub enum GarminApiError {
    RateLimited,      // ← recognized but not globally coordinated
    ServerError(u16),
    AuthFailed,
    Other(String),
}
```

### Fixes

1. **Global rate governor**: Use a token bucket in Redis shared across all sync
   workers:

   ```rust
   // Before any Garmin API call:
   let allowed = redis.eval(RATE_LIMIT_SCRIPT,
       &["garmin_global_rate"],
       &[max_per_second, now]).await?;
   if !allowed {
       tokio::time::sleep(Duration::from_secs(1)).await;
   }
   ```

2. **Exponential backoff on 429**: When any worker gets a 429:
   - Immediately notify all workers via Redis pub/sub
   - All workers pause for an exponentially increasing duration
   - This is the **circuit breaker pattern**

3. **IP rotation**: Use multiple outbound IPs (proxy pool) to distribute
   requests across different source addresses.

4. **Webhook-based sync**: Garmin does offer a partner API (Connect IQ SDK) with
   webhook push for device data. This eliminates polling entirely but requires
   partnership status with Garmin.

5. **Data freshness tiers**: Not all users need hourly sync:
   - Active users (logged in today): sync every hour
   - Recent users (logged in this week): sync every 6 hours
   - Dormant users (haven't logged in for a month): sync daily or not at all

> **SRE**, Chapter 22 (Addressing Cascading Failures): *"Circuit breakers
> and exponential backoff are essential when depending on external services.
> Without them, a temporary outage at the dependency becomes a permanent outage
> for your system."*

---

## 17. Security at Scale

### Current Security Layers

Your app has solid security fundamentals:

- **CSRF**: Origin/Referer verification in `src/middleware/csrf.rs`
- **Encryption**: ChaCha20Poly1305 for secrets at rest
- **Signed cookies**: HMAC-SHA256 session integrity
- **Rate limiting**: Per-IP sliding window
- **CSP + security headers**: Injected via middleware
- **Input sanitization**: Filename sanitization, `html_escape()` for user content

### What Changes at 1M Users

1. **Attack surface increases linearly**: 1M users means 1M potential
   compromised accounts, 1M potential sources of malicious input.

2. **DDoS becomes realistic**: 100-user apps don't get DDoSed. 1M-user apps
   absolutely do. Your rate limiter (even the Redis version) won't stop a
   volumetric DDoS. Use CloudFlare or AWS Shield.

3. **Credential storage is a high-value target**: 1M encrypted Garmin
   passwords in the database. A leaked `MASTER_KEY` decrypts all of them.
   Mitigations:
   - **HSM (Hardware Security Module)**: Store the master key in an HSM, not an
     env var. AWS KMS, GCP Cloud KMS, or a physical HSM.
   - **Per-user key derivation**: Derive a unique encryption key per user from
     the master key + user_id. A partial DB leak doesn't compromise all users:
     ```rust
     fn user_key(master: &[u8], user_id: Uuid) -> [u8; 32] {
         let mut mac = HmacSha256::new_from_slice(master).unwrap();
         mac.update(user_id.as_bytes());
         let result = mac.finalize().into_bytes();
         result.into()
     }
     ```

4. **SQL injection surface**: The `analyst.rs` module generates SQL from LLM
   output (even though it uses a safelist of allowed metrics). At scale, prompt
   injection attacks become more likely. Ensure the safelist is **enforced in
   code, not by LLM compliance**:
   ```rust
   // Good: code-level enforcement
   if !ALLOWED_METRICS.contains(&intent.metric.as_str()) {
       return Err(AppError::llm("Unknown metric"));
   }
   ```

5. **Audit logging**: At 1M users, you need to know who did what, when. Log
   every authentication event, settings change, and data export to an
   append-only audit trail.

> **USAH**, Chapter 27 (Security): *"Security at scale is about defense in
> depth. No single layer is sufficient. Assume every layer will be breached
> and design the next layer accordingly."*

---

## 18. The Migration Path

You don't jump from 100 to 1M users overnight. Here's a phased approach:

### Phase 1: 100 → 1,000 Users (Quick Wins)

**Effort**: Days

- [ ] Add database indexes (migration file, zero downtime)
- [ ] Make `max_connections` configurable via env var
- [ ] Add `acquire_timeout` and `idle_timeout` to sqlx pool
- [ ] Add concurrent batching to background sync (`buffer_unordered(50)`)
- [ ] Add Prometheus metrics endpoint
- [ ] Add structured logging with `tracing` fields

### Phase 2: 1,000 → 10,000 Users (Infrastructure)

**Effort**: Weeks

- [ ] Deploy PgBouncer between app and PostgreSQL
- [ ] Set up PostgreSQL streaming replication (1 primary + 1 replica)
- [ ] Split `Repository` into read/write pools
- [ ] Replace in-memory rate limiter with Redis
- [ ] Add Redis for SA token cache and LLM response cache
- [ ] Move file storage to MinIO/S3
- [ ] Run 2-3 Axum instances behind Nginx load balancer
- [ ] Add health check endpoints for load balancer
- [ ] Implement LLM request semaphore (backpressure)

### Phase 3: 10,000 → 100,000 Users (Architecture)

**Effort**: Months

- [ ] Separate sync workers from web servers
- [ ] Implement job queue for Garmin sync (SKIP LOCKED or Redis Streams)
- [ ] Add table partitioning for `garmin_daily_data` and `chat_messages`
- [ ] Implement data retention policies
- [ ] Add CDN for static assets and edge caching
- [ ] Set up distributed tracing (OpenTelemetry + Jaeger/Tempo)
- [ ] Implement Garmin API global rate governor with circuit breaker
- [ ] Add per-user LLM rate limiting
- [ ] Implement user activity tiers for sync frequency

### Phase 4: 100,000 → 1,000,000 Users (Scale)

**Effort**: Months to quarters

- [ ] Multiple PostgreSQL replicas with geographic distribution
- [ ] PgBouncer cluster (multiple PgBouncer instances)
- [ ] Kubernetes or similar orchestration for auto-scaling
- [ ] DDoS protection (CloudFlare/AWS Shield)
- [ ] HSM for key management
- [ ] Per-user key derivation for encryption
- [ ] Audit logging to append-only store
- [ ] Cost modeling: per-user LLM cost budgets
- [ ] Garmin partnership for webhook-based data push
- [ ] Consider CQRS: separate read models (materialized views) from write path
- [ ] Blue-green deployments for zero-downtime releases

---

## Day-2 Operations: Monitoring the Current Deployment

Before scaling, you need to know what's happening in the system you have.

### Key Metrics to Watch

The `/status` page (admin-only) shows runtime metrics:
- Garmin sync attempts and success count
- LLM API requests and token usage
- LLM connection health
- Last sync timestamp
- Active model name

For deeper monitoring, check the structured logs:

```bash
# Count Garmin sync failures in the last hour
journal -u gorilla-coach --since "1 hour ago" | grep -c "Background sync failed"

# Count LLM fallback events
journal -u gorilla-coach --since "1 hour ago" | grep -c "Falling back to"

# Count rate-limited API calls
journal -u gorilla-coach --since "1 hour ago" | grep -c "rate limited (429)"
```

### Database Size Monitoring

```sql
-- Total database size
SELECT pg_size_pretty(pg_database_size('gorilla_hq')) AS db_size;

-- Per-table sizes
SELECT relname AS table,
       pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
       n_live_tup AS live_rows
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(oid) DESC;

-- Dead tuple ratio (indicates vacuum health)
SELECT relname, n_dead_tup, n_live_tup,
       ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;
```

For a single user with 30 days of data, expect:
- `garmin_daily_data`: ~30 rows, <100KB
- `chat_messages`: 100–1000 rows, <1MB
- `training_set_logs`: 50–500 rows, <100KB
- Total database: <10MB

### Connection Pool Health

The connection pool (`PgPool`) is configured for 20 max connections. For a
single-user deployment, you'll rarely use more than 2–3 simultaneously. Signs
of pool exhaustion:
- Requests hanging for seconds before responding
- `pool timed out` errors in logs
- Background sync delaying dashboard loads

Check PostgreSQL connection count:

```sql
SELECT count(*) AS connections, state
FROM pg_stat_activity
WHERE datname = 'gorilla_hq'
GROUP BY state;
```

### When to Start Scaling

You're fine at current scale. Start thinking about scaling when:
- **Database > 1GB**: Add indexes on `chat_messages(user_id, time DESC)`
- **Sync takes > 30 minutes**: Move from serial sync to parallel job queue
- **LLM rate limits > 10/day**: Add more Gemini models or increase Ollama capacity
- **Multiple users hitting rate limiters**: Move from in-memory to Redis-backed rate limiting
- **File uploads > 10GB**: Move from local disk to S3-compatible object storage

---

## 19. Further Reading

### Architecture & Distributed Systems

- **Designing Data-Intensive Applications** — Martin Kleppmann (2017).
  *The most important book on building systems that handle real data at scale.
  Covers replication, partitioning, consistency, and stream processing.
  Every chapter directly applies to the problems in this tutorial.*

- **Site Reliability Engineering** — Beyer, Jones, Petoff & Murphy (2016).
  *How Google runs production. Chapters on monitoring, incident response,
  load balancing, and cascading failures are essential for anyone operating
  a service at scale.*

- **The Site Reliability Workbook** — Beyer et al. (2018).
  *Practical companion to the SRE book. More examples, less theory.*

- **Building Microservices** — Sam Newman, 2nd Ed. (2021).
  *When (and when not) to split a monolith. The section on data decomposition
  is particularly relevant to separating the sync worker.*

### Database & PostgreSQL

- **High Performance MySQL** — Schwartz, Zaitsev & Tkachenko, 4th Ed. (2021).
  *Despite the MySQL name, the indexing, query optimization, and replication
  chapters apply to any relational database. The B-tree internals chapter is
  the best concise explanation of how database indexes work.*

- **PostgreSQL Internals: A Deep Dive** — Egor Rogov (2024).
  *PostgreSQL-specific internals: buffer management, MVCC, vacuuming,
  partitioning. Essential for understanding why your queries are slow.*

- **The Art of PostgreSQL** — Dimitri Fontaine (2020).
  *Advanced SQL patterns, window functions, CTEs. Useful for optimizing the
  analyst queries in `analyst.rs`.*

- **PostgreSQL 16 Administration Cookbook** — Riggs & Ciolli (2024).
  *Step-by-step for replication, PgBouncer, monitoring, and backup.*

### Systems & Performance

- **UNIX and Linux System Administration Handbook** — Nemeth, Snyder, Hein,
  Whaley & Mackin, 5th Ed. (2017).
  *The bible of system administration. Covers everything from DNS to storage
  to security to configuration management. Chapter 24 (Databases) and
  Chapter 27 (Security) are directly relevant.*

- **Systems Performance: Enterprise and the Cloud** — Brendan Gregg, 2nd Ed.
  (2020). *The definitive work on understanding why systems are slow.
  CPU, memory, disk, network — Gregg's USE method (Utilization, Saturation,
  Errors) is the gold standard for diagnosing bottlenecks.*

- **BPF Performance Tools** — Brendan Gregg (2019).
  *When you need to trace exactly what your application is doing at the
  kernel level. Useful for debugging connection pool exhaustion, file
  I/O latency, and network stalls.*

### Rust-Specific

- **Rust in Action** — Tim McNamara (2021).
  *Practical Rust for systems programming. The concurrency chapter explains
  why Rust's ownership model makes the concurrent sync worker safe.*

- **Programming Rust** — Blandy, Orendorff & Tindall, 2nd Ed. (2021).
  *The deep dive. The async chapter explains how tokio's runtime works,
  which matters when you're running 50 concurrent sync tasks.*

- **Zero To Production in Rust** — Luca Palmieri (2022).
  *Building a production web service in Rust/Actix (similar patterns to
  Axum). Covers telemetry, error handling, database access, and
  deployment — exactly the concerns of this tutorial.*

### Security

- **Cryptography Engineering** — Ferguson, Schneier & Kohno (2010).
  *Understand why ChaCha20Poly1305 was the right choice, when to use
  key derivation, and how to design key rotation.*

- **The Web Application Hacker's Handbook** — Stuttard & Pinto, 2nd Ed. (2011).
  *Understand the attacks your app faces: session fixation, CSRF, injection,
  authentication bypass. Dated but foundational.*

- **Threat Modeling: Designing for Security** — Adam Shostack (2014).
  *Systematic approach to finding security issues before attackers do.
  At 1M users, you're a target worth attacking.*

### Operations & DevOps

- **The Phoenix Project** — Kim, Behr & Spafford (2013).
  *Novel format. Teaches the principles behind DevOps: flow, feedback,
  continuous learning. Context for why you need CI/CD, monitoring, and
  automated deployment.*

- **Infrastructure as Code** — Kief Morris, 2nd Ed. (2020).
  *When your docker-compose grows into a fleet, you need Terraform,
  Ansible, or Pulumi. This book covers the patterns.*

- **Kubernetes in Action** — Marko Lukša, 2nd Ed. (2024).
  *When docker-compose isn't enough. Kubernetes handles auto-scaling,
  rolling deployments, and service discovery for multi-instance
  deployments.*

---

## Closing Note

The beauty of Gorilla Coach's current architecture is its **simplicity**. A
single Rust binary with sqlx, Axum, and a PostgreSQL database is one of the
most operationally simple stacks possible. Every layer of complexity added for
scale is a layer that can fail, that must be monitored, and that someone must
understand.

The right answer is almost always: **optimize for the scale you have, architect
for the scale you expect.** Add indexes now (free). Add PgBouncer when you hit
1,000 users (an afternoon of work). Split the sync worker when you hit 10,000
(a week). Switch to S3 and Redis when you hit 100,000 (a sprint). Kubernetes
and HSMs when you hit 1,000,000 (a quarter).

> *"Simplicity is prerequisite for reliability."*
> — Edsger W. Dijkstra

> *"Do the simplest thing that could possibly work."*
> — Ward Cunningham
