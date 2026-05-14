# Recipients Restructure — Design Spec

**Date:** 2026-05-14
**Author:** Brad Hagstrom (with Claude)
**Status:** Draft pending review
**Scope:** Generator UI only — no email-output HTML changes

## Context

The V6/V7 Recipients work introduced multi-client support (`recipientsState.clients[]`, `activeClientIndex`) but did not surface it in the UI. During this brainstorm we discovered the consultant believed the tool stored only one client at a time — the active list of 11 AWC contacts plus a 2-contact "New Jersey" list were both present in localStorage but invisible because:

1. The client switcher dropdown renders only when the section is **expanded AND** not in manage mode.
2. The `+ New Client` button creates an empty client, makes it active, and leaves the user in manage mode looking at an empty editor — visually indistinguishable from "the previous client was overwritten."

The critique also flagged several adjacent issues:
- Collapsed Recipients header carries 5 affordances; the chevron loses contrast against two filled copy buttons.
- The active client is conveyed only by a faint badge inside the (collapsible) Recipients section. A consultant juggling 2–3 clients in a week has no chrome-level signal of which client is loaded.
- No keyboard-only path through manage mode; the section header is a `<div>` with `cursor:pointer`.
- The `Done` button in manage mode can silently discard unsaved edits.

## Goals

1. Make multi-client capability visible without forcing it on consultants who use the tool single-client.
2. Provide a chrome-level "you are sending to *this* client" signal that survives section collapse and mode switches.
3. Restore keyboard operability across the Recipients surface.
4. Prevent silent data loss in manage mode.
5. Leave the Recipients feature opt-in — consultants who paste addresses from elsewhere shouldn't see any new required step.

## Non-Goals

- Email-format validation in manage mode (separate task).
- Differentiating the twin segmented controls (`.mode-tabs` vs `.status-toggle` — separate task).
- Font-size ladder tightening (separate task).
- DESIGN.md self-violations (header gradient, toast radius, em dash — separate task).
- Any change to generated Outlook email HTML.

## Architecture

No data model changes. `recipientsState` already supports the full design:
```js
recipientsState = {
  clients: [{ name, contacts: [{ name, email, role }] }, ...],
  activeClientIndex: number,
  expanded: boolean,
  manageMode: boolean,
  overrides: { [contactIdx]: 'to' | 'cc' },
}
```

Three new pieces of state:
- `recipientsState.expanded` is newly persisted to localStorage (today resets each load).
- `recipientsState.manageDirty: boolean` tracks unsaved edits in manage mode.
- `recipientsState.popoverOpen: boolean` for the header pill popover (transient, not persisted).

Three UI surfaces with three distinct jobs:

| Surface | Job | Visibility |
|---|---|---|
| App header pill (top-right) | Orientation: which client am I sending to? | Always visible (chrome) |
| Preview envelope strip (above iframe) | Verification: counts and client name at-a-glance | Visible when active client has contacts |
| Recipients section (bottom of form) | Management: edit contacts, switch roles, copy lists | Collapsed by default; opt-in |

## §1. Header "Sending to:" Pill

**Placement.** Top-right of `.app-header`, left of the V7 version badge with an 8px gap.

**Resting visual.**
- Background `rgba(255, 255, 255, 0.18)`; hover `0.28`; active (popover open) flips to white with `#0D2C71` text.
- Padding `5px 10px`, radius `6px`.
- Font DM Sans 600, 11.5px, white.
- Trailing chevron 10px, white 70% opacity; rotates 180° when popover open.

**Copy variants (state-driven).**
| State | Pill text | Treatment |
|---|---|---|
| 0 clients | `+ Add recipients` | Dashed outline, transparent bg; click → manage mode for a new client |
| 1+ clients, active 0 contacts | `Sending to: AWC ▾` | Amber chevron warn tint |
| 1+ clients, active has contacts | `Sending to: AWC · 11 ▾` | Normal |
| Active client unnamed | `Sending to: (Untitled) ▾` | Amber chevron warn tint |
| Viewport < 1024px | Prefix `Sending to: ` drops; `AWC · 11 ▾` only | Same treatment |

**Popover.**
- 240px wide, white bg, `shadow-md`, 8px radius. Anchored beneath pill.
- Content order:
  1. Client list — each row: client name + ` · N contacts` muted suffix. Active row has 4px `#0D2C71` left-edge fill and bold name. Hover applies `--surface-bg` tint.
  2. 1px divider.
  3. `+ New client` link, Definian Blue, dashed underline. Click → enters manage mode with a new empty client and focuses the name field.
  4. `Manage clients...` link. Click → expands the Recipients section, enters manage mode for the active client, scrolls form panel to it, closes the popover.

**Interaction.**
- Click pill toggles popover. Click outside or Esc closes.
- Arrow Up/Down navigate rows; Enter selects an option; focus returns to pill on close.
- Switching active client triggers existing `recipientsState.activeClientIndex` assignment + `saveRecipientsState()` + `renderRecipients()` + `renderEnvelopeStrip()`.
- The pill button gets `aria-haspopup="menu"`, `aria-expanded`.

