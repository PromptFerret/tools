# SquishText

## What This Is
A single self-contained HTML file (`index.html`) that compresses text into a shareable encoded string, and decompresses it back. No server, no accounts, no dependencies — the compressed data IS the payload.

## How It Works

### Compression Pipeline
1. **Input**: raw text (any length)
2. **Compress**: deflate (raw, not gzip) via the browser `CompressionStream` API
3. **Encode**: base64-encode the compressed bytes into pasteable ASCII
4. **Output**: a single string that can be copied and shared

### Decompression Pipeline
1. **Input**: base64-encoded compressed string
2. **Decode**: base64 → raw bytes
3. **Decompress**: inflate via `DecompressionStream` API
4. **Output**: original text, byte-for-byte identical

### Why This Approach
- **DeflateRaw** (not gzip): smaller output — no gzip header/footer overhead
- **Base64**: universally pasteable — works in Discord, chat, email, text files, anywhere that accepts plain text
- **CompressionStream API**: built into all modern browsers (Chrome 80+, Firefox 113+, Safari 16.4+), zero JS library dependencies
- **Trade-off**: base64 adds ~33% overhead, so net compression on typical English text is ~35-50% smaller. Highly repetitive text (tables, lists, structured data) compresses much better.

## Use Cases
- Share a full D&D character sheet as a single pasteable blob
- Compress any structured text (session notes, item lists, build guides) for sharing
- Pack markdown output from the GSheet exporter into a compact shareable format
- Encode text so it's not casually readable (light obfuscation, not encryption)

## UI Design

### Layout
Single-page tool matching the PromptFerret tools dark theme (same CSS variables as GSheet exporter).

### Two Modes
- **Pack** (default): large text area for input, "Squish" button, output area with copy button
- **Unpack**: large text area for encoded input, "Unsquish" button, output area with copy button

Toggle between modes with toolbar buttons (same pattern as Preview/MD/JSON toggle in exporter).

### Stats Bar
After compression/decompression, show:
- Original size (characters / bytes)
- Compressed size (characters)
- Compression ratio (e.g., "62% smaller" or "1.4x larger")
- Visual indicator: green if smaller, yellow if similar, red if larger

### Error Handling
- Invalid base64 in unpack mode: show clear error message
- Decompression failure: show error (likely not a SquishText payload)
- Empty input: disable the action button

## Technical Notes

### CompressionStream API Usage
```js
async function compress(text) {
  const encoder = new TextEncoder();
  const stream = new Blob([encoder.encode(text)])
    .stream()
    .pipeThrough(new CompressionStream('deflate-raw'));
  const blob = await new Response(stream).blob();
  const buffer = await blob.arrayBuffer();
  return btoa(String.fromCharCode(...new Uint8Array(buffer)));
}

async function decompress(base64) {
  const bytes = Uint8Array.from(atob(base64), c => c.charCodeAt(0));
  const stream = new Blob([bytes])
    .stream()
    .pipeThrough(new DecompressionStream('deflate-raw'));
  const blob = await new Response(stream).blob();
  return await blob.text();
}
```

### Browser Compatibility
- `CompressionStream` / `DecompressionStream` require modern browsers
- No polyfill — if the API isn't available, show a message suggesting a modern browser
- This matches the GSheet exporter's approach (also requires modern browser for `fetch`)

### Payload Format
The output is plain base64 with no prefix or wrapper. This keeps it minimal and maximizes compatibility. The decompressor should detect and handle:
- Trailing whitespace / newlines (strip before decoding)
- URL-safe base64 variant (replace `-` with `+`, `_` with `/` if encountered)

## Development Philosophy

### Keep It Simple
- Single HTML file, no build step, no external dependencies
- Same dark theme as the rest of PromptFerret tools
- Must work when served via HTTP (GitHub Pages) — same CORS-friendly approach
- File:// should also work for this tool (no fetch calls needed)

### Prefer Local Execution Over Agent Processing
When testing or debugging, write and run a local script rather than doing it manually in conversation. The user's machine is Windows — use Python or Node, not bash.
