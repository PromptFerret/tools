# Resource Viewer

## What This Is

A single self-contained HTML file (`index.html`) that renders markdown documents loaded via URL hash. Drop it alongside a folder of `.md` files (e.g., an Obsidian vault export) and browse them with working cross-document links.

## How It Works

### Hash Routing

```
resource_viewer/#subfolder/my-doc
                 └─ hash ──────┘
Fetches: subfolder/my-doc.md (relative to resource_viewer/)
```

- `window.location.hash` → strip `#` → append `.md` → `fetch()`
- Empty hash or no hash → loads `WELCOME.md`
- `hashchange` event drives navigation without page reload
- Current document's folder path tracked as "base path" for resolving relative links/images

### Document Loading Pipeline

1. Extract path from hash (or default to `WELCOME`)
2. Compute base path (e.g., `notes/combat` → base = `notes/`)
3. `fetch(path + '.md')` with loading status
4. On success: parse markdown → render HTML → inject into `#docContent` → enhance copy buttons → scroll to top (or to anchor if specified)
5. On 404: show "Document not found" error in content area
6. On network error: show error status
7. Raw markdown stored in `rawMarkdown` for source view toggle

### In-Page Anchors vs Hash Routing

The URL hash is used for document paths, so in-page anchors use a different mechanism:

- All headings get auto-generated `id` attributes (e.g., `## Combat Rules` → `id="combat-rules"`)
- In-page anchor links (`[link](#combat-rules)`) are intercepted via click handler → `scrollIntoView()` without changing the hash
- Cross-doc anchor links (`[link](other.md#section)`) → navigate to `#other`, then after render, scroll to anchor
- Cross-doc anchors use `::` separator in hash: `#doc::anchor`

### Link Rewriting

All link rewriting happens at render time in `inlineFormat()`, using the current document's base path.

**Standard markdown links** `[text](href)`:
| `href` pattern | Rewrite |
|---|---|
| Ends with `.md` | Strip `.md`, resolve relative to base path → `#resolved-path` |
| Ends with `.md#anchor` | Same + post-load scroll to anchor |
| Starts with `http://` or `https://` | Leave as-is, add `target="_blank"` |
| Starts with `#` | In-page anchor → `scrollIntoView()` via click handler |
| Other relative | Resolve relative to base path |

**Obsidian wikilinks** `[[target]]` and `[[target|display]]`:
| Syntax | Result |
|---|---|
| `[[doc]]` | Link text = `doc`, href = `#doc` (resolved relative to base) |
| `[[folder/doc]]` | Link text = `doc`, href = `#folder/doc` |
| `[[doc\|Display]]` | Link text = `Display`, href = `#doc` |
| `[[doc#heading]]` | Link to `#doc` + scroll to heading |

**Paths must be explicit** — no vault-wide filename search. Bare `[[filename]]` resolves relative to the current document's folder only.

### Image Resolution

`![alt](src)`:
- Relative `src` → prepend current document's base path
- Absolute or external `src` (starts with `/`, `http://`, `https://`, `data:`) → leave as-is
- Images styled with `max-width: 100%`

### Path Resolution

`resolvePath(href, basePath)` handles `../` segments with a do/while loop that stops when no replacements occur. Unresolvable leading `../` (already at root) are stripped. `./` segments are removed.

## Markdown Parser

Hand-rolled line-by-line parser. Two main functions:

### `mdToHTML(md, basePath)` — Block-Level Processing

State machine iterating lines, tracking open block contexts:

