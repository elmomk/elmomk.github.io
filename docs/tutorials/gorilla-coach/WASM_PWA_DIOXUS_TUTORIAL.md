# WebAssembly, Progressive Web Apps & Dioxus — A Deep Dive

> *"The Web is the most hostile software engineering environment
> imaginable."*
> — Douglas Crockford, *JavaScript: The Good Parts* (2008)

This tutorial explores how Gorilla Coach delivers a native-quality
experience in the browser by compiling Rust to **WebAssembly** (Wasm),
packaging it as a **Progressive Web App** (PWA), and structuring the
client with **Dioxus 0.7**, a React-like framework written entirely in
Rust. We walk through the technology stack from the metal up — how bytes
move from `.rs` files to the browser's virtual machine, how the service
worker turns a website into an installable app, and how Dioxus's reactive
signals let you write rich UIs in the same language as your server.

Every code example is from the actual Gorilla Coach codebase. Nothing is
hypothetical.

---

## Reference Texts

| Abbreviation | Book |
|---|---|
| **PP** | Andrew Hunt & David Thomas — *The Pragmatic Programmer*, 20th Anniversary Ed. (2019) |
| **ZtP** | Luca Palmieri — *Zero To Production in Rust* (2022) |
| **CA** | Robert C. Martin — *Clean Architecture* (2017) |
| **DDIA** | Martin Kleppmann — *Designing Data-Intensive Applications* (2017) |
| **RiA** | Tim McNamara — *Rust in Action* (2021) |
| **OWASP** | OWASP Foundation — *OWASP Top 10* (2021) |
| **WasmSpec** | WebAssembly Community Group — *WebAssembly Specification* (2024) |
| **MDN-PWA** | Mozilla Developer Network — *Progressive Web Apps* (2024) |
| **HCD** | Steve Krug — *Don't Make Me Think, Revisited* (2014) |

---

## Table of Contents

