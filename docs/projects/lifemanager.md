# Life Manager

A cyberpunk-themed mobile-first PWA for managing daily life.

## Modules

- **To-Dos** — Task tracking with quick-add chips and optional due dates
- **Groceries** — Shopping list with swipe-to-complete and dynamic defaults
- **Shopee Pick-ups** — Package tracking with OCR-based code extraction from screenshots (Traditional Chinese + English)
- **Watchlist** — Movie / Series / Anime / Cartoon tracker
- **Cycle Tracker** — Period logging with symptom tracking, next-cycle prediction, and PMS care reminders

## Key Capabilities

- Swipe right to complete, swipe left to delete
- OCR reads Shopee screenshots — extracts product name, store location, and pickup code
- Tracks who completed each item via Tailscale identity
- Installable PWA with offline caching

## Stack

| Layer | Technology |
|-------|-----------|
| Language | Rust (2021 edition) |
| Frontend | Dioxus 0.7 (fullstack, WebAssembly) |
| Styling | Tailwind CSS v4 with custom cyberpunk theme |
| Database | SQLite with r2d2 connection pooling |
| OCR | Tesseract (chi_tra + eng) |
| Auth | Tailscale user headers |

## Screenshots

<div class="screenshot-grid">
  <figure>
    <img src="/assets/screenshots/lifemanager/todos.png" alt="To-Dos" width="250">
    <figcaption>To-Dos</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/lifemanager/groceries.png" alt="Groceries" width="250">
    <figcaption>Groceries</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/lifemanager/shopee.png" alt="Shopee" width="250">
    <figcaption>Shopee Pick-ups</figcaption>
  </figure>
  <figure>
    <img src="/assets/screenshots/lifemanager/watchlist.png" alt="Watchlist" width="250">
    <figcaption>Watchlist</figcaption>
  </figure>
</div>

<div class="project-banner cyber" style="margin-top: 3rem;">

<a href="https://github.com/elmomk/lifemanager" class="md-button">View on GitHub</a>

</div>
