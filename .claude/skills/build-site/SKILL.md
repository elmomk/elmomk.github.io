---
name: build-site
description: Build the Zensical static site locally and check for errors.
argument-hint: [--serve]
---

Build the elmomk.github.io site using Zensical.

## Steps

1. Run `pipx run zensical build --clean` from the repo root
2. Report any warnings or errors
3. If `--serve` argument is passed, run `pipx run zensical serve` instead (for live preview at localhost:8000)

## Output format

- **PASS**: "Site built successfully — N pages in Xs"
- **FAIL**: List each error with context