1. [Part I — WebAssembly Fundamentals](#part-i--webassembly-fundamentals)
2. [Part II — Rust to Wasm: The Compilation Pipeline](#part-ii--rust-to-wasm-the-compilation-pipeline)
3. [Part III — Dioxus: A Rust-Native UI Framework](#part-iii--dioxus-a-rust-native-ui-framework)
4. [Part IV — Progressive Web Apps](#part-iv--progressive-web-apps)
5. [Part V — JavaScript Interop & Browser APIs](#part-v--javascript-interop--browser-apis)
6. [Part VI — Offline-First with IndexedDB](#part-vi--offline-first-with-indexeddb)
7. [Part VII — Server Integration: Serving the Wasm Bundle](#part-vii--server-integration-serving-the-wasm-bundle)
8. [Part VIII — Performance & Size Optimization](#part-viii--performance--size-optimization)
9. [Part IX — Patterns & Pitfalls](#part-ix--patterns--pitfalls)

---

# Part I — WebAssembly Fundamentals

## What WebAssembly Actually Is

WebAssembly is a binary instruction format — a portable, size-efficient
compilation target designed to run at near-native speed in web browsers. It
is **not** a programming language; it is a *virtual machine specification*
(**WasmSpec**). Think of it as a CPU architecture that happens to exist
inside every browser.

Key properties:

| Property | Meaning |
|---|---|
| **Stack machine** | Instructions push/pop values from an implicit stack (no registers) |
| **Typed** | Four value types: `i32`, `i64`, `f32`, `f64` (plus `v128` for SIMD) |
| **Sandboxed** | Linear memory model — Wasm code cannot access host memory directly |
| **Deterministic** | Same inputs yield same outputs (no undefined behavior) |
| **Streamable** | Browsers can compile Wasm modules as they download (`WebAssembly.compileStreaming`) |

A `.wasm` binary is to WebAssembly what an `.elf` is to Linux — an
executable that a specific runtime knows how to load and execute. The
browser's Wasm runtime JIT-compiles these bytes to actual machine code
for the host CPU.

## Why Not Just JavaScript?

JavaScript is dynamically typed. Every property access is a hash table
lookup (or an optimistic JIT guess). Numbers are IEEE 754 doubles unless
the engine proves otherwise. Garbage collection pauses are unpredictable.

Wasm starts with none of these problems:

- **No GC**: Rust's ownership model means memory is freed deterministically.
  There are no stop-the-world pauses.
- **Predictable performance**: The Wasm type system maps directly to
  machine instructions. An `i32.add` compiles to a single CPU `add`.
- **Type safety at the wire boundary**: When Gorilla Coach's client
  deserializes a `DashboardResponse`, serde validates every field at
  compile time. A typo in a field name is a compilation error, not a
  runtime `undefined`.

That said, Wasm has a cost: **no direct DOM access**. Every interaction
with the browser's rendering engine must cross the Wasm ↔ JavaScript
boundary via `wasm-bindgen`. Frameworks like Dioxus abstract this away,
but the overhead is real — minimizing DOM updates is critical.

> **PP §9, "DRY"**: The shared types in `gorilla_shared` mean the server's
> `DashboardResponse` and the client's deserialization target are the *same
> struct*. Zero duplication across the wire.

## The Memory Model

Wasm operates on a contiguous block of bytes called **linear memory**.
It starts at a fixed size (typically 1–16 pages of 64 KiB each) and
grows on demand via `memory.grow`. Rust's allocator (`dlmalloc` or
`wee_alloc`) manages this linear memory the same way `malloc/free`
manage the heap on a native target.

```
┌─────────────────────────────────────────────────────┐
│                  Wasm Linear Memory                  │
│  ┌──────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │ Stack │ │ Static Data  │ │        Heap          │ │
│  │ ↓     │ │ (strings,    │ │ (Box, Vec, String,   │ │
│  │       │ │  vtables)    │ │  HashMap, ...)       │ │
│  └──────┘ └──────────────┘ └──────────────────────┘ │
└─────────────────────────────────────────────────────┘
         ↑ Accessible to Wasm
─────────┼─────────────────────────────────────
         ↓ NOT accessible — browser/OS memory
```

JavaScript can view this memory as a `Uint8Array` via
`WebAssembly.Memory.buffer`, but Wasm cannot reach outside. This
sandboxing is why Wasm is considered safe to run — a malicious module
cannot read cookies, touch the DOM, or exfiltrate data without
explicitly imported JavaScript functions (**OWASP** A01:2021, Broken
Access Control).

---

# Part II — Rust to Wasm: The Compilation Pipeline

## The Target Triple

Rust compiles to Wasm via the target `wasm32-unknown-unknown`:

| Component | Value | Meaning |
|---|---|---|
| Architecture | `wasm32` | 32-bit WebAssembly |
| Vendor | `unknown` | No specific vendor |
| OS | `unknown` | No operating system (bare metal / browser) |

This means anything that touches the OS — file I/O, sockets, threads,
`std::time::Instant` — is unavailable or stubbed. The `chrono` crate
works because it has a `wasmbind` feature that redirects `Utc::now()` to
JavaScript's `Date.now()`. The `uuid` crate works because it has a `js`
feature that uses `crypto.getRandomValues()`.

From `gorilla_client/Cargo.toml`:

```toml
chrono = { version = "0.4", features = ["serde", "wasmbind"] }
uuid   = { version = "1", features = ["v4", "serde", "js"] }
```

## The Build Toolchain

Gorilla Coach uses **dioxus-cli** (`dx`) to orchestrate the build:

```bash
cd gorilla_client && dx build --release
```

Under the hood, this runs:

1. `cargo build --target wasm32-unknown-unknown --release` → produces a
   `.wasm` binary
2. `wasm-bindgen` → generates JS glue code (the "shim") that imports and
   exports functions between Wasm and JavaScript
3. `wasm-opt` (optional) → runs binaryen optimizations on the `.wasm` file
4. Bundles everything into a directory with `index.html`, assets, and the
   `.wasm` binary

The output lands in:

```
target/dx/gorilla_client/release/web/public/
├── index.html          ← Dioxus-generated, patched by server
├── assets/
│   ├── dioxus/         ← Wasm binary + JS glue
│   │   ├── gorilla_coach_bg.wasm
│   │   └── gorilla_coach.js
│   ├── main.css
│   ├── charts.js
│   ├── manifest.json
│   ├── sw.js
│   ├── icon-192.png
│   └── icon-512.png
└── snippets/           ← wasm-bindgen inline JS snippets
```

## wasm-bindgen: The Bridge

`wasm-bindgen` is the foundational crate that makes Rust ↔ JavaScript
communication possible. It generates:

1. **JavaScript glue** that loads the `.wasm` binary, instantiates it,
   and wraps exported Rust functions as JavaScript callables.
2. **Rust bindings** for imported JavaScript functions (like `console.log`,
   `document.createElement`, `fetch`).

The `web-sys` crate is a thin `wasm-bindgen` wrapper around every Web API.
Gorilla Coach enables specific features:

```toml
web-sys = { version = "0.3", features = [
    "Window", "Navigator", "Storage", "Location",
    "IdbFactory", "IdbDatabase", "IdbObjectStore", "IdbRequest",
    "IdbTransaction", "IdbTransactionMode", "IdbOpenDbRequest",
    "IdbObjectStoreParameters", "IdbKeyRange",
    "EventSource", "EventSourceInit", "MessageEvent",
    "ServiceWorkerContainer", "ServiceWorkerRegistration",
    "Headers", "Request", "RequestInit", "RequestMode",
    "Response", "Url",
] }
```

Each feature flag corresponds to a Web API interface. Only what you use
gets compiled — this is tree-shaking at the type level.

## The Wasm Release Profile

Size matters in the browser. Every kilobyte is a download penalty. The
workspace `Cargo.toml` defines a dedicated profile:

```toml
[profile.wasm-release]
inherits = "release"
opt-level = "z"      # Optimize for size (not speed)
lto = true           # Link-time optimization across all crates
codegen-units = 1    # Single codegen unit for maximum optimization
panic = "abort"      # No unwinding — saves ~10% binary size
strip = true         # Strip debug symbols
```

| Setting | Effect |
|---|---|
| `opt-level = "z"` | Favors smaller code over faster code. Inlines less, uses smaller instruction sequences |
| `lto = true` | Cross-crate dead-code elimination. Removes unused functions from dependencies |
| `codegen-units = 1` | Forces single-threaded compilation for better optimization (slower build) |
| `panic = "abort"` | Removes the entire unwinding runtime (~50 KiB savings in the `.wasm`) |
| `strip = true` | Removes symbol names and debug info |

> **RiA §12**: Understanding compilation profiles is essential for
> production Rust. The difference between `opt-level = 3` and
> `opt-level = "z"` can be 40% of your binary size.

---

# Part III — Dioxus: A Rust-Native UI Framework

## Why Dioxus?

Dioxus is a React-like framework for Rust. It shares React's core ideas
— declarative UI, components, virtual DOM diffing — but implements them
in Rust with compile-time type safety.

Alternatives considered and rejected:

| Framework | Reason |
|---|---|
| **Yew** | Older, uses `html!` macro with HTML syntax. Heavier runtime. |
| **Leptos** | Excellent, but signals-based from day one with fine-grained reactivity. Dioxus 0.7 adopted signals too, making them feature-comparable. |
| **Sycamore** | Smaller community, fewer escape hatches for JS interop. |
| **Seed** | Elm-architecture, requires message passing for every state change. |
| **Raw wasm-bindgen** | Maximum control, minimum productivity. No VDOM, no reactivity. |

Dioxus 0.7 specifically was chosen because:

1. **RSX macro** — Write UI in Rust syntax, not string templates
2. **Signals** — Fine-grained reactivity (no need to clone entire state)
3. **Client-side routing** — Derive macro on an enum, done
4. **Multi-platform** — Same code can target web, desktop, mobile, SSR
5. **Active development** — Backed by the Dioxus Labs team with regular releases

## The RSX Macro

RSX (Rust Syntax Extension) is how you write UI in Dioxus. It looks like
a stripped-down JSX:

```rust
rsx! {
    div { class: "stat-card color-{color}",
        div { class: "stat-label", "{label}" }
        div { class: "stat-value",
            span { "{value}" }
            if !unit.is_empty() {
                span { class: "stat-unit", " {unit}" }
            }
        }
    }
}
```

Key differences from HTML:

| HTML | RSX |
|---|---|
| `<div class="foo">` | `div { class: "foo",` |
| `<input type="text" />` | `input { r#type: "text" }` |
| `{variable}` (JS) | `"{variable}"` (Rust format string) |
| `onClick={handler}` | `onclick: move \|_\| handler()` |
| `{items.map(i => <li>{i}</li>)}` | `for item in items { li { "{item}" } }` |

The `r#type` syntax is required because `type` is a Rust keyword. The
`r#` prefix is Rust's raw identifier escape.

The RSX macro expands at compile time into Rust code that constructs a
virtual DOM tree. Invalid attribute names, wrong types, or missing
closing braces are all compilation errors — not runtime surprises.

## Signals: Fine-Grained Reactivity

Signals are the reactive primitive in Dioxus 0.7. A signal is a value
that, when read during render, subscribes the component to future updates.
When written, it triggers re-renders of exactly the components that read it.

From `gorilla_client/src/pages/dashboard.rs`:

```rust
#[component]
pub fn Dashboard() -> Element {
    let mut view = use_signal(|| "day".to_string());
    let mut date = use_signal(|| chrono::Utc::now().date_naive());
    let mut sync_msg = use_signal(|| Option::<String>::None);
    let mut syncing = use_signal(|| false);
    // ...
}
```

### Signal Operations

| Operation | Syntax | Effect |
|---|---|---|
| **Read** (in RSX) | `"{view}"` or `*view.read()` | Subscribes component to changes |
| **Write** | `view.set("week".to_string())` | Triggers re-render of subscribers |
| **Mutate** | `messages.write().push(...)` | Write-lock, mutate in place, trigger re-render |
| **Read without subscribing** | `view.peek()` | Reads current value without tracking |

### How Signals Differ from React's `useState`

In React, `setState` re-renders the entire component and all its
children. In Dioxus, signals track which components read them. If
component A reads signal X but component B does not, writing to X only
re-renders A.

```rust
// When the user clicks "Week", only the parts of the UI that
// read `view` re-render — not the entire page.
button {
    class: if *view.read() == "week" { "sel" } else { "" },
    onclick: move |_| view.set("week".to_string()),
    "Week"
}
```

This is the same fine-grained reactivity model that SolidJS and Leptos
use. It avoids the "re-render the world" problem that plagues naive React
applications.

## use_resource: Async Data Fetching

`use_resource` is Dioxus's built-in hook for async data that should
re-fetch when dependencies change. It returns a `Resource<T>` that is
`None` while loading, `Some(Ok(T))` on success, and `Some(Err(E))` on
failure.

From the Dashboard:

```rust
let dashboard = use_resource(move || {
    let v = view.read().clone();
    let d = *date.read();
    async move {
        api::fetch_dashboard(&v, d).await
    }
});
```

**How it works:**

1. Dioxus calls the closure immediately on first render.
2. The closure reads `view` and `date` — this subscribes the resource
   to those signals.
3. The `async move` block executes, calling the API.
4. When the user changes `view` or `date`, Dioxus *automatically* re-runs
   the closure and the async block. No manual dependency arrays (unlike
   React's `useEffect`).

The rendering pattern is always the same:

```rust
match &*dashboard.read() {
    Some(Ok(data)) => rsx! { /* render data */ },
    Some(Err(e)) => rsx! { div { class: "error", "Error: {e}" } },
    None => rsx! { div { class: "loading", "Loading..." } },
}
```

This three-way match forces you to handle loading and error states —
it's impossible to forget. Compare this to React, where `useEffect` +
`useState` requires manual `isLoading` / `error` state management.

## use_hook: One-Time Side Effects

`use_hook` runs a closure once on component mount. It does *not*
re-run when signals change — it's the equivalent of React's
`useEffect(() => { ... }, [])` with an empty dependency array.

From the Training page:

```rust
use_hook(move || {
    spawn(async move {
        match api::fetch_training_files().await {
            Ok(fl) => {
                let current = selected_file.read().clone();
                let valid_current = current.as_ref()
                    .map_or(false, |c| fl.files.contains(c));

                if !valid_current {
                    let pick = fl.selected_file.as_ref()
                        .filter(|s| fl.files.contains(s))
                        .or_else(|| if fl.files.len() == 1 {
                            fl.files.first()
                        } else { None })
                        .cloned();

                    if let Some(ref f) = pick {
                        ls_set(LS_TRAINING_FILE_KEY, f);
                        selected_file.set(pick);
                    }
                }
                files.set(Some(Ok(fl)));
            }
            Err(e) => files.set(Some(Err(e))),
        }
    });
});
```

This fetches the list of training plan files exactly once, reconciles
the server's preference with localStorage, and never re-fetches. The
`spawn` call launches an async task that outlives the closure — it's the
Dioxus equivalent of JavaScript's fire-and-forget `Promise`.

**When to use which:**

| Hook | Re-runs on signal change? | Use for |
|---|---|---|
| `use_signal` | N/A (creates a signal) | Reactive local state |
| `use_resource` | Yes | Async data that depends on signals |
| `use_hook` | No (runs once) | One-time initialization, side effects |
| `use_effect` | Yes | Side effects that react to signal changes |
| `spawn` | N/A (fire-and-forget) | Async tasks triggered by user action |

## use_effect: Reactive Side Effects

`use_effect` runs whenever signals it reads change — but unlike
`use_resource`, it doesn't return a value. It's for *side effects*:
DOM manipulation, logging, JavaScript interop.

From the Dashboard's `RangeView`:

```rust
use_effect(move || {
    let json = chart_json.clone();
    let script = format!(
        "if(window.GC && window.GC.renderCharts) {{ window.GC.renderCharts({}); }}",
        json
    );
    let _ = js_sys::eval(&script);
});
```

This calls Chart.js (a JavaScript library) every time the chart data
changes. `use_effect` is deliberately thin — it doesn't manage async
state or return values. If you need that, use `use_resource`.

## Client-Side Routing

Dioxus uses a derive macro on an enum to define routes:

```rust
#[derive(Clone, Routable, PartialEq)]
#[rustfmt::skip]
enum Route {
    #[layout(AppShell)]
        #[route("/")]
        Chat {},
        #[route("/dashboard")]
        Dashboard {},
        #[route("/training")]
        Training {},
        #[route("/auto-reg")]
        AutoReg {},
        #[route("/files")]
        Files {},
        #[route("/settings")]
        Settings {},
    #[end_layout]
    #[route("/login")]
    Login {},
}
```

### Layouts

The `#[layout(AppShell)]` attribute wraps routes in a shell component.
Routes between `#[layout]` and `#[end_layout]` share the same layout.
The `Login` route is *outside* the layout — it has no sidebar navigation.

```rust
#[component]
fn AppShell() -> Element {
    rsx! {
        div { class: "app-shell",
            components::NavBar {}
            main { class: "content",
                Outlet::<Route> {}
            }
        }
    }
}
```

`Outlet::<Route> {}` is where the matched child component renders.
This is the Dioxus equivalent of React Router's `<Outlet />`.

### Navigation

Components use `Link` to navigate, and the `Route` enum for type-safe
destinations:

```rust
#[component]
fn NavLink(to: Route, icon: &'static str, label: &'static str) -> Element {
    rsx! {
        Link { to: to, class: "nav-link", title: "{label}",
            span { class: "material-icons", "{icon}" }
        }
    }
}
```

There is no string URL to mis-type. If you rename a route variant, the
compiler tells you every place that needs updating.

## Components

Components in Dioxus are functions annotated with `#[component]`:

```rust
#[component]
fn StatCard(
    label: &'static str,
    value: String,
    #[props(default)] unit: &'static str,
    #[props(default = "green")] color: &'static str,
) -> Element {
    rsx! {
        div { class: "stat-card color-{color}",
            div { class: "stat-label", "{label}" }
            div { class: "stat-value",
                span { "{value}" }
                if !unit.is_empty() {
                    span { class: "stat-unit", " {unit}" }
                }
            }
        }
    }
}
```

Props are function parameters. The `#[props(default)]` attribute makes
a prop optional (uses `Default::default()`), and `#[props(default = "green")]`
provides a specific default value. This is the Dioxus equivalent of React's
`defaultProps`.

Components compose naturally:

```rust
div { class: "stat-cards-grid",
    StatCard { label: "STEPS", value: fmt_opt(d.steps), color: "green" }
    StatCard { label: "RHR", value: fmt_opt(d.resting_heart_rate), unit: "bpm", color: "red" }
    StatCard { label: "HRV", value: fmt_opt_f(hrv, 0), unit: "ms", color: "cyan" }
    StatCard { label: "SLEEP", value: fmt_opt(d.sleep_score), unit: "/100", color: "purple" }
}
```

> **CA §14, "Component Principles"**: Components should be small, focused,
> and independently deployable (or in this case, independently renderable).
> `StatCard` knows nothing about dashboards — it just renders a label and
> a value.

---

# Part IV — Progressive Web Apps

## What Makes a PWA

A Progressive Web App is a website that can be "installed" like a native
app on desktop or mobile. The technology is defined by three pillars:

1. **HTTPS** — Required for service workers (Tailscale provides this)
2. **Web App Manifest** — JSON file describing the app's identity
3. **Service Worker** — JavaScript running *between* the browser and the
   network, enabling offline support and caching

When all three are present, the browser shows an "Install" prompt. On
mobile, the app gets a home screen icon. On desktop, it runs in its own
window without browser chrome.

## The Manifest

From `gorilla_client/assets/manifest.json`:

```json
{
    "name": "GORILLA COACH",
    "short_name": "Gorilla",
    "description": "AI Fitness Coach — Tactical Training System",
    "start_url": "/",
    "scope": "/",
    "display": "standalone",
    "orientation": "portrait",
    "background_color": "#050505",
    "theme_color": "#00ff41",
    "icons": [
        {
            "src": "/assets/icon-192.png",
            "sizes": "192x192",
            "type": "image/png"
        },
        {
            "src": "/assets/icon-512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ]
}
```

| Field | Purpose |
|---|---|
| `name` | Full name shown in install dialog |
| `short_name` | Name under the home screen icon |
| `start_url` | URL loaded when the app launches |
| `scope` | URL prefix the PWA "owns" — navigation outside this opens in a browser tab |
| `display: standalone` | No browser chrome (address bar, tabs) |
| `background_color` | Splash screen color while loading |
| `theme_color` | OS status bar / title bar color |
| `icons` | At least 192×192 and 512×512 for installability (**MDN-PWA**) |

The manifest is linked in the HTML `<head>`:

```html
<link rel="manifest" href="/manifest.json">
```

Gorilla Coach's server injects this at startup by patching the
Dioxus-generated `index.html` (more on this in Part VII).

## The Service Worker

A service worker is a JavaScript file that runs in a separate thread,
intercepting all network requests from the app. It is the single most
important piece of PWA technology.

From `gorilla_client/assets/sw.js`:

### Lifecycle

```
Install → Activate → Fetch (repeat)
   ↓          ↓
 Cache     Clean old
 assets    caches
```

**Install**: Caches static assets immediately.

```javascript
const CACHE_NAME = 'gorilla-coach-v64';
const STATIC_ASSETS = [
    '/',
    '/assets/main.css',
    '/assets/manifest.json',
];

self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then((cache) => cache.addAll(STATIC_ASSETS))
    );
    self.skipWaiting();
});
```

`skipWaiting()` forces the new service worker to activate immediately
instead of waiting for all tabs to close. Critical for single-user
self-hosted apps where cache staleness is more annoying than harmful.

**Activate**: Purges old caches when the version number changes.

```javascript
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then((names) =>
            Promise.all(
                names
                    .filter((name) => name !== CACHE_NAME)
                    .map((name) => caches.delete(name))
            )
        )
    );
    self.clients.claim();
});
```

`clients.claim()` takes control of all open tabs immediately, without
requiring a page reload.

**Fetch**: The strategy differs by request type.

```javascript
self.addEventListener('fetch', (event) => {
    const url = new URL(event.request.url);

    // API: network-first, cache fallback
    if (url.pathname.startsWith('/api/')) {
        event.respondWith(
            fetch(event.request)
                .then((response) => {
                    if (response.ok) {
                        const clone = response.clone();
                        caches.open(CACHE_NAME)
                            .then((cache) => cache.put(event.request, clone));
                    }
                    return response;
                })
                .catch(() => caches.match(event.request))
        );
        return;
    }

    // Static: cache-first (stale-while-revalidate)
    if (url.pathname.startsWith('/')) {
        event.respondWith(
            caches.match(event.request).then((cached) => {
                const fetchPromise = fetch(event.request).then((response) => {
                    if (response.ok) {
                        const clone = response.clone();
                        caches.open(CACHE_NAME)
                            .then((cache) => cache.put(event.request, clone));
                    }
                    return response;
                });
                return cached || fetchPromise;
            })
        );
        return;
    }
});
```

### Caching Strategies Explained

| Strategy | Used for | Behavior |
|---|---|---|
| **Network-first** | `/api/*` | Try network; on failure, serve from cache |
| **Cache-first** (stale-while-revalidate) | `/*` | Serve cached immediately; update cache in background |

The network-first strategy for API calls ensures the dashboard always
shows the latest Garmin data when online, but still works (with stale
data) when offline. The cache-first strategy for static assets means the
app shell loads instantly — the Wasm binary, CSS, and JavaScript are
served from disk, not the network.

### Cache Versioning

The cache name `gorilla-coach-v{N}` is a version string. The
`scripts/build.sh` script auto-increments this on every client build.
The `activate` handler deletes all caches that don't match the new name,
forcing fresh downloads. Additionally, the server appends a `?v={mtime}`
query parameter to CSS/JS asset URLs using the file's modification
timestamp, ensuring browsers fetch new assets even if the SW hasn't
updated yet.

> **PP §45, "No Broken Windows"**: A stale cache is a broken window.
> The version-bump strategy ensures there's exactly one cache at a time —
> no ambiguity about what's being served.

## Service Worker Registration

The server injects the registration script into `index.html`:

```html
<script>
if('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js', {scope: '/'});
}
</script>
```

The `scope` parameter limits the service worker to URLs under `/`.
It will not intercept requests to `/api/` directly — those go through the
app's `fetch` calls, which the service worker *does* intercept because
the fetch originates from a page within the scope.

---

# Part V — JavaScript Interop & Browser APIs

## Calling JavaScript from Rust

There are several levels of JS interop, from low-level to high-level:

### 1. js_sys::eval — Quick and Dirty

```rust
let _ = js_sys::eval(&format!(
    "if(window.GC && window.GC.renderCharts) {{ window.GC.renderCharts({}); }}",
    json_data
));
```

Used for calling global JavaScript functions that are already loaded
(e.g., Chart.js). Fast to write, but no type safety — if the function
name is wrong, you get a silent failure.

### 2. web_sys — Type-Safe Browser APIs

`web_sys` provides Rust bindings for every Web API:

```rust
fn ls_get(key: &str) -> Option<String> {
    web_sys::window()
        .and_then(|w| w.local_storage().ok().flatten())
        .and_then(|s| s.get_item(key).ok().flatten())
        .filter(|v| !v.is_empty())
}

fn ls_set(key: &str, value: &str) {
    if let Some(s) = web_sys::window()
        .and_then(|w| w.local_storage().ok().flatten())
    {
        let _ = s.set_item(key, value);
    }
}
```

Every call returns `Result` or `Option` — the Rust type system forces
you to handle the case where localStorage is unavailable (e.g., in
private browsing mode).

### 3. Direct DOM Manipulation

Sometimes the VDOM isn't enough. The Training page has a `<select>`
element that needs its value poked after render (because Dioxus renders
options *after* setting the value attribute):

```rust
use_effect(move || {
    let val = selected_file.read().clone().unwrap_or_default();
    if !val.is_empty() {
        spawn(async move {
            gloo_timers::future::TimeoutFuture::new(50).await;
            if let Some(w) = web_sys::window() {
                if let Some(d) = w.document() {
                    if let Ok(Some(el)) = d.query_selector("select.training-select") {
                        use web_sys::wasm_bindgen::JsCast;
                        if let Ok(sel) = el.dyn_into::<web_sys::HtmlSelectElement>() {
                            sel.set_value(&val);
                        }
                    }
                }
            }
        });
    }
});
```

This uses:
- `gloo_timers` for async setTimeout (lets the DOM update first)
- `web_sys::Document::query_selector` for CSS selector lookup
- `JsCast::dyn_into` for downcasting a generic `Element` to `HtmlSelectElement`

This is an escape hatch — most UIs should use signals and RSX. Direct DOM
manipulation is for browser quirks that the VDOM can't handle.

## Chart.js Integration

Gorilla Coach renders trend charts (HR, HRV, sleep, stress, activity, body
composition) using **Chart.js**, a JavaScript charting library. The Dioxus
client communicates with it via a global namespace:

From `gorilla_client/assets/charts.js`:

```javascript
(function() {
    // ...
    function renderCharts(rows) {
        destroyCharts();
        if (!rows || rows.length === 0) return;

        requestAnimationFrame(function() {
            var labels = rows.map(function(r) {
                return r.date ? r.date.slice(5) : '';
            });

            var hrEl = document.getElementById('chart-hr');
            if (hrEl) {
                charts.hr = new Chart(hrEl, {
                    type: 'line',
                    data: { labels, datasets: [
                        { label: 'Resting HR', data: rows.map(r => r.resting_heart_rate),
                          borderColor: '#ff4444', borderWidth: 1.5, tension: 0.3 },
                        // ... more datasets
                    ]},
                    options: mkOpts()
                });
            }
            // ... more chart types
        });
    }

    window.GC = { renderCharts };
})();
```

The Dioxus client serializes `Vec<GarminDailyData>` to JSON and passes it:

```rust
let chart_json = serde_json::to_string(&data).unwrap_or_default();

use_effect(move || {
    let json = chart_json.clone();
    let script = format!(
        "if(window.GC && window.GC.renderCharts) {{ window.GC.renderCharts({}); }}",
        json
    );
    let _ = js_sys::eval(&script);
});
```

The RSX provides `<canvas>` elements for Chart.js to render into:

```rust
div { class: "charts-grid",
    div { class: "chart-container", canvas { id: "chart-hr" } }
    div { class: "chart-container", canvas { id: "chart-hrv" } }
    div { class: "chart-container", canvas { id: "chart-sleep" } }
    // ...
}
```

This pattern — Rust owns the data and lifecycle, JavaScript owns the
rendering of complex visualizations — is common in Wasm applications.
Chart.js has 15 years of browser rendering optimizations; reimplementing
that in Wasm would be wasteful.

The Chart.js script is declared in `Dioxus.toml`:

```toml
[web.resource]
script = [
    "https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js",
    "assets/charts.js",
]
```

Dioxus injects these `<script>` tags into the generated `index.html`.

## Markdown Rendering in Wasm

The Chat page renders markdown responses from the LLM. Instead of using
a JavaScript library like `marked.js`, Gorilla Coach compiles
`pulldown-cmark` (a Rust markdown parser) directly to Wasm:

```rust
fn md_to_html(md: &str) -> String {
    use pulldown_cmark::{Parser, Options, html};
    let parser = Parser::new_ext(md, Options::all());
    let mut html_output = String::new();
    html::push_html(&mut html_output, parser);
    html_output
}
```

This is injected into the DOM via `dangerous_inner_html`:

```rust
div { class: "chat-content", dangerous_inner_html: "{content}" }
```

Why compile a markdown parser to Wasm instead of calling a JS library?

1. **Type safety**: The function signature guarantees `&str → String`.
2. **No JS dependency**: One less CDN call, one less supply-chain risk.
3. **Performance**: `pulldown-cmark` is a zero-copy streaming parser.
   It's faster than most JavaScript markdown libraries.
4. **Consistency**: Server-side reports use the exact same parser.

---

# Part VI — Offline-First with IndexedDB

## Why Not Just localStorage?

localStorage is synchronous, has a 5–10 MiB limit, and stores only
strings. IndexedDB is asynchronous, has effectively unlimited storage
(with user permission), and stores structured objects.

Gorilla Coach uses both:

| Storage | Used for | Reason |
|---|---|---|
| **localStorage** | Selected training file preference | Simple key-value, always needed |
| **IndexedDB** | Cached biometrics, pending workout sets, sync queue | Structured data, larger payloads |

## The IndexedDB Module

From `gorilla_client/src/storage.rs`:

```rust
const DB_NAME: &str = "gorilla_coach";
const DB_VERSION: u32 = 1;

pub const STORE_PENDING_SETS: &str = "pending_sets";
pub const STORE_CACHED_PLAN: &str = "cached_training_plan";
pub const STORE_SYNC_QUEUE: &str = "sync_queue";
pub const STORE_CACHED_BIOMETRICS: &str = "cached_biometrics";
```

Four object stores, each dedicated to a specific offline concern:

1. **pending_sets** — Workout sets logged while offline, queued for
   server sync
2. **cached_training_plan** — Last-fetched training plan, served while
   offline
3. **sync_queue** — Ordered list of operations to replay when back online
4. **cached_biometrics** — Dashboard data snapshot for offline viewing

### Opening the Database

```rust
pub async fn open_db() -> Result<IdbDatabase, JsValue> {
    let window = web_sys::window().expect("no window");
    let factory: IdbFactory = window.indexed_db()?.expect("IndexedDB not available");
    let open_req: IdbOpenDbRequest = factory.open_with_u32(DB_NAME, DB_VERSION)?;

    let on_upgrade = Closure::once(move |event: web_sys::Event| {
        let target = event.target().unwrap();
        let req: IdbOpenDbRequest = target.unchecked_into();
        let db: IdbDatabase = req.result().unwrap().unchecked_into();

        let stores = [STORE_PENDING_SETS, STORE_CACHED_PLAN,
                      STORE_SYNC_QUEUE, STORE_CACHED_BIOMETRICS];
        for name in stores {
            // ... create store if not exists
        }
    });
    // ...
}
```

IndexedDB is event-based, not Promise-based. The `Closure::once` wrapper
converts a Rust closure into a JavaScript callback. `JsFuture::from`
wraps the event-based API in a Rust `Future` that can be `.await`ed.

### Generic CRUD Operations

```rust
pub async fn put<T: serde::Serialize>(
    db: &IdbDatabase, store_name: &str, key: &str, value: &T
) -> Result<(), JsValue> {
    let tx = db.transaction_with_str_and_mode(store_name, IdbTransactionMode::Readwrite)?;
    let store = tx.object_store(store_name)?;
    let js_val = serde_wasm_bindgen::to_value(value)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;
    store.put_with_key(&js_val, &JsValue::from_str(key))?;
    Ok(())
}

pub async fn get<T: serde::de::DeserializeOwned>(
    db: &IdbDatabase, store_name: &str, key: &str
) -> Result<Option<T>, JsValue> {
    // ...
    let result = JsFuture::from(promise).await?;
    if result.is_undefined() || result.is_null() { return Ok(None); }
    let val: T = serde_wasm_bindgen::from_value(result)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;
    Ok(Some(val))
}
```

The `serde_wasm_bindgen` crate bridges Rust's serde ecosystem with
JavaScript values. `to_value` converts a Rust struct into a `JsValue`
(JavaScript object), and `from_value` does the reverse. This means you
can store your domain types directly in IndexedDB without manual
serialization.

> **DDIA §5, "Replication"**: The IndexedDB stores act as a local
> replica. The sync queue is a write-ahead log — operations are recorded
> locally, then replayed to the server when connectivity is restored.

---

# Part VII — Server Integration: Serving the Wasm Bundle

## The `nest_service` Pattern

The Axum server detects the built client directory at startup and
serves it as a nested service:

```rust
let release_dist = PathBuf::from("target/dx/gorilla_client/release/web/public");
let debug_dist = PathBuf::from("target/dx/gorilla_client/debug/web/public");
let client_dist = if release_dist.exists() { release_dist } else { debug_dist };

let app = if client_dist.exists() {
    tracing::info!("Serving Wasm PWA from {} at /*", client_dist.display());

    let index_path = client_dist.join("index.html");
    let patched_html = if index_path.exists() {
        let raw = std::fs::read_to_string(&index_path).unwrap_or_default();
        raw.replace("</head>", concat!(
            "<link rel=\"manifest\" href=\"/manifest.json\">\n",
            "<meta name=\"theme-color\" content=\"#00ff41\">\n",
            "<meta name=\"apple-mobile-web-app-capable\" content=\"yes\">\n",
            "<meta name=\"apple-mobile-web-app-status-bar-style\" content=\"black-translucent\">\n",
            "<link rel=\"apple-touch-icon\" href=\"/assets/icon-192.png\">\n",
            "<script>if('serviceWorker' in navigator){",
            "navigator.serviceWorker.register('/sw.js',{scope:'/'});",
            "}</script>\n",
            "</head>"
        ))
    } else {
        String::new()
    };

    // SPA fallback: serve patched HTML for all unmatched routes
    let fallback = {
        let html = patched_html;
        tower::service_fn(move |_req| {
            let body = html.clone();
            async move {
                Ok::<_, Infallible>(Response::builder()
                    .header("content-type", "text/html; charset=utf-8")
                    .body(Body::from(body))
                    .unwrap())
            }
        })
    };

    let serve_dir = ServeDir::new(&client_dist).fallback(fallback);
    app.nest_service("/app", serve_dir)
} else {
    tracing::info!("No Wasm bundle found — skipping PWA routes");
    app
};
```

### How This Works

1. **Read index.html** from the Dioxus build output directory.
2. **Patch it** by injecting PWA tags before `</head>`:
   - `<link rel="manifest">` — Tells the browser where the manifest lives
   - `<meta name="theme-color">` — Colors the browser status bar
   - `<meta name="apple-mobile-web-app-capable">` — iOS-specific PWA flag
   - `<link rel="apple-touch-icon">` — iOS home screen icon
   - `<script>` — Service worker registration
3. **Create an SPA fallback** — Any route under `/` that doesn't match
   a static file serves the patched HTML. This is what makes client-side
   routing work: `/dashboard`, `/training`, etc. all return the
   same HTML, and the Dioxus router handles the rest.
4. **Nest under `/`** — The entire bundle is scoped to `/*`, keeping
   it separate from the v1 HTML routes and the `/api/` endpoints.

### Release vs Debug Detection

The server checks for the release directory first, falling back to debug.
This means you can use `dx build` (debug, faster) during development and
`dx build --release` (optimized, smaller) for production — the server
adapts automatically.

If no built client exists at all, the server starts normally but without
PWA routes. The v1 HTML interface still works.

---

# Part VIII — Performance & Size Optimization

## Binary Size Budget

Every byte of the `.wasm` binary is a download penalty. On a good
connection, 1 MiB takes ~100ms. On 3G, it's 3 seconds. The profile
settings from Part II reduce the binary significantly:

| Technique | Approximate Savings |
|---|---|
| `opt-level = "z"` | 15–25% smaller than `opt-level = 3` |
| `lto = true` | 10–20% (eliminates dead code across crate boundaries) |
| `codegen-units = 1` | 5–10% (better inlining decisions) |
| `panic = "abort"` | ~50 KiB (removes unwinding tables) |
| `strip = true` | 10–30% (removes debug names) |
| Tree-shaking (Wasm) | Varies — unused functions not linked |

## DOM Update Minimization

The virtual DOM diff algorithm in Dioxus tries to produce the minimal
set of DOM operations. But the algorithm is only as good as the
component structure:

**Good**: Many small components with clear signal boundaries.

```rust
// Each StatCard subscribes only to its own data.
// Changing `d.steps` doesn't re-render the HRV card.
StatCard { label: "STEPS", value: fmt_opt(d.steps), color: "green" }
StatCard { label: "HRV", value: fmt_opt_f(hrv, 0), unit: "ms", color: "cyan" }
```

**Bad**: One giant component that reads every signal.

```rust
// If *any* signal changes, the *entire* component re-renders.
fn EverythingPage() -> Element {
    let a = use_signal(|| 0);
    let b = use_signal(|| 0);
    let c = use_signal(|| 0);
    rsx! {
        div { "a={a}, b={b}, c={c}" }  // reads all three signals
    }
}
```

This is the same principle as React's component splitting, but signals
make it even more powerful — you can have fine-grained reactivity
*within* a single component.

## Lazy Loading and Code Splitting

Dioxus does not currently support route-level code splitting for Wasm
targets (as of 0.7). The entire application compiles into a single
`.wasm` binary. For a focused app like Gorilla Coach, this is acceptable —
the binary is small enough that the up-front download cost is negligible
compared to the CDN-loaded Chart.js dependency.

If code splitting becomes important, the strategies are:

1. **Dynamic `import()`** for JavaScript dependencies (already done with
   Chart.js via CDN)
2. **Separate Wasm modules** loaded on demand (requires manual `wasm-bindgen`
   setup, not supported by Dioxus natively yet)

## HTTP Caching

The service worker provides application-level caching, but the server
should also set HTTP cache headers:

- **Wasm binary**: Long `Cache-Control: max-age` with content-hash in
  filename (Dioxus does this automatically)
- **CSS/JS**: Same strategy
- **index.html**: `Cache-Control: no-cache` (always check for updates)
- **API responses**: `no-store` (dynamic data)

The service worker's stale-while-revalidate strategy for static assets
means users see the cached version immediately, then get updates on the
next visit.

---

# Part IX — Patterns & Pitfalls

## Pattern: Async Task Spawning

When a user clicks a button, you need to run async code (API calls).
Dioxus provides `spawn` for this:

```rust
button {
    onclick: move |_| {
        syncing.set(true);
        sync_msg.set(Some("Syncing...".to_string()));

        spawn(async move {
            let result = api::trigger_sync().await;
            match result {
                Ok(msg) => sync_msg.set(Some(msg)),
                Err(e) if e.is_unauthorized() => {
                    // Redirect to login
                    navigator().push(Route::Login {});
                }
                Err(e) => sync_msg.set(Some(format!("Error: {e}"))),
            }
            syncing.set(false);
            dashboard.restart();  // Re-fetch data
        });
    },
    "Sync"
}
```

`spawn` launches a `Future` on the Wasm executor. It's non-blocking —
the UI remains responsive while the API call is in flight. The signal
updates inside the `async move` block trigger UI re-renders as they happen.

### Resource Restart

`dashboard.restart()` is a method on `Resource<T>` that re-runs the
async fetch. This is how you force a refresh after a side effect (sync,
save, delete) — you don't need to manually track a "refetch" signal.

## Pattern: Form Handling

The Chat page demonstrates form handling with `onsubmit` and `prevent_default`:

```rust
form { class: "chat-input-form",
    onsubmit: move |e| {
        e.prevent_default();
        let msg = input.read().clone();
        send_message(msg);
    },
    input {
        class: "chat-input",
        r#type: "text",
        placeholder: "Command...",
        value: "{input}",
        oninput: move |e| input.set(e.value()),
    }
    button { r#type: "submit", "SEND" }
}
```

The `value: "{input}"` binding + `oninput` handler create a controlled
input (React terminology: the signal is the source of truth, not the
DOM element).

## Pattern: SSE Streaming (Manual Parsing)

The chat endpoint returns Server-Sent Events. Since Dioxus/Wasm doesn't
have a native SSE client, the app makes a regular POST and manually
parses the response body:

```rust
let resp = Request::post("/api/chat/stream")
    .header("Content-Type", "application/x-www-form-urlencoded")
    .header("Accept", "text/event-stream")
    .body(form_body)
    .unwrap()
    .send()
    .await;

let text = response.text().await.unwrap_or_default();
for line in text.lines() {
    let line = line.trim();
    if !line.starts_with("data: ") { continue; }
    let json_str = &line[6..];
    if let Ok(data) = serde_json::from_str::<serde_json::Value>(json_str) {
        if let Some(token) = data.get("token").and_then(|v| v.as_str()) {
            raw_text.push_str(token);
            messages.write()[msg_idx].1 = md_to_html(&raw_text);
        }
    }
}
```

This isn't true streaming — it waits for the full response body. For a
self-hosted single-user app, this is acceptable because:

1. The LLM response arrives quickly over localhost or Tailscale.
2. The UI updates all at once rather than token-by-token, which is
   actually less distracting for report-style output.

For true token-by-token streaming, you would use `web_sys::EventSource`
(which Gorilla Coach enables in its Cargo.toml features) or the Fetch API's
`ReadableStream`.

## Pattern: Conditional Rendering

RSX supports inline conditionals naturally:

```rust
// if-expression
if !activities.is_empty() {
    div { class: "activities-section",
        for act in activities.iter() {
            ActivityCard { activity: act.clone() }
        }
    }
}

// if-let (destructuring)
if let Some(v) = avg_hr {
    MiniCard { label: "Avg HR", value: format!("{v}"), color: "red" }
}

// match expression
match &*dashboard.read() {
    Some(Ok(data)) => rsx! { DayView { data: data.clone() } },
    Some(Err(e)) => rsx! { div { "Error: {e}" } },
    None => rsx! { div { "Loading..." } },
}
```

There is no JSX-style `{condition && <Component />}` — Rust's
`if`/`match` expressions are more powerful and more readable.

## Pitfall: Select Element Values

HTML `<select>` elements set their `value` property based on which
`<option>` has a matching `value` attribute. But Dioxus renders the
`<select>`'s `value` attribute *before* rendering the child `<option>`
elements (because it processes attributes before children). This means
the browser ignores the value — there's no matching option yet.

The workaround is a `use_effect` that pokes the DOM after render:

```rust
use_effect(move || {
    let val = selected_file.read().clone().unwrap_or_default();
    if !val.is_empty() {
        spawn(async move {
            gloo_timers::future::TimeoutFuture::new(50).await;
            // query the DOM and set the value manually
        });
    }
});
```

The 50ms `setTimeout` gives the browser time to process the VDOM diff
and insert the `<option>` elements. This is an unfortunate but necessary
escape hatch.

## Pitfall: Closure Move Semantics

Every event handler in RSX must be `move` — it captures signals by value
(which is cheap; signals are `Copy`). But if you need to capture a
non-Copy value:

```rust
// THIS WON'T COMPILE — String is not Copy
let name = "hello".to_string();
button {
    onclick: move |_| { println!("{name}"); },  // moves name
    "Click"
}
button {
    onclick: move |_| { println!("{name}"); },  // ERROR: name already moved
    "Click 2"
}

// FIX: Clone before each closure
let name = "hello".to_string();
let name2 = name.clone();
button { onclick: move |_| { println!("{name}"); }, "Click" }
button { onclick: move |_| { println!("{name2}"); }, "Click 2" }
```

This is why Dioxus signals exist — they are `Copy`, so they can be
captured by multiple closures without cloning.

## Pitfall: `dangerous_inner_html`

The `dangerous_inner_html` attribute bypasses Dioxus's VDOM and injects
raw HTML. This is necessary for rendered markdown (the output of
`pulldown-cmark`) but is a potential XSS vector if the input is
user-controlled.

In Gorilla Coach, the markdown comes from the LLM (server-side), not
from user input injected directly. The server's CSRF middleware and
auth cookies prevent unauthorized content injection. Still, if you add
a feature where users write markdown that other users see, you would
need to sanitize the HTML output (**OWASP** A03:2021, Injection).

---

## Summary: The Full Request Lifecycle

To tie it all together, here's what happens when a user opens the
Dashboard:

```
1. Browser requests /dashboard
2. Service worker checks cache → cache-first
   a. If cached: return immediately (stale-while-revalidate)
   b. If not: fetch from server → Axum serves patched index.html
3. Browser parses HTML, loads:
   - gorilla_coach_bg.wasm (compiled Rust client)
   - gorilla_coach.js (wasm-bindgen glue)
   - main.css, Chart.js, charts.js
4. Wasm initializes → Dioxus mounts Router → matches /dashboard
5. Dashboard component renders:
   - use_signal creates `view` ("day") and `date` (today)
   - use_resource fires: GET /api/v2/dashboard?view=day&date=2026-02-18
6. Service worker intercepts the fetch → network-first
   a. If online: server → Axum handler → DB query → JSON response
   b. If offline: serve cached API response
7. Resource receives response → Some(Ok(DashboardResponse))
   - DayView renders: StatCards, ActivityCards, HR Zones
   - use_effect fires: Chart.js renders trend charts via js_sys::eval
8. User changes view to "week":
   - view.set("week") → signal subscribers notified
   - use_resource re-fires: GET /api/v2/dashboard?view=week&date=...
   - RangeView renders: summary averages, Chart.js charts, data table
9. All state lives in Wasm memory (no JS framework overhead)
   No GC pauses, no prototype chain lookups, no dynamic dispatch
```

Every step in this lifecycle is type-checked at compile time except the
JSON deserialization (which serde validates at runtime against the shared
`DashboardResponse` struct). The service worker provides resilience, the
Wasm binary provides performance, and Dioxus provides productivity.

> **CA §22, "The Clean Architecture"**: The dependency arrows point
> inward. The UI components depend on the API types, which depend on the
> domain types. Nothing depends on the UI. The server can evolve
> independently as long as the shared types remain compatible.

---

## Further Reading

- [WebAssembly Specification](https://webassembly.github.io/spec/) — The
  formal spec for understanding the VM
- [Rust and WebAssembly Book](https://rustwasm.github.io/docs/book/) —
  Official Rust Wasm working group guide
- [Dioxus Documentation](https://dioxuslabs.com/learn/0.7/) — Framework
  reference for 0.7
- [MDN: Progressive Web Apps](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps) —
  Comprehensive PWA guide
- [web.dev: Service Worker Lifecycle](https://web.dev/articles/service-worker-lifecycle) —
  Deep dive into SW states and events
- [The `pulldown-cmark` Crate](https://docs.rs/pulldown-cmark/) —
  CommonMark parser used for Markdown rendering
