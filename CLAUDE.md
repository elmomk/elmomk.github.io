# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Personal portfolio and documentation site for Mo (elmomk). Built with **Zensical** (a standalone static site generator from the creators of Material for MkDocs). Hosted on GitHub Pages at https://elmomk.github.io.

## Build & Preview

```bash
pipx run zensical build --clean   # build to site/
pipx run zensical serve           # local preview at localhost:8000
```

Or use the `/build-site` skill command.

Deployment is automatic — push to `main` triggers `.github/workflows/deploy.yml` which runs `zensical build --clean` and deploys the `site/` directory to GitHub Pages.

## Site Structure

- `mkdocs.yml` — main config: nav tree, theme, markdown extensions
- `docs/` — all content
  - `index.md` — homepage (custom HTML, not standard markdown)
  - `about.md` — about page
  - `projects/` — project intro pages
  - `tutorials/` — deep-dive tutorials organized by project
  - `blog/index.md` — blog listing (manually maintained)
  - `blog/posts/` — blog posts (plain markdown, no blog plugin)
  - `stylesheets/extra.css` — custom CSS (screenshot grids, project accents)
  - `assets/screenshots/` — project screenshots referenced from pages

## Content Conventions

- **Diagrams**: Use Mermaid fenced blocks (` ```mermaid `), never ASCII art
- **Markdown extensions**: admonitions (`!!! tip`), tabbed content (`=== "Tab"`), grid cards (`<div class="grid cards">`), Material icons (`:material-icon-name:`), emoji (`:fontawesome-brands-rust:`)
- **Blog posts**: Add new posts to `blog/posts/`, then manually update `blog/index.md` and add to `nav:` in `mkdocs.yml`
- **Screenshots**: use `{ width="300" }` attribute for consistent sizing

## Key Config Details

- Theme: Material with `modern` variant, teal accent, dark/light toggle
- Mermaid support enabled via `pymdownx.superfences` custom fence
- Zensical does NOT support mkdocs-material plugins (blog, tags, etc.) — blog is manual
- Nav is manually maintained in `mkdocs.yml` — new pages must be added there

## Cross-References

Project pages describe apps that live in separate repositories. When updating descriptions, verify accuracy against the actual source repos:

| Project | Repo path | Frontend | Key numbers |
|---------|-----------|----------|-------------|
| gorilla_coach | `~/git/gorilla_coach` | Dioxus 0.7 | — |
| gorilla_mcp | `~/git/gorilla_mcp` | — | 17 tools, 3 resources, 4 prompts |
| garmin_api (gapi) | `~/git/garmin_api` | Leptos 0.7 | 12+ sync endpoints |
