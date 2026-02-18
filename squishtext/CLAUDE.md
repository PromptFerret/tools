# SquishText

## What This Is
A single self-contained HTML file (`index.html`) that compresses text into a shareable encoded string, and decompresses it back. No server, no accounts, no dependencies — the compressed data IS the payload.

## How It Works

### Compression Pipeline
1. **Input**: raw text (any length)
2. **Compress**: deflate (raw, not gzip) via the browser `CompressionStream` API
3. **Encode**: base64-encode the compressed bytes into pasteable ASCII
4. **Checksum**: append CRC32 checksum (`.` + 8 hex chars) to detect corruption
5. **Header**: prepend a human/LLM-readable header with decode instructions and tool URL
6. **Output**: a single string that can be copied and shared

### Decompression Pipeline
1. **Input**: encoded string (with or without header)
2. **Strip header**: remove `[SquishText]...` line if present
3. **Verify checksum**: if `.8hexchars` suffix present, CRC32 verify before decoding
4. **Decode**: base64 → raw bytes (handles URL-safe variant automatically)
5. **Decompress**: inflate via `DecompressionStream` API
6. **Output**: original text, byte-for-byte identical

### Why This Approach
- **DeflateRaw** (not gzip): smaller output — no gzip header/footer overhead
- **Base64**: universally pasteable — works in Discord, chat, email, text files, anywhere that accepts plain text
- **CompressionStream API**: built into all modern browsers (Chrome 80+, Firefox 113+, Safari 16.4+), zero JS library dependencies
- **CRC32 checksum**: detects single-character corruption — deflate has zero error tolerance, so a clear error message beats a cryptic decompression failure
- **Trade-off**: base64 adds ~33% overhead, so net compression on typical English text is ~35-50% smaller. Highly repetitive text (tables, lists, structured data) compresses much better.

## Use Cases
- Share a full D&D character sheet as a single pasteable blob
- Compress any structured text (session notes, item lists, build guides) for sharing
- Pack markdown output from the GSheet exporter into a compact shareable format
- Encode text so it's not casually readable (light obfuscation, not encryption)
- Share via link — payload embedded in URL hash, auto-decodes on open

## Payload Format

### Text Blob (Copy button output)
```
[SquishText] To decode, base64-decode the string below then inflate (deflate-raw). Or paste at https://promptferret.github.io/tools/squishtext/
rVdtb9y4Ef6uX...abcd1234
```
- Header line: human-readable + LLM decode instructions (an LLM with code execution can decode it on sight)
- Body: standard base64 + `.` + 8-char CRC32 hex

### Share Link (Link button output)
```
https://promptferret.github.io/tools/squishtext/#rVdtb9y4Ef6uX...abcd1234
```
- URL-safe base64 in the hash fragment: `+` → `-`, `/` → `_`, trailing `=` stripped
- CRC32 checksum included after `.` separator
- Hash fragment never hits the server — purely client-side
- Page auto-unsquishes on load if hash is present

### Backward Compatibility
- Payloads without a checksum suffix decompress normally (no verification)
- Header line is stripped automatically — paste with or without it
- URL-safe and standard base64 both accepted (auto-converted)

## UI Design

### Layout
Two-pane layout (input left, output right) matching the PromptFerret tools dark theme. Panes stack vertically on mobile. Both panes scroll independently.

### Toolbar
- **Squish**: compress input text → encoded output with header + checksum
- **Unsquish**: decode input blob → original text
- **Copy**: copy output to clipboard
- **Link**: copy shareable URL with payload in hash (enabled after squishing)
- **Clear**: reset everything

No mode toggle — both action buttons are always visible. Paste text and pick which one you want.

### Pane Headers
Each pane has a header label and a Copy button (appears when content is available).

### Stats Bar
After compression/decompression, show:
- Original size (characters / bytes)
- Encoded size (characters)
- Compression ratio (e.g., "59% smaller" or "12% larger")
- Visual indicator: green if smaller, yellow if similar, red if larger

### Error Handling
- Invalid base64: "Invalid input — not a valid base64 string"
- CRC32 mismatch: "Checksum failed — this payload has been corrupted or modified"
- Decompression failure: "Decompression failed — this doesn't appear to be a SquishText payload"
- Empty input: action buttons disabled
- No CompressionStream: show browser upgrade message

### Auto-Load from URL Hash
On page load, if `window.location.hash` contains data, auto-unsquish and display. Clear button resets the hash via `history.replaceState`.

## Technical Notes

### CRC32 Implementation
Standard lookup-table CRC32 computed over the base64 string (before URL-safe conversion). Output is 8 lowercase hex characters. Cost: 9 extra characters per payload (`.` + 8 hex).

### Browser Compatibility
- `CompressionStream` / `DecompressionStream` require modern browsers (Chrome 80+, Firefox 113+, Safari 16.4+)
- No polyfill — if the API isn't available, show a message suggesting a modern browser
- Works from `file://` — no fetch calls needed

### URL Length Limits
| Browser | Practical URL length |
|---------|---------------------|
| Chrome | ~2MB total |
| Firefox | ~65,000 chars |
| Safari | ~80,000 chars |

Typical character sheets compress to 2,000–6,000 chars — well within all limits.

## Development Philosophy

### Keep It Simple
- Single HTML file, no build step, no external dependencies
- Same dark theme as the rest of PromptFerret tools
- Must work when served via HTTP (GitHub Pages) — same CORS-friendly approach
- File:// should also work for this tool (no fetch calls needed)

### Prefer Local Execution Over Agent Processing
When testing or debugging, write and run a local script rather than doing it manually in conversation. The user's machine is Windows — use Python or Node, not bash.
