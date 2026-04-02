# Posted Files Email Generator

## Architecture
- Single-file browser tool (`index.html`) — no build step, no dependencies, no backend
- Two rendering contexts: (1) generator UI (modern CSS) and (2) email output (Outlook-compatible)
- All state is ephemeral (browser session only)

## Outlook Email HTML Constraints (Critical)
- Table-based layout only — no flexbox, grid, or `<div>` layout
- All styles inline on every element — no `<style>` blocks, no CSS classes
- Font stack: `Arial, sans-serif` on every text element
- Spacing via `<tr><td height="N">&nbsp;</td></tr>` — not margin/padding
- Accent bars via `<td bgcolor="...">` — not CSS borders
- Links: `style="color: #1a73e8; text-decoration: underline;"`
- XHTML Transitional doctype required

## Brand Colors
- Definian Blue: `#0D2C71` (primary accent)
- Definian Green: `#00AB63` (secondary/success)

## Clipboard
- Subject line: `navigator.clipboard.writeText()`
- Email body: `ClipboardItem` with `text/html` blob (fallback: hidden div + execCommand)

## Commands
- No build/test commands — open `index.html` in browser to test
