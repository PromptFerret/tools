# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**PromptFerret Tools** — open-source, single-file HTML web tools for AI workflows, D&D utilities, and chatbot integrations. Deployed via GitHub Pages at https://promptferret.github.io/tools/.

## Architecture

Every tool is a **single self-contained HTML file** with embedded CSS and JavaScript. No build step, no bundler, no external dependencies, no frameworks. Each tool lives in its own GitHub repo under the PromptFerret org, with its own `index.html` and `CLAUDE.md`.

```
PromptFerret/
├── tools/                          # This repo — portal + redirect stubs
│   ├── index.html                  # Portal page linking to all tools
│   ├── squishtext/index.html       # Redirect → SquishText repo
│   ├── gsheet_md_exporter/index.html # Redirect → GSheetCharacterExporter repo
│   ├── encounter/index.html        # Redirect → EncounterManager repo
│   └── resource_viewer/index.html  # Redirect → MarkdownSite repo
├── SquishText/                     # Text compression/sharing tool
├── EncounterManager/               # D&D encounter manager & initiative tracker
├── GSheetCharacterExporter/        # D&D character sheet exporter
└── MarkdownSite/                   # Markdown document browser
```

This repo (`tools/`) is a portal landing page with redirect stubs for the old sub-directory URLs. Each tool is in its own repo, deployed via GitHub Pages at `promptferret.github.io/<RepoName>/`.

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

1. Create a new repo under the PromptFerret org — single self-contained `index.html`, same dark theme
2. Create `CLAUDE.md` in the repo — document architecture, position maps, key decisions
3. Update this repo's `index.html` — add a card linking to the new tool
4. Add a redirect stub in this repo for any legacy URL
5. Uses only native browser APIs — no npm, no CDN scripts

## Development Philosophy

- Single HTML file per tool, no external dependencies
- Prefer local script execution over agent processing when testing APIs or parsing data
- Use direct approaches — don't spawn expensive subagents for debugging
- ASK the user questions instead of burning tokens guessing
- Keep tools working offline where possible (no fetch = works from `file://`)

## Tool-Specific Documentation

Each tool repo has its own `CLAUDE.md` with detailed architecture docs (position maps, data pipelines, format specs). **Always read the tool-specific CLAUDE.md before modifying a tool.**

Tool repos:
- [SquishText](https://github.com/PromptFerret/SquishText) — text compression/sharing
- [EncounterManager](https://github.com/PromptFerret/EncounterManager) — D&D encounter manager & initiative tracker
- [GSheetCharacterExporter](https://github.com/PromptFerret/GSheetCharacterExporter) — D&D character sheet exporter
- [MarkdownSite](https://github.com/PromptFerret/MarkdownSite) — markdown document browser
