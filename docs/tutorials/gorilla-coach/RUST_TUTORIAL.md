# Building a Production Web Application in Rust

## A Deep-Dive Tutorial from First Principles to Advanced Patterns

*Using Gorilla Coach as a living case study — inspired by The Rust Programming Language, Zero to Production in Rust, and Rust for Rustaceans.*

---

## Table of Contents

- [Part I: Foundations](#part-i-foundations)
  - [Chapter 1: Why Rust for Web Services](#chapter-1-why-rust-for-web-services)
  - [Chapter 2: Cargo, Crates, and Project Structure](#chapter-2-cargo-crates-and-project-structure)
  - [Chapter 3: Ownership, Borrowing, and Lifetimes](#chapter-3-ownership-borrowing-and-lifetimes)
  - [Chapter 4: Structs, Enums, and Pattern Matching](#chapter-4-structs-enums-and-pattern-matching)
  - [Chapter 5: Error Handling — The Rust Way](#chapter-5-error-handling--the-rust-way)
  - [Chapter 6: Traits and Generics](#chapter-6-traits-and-generics)
  - [Chapter 7: Modules and Visibility](#chapter-7-modules-and-visibility)
- [Part II: Async Rust and the Web](#part-ii-async-rust-and-the-web)
  - [Chapter 8: Async/Await and Tokio](#chapter-8-asyncawait-and-tokio)
  - [Chapter 9: Axum — Building an HTTP Server](#chapter-9-axum--building-an-http-server)
  - [Chapter 10: Middleware — Tower, Layers, and Services](#chapter-10-middleware--tower-layers-and-services)
  - [Chapter 11: State Management and Dependency Injection](#chapter-11-state-management-and-dependency-injection)
  - [Chapter 12: Database Access with sqlx](#chapter-12-database-access-with-sqlx)
- [Part III: Production Patterns](#part-iii-production-patterns)
  - [Chapter 13: Cryptography — Encryption at Rest](#chapter-13-cryptography--encryption-at-rest)
  - [Chapter 14: Authentication and Session Management](#chapter-14-authentication-and-session-management)
  - [Chapter 15: Security Hardening — CSRF, Rate Limiting, CSP](#chapter-15-security-hardening--csrf-rate-limiting-csp)
  - [Chapter 16: Server-Sent Events and Streaming](#chapter-16-server-sent-events-and-streaming)
  - [Chapter 17: Background Workers and Graceful Shutdown](#chapter-17-background-workers-and-graceful-shutdown)
  - [Chapter 18: Configuration, CLI, and 12-Factor Design](#chapter-18-configuration-cli-and-12-factor-design)
  - [Chapter 19: Testing Without a Database](#chapter-19-testing-without-a-database)
  - [Chapter 20: Release Optimization and Docker](#chapter-20-release-optimization-and-docker)

---

## Part I: Foundations

### Chapter 1: Why Rust for Web Services

*(References: The Rust Book Ch. 1 "Getting Started"; Rust for Rustaceans Ch. 1 "Foundations"; Zero to Production §1.1 "Why Rust?")*

Rust occupies a unique niche in the web server space. Unlike Go, which trades expressiveness for simplicity, or Python, which trades performance for developer speed, Rust gives you **zero-cost abstractions**: the code you write compiles to machine code that is as fast as hand-written C, but with compile-time guarantees that eliminate entire classes of bugs.

For a self-hosted fitness coaching app like Gorilla Coach, this means:

- **Memory safety without garbage collection**: No GC pauses during SSE streaming to clients. When the Gorilla Coach streams LLM tokens to a browser over Server-Sent Events, each token is delivered on a predictable schedule — no 10ms GC pause breaks the smooth typing effect.
- **Thread safety at compile time**: The Rust compiler refuses to compile data races. When you have background sync workers, LLM streaming tasks, and HTTP handlers all running concurrently, the compiler verifies they can't corrupt shared state. Go achieves this with runtime panics and race detectors; Rust does it before you ever run the program.
- **Predictable performance**: A Garmin data sync that processes 42 biometric fields per day across 30 days completes in microseconds, not milliseconds. There's no JIT warmup, no interpreter overhead, no dynamic dispatch unless you explicitly opt into it.
- **Single binary deployment**: `cargo build --release` produces one binary per crate. The server binary (~15MB) is a complete web server with encryption, database drivers, HTTP client, and LLM integration. The Dioxus client compiles to a Wasm bundle served as static files.
- **Fearless concurrency**: The `Send` and `Sync` marker traits ensure that data is only accessed in thread-safe ways. The `Metrics` struct wraps in `Arc<Mutex<>>` — the type system *requires* this for shared mutable state. You can't accidentally share a `HashMap` across threads without synchronization.

The cost is the learning curve. Rust's ownership system, borrow checker, and trait system require a mental model shift. But as Zero to Production argues, "Rust makes the hard things possible and the impossible things compile errors." This tutorial walks you through that shift using real production code.

#### The Zero-Cost Abstraction Promise

"Zero-cost abstractions" (Stroustrup's principle, adopted by Rust) means: what you don't use, you don't pay for; what you use, you couldn't hand-code any better. Consider this actual code from the Gorilla Coach analyst:

```rust
let values: Vec<f64> = rows.iter()
    .filter_map(|d| extract_value(d, col, metric))
    .collect();
```

This chains an iterator adapter (`filter_map`) and collects into a `Vec`. In C, you'd write a loop with a null check and manual array management. The Rust version compiles to essentially the same code — the iterator adapters are inlined and optimized away by LLVM. The abstraction cost is zero at runtime; the readability benefit is immense.

---

### Chapter 2: Cargo, Crates, and Project Structure

*(References: The Rust Book Ch. 7 "Managing Growing Projects", Ch. 14 "More About Cargo"; Zero to Production §1.3 "Project Setup")*

Every Rust project starts with `Cargo.toml`. This is your project manifest — equivalent to `package.json` in Node or `pyproject.toml` in Python, but more powerful because it also controls compilation and optimization.

```toml
[package]
name = "gorilla_coach"
version = "2.0.0"
edition = "2021"

[profile.release]
lto = true        # Link Time Optimization (smaller binary, faster code)
codegen-units = 1 # Maximum optimization
strip = true      # Remove symbols
```

**Key concepts:**

- **`edition = "2021"`**: Rust uses editions (The Rust Book Appendix E) to introduce potentially-breaking changes without breaking old code. The 2021 edition enables disjoint capture in closures, `IntoIterator` for arrays, and more ergonomic pattern matching. Each crate opts into its edition independently — a crate using edition 2021 can depend on crates using edition 2018.
- **`[profile.release]`**: Release profiles control optimization (The Rust Book §14.1). Gorilla Coach aggressively optimizes the release binary:
  - `lto = true` — Link-Time Optimization. The compiler optimizes across crate boundaries, producing smaller and faster binaries at the cost of longer compile times. This is particularly valuable for an app that uses many crates (axum, sqlx, reqwest, serde) — LTO eliminates dead code and inlines across crate boundaries.
  - `codegen-units = 1` — Normally, the compiler splits each crate into 16 codegen units for parallel compilation. Reducing to 1 gives the optimizer visibility over the entire crate, enabling more aggressive inlining and dead code elimination.
  - `strip = true` — Remove DWARF debug symbols from the final binary, reducing binary size by ~50-70%.

#### Dependency Management with Feature Flags

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
axum = { version = "0.8", features = ["macros", "multipart"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-native-tls", "postgres", "chrono", "uuid", "migrate"] }
reqwest = { version = "0.12", features = ["json", "cookies", "stream"] }
```

Rust crates use **feature flags** (The Rust Book Ch. 14.3) to enable optional functionality. This is a zero-cost abstraction at the dependency level — you only compile what you need:

- `sqlx` with `["postgres", "chrono", "uuid", "migrate"]` gives you PostgreSQL support with type conversions for `chrono::DateTime` and `uuid::Uuid`, plus embedded migrations. It does NOT pull in MySQL, SQLite, or runtime features you don't need.
- `axum` with `["macros", "multipart"]` enables the `#[debug_handler]` proc macro (for better error messages during development) and multipart form parsing (needed for file uploads).
- `tokio` with `["full"]` enables all runtime features — in production, you'd be more selective, but for a single-binary server app, `full` is pragmatic.

Feature flags are additive — enabling a feature can never disable another. This is an invariant enforced by Cargo (Rust for Rustaceans §2.2) that prevents "feature hell" where combinations of features produce unexpected behavior.

#### The Binary/Library Split

*(References: Zero to Production §3.3 "The Binary vs Library Debate")*

Gorilla Coach uses the **binary + library** pattern recommended by Zero to Production:

```
src/
  main.rs       # Binary entry point — creates the server
  lib.rs        # Library root — exports all modules
  domain.rs     # Domain models
  repository.rs # Database access
  ...
```

`main.rs` is the executable. `lib.rs` is the library. The binary depends on the library:

```rust
// main.rs
use gorilla_coach::{
    config::{AppConfig, Cli, Commands},
    middleware::{csrf_middleware, RateLimiter},
    handlers,
    repository::Repository,
    state::{AppState, Metrics},
    vault::Vault,
    llm::LlmAdapter,
};
```

This separation enables two things:
1. **Integration tests** in `tests/` can import the library without starting the binary.
2. **Clean dependency graph** — `lib.rs` declares the public API, `main.rs` is just a thin entry point that wires everything together.

```rust
// gorilla_server/src/lib.rs
pub mod llm;
pub mod config;
pub mod error;
pub mod garmin;
pub mod handlers;
pub mod middleware;
pub mod reports;
pub mod repository;
pub mod state;
pub mod ui;     // legacy v1 server-rendered HTML
pub mod vault;
```

> **Note**: The project uses a three-crate Cargo workspace. Domain types and API
> contracts live in `gorilla_shared` (shared by both server and Dioxus Wasm
> client). `gorilla_server/src/lib.rs` above is the server crate's public API.
> The client crate (`gorilla_client`) compiles to WebAssembly and has its own
> module tree (`main.rs`, `api.rs`, `pages/`).

`pub mod` makes each module publicly accessible from outside the crate. The module tree in `lib.rs` is the crate's public API surface.

---

### Chapter 3: Ownership, Borrowing, and Lifetimes

*(References: The Rust Book Ch. 4 "Understanding Ownership"; Rust for Rustaceans Ch. 1 "Foundations" §1.1-1.3; Zero to Production §3.8 "Working with Shared State")*

This is the chapter that makes or breaks Rust developers. The ownership system is not an obstacle — it is a *design tool* that guides you toward correct concurrent code.

#### The Three Rules

1. **Each value has exactly one owner.** (The Rust Book §4.1)
2. **When the owner goes out of scope, the value is dropped (freed).** (The Rust Book §4.1)
3. **You can have either one mutable reference OR any number of immutable references, but not both.** (The Rust Book §4.2)

Rust for Rustaceans elaborates: these aren't arbitrary restrictions — they encode a fundamental truth about concurrent access. If multiple threads can read a value (shared references `&T`), no thread can write to it. If one thread can write (exclusive reference `&mut T`), no other thread can read or write. This eliminates data races at compile time.

#### Ownership in Practice: The Vault

```rust
pub struct Vault {
    cipher: ChaCha20Poly1305,
}

impl Vault {
    pub fn new(master_key: &str) -> Self {
        let key_bytes = master_key.as_bytes();
        // ...
        Self {
            cipher: ChaCha20Poly1305::new(chacha20poly1305::Key::from_slice(&key)),
        }
    }
}
```

`master_key: &str` is a **borrowed reference**. The function doesn't take ownership of the string — it just reads it. The `&` says "I'm borrowing this; the caller keeps ownership." This is critical: the `AppConfig` that holds the master key string isn't consumed when creating the Vault.

After construction, the `Vault` **owns** the `cipher`. When the `Vault` is dropped, the cipher's memory is zeroed and freed. No garbage collector needed. This is RAII (Resource Acquisition Is Initialization) — the same pattern C++ uses, but enforced by the compiler rather than convention.

#### Clone: Explicit Copies and Their Costs

*(References: Rust for Rustaceans §1.2 "References and Borrowing")*

When you need multiple owners, Rust makes you explicit about it:

```rust
#[derive(Clone)]
pub struct AppState {
    pub repo: Repository,
    pub cookie_key: Key,
    pub llm: Arc<dyn LlmAdapter>,
    pub metrics: Arc<Mutex<Metrics>>,
    // ...
}
```

`AppState` derives `Clone`. Every handler in Axum receives its own clone of the state. But what does "clone" actually mean here? Rust for Rustaceans (§1.2) distinguishes between **deep clones** and **shallow clones**:

| Field | Clone behavior | Cost |
|-------|---------------|------|
| `repo: Repository` | Wraps `PgPool` which is internally `Arc<Pool>` | O(1) — atomic increment |
| `cookie_key: Key` | Copies 64 bytes of key material | O(1) — stack copy |
| `llm: Arc<dyn LlmAdapter>` | Increments atomic reference count | O(1) — single atomic op |
| `metrics: Arc<Mutex<Metrics>>` | Increments atomic reference count | O(1) — single atomic op |
| `vault: Arc<Vault>` | Increments atomic reference count | O(1) — single atomic op |
| `config: AppConfig` | Deep clone of all strings in config | O(n) — allocations |
| `http_client: reqwest::Client` | Internally `Arc`, cheap clone | O(1) — atomic increment |

The key insight: **`Clone` on `Arc<T>` is O(1), not O(n)**. `Arc` (Atomic Reference Counter) stores a reference count alongside the data. Cloning increments the count; dropping decrements it. When the count reaches zero, the data is freed. All handlers share the *same* LLM adapter, metrics, and vault — they just have independent reference-counted pointers to it.

#### Borrowing and the Borrow Checker in Detail

*(References: The Rust Book §4.2 "References and Borrowing"; Rust for Rustaceans §1.3 "Lifetimes")*

```rust
pub fn encrypt(&self, data: &str) -> (String, String) {
    let mut nonce_bytes = [0u8; 12];
    rand::rng().fill_bytes(&mut nonce_bytes);
    let nonce = Nonce::from_slice(&nonce_bytes);
    let ciphertext = self.cipher
        .encrypt(nonce, data.as_bytes())
        .expect("Encryption failure");
    (
        BASE64_STANDARD.encode(ciphertext),
        BASE64_STANDARD.encode(nonce_bytes),
    )
}
```

Study the borrows here:
- `&self`: immutable borrow of the Vault. Multiple handlers can encrypt concurrently.
- `data: &str`: immutable borrow of the plaintext. The caller retains ownership.
- `&mut nonce_bytes`: mutable borrow. Only this function can mutate the nonce array.
- `Nonce::from_slice(&nonce_bytes)`: borrows the nonce bytes immutably (after the mutable borrow ends).

The mutable borrow of `nonce_bytes` in `fill_bytes(&mut nonce_bytes)` ends before the immutable borrow in `from_slice(&nonce_bytes)` begins. The borrow checker verifies this at compile time — you cannot accidentally read partially-written random bytes. This is what Rust for Rustaceans calls "temporal exclusivity": a mutable borrow doesn't just prevent simultaneous access — it prevents access until the borrow ends.

#### Lifetimes: When Borrows Need Names

*(References: The Rust Book Ch. 10.3 "Validating References with Lifetimes"; Rust for Rustaceans §1.3 "Lifetimes")*

Most of the time, Rust's **lifetime elision rules** (The Rust Book §10.3) infer lifetimes automatically. The compiler applies three rules:
1. Each input reference gets its own lifetime parameter.
2. If there's exactly one input lifetime, it's assigned to all output lifetimes.
3. If there's a `&self` or `&mut self` parameter, its lifetime is assigned to all output lifetimes.

But sometimes you need to be explicit:

```rust
fn metric_column(metric: &str) -> Option<&'static str> {
    match metric {
        "hrv" => Some("hrv_last_night"),
        "rhr" => Some("resting_heart_rate"),
        // ...
        _ => None,
    }
}
```

The return type is `&'static str` — a string reference that lives for the entire program. This is because the returned strings are **string literals**, compiled into the binary's read-only data segment. The `'static` lifetime tells the compiler: "this reference is always valid."

Without `'static`, the compiler would need to prove that the returned reference outlives the caller — and since the input `metric` is a borrow, the default lifetime elision would wrongly tie the output's lifetime to the input's lifetime.

This static mapping is also a critical **security boundary** — the column name returned to SQL can only ever be one of these compile-time constants. No user input reaches the SQL layer. More on this in the LLM Agent tutorial.

#### Move Semantics and `tokio::spawn`

*(References: The Rust Book §4.1 "What is Ownership?"; Rust for Rustaceans §1.1 "Memory Model")*

When you spawn an async task, the new task must own all its data:

```rust
let bg_state = state.clone();
tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(60 * 60));
    loop {
        interval.tick().await;
        match bg_state.repo.get_users_with_garmin().await {
            // ...
        }
    }
});
```

The `async move` block **moves** `bg_state` into the spawned task. The task takes ownership. This is required because `tokio::spawn` needs the future to be `'static` — it can't hold references to the caller's stack frame, which might be gone by the time the task runs. The signature is:

```rust
pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
```

The `'static` bound means the future cannot contain any non-`'static` borrows. The `Send` bound means it must be safe to move between threads.

`state.clone()` before the spawn creates a new `AppState`. Since `AppState`'s fields are `Arc`-wrapped, this is cheap. The spawned task gets its own reference-counted handle to the same database pool, LLM adapter, etc.

#### Interior Mutability: `Arc<Mutex<T>>` vs `Arc<tokio::sync::Mutex<T>>`

*(References: The Rust Book §15.5 "RefCell and Interior Mutability"; Rust for Rustaceans §1.4 "Interior Mutability")*

Rust's ownership rules say: shared XOR mutable. But sometimes you need shared *and* mutable access. This is where **interior mutability** patterns come in:

```rust
pub metrics: Arc<Mutex<Metrics>>,           // std::sync::Mutex
pub sa_token_cache: Arc<tokio::sync::Mutex<Option<(String, Instant)>>>,  // tokio::sync::Mutex
```

Both use `Mutex` for mutual exclusion, but they're different implementations for different use cases:

**`std::sync::Mutex`** (used for `metrics`):
- Blocks the OS thread while waiting for the lock.
- Efficient when the critical section is tiny and never `await`s.
- The `Metrics` update is a few integer increments — nanoseconds of work.
- Using `tokio::sync::Mutex` here would add unnecessary overhead (yielding to the scheduler when the lock is already available).

**`tokio::sync::Mutex`** (used for `sa_token_cache`):
- Yields the async task to the runtime while waiting for the lock.
- Required when you need to hold the lock across `.await` points.
- The token cache might be held while fetching a Google OAuth token — an HTTP request that could take seconds.
- Using `std::sync::Mutex` here would block an entire Tokio worker thread, potentially causing deadlocks with enough concurrent requests.

Rust for Rustaceans explains this distinction as the difference between "blocking" and "non-blocking" synchronization. The rule is simple: **if you `await` inside the critical section, use `tokio::sync::Mutex`. Otherwise, use `std::sync::Mutex`.**

---

### Chapter 4: Structs, Enums, and Pattern Matching

*(References: The Rust Book Ch. 5 "Using Structs", Ch. 6 "Enums and Pattern Matching"; Rust for Rustaceans §2.1 "Types in Depth")*

#### Structs with Derive Macros

```rust
#[derive(Debug, Serialize, Deserialize, FromRow, Clone)]
pub struct GarminDailyData {
    pub user_id: Uuid,
    pub date: NaiveDate,
    pub steps: Option<i64>,
    pub distance_meters: Option<f64>,
    pub active_calories: Option<i64>,
    // ... 40+ fields
    pub synced_at: DateTime<Utc>,
}
```

The derive macros are doing heavy lifting:
- **`Debug`**: Enables `{:?}` formatting for logging. The compiler generates a `fmt::Debug` implementation that prints every field.
- **`Serialize, Deserialize`**: serde generates JSON serialization/deserialization code at compile time. No reflection, no runtime overhead.
- **`FromRow`**: sqlx generates code to map a SQL result row to this struct. Each field name must match a column name.
- **`Clone`**: Generates a deep copy. For 42 `Option<T>` fields, this is straightforward — `Option<i64>` implements `Copy`, so cloning is memcpy.

The `#[sqlx(default)]` attribute on `Option` fields tells sqlx to use `Default::default()` (which is `None` for `Option`) when a column is missing from the query result. This is essential for schema evolution — you can add new columns without breaking existing queries.

#### Enums: Rust's Killer Feature — Sum Types

*(References: The Rust Book Ch. 6 "Enums and Pattern Matching"; Rust for Rustaceans §2.1 "Types in Depth")*

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("LLM error: {0}")]
    Llm(String),
    #[error("Sheets API error: {0}")]
    Sheets(String),
    #[error("Garmin API error: {0}")]
    Garmin(String),
    #[error("Auth error: {0}")]
    Auth(String),
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}
```

Rust enums are **tagged unions** — also called algebraic data types (ADTs) or sum types (Rust for Rustaceans §2.1). Each variant can carry different data. `AppError::Llm` carries a `String`, `AppError::Internal` carries an `anyhow::Error`.

The `#[from]` attribute on `Internal` generates a `From<anyhow::Error>` implementation. This means any function returning `Result<T, AppError>` can use the `?` operator on `anyhow::Result<T>`:

```rust
pub async fn migrate(&self) -> anyhow::Result<()> {
    sqlx::migrate!("./migrations").run(&self.pool).await?;
    Ok(())
}
```

The `?` operator: if the `Result` is `Err`, it converts the error using `From` and returns early. If `Ok`, it unwraps the value. This replaces entire pages of try-catch blocks in other languages.

#### Pattern Matching: Exhaustiveness and Guards

*(References: The Rust Book Ch. 18 "Patterns and Matching")*

```rust
let llm: Arc<dyn LlmAdapter> = match cfg.llm_provider.as_str() {
    "ollama" => {
        tracing::info!("LLM: primary=ollama");
        Arc::new(FallbackLlmAdapter::new(ollama_adapter, gemini_adapter))
    }
    _ => {
        tracing::info!("LLM: primary=gemini");
        Arc::new(FallbackLlmAdapter::new(gemini_adapter, ollama_adapter))
    }
};
```

Rust's `match` is exhaustive — the compiler verifies every possible case is handled. The `_` wildcard catches everything else. If you add a new variant to an enum and forget to handle it, the code **won't compile**.

More powerful pattern matching with `Option`:

```rust
let (act_date, act) = match last_activity {
    Some(a) => a,
    None => return Ok("No activities found in the last 7 days.".into()),
};
```

This destructures the `Option`, extracting the inner value if `Some`, or returning early if `None`. The compiler ensures you handle both cases.

---

### Chapter 5: Error Handling — The Rust Way

*(References: The Rust Book Ch. 9 "Error Handling"; Zero to Production §5 "Going Live" — error handling in web apps; Rust for Rustaceans §4.1 "Error Handling")*

Rust has no exceptions. No try-catch. No null pointer exceptions at runtime. Instead, it uses the type system to make errors **visible and mandatory to handle**.

#### The Two Error Types: `Result<T, E>` and `Option<T>`

```rust
// Option: value that might not exist
pub async fn get_garmin_daily(&self, user_id: Uuid, date: NaiveDate)
    -> anyhow::Result<Option<GarminDailyData>>
```

This return type says: "I might fail (database error), and even if I succeed, there might be no data for that date." The caller must handle both cases — the compiler enforces it.

#### `anyhow` vs `thiserror`: Application vs Library Error Handling

*(References: Zero to Production §5.3 "Error Handling in Web Applications"; Rust for Rustaceans §4.1)*

**`thiserror`** is for **defining** error types with clean display messages:

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("LLM error: {0}")]
    Llm(String),
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}
```

**`anyhow`** is for **propagating** errors without caring about the specific type:

```rust
pub async fn upsert_garmin_daily(&self, d: &GarminDailyData) -> anyhow::Result<()> {
    sqlx::query(/* ... */)
        .execute(&self.pool)
        .await?;
    Ok(())
}
```

The `?` operator here converts `sqlx::Error` → `anyhow::Error` automatically via the `From` trait. The repository layer doesn't need to know about `AppError` — it just says "something went wrong" and lets the caller decide how to handle it.

Rust for Rustaceans and Zero to Production converge on this rule: **use `thiserror` at your API boundary (handlers, public types) and `anyhow` internally (repository, service logic)**. The reasoning:

- The **repository layer** doesn't need to distinguish between a connection timeout and a constraint violation from the handler's perspective. It just says "something went wrong" via `anyhow::Error`, which captures the full error chain.
- The **handler layer** needs domain-specific error types (LLM error, auth error, garmin error) to decide what HTTP response to return. `AppError` provides this via explicit variants.
- The `#[from] anyhow::Error` on `Internal` bridges the two: `?` on an `anyhow::Result` automatically converts to `AppError::Internal`, carrying the full error context.

#### Convenience Constructors with `impl Into<String>`

```rust
impl AppError {
    pub fn llm(msg: impl Into<String>) -> Self {
        Self::Llm(msg.into())
    }
}
```

The `impl Into<String>` accepts any type that can be converted into a `String` — both `&str` and `String`. This avoids forcing callers to allocate:

```rust
// Both work:
AppError::llm("static message")        // &str, converted lazily
AppError::llm(format!("dynamic {}", x)) // String, moved in
```

#### The Error Flow in Gorilla Coach

In Gorilla Coach, errors flow like this:

```
Repository        → anyhow::Result<T>  (generic, catches all DB errors)
    ↓ (? operator)
Handler           → AppError            (converted via From<anyhow::Error>)
    ↓                                   (or explicit AppError::llm("msg"))
Response          → Json / Html          (v2 returns JSON ApiError; v1 renders HTML)
```

This layered approach means the database layer doesn't know about HTTP, and the handlers don't know about SQL error types. Each layer has the right level of error granularity.

---

### Chapter 6: Traits and Generics

*(References: The Rust Book Ch. 10 "Generic Types, Traits, and Lifetimes"; Rust for Rustaceans Ch. 3 "Designing Interfaces"; Zero to Production §3.7 "Working with Traits")*

Traits are Rust's mechanism for polymorphism. They define shared behavior — like interfaces in Java/Go, but more powerful because they support default implementations, associated types, and blanket implementations.

#### Defining a Trait: The LLM Adapter

```rust
#[async_trait]
pub trait LlmAdapter: Send + Sync {
    async fn generate(&self, prompt: &str, system_instruction: &str)
        -> Result<String, AppError>;

    fn model_name(&self) -> String;

    async fn health_check(&self) -> bool {
        false  // default implementation
    }

    async fn generate_stream(
        &self,
        prompt: &str,
        system_instruction: &str,
        tx: mpsc::Sender<Result<String, String>>,
    ) {
        // Default: fall back to non-streaming
        let result = self.generate(prompt, system_instruction).await;
        let _ = tx.send(result.map_err(|e| e.to_string())).await;
    }

    async fn generate_with_tools(
        &self,
        prompt: &str,
        system_instruction: &str,
        _tools: &[ToolDef],
        _executor: &(dyn ToolExecutor + Sync),
    ) -> Result<String, AppError> {
        self.generate(prompt, system_instruction).await
    }
}
```

**Unpacking this:**

- **`Send + Sync` supertraits**: Any type implementing `LlmAdapter` must be safe to send between threads (`Send`) and safe to share between threads (`Sync`). This is required because Axum handlers run on a multi-threaded Tokio runtime. `Send + Sync` bounds propagate (Rust for Rustaceans §3.2): if `LlmAdapter: Send + Sync`, then `Arc<dyn LlmAdapter>` is also `Send + Sync`. Without these bounds, you couldn't put the adapter in `AppState` (which is cloned across threads).
- **`#[async_trait]`**: Rust's traits don't natively support `async fn` with dynamic dispatch (Rust for Rustaceans §3.4). The `async_trait` crate transforms `async fn` into `fn -> Pin<Box<dyn Future>>`. This adds one heap allocation per trait method call — a negligible cost for network-bound I/O that takes milliseconds. Rust 1.75+ has native `async fn in trait`s, but they don't support `dyn Trait` yet — `async_trait` remains the pragmatic choice when you need trait objects.
- **Default implementations**: `health_check`, `generate_stream`, and `generate_with_tools` have default implementations. `OllamaAdapter` only needs to implement `generate` and `model_name` to be a valid `LlmAdapter` — the defaults handle the rest. This is the **progressive enhancement** pattern — new capabilities (streaming, tools) are opt-in.

#### Implementing a Trait

```rust
#[async_trait]
impl LlmAdapter for GeminiAdapter {
    async fn generate(&self, prompt: &str, system_instruction: &str)
        -> Result<String, AppError>
    {
        // 300 lines of model rotation, retry logic, response parsing...
    }

    fn model_name(&self) -> String {
        let idx = self.model_index.load(Ordering::Relaxed) % self.models.len();
        self.models[idx].clone()
    }

    async fn health_check(&self) -> bool {
        // Override the default: actually check Gemini API
        let url = format!(".../{}", self.models[idx]);
        matches!(req.send().await, Ok(r) if r.status().is_success())
    }

    // generate_stream and generate_with_tools also overridden...
}
```

#### Trait Objects: Dynamic Dispatch

*(References: The Rust Book §17.2 "Using Trait Objects"; Rust for Rustaceans §2.3 "Trait Bounds" and §3.4 "Object Safety")*

```rust
pub llm: Arc<dyn LlmAdapter>,
```

`dyn LlmAdapter` is a **trait object** — a fat pointer containing:
- A data pointer to the concrete value on the heap
- A vtable pointer to a table of function pointers for the trait methods

The vtable for `GeminiAdapter` contains pointers to `GeminiAdapter::generate`, `GeminiAdapter::model_name`, etc. Method calls go through the vtable at runtime — this is **dynamic dispatch**, similar to virtual methods in C++.

**Object safety** (Rust for Rustaceans §3.4): Not all traits can become trait objects. The trait must be "object safe":
- No `Self: Sized` bound on the trait itself
- No methods that return `Self` (the concrete type is erased)
- No methods with generic type parameters (they'd require monomorphization, defeating dynamic dispatch)

`LlmAdapter` is object safe because none of its methods are generic or return `Self`.

`Arc<dyn LlmAdapter>` combined with `Send + Sync` means: "a thread-safe, reference-counted pointer to any type that implements `LlmAdapter`." At runtime, this could be a `GeminiAdapter`, `OllamaAdapter`, or `FallbackLlmAdapter` — the code using it doesn't care.

This is how we achieve runtime polymorphism:

```rust
let llm: Arc<dyn LlmAdapter> = match cfg.llm_provider.as_str() {
    "ollama" => Arc::new(FallbackLlmAdapter::new(ollama_adapter, gemini_adapter)),
    _ => Arc::new(FallbackLlmAdapter::new(gemini_adapter, ollama_adapter)),
};
```

The same `llm` variable holds different concrete types depending on configuration. The handler code just calls `llm.generate(...)` and the right implementation runs.

#### The Decorator Pattern via Trait Composition

*(References: Zero to Production §3.7; Rust for Rustaceans §3.3 "Composition with Traits")*

`FallbackLlmAdapter` wraps two other `LlmAdapter` implementations:

```rust
pub struct FallbackLlmAdapter {
    primary: Arc<dyn LlmAdapter>,
    fallback: Arc<dyn LlmAdapter>,
}

#[async_trait]
impl LlmAdapter for FallbackLlmAdapter {
    async fn generate(&self, prompt: &str, system_instruction: &str)
        -> Result<String, AppError>
    {
        match self.primary.generate(prompt, system_instruction).await {
            Ok(result) => Ok(result),
            Err(e) if is_connection_error(&e) => {
                tracing::warn!("Primary unavailable, falling back");
                self.fallback.generate(prompt, system_instruction).await
            }
            Err(e) => Err(e),
        }
    }
}
```

This is the **decorator pattern** (Gang of Four, realized through Rust traits). `FallbackLlmAdapter` *is* an `LlmAdapter` that wraps other `LlmAdapter`s. The handler code doesn't know it's talking to a fallback — it just sees `Arc<dyn LlmAdapter>`. You could stack decorators: a rate-limiting adapter wrapping a caching adapter wrapping a fallback adapter wrapping concrete providers.

Note the guard pattern in `Err(e) if is_connection_error(&e)` — this is a **match guard**. It only matches the `Err` variant if the error is a connection error. Other errors (like API key invalid) propagate without fallback.

#### Generics vs Trait Objects: When to Use Which

*(References: Rust for Rustaceans §2.3 "Trait Bounds")*

- **Generics** (`fn foo<T: LlmAdapter>(adapter: T)`) use **monomorphization** — the compiler generates a separate copy of the function for each concrete type. Zero runtime overhead (direct function calls), but the concrete type must be known at compile time. Binary size grows with each instantiation.
- **Trait objects** (`fn foo(adapter: &dyn LlmAdapter)`) use **dynamic dispatch** — one copy of the function, vtable lookup at runtime. Slight overhead (~1 ns per call), but allows runtime polymorphism. Binary stays small.

Gorilla Coach uses trait objects because the LLM adapter is chosen at runtime based on configuration. You can't monomorphize when the type depends on an environment variable.

---

### Chapter 7: Modules and Visibility

*(References: The Rust Book Ch. 7 "Managing Growing Projects with Packages, Crates, and Modules"; Rust for Rustaceans §2.2 "Crate Organization")*

Rust's module system is directory-based with explicit declarations. Unlike Python (where any `.py` file is automatically a module), Rust requires you to declare every module explicitly.

#### Flat Modules

```rust
// lib.rs
pub mod domain;    // loads gorilla_shared/src/domain.rs (via dependency)
pub mod error;     // loads src/error.rs
pub mod vault;     // loads src/vault.rs
```

Each `pub mod name;` declaration tells the compiler to look for `src/name.rs` and make it publicly accessible.

#### Nested Module Directories

```rust
// lib.rs
pub mod llm;       // loads src/llm/mod.rs (directory module)
pub mod handlers;  // loads src/handlers/mod.rs
pub mod middleware; // loads src/middleware/mod.rs
```

When a module has sub-modules, you create a directory with `mod.rs`:

```
src/llm/
  mod.rs        # Module root — re-exports public API
  adapter.rs    # Trait definitions
  gemini.rs     # Gemini implementation
  ollama.rs     # Ollama implementation
  fallback.rs   # Fallback wrapper
  analyst.rs    # Two-stage analyst
  utils.rs      # Retry helpers
```

```rust
// src/llm/mod.rs
mod adapter;
mod gemini;
mod ollama;
mod fallback;
mod utils;
pub mod analyst;

pub use adapter::{LlmAdapter, ToolDef, ToolExecutor};
pub use gemini::GeminiAdapter;
pub use ollama::OllamaAdapter;
pub use fallback::FallbackLlmAdapter;
pub use analyst::{detect_intent, get_metric_stats, ALLOWED_METRICS};
```

**Visibility rules:**
- `mod adapter;` (no `pub`): private to the `llm` module. External code can't `use gorilla_coach::llm::adapter`.
- `pub use adapter::LlmAdapter;`: re-exports `LlmAdapter` at the `llm` module level. External code uses `gorilla_coach::llm::LlmAdapter`.
- `pub mod analyst;`: makes the entire `analyst` module public, including its internal symbols.

This pattern — private modules with selective re-exports — is what Zero to Production calls the **facade pattern**: external consumers see a flat API (`llm::LlmAdapter`, `llm::GeminiAdapter`) while the implementation is organized into focused files. Adding a new adapter (e.g., `ClaudeAdapter`) means adding `src/llm/claude.rs` and one `pub use` line in `mod.rs`.

#### The `crate::` Path Prefix

```rust
use crate::error::AppError;
use crate::domain::AnalystIntent;
```

`crate::` is the absolute path to the crate root. It's like `/` in a filesystem. This is preferred over relative paths (`super::`) for clarity, especially in deeply nested modules.

---

## Part II: Async Rust and the Web

### Chapter 8: Async/Await and Tokio

*(References: The Rust Book Ch. 17 "Async and Await" (edition 2024 preview); Rust for Rustaceans Ch. 8 "Asynchronous Interfaces"; Zero to Production §2.3 "Runtime: Tokio")*

Understanding async Rust is essential for web development. Every HTTP handler, database query, and LLM call is asynchronous.

#### What `async` Actually Does

*(References: Rust for Rustaceans §8.1 "How Async Works")*

When you write:

```rust
async fn get_garmin_daily(&self, user_id: Uuid, date: NaiveDate)
    -> anyhow::Result<Option<GarminDailyData>>
{
    let row = sqlx::query_as::<_, GarminDailyData>(
        "SELECT * FROM garmin_daily_data WHERE user_id = $1 AND date = $2"
    )
    .bind(user_id).bind(date)
    .fetch_optional(&self.pool)
    .await?;
    Ok(row)
}
```

The compiler transforms this into a **state machine**. The `async fn` doesn't execute immediately — it returns a `Future` that, when polled, advances through states:

1. **State 0**: Construct the SQL query, send it to the database.
2. **Suspended**: Yield control to the runtime while waiting for the database response.
3. **State 1**: Database responded. Parse the row, return the result.

The `.await` points are where suspension can happen. Between `.await` points, the code runs synchronously on whichever thread the runtime scheduled it on.

**Critical implication**: anything held across an `.await` must be `Send` (safe to move between threads) because the runtime may resume the task on a different thread. This is why `std::sync::MutexGuard` can't be held across `.await` — it's not `Send`. Use the scoping trick:

```rust
// WRONG: MutexGuard held across await
let guard = state.metrics.lock().unwrap();
guard.total_requests += 1;  // Won't compile if followed by .await

// RIGHT: Drop guard before await
{
    let mut guard = state.metrics.lock().unwrap();
    guard.total_requests += 1;
} // guard dropped here
do_async_work().await;  // Now safe
```

#### Tokio: The Async Runtime

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // ...
}
```

`#[tokio::main]` is a macro that expands to:

```rust
fn main() -> anyhow::Result<()> {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async { /* your code */ })
}
```

Tokio's multi-threaded runtime uses **work-stealing** (Rust for Rustaceans §8.2): multiple OS threads (default: one per CPU core), each with a local task queue. When one thread runs out of work, it steals tasks from another thread's queue. This distributes load automatically without manual thread management.

#### `tokio::spawn`: Detached Tasks

```rust
tokio::spawn(async move {
    let result = llm.generate_with_tools(&prompt, &system, &tools, &executor).await;
    match result {
        Ok(response) => {
            for chunk in response.chars().collect::<Vec<_>>().chunks(20) {
                let token: String = chunk.iter().collect();
                if tx.send(Ok(token)).await.is_err() { break; }
            }
        }
        Err(e) => { let _ = tx.send(Err(e.to_string())).await; }
    }
});
```

`tokio::spawn` creates a new task on the runtime. This task runs **independently** of the calling task. If the HTTP client disconnects, the spawned LLM task keeps running until completion — this is intentional, so partial results still get saved to the database.

The `async move` closure takes ownership of all captured variables. The `'static` bound on `tokio::spawn` means the future cannot hold any borrowed references — everything must be owned or `Arc`'d.

#### Channels: `mpsc` and `broadcast`

*(References: Rust for Rustaceans §8.3 "Asynchronous Communication")*

Gorilla Coach's streaming handler uses a two-stage channel architecture:

```rust
// mpsc: many producers, single consumer
let (tx, mut rx) = tokio::sync::mpsc::channel::<Result<String, String>>(64);

// broadcast: one producer, many consumers
let (btx, mut save_rx) = tokio::sync::broadcast::channel::<Result<String, String>>(256);
```

The LLM task sends tokens into the `mpsc` channel. A forwarder task reads from `mpsc` and broadcasts to multiple consumers: the SSE stream (sends to browser) and the saver (persists to database). This ensures the response is saved even if the client disconnects mid-stream.

```
LLM Task ──→ mpsc tx ──→ mpsc rx ──→ broadcast tx ──→ SSE Stream (client)
                                                   └─→ Saver Task (database)
```

This is a form of the **fan-out pattern**, implemented with zero-copy message passing.

---

### Chapter 9: Axum — Building an HTTP Server

*(References: Zero to Production §3 "Sign Up a New Subscriber" — the Axum/Actix comparison; §5 "Going Live")*

Axum is a web framework built on `tower` (the middleware stack) and `hyper` (the HTTP implementation). It's designed from the ground up for Rust's async ecosystem.

#### Routing

```rust
let app = Router::new()
    .route("/", get(handlers::chat_page))
    .route("/dashboard", get(handlers::dashboard_page))
    .route("/api/settings", post(handlers::update_settings))
    .route("/api/files/{name}", get(handlers::files_serve).delete(handlers::files_delete))
    // ...
    .with_state(state.clone());
```

Routes map HTTP methods to handler functions. `get(handler)` creates a GET route, `post(handler)` creates a POST route. You can chain methods: `get(serve).delete(delete)` handles both GET and DELETE on the same path.

`{name}` is a path parameter — Axum extracts it automatically.

#### Extractors: Type-Driven Request Parsing

*(References: Zero to Production §3.5 "Extractors")*

Axum handlers declare what they need as function parameters. The framework extracts each parameter from the request automatically:

```rust
pub async fn my_handler(
    State(state): State<AppState>,        // Extracts shared application state
    jar: SignedCookieJar,                  // Extracts signed cookies from request
    Form(input): Form<MyForm>,            // Extracts URL-encoded form data
) -> impl IntoResponse { ... }
```

Each parameter type implements `FromRequest` or `FromRequestParts`. Axum calls these extractors in order:

1. `State<AppState>`: pulls from the router's `with_state()`.
2. `SignedCookieJar`: reads the `Cookie` header, verifies HMAC signatures.
3. `Form<MyForm>`: reads and deserializes the request body.

**Critical rule**: Body-consuming extractors (`Form`, `Multipart`, `Json`) must be the **last** parameter. The body can only be consumed once. If you put `Multipart` before `State`, the compiler produces a confusing error — this is why `#[debug_handler]` exists: it produces clear, human-readable error messages.

#### The `impl IntoResponse` Return Type

Handlers return `impl IntoResponse` — any type that can be converted into an HTTP response:

```rust
pub async fn chat_page(...) -> impl IntoResponse {
    ui::chat_page(&messages)  // Returns Html<String>
}
```

String, `Html<String>`, `(StatusCode, String)`, `Redirect`, `Response` — all implement `IntoResponse`. This gives handlers flexibility without a fixed return type.

#### Splitting the Router

```rust
// Auth routes with stricter rate limiting
let auth_router = Router::new()
    .route("/auth/login", get(handlers::login_page).post(handlers::login))
    .route("/auth/logout", post(handlers::logout))
    .layer(axum::middleware::from_fn(auth_limiter.into_layer()))
    .with_state(state.clone());

// SSE streaming — no compression (prevents browsers from buffering)
let stream_router = Router::new()
    .route("/api/chat/stream", post(handlers::chat_stream_handler))
    .with_state(state.clone());

// Main router with compression
let app = Router::new()
    .route("/", get(handlers::chat_page))
    // ... many routes ...
    .layer(CompressionLayer::new())
    .with_state(state.clone())
    .merge(stream_router)     // SSE routes bypass compression
    .merge(auth_router);       // Auth routes have their own rate limiter
```

`merge()` combines routers. Routes in `stream_router` don't get the `CompressionLayer` (SSE doesn't work well with gzip). Routes in `auth_router` get their own rate limiter.

This is **layer composition** — different route groups can have different middleware stacks.

---

### Chapter 10: Middleware — Tower, Layers, and Services

*(References: Zero to Production §4.3 "Middleware"; Rust for Rustaceans §8.4 "Tower and the Service Trait")*

Tower is the middleware abstraction that Axum is built on. Understanding it is key to writing custom middleware.

#### The Service Trait (Conceptual)

At its core, a Tower Service is:

```rust
// Simplified for understanding
trait Service<Request> {
    type Response;
    type Error;
    fn call(&self, req: Request) -> Future<Output = Result<Response, Error>>;
}
```

A service takes a request and returns a future that resolves to a response. Middleware wraps one service to produce another service.

#### Writing Middleware as Functions

Axum provides `middleware::from_fn` for simple middleware:

```rust
async fn trace_id_middleware(
    req: Request<axum::body::Body>,
    next: Next,
) -> impl IntoResponse {
    let trace_id = Uuid::new_v4().to_string();
    let mut res = next.run(req).await;
    res.headers_mut().insert("X-Trace-Id", trace_id.parse().unwrap());
    res
}
```

The pattern:
1. Receive the request and `Next` (the inner service).
2. Optionally modify the request.
3. Call `next.run(req).await` to execute the inner service.
4. Optionally modify the response.
5. Return the response.

#### CSRF Middleware: A Real Security Layer

```rust
pub async fn csrf_middleware(
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let dominated = matches!(
        *request.method(),
        Method::POST | Method::PUT | Method::DELETE | Method::PATCH
    );

    if !dominated {
        return Ok(next.run(request).await);
    }

    let headers = request.headers();
    let host = match extract_host(headers) {
        Some(h) => h,
        None => return Err(StatusCode::FORBIDDEN),
    };

    if let Some(origin) = headers.get("origin").and_then(|v| v.to_str().ok()) {
        if let Some(origin_host) = origin_to_host(origin) {
            if origin_host == host {
                return Ok(next.run(request).await);
            }
        }
        return Err(StatusCode::FORBIDDEN);
    }

    // Fall through: no Origin or Referer — allow (OWASP relaxed stance)
    Ok(next.run(request).await)
}
```

This implements OWASP's recommended CSRF protection: verify that the `Origin` header (or `Referer` fallback) matches the expected `Host`. GET/HEAD/OPTIONS are always allowed (they should be idempotent).

The return type `Result<Response, StatusCode>` means the middleware can short-circuit — returning `Err(StatusCode::FORBIDDEN)` stops the request before it reaches the handler.

#### Rate Limiting: Stateful Middleware

```rust
pub struct RateLimiter {
    state: Arc<Mutex<HashMap<std::net::IpAddr, (u32, Instant)>>>,
    max_requests: u32,
    window: Duration,
}

impl RateLimiter {
    pub fn check(&self, ip: IpAddr) -> bool {
        let now = Instant::now();
        let mut map = self.state.lock().unwrap();

        // Periodic cleanup
        if map.len() > 1000 {
            let cutoff = now - self.window * 2;
            map.retain(|_, (_, start)| *start > cutoff);
        }

        let entry = map.entry(ip).or_insert((0, now));
        if now.duration_since(entry.1) > self.window {
            *entry = (1, now);
            true
        } else {
            entry.0 += 1;
            entry.0 <= self.max_requests
        }
    }
}
```

The rate limiter uses `Arc<Mutex<HashMap<IpAddr, (count, window_start)>>>`. It's a per-IP sliding window counter. Each request locks the mutex, checks/updates the counter, and releases.

**Why `Mutex` and not `RwLock`?** Every request mutates the map (incrementing the counter), so there's no read-only path. `Mutex` is simpler and faster for write-heavy workloads.

**Why `std::sync::Mutex` not `tokio::sync::Mutex`?** The critical section is tiny (a HashMap lookup and increment). It never awaits inside the lock. Tokio's mutex exists for when you need to hold a lock across `.await` points — we don't, so the non-async mutex is faster (no overhead of yielding to the scheduler).

#### Layer Ordering

```rust
.layer(axum::middleware::from_fn(csrf_middleware))
.layer(axum::middleware::from_fn(security_headers_middleware))
.layer(axum::middleware::from_fn(trace_id_middleware))
.layer(TraceLayer::new_for_http())
.layer(CompressionLayer::new())
```

**Layers execute in reverse order** for requests, forward order for responses. This means:

Request flow: Compression → TraceLayer → trace_id → security_headers → CSRF → Handler

Response flow: Handler → CSRF → security_headers → trace_id → TraceLayer → Compression

The CSRF check happens last before the handler (first in the list), meaning it runs after tracing and security headers are already set up.

---

### Chapter 11: State Management and Dependency Injection

*(References: Zero to Production §3.8 "Working with Shared State"; Rust for Rustaceans §1.4 "Interior Mutability")*

In a web application, you need shared resources: database connections, API clients, configuration. Axum uses a simple but powerful pattern.

#### AppState: The Central Nerve

```rust
#[derive(Clone)]
pub struct AppState {
    pub repo: Repository,
    pub cookie_key: Key,
    pub llm: Arc<dyn LlmAdapter>,
    pub metrics: Arc<Mutex<Metrics>>,
    pub llm_logger: LlmLogger,
    pub vault: Arc<Vault>,
    pub config: AppConfig,
    pub http_client: reqwest::Client,
    pub sa_token_cache: Arc<tokio::sync::Mutex<Option<(String, Instant)>>>,
}
```

**Why each field is wrapped the way it is:**

| Field | Wrapper | Reason |
|-------|---------|--------|
| `repo` | None | `PgPool` is internally `Arc`-wrapped. `Clone` is cheap. |
| `llm` | `Arc<dyn T>` | Trait object needs heap allocation. `Arc` for shared ownership. |
| `metrics` | `Arc<Mutex<T>>` | Mutable shared state. Mutex for exclusive access. |
| `vault` | `Arc<T>` | Shared across handlers. Never mutated. |
| `config` | None | `Clone` derives a deep copy. Read-only after construction. |
| `http_client` | None | `reqwest::Client` is internally `Arc`-wrapped. |
| `sa_token_cache` | `Arc<tokio::sync::Mutex<T>>` | Mutable, might be held across `.await`. Uses tokio's async Mutex. |

Notice `sa_token_cache` uses `tokio::sync::Mutex` while `metrics` uses `std::sync::Mutex`. The difference: the token cache might be held while an async HTTP request is in flight. Standard mutexes + await = thread starvation. Tokio's mutex yields the task while waiting.

#### Axum's FromRef

```rust
impl axum::extract::FromRef<AppState> for Key {
    fn from_ref(state: &AppState) -> Self {
        state.cookie_key.clone()
    }
}
```

This tells Axum how to extract a `Key` from `AppState`. The `SignedCookieJar` extractor needs a `Key` — this `FromRef` implementation provides it, so handlers can use `SignedCookieJar` without manually extracting the key.

#### Construction Order Matters

```rust
let repo = Repository::new(&cfg.database_url).await?;
repo.migrate().await?;

let cookie_key = derive_cookie_key(&cfg.cookie_secret);
let vault = Vault::new(&cfg.master_key);

let ollama_adapter: Arc<dyn LlmAdapter> = Arc::new(OllamaAdapter::new(...));
let gemini_adapter: Arc<dyn LlmAdapter> = Arc::new(GeminiAdapter::new(...));
let llm: Arc<dyn LlmAdapter> = Arc::new(FallbackLlmAdapter::new(primary, fallback));

let state = AppState { repo, cookie_key, llm, ... };
```

Everything is created eagerly at startup. If the database is unreachable, the app fails fast — it doesn't wait for the first request to discover the problem. This is the **fail-fast principle** (Zero to Production §5.1): catch configuration errors before the server starts accepting traffic.

---

### Chapter 12: Database Access with sqlx

*(References: Zero to Production §3.6 "Database" and §7 "Reject Invalid Subscribers"; Rust for Rustaceans §4.2 "Working with External Systems")*

sqlx is a compile-time verified, async SQL toolkit. Unlike ORMs (Diesel, SeaORM), you write raw SQL — but the compiler checks your queries against the database schema.

#### The Repository Pattern

*(References: Zero to Production §3.6.2 "Repository Pattern")*

```rust
#[derive(Clone)]
pub struct Repository {
    pool: PgPool,
}

impl Repository {
    pub async fn new(url: &str) -> anyhow::Result<Self> {
        let pool = PgPoolOptions::new()
            .max_connections(20)
            .connect(url)
            .await?;
        Ok(Self { pool })
    }
}
```

All database access goes through `Repository`. This centralizes SQL queries, makes them testable, provides a clean abstraction for handlers, and — critically — ensures every query uses parameterized bindings. No handler ever constructs SQL directly.

The `pool` field is private (`pool: PgPool`, not `pub pool`). External code must use `Repository`'s methods. This is encapsulation — the pool configuration (max connections, timeout) is an implementation detail.

#### Compile-Time Migrations

```rust
pub async fn migrate(&self) -> anyhow::Result<()> {
    sqlx::migrate!("./migrations").run(&self.pool).await?;
    Ok(())
}
```

`sqlx::migrate!("./migrations")` is a **proc macro** that reads all `.sql` files in the `migrations/` directory at compile time and embeds them in the binary. No external migration tool needed. At runtime, it checks which migrations have been applied and runs the new ones.

#### Query Mapping with `query_as`

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

`query_as::<_, User>` maps SQL result rows to the `User` struct. The `FromRow` derive on `User` generates the mapping code. Each `$1`, `$2` placeholder is bound via `.bind()` — **parameterized queries prevent SQL injection by construction, not by careful escaping**.

#### The Upsert Pattern

```rust
"INSERT INTO garmin_daily_data (user_id, date, steps, ...)
 VALUES ($1, $2, $3, ...)
 ON CONFLICT (user_id, date) DO UPDATE SET
     steps=COALESCE(excluded.steps, garmin_daily_data.steps),
     distance_meters=COALESCE(excluded.distance_meters, garmin_daily_data.distance_meters),
     ..."
```

`ON CONFLICT ... DO UPDATE SET` is PostgreSQL's upsert. `COALESCE(excluded.X, garmin_daily_data.X)` means: "use the new value if it's not null, otherwise keep the existing value." This preserves data — if today's sync has steps but not sleep data, yesterday's sleep data survives.

#### QueryBuilder for Dynamic Queries

```rust
pub async fn upsert_training_sets(
    &self,
    user_id: Uuid,
    plan_file: &str,
    day_label: &str,
    exercise_name: &str,
    sets: &[(i32, Option<String>, Option<String>, Option<String>)],
    performed_at: Option<DateTime<Utc>>,
) -> anyhow::Result<()> {
    let mut qb: sqlx::QueryBuilder<sqlx::Postgres> = sqlx::QueryBuilder::new(
        "INSERT INTO training_set_logs (...) "
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
    qb.push(" ON CONFLICT (...) DO UPDATE SET ...");
    qb.build().execute(&self.pool).await?;
    Ok(())
}
```

`QueryBuilder` is for queries where the number of rows is dynamic. `push_values` generates the correct `VALUES ($1,$2,...), ($3,$4,...), ...` with proper bind parameter numbering. Every value goes through bind parameters — **no string interpolation, no SQL injection**.

---

## Part III: Production Patterns

### Chapter 13: Cryptography — Encryption at Rest

*(References: Rust for Rustaceans §10.1 "Security Considerations")*

Gorilla Coach stores Garmin passwords and OAuth tokens encrypted in the database. If the database is compromised, the attacker gets ciphertext, not plaintext.

#### ChaCha20Poly1305: Authenticated Encryption

```rust
pub struct Vault {
    cipher: ChaCha20Poly1305,
}

impl Vault {
    pub fn new(master_key: &str) -> Self {
        let key_bytes = master_key.as_bytes();
        assert!(key_bytes.len() >= 32, "MASTER_KEY must be at least 32 bytes");
        let key: [u8; 32] = key_bytes[..32].try_into().expect("key slice");
        Self {
            cipher: ChaCha20Poly1305::new(Key::from_slice(&key)),
        }
    }
}
```

**ChaCha20Poly1305** is an AEAD (Authenticated Encryption with Associated Data) cipher. It provides:
- **Confidentiality**: The data is encrypted (ChaCha20 stream cipher).
- **Integrity**: An authentication tag (Poly1305 MAC) detects any modification.

**Why ChaCha20 over AES?** ChaCha20 is faster in software (AES needs hardware acceleration). On servers without AES-NI, ChaCha20 is 3x faster. It's the cipher used by TLS 1.3, WireGuard, and Signal.

#### Nonce Management

```rust
pub fn encrypt(&self, data: &str) -> (String, String) {
    let mut nonce_bytes = [0u8; 12];
    rand::rng().fill_bytes(&mut nonce_bytes);
    let nonce = Nonce::from_slice(&nonce_bytes);
    let ciphertext = self.cipher
        .encrypt(nonce, data.as_bytes())
        .expect("Encryption failure");
    (
        BASE64_STANDARD.encode(ciphertext),
        BASE64_STANDARD.encode(nonce_bytes),
    )
}
```

A 12-byte random nonce is generated for **every encryption**. Reusing a nonce with the same key catastrophically breaks ChaCha20Poly1305 — the attacker can XOR two ciphertexts to recover plaintext. Random nonces from a CSPRNG (cryptographically secure pseudo-random number generator) make reuse astronomically unlikely.

Both the ciphertext and nonce are base64-encoded for storage in text columns. The nonce is not secret — it just must be unique.

---

### Chapter 14: Authentication and Session Management

#### Signed Cookies

```rust
fn derive_cookie_key(secret: &str) -> Key {
    use hmac::{Hmac, Mac};
    use sha2::Sha256;
    type HmacSha256 = Hmac<Sha256>;
    let mut mac = HmacSha256::new_from_slice(b"gorilla-coach-cookie-key").expect("...");
    mac.update(secret.as_bytes());
    let result = mac.finalize().into_bytes();
    let mut key_bytes = [0u8; 64];
    key_bytes[..32].copy_from_slice(&result);
    key_bytes[32..].copy_from_slice(&result);
    Key::from(&key_bytes)
}
```

The cookie signing key is derived from a secret using HMAC-SHA256. This ensures a full 256-bit key even if the environment variable is shorter. `Key::from` requires 64 bytes (for HMAC signing + encryption), so the 32-byte hash is repeated.

The `SignedCookieJar` from `axum-extra` uses this key to HMAC-sign every cookie. When a cookie comes back from the browser, the signature is verified. An attacker can't forge a session cookie without knowing the secret.

#### Session Pattern

```rust
pub fn get_session(jar: &SignedCookieJar) -> Option<(Uuid, String)> {
    jar.get("session").and_then(|c| {
        let parts: Vec<&str> = c.value().split('|').collect();
        if parts.len() == 2 {
            Uuid::parse_str(parts[0]).ok().map(|uid| (uid, parts[1].to_string()))
        } else {
            None
        }
    })
}
```

The session cookie contains `user_id|email`, signed by HMAC. There's no server-side session store — the cookie IS the session. This is stateless authentication, perfect for horizontal scaling.

Handlers check the session at the start:

```rust
let (user_id, email) = match get_session(&jar) {
    Some(u) => u,
    None => return redirect_to_login().into_response(),
};
```

---

### Chapter 15: Security Hardening — CSRF, Rate Limiting, CSP

#### Defense in Depth

The security layers work together — no single layer is sufficient:

| Layer | Prevents | Mechanism |
|-------|----------|----------|
| CSRF middleware | Cross-origin state changes | Origin/Referer header verification |
| Rate limiting | Brute-force attacks | Per-IP sliding window (10 req/60s for auth) |
| CSP headers | XSS from loading external scripts | `connect-src 'self'`; `script-src` allowlist |
| Signed cookies | Session forgery | HMAC verification |
| Encryption at rest | Data exposure from DB compromise | ChaCha20Poly1305 |
| Parameterized SQL | SQL injection | Bind parameters via sqlx |
| `sanitize_filename()` | Path traversal | Allowlist: `[a-zA-Z0-9._-]` |
| `html_escape()` | Stored XSS | Entity encoding of user content in HTML |

Each layer defends against a different attack vector. If CSP is bypassed (e.g., via a CDN compromise), CSRF still blocks cross-origin requests. If CSRF is bypassed (e.g., via a same-origin XSS), rate limiting slows down exploitation. This is the **defense in depth** principle from OWASP.

---

### Chapter 16: Server-Sent Events and Streaming

#### The SSE Pattern

Server-Sent Events (SSE) is HTTP's built-in server push mechanism. The server sends a stream of events over a long-lived HTTP connection.

```rust
let stream = async_stream::stream! {
    while let Ok(result) = stream_rx.recv().await {
        match result {
            Ok(token) => {
                let escaped = token.replace('\\', "\\\\")
                    .replace('"', "\\\"")
                    .replace('\n', "\\n")
                    .replace('\r', "\\r");
                yield Ok::<_, std::convert::Infallible>(
                    format!("data: {{\"token\":\"{}\"}}\n\n", escaped)
                );
            }
            Err(e) => {
                yield Ok::<_, std::convert::Infallible>(
                    format!("data: {{\"error\":\"{}\"}}\n\n", escaped)
                );
                break;
            }
        }
    }
    yield Ok::<_, std::convert::Infallible>("data: {\"done\":true}\n\n".to_string());
};
```

The `async_stream::stream!` macro creates an async iterator. Each `yield` sends one SSE event. The SSE format is `data: <json>\n\n` — two newlines terminate each event.

#### Why Not WebSockets?

SSE is simpler than WebSockets for one-way server-to-client streaming:
- No upgrade handshake. Standard HTTP.
- Automatic reconnection built into browsers.
- Works through HTTP proxies and CDNs.
- The Dioxus Wasm client parses SSE text responses in `chat.rs`; the v1 UI used htmx's native SSE support.

WebSockets are bidirectional — overkill when the client only sends one request (the chat message) and the server streams the response.

#### The SSE Router Separation

```rust
let stream_router = Router::new()
    .route("/api/chat/stream", post(handlers::chat_stream_handler))
    .with_state(state.clone());

let app = Router::new()
    // ... main routes with compression ...
    .layer(CompressionLayer::new())
    .merge(stream_router)  // SSE routes bypass compression
```

SSE routes are merged **after** the compression layer. This is critical — gzip compression buffers output, which destroys real-time streaming. The SSE router doesn't get the compression layer, so tokens flow to the browser immediately.

---

### Chapter 17: Background Workers and Graceful Shutdown

*(References: Zero to Production §10 "Background Workers")*

#### The Background Sync Worker

```rust
let bg_state = state.clone();
tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(60 * 60));
    loop {
        interval.tick().await;
        tracing::info!("Background sync: checking users...");
        match bg_state.repo.get_users_with_garmin().await {
            Ok(users) => {
                for s in users {
                    if let Err(e) = handlers::perform_user_sync(&bg_state, s.user_id).await {
                        tracing::warn!("Background sync failed for user {}: {}", s.user_id, e);
                    }
                }
            }
            Err(e) => tracing::error!("Background sync: failed to fetch users: {}", e),
        }
    }
});
```

`tokio::time::interval` creates a recurring timer. The loop runs every hour, syncing Garmin data for all users with stored credentials.

Error handling is deliberately lenient — individual user sync failures are logged and skipped. The loop continues. A failing sync for one user doesn't crash the entire worker. This is the **bulkhead pattern** — isolating failures so they don't cascade.

#### Graceful Shutdown

```rust
async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };
    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => tracing::info!("Ctrl+C received"),
        _ = terminate => tracing::info!("SIGTERM received"),
    }
}
```

`tokio::select!` waits for the first of multiple futures to complete. When either Ctrl+C or SIGTERM arrives, the signal is caught. Axum's `with_graceful_shutdown` then stops accepting new connections and waits for in-flight requests to complete.

The `#[cfg(unix)]` conditional compilation ensures SIGTERM handling is only compiled on Unix systems. On Windows, `std::future::pending()` never completes, so only Ctrl+C works.

---

### Chapter 18: Configuration, CLI, and 12-Factor Design

*(References: Zero to Production §5.2 "Configuration Management"; The Rust Book Ch. 12 "Command Line Programs")*

#### 12-Factor Config: Environment Variables

```rust
impl AppConfig {
    pub fn from_env() -> Self {
        let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
        let port = env::var("PORT").ok().and_then(|v| v.parse().ok()).unwrap_or(3000);
        let gemini_models = env::var("GEMINI_MODELS")
            .map(|s| s.split(',').map(|m| m.trim().to_string())
                .filter(|m| !m.is_empty()).collect())
            .unwrap_or_else(|_| vec!["gemini-2.0-flash".to_string()]);
        // ...
    }
}
```

[12-Factor App](https://12factor.net/) principle III: Store config in the environment. Required values (`DATABASE_URL`, `MASTER_KEY`, `COOKIE_SECRET`) use `.expect()` — the app panics at startup if they're missing. Optional values use `.unwrap_or()` with sensible defaults.

**Why panic at startup?** Failing fast with a clear error message is better than running with missing config and failing unpredictably later. Missing `MASTER_KEY` means the vault can't decrypt anything — better to discover this immediately.

#### CLI with clap

```rust
#[derive(Parser)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Subcommand)]
pub enum Commands {
    Server,
    InitDb,
    TestVault { secret: String },
}
```

`clap`'s derive macros generate a full CLI parser from struct definitions. `Commands` is an enum — Rust's enums naturally model subcommands. `TestVault { secret: String }` takes a positional argument.

```bash
gorilla_coach server          # Start the HTTP server
gorilla_coach init-db         # Run migrations
gorilla_coach test-vault "my secret"  # Test encryption
```

---

### Chapter 19: Testing Without a Database

*(References: Zero to Production §3.9 "Testing"; Rust for Rustaceans Ch. 5 "Testing")*

Gorilla Coach has 37 tests, none requiring a running database.

#### Unit Testing Pure Functions

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_origin_to_host() {
        assert_eq!(origin_to_host("https://example.com"), Some("example.com".into()));
        assert_eq!(origin_to_host("http://localhost:3000"), Some("localhost:3000".into()));
        assert_eq!(origin_to_host("ftp://bad.com"), None);
    }

    #[test]
    fn test_metric_column_mapping() {
        assert_eq!(metric_column("hrv"), Some("hrv_last_night"));
        assert_eq!(metric_column("bogus"), None);
    }
}
```

`#[cfg(test)]` means this module is only compiled during `cargo test`. The `super::*` import brings private functions into scope — tests can access private internals of their parent module. This is unique to Rust: test modules live *inside* the module they test, giving them privileged access without `pub` exposure.

#### Testing Security Invariants

```rust
#[test]
fn test_classifier_prompt_not_empty() {
    assert!(!CLASSIFIER_SYSTEM.is_empty());
    assert!(CLASSIFIER_SYSTEM.contains("METRICS ALLOWED"));
}
```

This test verifies the LLM classifier prompt exists and contains expected content. It's a smoke test — if someone accidentally deletes the prompt, the test catches it at compile time.

#### Integration Tests

```rust
// tests/integration_tests.rs
use gorilla_coach::domain::*;
use gorilla_coach::vault::Vault;

#[test]
fn test_vault_encrypt_decrypt() {
    let vault = Vault::new("a]very_secure_master_key_32bytes!");
    let (enc, nonce) = vault.encrypt("my secret");
    let dec = vault.decrypt(&enc, &nonce).unwrap();
    assert_eq!(dec, "my secret");
}
```

Integration tests live in `tests/` and import the library as an external consumer would. They test the public API without access to private internals.

#### What NOT to Unit Test

- **LLM output quality** — non-deterministic, changes with model versions
- **Full request/response cycles** — requires a running server and database
- **External API integration** — use integration tests with mocks in CI

The principle (Rust for Rustaceans §5.1): **test the harness, not the oracle**. The harness (tool mapping, prompt construction, security boundaries, data transformation) is deterministic and testable. The model is evaluated, not unit-tested.

---

### Chapter 20: Release Optimization and Docker

#### Profile Optimization

```toml
[profile.release]
lto = true
codegen-units = 1
strip = true
```

The release binary benefits from:
- **LTO**: Cross-crate optimization. Functions from dependencies can be inlined into your code.
- **Single codegen unit**: The compiler sees the entire crate at once, enabling more aggressive optimization.
- **Symbol stripping**: Removes debug symbols, reducing binary size by ~50%.

A typical Gorilla Coach release binary is ~15MB — a fully static, self-contained web server with encryption, database drivers, and HTTP client.

#### Multi-Stage Docker Build

```dockerfile
FROM rust:1.83-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /target/release/gorilla_coach /usr/local/bin/
CMD ["gorilla_coach", "server"]
```

The multi-stage build compiles in a full Rust environment, then copies only the binary to a minimal Debian image. The final image is ~80MB instead of ~2GB.

---

## Day-2 Operations: Running Rust in Production

### Build and Deployment

```bash
# Development build (fast compile, slow runtime)
cargo run -- server

# Release build (slow compile, fast runtime, ~15MB binary)
cargo build --release
./target/release/gorilla_coach server

# Docker build (multi-stage, ~80MB image)
docker build -t gorilla-coach .
```

Release profile settings in `Cargo.toml`:
- `lto = true` — link-time optimization across crate boundaries
- `codegen-units = 1` — single codegen unit for maximum optimization
- `strip = true` — removes debug symbols (~50% smaller binary)

### Required Environment Variables

The server will panic on startup if these are missing:

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Postgres connection string |
| `MASTER_KEY` | At least 32 bytes — encryption key for Vault |
| `COOKIE_SECRET` | HMAC-derived key for signed session cookies |

Optional but important:
- `COOKIE_SECURE=true` — set in production (requires HTTPS)
- `RUST_LOG=gorilla_coach=debug` — enable debug logging
- `PORT` (default: 3000)

### Debugging Runtime Issues

**Logging levels**: Controlled by `RUST_LOG` environment variable:

```bash
# All app logging
RUST_LOG=gorilla_coach=debug cargo run -- server

# Specific module
RUST_LOG=gorilla_coach::garmin=debug cargo run -- server

# Include framework traces
RUST_LOG=gorilla_coach=debug,tower_http=debug cargo run -- server
```

**Panic backtraces**: If the server panics, get a full backtrace:

```bash
RUST_BACKTRACE=1 cargo run -- server
```

### CLI Subcommands

The binary supports multiple subcommands beyond `server`:

```bash
cargo run -- server      # Start the web server
cargo run -- init-db     # Run migrations only (no server)
cargo run -- test-vault "my secret"  # Verify encryption round-trip
```

`test-vault` is useful for verifying `MASTER_KEY` is correct without starting
the full server. If the key is wrong, `decrypt()` will fail with
`"Decryption failed"`.

### CSRF Middleware Troubleshooting

The CSRF middleware checks `Origin` or `Referer` headers against `Host` on
mutating requests (POST/PUT/DELETE/PATCH). Common issues:

```
CSRF: Origin mismatch: origin=https://tailscale-host, host=localhost:3000
CSRF: Referer mismatch: referer=https://proxy-host/path, host=app:3000
```

This happens when a reverse proxy (Tailscale serve, nginx) changes the `Host`
header but the browser sends the original `Origin`. Fix: ensure the proxy
forwards the original `Host` header, or the app binds to the domain the
browser sees.

**Relaxed behavior**: If neither `Origin` nor `Referer` is present, the
request is allowed (OWASP relaxed stance). SameSite=Lax cookies provide
backup CSRF protection.

### Rate Limiter Behavior

Two rate limiters are configured:
- **Auth routes**: 10 requests per 60 seconds per IP
- **Upload routes**: 30 requests per 60 seconds per IP

Rate-limited requests return HTTP 429 with plain text body
`"Rate limited. Try again later."` — no structured error, no retry-after
header. The rate limiter does not log — look for 429 responses in the trace
layer output.

The in-memory rate limiter prunes entries when the map exceeds 1000 IPs
(removes entries older than 2× the window).

### Running Tests

```bash
# All 37 tests (no database needed)
cargo test

# Specific test module
cargo test --lib llm::analyst

# With output
cargo test -- --nocapture
```

Tests cover: domain models, vault encryption, CSRF middleware, AI analyst
mappings, CSV parsing. No integration tests require a database.

---

## Epilogue: The Rust Mental Model

After working through this tutorial, you should have internalized the Rust mental model:

1. **Types are documentation.** `Option<i64>` says "this might be null" more clearly than any comment. `Result<T, AppError>` says "this can fail" and forces you to handle it.

2. **The compiler is your pair programmer.** When it rejects your code, it's usually catching a real bug — a data race, a use-after-free, an unhandled error case. Fight the compiler long enough and you stop fighting it — you start designing your code the way it wants.

3. **Zero-cost abstractions are real.** Traits, generics, iterators, async/await — they compile to the same machine code you'd write by hand. The abstractions exist at compile time; the binary knows only concrete types and direct function calls.

4. **Ownership is a design tool.** It doesn't just prevent bugs — it guides you toward better architecture. When the borrow checker rejects a pattern, it's often because the pattern has a subtle concurrency issue that would be a race condition in any other language. The Gorilla Coach codebase's `Arc<Mutex<>>` patterns aren't boilerplate — they're the compiler-verified evidence that shared state is synchronized.

The Gorilla Coach codebase demonstrates these principles in ~3500 lines of production Rust. Every pattern in this tutorial exists in working code, processing real biometric data, protecting real user secrets, and serving real HTTP responses.

---

*This tutorial is part of the Gorilla Coach documentation. For the LLM agent architecture, see [LLM_AGENTS_TUTORIAL.md](LLM_AGENTS_TUTORIAL.md). For infrastructure plans, see [INFRASTRUCTURE_TODO.md](INFRASTRUCTURE_TODO.md).*
