# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal knowledge documentation site built with MkDocs and Material for MkDocs, deployed to GitHub Pages.

## Build Commands

```bash
# Build documentation (creates isolated venv, builds, then cleans up)
./bin/build

# Build and deploy to GitHub Pages (build + git add/commit/push)
./bin/deploy

# Clean build artifacts (removes docs/, _build/, .venv/)
./bin/clean
```

**Prerequisites:** Python 3.x

## Automated Deployment

GitHub Actions automatically builds and deploys when:
- Files in `src/` are modified
- `requirements.txt` is changed
- Push to `main` branch

## Architecture

- **Source files:** `src/` (Markdown documentation)
- **Build output:** `docs/` (generated HTML for GitHub Pages)
- **Config:** `mkdocs.yml` (site configuration, navigation, theme)
- **Build tools:** `bin/` (Python scripts for build/deploy/clean)

### Adding New Documentation

1. Create `.md` file in `src/`
2. Add entry to `nav:` section in `mkdocs.yml`
3. Build and deploy

### Supported Markdown Features

- Admonitions (`!!! note "Title"` blocks)
- Fenced code blocks with syntax highlighting
- Mermaid diagrams
- Tables
- PyMdown extensions (details, superfences, highlight, snippets)
