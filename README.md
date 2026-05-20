---
Author: Tomás Vírseda
Category: Note
Date: 2026-05-20
DocType: Explanation
Tag: techdoc, kb4it, notes, github-pages
---

# t00m's tech notes

Live demo of [KB4IT](https://github.com/t00m/KB4IT) using the **techdoc** theme: <https://t00m.github.io/techdoc/index.html>

## Configuration

- `config/repo.json` -- incremental build (`"force": false`, `"sort": "Date"`, `"title": "t00m's tech notes"`).
- `config/repo_force_mode.json` -- full rebuild (`"force": true`, `"sort": "Updated"`, `"title": "KB4IT"`).

Both use `"source": "source"` and `"target": "docs"`.

## CI/CD

Two GitHub Actions workflows handle the build:

- **Incremental** -- triggered on `source/**`, uses `config/repo.json`.
- **Force** -- triggered on `config/**`, uses `config/repo_force_mode.json`.

Workflows clone KB4IT, run `kb4it build`, and commit generated output to `docs/` and `var/`.

## GitHub Pages setup

To enable the published site, configure GitHub Pages at
`https://github.com/t00m/techdoc/settings/pages`:

- **Build and deployment**
  - **Source:** Deploy from a branch
  - **Branch:** `main`
  - **Directory:** `/docs`
