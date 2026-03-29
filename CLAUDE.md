# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hugo static site for "Psammites Lab" blog. No external dependencies — pure Hugo project with no package manager.

## Commands

- `hugo server` — local dev server at http://localhost:1313
- `hugo` — build static site to `public/`
- `hugo new content/<section>/<slug>` — create new content as a page bundle using archetypes

No test or lint setup exists.

## CI/CD

GitHub Actions workflow at `.github/workflows/hugo.yml`:
- **Trigger**: push to `main` or manual `workflow_dispatch`
- **Build**: Hugo extended v0.155.1 with Go 1.25.6 on Ubuntu
- **Deploy**: GitHub Pages via `actions/deploy-pages@v4`
- **Caching**: Hugo cache saved/restored between runs
- **Concurrency**: `pages` group, does not cancel in-progress builds

## Architecture

**Styling**: Single CSS file at `assets/css/main.css` (no preprocessor).

**Config**: `hugo.toml` for Hugo settings