## §2. Recipients Section — Collapsed Header

Five affordances reduce to three weight tiers.

```
[ Recipients ]  [ AWC · 11 ]              [ Copy To ] [ Copy CC ]  [▾]
   title          badge                    secondary    secondary   chevron (primary affordance)
```

**Changes from today:**
- Both inline copy buttons drop to secondary style (transparent bg, `#0D2C71` text, 1.5px border). Consistency: today's collapsed-state Copy To is primary-filled while Copy CC is secondary; both become secondary because at this altitude the primary action is "expand," not "copy."
- Chevron color shifts from `--text-secondary` to `--text-primary` for stronger affordance contrast.
- Section header element converts from `<div cursor:pointer>` to `<button>`. Embedded child buttons (Copy To, Copy CC) keep `e.stopPropagation()` and a `closest('button')` guard on the parent click handler.
- `recipientsState.expanded` is persisted to localStorage; restored on load. Default still collapsed for new sessions.

**Expanded view:** no structural changes. Same contact list with role pills, same `Copy To` (primary) / `Copy CC` (secondary) / `Manage` / `+ New Client` buttons.

## §3. Manage Mode — Multi-Client Visibility

The bug we found is here: today's manage mode shows only one client at a time and provides no signal that others exist.

**New layout:**
```
┌─ Manage Clients ──────────────────────────── ← Done ┐
│ ┌─ Rail ─────────┐  ┌─ Editor ─────────────────┐   │
│ │ ● AWC      11  │  │ Client name              │   │
│ │   New Jersey 2 │  │ [AWC                   ] │   │
│ │   (Untitled) 0 │  │                          │   │
│ │ ───────────────│  │ Contacts                 │   │
│ │ + New client   │  │ [Name] [email]  [To] ×   │   │
│ │                │  │ ...                      │   │
│ │ [Delete client]│  │ + Add contact            │   │
│ └────────────────┘  │ [Save]                   │   │
│                     └──────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**Left rail (new):**
- Width 180px on viewports ≥1024px; collapses to horizontal pill scroller above editor on narrow viewports.
- Each row: `<name> · <contactCount>`. Active row has 4px `#0D2C71` left-edge fill (matches popover treatment for visual continuity).
- Click any row switches the editor. In-memory edits to the previous client persist to `recipientsState` on switch — no data loss. Dirty-state warning fires on **Done**, not on switch.
- `+ New client` lives at the bottom of the rail. Appends to `recipientsState.clients`, switches editor to it, focuses the name field. Rail visibly grows so previous clients remain visible.
- `Delete client` button below `+ New client`. Acts on the currently-selected client; only renders when `clients.length > 1`.
- Rail is a `role="listbox"` with arrow-key navigation, Enter to select.

**Right editor:**
- Same content as today's manage form (client name input, contacts table, `+ Add contact`, `Save`).
- Removes the inline "Delete" button next to the client name (now in rail).
- Removes the `+ New Client` button at the bottom (now in rail).

**Save semantics.** The `Save` button commits **all in-memory `recipientsState` changes** to localStorage, not just the visible client's edits. This matches the existing `saveRecipientsState()` behavior (whole-state dump) and keeps the model simple: rail-switching mid-edit accumulates dirty changes across clients, then one `Save` flushes everything atomically.

