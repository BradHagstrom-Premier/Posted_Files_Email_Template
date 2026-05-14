# To/CC Recipients — Design Spec

**Date:** 2026-05-14
**Author:** Brad Hagstrom (with Claude)
**Status:** Approved
**Builds on:** [2026-05-13-client-email-list-design.md](2026-05-13-client-email-list-design.md)

## Problem

Some client recipients belong on the **To** line of an Outlook email, others on **CC**. The current Recipients section (V6) treats every checked contact identically and produces a single semicolon-separated string for paste into one field. Consultants currently sort the To/CC split manually each time, in Outlook, after pasting.

## Goal

Extend the Recipients section so each contact carries a To/CC role, and the copy flow produces two strings — one for Outlook's To field, one for the CC field. Persistent defaults set in Manage; per-send overrides available in the expanded view.

## Non-goals

- **BCC support.** Same pattern would extend to a third role; not in this iteration.
- **Persisting per-send overrides across reloads.** Overrides are intentionally session-only — they represent a per-email judgment call, not a setting.
- **Visual "overridden" marker on the pill.** The pill shows the current effective role with no history annotation, to keep the row scannable.
- **Touching the email output.** Recipients remain UI-only; nothing about To/CC appears in the generated email HTML.

## Decisions

| Decision | Choice |
|---|---|
| Role model | Persistent default per contact + per-send override |
| Override UI (expanded) | Click the role pill to flip To↔CC |
| Manage UI | Same role pill, click toggles persistent default |
| Copy flow | Two side-by-side buttons: `Copy To` / `Copy CC` |
| Collapsed quick-copy | Two quick buttons, mirroring expanded |
| Default role for new contact | `to` |
| Migration for existing data | All existing contacts default to `role: 'to'` |
| Override state storage | Separate `recipientsState.overrides` map (not persisted) |

## Data model

### Persistent — `contact` shape (extended)

```js
{
  name: "Jane Smith",
  email: "jane@awc.com",
  checked: true,
  role: "to"   // NEW: 'to' | 'cc'
}
```

Persisted to `localStorage['definian_recipients']` via the existing `saveRecipientsState()`. No change to save logic needed — `role` flows through naturally because save dumps the full `clients` array.

### Transient — `recipientsState.overrides`

```js
recipientsState.overrides = {}   // { [contactIndex: number]: 'to' | 'cc' }
```

- Scoped to the **currently active client**. Indexes refer to `clients[activeClientIndex].contacts`.
- **Never persisted.** `saveRecipientsState()` explicitly picks `activeClientIndex` and `clients` only — `overrides` is naturally excluded.
- **Cleared when:**
  - The active client changes (dropdown change handler).
  - Manage mode is exited via Save (contact order/membership may have shifted).
  - Page loads (state starts fresh anyway).
- **Not cleared when:** A checkbox toggles, a pill is clicked, or Manage mode is exited via Done with no save (overrides should survive incidental clicks).

### Effective role

```js
function effectiveRole(idx) {
  const client = recipientsState.clients[recipientsState.activeClientIndex];
  return recipientsState.overrides[idx] ?? client.contacts[idx].role ?? 'to';
}
```

The trailing `?? 'to'` is a defense-in-depth fallback in case migration missed a contact.

## Migration

In `loadRecipientsState()`, after the existing `JSON.parse` and `recipientsState.clients = saved.clients || []` line, walk every contact and fill in a missing `role`:

```js
recipientsState.clients.forEach(cl => {
  (cl.contacts || []).forEach(c => {
    if (!c.role) c.role = 'to';
  });
});
```

Idempotent — running it on already-migrated data is a no-op. No version flag needed; absence of `role` is the migration signal.

## Rendering — Expanded view

### Contact row

Layout adds a clickable role pill between the contact name and the email:

```
☑  Jane Smith   [To]  jane@awc.com
☑  Bob Lee      [CC]  bob@awc.com
☑  Maria Cruz   [CC]  maria@awc.com
```

