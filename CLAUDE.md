# Posted Files Email Generator

## Architecture
- Single-file browser tool (`index.html`) — no build step, no dependencies, no backend
- Two email modes: **Posted Files** (pre-load notification) and **Recon Report** (post-load validation result)
- Mode is controlled by `activeMode` JS variable (`'posted'` | `'recon'`); tab strip in the UI switches it
- Conversion Name and Cycle are shared state across both modes — all other fields are mode-specific
- Two rendering contexts: (1) generator UI (DM Sans, modern CSS) and (2) email output (Outlook-compatible, Arial only)
- All state is ephemeral (browser session only)

## Outlook Email HTML Constraints (Critical)
- Table-based layout only — no flexbox, grid, or `<div>` layout
- All styles inline on every element — no `<style>` blocks, no CSS classes
- Font stack: `Arial, sans-serif` on every text element
- Spacing via `<tr><td height="N">&nbsp;</td></tr>` — not margin/padding
- Accent bars via `<td bgcolor="...">` — not CSS borders
- Links: `style="color: #1a73e8; text-decoration: underline;"`
- XHTML Transitional doctype required
- Outer body `<table>` must NOT set `align="left"` — Outlook's Word renderer treats it as floated, causing CodeTwo-appended signatures to render alongside the body in the compose draft view. Use `width="600"` only.
- Email body ends at its last content row immediately followed by `</table>` — no trailing "Thank you." line, no trailing spacer. CodeTwo's signature provides its own top margin. Gap between body and signature is controlled by CodeTwo's anchor, not by our HTML.

## Brand Colors
- Definian Blue: `#0D2C71` (primary accent)
- Definian Green: `#00AB63` (secondary/success)

## Clipboard
- Subject line: `navigator.clipboard.writeText()`
- Email body: `ClipboardItem` with `text/html` blob (fallback: hidden div + execCommand)

## JS Patterns
- `generateSubjectLine()` and `generateEmailHTML()` branch on `activeMode` at the top of each function — follow this pattern for any future mode-specific logic
- Copy File Locations handler also branches: `activeMode === 'recon' ? generateReconFileLocationsHTML() : generateFileLocationsHTML()`
- Recon status toggle tracks `reconStatus` (`'clean'` | `'issues'`) separately from `activeMode`

## Commands
- No build/test commands — open `index.html` in browser to test
- Verify both tabs: Posted Files and Recon Report, including Clean/Issues toggle and Copy buttons

## graphify

This project has a graphify knowledge graph at graphify-out/.

Rules:
- Before answering architecture or codebase questions, read graphify-out/GRAPH_REPORT.md for god nodes and community structure
- If graphify-out/wiki/index.md exists, navigate it instead of reading raw files
- After modifying code files in this session, run `python3 -c "from graphify.watch import _rebuild_code; from pathlib import Path; _rebuild_code(Path('.'))"` to keep the graph current