| State | Trigger | Behavior |
|---|---|---|
| Fenced code | Opening `` ``` `` (3+ backticks) | Accumulate lines verbatim until closing fence of equal or greater length, emit `<pre><code>` |
| `inBlockquote` | `> ` prefix or bare `>` | Accumulate lines, flush as callout or `<blockquote>` |
| `inUl` | `- ` or `* ` prefix | Emit `<ul><li>` items |
| `inOl` | `N. ` prefix | Emit `<ol><li>` items |
| `inTable` | `\|...\|` lines | Header detection via separator row, emit `<table>` |
| (none) | Blank line | Close any open block |
| (none) | `# ` through `###### ` | Emit heading with auto `id` |
| (none) | `---` / `***` / `___` | Emit `<hr>` |
| (none) | Anything else | Emit `<p>` |

**Variable-length fenced code blocks**: Opening fence length is tracked. Only a closing fence with equal or greater backtick count closes the block. This allows 4-backtick fences to wrap 3-backtick content (used in PLAYGROUND.md).

### `inlineFormat(text, basePath)` — Inline Processing

Regex-based, applied to all non-code content. **Order matters**:

1. HTML escape (`&`, `<`, `>`, `"`)
2. Inline code (`` `text` ``) — extracted to placeholders to prevent inner formatting
3. Images (`![alt](src)`) — resolve src relative to basePath
4. Links (`[text](href)`) — rewrite per strategy above
5. Wikilinks (`[[target]]`, `[[target|display]]`) — rewrite per strategy above
6. Bold+italic (`***text***`)
7. Bold (`**text**`)
8. Italic (`*text*`)
9. Strikethrough (`~~text~~`)
10. Restore inline code placeholders

### `flushBlockquote()` — Callout Detection

When a blockquote is flushed, the first line is checked for callout syntax `[!type]` or `[!type]-`. If matched:
- Type is resolved via `CALLOUT_TYPE_MAP` (handles aliases)
- Icon looked up from `CALLOUT_ICONS` (emoji-based, no FontAwesome)
- `-` modifier creates a collapsible callout (starts collapsed, click header to toggle)
- Title is optional — if omitted, only the icon shows in the header
- **Content is rendered via recursive `mdToHTML()`** — supports full block elements (lists, tables, code blocks, headings, HRs) inside callouts

If no callout syntax, renders as a plain `<blockquote>`.

## Callout Types

14 distinct types with aliases:

| Type | Aliases | Icon |
|---|---|---|
| note | — | pencil |
| abstract | tldr | page |
| summary | — | book |
| info | todo | info |
| tip | hint, important | fire |
| success | check, done | check |
| question | help, faq | question |
| warning | caution, attention | warning |
| failure | fail, missing | cross |
| danger | error | lightning |
| bug | — | bug |
| example | — | clipboard |
| quote | cite | quote |
| statblock | — | dice |

### Statblock Callout

Special D&D parchment styling: `#fdf1dc` background, Georgia serif font, dark red accents (`#a52a2a`), box shadow, themed `h3`/`h4` headings with `#c88` borders, parchment-colored horizontal rules. Designed for monster stat blocks with `### Traits`, `### Actions`, etc.

## Toolbar

```
[Rendered] [Source] | [Thin] [Normal] [Wide]  ──────  [Copy Link]
```

- **Rendered / Source** — toggle between rendered markdown and raw source (`<pre><code>`)
- **Thin / Normal / Wide** — document max-width: 600px / 800px / 1200px
- **Copy Link** — copies current URL (with hash) to clipboard
- State persists across document navigations (view mode, width)
- Button styles scoped to `.toolbar button` to avoid bleeding into copy buttons

## Copy Buttons

Post-render DOM enhancement via `enhanceCopyButtons()`:

- **Code blocks** (`<pre>`) — clipboard icon, top-right, visible on hover (opacity 0.15 → 0.7)
- **Images** (`<img>`) — wrapped in `.image-container`, copies image URL, dark backdrop button
- **Inline code** (`<code>` inside p/li/td/th/blockquote/callout) — tiny button, appears on hover

All use SVG clipboard icon → checkmark on success (1.5s timeout).

## Customization Files

On init, the viewer fetches optional customization files in parallel. All are optional — missing files (404) are silently ignored.