- Pill is **always rendered**, regardless of `checked` state, so users see defaults before checking.
- Pill HTML: `<button type="button" class="role-pill role-pill--to" data-contact-index="${i}">To</button>` (class swap between `--to` and `--cc`).
- Pill stops propagation on click — toggling role must not also toggle the checkbox.
- Pill styling (uses existing CSS variables from `:root` at `index.html:19`):
  - `.role-pill--to`: `background: var(--blue);` (`#0D2C71`), `color: #ffffff;`.
  - `.role-pill--cc`: `background: var(--blue-muted);` (`rgba(13, 44, 113, 0.08)`), `color: var(--text-secondary);` (`#6b7280`).
  - Shared shape: small, rounded (`border-radius: var(--radius-sm)` = 5px), tight padding (~2px 8px), DM Sans (inherits), ~11px font, `cursor: pointer`, no border, `transition: var(--transition)` on background.

### Copy buttons

Replace the single `Copy Recipients` button with two:

```
[Copy To]  [Copy CC]                          Manage
```

- `Copy To`: filters checked contacts whose effective role is `'to'`, joins emails with `; `, writes to clipboard. Toast: `"To addresses copied!"`.
- `Copy CC`: same for `'cc'`. Toast: `"CC addresses copied!"`.
- Each is independently disabled when its filtered list is empty.
- `Copy To` keeps the existing primary button styling (`.btn-copy-recipients`). `Copy CC` gets a secondary variant (`.btn-copy-recipients--secondary`): white background, `1px solid var(--blue)` border, `color: var(--blue)` text. Hover swaps to `background: var(--blue-muted)`. Disabled state matches the primary's disabled treatment (existing `:disabled` rule at `index.html:780`).

### Badge

Header badge stays as `<Client> · <total checked count>`. No split into `2 To · 1 CC` — the breakdown is already visible in the two buttons.

## Rendering — Collapsed header

The collapsed header's current single quick-copy button becomes two:

```
Recipients   AWC · 3       [Copy To ▸] [Copy CC ▸]   ▼
```

- Each quick button mirrors its expanded counterpart's enabled/disabled state and filter behavior.
- Both stop event propagation so clicking them doesn't also toggle the header's expand/collapse.

## Rendering — Manage mode

Each manage contact row adds a pill between the email input and the delete button:

```
Manage Clients                            ← Done

Client name
[ AWC                                          ]

Contacts
[ Jane Smith ] [ jane@awc.com  ] [To]  ×
[ Bob Lee    ] [ bob@awc.com   ] [CC]  ×
[ Maria Cruz ] [ maria@awc.co  ] [CC]  ×
+ Add contact

[Save]   [+ New Client]
```

- Clicking the pill in Manage flips the **persistent default** for that row — it's an unsaved edit, applied to `contact.role` only when Save is clicked, the same way name/email edits are.
- New rows added via `+ Add contact` start with `role: 'to'`.
- On Save, the rebuilt `newContacts` array (existing code around `index.html:1838`) carries `role` through:

  ```js
  const newContacts = Array.from(rows).map((row, i) => {
    const nameVal = row.querySelector('.manage-contact-name').value.trim();
    const emailVal = row.querySelector('.manage-contact-email').value.trim();
    const roleVal = row.querySelector('.manage-contact-role').dataset.role; // 'to' | 'cc'
    return {
      name: nameVal,
      email: emailVal,
      checked: oldContacts[i] !== undefined ? oldContacts[i].checked : true,
      role: roleVal || 'to'
    };
  }).filter(c => c.name || c.email);
  ```

- On Save: `recipientsState.overrides = {}` is cleared — contact indexes may have shifted after add/delete/reorder.

## Event wiring

Additions to `wireRecipientsEvents()`:

