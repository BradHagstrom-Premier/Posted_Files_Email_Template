# Posted Files Email Generator

A self-contained, browser-based tool that generates Outlook-compatible HTML
emails for Definian's Oracle Cloud conversion engagements.

## What It Does

Two email modes for every test cycle:

- **Posted Files** — pre-load notification when conversion files land in SharePoint
- **Recon Report** — post-load validation result, with a Clean / Issues status toggle

Output renders correctly in Microsoft Outlook's Word-based HTML engine and
plays nicely with CodeTwo's appended signature.

## How to Use

1. **Open `index.html`** in Chrome or Edge — no server, no install
2. **Pick a mode** from the tab strip (Posted Files / Recon Report)
3. **Fill in the form**:
   - Conversion Name, Cycle, Project Prefix (defaults to `Wave 2C` — clear it for other projects)
   - Mode-specific fields (file rows for Posted Files; status, summary, and report URL for Recon)
   - Optional Posted At timestamp (Posted Files only)
4. **Manage recipients** in the collapsible section at the bottom of the form panel — multi-client, To/CC split, persisted to localStorage
5. **Copy** with the buttons or keyboard:
   - `Ctrl+Enter` — Copy Email
   - `Ctrl+Shift+C` — Copy Subject
   - `Ctrl+Shift+T` / `Ctrl+Shift+Y` — Copy To / CC
6. **Paste into Outlook** — formatting is preserved

## Technical Details

- **Zero dependencies** — single HTML file, no build step, no backend
- **Works offline** via `file://`
- **Outlook-compatible output** — table-based layout, inline styles, Arial only
- **State** — form state is ephemeral; recipients persist to `localStorage`

## Verification

There is no automated test suite. To check the design system invariants:

```bash
npx impeccable --json index.html
```

Two persistent false positives are expected and documented in `CLAUDE.md`.

## Project Docs

- [`PRODUCT.md`](PRODUCT.md) — users, purpose, brand personality, design principles
- [`DESIGN.md`](DESIGN.md) — design system, color and type rules, components
- [`CLAUDE.md`](CLAUDE.md) — architecture, Outlook constraints, JS patterns (for AI assistants and contributors)
- [`docs/superpowers/`](docs/superpowers/) — implementation plans and design specs by feature

## Who This Is For

Definian consultants on Oracle Cloud conversion projects.
