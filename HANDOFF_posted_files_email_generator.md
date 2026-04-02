# Handoff: Conversion File Posting Email Generator

## Context

Definian consultants (Brad Hagstrom, Fyzia Shaik, and others) post conversion files (FBDI loads, validation files, conversion logs) to SharePoint at the end of every test cycle and notify AWC and Deloitte contacts by email. Today these emails are written manually in plain text — functional but inconsistent with no visual hierarchy or standardized structure.

This tool is a self-contained browser-based HTML generator. The consultant fills in details for a specific posting, gets a formatted Outlook-compatible email body (plus subject line), and copies both into Outlook. No backend. No persistence. Zero external dependencies.

### Critical architectural constraint — two rendering contexts

**Context 1: The generator UI** (runs in a browser)
- Full modern CSS, polished UX, Definian brand colors
- This is where the `frontend-design` skill earns its keep

**Context 2: The generated email HTML** (pasted into Outlook)
- Outlook's Word-based HTML renderer ignores flexbox, grid, CSS classes, shorthand properties, and most modern CSS
- Must use **table-based layout exclusively**
- All styles must be **inline on every element** — no `<style>` blocks, no CSS classes
- Explicit `width`, `cellpadding`, `cellspacing`, `border` attributes on all tables
- Font stack: `Arial, sans-serif` everywhere
- No `<div>` for layout — only `<table>`, `<tr>`, `<td>`
- `mso-` conditional comments where needed for Outlook-specific rendering fixes
- `&nbsp;` for spacing, not margin/padding tricks Outlook ignores
- Spacing between sections via empty `<tr>` with explicit `height` attribute
- Colored accent bars via `<td bgcolor="...">` with explicit width/height — not `border-left` (unreliable in Outlook)
- Left-border callout accent (for the Errors/Fallout section) via a 3px-wide `<td>` inside a nested table — not CSS `border-left`

**Do not compromise Outlook compatibility for aesthetics.** A slightly less beautiful email that renders perfectly in Outlook is the correct tradeoff.

---

## Brand Colors

| Name | Hex |
|------|-----|
| Definian Blue | `#0D2C71` |
| Definian Green | `#00AB63` |

Use Definian Blue as the primary accent in both the generator UI and the email output. Definian Green is available for secondary accents (e.g., success states in the UI, secondary highlights). Do not over-brand — these are enterprise emails going to AWC operations and IT contacts who value clarity over style.

---

## Scope

**Repo:** `https://github.com/BradHagstrom-Premier/Posted_Files_Email_Template`

This repo is yours to set up fully. Create whatever supporting files are appropriate:
- `index.html` — the generator tool (primary deliverable)
- `README.md` — describe what the tool is, how to use it, and how to open it
- `.gitignore` — standard web project ignore file
- `CLAUDE.md` — project context and architecture notes for future Claude Code sessions
All deliverables sit flat in the repo root unless you have a strong structural reason otherwise. No build step, no package.json, no external dependencies of any kind.

---

## Step-by-Step Instructions

### Step 1: Initialize the repo and plan

Before writing any application code:

1. Create `README.md`, `.gitignore`, and `CLAUDE.md` with appropriate content and ensure you utilize the claude-md-management skill that you have.
2. Invoke the **`brainstorming`** skill to evaluate:
   - Layout options for the generator form (split-panel left form / right preview vs. top-to-bottom flow — split panel is likely better for live preview UX on a laptop)
   - The email template structure (see content spec in Step 3)
   - How to handle copy-to-clipboard so Outlook receives it as rich HTML, not plain text
   - How to make the tool immediately intuitive for a non-technical consultant (Fyzia) who will use this independently without training
3. Invoke the **`writing-plans`** skill to produce a concrete implementation plan before writing any HTML/CSS/JS.

---

### Step 2: Build the generator UI

Invoke the **`frontend-design`** skill before writing the generator interface. Follow its guidance fully — this is the primary skill investment for this project. The UI should be polished and professional, not a generic form.

**Generator form fields:**

| Field | Type | Details |
|-------|------|---------|
| Conversion Name | Text input | e.g., "Item Master", "Purchase Orders", "On-Hand Balances" |
| Cycle | Editable dropdown | Options: SIT1, SIT2, UAT, Mock 1, Mock 2, Mock 3, D-Cut — plus free-text custom entry |
| Summary of Changes | Textarea | Optional. What changed from last posting, what's new, notable decisions. |
| File Locations | Repeatable row group | Each row: label (text input) + URL (text input). Add/remove rows dynamically. Default 3 rows pre-labeled: "Conversion/Error Log", "Validation Files", "FBDI File(s)". Min 1 row, no max. |
| Errors / Fallout Notes | Textarea | Optional. Free-form notes on missing items, known issues, records excluded. |

**Generator UI requirements:**

