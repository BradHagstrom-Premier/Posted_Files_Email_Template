# Posted Files Email Generator — Design Spec

## Overview

A self-contained, single-file browser-based tool (`index.html`) that generates professionally formatted, Outlook-compatible HTML emails for notifying AWC and Deloitte contacts when conversion files are posted to SharePoint. No dependencies, no backend, no build step.

## Architecture

### Two Rendering Contexts

1. **Generator UI** — runs in a browser with full modern CSS (flexbox, CSS custom properties, transitions)
2. **Email output** — Outlook-compatible HTML using exclusively table-based layout with inline styles

### Single Render Function (Approach A)

One `generateEmailHTML()` function produces the complete Outlook-compatible XHTML string. This same string is:
- Rendered in the preview pane via `<iframe srcdoc>`
- Copied to clipboard when the user clicks "Copy Email HTML"

Zero drift between preview and output. What you see is what Outlook receives.

## Generator UI

### Layout

Split-panel layout with responsive breakpoint at 1024px:

- **Left panel (~45%):** Form inputs, vertically stacked, independently scrollable
- **Right panel (~55%):** Subject line display + "Copy Subject" button at top, live preview iframe in the middle, "Copy Email HTML" button at bottom
- **Header bar:** Definian Blue (#0D2C71) strip with tool title
- **Below 1024px:** Panels stack vertically (form first, preview second)

### Form Fields

| Field | Type | Required | Details |
|-------|------|----------|---------|
| Conversion Name | Text input | Yes | Placeholder: "e.g., Item Master, Purchase Orders" |
| Cycle | Editable dropdown | Yes | Options: SIT1, SIT2, UAT, Mock 1, Mock 2, Mock 3, D-Cut, Other. Selecting "Other" shows a text input replacing the dropdown for free-text entry, with a link to switch back to the dropdown. |
| Summary of Changes | Textarea | No | What changed from last posting |
| File Locations | Repeatable row group | Yes (min 1) | Each row: label + URL. Default 3 rows pre-labeled. Add/remove dynamically. |
| Errors / Fallout Notes | Textarea | No | Known issues, missing items, exclusions |

### Real-time Preview

All fields fire `input` events that trigger `generateEmailHTML()` and update the iframe `srcdoc`. No submit button — preview is always current.

### Brand Colors

- Definian Blue: `#0D2C71` (primary accent — header, borders, buttons)
- Definian Green: `#00AB63` (success states — copy confirmation toasts)

## Email Template

### Subject Line

```
Wave 2C [Conversion Name] Files Posted - [Cycle]
```

Generated dynamically. Displayed prominently in the UI with its own "Copy Subject" button.

### Body Structure

1. **Header accent bar** — `<td bgcolor="#0D2C71" height="4">`, thin blue strip
2. **Greeting** — "Hi All,"
3. **Status line** — "Wave 2C [Conversion Name] [Cycle] files have been posted to SharePoint for pre-load validation."
4. **Summary of Changes** — OMIT ENTIRELY when empty. Bold section header + content when present.
5. **File Locations** — Always present. Numbered list with each label as a hyperlink to the URL. Empty URLs render as plain text.
6. **Errors / Fallout Notes** — OMIT ENTIRELY when empty. When present: nested table with 3px Definian Blue left accent `<td>` + light background (#f0f4fa) callout.
7. **Closing** — "Thank You." No signature (Outlook plugin appends it).

### Outlook HTML Compliance

- XHTML Transitional doctype
- Outer `<table width="600" align="center">`
- All styles inline on every element — `font-family: Arial, sans-serif; font-size: 14px; color: #333333;`
- Links: `style="color: #1a73e8; text-decoration: underline;"`
- Section headers: inline bold + larger font-size on `<td>` (no `<h1>`-`<h6>`)
- Spacing: `<tr><td height="16">&nbsp;</td></tr>` (no margin/padding)
- Accent bars: `<td bgcolor="...">` with explicit height (no CSS borders)
- Fallout callout: nested 2-column table — left `<td width="3" bgcolor="#0D2C71">` + right `<td bgcolor="#f0f4fa">`
- Zero CSS classes, `<style>` blocks, flexbox, grid, `<div>` layout, or CSS shorthand

## Clipboard

### Subject Line

`navigator.clipboard.writeText(subjectString)` — plain text.

### Email Body

Primary:
```javascript
const blob = new Blob([emailHtml], { type: 'text/html' });
const item = new ClipboardItem({ 'text/html': blob });
await navigator.clipboard.write([item]);
```

Fallback (for `file://` or older browsers):
Hidden `<div>` injection + `document.createRange()` / `window.getSelection()` + `document.execCommand('copy')`.

### Toast Feedback

Floating element, bottom-center, Definian Green background, "Copied!" text, ~2 second display with CSS fade animation. No `alert()` boxes. Copy buttons get brief disabled state during toast.

### Copy Failure

If both methods fail: toast with "Copy failed — try Ctrl+C from the preview."

## Edge Cases

- **Empty required fields:** Preview shows bracketed placeholders (e.g., "[Conversion Name]") until filled. No aggressive validation.
- **Empty URLs:** Label renders as plain text, not a broken hyperlink.
- **No persistence:** Refreshing clears all state. No warning — tool is for quick one-off use.

## Deliverables

- `index.html` — the generator tool (all HTML, CSS, JS in one file)
- `README.md` — usage instructions
- `.gitignore` — standard web ignores
- `CLAUDE.md` — project context for future sessions
