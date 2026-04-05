# gapi

Standalone Garmin Connect API service with event-driven architecture and a full health dashboard. Handles OAuth authentication, background data sync, encrypted credential storage, webhook-based event dispatch, and a Leptos WASM dashboard.

## Stack

| Layer | Technology |
|-------|-----------|
| Backend | Rust (Axum 0.8, Tokio) |
| Frontend | Leptos 0.7 (CSR, WASM) |
| Database | SQLite (rusqlite + r2d2, WAL mode) |
| Encryption | ChaCha20Poly1305 |
| Deployment | Docker + Tailscale |

## Features

- Garmin Connect SSO authentication (OAuth1/OAuth2 + MFA)
- 14 health endpoints synced in parallel: HR, HRV, sleep, stress, body battery, SpO2, respiration, weight, training readiness, VO2 max, fitness age, race predictions, activities, steps
- 50+ daily metrics + intraday time series
- Webhook events with HMAC-SHA256 signing
- REST API for downstream consumers ([gorilla_coach](gorilla-coach.md), [lifemanager](lifemanager.md))
- Cyberpunk-themed health dashboard (7 pages)

## Dashboard Screenshots

<div class="screenshot-grid">
  <figure>
    <img src="/assets/screenshots/gapi/dashboard.png" alt="Dashboard" width="300">
    <figcaption>Dashboard — recovery score, vitals, alerts</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/gapi/heart.png" alt="Heart & Body" width="300">
    <figcaption>Heart & Body — HR, HRV, stress, SpO2</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/gapi/sleep.png" alt="Sleep" width="300">
    <figcaption>Sleep — debt, stages, efficiency</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/gapi/training.png" alt="Training" width="300">
    <figcaption>Training — readiness, VO2 max, load</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/gapi/activity.png" alt="Activity" width="300">
    <figcaption>Activity — consistency, volume</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/gapi/trends.png" alt="Trends" width="300">
    <figcaption>Trends — 30+ charts, correlations</figcaption>
  </figure>
</div>

<div class="project-banner gorilla" style="margin-top: 3rem;">

<a href="https://github.com/elmomk/gapi" class="md-button">View on GitHub</a>

</div>
