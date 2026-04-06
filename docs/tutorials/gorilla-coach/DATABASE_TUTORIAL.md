# Database Architecture in Gorilla Coach

## From Schema Design to Production Queries — A Deep Dive

> *"The limits of my language mean the limits of my world."*
> — Ludwig Wittgenstein, *Tractatus Logico-Philosophicus*
>
> In databases, the limits of your schema mean the limits of your application.
> Every query you can run, every insight you can extract, every invariant you
> can enforce — all are consequences of decisions made when the tables were
> defined. This tutorial examines those decisions in detail.

This tutorial covers the complete database layer of Gorilla Coach: how the
schema evolved across six migrations, why the Repository pattern was chosen over
an ORM, how sqlx provides compile-time safety without runtime overhead, how
COALESCE-based upserts preserve partial data from unreliable external APIs, and
how the Two-Stage Rocket analyst safely generates SQL from LLM output. Every
concept is grounded in the actual codebase and cross-referenced with foundational
texts.

---

## Reference Texts

| Abbreviation | Book |
|---|---|
| **DDIA** | Martin Kleppmann — *Designing Data-Intensive Applications* (2017) |
| **PoEAA** | Martin Fowler — *Patterns of Enterprise Application Architecture* (2002) |
| **ZtP** | Luca Palmieri — *Zero To Production in Rust* (2022) |
| **PiP** | Egor Rogov — *PostgreSQL Internals: A Deep Dive* (2024) |
| **AoP** | Dimitri Fontaine — *The Art of PostgreSQL* (2020) |
| **FDK** | C.J. Date — *Database Design and Relational Theory* (2019) |
| **HPMS** | Baron Schwartz, Peter Zaitsev & Vadim Tkachenko — *High Performance MySQL*, 4th Ed. (2021) |
| **RfR** | Jon Gjengset — *Rust for Rustaceans* (2021) |
| **PP** | Andrew Hunt & David Thomas — *The Pragmatic Programmer*, 20th Anniversary Ed. (2019) |
| **DDD** | Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) |

---

## Table of Contents