**Dirty-state Done guard.**
- `recipientsState.manageDirty` flag, set on any input change inside manage mode. Persists across rail switches.
- Click Done (or press Esc) with `manageDirty: true` → inline confirm renders in the manage header: `Discard unsaved changes? [Save & Close] [Discard]`. No browser `confirm()` dialog.
- Click Done with `manageDirty: false` → today's behavior (auto-clean empty unnamed clients, exit manage mode).
- Clicking `Save` clears `manageDirty` and stays in manage mode (today's behavior).

## §4. Preview Envelope Strip

Lives above the preview iframe, below the `Copy Email` / `Copy File Locations` action bar.

**Visual.**
- 28px total height (text + 8px vertical padding).
- DM Sans 600, 12.5px, tabular-nums on digits.
- Labels (`To:`, `CC:`) in `--text-secondary`; values and client name in `--text-primary`.
- Separator: ` · ` (middle dot with breathing room, matches existing badge separator).
- Background `--surface-white`. Top and bottom 1px `--border-default` dividers frame the strip without making it a card.

**Content order:** `To: N · CC: N · Client` — counts first (verification), name last (qualifier).

**State variants:**
| Active client state | Strip content |
|---|---|
| No client configured | Hidden entirely (strip + dividers removed) |
| Client configured, 0 contacts | `No recipients yet · AWC` muted, with subtle `+ Add` link right-aligned |
| Client configured, contacts (CC may be 0) | `To: 11 · CC: 1 · AWC` |

**Interactivity.**
- Whole strip is a quiet `<button>`. Click scrolls to and expands the Recipients section.
- Hover lifts background to `--surface-bg` faintly. No outline, no chevron — discoverable through hover.
- `aria-label="Recipients: 11 to, 1 cc, AWC. Click to manage."`

**Reactivity.** A new `renderEnvelopeStrip()` function runs alongside `renderRecipients()` after every `recipientsState` mutation. Both pull from the same in-memory state.

## §5. Keyboard Shortcuts

Folded into this spec because the shortcuts touch the new surfaces.

| Shortcut | Action |
|---|---|
| `Ctrl+Enter` (any form field) | Copy Email + toast |
| `Ctrl+Shift+C` | Copy Subject + toast |
| `Ctrl+Shift+T` | Copy To recipients + toast |
| `Ctrl+Shift+Y` | Copy CC recipients + toast |
| `←` / `→` on `.mode-tabs` or `.status-toggle` | Switch between tablist options |
| `Esc` | Close popover; close manage mode (with dirty guard); collapse expanded Recipients |
| `Space` / `Enter` on Recipients section header | Toggle expand/collapse |

A small `?` info pill in the header chrome (between the "Sending to:" pill and the V7 badge) opens a popover listing the shortcuts.

## §6. Accessibility

- Header pill: `<button aria-haspopup="menu" aria-expanded>`. Popover `role="menu"`, items `role="menuitem"`.
- Recipients section header: `<button>` (no more `<div cursor:pointer>`).
- Manage mode rail: `role="listbox"`, items `role="option"` with `aria-selected`.
- Role pill on contacts: `aria-pressed="true"` when role is CC, `"false"` for To. Label remains visible.
- Preview envelope strip: `<button>` with `aria-label` covering counts and client name.
- All new interactive surfaces respect the existing `:focus-visible` halo (3px `rgba(13, 44, 113, 0.08)`).
- All new animations (popover open, rail switch transitions) respect the existing `@media (prefers-reduced-motion: reduce)` block.

## §7. State Matrix

| Clients | Active client | Pill | Strip | Section badge |
|---|---|---|---|---|
| 0 | n/a | `+ Add recipients` (dashed) | Hidden | `No client configured` (muted) |
| 1+ | named, 0 contacts | `Sending to: AWC ▾` (amber) | `No recipients yet · AWC` | `AWC · 0` |
| 1+ | named, N contacts | `Sending to: AWC · N ▾` | `To: x · CC: y · AWC` | `AWC · N` |
| 1+ | unnamed | `Sending to: (Untitled) ▾` (amber) | `No recipients yet · (Untitled)` | `(Untitled) · 0` |
| 2+ | any | Same as 1-client variants; popover lists all | Same | Same |
| any | manage mode active | Chevron points up; popover disabled while editing | Reflects last-saved state | Section renders manage editor |

## §8. Implementation Notes

- All work in a single file: `index.html`. No new files, no build step, no dependencies.
- New CSS rules added to the existing `<style>` block, scoped to new class names (`.recipients-pill`, `.recipients-popover`, `.recipients-rail`, `.recipients-envelope-strip`, etc.). No global rule changes that could affect email-output HTML.
- New JS functions: `renderHeaderPill()`, `renderPopover()`, `renderEnvelopeStrip()`, `renderManageRail()`. All called from existing `renderRecipients()` orchestrator. Existing `wireRecipientsEvents()` extends to wire new elements.
- localStorage migration: read existing `definian_recipients`, add `expanded: false` default if absent. `manageDirty` is in-memory only.

## §9. Testing Approach

No automated test suite exists. Verification is manual via Chrome DevTools MCP (per `CLAUDE.md`).

1. **Empty-state walkthrough** — clear localStorage. Pill reads `+ Add recipients`, strip hidden, click pill enters manage mode for new client.
2. **The bug repro path** — populate localStorage with the existing 3-client state (AWC, New Jersey, untitled empty). Open tool, click `+ New Client` in manage mode. Verify rail shows all clients, the new one is active, previous clients are visible and one click away.
3. **Wrong-client scenario** — switch active client via pill popover. Verify pill, strip, and section badge update simultaneously. Verify email body preview did NOT change (body and recipients are independent).
4. **Manage mode dirty guard** — edit a contact name, click Done without Save. Verify inline confirm appears with Save & Close / Discard options.
5. **Keyboard-only run** — complete a full Posted Files email flow without mouse: Tab through form, expand Recipients with Space, navigate manage mode rail with arrows, copy email with Ctrl+Enter.
6. **Both tabs** — verify pill and strip render identically on Posted Files and Recon Report.
7. **prefers-reduced-motion** — verify popover open and rail transitions respect the reduced-motion media query.
8. **localStorage migration** — existing users have `recipientsState` saved. Load with `expanded` absent; verify defaults applied without error.

## §10. Out of Scope (Tracked Separately)

- Twin segmented controls differentiation (Task #1)
- Email-format validation in manage mode (Task #4)
- Font-size ladder tightening (Task #5)
- DESIGN.md self-violations: header gradient, toast radius, em dash (Task #6)
- Minor observations: To/CC button parity in collapsed header is *resolved by this spec* in §2; iPad portrait preview height, `--radius` vs `--radius-sm` split remain in Task #9
- Final polish pass (Task #11)
