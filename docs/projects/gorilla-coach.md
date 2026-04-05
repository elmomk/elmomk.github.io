# gorilla_coach

Full-stack fitness coaching app built entirely in Rust.

## Stack

| Layer | Technology |
|-------|-----------|
| Frontend | [Leptos](https://leptos.dev/) (Rust WASM) |
| Backend | [Axum](https://github.com/tokio-rs/axum) |
| Database | PostgreSQL |
| Deployment | Docker |

## Features

- Strength training program management
- Mesocycle and macrocycle planning
- AI-powered coaching via MCP integration ([gorilla_chatbot](https://github.com/elmomk/gorilla_chatbot))
- Garmin Connect data sync via [gapi](gapi.md)
- Mobile-first PWA

## Screenshots

<div class="screenshot-grid">
  <figure>
    <img src="/assets/screenshots/gorilla-coach/dashboard.png" alt="Dashboard" width="250">
    <figcaption>Dashboard — Garmin biometrics</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/gorilla-coach/training.png" alt="Training" width="250">
    <figcaption>Training — log sets</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/gorilla-coach/chat.png" alt="Chat" width="250">
    <figcaption>AI coaching chat</figcaption>
  </figure>
</div>

<div class="project-banner gorilla" style="margin-top: 3rem;">

<a href="https://github.com/elmomk/rusty_gorilla_coach" class="md-button">View on GitHub</a>

</div>
