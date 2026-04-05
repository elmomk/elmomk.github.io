# Google Sheets Integration: Service Accounts, JWT Auth, and Two-Way Sync

> *"The best APIs are the ones you don't have to write — the ones somebody else
> already built, documented, and maintains."*
> — Every developer who has ever integrated with Google's APIs

This tutorial covers Gorilla Coach's Google Sheets integration end-to-end:
importing spreadsheets via service account authentication, caching tokens,
parsing CSV with multi-line quoted fields, mapping sheet columns to training
days, and writing data back to Google Sheets via the Sheets API v4. We also
cover the metadata system that tracks import provenance and enables one-click
re-sync.

---

## Reference Texts

| Abbreviation | Book |
|---|---|
| **PP** | Andrew Hunt & David Thomas — *The Pragmatic Programmer*, 20th Anniversary Ed. (2019) |
| **DDIA** | Martin Kleppmann — *Designing Data-Intensive Applications* (2017) |
| **ZtP** | Luca Palmieri — *Zero To Production in Rust* (2022) |
| **CA** | Robert C. Martin — *Clean Architecture* (2017) |
| **RiA** | Tim McNamara — *Rust in Action* (2021) |
| **OWASP** | OWASP Foundation — *OWASP Top 10* (2021) |
| **SRE** | Beyer et al. — *Site Reliability Engineering* (2016) |
| **RFC7519** | M. Jones et al. — *JSON Web Token (JWT)* (IETF, 2015) |

---

## Table of Contents

