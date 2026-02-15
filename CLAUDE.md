# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hugo static site for "Psammites Lab" blog. No external dependencies — pure Hugo project with no package manager.

## Commands

- `hugo server` — local dev server at http://localhost:1313
- `hugo` — build static site to `public/`
- `hugo new content/<section>/<slug>.md` — create new content using archetypes

No test, lint, or CI setup exists.

## Architecture

**Styling**: Single CSS file at `assets/css/main.css` (no preprocessor).

**Config**: `hugo.toml` for Hugo settings
