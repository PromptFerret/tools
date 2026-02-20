# PromptFerret Tools

Open-source, single-file HTML web tools for AI workflows, D&D utilities, and chatbot integrations. No build steps, no frameworks, no external dependencies — every tool is a single `index.html` you can host anywhere.

**Portal:** [promptferret.github.io/tools](https://promptferret.github.io/tools/)

## Tools

| Tool | Description | Live Link |
|------|-------------|-----------|
| [SquishText](https://github.com/PromptFerret/SquishText) | Compress text into shareable encoded strings | [Launch](https://promptferret.github.io/SquishText/) |
| [Encounter Manager](https://github.com/PromptFerret/EncounterManager) | D&D encounter manager & initiative tracker | [Launch](https://promptferret.github.io/EncounterManager/) |
| [GSheet Character Exporter](https://github.com/PromptFerret/GSheetCharacterExporter) | Export Google Sheets D&D characters to Markdown/JSON | [Launch](https://promptferret.github.io/GSheetCharacterExporter/) |
| [MarkdownSite](https://github.com/PromptFerret/MarkdownSite) | Single-file markdown document browser with Obsidian support | [Launch](https://promptferret.github.io/MarkdownSite/) |

## Architecture

Each tool is a standalone GitHub repo under the [PromptFerret](https://github.com/PromptFerret) org, deployed via GitHub Pages at `promptferret.github.io/<RepoName>/`.

This repo (`tools`) serves as:
- **Portal page** — [promptferret.github.io/tools](https://promptferret.github.io/tools/) with cards linking to each tool
- **Redirect stubs** — the tools used to live as subdirectories here; old URLs (like `/tools/squishtext/`) redirect to the new standalone locations

## Shared Design

All tools follow the same conventions:

- **Single HTML file** with embedded CSS and JavaScript
- **Dark theme** (`--bg: #0f0f17`, `--surface: #1a1a2e`, `--accent: #7c5cbf`)
- **Responsive** — stacks at 640px breakpoint
- **No external dependencies** — vanilla browser APIs only
- **No build step** — edit and refresh

## Running Locally

Each tool is a single `index.html`. Clone any repo and open the file:

```bash
git clone https://github.com/PromptFerret/SquishText.git
open SquishText/index.html
```

Some tools require an HTTP server (GSheet exporter for CORS, MarkdownSite for `fetch()`):

```bash
python3 -m http.server 8000
```

## Contributing

Each tool has its own repo with a `CLAUDE.md` containing detailed architecture documentation. Read it before making changes.

1. Fork the tool's repo
2. Edit `index.html`
3. Test in a browser
4. Submit a pull request

## License

All tools are licensed under the [MIT License](LICENSE).

---

Created and maintained by **PromptFerret**
https://promptferret.github.io

Vibe coded with [Claude Code](https://docs.anthropic.com/en/docs/claude-code).