- **Live preview pane** that updates in real-time as the user types — this is the single most important UX feature
- **Subject line display** — show the generated subject line above the preview, prominently, with its own "Copy Subject" button
- **"Copy Email HTML" button** — copies the raw Outlook-compatible HTML (not rendered text) to clipboard
- Visual confirmation on copy — brief toast/flash, not a browser alert box
- Clean, professional aesthetic using Definian Blue (`#0D2C71`) as the primary accent; Definian Green (`#00AB63`) for secondary/success states
- Responsive enough to work on a laptop screen without horizontal scrolling
- All state lives in the browser session — no backend, no persistence needed
- No external dependencies — file must work by double-clicking it locally (file:// protocol)

---

### Step 3: Build the Outlook-compatible email template (guideline but can be changed if clear improvement exists)

Humanize (/humanizer-skill:humanizer) the email.

**Subject line format:**
```
Wave 2C [Conversion Name] Files Posted - [Cycle]
```
Generated dynamically from the Conversion Name and Cycle fields. Displayed prominently in the UI as a separately copyable field.

**Email body structure:**

```
1. Greeting: "Hi All,"

2. Status line:
   "Wave 2C [Conversion Name] [Cycle] files have been posted to SharePoint for pre-load validation."

3. Summary of Changes section
   — OMIT ENTIRELY (no header, no whitespace) if the field is empty

4. File Locations section
   — Numbered list with each label as a hyperlink to the provided URL
   — Always rendered (at least 1 row is required)

5. Errors / Fallout Notes section
   — OMIT ENTIRELY if the field is empty
   — If content is present, render inside a visual callout:
     nested table with a 3px Definian Blue left-edge <td> and a very light
     background (#f0f4fa or similar) — so it is not overlooked by reviewers


```

> Note: No Outlook signature is generated. Definian uses an Outlook plugin that appends the official company signature automatically when the consultant sends.

**Outlook HTML compliance rules — enforce strictly:**

- Wrap entire email in `<table width="600" align="center">`
- Every visual element uses `<table>` layout with inline styles
- Every `<td>` and `<p>` carries full font declaration inline: `style="font-family: Arial, sans-serif; font-size: 14px; color: #333333;"`
- Links styled inline: `style="color: #1a73e8; text-decoration: underline;"`
- Section headers use inline bold + slightly larger font-size on `<td>` — not `<h1>`–`<h6>` tags (Outlook renders heading tags inconsistently)
- Spacing between sections: `<tr><td height="16">&nbsp;</td></tr>` — not margin or padding
- Header accent bar: `<td bgcolor="#0D2C71">` with explicit `height` attribute
- Fallout callout: nested 2-column table — left `<td width="3" bgcolor="#0D2C71">` + right `<td>` with light background and content
- Output includes proper XHTML doctype: `<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">`
- Zero CSS classes in email output
- Zero `<style>` blocks in email output
- Zero flexbox, grid, or CSS shorthand in email output
- Zero `<div>` elements used for layout in email output

---

### Step 4: Implement copy-to-clipboard

**Subject line copy:** Plain text — `navigator.clipboard.writeText(subjectString)`.

**Email body copy:** Must deliver raw HTML so Outlook receives it as rich HTML on paste.

Primary method:
```javascript
const blob = new Blob([emailHtml], { type: 'text/html' });
const item = new ClipboardItem({ 'text/html': blob });
await navigator.clipboard.write([item]);
```

Fallback (if `ClipboardItem` is unavailable):
Write the HTML into a hidden `<div>`, select its contents with `document.createRange()` / `window.getSelection()`, then `document.execCommand('copy')`.

Show a brief toast ("Copied!") on success for both copy actions.

---

### Step 5: Verify before completing

Invoke the **`verification-before-completion`** skill. Verify all of the following:

- [ ] Generator renders cleanly in Chrome and Edge without horizontal scroll on a standard laptop viewport
- [ ] All form fields populate the preview in real-time with no perceptible lag
- [ ] Subject line generates correctly from Conversion Name + Cycle fields; "Copy Subject" button works
- [ ] "Copy Email HTML" copies Outlook-compatible HTML as rich HTML (not plain text)
- [ ] Pasting into Outlook compose (or Outlook Web) renders correctly — tables, colors, links, and spacing all intact
- [ ] Optional sections (Summary of Changes, Errors/Fallout) are completely absent from output when empty — no empty headers, no blank space in their place
- [ ] File location rows can be added and removed dynamically; all links render as clickable hyperlinks in the email output
- [ ] The tool works as a standalone `.html` file opened locally via file:// protocol
- [ ] Email output contains zero CSS classes, zero `<style>` blocks, zero flexbox/grid, zero `<div>` layout elements, zero CSS shorthand properties
- [ ] The Errors/Fallout callout section renders with the left-border accent when content is present; is fully absent when content is not present
- [ ] You have used the humanizer plugin and skill (/humanizer-skill:humanizer) to review the written contents of the email.

---

### Step 6: Commit

Use **`commit-commands`** to commit all files with a descriptive initial commit message.

---

## Reference: Current Email Baseline (Fyzia's format)

Subject: `Wave 2C [Conversion Name] Files Posted - [Cycle]`

```
Hi All,

[Cycle] [Conversion Name] Initial files have been posted to SharePoint for pre-load validation.

File locations:
  • Conversion/Error Log: [link]
  • Validation Files: [link]
  • FBDI File(s): [link]

As discussed, please note the below [N] Missing Item Numbers were not found in JDE F4101.

[list of items]

Thank You.

[Signature appended automatically by Outlook plugin]
```

The generator should improve on this baseline with better visual hierarchy and consistent structure. Keep the same professional, concise tone. Do not make the email flashy — it goes to enterprise IT and operations contacts who value clarity.

---

## Key Decisions Made

- No signature block is generated — Definian has an Outlook plugin that appends a standardized company signature automatically; do not attempt to replicate it
- Subject line is generated and copyable as a separate action from the email body
- Repo initialization (README, .gitignore, CLAUDE.md) is fully in scope for this session
- All state is ephemeral (browser session only) — no localStorage, no backend
- Brand colors: Definian Blue `#0D2C71`, Definian Green `#00AB63`