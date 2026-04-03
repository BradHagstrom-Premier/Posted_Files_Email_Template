# Recon Report Email Mode — Design Spec

**Date:** 2026-04-03  
**Project:** Posted Files Email Generator (`index.html`)  
**Status:** Approved

---

## Overview

Add a second email type — Recon Report — to the existing single-file Posted Files Email Generator. Recon emails go out after conversion files are loaded into Oracle, confirming that reconciliation reports have been posted and noting any discrepancies.

The two email types share enough context (Conversion Name, Cycle) that they belong in the same tool, and consultants will often generate both in the same session for the same conversion.

---

## Architecture

### Mode switching

A tab strip replaces the current `Email Details` section title at the top of the form panel. Two tabs:

- **Posted Files** (existing behavior, unchanged)
- **Recon Report** (new)

Tab state is tracked in a JS variable (`activeMode`). Switching tabs swaps the mode-specific form fields and immediately re-renders the preview and subject line. No page reload, no lost state.

### Shared state

Conversion Name and Cycle persist across tab switches. They are rendered in the form regardless of active tab; only the mode-specific fields below them swap.

### Email generation

`generateEmailHTML()` and `generateSubjectLine()` branch on `activeMode`. Each mode has its own generation path. The copy bar buttons (Copy Subject, Copy File Locations, Copy Email HTML) work the same in both modes.

---

## Form — Recon Report tab

Four fields total; two shared, two new.

### Shared fields (same inputs, same values as Posted Files tab)

- **Conversion Name** — text input
- **Cycle** — dropdown + custom toggle (existing logic)

### Recon-specific fields

**1. Recon Report Location**

A single label + URL row using the same pattern as the existing File Locations rows. Label pre-filled with `"Premier Recon Reports"`. URL optional. If URL is provided, the folder name renders as a hyperlink in the email; otherwise plain text.

**2. Status toggle — Clean / Issues Found**

A two-segment pill toggle (not a dropdown). 

- **Clean** (default): textarea appears pre-filled with the default no-issues message (editable). Toggle accent: Definian Green `#00AB63`.
- **Issues Found**: textarea clears, placeholder prompts free-text description of discrepancies. Toggle accent: Definian Blue `#0D2C71`.

The textarea in both states is fully editable. The pre-fill for Clean is a starting point, not locked copy.

---

## Email output — Recon Report

Outlook-compatible, table-based, Arial — same constraints as Posted Files. No flexbox, no CSS classes, all styles inline.

### Subject line

```
Wave 2C {Conversion Name} Recon Report - {Cycle}
```

Mirrors the existing Posted Files subject line pattern.

### Body structure

```
Hi team,

{Conversion Name} Recon Reports have been posted to SharePoint: {folder label, linked if URL provided}.

{IF CLEAN}
All records loaded as expected. No discrepancies found.

{IF ISSUES FOUND}
{Free text from textarea — line breaks preserved as <br />}

Thank you.
```

### Copy bar

Same three buttons as Posted Files:
- **Copy Subject** — copies subject line as plain text
- **Copy File Locations** — copies the recon report location row as rich HTML
- **Copy Email HTML** — copies the full email as rich HTML

---

## Humanized copy

The generator pre-fills and generates the following natural-language copy:

- Greeting: `Hi team,` (not "Hi All,")
- Clean status: `All records loaded as expected. No discrepancies found.`
- No "Please note that..." opener for issues — issue text is dropped in directly
- No "There are no issues to callout" phrasing

---

## UI enhancements (frontend-design)

Applied to the generator UI only. Email output HTML is unchanged.

### Typography

Replace `Segoe UI, system-ui` with `DM Sans` (Google Fonts). Degrades to `system-ui` if offline. Applied to the form panel only — not the email output.

### Tab strip

Pill-style tabs with:
- Definian Blue active state
- Thin Definian Green underline indicator on active tab
- 150ms slide transition on the indicator

### Status toggle

Inline two-segment pill:
- Checkmark icon + "Clean" label
- Flag icon + "Issues Found" label
- Active segment fills with the accent color for that state (green/blue)
- CSS-only transition on active segment swap

### Tab content transition

Form content fades in on tab switch: 120ms opacity + 6px translateY. Prevents the hard swap feel.

### No changes to the email output HTML

Outlook constraints are inviolable. All design work stays in the generator UI layer.

---

## Out of scope

- Multiple recon report locations (one folder row only)
- Record count fields or structured discrepancy tables
- Saving/loading session state
- Any backend or build step
