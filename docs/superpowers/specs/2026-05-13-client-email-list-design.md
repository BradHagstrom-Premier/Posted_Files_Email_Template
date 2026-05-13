# Client Email List — Design Spec

**Date:** 2026-05-13
**Status:** Approved

## Problem

Clients periodically request specific people be included on Posted Files and Recon Report emails. Consultants currently track this manually. The tool needs a built-in, per-client contact list so consultants always know who to address and can copy a ready-made To: line into Outlook.

## Scope

- Single HTML file, no backend, no build step — consistent with existing architecture
- localStorage persistence — each consultant's copy of the file maintains their own client lists independently
- Both email modes (Posted Files and Recon Report) share the same Recipients section

## UI Placement

A collapsible "Recipients" section added at the bottom of the form panel, below the mode-specific fields, separated by a divider. Collapsed by default to keep the form uncluttered.

## Three States

### Collapsed
The section header shows: "RECIPIENTS" label, active client name + contact count as a badge, and a quick "Copy ▸" button. One click copies without expanding.

### Expanded
- Client dropdown (lists all configured clients; selecting switches the active client)
- Contact list: each contact shown as `[checkbox] Name — email`, all checked by default
- "+ Add contact" link below the list
- "Copy Recipients" primary button (disabled + relabeled "No contacts selected" when none are checked)
- "Manage" text link that switches to manage mode

### Manage Mode
Replaces the section body inline — no modal. Contains:
- Client name text input with a "Delete client" button (red, destructive)
- Contact rows: name input + email input + "×" delete button, one row per contact
- "+ Add contact" link appends a new empty row
- "Save" button persists changes to localStorage
- "+ New Client" button creates a blank client and switches to editing it
- "← Done" link in the header returns to expanded view; any unsaved edits are discarded (Save is the explicit commit action)

## Data Model

Stored in localStorage under the key `definian_recipients`.

```json
{
  "activeClientIndex": 0,
  "clients": [
    {
      "name": "AWC",
      "contacts": [
        { "name": "Patty Smith", "email": "patty.smith@awc.gov", "checked": true },
        { "name": "Linh Truong", "email": "linh.truong@awc.gov", "checked": true }
      ]
    }
  ]
}
```

- `activeClientIndex` — index into `clients` array; determines which client is displayed
- `checked` — persisted per contact so unchecked state survives page refresh
- On page load: read from localStorage; if key is absent, start in empty state

## Copy Recipients

- Output format: `email1@domain.com; email2@domain.com` (semicolon-separated, Outlook-compatible)
- Only contacts with `checked: true` are included
- Written to clipboard via `navigator.clipboard.writeText()` — same pattern as subject line copy
- Toast notification on success (reuses existing toast infrastructure)
- If no contacts are checked: button is disabled, label changes to "No contacts selected"

## Edge Cases

| Scenario | Behavior |
|---|---|
| No clients configured | Section shows "No clients — click Set up to add your first" with inline CTA |
| Active client has no contacts | Contact area shows "No contacts yet — add one" |
| All contacts unchecked | Copy button disabled, labeled "No contacts selected" |
| Active client deleted | Switches to next available client; if none remain, shows empty state |
| Single client (no dropdown needed) | Dropdown still renders — keeps UI consistent, allows adding more clients later |

## JS Patterns

- `recipientsState` object holds runtime state (mirrors localStorage schema)
- `loadRecipientsState()` — reads localStorage on page load, sets defaults if absent
- `saveRecipientsState()` — writes current `recipientsState` to localStorage; called on every mutation
- `renderRecipients()` — full re-render of the section DOM from `recipientsState`
- `copyRecipients()` — builds semicolon-separated string from checked contacts, writes to clipboard
- Section toggle, client switch, contact toggle, manage mode enter/exit all call `saveRecipientsState()` then `renderRecipients()`
- No interaction with `updatePreview()` — recipients are not part of the email body or subject line

## Out of Scope

- Recipients do not appear in the generated email HTML or subject line
- No import/export of contact lists
- No sync across machines or consultants
- No per-mode (Posted Files vs. Recon) contact filtering