| File | Purpose | Loaded by |
|---|---|---|
| `HEADER.md` | Rendered above toolbar as page header | `loadChrome()` |
| `FOOTER.md` | Rendered below content as page footer | `loadChrome()` |
| `STYLE.css` | CSS overrides injected after built-in styles | `loadStyle()` |
| `{name}.css` | Named theme, loaded via `?style={name}` param | `loadStyle()` |

### Chrome (HEADER.md / FOOTER.md)

- Rendered via `mdToHTML()` into `.header` / `.footer` divs
- Both containers start with `display: none`, set to `display: 'block'` on successful load
- Content is rendered once at startup, not re-fetched on navigation
- Base path is `''` (root-relative) since chrome files sit alongside `index.html`
- Skipped entirely when `?chrome=false` is in the URL

### Style Override (STYLE.css / ?style=NAME)

- `loadStyle()` runs independently of `loadChrome()` — always loads, even with `?chrome=false`
- Default: fetches `STYLE.css`
- With `?style=NORD`: fetches `NORD.css` instead
- Fetched CSS is injected as a `<style>` tag appended to `<head>`, after the built-in styles
- Later source order means it only overrides what it explicitly declares — all other styles remain
- Minimal theme override only needs `:root` variable redefinitions

### Query Parameters

| Param | Effect | Persists? |
|---|---|---|
| `?chrome=false` | Skips HEADER.md / FOOTER.md loading | Yes (hash navigation preserves query) |
| `?style=NAME` | Loads `NAME.css` instead of `STYLE.css` | Yes |

Both params can be combined: `?chrome=false&style=NORD#PLAYGROUND`

## UI Structure

```
┌─────────────────────────────────────┐
│  header (from HEADER.md, optional)  │
├─────────────────────────────────────┤
│  toolbar: view / width / copy link  │
├─────────────────────────────────────┤
│  status bar (loading/error/success) │
├─────────────────────────────────────┤
│                                     │
│  .content (flex:1, scrollable)      │
│    .document (max-width, centered)  │
│      rendered markdown              │
│                                     │
├─────────────────────────────────────┤
│  footer (from FOOTER.md, optional)  │
└─────────────────────────────────────┘
```

## CSS Theme

Same shared PromptFerret dark theme (`--bg`, `--surface`, `--accent`, etc.).

Additional document-specific styles:
- `.document` — max-width 800px (adjustable via toolbar), centered, readable line-height
- Code blocks — darker background (`#0a0a12`), monospace, scrollable
- Inline code — surface2 background, rounded
- Blockquotes — accent left border, muted text
- Links — accent color, underline on hover
- Images — max-width 100%, rounded corners
- Callouts — 14 color schemes, collapsible via CSS `.collapsed` class
- Statblock — parchment theme overriding dark theme colors
- Copy buttons — absolute positioned, opacity transitions, SVG icons

## Key Decisions

- **No external dependencies** — parser is hand-rolled, not a library
- **No file index** — wikilinks require explicit paths, no vault search
- **No sidebar/navigation** — purely URL-driven
- **No build step** — single HTML file, edit and refresh
- **No embedded HTML** — all content is escaped, HTML tags render as text
- **Requires HTTP** — uses fetch(), will not work from file://
- **Hash routing** — document path in fragment, never hits server
- **Recursive callout content** — callout bodies go through full `mdToHTML`, enabling block elements inside callouts
- **Toolbar scoped styles** — button CSS uses `.toolbar button` selector to avoid bleeding into copy buttons
- **Markdown-driven chrome** — header and footer loaded from `HEADER.md` / `FOOTER.md`, not hardcoded; delete either file for a chrome-free viewer
- **CSS theme override** — `STYLE.css` or `?style=NAME` injects custom CSS after built-in styles; only overrides what it declares
- **Query param persistence** — `?chrome=false` and `?style=NAME` survive hash navigation since query precedes fragment in URLs