1. [Why PostgreSQL](#1-why-postgresql)
2. [The Schema: Six Migrations That Tell a Story](#2-the-schema)
3. [Table Design Decisions](#3-table-design-decisions)
4. [The Repository Pattern in Rust](#4-the-repository-pattern-in-rust)
5. [sqlx: Compile-Time SQL Without an ORM](#5-sqlx-compile-time-sql-without-an-orm)
6. [Connection Pooling and PgPool](#6-connection-pooling-and-pgpool)
7. [Domain Models and FromRow](#7-domain-models-and-fromrow)
8. [UPSERT: The COALESCE Pattern](#8-upsert-the-coalesce-pattern)
9. [Dynamic Queries with QueryBuilder](#9-dynamic-queries-with-querybuilder)
10. [The Two-Stage Rocket: Safe SQL from LLM Output](#10-the-two-stage-rocket)
11. [Migration Strategy and Schema Evolution](#11-migration-strategy-and-schema-evolution)
12. [Encryption at the Data Layer](#12-encryption-at-the-data-layer)
13. [Query Patterns Across Handlers](#13-query-patterns-across-handlers)
14. [Testing Without a Database](#14-testing-without-a-database)
15. [TimescaleDB: The Optional Superpower](#15-timescaledb-the-optional-superpower)
16. [What's Missing: Indexes, Constraints, and Gaps](#16-whats-missing)
17. [The Full Entity-Relationship Model](#17-the-full-entity-relationship-model)
18. [Further Reading](#18-further-reading)

---

## 1. Why PostgreSQL

Gorilla Coach uses PostgreSQL 16 (via TimescaleDB's image, which bundles the
TimescaleDB extension on top of standard PostgreSQL). The choice of PostgreSQL
over alternatives is deliberate:

### PostgreSQL vs SQLite

SQLite is the default choice for single-user embedded applications — it requires
no server process, stores everything in a single file, and is the most deployed
database in the world. For a self-hosted fitness app, it might seem like the
natural choice.

But Gorilla Coach has requirements that push beyond SQLite's comfort zone:

1. **Concurrent writes from background workers**: The hourly Garmin sync worker
   writes to `garmin_daily_data` while the user might be logging training sets
   or chatting with the AI. SQLite uses database-level locking — concurrent
   writes are serialized. PostgreSQL uses MVCC (Multi-Version Concurrency
   Control), allowing reads and writes to proceed simultaneously without
   blocking.

2. **Complex types**: PostgreSQL natively supports `UUID`, `TIMESTAMPTZ` (with
   timezone), `DOUBLE PRECISION`, and `TEXT` with no length limits. SQLite has
   a simpler type system where dates are stored as text strings, and UUIDs
   require manual encoding.

3. **Upsert semantics**: `ON CONFLICT ... DO UPDATE SET` with `COALESCE` is a
   PostgreSQL strength. SQLite added `ON CONFLICT` support in 3.24, but the
   `COALESCE` pattern for merge-preserving updates is more naturally expressed
   in PostgreSQL.

4. **TimescaleDB extensibility**: The `docker-compose.yaml` uses
   `timescale/timescaledb:latest-pg16`, which provides hypertable support for
   time-series data. While Gorilla Coach currently only uses this for the
   legacy `daily_logs` table, it opens the door to automatic partitioning and
   continuous aggregates for biometric data.

5. **Path to production**: If Gorilla Coach ever needs to scale beyond a
   single machine, PostgreSQL's streaming replication, logical replication,
   and connection pooling (PgBouncer) are battle-tested infrastructure.
   SQLite has no native replication.

> **DDIA**, Chapter 2 (Data Models and Query Languages): *"Relational databases
> have proved remarkably versatile... The dominance of relational databases
> has lasted around 40 years — an eternity in computing."*

### The docker-compose.yaml Configuration

```yaml
gorilla-db:
  image: timescale/timescaledb:latest-pg16
  container_name: gorilla-db
  environment:
    - POSTGRES_USER=${DB_USER:-gorilla}
    - POSTGRES_PASSWORD=${DB_PASSWORD:-tactical_pass}
    - POSTGRES_DB=${DB_NAME:-gorilla_hq}
  ports:
    - "5432:5432"
  volumes:
    - gorilla_data:/var/lib/postgresql/data
  restart: unless-stopped
```

**Key decisions:**

- **Named volume `gorilla_data`**: Data survives container restarts and
  rebuilds. This is critical — without a named volume, rebuilding the container
  would destroy all data.
- **Environment variable defaults**: `${DB_USER:-gorilla}` uses the `.env` value
  if set, or falls back to `gorilla`. This follows the 12-Factor principle of
  environment-based configuration.
- **Port exposure**: 5432 is exposed for local development (connecting via
  `psql`, pgAdmin, or DBeaver). In production, you'd limit this to the
  application network.
- **`restart: unless-stopped`**: The database restarts automatically after
  crashes or host reboots, unless explicitly stopped.

> **AoP**, Chapter 1 (SQL Is Code): *"PostgreSQL is not just a database server;
> it is a data processing platform. Treat your SQL as code: version it,
> review it, test it."*

---

## 2. The Schema: Six Migrations That Tell a Story

Gorilla Coach's schema evolved across six migration files. Each one tells a
story about a feature addition or architectural decision. Reading them in order
is reading the application's development history.

### Migration Naming Convention

Files follow the format `YYYYMMDDNN_description.sql`:

```
2026021500_init_schema.sql          — Foundation: users, settings, daily_logs, chat
2026021501_add_mfa_support.sql      — Garmin MFA and OAuth tokens
2026021600_garmin_daily_data.sql    — Comprehensive biometric data table
2026021700_manual_body_metrics.sql  — Manual body fat tracking
2026021701_add_google_oauth.sql     — Google OAuth refresh tokens
2026021800_training_and_extra_columns.sql — Training logs + extra Garmin fields
2026022100_training_day_done.sql    — Persistent day-completion tracking
2026030900_training_day_schedule.sql — 10-day microcycle schedule projection
```

The `NN` suffix is a sequence number within the day (00, 01, ...). This handles
the case where multiple migrations are created on the same day and ensures
deterministic ordering.

### Migration 1: `2026021500_init_schema.sql` — The Foundation

```sql
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY,
    email TEXT UNIQUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS user_settings (
    user_id UUID PRIMARY KEY REFERENCES users(id),
    garmin_username TEXT,
    encrypted_garmin_password TEXT,
    nonce TEXT
);

CREATE TABLE IF NOT EXISTS daily_logs (
    user_id UUID NOT NULL REFERENCES users(id),
    time TIMESTAMPTZ NOT NULL,
    hrv INT,
    rhr INT,
    sleep_score INT,
    training_load INT,
    notes TEXT
);

CREATE TABLE IF NOT EXISTS chat_messages (
    user_id UUID NOT NULL REFERENCES users(id),
    role TEXT,
    content TEXT,
    time TIMESTAMPTZ DEFAULT NOW()
);
```

**Design decisions:**

- **UUID primary keys**: Not auto-incrementing integers. UUIDs are generated
  application-side (`Uuid::new_v4()` in Rust), which means the application
  doesn't need to round-trip to the database to get an ID before using it.
  This is essential for the `get_or_create_user()` upsert pattern where the ID
  is generated speculatively — if the user already exists, the generated UUID
  is discarded.

- **`TIMESTAMPTZ` not `TIMESTAMP`**: All timestamps include timezone information.
  PostgreSQL stores `TIMESTAMPTZ` as UTC internally and converts to the
  client's timezone on read. This prevents the "what timezone is this date in?"
  ambiguity that plagues applications using `TIMESTAMP WITHOUT TIME ZONE`.

- **`TEXT` not `VARCHAR(N)`**: PostgreSQL's `TEXT` and `VARCHAR` have identical
  performance — both use the same varlena storage. `TEXT` is simpler (no
  arbitrary length limit to choose) and more aligned with Rust's `String`.

- **`IF NOT EXISTS`**: Migrations are idempotent. Running the same migration
  twice is harmless. This is defensive — if the migration tracking table gets
  corrupted, re-running migrations won't break the schema.

- **Foreign keys**: `user_settings.user_id REFERENCES users(id)` enforces
  referential integrity. You cannot create settings for a non-existent user.
  `chat_messages.user_id NOT NULL REFERENCES users(id)` ensures every message
  belongs to a real user.

- **`user_settings` as 1:1 with `users`**: The primary key `user_id` references
  `users(id)`, making this a strict one-to-one relationship. A user has at most
  one settings row. This is the **Supplementary Table** pattern — keeping
  optional or sensitive data in a separate table from the core user record.

> **FDK**, Chapter 5 (Normal Forms): *"Normalization is about eliminating
> redundancy. Every fact should be stored in exactly one place. If you need
> to update a fact, you update it in one row, not fifty."*

The `daily_logs` table also gets an optional TimescaleDB hypertable conversion:

```sql
DO $$ BEGIN
    PERFORM create_hypertable('daily_logs', 'time', if_not_exists => TRUE);
EXCEPTION WHEN OTHERS THEN
    RAISE NOTICE 'TimescaleDB not available, using plain Postgres table for daily_logs';
END $$;
```

This is a **graceful degradation** pattern. On a TimescaleDB-enabled instance,
`daily_logs` becomes a hypertable with automatic time-based partitioning. On
plain PostgreSQL, it remains a regular table. The application code doesn't know
or care which it is — the SQL interface is identical.

### Migration 2: `2026021501_add_mfa_support.sql` — Adding Columns

```sql
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS mfa_challenge_required BOOLEAN DEFAULT FALSE;
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS mfa_challenge_code TEXT;
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS mfa_challenge_expires_at TIMESTAMPTZ;
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS garmin_oauth_token TEXT;
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS garmin_oauth_token_nonce TEXT;
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS manual_body_fat_pct DOUBLE PRECISION;
ALTER TABLE users DROP COLUMN IF EXISTS google_sub;
```

**Schema evolution facts:**

- **`ADD COLUMN IF NOT EXISTS`**: Idempotent column additions. Safe to re-run.
- **`DEFAULT FALSE`**: The `mfa_challenge_required` column defaults to `FALSE`
  so existing rows don't need backfilling.
- **All new columns are nullable**: No `NOT NULL` constraint. This is intentional
  — existing rows don't have MFA data, and forcing a value would require a
  data migration.
- **`DROP COLUMN IF EXISTS`**: The `google_sub` column on `users` was removed,
  indicating an architectural change — Google authentication moved from OIDC
  subject IDs to OAuth refresh token flow (stored in `user_settings`).

> **DDIA**, Chapter 4 (Encoding and Evolution): *"Schema changes in relational
> databases are generally easy... ALTER TABLE can add columns, and most
> databases can do so without rewriting the entire table."*

In PostgreSQL, `ALTER TABLE ... ADD COLUMN` with a non-volatile default is
essentially free — it doesn't rewrite the table. The default is stored in the
catalog, and existing rows lazily adopt it on read. This is a PostgreSQL-specific
optimization since version 11.

> **PiP**, Chapter 8 (Updates and HOT): *"Adding a column with a default value
> in PostgreSQL 11+ only modifies the catalog, not the heap. Existing tuples
> are updated lazily on access — making column additions an O(1) DDL operation
> regardless of table size."*

### Migration 3: `2026021600_garmin_daily_data.sql` — The Biometric Data Model

```sql
CREATE TABLE IF NOT EXISTS garmin_daily_data (
    user_id UUID NOT NULL REFERENCES users(id),
    date DATE NOT NULL,
    -- Steps & Activity
    steps BIGINT,
    distance_meters DOUBLE PRECISION,
    active_calories BIGINT,
    -- ... 38 more columns ...
    synced_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, date)
);
```

This is the largest and most important table. **Design decisions in depth:**

**Composite primary key `(user_id, date)`**: One row per user per day. This is a
**natural key** — the combination of user identity and calendar date uniquely
identifies a biometric summary. No surrogate key needed.

> **FDK**, Chapter 4 (Keys and Constraints): *"A natural key is one whose
> component attributes already have meaning in the domain. A surrogate key is
> one invented solely for identification. Natural keys are preferred when they
> are stable and unique — they encode domain semantics directly into the
> schema."*

The `(user_id, date)` primary key also automatically creates a B-tree index on
`(user_id, date)`. This means queries like `WHERE user_id = $1 AND date >= $2
AND date <= $3` are **index-only scannable** — PostgreSQL can satisfy the
predicate entirely from the primary key index without touching secondary indexes.

**Why `BIGINT` not `INT`**: `steps`, `calories`, `sleep_duration_secs`, etc. use
`BIGINT` (8 bytes) instead of `INT` (4 bytes). For current Garmin data ranges,
`INT` (max 2.1 billion) would suffice. But `BIGINT` prevents overflow issues if
data semantics change (e.g., if Garmin starts reporting sub-second durations as
milliseconds), and the per-row storage cost (4 bytes extra per column) is
negligible compared to the `TEXT` columns.

**Why `DOUBLE PRECISION` not `NUMERIC`**: Fields like `hrv_last_night`,
`weight_grams`, `avg_spo2` use `DOUBLE PRECISION` (IEEE 754, 8 bytes) instead
of `NUMERIC` (arbitrary precision, variable size). This is a deliberate
trade-off:

- `DOUBLE PRECISION` matches Rust's `f64` exactly — no conversion overhead.
- `NUMERIC` is for financial data where exact decimal representation matters
  (e.g., currency). Biometric data is inherently imprecise — an HRV reading of
  `65.3` vs `65.300000000000004` is meaningless.
- `DOUBLE PRECISION` arithmetic is hardware-accelerated via FPU instructions.
  `NUMERIC` uses software bignum arithmetic.

> **AoP**, Chapter 4 (Data Types): *"Use NUMERIC for money. Use DOUBLE
> PRECISION for measurements. The difference is whether you need exact decimal
> representation or fast approximate computation."*

**All data columns are nullable**: Every biometric column (`steps`,
`sleep_score`, `hrv_last_night`, etc.) allows `NULL`. This is critical for
interacting with the Garmin API — not every endpoint returns every field for
every day. A day might have steps data but no HRV (if the watch wasn't worn
overnight). The `COALESCE` upsert pattern (Section 8) relies on this nullability
to merge partial data from multiple sync passes.

**`synced_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`**: The only non-nullable data
column. Records when this row was last synced, used for sync optimization
(`get_recently_synced_dates` skips days that were synced in the last N hours).

### Migration 4: `2026021700_manual_body_metrics.sql` — Supplementary Data

```sql
CREATE TABLE IF NOT EXISTS manual_body_metrics (
    user_id UUID NOT NULL REFERENCES users(id),
    date DATE NOT NULL,
    body_fat_pct DOUBLE PRECISION NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, date)
);
```

This table stores manually entered body composition data that supplements Garmin
data. The key insight: `body_fat_pct` here is `NOT NULL` — unlike the same field
in `garmin_daily_data`. Manual entries always have a value (the user explicitly
entered it). Garmin entries might not (the watch didn't read it that day).

The `format_daily_summary_with_bf` method in the Repository **merges** these two
sources with a forward-fill strategy: it finds the most recent manual body fat
entry on or before each Garmin date and uses it if the Garmin data lacks its own
reading:

```rust
let manual_bf = bf_entries.iter().rev()
    .find(|(date, _)| *date <= d.date)
    .map(|(_, v)| *v);
```

This is **temporal forward-fill** — a time-series technique where the last known
value carries forward until superseded. It avoids gaps in the biometric summary
without fabricating data.

### Migration 5: `2026021701_add_google_oauth.sql` — Encrypted Tokens

```sql
ALTER TABLE user_settings
    ADD COLUMN IF NOT EXISTS google_refresh_token TEXT,
    ADD COLUMN IF NOT EXISTS google_refresh_token_nonce TEXT;
```

Google OAuth refresh tokens are stored encrypted (via ChaCha20Poly1305 in the
Vault). The `_nonce` companion column stores the per-encryption nonce needed for
decryption. This pairing pattern (`encrypted_value` + `nonce`) appears three
times in `user_settings`:

1. `encrypted_garmin_password` + `nonce`
2. `garmin_oauth_token` + `garmin_oauth_token_nonce`
3. `google_refresh_token` + `google_refresh_token_nonce`

Section 12 covers this encryption pattern in detail.

### Migration 6: `2026021800_training_and_extra_columns.sql` — New Domain

```sql
CREATE TABLE IF NOT EXISTS training_set_logs (
    user_id UUID NOT NULL REFERENCES users(id),
    plan_file TEXT NOT NULL,
    day_label TEXT NOT NULL,
    exercise_name TEXT NOT NULL,
    set_number INT NOT NULL,
    actual_weight TEXT,
    actual_reps TEXT,
    technique TEXT,
    logged_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, plan_file, day_label, exercise_name, set_number)
);
```

**A five-column composite primary key.** This is unusual and worth examining.

The key encodes a **hierarchical path**: user → training plan → day → exercise →
set number. Each level narrows the scope. The primary key says: "for a given
user, in a given plan file, on a given training day, for a given exercise, set
number N is unique."

This enables the upsert in `upsert_training_sets`: if the user re-logs set 3 of
bench press on day A of their plan, the `ON CONFLICT` clause updates the existing
row instead of creating a duplicate.

**`actual_weight TEXT` and `actual_reps TEXT`**: These are `TEXT`, not numeric
types. At first glance this seems wrong — weight and reps are numbers. But the
training CSV format supports ranges and annotations: `"80-85"`, `"8-10"`,
`"bodyweight"`, `"failure"`. Storing as text preserves the user's original
notation. The parsing happens application-side when needed.

The migration also adds columns to `garmin_daily_data`:

```sql
ALTER TABLE garmin_daily_data ADD COLUMN IF NOT EXISTS sleep_restless_moments BIGINT;
ALTER TABLE garmin_daily_data ADD COLUMN IF NOT EXISTS sleep_avg_overnight_hr DOUBLE PRECISION;
ALTER TABLE garmin_daily_data ADD COLUMN IF NOT EXISTS skin_temp_overnight DOUBLE PRECISION;
```

These were added as the Garmin sync expanded to pull more data fields. The
`ADD COLUMN` approach is preferred over creating a new table because these fields
belong to the same entity (a day's biometric summary) and adding nullable columns
is free in PostgreSQL 11+.

### Migration 7: `2026022100_training_day_done.sql` — Day Completion Tracking

```sql
CREATE TABLE IF NOT EXISTS training_day_done (
    user_id UUID NOT NULL REFERENCES users(id),
    plan_file TEXT NOT NULL,
    day_label TEXT NOT NULL,
    marked_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, plan_file, day_label)
);
```

A **three-column composite primary key** that tracks which training days the
user has marked as complete. This is separate from `training_set_logs` because a
day can be "done" even if not every exercise was logged — the user decides.

The separation also enables cross-device persistence: the "Mark Day Done"
button in the training tracker UI calls `POST /api/training/mark-day-done`,
which inserts a row here. The auto-select logic on page load uses this table
(combined with logged days) to skip completed days and jump to the next
relevant workout.

`INSERT ... ON CONFLICT DO NOTHING` makes marking idempotent. Unmarking
deletes the row. No boolean column needed — row presence *is* the state.

### Migration 8: `2026030900_training_day_schedule.sql` — Schedule Projection

```sql
CREATE TABLE IF NOT EXISTS training_day_schedule (
    user_id UUID NOT NULL REFERENCES users(id),
    plan_file TEXT NOT NULL,
    day_label TEXT NOT NULL,
    day_order INT NOT NULL DEFAULT 0,
    scheduled_date DATE NOT NULL,
    PRIMARY KEY (user_id, plan_file, day_label)
);
```

This table stores the **projected training schedule** — mapping each training
day from a plan CSV onto a calendar date using a 10-day microcycle pattern.
The cycle uses offsets `[0, 2, 5, 8]` within a 10-day window, repeating for
all days in the plan.

The composite primary key `(user_id, plan_file, day_label)` ensures one
schedule entry per day per plan. `day_order` preserves the original ordering
from the CSV. `scheduled_date` is the computed calendar date.

The schedule is initialized via `POST /api/v2/training/init-schedule` (which
projects all days from the anchor date) and can be shifted forward/backward
via `POST /api/v2/training/shift-schedule`. The calendar sync endpoint
(`POST /api/v2/sync-calendar`) reads this table to create Google Calendar
events for each scheduled training day.

`DELETE + INSERT` pattern is used for re-projection rather than upserts,
since a schedule shift replaces all dates at once.

---

## 3. Table Design Decisions

### The Wide Table vs Narrow Table Debate

`garmin_daily_data` has **43 columns**. In database design, this invites the
question: should this be one wide table or multiple narrow tables?

**Wide table (current design):**
```
garmin_daily_data
  (user_id, date, steps, distance, calories, ..., hrv, sleep_score, ..., synced_at)
```

**Narrow tables (EAV or normalized):**
```
garmin_metrics
  (user_id, date, metric_name TEXT, metric_value DOUBLE PRECISION)

-- Or split by domain:
garmin_heart_rate (user_id, date, rhr, max_hr, min_hr, avg_hr)
garmin_sleep      (user_id, date, score, duration_secs, deep, light, rem, awake)
garmin_activity   (user_id, date, steps, distance, calories, floors)
```

**Why wide wins here:**

1. **Single fetch**: The dashboard, chat context, and analyst all need multiple
   metrics for a date range. One `SELECT *` returns everything. With narrow
   tables, you need joins or multiple queries.

2. **Atomic upsert**: One `INSERT ... ON CONFLICT ... DO UPDATE SET` handles all
   fields. With narrow tables, you'd need 6+ upserts per day synced.

3. **No type coercion**: Steps (integer), HRV (float), and HRV status (text)
   have different types. An EAV table with `metric_value TEXT` loses type
   safety and requires casting everywhere.

4. **Query simplicity**: `WHERE hrv_last_night IS NOT NULL` is simpler than
   `WHERE metric_name = 'hrv' AND metric_value IS NOT NULL`.

> **HPMS**, Chapter 6 (Schema Design): *"Avoid the Entity-Attribute-Value (EAV)
> pattern when the attributes are known at design time. EAV converts compile-time
> type errors into runtime data errors, makes queries harder to write and slower
> to execute, and defeats the purpose of a relational schema."*

The trade-off: adding a new Garmin metric requires a schema migration
(`ALTER TABLE ADD COLUMN`). With EAV, you'd just insert a new `metric_name`.
But given that PostgreSQL column additions are O(1) and the migration is a
single line of SQL, this trade-off is acceptable. Type safety and query
simplicity outweigh migration convenience.

> **PoEAA**, Chapter 12 (Object-Relational Structural Patterns): *"The choice
> between one table and many tables for closely related data is fundamentally
> about query patterns. If you always access the data together, store it
> together."*

### Why `chat_messages` Has No Primary Key

The `chat_messages` table has no primary key and no indexes:

```sql
CREATE TABLE IF NOT EXISTS chat_messages (
    user_id UUID NOT NULL REFERENCES users(id),
    role TEXT,
    content TEXT,
    time TIMESTAMPTZ DEFAULT NOW()
);
```

This is an **append-only log**. Messages are never updated or deleted — only
inserted and read. The access pattern is always `WHERE user_id = $1 ORDER BY
time DESC LIMIT $2`. There's no need to identify individual messages by key.

This is a deliberate simplicity choice for the current scale. At larger scale,
you'd add a primary key (for de-duplication and point lookups) and an index on
`(user_id, time DESC)` — see Section 16.

> **DDIA**, Chapter 3 (Storage and Retrieval): *"Log-structured storage systems
> treat data as an append-only sequence. This simplifies concurrent writes
> (no in-place updates) but requires indexes for efficient reads."*

### The `user_settings` Mega-Row

`user_settings` has grown through migrations to include:

- Garmin credentials (username, encrypted password, nonce)
- MFA state (challenge required, code, expiry)
- OAuth tokens (Garmin OAuth, Google OAuth — encrypted)
- Profile data (manual body fat percentage)
- Sync metadata (last sync timestamp)

This violates strict normalization: MFA state, OAuth tokens, and profile data
are different concerns stored in one row. But the practical justification is
strong:

1. **Always accessed together**: The settings page loads all of this at once.
   The sync worker needs credentials + OAuth + last sync time.

2. **One-to-one with users**: There's exactly one settings row per user. No
   join needed — `WHERE user_id = $1` is a primary key lookup.

3. **Atomic updates**: Updating the Garmin password and OAuth token in the same
   transaction is trivial with one table.

The cost: the `UserSettings` struct must include `#[sqlx(default)]` on columns
that might not be selected (when the query doesn't include them), adding a
maintenance burden when columns are added.

> **PP**, Chapter 14 (Domain Languages): *"Good enough software is better than
> perfect software that was never shipped. The relentless pursuit of
> normalization can produce schemas that are technically correct and practically
> unusable."*

---

## 4. The Repository Pattern in Rust

### What It Is

All database access in Gorilla Coach goes through a single struct:

```rust
#[derive(Clone)]
pub struct Repository {
    pool: PgPool,
}
```

Handlers never construct SQL directly. They call repository methods:

```rust
// Handler calls:
let user = state.repo.get_or_create_user(&email).await?;
let data = state.repo.get_garmin_range(user_id, start, end).await?;
state.repo.save_chat_message(user_id, "user", &content).await?;
```

The `pool` field is private — external code cannot access the connection pool
directly. All SQL is encapsulated within `Repository` methods.

> **PoEAA**, Chapter 10 (Repository): *"A Repository mediates between the
> domain and data mapping layers using a collection-like interface for
> accessing domain objects. Client objects construct query specifications
> declaratively and submit them to Repository for satisfaction."*

### Why Not an ORM?

Rust has several ORMs: Diesel (fully compiled queries, strict type safety),
SeaORM (async, ActiveRecord-like), and Diesel-async. Gorilla Coach uses none
of them, instead writing raw SQL with sqlx. This is a deliberate choice:

**1. SQL is the best DSL for SQL**

Consider the `upsert_garmin_daily` method. The `ON CONFLICT ... DO UPDATE SET`
with `COALESCE` across 40+ columns is straightforward SQL. Expressing this in
Diesel's type-safe DSL would require importing and composing dozens of schema
types, table macros, and conflict target structs. The SQL is more readable:

```sql
ON CONFLICT (user_id, date) DO UPDATE SET
    steps=COALESCE(excluded.steps, garmin_daily_data.steps),
    distance_meters=COALESCE(excluded.distance_meters, garmin_daily_data.distance_meters),
    ...
```

Vs the hypothetical Diesel version:

```rust
diesel::insert_into(garmin_daily_data::table)
    .values(&new_data)
    .on_conflict((garmin_daily_data::user_id, garmin_daily_data::date))
    .do_update()
    .set((
        garmin_daily_data::steps.eq(coalesce(
            excluded(garmin_daily_data::steps),
            garmin_daily_data::steps,
        )),
        // ... repeat 40 times
    ))
```

The ORM version is longer, harder to read, and offers no additional safety
beyond what sqlx provides. The SQL version is what you'd write in `psql`, what
a DBA would review, and what `EXPLAIN ANALYZE` operates on.

> **ZtP**, §3.6 (Database): *"sqlx gives you the best of both worlds: you write
> SQL (the language of databases) and get compile-time verification (the
> guarantee of type safety). Unlike ORMs, there's no abstraction layer between
> your intent and the database's execution."*

**2. No schema duplication**

ORMs require you to maintain a Rust-side schema definition that mirrors the
database schema. In Diesel, this is the `schema.rs` file generated by
`diesel print-schema`. Every migration requires regenerating this file and
ensuring the Rust structs stay in sync.

With sqlx, the schema exists in one place: the migration files. The `FromRow`
derive on domain structs maps directly from SQL result sets. If you add a column
to the database, you add a field to the struct. No intermediate schema file.

**3. Full SQL power**

The analyst module uses `SELECT *` with dynamic date ranges. The training log
uses `QueryBuilder` for batch inserts with variable row counts. The chat context
method joins biometric data with chat messages through application-level logic.
None of these require ORM abstractions — they're naturally expressed as SQL
queries called from Rust.

> **AoP**, Chapter 2 (Thinking in SQL): *"SQL is a fourth-generation language.
> It describes what you want, not how to get it. The database engine's query
> planner has 40+ years of optimization behind it. When you use an ORM, you're
> asking a library author to out-think PostgreSQL's query planner — and they
> can't."*

### The Transaction Script Pattern

Gorilla Coach's Repository implements the **Transaction Script** pattern from
Fowler's PoEAA:

> **PoEAA**, Chapter 9 (Transaction Script): *"A Transaction Script organizes
> business logic by procedures where each procedure handles a single request
> from the presentation... Its simplicity is its greatest asset; it works well
> for applications with a small amount of logic."*

Each repository method is a self-contained unit of work: `get_or_create_user`
performs one SQL statement, `save_chat_message` performs one INSERT,
`upsert_garmin_daily` performs one upsert. There are no multi-statement
transactions in the current codebase — each method is a single SQL statement
that is atomic by default.

This is a conscious simplicity choice. If a handler needs to update settings
AND save an OAuth token, it calls two repository methods. If the second fails,
the first has already committed. For the current use case (self-hosted, single
user), this is fine. If atomicity across multiple operations becomes necessary,
you'd use `pool.begin()` for explicit transactions:

```rust
// Hypothetical explicit transaction (not currently in the codebase)
let mut tx = self.pool.begin().await?;
sqlx::query("UPDATE user_settings SET ...").execute(&mut *tx).await?;
sqlx::query("INSERT INTO audit_log ...").execute(&mut *tx).await?;
tx.commit().await?;
```

---

## 5. sqlx: Compile-Time SQL Without an ORM

### What sqlx Does

sqlx is an async SQL toolkit for Rust. Its defining feature: SQL queries are
checked at compile time against a live database (or cached schema metadata).
If your SQL references a non-existent column, the compiler catches it.

From `Cargo.toml`:

```toml
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-native-tls", "postgres", "chrono", "uuid", "migrate"] }
```

**Feature flags explained:**

- `runtime-tokio`: Use Tokio as the async runtime (matches the app's runtime).
- `tls-native-tls`: Use the system's native TLS stack for database connections.
  Alternative: `tls-rustls` for a pure-Rust TLS implementation.
- `postgres`: PostgreSQL driver. Excludes MySQL, SQLite to avoid compiling
  unused code.
- `chrono`: Automatic conversion between PostgreSQL `TIMESTAMPTZ`/`DATE` and
  Rust `chrono::DateTime<Utc>`/`chrono::NaiveDate`.
- `uuid`: Automatic conversion between PostgreSQL `UUID` and Rust `uuid::Uuid`.
- `migrate`: Embeds migration files at compile time via the `sqlx::migrate!`
  macro.

### The Two APIs: `query` and `query_as`

**`sqlx::query`** — for statements that don't return rows (INSERT, UPDATE, DELETE)
or return untyped rows:

```rust
sqlx::query("INSERT INTO chat_messages (user_id, role, content) VALUES ($1, $2, $3)")
    .bind(user_id).bind(role).bind(content)
    .execute(&self.pool).await?;
```

**`sqlx::query_as`** — for SELECT statements that map rows to a Rust struct:

```rust
let user = sqlx::query_as::<_, User>(
    "INSERT INTO users (id, email, created_at) VALUES ($1, $2, NOW())
     ON CONFLICT (email) DO UPDATE SET email = excluded.email
     RETURNING *"
)
.bind(Uuid::new_v4())
.bind(email)
.fetch_one(&self.pool)
.await?;
```

The `<_, User>` type parameter tells sqlx to deserialize each row into a `User`
struct. The `_` is the database type (inferred as Postgres from the pool).

### Fetch Methods

sqlx provides four fetch methods with distinct semantics:

| Method | Returns | Use When |
|---|---|---|
| `fetch_one` | `T` | Exactly one row expected. Errors if 0 or 2+ rows. |
| `fetch_optional` | `Option<T>` | Zero or one row expected. |
| `fetch_all` | `Vec<T>` | Any number of rows. Loads all into memory. |
| `fetch` | `Stream<T>` | Large result sets. Processes rows one at a time. |

Gorilla Coach uses all four:

```rust
// fetch_one: get_or_create_user always returns exactly one row (RETURNING *)
let user = sqlx::query_as::<_, User>(...).fetch_one(&self.pool).await?;

// fetch_optional: user might not have settings yet
let s = sqlx::query_as::<_, UserSettings>(...).fetch_optional(&self.pool).await?;

// fetch_all: get all chat messages (bounded by LIMIT)
let msgs = sqlx::query_as::<_, ChatMessage>(...).fetch_all(&self.pool).await?;
```

`fetch` (streaming) is not currently used because result sets are bounded by
`LIMIT` clauses. At larger scale, streaming would be appropriate for the
background sync's `get_users_with_garmin()` call, which currently loads all
users into memory.

### Parameterized Queries: Security by Construction

Every value in every query goes through `.bind()`:

```rust
sqlx::query("SELECT * FROM garmin_daily_data WHERE user_id = $1 AND date >= $2 AND date <= $3")
    .bind(user_id).bind(start).bind(end)
```

The `$1`, `$2`, `$3` are positional parameters. sqlx sends the query and
parameters separately to PostgreSQL — the query is parsed and planned once,
parameters are substituted safely. There is no string interpolation, no escaping,
no possibility of SQL injection.

This is **parameterized queries by construction**: the API doesn't offer a way
to interpolate user input into query strings. The only way to pass data is
through `.bind()`, which uses PostgreSQL's binary protocol to transmit values
with their native types. A `Uuid` is sent as 16 bytes, a `NaiveDate` as a
4-byte date, a `String` as a length-prefixed byte array. No quoting, no
escaping, no injection surface.

> **ZtP**, §7.2 (SQL Injection): *"Parameterized queries are not a mitigation
> for SQL injection — they are the elimination of SQL injection. The attack
> surface doesn't exist because user data never enters the query string."*

---

## 6. Connection Pooling and PgPool

### Pool Configuration

```rust
pub async fn new(url: &str) -> anyhow::Result<Self> {
    let pool = PgPoolOptions::new()
        .max_connections(20)
        .connect(url)
        .await?;
    Ok(Self { pool })
}
```

`PgPool` is sqlx's connection pool. It maintains a set of open connections to
PostgreSQL and distributes them to concurrent tasks.

**Why connection pooling matters:**

Opening a PostgreSQL connection requires TCP handshake + TLS negotiation +
PostgreSQL authentication + connection setup. This takes ~5-50ms depending on
network and TLS configuration. If every query opened a new connection, a handler
making 3 queries would add 15-150ms of overhead. The pool amortizes this cost:
connections are opened once and reused across requests.

**Why 20 connections:**

PostgreSQL creates a process per connection. Each process consumes ~5-10MB of
RAM. With `max_connections = 20`, that's 100-200MB committed to connection
overhead. For a single-server deployment serving a handful of users, 20 is more
than enough — each handler holds a connection for the duration of a single
query (typically <1ms), so 20 connections can serve hundreds of requests per
second.

> **HPMS**, Chapter 5 (Connection Management): *"Setting the pool size too high
> wastes memory and increases contention on internal PostgreSQL locks. Setting it
> too low causes application-level queuing. The sweet spot depends on the number
> of CPU cores, disk speed, and query complexity — but for OLTP workloads,
> 2× CPU cores is a good starting point."*

### PgPool Internal Architecture

`PgPool` is internally `Arc<SharedPool>` — cloning a pool is O(1) (incrementing
a reference counter). This is why `Repository` can be `#[derive(Clone)]` and
why `AppState` can clone the repository into every handler without performance
concern:

```rust
#[derive(Clone)]
pub struct Repository {
    pool: PgPool,  // internally Arc — clone is O(1)
}
```

When a handler calls `.fetch_one(&self.pool)`, sqlx:

1. Acquires a connection from the pool (or waits if all are in use).
2. Sends the query over the connection.
3. Reads the response.
4. Returns the connection to the pool.

The connection lifecycle is managed by Rust's ownership system: the connection is
borrowed for the duration of the `await`, then released. No explicit `close()` or
`return()` needed.

> **RfR**, Chapter 1 (Foundations): *"Reference counting via Arc is Rust's
> pragmatic answer to shared ownership. When the exact lifetime of shared data
> is determined by runtime control flow rather than lexical scope, Arc provides
> safe garbage collection with deterministic cleanup."*

---

## 7. Domain Models and FromRow

### The Derive Macro Stack

Every database-mapped struct in Gorilla Coach uses this derive pattern:

```rust
#[derive(Debug, Serialize, Deserialize, FromRow, Clone)]
pub struct GarminDailyData {
    pub user_id: Uuid,
    pub date: NaiveDate,
    pub steps: Option<i64>,
    // ... 40+ fields
    pub synced_at: DateTime<Utc>,
}
```

Each derive macro serves a specific layer:

| Derive | Purpose | Used By |
|---|---|---|
| `Debug` | `{:?}` formatting for logging | `tracing::debug!("{:?}", data)` |
| `Serialize` | Rust → JSON | JSON responses, `serde_json::to_string` |
| `Deserialize` | JSON → Rust | Parsing Garmin API responses |
| `FromRow` | SQL row → Rust struct | `sqlx::query_as` result mapping |
| `Clone` | Value duplication | Passing data across handler boundaries |

### FromRow: Column-to-Field Mapping

`FromRow` is sqlx's key derive macro. It generates a `from_row` implementation
that:

1. Reads each field from the SQL result row by **column name** (not position).
2. Converts the PostgreSQL type to the Rust type using type registrations.
3. Handles `Option<T>` as nullable columns (SQL `NULL` → Rust `None`).

The column name must exactly match the Rust field name. For `GarminDailyData`,
the `resting_heart_rate` field maps to the `resting_heart_rate` SQL column.

### The `#[sqlx(default)]` Attribute

```rust
#[derive(Debug, Serialize, Deserialize, FromRow, Clone)]
pub struct UserSettings {
    pub user_id: Uuid,
    pub garmin_username: Option<String>,
    // ...
    #[sqlx(default)]
    pub garmin_oauth_token: Option<String>,
    #[sqlx(default)]
    pub manual_body_fat_pct: Option<f64>,
    #[sqlx(default)]
    pub google_refresh_token: Option<String>,
    #[sqlx(default)]
    pub google_refresh_token_nonce: Option<String>,
}
```

`#[sqlx(default)]` tells the `FromRow` derive: "if this column is missing from
the query result, use `Default::default()` instead of erroring." For
`Option<T>`, the default is `None`.

This is essential for **schema evolution**: queries written before a column was
added (like `google_refresh_token`) still work because the missing column
defaults to `None`. You don't need to update every `SELECT` statement when
adding a column — only the ones that need the new data.

This is also used deliberately in `get_user_settings`, where the query
explicitly lists columns:

```rust
sqlx::query_as::<_, UserSettings>(
    "SELECT user_id, garmin_username, encrypted_garmin_password, nonce,
     garmin_oauth_token, garmin_oauth_token_nonce, manual_body_fat_pct,
     google_refresh_token, google_refresh_token_nonce
     FROM user_settings WHERE user_id = $1"
)
```

Not all columns are selected (MFA columns are omitted). `#[sqlx(default)]`
prevents errors for the unselected fields.

### Type Mapping: PostgreSQL ↔ Rust

The `chrono` and `uuid` feature flags in sqlx enable automatic type conversions:

| PostgreSQL Type | Rust Type | Notes |
|---|---|---|
| `UUID` | `uuid::Uuid` | 16-byte binary on both sides |
| `TIMESTAMPTZ` | `chrono::DateTime<Utc>` | PostgreSQL stores as microseconds since epoch |
| `DATE` | `chrono::NaiveDate` | Calendar date, no timezone |
| `BIGINT` | `i64` | 8-byte signed integer |
| `DOUBLE PRECISION` | `f64` | IEEE 754 double |
| `TEXT` | `String` | Variable-length UTF-8 |
| `BOOLEAN` | `bool` | 1-byte true/false |
| `INT` | `i32` | 4-byte signed integer |

When a column is nullable (no `NOT NULL` constraint), sqlx maps it to
`Option<T>`. SQL `NULL` becomes Rust `None`. This is a natural fit —
`Option` is Rust's way of expressing "this value might not exist," which
is exactly what SQL `NULL` means.

> **RfR**, Chapter 2 (Types): *"Rust's Option type and SQL NULL encode the
> same semantic: the absence of a value. sqlx bridges them automatically, giving
> you type-safe nullable column access without manual null checking."*

---

## 8. UPSERT: The COALESCE Pattern

### The Problem

Garmin data arrives from multiple API endpoints at different times. Monday's sync
might return steps and heart rate. Tuesday's re-sync of Monday might add sleep
data (which wasn't available yet on Monday). Wednesday's sync might add the
resolved HRV value.

A naive `INSERT` would fail on the second sync (duplicate key). A naive
`INSERT ... ON CONFLICT DO UPDATE SET steps = excluded.steps` would **overwrite**
Monday's steps data with `NULL` if Tuesday's API call didn't return steps.

### The Solution: COALESCE Merge

```sql
INSERT INTO garmin_daily_data (user_id, date, steps, distance_meters, ...)
VALUES ($1, $2, $3, $4, ...)
ON CONFLICT (user_id, date) DO UPDATE SET
    steps=COALESCE(excluded.steps, garmin_daily_data.steps),
    distance_meters=COALESCE(excluded.distance_meters, garmin_daily_data.distance_meters),
    ...
    synced_at=excluded.synced_at
```

`COALESCE(a, b)` returns the first non-null argument. Here:

- `excluded.steps` is the new value being inserted.
- `garmin_daily_data.steps` is the existing value in the row.

The logic: **"Use the new value if it's not null. Otherwise, keep what we
already have."**

This means:
- New data always writes (first insert or update with non-null value).
- Null incoming data never overwrites existing data.
- `synced_at` always updates (no `COALESCE`) to track when the row was last
  touched.

This pattern appears in Gorilla Coach for **40+ columns** in
`upsert_garmin_daily`. It's the same SQL repeated with different column names —
verbose but correct and explicit.

> **AoP**, Chapter 8 (Writing SQL): *"COALESCE is one of the most underused
> functions in application SQL. It replaces entire application-level null-check
> trees with a single expression in the query. When combined with ON CONFLICT,
> it creates merge semantics that would require complex application logic to
> replicate."*

### Why This Matters for External API Data

Garmin Connect is not a well-documented public API — it's a scrape of internal
endpoints. Different endpoints return different fields. Some fields are
populated hours after the fact (HRV needs post-processing, sleep analysis
runs retrospectively). The `COALESCE` pattern makes the sync idempotent and
accumulative:

1. **First sync**: Inserts whatever data is available. Missing fields are `NULL`.
2. **Second sync**: Updates only fields that have new non-null values. Existing
   non-null fields are preserved.
3. **Third sync**: Same behavior. Data accumulates without loss.

The only exception is `synced_at`, which always updates to mark the latest
sync time.

### The Rust Side

The `upsert_garmin_daily` method binds all 43 values positionally:

```rust
.bind(d.user_id).bind(d.date)
.bind(d.steps).bind(d.distance_meters).bind(d.active_calories).bind(d.total_calories)
.bind(d.floors_climbed).bind(d.intensity_minutes)
// ... 37 more binds
```

Each `.bind()` call corresponds to a `$N` parameter. The ordering must match
exactly — `$1` is `user_id`, `$2` is `date`, `$3` is `steps`, etc. With 43
parameters, this is error-prone. The positional coupling between the SQL string
and the `.bind()` chain is the main maintenance risk in this method.

An alternative approach would use `sqlx::query_as!` (the macro version) which
verifies column/parameter correspondence at compile time. However, the macro
version requires a `DATABASE_URL` at compile time, which complicates CI builds.
The runtime `query_as` function is the pragmatic choice.

---

## 9. Dynamic Queries with QueryBuilder

### The Problem

`upsert_training_sets` needs to insert a variable number of rows in one
statement. The number of sets per exercise varies — bench press might have 3
sets, deadlifts might have 5. You can't use a fixed-parameter query.

### The Solution: QueryBuilder

```rust
pub async fn upsert_training_sets(
    &self,
    user_id: Uuid,
    plan_file: &str,
    day_label: &str,
    exercise_name: &str,
    sets: &[(i32, Option<String>, Option<String>, Option<String>)],
    performed_at: Option<chrono::DateTime<chrono::Utc>>,
) -> anyhow::Result<()> {
    if sets.is_empty() {
        return Ok(());
    }
    let ts = performed_at.unwrap_or_else(chrono::Utc::now);

    let mut qb: sqlx::QueryBuilder<sqlx::Postgres> = sqlx::QueryBuilder::new(
        "INSERT INTO training_set_logs \
         (user_id, plan_file, day_label, exercise_name, set_number, \
          actual_weight, actual_reps, logged_at, technique) "
    );

    qb.push_values(sets, |mut b, (set_num, weight, reps, technique)| {
        b.push_bind(user_id)
            .push_bind(plan_file)
            .push_bind(day_label)
            .push_bind(exercise_name)
            .push_bind(set_num)
            .push_bind(weight)
            .push_bind(reps)
            .push_bind(ts)
            .push_bind(technique);
    });

    qb.push(
        " ON CONFLICT (user_id, plan_file, day_label, exercise_name, set_number) \
          DO UPDATE SET actual_weight = EXCLUDED.actual_weight, \
          actual_reps = EXCLUDED.actual_reps, logged_at = EXCLUDED.logged_at, \
          technique = EXCLUDED.technique"
    );

    qb.build().execute(&self.pool).await?;
    Ok(())
}
```

**How `push_values` works:**

For 3 sets, `push_values` generates:

```sql
VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9), ($10,$11,$12,$13,$14,$15,$16,$17,$18), ($19,$20,$21,$22,$23,$24,$25,$26,$27)
```

Each value goes through a bind parameter — **no string interpolation, no
injection risk**, even with dynamic row counts. `QueryBuilder` manages parameter
numbering internally.

**The guard clause:**

```rust
if sets.is_empty() {
    return Ok(());
}
```

An empty `push_values` call produces invalid SQL (`VALUES` with no rows).
The guard clause prevents this.

**`ON CONFLICT` with `EXCLUDED`:**

The `EXCLUDED` pseudo-table references the values that would have been inserted.
Unlike the `COALESCE` pattern used for Garmin data, training logs use a
**full overwrite** on conflict — if the user re-logs set 3 of bench press, the
new weight and reps replace the old values entirely. This is intentional:
training logs represent the user's explicit input, not accumulated API data.

> **PiP**, Chapter 10 (Updates and HOT): *"ON CONFLICT processing in
> PostgreSQL creates a new tuple version rather than modifying in place, which
> maintains MVCC guarantees. The old version becomes dead and is later cleaned
> by VACUUM."*

---

## 10. The Two-Stage Rocket: Safe SQL from LLM Output

### The Danger

The AI analyst needs to query biometric data based on natural language input.
"How's my HRV trending?" should query `hrv_last_night` over the last 7 days.
"Analyze my sleep for the past month" should query `sleep_score` over 30 days.

The naive approach — ask the LLM to generate a SQL query directly — would be a
**critical security vulnerability**. An LLM could generate:

```sql
DROP TABLE users;
-- or
SELECT * FROM user_settings;  -- leak encrypted credentials
-- or
'; UPDATE user_settings SET encrypted_garmin_password = 'hacked' WHERE 1=1; --
```

Even with "system prompt engineering," LLMs can be jailbroken. Relying on an
LLM to generate safe SQL is like relying on user input to be well-formatted —
it's a TOCTOU violation waiting to happen.

> **DDIA**, Chapter 4 (Encoding and Evolution): *"Any boundary between trusted
> and untrusted code must be enforced by the program, not by convention or
> instructions. LLM output is untrusted input."*

### The Two-Stage Rocket Architecture

Gorilla Coach's `analyst.rs` implements a two-stage approach that eliminates
the risk entirely:

**Stage 1 — Classification (LLM):**

The LLM receives the user's message and a constrained system prompt. Its
**only job** is to output a JSON object identifying the metric, time range,
and comparison flag:

```json
{"metric": "hrv", "range_days": 14, "comparison": true}
```

The LLM never sees SQL, never generates SQL, never touches the database. It
only classifies intent into a structured `AnalystIntent`.

**Stage 2 — Data Fetch (Code):**

The Rust code maps the classified intent to pre-written, parameterized queries:

```rust
fn metric_column(metric: &str) -> Option<&'static str> {
    match metric {
        "hrv" => Some("hrv_last_night"),
        "rhr" => Some("resting_heart_rate"),
        "weight" => Some("weight_grams"),
        "sleep" => Some("sleep_score"),
        "stress" => Some("avg_stress"),
        "body_battery" => Some("body_battery_high"),
        "training_load" => Some("training_load"),
        "spo2" => Some("avg_spo2"),
        _ => None,
    }
}
```

The return type is `Option<&'static str>` — a **compile-time string literal**.
The column name can only ever be one of these 8 values. If the LLM outputs
`"metric": "'; DROP TABLE users; --"`, `metric_column` returns `None`, and the
function errors with `"Unknown metric"`. No SQL is generated, no query is
executed.

The actual data fetch uses the standard Repository method:

```rust
let rows = repo
    .get_garmin_range(user_id, start, today)
    .await
    .map_err(|e| AppError::llm(format!("DB error: {e}")))?;
```

This is the same `get_garmin_range` used by the dashboard — `SELECT * FROM
garmin_daily_data WHERE user_id = $1 AND date >= $2 AND date <= $3`. The metric
column name is used only in the application-level `extract_value` function, not
in SQL:

```rust
fn extract_value(d: &GarminDailyData, col: &str, metric: &str) -> Option<f64> {
    match col {
        "hrv_last_night" => d.hrv_last_night,
        "resting_heart_rate" => d.resting_heart_rate.map(|v| v as f64),
        "weight_grams" => d.weight_grams,
        // ...
        _ => None,
    }
}
```

The "SQL" is actually a Rust `match` over struct fields. No user input — not
even LLM-processed user input — reaches any SQL query.

### The Safety Invariant

The security of this system rests on a simple, verifiable invariant:

**All SQL queries in the codebase are string literals with only bind parameters.
No variable is ever interpolated into a query string.**

This can be verified by grepping:

```bash
grep -rn 'format!.*SELECT\|format!.*INSERT\|format!.*UPDATE\|format!.*DELETE' gorilla_server/src/repository/
# Expected output: zero results
```

The `QueryBuilder` in `upsert_training_sets` is the one exception, but it uses
`.push_bind()` for all values — the only non-literal parts of the query are
bind parameter placeholders generated by sqlx.

> **PP**, Chapter 25 (Assertive Programming): *"Don't assume it — prove it.
> Security boundaries should be enforced by code structure, not by hoping that
> no one puts user data where it shouldn't go."*

### ALLOWED_METRICS: The Safelist

```rust
pub const ALLOWED_METRICS: &[&str] = &[
    "hrv", "rhr", "weight", "sleep", "stress",
    "body_battery", "training_load", "spo2", "training_volume",
];
```

This constant defines the complete set of metrics the analyst can query. It
serves as documentation, validation, and the source of truth. The chat tool
executor checks against this list before calling `get_metric_stats`.

The list includes `"training_volume"`, which has a separate code path — it
queries `garmin_daily_data.activities_count` and
`garmin_daily_data.training_load` instead of a single column. This shows the
flexibility of the code-based approach: special-cased metrics are handled by
branching in Rust, not by SQL gymnastics.

---

## 11. Migration Strategy and Schema Evolution

### Embedded Migrations

```rust
pub async fn migrate(&self) -> anyhow::Result<()> {
    sqlx::migrate!("./migrations").run(&self.pool).await?;
    Ok(())
}
```

`sqlx::migrate!("./migrations")` is a **compile-time macro** that:

1. Reads all `.sql` files in the `migrations/` directory.
2. Computes a hash of each file's contents.
3. Embeds the filenames, SQL, and hashes into the binary.

At runtime, `.run(&self.pool)`:

1. Creates a `_sqlx_migrations` table (if it doesn't exist).
2. Checks which migrations have been applied (by filename and hash).
3. Runs unapplied migrations in filename order.
4. Records each applied migration with its hash.

**No external migration tool needed.** The binary is self-migrating. The
`Commands::Server` variant calls `repo.migrate().await?` before starting the
web server. The `Commands::InitDb` variant calls it explicitly for manual
setup.

### Migration Runs on Startup

```rust
Commands::Server => {
    repo.migrate().await?;
    // ... proceed to start server
}
```

This means **every deployment automatically applies pending migrations**. Deploy
a new binary with a new migration, restart the service, and the schema updates.
No separate migration step.

> **ZtP**, §7.3 (Database Migrations): *"Automatic migration on startup is the
> simplest deployment strategy for small services. The risk is that a failed
> migration can prevent the service from starting. The mitigation is testing
> migrations in staging before production."*

### Idempotent Migrations

Every migration uses `IF NOT EXISTS` or `IF EXISTS` guards:

```sql
CREATE TABLE IF NOT EXISTS garmin_daily_data ...
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS garmin_oauth_token TEXT;
ALTER TABLE users DROP COLUMN IF EXISTS google_sub;
```

If a migration partially applies (e.g., server crashes mid-migration), re-running
it is safe. The guards prevent `already exists` errors.

### Schema Migration vs Data Migration

All six migrations are **schema migrations** — they change table structure (DDL).
None of them are **data migrations** — they don't transform existing data (DML).

This is partly by design (data migrations are riskier and slower) and partly
because the application handles schema evolution gracefully:

- All new columns are nullable, so existing rows don't need backfilling.
- `#[sqlx(default)]` in Rust handles missing columns at the query level.
- The `COALESCE` upsert pattern handles partial data at the write level.

If a data migration were needed (e.g., converting `weight_grams` to
`weight_kg`), it would be a separate migration file with `UPDATE` statements,
run within a transaction:

```sql
-- Hypothetical data migration
BEGIN;
ALTER TABLE garmin_daily_data ADD COLUMN weight_kg DOUBLE PRECISION;
UPDATE garmin_daily_data SET weight_kg = weight_grams / 1000.0 WHERE weight_grams IS NOT NULL;
ALTER TABLE garmin_daily_data DROP COLUMN weight_grams;
COMMIT;
```

---

## 12. Encryption at the Data Layer

### The Encryption Pattern

Sensitive data is encrypted before reaching the database. The `Vault` (ChaCha20Poly1305)
encrypts plaintext into a `(ciphertext, nonce)` pair, both base64-encoded for
storage in `TEXT` columns:

```rust
let (ciphertext, nonce) = vault.encrypt(&plaintext);
let plaintext = vault.decrypt(&ciphertext, &nonce)?;
```

### Column Pairing Convention

Every encrypted value has a companion nonce column:

| Value Column | Nonce Column | Stores |
|---|---|---|
| `encrypted_garmin_password` | `nonce` | Garmin Connect password |
| `garmin_oauth_token` | `garmin_oauth_token_nonce` | Garmin OAuth2 token blob |
| `google_refresh_token` | `google_refresh_token_nonce` | Google OAuth refresh token |

The nonce naming is inconsistent (`nonce` for the original, `X_nonce` for later
additions). This is a consequence of incremental development — the first
encryption was added early, the convention evolved for later columns.

### Why Encrypt in the Application, Not the Database?

PostgreSQL has `pgcrypto` for in-database encryption. Gorilla Coach encrypts
in the application layer (Vault) instead. Reasons:

1. **Key isolation**: The `MASTER_KEY` never touches the database. If the
   database is compromised (SQL dump, backup theft, unauthorized access), the
   attacker gets ciphertext that's useless without the key. With `pgcrypto`,
   the encryption key would be in the SQL query — visible in `pg_stat_statements`,
   query logs, and memory.

2. **Algorithm control**: The application uses ChaCha20Poly1305, a modern AEAD
   cipher with authentication. `pgcrypto` defaults to older algorithms (AES-CBC
   without authentication). Using application-level encryption lets you choose
   the algorithm, manage rotation, and audit the implementation.

3. **Defense in depth**: Even if an attacker gains SQL access (via SQL injection
   in another application sharing the database, for example), they cannot decrypt
   credential data because the key is not in the database layer.

> **DDIA**, Chapter 7 (Transactions): *"Encryption at rest should be applied
> at the lowest layer that can hold the key securely. For application-managed
> secrets, this is the application layer, not the database."*

### The COALESCE Pattern with Encrypted Data

The `update_settings` upsert uses `COALESCE` for OAuth tokens:

```sql
ON CONFLICT (user_id) DO UPDATE SET
    garmin_username=excluded.garmin_username,
    encrypted_garmin_password=excluded.encrypted_garmin_password,
    nonce=excluded.nonce,
    garmin_oauth_token=COALESCE(excluded.garmin_oauth_token, user_settings.garmin_oauth_token),
    garmin_oauth_token_nonce=COALESCE(excluded.garmin_oauth_token_nonce, user_settings.garmin_oauth_token_nonce)
```

Credentials are always overwritten (the user explicitly re-entered them).
OAuth tokens use `COALESCE` because they might not be available at the time
of settings update — they're obtained separately during the login flow. This
prevents the settings form from accidentally clearing a valid OAuth token.

---

## 13. Query Patterns Across Handlers

### The `get_or_create_user` Pattern

```rust
pub async fn get_or_create_user(&self, email: &str) -> anyhow::Result<User> {
    let user = sqlx::query_as::<_, User>(
        "INSERT INTO users (id, email, created_at) VALUES ($1, $2, NOW())
         ON CONFLICT (email) DO UPDATE SET email = excluded.email
         RETURNING *"
    )
    .bind(Uuid::new_v4())
    .bind(email)
    .fetch_one(&self.pool)
    .await?;
    Ok(user)
}
```

This is a **speculative insert** — it generates a UUID, attempts to insert, and
if the email already exists, updates nothing and returns the existing row. The
`RETURNING *` clause returns the row regardless of whether it was inserted or
updated.

The `DO UPDATE SET email = excluded.email` is a no-op update (setting email to
itself). This is necessary because `ON CONFLICT DO NOTHING` doesn't return a
row with `RETURNING`. The no-op update forces PostgreSQL to return the existing
row.

> **AoP**, Chapter 8 (Writing SQL): *"An INSERT ... ON CONFLICT DO UPDATE ...
> RETURNING is the most efficient way to implement 'get or create' in
> PostgreSQL. It's a single round-trip, atomic, and handles concurrency
> correctly — two simultaneous requests for the same email won't create
> duplicate users."*

### The Chat Context Pattern

```rust
pub async fn get_chat_context(&self, user_id: Uuid, limit: i64)
    -> anyhow::Result<(String, Vec<ChatMessage>)>
{
    let today = chrono::Utc::now().date_naive();
    let start = today - chrono::Duration::days(2);
    let rows = self.get_garmin_range(user_id, start, today).await.unwrap_or_default();
    let bf_entries = self.get_manual_body_fat_entries(user_id, today).await.unwrap_or_default();

    let bio_lines: Vec<String> = rows.iter().map(|d| {
        let manual_bf = bf_entries.iter().rev()
            .find(|(date, _)| *date <= d.date)
            .map(|(_, v)| *v);
        Self::format_daily_summary_with_bf(d, manual_bf)
    }).collect();

    let msgs = sqlx::query_as::<_, ChatMessage>(...)
        .fetch_all(&self.pool).await?;

    let bio_ctx = bio_lines.join("\n");
    Ok((bio_ctx, msgs.into_iter().rev().collect()))
}
```

This method combines **three data sources** in application code:

1. **Garmin biometric data** (last 3 days) from `garmin_daily_data`.
2. **Manual body fat entries** from `manual_body_metrics` (for forward-fill).
3. **Chat history** from `chat_messages` (most recent N messages).

The merge happens in Rust, not SQL. A pure-SQL approach would use a `LEFT JOIN`
between `garmin_daily_data` and `manual_body_metrics` with a lateral subquery for
the forward-fill. This would be more efficient (one round-trip instead of three)
but significantly harder to read and maintain. The current approach makes three
simple queries and merges in application code — clear, debuggable, and fast
enough for the current scale.

Note the **defensive `unwrap_or_default()`**: if the Garmin or body fat query
fails, the method returns empty data instead of propagating the error. The chat
context is "best effort" — missing biometric data shouldn't prevent the user
from chatting.

### The Date Optimization Pattern

```rust
pub async fn get_recently_synced_dates(
    &self,
    user_id: Uuid,
    start: chrono::NaiveDate,
    end: chrono::NaiveDate,
    min_synced_at: chrono::DateTime<chrono::Utc>,
) -> anyhow::Result<Vec<chrono::NaiveDate>> {
    let rows: Vec<(chrono::NaiveDate,)> = sqlx::query_as(
        "SELECT date FROM garmin_daily_data
         WHERE user_id = $1 AND date >= $2 AND date <= $3 AND synced_at >= $4"
    )
    .bind(user_id).bind(start).bind(end).bind(min_synced_at)
    .fetch_all(&self.pool)
    .await?;
    Ok(rows.into_iter().map(|r| r.0).collect())
}
```

This is a **sync optimization query**. Before fetching data from Garmin for a
date, the sync worker checks if data was already synced recently
(`synced_at >= threshold`). Days that were synced in the last N hours are
skipped, avoiding redundant API calls.

The query returns only dates (not full rows) — a lightweight projection that
minimizes data transfer. The caller uses these dates as a "skip set" when
iterating over the date range.

### The Reverse-Chronological Chat Pattern

```rust
pub async fn get_chat_history(&self, user_id: Uuid, limit: i64)
    -> anyhow::Result<Vec<ChatMessage>>
{
    let msgs = sqlx::query_as::<_, ChatMessage>(
        "SELECT role, content FROM chat_messages
         WHERE user_id = $1 ORDER BY time DESC LIMIT $2"
    )
    .bind(user_id).bind(limit)
    .fetch_all(&self.pool).await?;
    Ok(msgs.into_iter().rev().collect())
}
```

The query fetches the **N most recent messages** by ordering `DESC` and limiting.
But the UI needs them in **chronological order** (oldest first). The
`.rev().collect()` reverses the vector in application code.

Why not `ORDER BY time ASC LIMIT N`? Because that would return the **oldest** N
messages, not the most recent. To get the most recent in ascending order, you'd
need a subquery:

```sql
SELECT * FROM (
    SELECT role, content FROM chat_messages
    WHERE user_id = $1 ORDER BY time DESC LIMIT $2
) sub ORDER BY time ASC
```

The application-level `.rev()` is simpler and avoids the subquery overhead.
With the `LIMIT` bounded (typically 50-100 messages), the in-memory reversal is
trivial.

---

## 14. Testing Without a Database

### The Design Constraint

Gorilla Coach's 37 tests run with `cargo test` — no database required. This
means no `Repository` methods are tested directly. Instead, tests focus on:

1. **Domain model correctness**: Can you create structs? Do defaults work?
2. **Vault encryption**: Does encrypt → decrypt round-trip correctly?
3. **Analyst logic**: Do metric column mappings return the right values?
4. **Format functions**: Does biometric data format correctly?

### What's Tested

From `tests/integration_tests.rs`:

```rust
#[test]
fn test_vault_encryption_roundtrip() {
    let vault = gorilla_coach::vault::Vault::new("12345678901234567890123456789012");
    let plaintext = "test_password_secret";
    let (ciphertext, nonce) = vault.encrypt(plaintext);
    let decrypted = vault.decrypt(&ciphertext, &nonce).unwrap();
    assert_eq!(plaintext, decrypted);
}
```

This test verifies the encryption layer without touching the database. The
Vault operates on strings — it doesn't know or care about database columns.

From `src/llm/analyst.rs`:

```rust
#[test]
fn test_metric_column_mapping() {
    assert_eq!(metric_column("hrv"), Some("hrv_last_night"));
    assert_eq!(metric_column("rhr"), Some("resting_heart_rate"));
    assert_eq!(metric_column("bogus"), None);
}
```

This test verifies the **safety-critical** `metric_column` function — the
boundary between LLM output and database columns. If this function returns an
incorrect column name, the analyst produces wrong data. If it returns `None`
for an unknown metric, the function errors safely.

### What's NOT Tested

No tests verify:
- SQL query correctness (does the upsert actually preserve existing data?)
- Migration application (do migrations run in order?)
- Connection pool behavior (do concurrent queries deadlock?)
- Data type conversions (does `FromRow` correctly map nullable columns?)

These would require integration tests with a real database. The pragmatic
approach: the application is the test. In a self-hosted, single-user context,
the developer IS the user and catches data issues immediately.

For a team-scale project, you'd add:

```rust
#[sqlx::test]
async fn test_upsert_preserves_existing_data(pool: PgPool) {
    let repo = Repository::from_pool(pool);
    // Insert row with steps=100, hrv=None
    repo.upsert_garmin_daily(&GarminDailyData { steps: Some(100), hrv_last_night: None, ... }).await.unwrap();
    // Upsert with steps=None, hrv=Some(65.0)
    repo.upsert_garmin_daily(&GarminDailyData { steps: None, hrv_last_night: Some(65.0), ... }).await.unwrap();
    // Verify both values are present
    let row = repo.get_garmin_daily(user_id, date).await.unwrap().unwrap();
    assert_eq!(row.steps, Some(100));           // preserved
    assert_eq!(row.hrv_last_night, Some(65.0)); // new value
}
```

> **ZtP**, §3.9 (Testing): *"sqlx::test provides a test database that is
> automatically created, migrated, and dropped for each test. Each test runs
> in a transaction that is rolled back, ensuring isolation."*

---

## 15. TimescaleDB: The Optional Superpower

### What TimescaleDB Adds

The `docker-compose.yaml` uses `timescale/timescaledb:latest-pg16`. TimescaleDB
is a PostgreSQL extension that adds:

- **Hypertables**: Automatic time-based partitioning. A hypertable looks like a
  regular table but is internally split into chunks by time range.
- **Continuous aggregates**: Materialized views that auto-refresh, perfect for
  precomputing daily/weekly/monthly summaries.
- **Compression**: Columnar compression for old data (10-20× space reduction).
- **Data retention policies**: Automatic deletion of old data by time range.

### Current Usage

Only `daily_logs` is converted to a hypertable (in the first migration):

```sql
DO $$ BEGIN
    PERFORM create_hypertable('daily_logs', 'time', if_not_exists => TRUE);
EXCEPTION WHEN OTHERS THEN
    RAISE NOTICE 'TimescaleDB not available, using plain Postgres table for daily_logs';
END $$;
```

`daily_logs` is a legacy table from the earliest version of the app. The active
biometric data table (`garmin_daily_data`) is **not** a hypertable. This is
because `garmin_daily_data` uses a composite primary key `(user_id, date)`, and
TimescaleDB hypertables work best with time as the primary partitioning
dimension. The `user_id` prefix in the key is more selective than `date` for
this application's query patterns (always filtering by user first).

### Potential Future Use

If Gorilla Coach stored higher-frequency data (per-minute heart rate, per-second
accelerometer data), TimescaleDB hypertables would become essential:

```sql
-- Hypothetical: per-minute heart rate data
CREATE TABLE heart_rate_samples (
    user_id UUID NOT NULL,
    time TIMESTAMPTZ NOT NULL,
    bpm INT NOT NULL
);
SELECT create_hypertable('heart_rate_samples', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Continuous aggregate: daily average HR
CREATE MATERIALIZED VIEW daily_hr_avg
WITH (timescaledb.continuous) AS
    SELECT user_id,
           time_bucket('1 day', time) AS day,
           avg(bpm) AS avg_bpm,
           min(bpm) AS min_bpm,
           max(bpm) AS max_bpm
    FROM heart_rate_samples
    GROUP BY user_id, day;
```

The continuous aggregate auto-refreshes, giving you instantly queryable daily
summaries without recomputing from raw data.

> **PiP**, Chapter 12 (Extensions): *"Extensions are PostgreSQL's mechanism for
> adding functionality without forking the core. TimescaleDB extends the query
> planner and storage engine to handle time-series workloads that would bring
> vanilla PostgreSQL to its knees at scale."*

---

## 16. What's Missing: Indexes, Constraints, and Gaps

The current schema is functional but has several gaps that would matter at
larger scale or with stricter requirements.

### Missing Indexes

**`chat_messages`** has no indexes at all. The `get_chat_history` query:

```sql
SELECT role, content FROM chat_messages
WHERE user_id = $1 ORDER BY time DESC LIMIT $2
```

On a table with N rows, this is a sequential scan filtered by `user_id`, then
sorted by `time`. At 100 users with 1,000 messages each, PostgreSQL scans
100K rows. At 1M users, it scans 200M rows. Adding an index:

```sql
CREATE INDEX idx_chat_messages_user_time
    ON chat_messages (user_id, time DESC);
```

This turns the sequential scan into an index scan that reads exactly `LIMIT`
rows from the index — O(limit) regardless of table size.

**`garmin_daily_data`** relies on its primary key `(user_id, date)`. This works
for all current queries (`WHERE user_id = $1 AND date >= $2 AND date <= $3`)
because the primary key's B-tree index supports range scans on the second
column after an equality match on the first.

**`manual_body_metrics`** likewise relies on `(user_id, date)` primary key,
which covers the only query pattern (`WHERE user_id = $1 AND date <= $2`).

### Missing Constraints

**No `CHECK` constraints on numeric ranges:**

```sql
-- These would be useful:
ALTER TABLE garmin_daily_data
    ADD CONSTRAINT chk_steps_positive CHECK (steps IS NULL OR steps >= 0),
    ADD CONSTRAINT chk_rhr_range CHECK (resting_heart_rate IS NULL OR resting_heart_rate BETWEEN 20 AND 250),
    ADD CONSTRAINT chk_sleep_score CHECK (sleep_score IS NULL OR sleep_score BETWEEN 0 AND 100);
```

Currently, the application trusts the Garmin API to return sensible values.
If the API returns `steps = -1` (which has been observed in the wild with other
fitness APIs), it's stored without validation.

**No `NOT NULL` on critical columns:**

`chat_messages.role` and `chat_messages.content` are nullable. Every message
has a role and content — these should probably be `NOT NULL`. But adding
`NOT NULL` retroactively requires ensuring no existing rows violate the
constraint:

```sql
ALTER TABLE chat_messages ALTER COLUMN role SET NOT NULL;
ALTER TABLE chat_messages ALTER COLUMN content SET NOT NULL;
```

This acquires an `ACCESS EXCLUSIVE` lock and scans the entire table. Safe
during a maintenance window, risky on a live system.

### Missing Audit Trail

There's no record of when data was created or modified (except `synced_at` on
`garmin_daily_data` and `created_at` on `manual_body_metrics`). For a fitness
coaching app, this isn't critical. For a system handling medical or financial
data, you'd add `created_at` and `updated_at` timestamps to every table.

### No Soft Deletes

Data deletion is hard delete — `DELETE FROM`. There's no `deleted_at`
timestamp for soft deletes. For the current use case (personal fitness data),
this is fine. Users who delete data actually want it gone.

> **FDK**, Chapter 7 (Constraints and Assertions): *"Constraints are the
> database's last line of defense. If the application allows bad data through
> (due to a bug, an edge case, or a schema mismatch), constraints catch it
> at the database level. Defense in depth applies to data quality just as it
> does to security."*

---

## 17. The Full Entity-Relationship Model

```mermaid
erDiagram
    users {
        UUID id PK
        TEXT email UK
        TSTZ created_at
    }
    user_settings {
        UUID user_id PK, FK
        TEXT garmin_username
        BYTEA enc_pass
        BYTEA nonce
        TEXT oauth_token
        TEXT google_refresh
        TSTZ last_sync_at
        TEXT mfa_star
    }
    chat_messages {
        UUID user_id FK
        TEXT role
        TEXT content
        TSTZ time
    }
    garmin_daily_data {
        UUID user_id PK, FK
        DATE date PK
        INT steps
        FLOAT hrv_last_night
        FLOAT sleep_score
        TEXT "...40 cols"
        TSTZ synced_at
    }
    manual_body_metrics {
        UUID user_id PK, FK
        DATE date PK
        FLOAT body_fat_pct
        TSTZ created_at
    }
    training_set_logs {
        UUID user_id PK, FK
        TEXT plan_file PK
        TEXT day_label PK
        TEXT exercise_name PK
        INT set_number PK
        FLOAT actual_weight
        INT actual_reps
        TEXT technique
        TSTZ logged_at
    }
    training_day_done {
        UUID user_id PK, FK
        TEXT plan_file PK
        TEXT day_label PK
        TSTZ marked_at
    }

    users ||--|| user_settings : "1:1 shared PK"
    users ||--o{ chat_messages : "1:many"
    users ||--o{ garmin_daily_data : "1:many"
    users ||--o{ manual_body_metrics : "1:many"
    users ||--o{ training_set_logs : "1:many"
    users ||--o{ training_day_done : "1:many"
```

**Relationships:**

- **users ↔ user_settings**: 1:1 (shared primary key)
- **users ↔ chat_messages**: 1:many (foreign key, no cascade)
- **users ↔ garmin_daily_data**: 1:many (composite PK includes user_id)
- **users ↔ manual_body_metrics**: 1:many (composite PK includes user_id)
- **users ↔ training_set_logs**: 1:many (5-column composite PK includes user_id)
- **users ↔ training_day_done**: 1:many (3-column composite PK includes user_id)
- **daily_logs**: Legacy table, same structure as early biometric logging. Now
  superseded by `garmin_daily_data`.

All foreign keys reference `users(id)` with no cascade behavior. Deleting a
user requires manually deleting all related rows first. This is intentional —
cascading deletes of years of biometric data should be explicit, not automatic.

---

## 18. Further Reading

### SQL and Database Fundamentals

- **The Art of PostgreSQL** — Dimitri Fontaine (2020).
  *The best book on thinking in SQL rather than ORMs. Covers window functions,
  CTEs, array operations, and JSON — all directly applicable to Gorilla Coach's
  biometric queries. The chapter on data modeling for time-series data is
  particularly relevant.*

- **Database Design and Relational Theory** — C.J. Date (2019).
  *The theoretical foundation: normalization, keys, constraints, functional
  dependencies. Explains why the garmin_daily_data wide table is a defensible
  design choice (denormalization for read performance) and when to normalize
  (eliminating update anomalies). Dense but precise.*

- **SQL Performance Explained** — Markus Winand (2012).
  *The definitive guide to indexing and query optimization. The B-tree chapter
  explains exactly why the (user_id, date) composite primary key works for
  range queries, and the EXPLAIN chapter teaches you to read query plans.
  Also available free at use-the-index-luke.com.*

### PostgreSQL Internals

- **PostgreSQL Internals: A Deep Dive** — Egor Rogov (2024).
  *How PostgreSQL works under the hood: MVCC, buffer management, WAL, vacuum,
  index internals. Essential for understanding why ADD COLUMN is O(1), why
  COALESCE upserts create new tuple versions, and why connection count matters.
  Read this when you need to optimize beyond the obvious.*

- **High Performance MySQL** — Schwartz, Zaitsev & Tkachenko, 4th Ed. (2021).
  *Despite the MySQL title, the indexing, schema design, and query optimization
  chapters are database-universal. The B-tree index chapter is clearer than
  most PostgreSQL-specific resources. The connection pooling chapter applies
  directly to PgBouncer.*

### Application Architecture

- **Patterns of Enterprise Application Architecture** — Martin Fowler (2002).
  *The origin of the Repository, Transaction Script, Data Mapper, and
  Active Record patterns. Chapter 9 (Transaction Script) describes exactly
  what Gorilla Coach's Repository does. Chapter 10 (Domain Model) describes
  what it would evolve into with more complex business logic.*

- **Zero To Production in Rust** — Luca Palmieri (2022).
  *Building a production web service in Rust with sqlx and Actix (patterns
  apply to Axum). The database chapter covers connection pooling,
  migrations, and the same Repository pattern used here. The testing
  chapter shows how to use sqlx::test for database integration tests.*

- **Designing Data-Intensive Applications** — Martin Kleppmann (2017).
  *The systems-level view: how relational databases compare to document
  stores, how replication and partitioning work, how transactions provide
  guarantees. Chapter 2 (Data Models) and Chapter 7 (Transactions) are
  directly relevant. Required reading for anyone designing a data layer.*

### Domain-Driven Design

- **Domain-Driven Design** — Eric Evans (2003).
  *The bounded context concept explains why user_settings, garmin_daily_data,
  and chat_messages belong to different domains but share a database. Chapter
  14 (Context Mapping) describes the relationships between these domains —
  the chat module is a downstream consumer of biometric and training data.*

### Rust-Specific Database Access

- **Rust for Rustaceans** — Jon Gjengset (2021).
  *Chapter 1 (Foundations) explains why PgPool can be Clone'd cheaply (Arc
  internals), why Mutex is needed for shared metrics, and why async database
  access requires Send + 'static bounds. Chapter 3 (Designing Interfaces)
  covers the trait patterns that make the Repository composable.*

- [**sqlx documentation**](https://docs.rs/sqlx/latest/sqlx/) — The official
  docs. The `query_as`, `QueryBuilder`, and `migrate!` macro documentation
  are essential references. The GitHub repository's examples show patterns
  for advanced use cases like custom type encoding and compile-time query
  checking.

---

## Closing Note

The database layer is the most conservative part of Gorilla Coach — and
that's exactly right. The LLM integration experiments with cutting-edge AI
models. The Garmin sync reverse-engineers an undocumented API. The UI delivers a
Dioxus Wasm PWA with reactive components. But the database is plain PostgreSQL with plain SQL
queries, plain parameterized bindings, and plain `FromRow` mapping.

This is deliberate. Databases outlive applications. The PostgreSQL schema and
the data in it will survive framework migrations, language rewrites, and
architectural shifts. The simpler and more standard the data layer, the more
future-proof the system.

> *"Data dominates. If you've chosen the right data structures and organized
> things well, the algorithms will almost always be self-evident."*
> — Rob Pike, *Notes on Programming in C* (1989)

> *"Show me your flowcharts and conceal your tables, and I shall continue to
> be mystified. Show me your tables, and I won't usually need your flowcharts;
> they'll be obvious."*
> — Fred Brooks, *The Mythical Man-Month* (1975)

---

*This tutorial is part of the Gorilla Coach documentation. For the Rust
language tutorial, see [RUST_TUTORIAL.md](RUST_TUTORIAL.md). For the
architecture patterns discussion, see
[ARCHITECTURE_PATTERNS_TUTORIAL.md](ARCHITECTURE_PATTERNS_TUTORIAL.md). For
scaling considerations including database scaling, see
[SCALING_TUTORIAL.md](SCALING_TUTORIAL.md). For the LLM agent architecture
(including the analyst's two-stage rocket), see
[LLM_AGENTS_TUTORIAL.md](LLM_AGENTS_TUTORIAL.md).*