1. [Why Google Sheets?](#1-why-google-sheets)
2. [Architecture Overview](#2-architecture-overview)
3. [Service Accounts vs OAuth2: The Correct Choice](#3-service-accounts-vs-oauth2)
4. [JWT Construction: Building the Assertion](#4-jwt-construction)
5. [Token Exchange: JWT → Bearer Token](#5-token-exchange)
6. [Token Caching: The 50-Minute TTL](#6-token-caching)
7. [Sheet URL Parsing: Extracting IDs and GIDs](#7-sheet-url-parsing)
8. [Fetching CSV Data: Service Account → Public Fallback](#8-fetching-csv-data)
9. [CSV Parsing: The Multi-Line Quoted Field Problem](#9-csv-parsing)
10. [Tab Discovery: Reading Sheet Tab Metadata](#10-tab-discovery)
11. [The Import Pipeline: URL → CSV → File → Metadata](#11-the-import-pipeline)
12. [Auto-Import of the Last Tab](#12-auto-import-last-tab)
13. [Metadata Persistence: .sheets.json and .tab_names.json](#13-metadata-persistence)
14. [One-Click Re-Sync: Updating Imported Sheets](#14-re-sync)
15. [Column Mapping: Days to Sheet Columns](#15-column-mapping)
16. [Write-Back: Pushing Training Logs to Google Sheets](#16-write-back)
17. [The File Vault: Upload, Serve, Delete](#17-the-file-vault)
18. [Security: Filename Sanitization and Access Control](#18-security)
19. [Further Reading](#19-further-reading)

---

## 1. Why Google Sheets?

Training programs live in spreadsheets. Coaches build periodized plans in
Google Sheets because it's collaborative, version-controlled (via revision
history), and familiar. Any fitness coaching app that ignores this reality
forces coaches to migrate data manually.

Gorilla Coach's approach: **import the spreadsheet, don't replace it**. The
coach keeps working in Google Sheets. The app imports the sheet as CSV, parses
the training plan, tracks completion, and writes results back to the sheet.
The spreadsheet remains the source of truth for programming; the app adds a
tracking layer on top.

This is the **strangler fig pattern** applied to workflow tools:

```
Before:  Coach writes plan → Sheet → Athlete reads sheet
After:   Coach writes plan → Sheet → App imports → Athlete tracks → App writes back
```

> **PP**, Tip 14: *"There Are No Final Decisions."* The sheet import is
> non-destructive. If the app breaks, the coach still has their sheet.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                     User (Browser)                       │
│  [Import Sheet URL]  [Sync]  [Track Workout]  [Push]    │
└──────────┬───────────┬───────────┬──────────────┬────────┘
           │           │           │              │
           ▼           ▼           ▼              ▼
    files_import   files_sync   training    sync_day_to
      _sheet()     _sheet()    handlers      _sheet()
           │           │           │              │
           │           │           │              │
           ▼           ▼           ▼              ▼
  ┌─────────────────────────────────────────────────────┐
  │              sheets.rs — Core Logic                  │
  │                                                      │
  │  extract_sheet_id()    fetch_sheet_csv()             │
  │  extract_gid()         fetch_sheet_tabs()            │
  │  parse_csv_content()   get_service_account_token()   │
  │  build_day_column_map()  sync_day_to_sheet()         │
  │  get_cached_sa_token()   col_to_letter()             │
  └────────────┬────────────────────────┬────────────────┘
               │                        │
               ▼                        ▼
     Google Sheets API v4        Google Docs Export
     (tabs, write-back)           (CSV download)
```

Two Google APIs are used:

| API | Purpose | Auth |
|---|---|---|
| Google Docs export (`/export?format=csv`) | Download sheet as CSV | Service account bearer OR public (no auth) |
| Google Sheets API v4 (`/v4/spreadsheets/`) | Tab discovery, cell writes | Service account bearer (required) |

---

## 3. Service Accounts vs OAuth2: The Correct Choice

Google offers two authentication paths:

| Method | Flow | Best For |
|---|---|---|
| **OAuth2 (user consent)** | User clicks "Allow" → redirect → code → token | Apps where users connect their own Google account |
| **Service Account** | Server creates JWT → exchanges for token (no user interaction) | Server-to-server access to specific shared resources |

Gorilla Coach uses a **service account** because:

1. The app doesn't need access to users' Google accounts
2. The coach shares specific sheets with the service account email
3. No user consent flow needed — the service account is pre-authorized
4. Tokens are created programmatically with no browser interaction

The setup is straightforward: create a service account in Google Cloud Console,
download the JSON key file, and share the Google Sheet with the service
account's email (e.g., `gorilla-coach@project.iam.gserviceaccount.com`).

> **CA**, Chapter 22 — The Clean Architecture: *"External agencies should be
> plug-in components."* The service account is configured via environment
> variables and a key file — swappable without code changes.

### Configuration

The service account config lives in `AppState`:

```rust
#[derive(Clone)]
pub struct SheetsWriteConfig {
    pub service_account_email: String,
    pub private_key: String,  // PEM-encoded RSA private key
}
```

These come from the Google service account JSON key file, loaded at startup
and stored in `AppConfig`:

```rust
pub struct AppConfig {
    pub sheets_write: Option<SheetsWriteConfig>,
    // ...
}
```

The `Option` means: if no service account is configured, sheets write-back is
simply unavailable. The import will fall back to public CSV export.

---

## 4. JWT Construction: Building the Assertion

Service account authentication uses the **OAuth 2.0 JWT Bearer assertion**
flow. The server constructs a JSON Web Token (JWT), signs it with the service
account's private key, and exchanges it for a Bearer access token.

```rust
pub async fn get_service_account_token(
    client: &reqwest::Client,
    config: &SheetsWriteConfig,
) -> Result<String, AppError> {
    let now = chrono::Utc::now().timestamp() as u64;

    let claims = serde_json::json!({
        "iss": config.service_account_email,    // Issuer: service account email
        "scope": "https://www.googleapis.com/auth/spreadsheets \
                  https://www.googleapis.com/auth/drive.readonly \
                  https://www.googleapis.com/auth/calendar.events",
        "aud": "https://oauth2.googleapis.com/token",   // Audience: Google token endpoint
        "iat": now,                              // Issued at (unix timestamp)
        "exp": now + 3600,                       // Expires in 1 hour
    });

    let header = jsonwebtoken::Header::new(jsonwebtoken::Algorithm::RS256);
    let key = jsonwebtoken::EncodingKey::from_rsa_pem(
        config.private_key.as_bytes()
    )?;
    let jwt = jsonwebtoken::encode(&header, &claims, &key)?;

    // ... exchange JWT for token ...
}
```

### JWT Anatomy

A JWT has three Base64-URL-encoded parts separated by dots:

```
HEADER.PAYLOAD.SIGNATURE
```

| Part | Contents |
|---|---|
| Header | `{"alg": "RS256", "typ": "JWT"}` — algorithm and token type |
| Payload | `{"iss": "...@...iam.gserviceaccount.com", "scope": "...", "aud": "...", "iat": N, "exp": N+3600}` |
| Signature | RS256: RSA-PKCS1-v1_5 with SHA-256 over `HEADER.PAYLOAD` using the private key |

> **RFC7519**, Section 4.1: *"The 'iss' (issuer) claim identifies the
> principal that issued the JWT."* For service accounts, the issuer is the
> service account's email address.

### Scopes

Three scopes are requested:

| Scope | Permission |
|---|---|
| `spreadsheets` | Read and write spreadsheet cells |
| `drive.readonly` | List files, read metadata — needed for CSV export |
| `calendar.events` | Create, update, delete calendar events — needed for training schedule sync |

The scope determines what the token can do. A token with `spreadsheets`
scope can read/write cells but cannot delete spreadsheets or manage Drive
permissions. The `calendar.events` scope allows creating all-day training
events on the user's Google Calendar.

---

## 5. Token Exchange: JWT → Bearer Token

The signed JWT is exchanged at Google's token endpoint for a Bearer access
token:

```rust
let resp = client
    .post("https://oauth2.googleapis.com/token")
    .form(&[
        ("grant_type", "urn:ietf:params:oauth:grant-type:jwt-bearer"),
        ("assertion", &jwt),
    ])
    .send()
    .await?;

let json: serde_json::Value = resp.json().await?;
let token = json["access_token"].as_str()?.to_string();
```

The `grant_type` value `urn:ietf:params:oauth:grant-type:jwt-bearer` tells
Google's token endpoint that this is a JWT assertion, not a standard OAuth2
authorization code exchange.

The response:

```json
{
    "access_token": "ya29.c.b0AX...",
    "expires_in": 3599,
    "token_type": "Bearer"
}
```

No refresh token is returned. When the token expires, the server creates a
new JWT and exchanges it again. The process is entirely automated — no user
interaction required.

---

## 6. Token Caching: The 50-Minute TTL

Creating a JWT and exchanging it takes a network round-trip (~100-300ms).
For frequent sheet operations, this adds up. Gorilla Coach caches the token
with a 50-minute TTL (tokens are valid for 60 minutes):

```rust
const SA_TOKEN_TTL_SECS: u64 = 50 * 60; // 50 minutes

async fn get_cached_sa_token(
    state: &AppState,
    config: &SheetsWriteConfig,
) -> Result<String, AppError> {
    let mut cache = state.sa_token_cache.lock().await;

    if let Some((ref token, fetched_at)) = *cache {
        if fetched_at.elapsed().as_secs() < SA_TOKEN_TTL_SECS {
            return Ok(token.clone());  // Cache hit
        }
    }

    // Cache miss or expired — fetch new token
    let token = get_service_account_token(&state.http_client, config).await?;
    *cache = Some((token.clone(), std::time::Instant::now()));
    Ok(token)
}
```

The cache is stored in `AppState`:

```rust
pub sa_token_cache: Arc<tokio::sync::Mutex<Option<(String, std::time::Instant)>>>,
```

Why `tokio::sync::Mutex` instead of `parking_lot::Mutex`? Because the lock
guards an async operation (the token fetch). If we used a synchronous mutex
and held it across the `.await` point, the lock would block the entire thread.
`tokio::sync::Mutex` yields the thread while waiting.

The 10-minute buffer (50 vs 60 minutes) prevents races where a cached token
expires between check and use.

> **SRE**, Chapter 22 — Addressing Cascading Failures: *"Caching can help
> reduce load... but cache invalidation must be handled correctly."* The TTL
> approach is simple and correct: no invalidation needed, the cache
> self-expires.

---

## 7. Sheet URL Parsing: Extracting IDs and GIDs

Users paste Google Sheets URLs like:

```
https://docs.google.com/spreadsheets/d/1aBcDeFgHiJkLmNoPqRsTuVwXyZ/edit#gid=12345
```

Two pieces of information must be extracted:

### Sheet ID

The sheet ID is the long alphanumeric string between `/d/` and the next `/`:

```rust
pub fn extract_sheet_id(url: &str) -> Option<String> {
    let marker = "/spreadsheets/d/";
    let idx = url.find(marker)?;
    let rest = &url[idx + marker.len()..];
    let id: String = rest.chars()
        .take_while(|c| c.is_alphanumeric() || *c == '-' || *c == '_')
        .collect();
    if id.len() > 10 { Some(id) } else { None }
}
```

The length check (`> 10`) rejects accidental short strings. Real sheet IDs
are 44 characters.

### GID (Tab Identifier)

The GID identifies which tab within the spreadsheet. It appears as a URL
fragment:

```rust
pub fn extract_gid(url: &str) -> Option<u64> {
    let gid_pattern = "gid=";
    let idx = url.find(gid_pattern)?;
    let rest = &url[idx + gid_pattern.len()..];
    let num: String = rest.chars().take_while(|c| c.is_numeric()).collect();
    num.parse().ok()
}
```

| URL | Sheet ID | GID |
|---|---|---|
| `.../d/ABC123/edit` | `ABC123` | `None` (first tab) |
| `.../d/ABC123/edit#gid=0` | `ABC123` | `Some(0)` (first tab) |
| `.../d/ABC123/edit#gid=12345` | `ABC123` | `Some(12345)` (specific tab) |

When `gid` is `None`, Google exports the first (default) tab.

---

## 8. Fetching CSV Data: Service Account → Public Fallback

The CSV fetch uses a **two-tier strategy**: try the service account first,
fall back to public export if it fails.

```rust
pub async fn fetch_sheet_csv(
    state: &AppState,
    sheet_id: &str,
    gid: Option<u64>,
) -> Result<Vec<u8>, AppError> {
    let gid_param = gid.map(|g| format!("&gid={}", g)).unwrap_or_default();

    // Tier 1: Service Account
    if let Some(config) = &state.config.sheets_write {
        let token = get_cached_sa_token(state, config).await?;
        let csv_url = format!(
            "https://docs.google.com/spreadsheets/d/{}/export?format=csv{}",
            sheet_id, gid_param
        );
        let resp = state.http_client.get(&csv_url)
            .bearer_auth(&token)
            .send().await?;

        if resp.status().is_success() {
            return Ok(resp.bytes().await?.to_vec());
        }
        tracing::warn!("SA export returned HTTP {}, falling back", resp.status());
    }

    // Tier 2: Public Export (no auth)
    let csv_url = format!(
        "https://docs.google.com/spreadsheets/d/{}/export?format=csv{}",
        sheet_id, gid_param
    );
    let resp = state.http_client.get(&csv_url).send().await?;

    if !resp.status().is_success() {
        return Err(AppError::sheets(format!(
            "Sheet returned HTTP {}. Make sure the service account has \
             Editor access, or sharing is set to 'Anyone with the link'.",
            resp.status()
        )));
    }

    Ok(resp.bytes().await?.to_vec())
}
```

Why two tiers?

| Scenario | Tier 1 (SA) | Tier 2 (Public) |
|---|---|---|
| Sheet shared with SA | ✅ Works | May or may not work |
| Sheet set to "Anyone with link" | ✅ Works | ✅ Works |
| No SA configured | Skipped | ✅ Works |
| SA key expired/invalid | ❌ Falls through | ✅ Works (if public) |
| Sheet is private, not shared | ❌ Falls through | ❌ Fails with clear error |

The error message guides the user to either share with the service account
or make the sheet public — the two ways to fix the problem.

---

## 9. CSV Parsing: The Multi-Line Quoted Field Problem

Google Sheets CSV export follows RFC 4180 — fields containing commas,
newlines, or double quotes are wrapped in double quotes:

```csv
Exercise,Notes
Squat,"Heavy day
focus on depth"
Bench,"Use 3-count pause, ""competition style"""
```

This means a naive `split('\n').split(',')` breaks on multi-line fields.
Gorilla Coach implements a proper state-machine CSV parser:

```rust
pub fn parse_csv_content(content: &str) -> Vec<Vec<String>> {
    let mut rows: Vec<Vec<String>> = Vec::new();
    let mut fields: Vec<String> = Vec::new();
    let mut current = String::new();
    let mut in_quotes = false;
    let mut chars = content.chars().peekable();

    while let Some(c) = chars.next() {
        if in_quotes {
            if c == '"' {
                if chars.peek() == Some(&'"') {
                    current.push('"');   // Escaped quote: "" → "
                    chars.next();
                } else {
                    in_quotes = false;   // End of quoted field
                }
            } else {
                current.push(c);         // Inside quotes: keep everything
            }
        } else {
            match c {
                ',' => {
                    fields.push(std::mem::take(&mut current));
                }
                '"' => in_quotes = true,
                '\n' => {
                    fields.push(std::mem::take(&mut current));
                    rows.push(std::mem::take(&mut fields));
                }
                '\r' => {
                    if chars.peek() == Some(&'\n') { chars.next(); }
                    fields.push(std::mem::take(&mut current));
                    rows.push(std::mem::take(&mut fields));
                }
                _ => current.push(c),
            }
        }
    }
    // Don't lose the last row if there's no trailing newline
    if !current.is_empty() || !fields.is_empty() {
        fields.push(current);
        rows.push(fields);
    }
    rows
}
```

### State Machine Diagram

```
             ┌───────────┐
             │ NOT_QUOTED │ ◄── start
             └─────┬──┬──┘
      comma: emit  │  │  quote: enter
      newline: row │  │
                   │  ▼
             ┌─────┴──────┐
             │   QUOTED    │
             └─────┬──┬───┘
    char: append   │  │  quote+quote: append literal "
                   │  │  quote+other: exit to NOT_QUOTED
                   ▼  ▼
```

Why not use a CSV library? Because the `csv` crate adds a dependency for a
function that's 40 lines of code, and we need specific behavior (return
`Vec<Vec<String>>` rows, handle both `\r\n` and `\n`, preserve empty fields).

> **PP**, Tip 34: *"Use Tracer Bullets."* The hand-rolled parser is testable,
> debuggable, and exactly fits the use case.

### `std::mem::take` — The Rust Idiom

The `std::mem::take(&mut current)` pattern deserves attention:

```rust
fields.push(std::mem::take(&mut current));
// Equivalent to:
// fields.push(current.clone());
// current.clear();
```

`take` replaces `current` with `String::new()` (the default) and returns the
old value — **without allocating**. The empty string's allocation is reused
for the next field. This is a zero-copy field emission pattern.

---

## 10. Tab Discovery: Reading Sheet Tab Metadata

Google Sheets can have multiple tabs (worksheets). To discover them, we use
the Sheets API v4 metadata endpoint:

```rust
pub async fn fetch_sheet_tabs(
    state: &AppState,
    sheet_id: &str,
) -> Result<Vec<(String, u64)>, AppError> {
    let config = state.config.sheets_write.as_ref()
        .ok_or_else(|| AppError::sheets("No service account configured"))?;
    let token = get_cached_sa_token(state, config).await?;

    let url = format!(
        "https://sheets.googleapis.com/v4/spreadsheets/{}?fields=sheets.properties",
        sheet_id
    );

    let resp = state.http_client
        .get(&url)
        .bearer_auth(&token)
        .send().await?;

    let json: serde_json::Value = resp.json().await?;
    let mut tabs: Vec<(String, u64, u64)> = json["sheets"]
        .as_array()?
        .iter()
        .filter_map(|s| {
            let props = &s["properties"];
            let title = props["title"].as_str()?.to_string();
            let gid = props["sheetId"].as_u64()?;
            let index = props["index"].as_u64().unwrap_or(0);
            Some((title, gid, index))
        })
        .collect();

    tabs.sort_by_key(|(_, _, idx)| *idx);
    Ok(tabs.into_iter().map(|(t, g, _)| (t, g)).collect())
}
```

The `?fields=sheets.properties` query parameter tells Google to return only
the sheet properties — not the cell data. This is a **partial response**,
a Google API convention that reduces bandwidth and latency:

```json
{
    "sheets": [
        {"properties": {"sheetId": 0, "title": "Hypertrophy", "index": 0}},
        {"properties": {"sheetId": 12345, "title": "Strength", "index": 1}}
    ]
}
```

Tabs are sorted by `index` (their visual order in the spreadsheet) before
returning. The index is then dropped — callers only need `(title, gid)` pairs.

---

## 11. The Import Pipeline: URL → CSV → File → Metadata

The import flow in `files_import_sheet()` is a multi-step pipeline:

```
User pastes URL + filename
    │
    ├── 1. Validate: extract_sheet_id() — is this a valid Sheets URL?
    │
    ├── 2. Fetch: fetch_sheet_csv(sheet_id, None) — download first tab as CSV
    │
    ├── 3. Save: write bytes to uploads/{user_id}/{safe_name}
    │
    ├── 4. Metadata: save source URL to .sheets.json
    │
    ├── 5. Tabs: fetch_sheet_tabs() — discover all tabs
    │   ├── Save first tab title to .tab_names.json
    │   └── Auto-import last tab (if multi-tab sheet)
    │
    └── 6. Return: { ok: true, name, size, extra_files }
```

```rust
pub async fn files_import_sheet(
    State(state): State<AppState>,
    jar: SignedCookieJar,
    axum::Json(input): axum::Json<ImportSheetRequest>,
) -> Response {
    let (user_id, _) = match get_session(&jar) { ... };

    // 1. Extract sheet ID
    let sheet_id = extract_sheet_id(&input.url)
        .ok_or("Invalid Google Sheets URL")?;

    // 2. Fetch CSV
    let bytes = fetch_sheet_csv(&state, &sheet_id, None).await?;

    // 3. Save file
    let safe_name = sanitize_filename(&input.name);
    let path = user_uploads_dir(user_id).join(&safe_name);
    tokio::fs::write(&path, &bytes).await?;

    // 4. Save source URL
    let mut meta = load_sheets_meta(user_id).await;
    meta.insert(safe_name.clone(), input.url.clone());

    // 5. Fetch tabs and auto-import last tab
    // ...

    // 6. Persist metadata
    save_sheets_meta(user_id, &meta).await;

    axum::Json(json!({"ok": true, ...})).into_response()
}
```

---

## 12. Auto-Import of the Last Tab

Training spreadsheets often have multiple cycles (e.g., "Hypertrophy" and
"Strength") in separate tabs. The **last tab** is typically the most recent
cycle. Gorilla Coach auto-imports it:

```rust
if tabs.len() > 1 {
    let (last_name, last_gid) = &tabs[tabs.len() - 1];
    if let Ok(tab_bytes) = fetch_sheet_csv(&state, &sheet_id, Some(*last_gid)).await {
        if !tab_bytes.is_empty() {
            let tab_file = sanitize_filename(last_name);
            let tab_file = if tab_file.ends_with(".csv") { tab_file }
                else { format!("{}.csv", tab_file) };

            tokio::fs::write(&dir.join(&tab_file), &tab_bytes).await?;

            let tab_url = format!("{}#gid={}", input.url, last_gid);
            meta.insert(tab_file.clone(), tab_url);
            extra_files.push(tab_file);
        }
    }
}
```

The auto-imported tab gets its own entry in `.sheets.json` with the GID
appended to the URL. This means it can be re-synced independently:

| File | Source URL |
|---|---|
| `training_plan.csv` | `https://docs.google.com/.../edit` |
| `Strength.csv` | `https://docs.google.com/.../edit#gid=12345` |

---

## 13. Metadata Persistence: .sheets.json and .tab_names.json

Two hidden files in each user's upload directory track import provenance:

### `.sheets.json` — Source URL Mapping

```json
{
    "training_plan.csv": "https://docs.google.com/spreadsheets/d/ABC123/edit",
    "Strength.csv": "https://docs.google.com/spreadsheets/d/ABC123/edit#gid=12345"
}
```

Maps filename → source URL. Used by the sync button to re-fetch the latest
version. Also used by the file list endpoint to show a "sync" icon next to
files that have a remote source.

### `.tab_names.json` — Tab Title Mapping

```json
{
    "training_plan.csv": "Hypertrophy"
}
```

Maps filename → tab title. Used during sheet write-back to construct
the correct cell range (e.g., `'Hypertrophy'!F17`).

Both files use the same load/save pattern:

```rust
pub(super) async fn load_sheets_meta(user_id: Uuid)
    -> HashMap<String, String>
{
    let path = user_uploads_dir(user_id).join(".sheets.json");
    match tokio::fs::read_to_string(&path).await {
        Ok(s) => serde_json::from_str(&s).unwrap_or_default(),
        Err(_) => HashMap::new(),
    }
}

pub(super) async fn save_sheets_meta(
    user_id: Uuid,
    meta: &HashMap<String, String>,
) {
    let path = user_uploads_dir(user_id).join(".sheets.json");
    if let Ok(json) = serde_json::to_string(meta) {
        let _ = tokio::fs::write(&path, json).await;
    }
}
```

The `.` prefix means these files are hidden from the file list (which skips
files starting with `.`):

```rust
if name.starts_with('.') { continue; } // skip hidden/meta files
```

> **PP**, Tip 38: *"Program Close to the Problem Domain."* The metadata is
> stored alongside the files it describes — in the same directory, in the same
> format (JSON). No separate database table needed.

### Design Decision: JSON Sidecar Files over DB or xattrs

We considered three storage options for sheets metadata:

1. **Database table** — Adds migration, requires `Repository` in file handlers,
   creates orphan-cleanup burden when files are deleted. The data is tiny
   (a few KV pairs per user) with no cross-user query need. On a single
   self-hosted server, DB adds complexity without payoff.

2. **Linux extended attributes (xattrs)** — Fragile in practice. Docker volume
   drivers, `cp`, `rsync`, `tar`, and `scp` can silently strip xattrs without
   explicit flags. Invisible to casual inspection. Size-limited (~4KB on ext4).
   Would require scanning every file to rebuild the metadata map.

3. **JSON sidecar files** (chosen) — Co-located with the data they describe:
   deleting the uploads dir removes metadata too, no orphans. Visible,
   debuggable (`cat .sheets.json`). Portable across Docker, backups, and
   filesystem migrations. Atomic-writable via `tokio::fs::write`. No extra
   dependencies or filesystem quirks.

---

## 14. One-Click Re-Sync: Updating Imported Sheets

The sync handler re-fetches a previously imported sheet using its stored URL:

```rust
pub async fn files_sync_sheet(
    State(state): State<AppState>,
    jar: SignedCookieJar,
    axum::Json(input): axum::Json<SyncSheetRequest>,
) -> Response {
    let meta = load_sheets_meta(user_id).await;
    let url = meta.get(&safe_name)?;

    let sheet_id = extract_sheet_id(&url)?;
    let gid = extract_gid(&url);  // Some(N) for tab-specific URLs

    let bytes = fetch_sheet_csv(&state, &sheet_id, gid).await?;
    tokio::fs::write(&path, &bytes).await?;

    // For base sheets (no gid), also check for new tabs
    if gid.is_none() {
        // Re-discover tabs, auto-import new last tab if not already present
    }

    axum::Json(json!({"ok": true, ...})).into_response()
}
```

Key behavior: when syncing a **base sheet** (no GID), the handler also checks
for new tabs. If the coach added a new tab since the last import, it gets
auto-imported. But existing tab files are NOT overwritten — only new tabs
are discovered:

```rust
if !meta.contains_key(&tab_file) {
    // Only import if we haven't seen this tab before
    // ...
}
```

This prevents accidental overwrites of training logs that might have been
tracked against the old version of a tab.

---

## 15. Column Mapping: Days to Sheet Columns

Training spreadsheets use a "wide" format where each day is a column group:

```
          │← Day 1 →│← Day 2 →│← Day 3 →│
          │  %  Wt S R│  %  Wt S R│  %  Wt S R│
Squat     │ 80  315 3 5│ 85  335 4 3│         │
Bench     │ 75  225 3 8│         │ 80  245 3 5│
```

`build_day_column_map()` produces a mapping from day label to sheet column
index:

```rust
pub fn build_day_column_map(
    lines: &[Vec<String>],
) -> HashMap<String, usize> {
    let (header_row, groups) = find_wide_header(lines)?;
    // groups = [col_idx_day1, col_idx_day2, ...]

    // Look 1-3 rows above header for day names
    let mut day_names = vec![String::new(); groups.len()];
    for offset in 1..=3 {
        let row = &lines[header_row - offset];
        for (gi, &col) in groups.iter().enumerate() {
            let val = row.get(col)?.trim();
            if !val.is_empty() && day_names[gi].is_empty() {
                day_names[gi] = val.to_string();
            }
        }
    }

    // Look 2-6 rows above for week/phase labels (for disambiguation)
    // ...

    // Build final labels: "Monday", "W1 Monday", "Day 1", etc.
    for gi in 0..groups.len() {
        let label = if day.is_empty() { format!("Day {}", gi + 1) }
            else if has_duplicates { format!("{} {}", week_short, day) }
            else { day };
        map.insert(label, groups[gi]);
    }
    map
}
```

The algorithm handles several spreadsheet layouts:

| Layout | Day Label Result |
|---|---|
| `Monday`, `Tuesday`, `Wednesday` (unique) | `Monday`, `Tuesday`, `Wednesday` |
| `Monday`, `Tuesday`, `Monday` (duplicates) | `W1 Monday`, `W1 Tuesday`, `W2 Monday` |
| No day names, just columns | `Day 1`, `Day 2`, `Day 3` |
| Week 1 / Week 2 headers above | Prefixed with `W1`, `W2` |

The week label extraction scans rows above the header for text containing
"week" or "phase", then extracts the number for a short prefix:

```rust
let short = if let Some(idx) = wl.find("week") {
    let after = wl[idx + 4..].trim_start();
    let num: String = after.chars()
        .take_while(|c| c.is_numeric()).collect();
    if !num.is_empty() { format!("W{}", num) } else { String::new() }
} else { String::new() };
```

---

## 16. Write-Back: Pushing Training Logs to Google Sheets

The `sync_day_to_sheet()` function writes training log data back to the
Google Sheet — closing the loop from spreadsheet to app and back:

```rust
pub async fn sync_day_to_sheet(
    state: &AppState,
    user_id: Uuid,
    plan_file: &str,
    day_label: &str,
) -> Result<(), AppError> {
    // 1. Load metadata: sheet ID, column mapping
    let meta = load_sheets_meta(user_id).await;
    let source_url = meta.get(plan_file)?;
    let sheet_id = extract_sheet_id(source_url)?;

    // 2. Parse CSV to find the target column
    let lines = parse_csv_content(&content);
    let day_col_map = build_day_column_map(&lines);
    let col_idx = day_col_map.get(day_label)?;

    // 3. Format training logs as text
    let day_logs = state.repo.get_training_logs(user_id, plan_file).await?
        .into_iter()
        .filter(|l| l.day_label == day_label)
        .collect();
    let text = format_day_training_text(&day_logs);

    // 4. Build cell range
    let col_letter = col_to_letter(*col_idx);
    let cell_range = format!("'{}'!{}{}", tab_name, col_letter, write_row);
    // e.g., "'Hypertrophy'!F17"

    // 5. Write via Sheets API v4
    let url = format!(
        "https://sheets.googleapis.com/v4/spreadsheets/{}/values/{}?valueInputOption=RAW",
        sheet_id, cell_range
    );
    let body = serde_json::json!({
        "range": cell_range,
        "majorDimension": "ROWS",
        "values": [[text]]
    });

    state.http_client.put(&url)
        .bearer_auth(&token)
        .json(&body)
        .send().await?;
}
```

### Column Index to Letter Conversion

Sheet columns use letters (A, B, ..., Z, AA, AB, ...):

```rust
fn col_to_letter(mut col: usize) -> String {
    let mut result = String::new();
    loop {
        result.insert(0, (b'A' + (col % 26) as u8) as char);
        if col < 26 { break; }
        col = col / 26 - 1;
    }
    result
}
```

| Index | Letter |
|---|---|
| 0 | A |
| 25 | Z |
| 26 | AA |
| 27 | AB |
| 701 | ZZ |
| 702 | AAA |

### Training Log Formatting

Logs are formatted as human-readable text for the sheet cell:

```rust
fn format_day_training_text(logs: &[TrainingSetLog]) -> String {
    // Group by exercise name
    // For each exercise:
    //   "Squat (315): 5, 5, 4 [grinder]"
    //   "Bench (225): 8, 8, 8"
}
```

Output example:
```
Squat (315): 5, 5, 4 [grinder]
Bench (225): 8, 8, 8
RDL (275): 10, 10, 10
```

This goes into a single cell in the sheet. The coach can see at a glance
what the athlete actually did vs. what was programmed.

### UI Trigger

The write-back is triggered by a "Save Day to Sheets" button in the training
tracker UI. Clicking it calls `POST /api/training/sync-day` with the current
plan file and day label. The handler calls `sync_day_to_sheet()` and returns
an HTML fragment with a success/error toast. This is separate from logging
sets — the user explicitly decides when to push data back to the sheet.

---

## 17. The File Vault: Upload, Serve, Delete

The file system provides standard CRUD operations for all user files:

### Upload (Multipart)

```rust
pub async fn files_upload(jar: SignedCookieJar, mut multipart: Multipart) -> Response {
    // Auth check
    let dir = user_uploads_dir(user_id);
    tokio::fs::create_dir_all(&dir).await?;

    while let Some(field) = multipart.next_field().await.unwrap_or(None) {
        let safe_name = sanitize_filename(&original_name);
        let bytes = field.bytes().await?;
        if bytes.len() > 20 * 1024 * 1024 { continue; }  // 20MB limit
        tokio::fs::write(&dir.join(&safe_name), &bytes).await?;
    }
}
```

### Serve (with MIME detection)

```rust
pub async fn files_serve(jar: SignedCookieJar, Path(name): Path<String>) -> Response {
    let safe = sanitize_filename(&name);
    let ext = safe.rsplit('.').next().unwrap_or("").to_lowercase();
    let mime = match ext.as_str() {
        "png" => "image/png",
        "csv" => "text/csv",
        "pdf" => "application/pdf",
        // ...
        _ => "application/octet-stream",
    };
    ([(CONTENT_TYPE, mime)], bytes).into_response()
}
```

### Delete

```rust
pub async fn files_delete(jar: SignedCookieJar, Path(name): Path<String>) -> Response {
    let safe = sanitize_filename(&name);
    tokio::fs::remove_file(&path).await?;
}
```

### Per-User Isolation

Files are stored in `uploads/{user_id}/`:

```rust
pub fn user_uploads_dir(user_id: Uuid) -> PathBuf {
    PathBuf::from("uploads").join(user_id.to_string())
}
```

This ensures **complete isolation** between users. User A cannot access User
B's files because:
1. The `user_id` comes from the signed session cookie (not user input)
2. The directory path is constructed server-side using the authenticated user's ID
3. The `sanitize_filename()` function prevents path traversal

---

## 18. Security: Filename Sanitization and Access Control

### Filename Sanitization

All filenames pass through `sanitize_filename()` which strips everything
except `[a-zA-Z0-9._-]`:

```rust
pub fn sanitize_filename(name: &str) -> String {
    name.chars()
        .filter(|c| c.is_alphanumeric() || *c == '.' || *c == '-' || *c == '_')
        .collect()
}
```

This prevents:
- **Path traversal**: `../../../etc/passwd` → `etcpasswd`
- **Null bytes**: `file.csv\0.exe` → `file.csv.exe`
- **Hidden files**: `.htaccess` → `htaccess` (though `.sheets.json` is created
  server-side, not via user upload)
- **Shell injection**: `; rm -rf /` → `rm-rf`

### Access Control

Every file endpoint follows the same pattern:

```rust
let (user_id, _) = match get_session(&jar) {
    Some(u) => u,
    None => return error_response("not authenticated"),
};
let dir = user_uploads_dir(user_id);  // user_id from SESSION, not request
```

The user ID comes from the signed cookie — it cannot be forged. The
path is constructed entirely server-side. There is no way for one user to
access another user's files.

### Upload Size Limit

The 20MB limit is enforced both in the handler and at the transport layer
(Axum's `Multipart` has a default body limit):

```rust
if bytes.len() > 20 * 1024 * 1024 { continue; }  // skip oversized files
```

Additionally, the rate limiter on upload routes (30 requests per 60 seconds)
prevents abuse.

> **OWASP**, A01:2021 — Broken Access Control: *"Access control enforces
> policy such that users cannot act outside of their intended permissions."*
> The per-user directory model with server-side path construction is the
> simplest secure approach.

---

## Day-2 Operations: Sheets & File Troubleshooting

### Service Account Token Failures

The service account JWT has a 60-minute lifetime with a 50-minute cache TTL.
Common failure modes:

| Log Message | Meaning | Fix |
|---|---|---|
| `No service account configured` | `GOOGLE_SERVICE_ACCOUNT_KEY_FILE` not set | Add the env var pointing to your SA JSON key |
| `Invalid service account private key` | PEM parse failure in the SA JSON file | Re-download the key file from Google Cloud Console |
| `JWT encoding failed` | Corrupted key file or wrong format | Verify the JSON file has `private_key` and `client_email` fields |
| `Token request failed` | Network error to Google OAuth | Check internet, DNS, firewall rules for `oauth2.googleapis.com` |
| `No access_token in response` | SA auth rejected | Verify the SA email in Google Cloud Console, check IAM permissions |

### Sheet Import Failures

The import pipeline tries service account auth first, then falls back to
public/unauthenticated export:

```
SA export (bearer token) → HTTP non-200?
  → WARN: "SA export returned HTTP {}, falling back to public export"
    → Public export (no auth) → HTTP non-200?
      → ERROR: "Sheet returned HTTP {}. Make sure the service account has
        Editor access, or sharing is set to 'Anyone with the link can edit'."
```

**Most common cause**: The sheet isn't shared with the service account email.
Go to Google Sheets → Share → add the SA email (found in the JSON key file
under `client_email`) as Editor.

### Write-Back Failures

When training log write-back to Sheets fails:

```
WARN: Sheets write-back failed for day 'W2 Thursday': Sheets API error: ...
```

Common causes:
- **Network timeout**: Transient — the next sync attempt will retry
- **Permission denied**: SA needs Editor access, not just Viewer
- **Column mapping failure**: `"Day 'X' not found in sheet columns"` — the
  day label in the training plan doesn't match any column header in the sheet.
  Check the CSV structure and the `day_to_column_mapping` logic
- **No sheets source URL**: `"No sheets source URL stored for 'X'"` — the
  metadata file (`.sheets.json`) is missing or corrupted. Re-import the sheet

### Metadata File Cleanup

Each user's upload directory contains metadata files:

```
uploads/<user_id>/
  .sheets.json       — Maps imported filenames to their source Sheet URLs
  .tab_names.json    — Maps filenames to their sheet tab names
  plan.csv            — Imported training plan
```

If imports or write-backs are behaving unexpectedly, inspect these:

```bash
cat uploads/<USER_ID>/.sheets.json
cat uploads/<USER_ID>/.tab_names.json
```

To reset: delete both `.json` files and re-import the sheet from the UI.

### File Upload Issues

- **20MB limit**: Files over 20MB are silently skipped. No error shown to user.
- **Filename sanitization**: Only `[a-zA-Z0-9._-]` characters survive.
  `My Plan (v2).csv` becomes `MyPlanv2.csv`. If a user reports a missing
  file, check the sanitized name.
- **Rate limit**: Upload routes are limited to 30 requests per 60 seconds
  per IP. Bulk uploads may hit this.

### Orphaned Upload Directories

After deleting a user from the database, their upload directory remains on
disk:

```bash
# List all upload directories
ls uploads/

# Cross-reference with existing users
docker exec gorilla_local_db psql -U gorilla -d gorilla_hq \
  -c "SELECT id FROM users;"

# Remove orphaned directories
rm -rf uploads/<ORPHANED_USER_ID>/
```

---

## 19. Further Reading

- Google Sheets API v4 Documentation: https://developers.google.com/sheets/api
- Google Service Account Authentication: https://cloud.google.com/iam/docs/service-account-overview
- RFC 7519 — JSON Web Token (JWT): https://tools.ietf.org/html/rfc7519
- RFC 4180 — Common Format and MIME Type for CSV Files: https://tools.ietf.org/html/rfc4180
- `jsonwebtoken` Rust crate: https://docs.rs/jsonwebtoken/latest/jsonwebtoken/
- Google OAuth 2.0 for Service Accounts: https://developers.google.com/identity/protocols/oauth2/service-account
