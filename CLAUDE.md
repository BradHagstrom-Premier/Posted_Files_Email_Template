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
- Links: `style="color: #0D2C71; text-decoration: underline;"` — Definian Blue, not Google Material Blue
- Recon Report accent bar color is conditional on `reconStatus`: `#00AB63` (Signal Green) when clean, `#0D2C71` (Definian Blue) when issues
- XHTML Transitional doctype required
- Outer body `<table>` must NOT set `align="left"` — Outlook's Word renderer treats it as floated, causing CodeTwo-appended signatures to render alongside the body in the compose draft view. Use `width="600"` only.
- Email body ends at its last content row immediately followed by `</table>` — no trailing "Thank you." line, no trailing spacer. CodeTwo's signature provides its own top margin. Gap between body and signature is controlled by CodeTwo's anchor, not by our HTML.

## Brand Colors
- Definian Blue: `#0D2C71` (primary accent)
- Definian Green: `#00AB63` (secondary/success)

## Clipboard
- Subject line: `navigator.clipboard.writeText()`
- Email body: `ClipboardItem` with `text/html` blob (fallback: hidden div + execCommand)

## Shared Form Fields
- **Conversion Name**, **Cycle**, and **Project Prefix** are shared across both modes
- **Project Prefix** defaults to `"Wave 2C"` (pre-filled); consultants on other projects clear or change it. Omitting it drops the prefix entirely — no leading space.
- **Posted At** is Posted Files mode only (optional timestamp appended to the opening sentence)
- File row label inputs show an amber warning border (`.form-input.warn:not(:focus)`) when a URL is present but the label field is empty

## JS Patterns
- `generateSubjectLine()` and `generateEmailHTML()` branch on `activeMode` at the top of each function — follow this pattern for any future mode-specific logic
- Copy File Locations handler also branches: `activeMode === 'recon' ? generateReconFileLocationsHTML() : generateFileLocationsHTML()`
- Recon status toggle tracks `reconStatus` (`'clean'` | `'issues'`) separately from `activeMode`
- `checkFileRowWarnings()` is called from `updatePreview()` on every state change; it applies/removes `.warn` on Posted Files file row label inputs
- Recon email sentence suppresses the location label when no URL is provided: `': [link]'` only appears when `reconReportUrl` has a value
- `escapeHtml(str)` is already defined (uses `textContent` → `innerHTML` trick) — reuse it; do not re-declare

## Recipients (Client Email List)
- Collapsible section at bottom of form panel — UI-only, NOT included in email output (no `updatePreview()` coupling)
- State in `recipientsState` JS object; mirrored to localStorage under `definian_recipients`
- State shape: `{ activeClientIndex, expanded, manageMode, manageDirty, popoverOpen, clients: [{ name, contacts: [{ name, email, role, checked }] }], overrides }`. Multi-client is supported via the `clients[]` array — only `activeClientIndex`, `expanded`, and `clients` are persisted.
- Render pattern: every mutation calls `saveRecipientsState()` then `renderRecipients()` which does full `innerHTML` replacement and re-wires events via `wireRecipientsEvents()`
- `renderRecipients()` is the orchestrator — it calls `renderHeaderPill()` and `renderEnvelopeStrip()` on every render, then either `renderManageMode()` (when `manageMode`) or builds the collapsed/expanded body inline.
- **Three persistent UI surfaces, three distinct jobs**: (1) header pill `Sending to: <client> · <count> ▾` for orientation + client switching; (2) envelope strip above preview iframe for at-a-glance `To: N · CC: N · Client` verification; (3) collapsible section at form panel bottom for management. Don't conflate them.
- `renderManageMode()` is the inline editor; toggled by `recipientsState.manageMode`
- Manage mode has a left rail listing all clients — rail switches preserve in-memory edits across clients; Save commits the whole `recipientsState` atomically; Done with `manageDirty: true` shows an inline Save & Close / Discard confirm
- Visible across both tabs (lives outside `#postedFilesFields` / `#reconFields`)

## Design Constraints (locked-in)
- **Font-size ladder** is a closed set of 6 documented DESIGN.md steps: `15 / 14 / 13.5 / 12.5 / 11.5 / 11px`. Don't add new sizes — snap to nearest. Email-output sizes (14/15px inside JS template literals) are an independent Outlook-mandated set.
- **Keyboard shortcuts**: `Ctrl+Enter` Copy Email, `Ctrl+Shift+C` Copy Subject, `Ctrl+Shift+T` Copy To, `Ctrl+Shift+Y` Copy CC, `←/→` on mode tabs and status toggle, `Esc` layered (popover → manage mode w/ dirty guard → expanded recipients). `?` pill in header opens the reference.
- **`.form-input.warn:not(:focus)`** is the non-blocking warn pattern (amber border, amber halo). Reused on file-row labels (URL without label) and manage-mode email inputs (invalid format). Don't block save; just signal.

## Commands
- No build/test commands — open `index.html` in browser to test
- Verify both tabs: Posted Files and Recon Report, including Clean/Issues toggle and Copy buttons
- Implementation plans live in `docs/superpowers/plans/`; design specs in `docs/superpowers/specs/`
- Chrome DevTools MCP is preferred for scripted E2E verification (no automated test suite exists)
- `npx impeccable --json index.html` runs the detector. Two persistent false positives: `overused-font: arial` (Arial only in email-output JS template literals — Outlook constraint) and `flat-type-hierarchy: 1.4:1` (intentional dense Product-register ladder matching DESIGN.md exactly).

## graphify

This project has a graphify knowledge graph at graphify-out/.

Rules:
- Before answering architecture or codebase questions, read graphify-out/GRAPH_REPORT.md for god nodes and community structure
- If graphify-out/wiki/index.md exists, navigate it instead of reading raw files
- After modifying code files in this session, run `python3 -c "from graphify.watch import _rebuild_code; from pathlib import Path; _rebuild_code(Path('.'))"` to keep the graph current
