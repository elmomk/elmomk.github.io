# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Personal portfolio and documentation site for Mo (elmomk). Built with **Zensical** (a Material for MkDocs distribution). Hosted on GitHub Pages at https://elmomk.github.io.

## Build & Preview

```bash
pip install zensical          # one-time setup
zensical build --clean        # build to site/
zensical serve                # local preview at localhost:8000
```

Deployment is automatic — push to `main` triggers `.github/workflows/deploy.yml` which runs `zensical build --clean` and deploys the `site/` directory to GitHub Pages.

## Site Structure

- `mkdocs.yml` — main config: nav tree, theme, plugins, markdown extensions
- `docs/` — all content
  - `index.md` — homepage (custom HTML, not standard markdown)
  - `about.md` — about page
  - `projects/` — project intro pages (gorilla-coach, gapi, gorilla-mcp, cc-watcher, tmux-cc-attention, lifemanager, retrogames)
  - `tutorials/` — deep-dive tutorials organized by project
  - `blog/posts/` — blog posts (mkdocs-material blog plugin)
  - `blog/.authors.yml` — blog author definitions
  - `stylesheets/extra.css` — custom CSS (screenshot grids, project accents)
  - `assets/screenshots/` — project screenshots referenced from pages
- `overrides/` — theme template overrides (currently empty)

## Content Conventions

- **Diagrams**: Use Mermaid fenced blocks (` ```mermaid `), never ASCII art. All ASCII diagrams have been converted.
- **Markdown extensions**: admonitions (`!!! tip`), tabbed content (`=== "Tab"`), grid cards (`<div class="grid cards">`), Material icons (`:material-icon-name:`), emoji (`:fontawesome-brands-rust:`)
- **Blog posts**: frontmatter requires `date`, `authors`, `categories`. Use `<!-- more -->` for the excerpt separator.
- **Screenshots**: use `{ width="300" }` attribute for consistent sizing

## Key Config Details

- Theme: Material with `modern` variant, teal accent, dark/light toggle
- Mermaid support enabled via `pymdownx.superfences` custom fence
- Blog plugin: `blog_dir: blog`, `authors: true`
- Nav is manually maintained in `mkdocs.yml` — new pages must be added there

## Cross-References

Project pages describe apps that live in separate repositories. When updating descriptions, verify accuracy against the actual source repos:

| Project | Repo path | Frontend | Key numbers |
|---------|-----------|----------|-------------|
| gorilla_coach | `~/git/gorilla_coach` | Dioxus 0.7 | — |
| gorilla_mcp | `~/git/gorilla_mcp` | — | 17 tools, 3 resources, 4 prompts |
| garmin_api (gapi) | `~/git/garmin_api` | Leptos 0.7 | 12+ sync endpoints |
