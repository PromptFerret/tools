# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**PromptFerret Tools** — open-source, single-file HTML web tools for AI workflows, D&D utilities, and chatbot integrations. Deployed via GitHub Pages at https://promptferret.github.io/tools/.

## Architecture

Every tool is a **single self-contained HTML file** with embedded CSS and JavaScript. No build step, no bundler, no external dependencies, no frameworks. Each tool lives in its own directory with an `index.html` and a tool-specific `CLAUDE.md`.

```
tools/
├── index.html                    # Portal page linking to all tools
├── squishtext/index.html         # Text compression/sharing tool
├── squishtext/CLAUDE.md
├── gsheet_md_exporter/index.html # D&D character sheet exporter
└── gsheet_md_exporter/CLAUDE.md
```

The portal (`index.html` at root) is a static landing page with cards linking to each tool.

## Development Workflow

There is no build, lint, or test command. Tools are vanilla HTML/CSS/JS served as-is. Development is edit-and-refresh.

- **Local testing**: Open `index.html` in a browser. SquishText works from `file://`; GSheet exporter requires HTTP origin (CORS).
- **Deployment**: Push to `main` — GitHub Pages auto-deploys.

## Shared Conventions

### CSS Theme (all tools must match)
```css
--bg: #0f0f17;
--surface: #1a1a2e;
--accent: #7c5cbf;
--text: #e8e6f0;
--mono: 'Cascadia Code', 'Fira Code', 'Consolas', monospace;
```
Dark theme, responsive (stacks at 640px), uses `100dvh` with `@supports` fallback.

### UI Pattern
Every tool follows: Header (h1 + tagline) → Toolbar (action buttons) → Status bar → Content area → Footer (links to GitHub, license, PromptFerret).

### Status messaging
`setStatus(msg, type)` with color coding: green (success), yellow (warning), red (error).

## Adding a New Tool

1. Create `toolname/index.html` — single self-contained file, same dark theme
2. Create `toolname/CLAUDE.md` — document architecture, position maps, key decisions
3. Update root `index.html` — add a card linking to the new tool
4. Uses only native browser APIs — no npm, no CDN scripts

## Development Philosophy

- Single HTML file per tool, no external dependencies
- Prefer local script execution over agent processing when testing APIs or parsing data
- Use direct approaches — don't spawn expensive subagents for debugging
- ASK the user questions instead of burning tokens guessing
- Keep tools working offline where possible (no fetch = works from `file://`)

## Tool-Specific Documentation

Each tool has its own `CLAUDE.md` with detailed architecture docs (position maps, data pipelines, format specs). **Always read the tool-specific CLAUDE.md before modifying a tool.**