1. **Expanded-view pill clicks.** Delegated listener on `.recipients-contact-list`, looking for `[data-contact-index]` on a `.role-pill`. Toggles `recipientsState.overrides[idx]` between `'to'` and `'cc'` (computed from `effectiveRole`), then re-renders. `e.stopPropagation()` so the checkbox doesn't fire.
2. **Collapsed quick-copy buttons.** `recipientsCopyQuickTo` and `recipientsCopyQuickCc` IDs, each calling `copyRecipients('to')` or `copyRecipients('cc')` with `e.stopPropagation()`.
3. **Expanded copy buttons.** `recipientsCopyTo` and `recipientsCopyCc` IDs, each calling the same function.
4. **Manage pill clicks.** Delegated listener on `#manageContactRows`, toggles `data-role` attribute on the pill element (and swaps CSS class). Read at Save time.
5. **Client select change.** Existing handler gains one line: `recipientsState.overrides = {};` before `renderRecipients()`.
6. **Manage Save.** After successful save, `recipientsState.overrides = {};`.

## Copy function

Refactor `copyRecipients()` to take a role argument:

```js
function copyRecipients(role) {
  const client = recipientsState.clients[recipientsState.activeClientIndex];
  if (!client) return;
  const line = client.contacts
    .map((c, i) => ({ c, i }))
    .filter(({ c, i }) => c.checked && effectiveRole(i) === role)
    .map(({ c }) => c.email)
    .join('; ');
  if (!line) return;
  navigator.clipboard.writeText(line).then(() => {
    showToast(role === 'to' ? 'To addresses copied!' : 'CC addresses copied!');
  }).catch(() => {
    showToast('Copy failed — try again');
  });
}
```

## Edge cases

| Case | Behavior |
|---|---|
| No clients yet | Unchanged — existing empty state. |
| All checked are To | `Copy CC` disabled (expanded and quick). |
| All checked are CC | `Copy To` disabled (expanded and quick). |
| Nothing checked | Both buttons disabled, mirroring today's single-button disabled. |
| Override set, then user unchecks then re-checks contact | Override survives — keyed by index, not by checked state. |
| Override set on contact N, then user enters Manage and reorders/deletes contacts | `overrides = {}` cleared on Save, so no stale index references. |
| Override set, then user clicks Manage but exits via Done (no save) | Override survives — only Save clears. |
| Override set, then user switches active client via dropdown | Override cleared — prevents cross-client leakage. |

## Files touched

- `index.html` — only file. CSS additions for `.role-pill`, `.role-pill--to`, `.role-pill--cc`, `.btn-copy-recipients--secondary`. JS additions in the Recipients section (≈ lines 1560–1925).

## Testing (manual, via Chrome DevTools MCP)

1. **Migration.** With existing V6 localStorage data (contacts have no `role`), reload page. Inspect: every contact has `role: 'to'` after `loadRecipientsState()`. Saving doesn't write redundant data.
2. **Persistent default in Manage.** Open Manage on an existing client. Flip a contact's pill from To to CC. Click Save. Reload page. Verify pill still shows CC and localStorage reflects `role: 'cc'`.
3. **Per-send override in expanded view.** With three To contacts, click the pill on the middle one — flips to CC. Re-open / collapse / re-open the section: override survives. Reload page: override is gone (transient). Persistent role still To.
4. **Copy To / Copy CC.** With 2 checked To + 1 checked CC, click Copy To → clipboard contains 2 emails joined with `; `. Click Copy CC → clipboard contains the 1 email. Toasts match.
5. **Disabled states.** Uncheck all To contacts → Copy To disabled (expanded and quick header). Uncheck all → both disabled.
6. **Override clears on client switch.** Set override on Contact A in Client 1. Switch dropdown to Client 2, then back to Client 1. Override is gone (pill back to default).
7. **Override clears on Save.** Set override, click Manage, click Save (no edits). Override is gone.
8. **Override survives Done (no save).** Set override, click Manage, click Done (with no edits). Override survives.
9. **Pill click doesn't toggle checkbox.** Click pill on a checked contact → role flips, checkbox stays checked.
10. **Quick-copy buttons don't toggle expand/collapse.** Click `Copy To ▸` in collapsed header → copies, section stays collapsed.
11. **Outlook end-to-end.** In a real Outlook compose window: paste Copy To result into To field, paste Copy CC result into CC field. Both resolve to recipients without manual fixup.

## Open questions

None.
