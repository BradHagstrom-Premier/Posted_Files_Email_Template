# To/CC Recipients Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-contact To/CC role assignments to the Recipients section, with persistent defaults set in Manage and per-send overrides in the expanded view. Produce two clipboard copy actions — one for Outlook's To field, one for the CC field.

**Architecture:** Single-file browser tool (`index.html`). All changes are additive edits inside the existing Recipients section (CSS ~620–880, JS ~1560–1925). Persistent role stored on each contact (`contact.role`); transient per-send override stored on `recipientsState.overrides` (an in-memory map, never written to localStorage). One-pass idempotent migration in `loadRecipientsState()` backfills `role: 'to'` on existing contacts.

**Tech Stack:** Vanilla JavaScript (browser), HTML, CSS using existing custom properties from `:root` (`--blue`, `--blue-muted`, `--text-secondary`, `--radius-sm`, `--transition`). No build, no dependencies. Manual verification via Chrome DevTools MCP per CLAUDE.md (no automated test suite exists).

**Spec:** [docs/superpowers/specs/2026-05-14-to-cc-recipients-design.md](../specs/2026-05-14-to-cc-recipients-design.md)

**Verification model:** This codebase has no test runner. Each task's verification is a sequence of browser interactions plus state inspection via the DevTools console or Chrome DevTools MCP `evaluate_script`. The engineer should open `index.html` in Chrome and have DevTools console open throughout.

---

## File Structure

All changes are made to a **single file**: `index.html`. Sections affected:

| Concern | Approx. lines | Change type |
|---|---|---|
| `:root` CSS variables | 19–37 | none (reuse existing) |
| Recipients CSS block | 632–880 | add `.role-pill`, `.role-pill--to`, `.role-pill--cc`, `.btn-copy-recipients--secondary` rules |
| `recipientsState` declaration | 1566–1571 | add `overrides: {}` field |
| `loadRecipientsState()` | 1573–1588 | add migration loop after parse |
| `renderRecipients()` — collapsed header | 1615–1623, 1675–1682 | replace single quick-copy with two |
| `renderRecipients()` — expanded contact rows | 1647–1654 | add pill to each row |
| `renderRecipients()` — expanded actions row | 1667–1670 | replace single button with two |
| `renderManageMode()` — contact row template | 1692–1697 | add pill |
| `renderManageMode()` — add-contact template | 1866–1870 | add pill to new rows |
| `wireRecipientsEvents()` — new handlers | 1726–1908 | pill clicks (expanded + manage), copy buttons (×2), quick copies (×2), client switch override-clear, save override-clear |
| `copyRecipients()` | 1910–1923 | refactor to take `role` parameter, filter by effective role |
| New helper: `effectiveRole(idx)` | new, near 1597 | added |

---

## Pre-flight

- [ ] **Open `index.html` in Chrome.** Open DevTools console. Confirm the Recipients section renders (collapsed) at the bottom of the form panel.
- [ ] **Snapshot current localStorage.** In console: `localStorage.getItem('definian_recipients')`. Save the output to a scratch note. You'll need it later to verify migration ran cleanly on real data. If empty, create a test client + 2 contacts via the existing UI before continuing.
- [ ] **Confirm baseline.** In console: `recipientsState` — verify it has `clients`, `activeClientIndex`, `expanded`, `manageMode`. Verify each contact has `name`, `email`, `checked` (no `role` field yet).

---

### Task 1: Data model — add `overrides` field, migration, and `effectiveRole()` helper

**Files:**
- Modify: `index.html:1566-1571` (`recipientsState` declaration)
- Modify: `index.html:1573-1588` (`loadRecipientsState()`)
- Modify: `index.html:1597` (insert `effectiveRole()` helper after `loadRecipientsState()` call)

- [ ] **Step 1: Add `overrides` field to `recipientsState`**

Replace:

```js
    let recipientsState = {
        activeClientIndex: 0,
        expanded: false,
        manageMode: false,
        clients: []
    };
```

With:

```js
    let recipientsState = {
        activeClientIndex: 0,
        expanded: false,
        manageMode: false,
        clients: [],
        overrides: {}   // { [contactIndex]: 'to' | 'cc' } — transient, never persisted
    };
```

- [ ] **Step 2: Add migration loop in `loadRecipientsState()`**

Inside `loadRecipientsState()`, after the line `recipientsState.activeClientIndex = saved.activeClientIndex || 0;` and the `if (recipientsState.activeClientIndex >= ...)` block, add the migration loop. The full function becomes:

```js
    function loadRecipientsState() {
        try {
            const raw = localStorage.getItem(RECIPIENTS_KEY);
            if (raw) {
                const saved = JSON.parse(raw);
                recipientsState.clients = saved.clients || [];
                recipientsState.activeClientIndex = saved.activeClientIndex || 0;
                if (recipientsState.activeClientIndex >= recipientsState.clients.length) {
                    recipientsState.activeClientIndex = 0;
                }
                // Migration: backfill role='to' on any contact missing role
                recipientsState.clients.forEach(cl => {
                    (cl.contacts || []).forEach(c => {
                        if (!c.role) c.role = 'to';
                    });
                });
            }
        } catch {
            recipientsState.clients = [];
            recipientsState.activeClientIndex = 0;
        }
    }
```

- [ ] **Step 3: Add `effectiveRole()` helper**

Insert immediately after the existing `loadRecipientsState();` call line (around line 1597), before the `/* ═══...  Recipients — Render ═══...` comment block:

```js
    function effectiveRole(idx) {
        const client = recipientsState.clients[recipientsState.activeClientIndex];
        if (!client || !client.contacts[idx]) return 'to';
        return recipientsState.overrides[idx] ?? client.contacts[idx].role ?? 'to';
    }
```

- [ ] **Step 4: Verify migration runs**

Reload `index.html` in Chrome. In console:

```js
recipientsState.clients[0].contacts.forEach(c => console.log(c.name, c.role));
```

Expected: every contact prints with `'to'` as its role (or its previously-set role if you re-test after Task 9).

Run: `localStorage.getItem('definian_recipients')` — `role` is **not** yet in localStorage (migration is in-memory; the next save will persist it). That's expected.

Run: `recipientsState.overrides` — Expected: `{}`.

Run: `effectiveRole(0)` — Expected: `'to'`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(recipients): add role field, migration, and effectiveRole helper"
```

---

### Task 2: CSS — role pill and secondary copy button

**Files:**
- Modify: `index.html:780` (immediately after the existing `.btn-copy-recipients:disabled` rule, before the `/* Manage mode */` comment)

- [ ] **Step 1: Add pill and secondary-button styles**

Insert this CSS block immediately after the `.btn-copy-recipients:disabled { ... }` rule (around line 783), before the `/* Manage mode */` comment:

```css
        .role-pill {
            display: inline-block;
            padding: 2px 8px;
            border-radius: var(--radius-sm);
            font-size: 11px;
            font-weight: 600;
            font-family: inherit;
            border: none;
            cursor: pointer;
            transition: background var(--transition);
            user-select: none;
            flex-shrink: 0;
        }

        .role-pill--to {
            background: var(--blue);
            color: #ffffff;
        }

        .role-pill--to:hover {
            background: var(--blue-light);
        }

        .role-pill--cc {
            background: var(--blue-muted);
            color: var(--text-secondary);
        }

        .role-pill--cc:hover {
            background: rgba(13, 44, 113, 0.14);
        }

        .btn-copy-recipients--secondary {
            background: #ffffff;
            color: var(--blue);
            border: 1px solid var(--blue);
        }

        .btn-copy-recipients--secondary:hover:not(:disabled) {
            background: var(--blue-muted);
        }
```

- [ ] **Step 2: Verify styles parse and don't break the page**

Reload `index.html`. Confirm: no rendering changes yet (no element uses these classes), no console errors.

In console: `getComputedStyle(document.createElement('button'))` won't help here. Instead, briefly add a test element to confirm:

```js
const t = document.createElement('button');
t.className = 'role-pill role-pill--to';
t.textContent = 'To';
document.body.appendChild(t);
```

Expected: a small blue pill with "To" appears at the bottom-left of the page. Remove with `t.remove();`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(recipients): add role pill and secondary copy button styles"
```

---

### Task 3: Refactor `copyRecipients()` to accept a role parameter

**Files:**
- Modify: `index.html:1910-1923` (`copyRecipients` function)

- [ ] **Step 1: Replace `copyRecipients()` body**

Replace the entire function:

```js
    function copyRecipients() {
        const client = recipientsState.clients[recipientsState.activeClientIndex];
        if (!client) return;
        const line = client.contacts
            .filter(c => c.checked)
            .map(c => c.email)
            .join('; ');
        if (!line) return;
        navigator.clipboard.writeText(line).then(() => {
            showToast('Recipients copied!');
        }).catch(() => {
            showToast('Copy failed — try again');
        });
    }
```

With:

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

- [ ] **Step 2: Verify the refactor compiles**

Reload `index.html`. Existing UI still renders. Existing buttons that call `copyRecipients()` with no argument will now produce no output (role is `undefined`, no contact matches) — that's expected; they'll be replaced in Tasks 5 and 6.

In console (with at least one checked contact, all defaulting to `role: 'to'`):

```js
copyRecipients('to');
```

Expected: toast "To addresses copied!" appears. Paste into a scratch text area or run `navigator.clipboard.readText().then(s => console.log(JSON.stringify(s)))` — expected: a semicolon-separated list of checked emails.

```js
copyRecipients('cc');
```

Expected: no toast (no contacts have role `'cc'` yet).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "refactor(recipients): copyRecipients takes role parameter, filters by effective role"
```

---

### Task 4: Expanded view — role pill on each contact row + click handler

**Files:**
- Modify: `index.html:1647-1654` (contact row template inside `renderRecipients()`)
- Modify: `index.html:1726-1908` (`wireRecipientsEvents()` — add pill click handler)

- [ ] **Step 1: Update the contact row template to include the pill**

Find the existing contact row template inside the `else` branch (around line 1646):

```js
                    contactsHTML = `<div class="recipients-contact-list">` +
                        client.contacts.map((c, i) => `
                        <div class="recipients-contact-row">
                            <input type="checkbox" ${c.checked ? 'checked' : ''} data-contact-index="${i}"
                                   style="accent-color:var(--blue);width:14px;height:14px;flex-shrink:0;cursor:pointer;"
                                   aria-label="${escapeHtml(c.name)}">
                            <span class="recipients-contact-name">${escapeHtml(c.name)}</span>
                            <span class="recipients-contact-email">${escapeHtml(c.email)}</span>
                        </div>`).join('') +
                    `</div>`;
```

Replace with:

```js
                    contactsHTML = `<div class="recipients-contact-list">` +
                        client.contacts.map((c, i) => {
                            const role = effectiveRole(i);
                            return `
                        <div class="recipients-contact-row">
                            <input type="checkbox" ${c.checked ? 'checked' : ''} data-contact-index="${i}"
                                   style="accent-color:var(--blue);width:14px;height:14px;flex-shrink:0;cursor:pointer;"
                                   aria-label="${escapeHtml(c.name)}">
                            <span class="recipients-contact-name">${escapeHtml(c.name)}</span>
                            <button type="button" class="role-pill role-pill--${role}" data-role-index="${i}" title="Click to flip To/CC for this email">${role === 'to' ? 'To' : 'CC'}</button>
                            <span class="recipients-contact-email">${escapeHtml(c.email)}</span>
                        </div>`;
                        }).join('') +
                    `</div>`;
```

- [ ] **Step 2: Add pill click handler in `wireRecipientsEvents()`**

Find the existing contact-list change handler in `wireRecipientsEvents()` (around line 1757):

```js
        // Contact checkboxes
        const contactList = document.querySelector('.recipients-contact-list');
        if (contactList) {
            contactList.addEventListener('change', (e) => {
                ...
            });
        }
```

Immediately after that block (before the `// Copy Recipients button (expanded state)` comment), insert:

```js
        // Role pill click — toggle per-send override (expanded view)
        if (contactList) {
            contactList.addEventListener('click', (e) => {
                const pill = e.target.closest('.role-pill[data-role-index]');
                if (!pill) return;
                e.stopPropagation();
                const idx = parseInt(pill.dataset.roleIndex, 10);
                const current = effectiveRole(idx);
                recipientsState.overrides[idx] = current === 'to' ? 'cc' : 'to';
                renderRecipients();
            });
        }
```

- [ ] **Step 3: Verify pill rendering and toggle behavior**

Reload `index.html`. Open the Recipients section (expand it). With at least one client + one contact:

1. **Pill renders.** Each contact row shows a `To` pill (blue, white text) between the name and the email.
2. **Click flips role.** Click the pill — it changes to `CC` (faint blue background, grey text). Re-render preserves the rest of the row.
3. **Override is transient.** In console: `recipientsState.overrides` — shows e.g. `{ 0: 'cc' }`. `localStorage.getItem('definian_recipients')` — does **not** contain an `overrides` key.
4. **Persistent role unchanged.** `recipientsState.clients[0].contacts[0].role` — still `'to'`.
5. **Click doesn't toggle checkbox.** Click the pill on a row whose checkbox is checked. Checkbox stays checked.
6. **Click toggles back.** Click the pill again — flips to `To`. `recipientsState.overrides[0]` is now `'to'`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(recipients): role pill on contact rows with per-send override toggle"
```

---

### Task 5: Expanded view — replace single Copy button with Copy To + Copy CC

**Files:**
- Modify: `index.html:1657-1670` (expanded actions block inside `renderRecipients()`)
- Modify: `index.html:1772-1776` (existing copy button handler in `wireRecipientsEvents()`)

- [ ] **Step 1: Replace the actions HTML with two buttons**

Find the existing actions block inside the `else` branch of `renderRecipients()` (around line 1657):

```js
                const copyDisabled = checkedContacts.length === 0 ? 'disabled' : '';
                const copyLabel = checkedContacts.length === 0 ? 'No contacts selected' : 'Copy Recipients';

                bodyHTML = `
                <div class="recipients-body">
                    <label class="recipients-client-label" for="recipientsClientSelect">Client</label>
                    <select class="form-select" id="recipientsClientSelect">${options}</select>
                    ${contactsHTML}
                    <button type="button" class="recipients-add-link" id="recipientsAddContact" style="margin-bottom:10px;display:inline-block;">+ Add contact</button>
                    <div class="recipients-actions">
                        <button type="button" class="btn-copy-recipients" id="recipientsCopy" ${copyDisabled}>${copyLabel}</button>
                        <button type="button" class="recipients-manage-link" id="recipientsManageBtn">Manage</button>
                    </div>
                </div>`;
```

Replace with:

```js
                const toCount = client.contacts.filter((c, i) => c.checked && effectiveRole(i) === 'to').length;
                const ccCount = client.contacts.filter((c, i) => c.checked && effectiveRole(i) === 'cc').length;
                const toDisabled = toCount === 0 ? 'disabled' : '';
                const ccDisabled = ccCount === 0 ? 'disabled' : '';

                bodyHTML = `
                <div class="recipients-body">
                    <label class="recipients-client-label" for="recipientsClientSelect">Client</label>
                    <select class="form-select" id="recipientsClientSelect">${options}</select>
                    ${contactsHTML}
                    <button type="button" class="recipients-add-link" id="recipientsAddContact" style="margin-bottom:10px;display:inline-block;">+ Add contact</button>
                    <div class="recipients-actions">
                        <button type="button" class="btn-copy-recipients" id="recipientsCopyTo" ${toDisabled} title="${toCount === 0 ? 'No To recipients selected' : ''}">Copy To</button>
                        <button type="button" class="btn-copy-recipients btn-copy-recipients--secondary" id="recipientsCopyCc" ${ccDisabled} title="${ccCount === 0 ? 'No CC recipients selected' : ''}">Copy CC</button>
                        <button type="button" class="recipients-manage-link" id="recipientsManageBtn">Manage</button>
                    </div>
                </div>`;
```

- [ ] **Step 2: Replace the existing copy button handler with two**

Find the existing handler in `wireRecipientsEvents()` (around line 1772):

```js
        // Copy Recipients button (expanded state)
        const copyBtn = document.getElementById('recipientsCopy');
        if (copyBtn) {
            copyBtn.addEventListener('click', copyRecipients);
        }
```

Replace with:

```js
        // Copy To / Copy CC buttons (expanded state)
        const copyToBtn = document.getElementById('recipientsCopyTo');
        if (copyToBtn) {
            copyToBtn.addEventListener('click', () => copyRecipients('to'));
        }
        const copyCcBtn = document.getElementById('recipientsCopyCc');
        if (copyCcBtn) {
            copyCcBtn.addEventListener('click', () => copyRecipients('cc'));
        }
```

- [ ] **Step 3: Verify two-button copy flow**

Reload `index.html`. Expand the Recipients section. With at least two checked contacts:

1. **Both buttons appear.** Copy To (filled blue), Copy CC (white with blue border).
2. **All-To case.** With all checked contacts at role `'to'`: Copy To is enabled, Copy CC is disabled (and shows tooltip on hover).
3. **Mixed case.** Flip one contact's pill to CC. Both buttons are now enabled.
4. **Copy To behavior.** Click Copy To: toast "To addresses copied!" Verify clipboard via `navigator.clipboard.readText().then(s => console.log(JSON.stringify(s)))` — semicolon-joined list of only the To-role checked emails.
5. **Copy CC behavior.** Click Copy CC: toast "CC addresses copied!" Verify clipboard contains only the CC-role checked emails.
6. **All-CC case.** Flip every checked contact to CC: Copy To disabled, Copy CC enabled.
7. **Nothing checked.** Uncheck everyone: both disabled.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(recipients): split Copy Recipients into Copy To and Copy CC buttons"
```

---

### Task 6: Collapsed header — two quick-copy buttons

**Files:**
- Modify: `index.html:1615-1623` (collapsed header build in `renderRecipients()`)
- Modify: `index.html:1737-1744` (existing quick-copy handler in `wireRecipientsEvents()`)

- [ ] **Step 1: Replace the collapsed quick-copy build**

Find the existing collapsed header build inside `renderRecipients()` (around line 1615):

```js
        // Build collapsed header (always shown)
        let badgeHTML = '';
        let quickCopyHTML = '';
        if (hasClients && client) {
            badgeHTML = `<span class="recipients-badge">${escapeHtml(client.name)} &middot; ${checkedContacts.length}</span>`;
            quickCopyHTML = `<button type="button" class="btn-copy-quick" id="recipientsCopyQuick">Copy &#9658;</button>`;
        }
```

Replace with:

```js
        // Build collapsed header (always shown)
        let badgeHTML = '';
        let quickCopyHTML = '';
        if (hasClients && client) {
            badgeHTML = `<span class="recipients-badge">${escapeHtml(client.name)} &middot; ${checkedContacts.length}</span>`;
            const quickToCount = client.contacts.filter((c, i) => c.checked && effectiveRole(i) === 'to').length;
            const quickCcCount = client.contacts.filter((c, i) => c.checked && effectiveRole(i) === 'cc').length;
            const quickToDisabled = quickToCount === 0 ? 'disabled' : '';
            const quickCcDisabled = quickCcCount === 0 ? 'disabled' : '';
            quickCopyHTML = `
                <button type="button" class="btn-copy-quick" id="recipientsCopyQuickTo" ${quickToDisabled}>Copy To &#9658;</button>
                <button type="button" class="btn-copy-quick" id="recipientsCopyQuickCc" ${quickCcDisabled}>Copy CC &#9658;</button>`;
        }
```

- [ ] **Step 2: Replace the existing quick-copy handler with two**

Find the existing handler (around line 1737):

```js
        // Quick copy (collapsed state)
        const quickCopy = document.getElementById('recipientsCopyQuick');
        if (quickCopy) {
            quickCopy.addEventListener('click', (e) => {
                e.stopPropagation();
                copyRecipients();
            });
        }
```

Replace with:

```js
        // Quick copy buttons (collapsed state)
        const quickCopyTo = document.getElementById('recipientsCopyQuickTo');
        if (quickCopyTo) {
            quickCopyTo.addEventListener('click', (e) => {
                e.stopPropagation();
                copyRecipients('to');
            });
        }
        const quickCopyCc = document.getElementById('recipientsCopyQuickCc');
        if (quickCopyCc) {
            quickCopyCc.addEventListener('click', (e) => {
                e.stopPropagation();
                copyRecipients('cc');
            });
        }
```

- [ ] **Step 3: Verify quick-copy buttons**

Reload `index.html`. Collapse the Recipients section. With at least one checked To contact and one checked CC contact (use the pill in expanded view to set CC, then collapse):

1. **Both buttons appear in header.** `Copy To ▸` and `Copy CC ▸` side by side, before the chevron.
2. **Click Copy To ▸ in collapsed state.** Toast appears; section does **not** expand. Clipboard contains To emails.
3. **Click Copy CC ▸ in collapsed state.** Toast appears; section does **not** expand. Clipboard contains CC emails.
4. **Disabled when empty.** Uncheck all To: `Copy To ▸` becomes disabled. Uncheck all: both disabled.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(recipients): two quick-copy buttons in collapsed header"
```

---

### Task 7: Clear overrides when active client changes

**Files:**
- Modify: `index.html:1746-1754` (client select change handler in `wireRecipientsEvents()`)

- [ ] **Step 1: Add override clear to the client switch handler**

Find the existing handler (around line 1746):

```js
        // Client select dropdown
        const clientSelect = document.getElementById('recipientsClientSelect');
        if (clientSelect) {
            clientSelect.addEventListener('change', () => {
                recipientsState.activeClientIndex = parseInt(clientSelect.value, 10);
                saveRecipientsState();
                renderRecipients();
            });
        }
```

Replace with:

```js
        // Client select dropdown
        const clientSelect = document.getElementById('recipientsClientSelect');
        if (clientSelect) {
            clientSelect.addEventListener('change', () => {
                recipientsState.activeClientIndex = parseInt(clientSelect.value, 10);
                recipientsState.overrides = {};   // overrides are scoped to active client
                saveRecipientsState();
                renderRecipients();
            });
        }
```

- [ ] **Step 2: Verify override clears on client switch**

Reload `index.html`. Requires at least two clients with at least one contact each.

1. **Set an override.** Expand, click a contact's pill to flip to CC. `recipientsState.overrides` — shows e.g. `{ 0: 'cc' }`.
2. **Switch client via dropdown.** `recipientsState.overrides` — now `{}`.
3. **Switch back.** Override is still gone (intentional); the pill shows the persistent default again.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(recipients): clear overrides when active client changes"
```

---

### Task 8: Manage mode — pill on each contact row + click handler

**Files:**
- Modify: `index.html:1692-1697` (manage row template inside `renderManageMode()`)
- Modify: `index.html:1862-1872` (`manageAddContact` handler — template for new rows)
- Modify: `index.html:1857` (insert new pill click handler in `wireRecipientsEvents()` before the existing add-contact block)

- [ ] **Step 1: Update the manage row template to include the pill**

Find the existing manage row template inside `renderManageMode()` (around line 1690):

```js
        let contactRowsHTML = '';
        if (client) {
            contactRowsHTML = client.contacts.map((c, i) => `
                <div class="recipients-manage-contact-row" data-row="${i}">
                    <input type="text" class="form-input manage-contact-name" value="${escapeHtml(c.name)}" placeholder="Name" style="flex:2;">
                    <input type="text" class="form-input manage-contact-email" value="${escapeHtml(c.email)}" placeholder="email@client.com" style="flex:3;">
                    <button type="button" class="btn-delete-contact" data-row="${i}" title="Remove contact" aria-label="Remove contact">&times;</button>
                </div>`).join('');
        }
```

Replace with:

```js
        let contactRowsHTML = '';
        if (client) {
            contactRowsHTML = client.contacts.map((c, i) => {
                const role = c.role || 'to';
                return `
                <div class="recipients-manage-contact-row" data-row="${i}">
                    <input type="text" class="form-input manage-contact-name" value="${escapeHtml(c.name)}" placeholder="Name" style="flex:2;">
                    <input type="text" class="form-input manage-contact-email" value="${escapeHtml(c.email)}" placeholder="email@client.com" style="flex:3;">
                    <button type="button" class="role-pill role-pill--${role}" data-role="${role}" title="Click to flip default To/CC">${role === 'to' ? 'To' : 'CC'}</button>
                    <button type="button" class="btn-delete-contact" data-row="${i}" title="Remove contact" aria-label="Remove contact">&times;</button>
                </div>`;
            }).join('');
        }
```

- [ ] **Step 2: Update the add-contact-row template to include the pill (default `to`)**

Find the existing `manageAddContact` handler (around line 1858):

```js
        // Manage mode — Add contact row
        const addContactBtn = document.getElementById('manageAddContact');
        if (addContactBtn) {
            addContactBtn.addEventListener('click', () => {
                const rows = document.getElementById('manageContactRows');
                const newRow = document.createElement('div');
                newRow.className = 'recipients-manage-contact-row';
                const rowIndex = rows.querySelectorAll('.recipients-manage-contact-row').length;
                newRow.dataset.row = rowIndex;
                newRow.innerHTML = `
                    <input type="text" class="form-input manage-contact-name" placeholder="Name" style="flex:2;">
                    <input type="text" class="form-input manage-contact-email" placeholder="email@client.com" style="flex:3;">
                    <button type="button" class="btn-delete-contact" data-row="${rowIndex}" title="Remove contact" aria-label="Remove contact">&times;</button>`;
                rows.appendChild(newRow);
                newRow.querySelector('.manage-contact-name').focus();
            });
        }
```

Replace with:

```js
        // Manage mode — Add contact row
        const addContactBtn = document.getElementById('manageAddContact');
        if (addContactBtn) {
            addContactBtn.addEventListener('click', () => {
                const rows = document.getElementById('manageContactRows');
                const newRow = document.createElement('div');
                newRow.className = 'recipients-manage-contact-row';
                const rowIndex = rows.querySelectorAll('.recipients-manage-contact-row').length;
                newRow.dataset.row = rowIndex;
                newRow.innerHTML = `
                    <input type="text" class="form-input manage-contact-name" placeholder="Name" style="flex:2;">
                    <input type="text" class="form-input manage-contact-email" placeholder="email@client.com" style="flex:3;">
                    <button type="button" class="role-pill role-pill--to" data-role="to" title="Click to flip default To/CC">To</button>
                    <button type="button" class="btn-delete-contact" data-row="${rowIndex}" title="Remove contact" aria-label="Remove contact">&times;</button>`;
                rows.appendChild(newRow);
                newRow.querySelector('.manage-contact-name').focus();
            });
        }
```

- [ ] **Step 3: Add manage pill click handler in `wireRecipientsEvents()`**

The existing `// Manage mode — Delete contact row (event delegation)` block (around line 1875) attaches to `#manageContactRows`. We'll add another delegated listener on the same element for pill clicks. Insert immediately after that delete handler block:

```js
        // Manage mode — Role pill click (toggle persistent default; unsaved until Save)
        if (contactRows) {
            contactRows.addEventListener('click', (e) => {
                const pill = e.target.closest('.role-pill[data-role]');
                if (!pill) return;
                const next = pill.dataset.role === 'to' ? 'cc' : 'to';
                pill.dataset.role = next;
                pill.classList.toggle('role-pill--to', next === 'to');
                pill.classList.toggle('role-pill--cc', next === 'cc');
                pill.textContent = next === 'to' ? 'To' : 'CC';
            });
        }
```

Note: `contactRows` is already declared on the line above (existing `const contactRows = document.getElementById('manageContactRows');`). Reuse the same `const`. Verify by reading the existing code — if `contactRows` was declared inside an `if` block scope, hoist the declaration to a single `const` shared by both listeners. Concretely the block becomes:

```js
        // Manage mode — Delete contact row + Role pill click (event delegation)
        const contactRows = document.getElementById('manageContactRows');
        if (contactRows) {
            contactRows.addEventListener('click', (e) => {
                // Delete contact row
                const btn = e.target.closest('.btn-delete-contact');
                if (btn) {
                    btn.closest('.recipients-manage-contact-row').remove();
                    return;
                }
                // Role pill toggle (persistent default; unsaved until Save)
                const pill = e.target.closest('.role-pill[data-role]');
                if (pill) {
                    const next = pill.dataset.role === 'to' ? 'cc' : 'to';
                    pill.dataset.role = next;
                    pill.classList.toggle('role-pill--to', next === 'to');
                    pill.classList.toggle('role-pill--cc', next === 'cc');
                    pill.textContent = next === 'to' ? 'To' : 'CC';
                }
            });
        }
```

This replaces the existing delete handler block at lines 1875–1883. Make sure to delete the old block.

- [ ] **Step 4: Verify Manage pill rendering and toggle**

Reload `index.html`. Open Recipients, expand, click Manage.

1. **Pill on each row.** Every existing contact row shows a pill (To or CC) between the email input and the delete button.
2. **Click flips visually.** Click a pill: text flips, color flips. `data-role` attribute reflects the new value (inspect element in DevTools).
3. **Add contact starts with To.** Click `+ Add contact`. New row appears with a `To` pill.
4. **Add → flip → Done (no save).** Add a contact, flip its pill to CC, type a name + email, click Done (not Save). Re-enter Manage. The new contact is **gone** (Done discarded the row) — this is existing behavior unchanged.
5. **No persistence yet.** Flip an existing row's pill, click Done (not Save). Re-enter Manage. Pill is back to its original value (persistent role unchanged until Save — that's wired up next task).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(recipients): role pill on manage rows with visual toggle"
```

---

### Task 9: Manage Save — persist role + clear overrides

**Files:**
- Modify: `index.html:1830-1855` (Manage Save handler in `wireRecipientsEvents()`)

- [ ] **Step 1: Update Save to read role from pill and clear overrides**

Find the existing Save handler (around line 1830):

```js
        // Manage mode — Save
        const saveBtn = document.getElementById('manageSaveBtn');
        if (saveBtn) {
            saveBtn.addEventListener('click', () => {
                const nameInput = document.getElementById('manageClientName');
                const rows = document.querySelectorAll('#manageContactRows .recipients-manage-contact-row');
                const client = recipientsState.clients[recipientsState.activeClientIndex];
                if (!client) return;

                const oldContacts = client.contacts;
                const newContacts = Array.from(rows).map((row, i) => {
                    const nameVal = row.querySelector('.manage-contact-name').value.trim();
                    const emailVal = row.querySelector('.manage-contact-email').value.trim();
                    return {
                        name: nameVal,
                        email: emailVal,
                        checked: oldContacts[i] !== undefined ? oldContacts[i].checked : true
                    };
                }).filter(c => c.name || c.email); // drop fully empty rows

                client.name = nameInput ? nameInput.value.trim() : client.name;
                client.contacts = newContacts;
                saveRecipientsState();
                recipientsState.manageMode = false;
                renderRecipients();
                showToast('Saved!');
            });
        }
```

Replace with:

```js
        // Manage mode — Save
        const saveBtn = document.getElementById('manageSaveBtn');
        if (saveBtn) {
            saveBtn.addEventListener('click', () => {
                const nameInput = document.getElementById('manageClientName');
                const rows = document.querySelectorAll('#manageContactRows .recipients-manage-contact-row');
                const client = recipientsState.clients[recipientsState.activeClientIndex];
                if (!client) return;

                const oldContacts = client.contacts;
                const newContacts = Array.from(rows).map((row, i) => {
                    const nameVal = row.querySelector('.manage-contact-name').value.trim();
                    const emailVal = row.querySelector('.manage-contact-email').value.trim();
                    const pill = row.querySelector('.role-pill[data-role]');
                    const roleVal = pill && pill.dataset.role === 'cc' ? 'cc' : 'to';
                    return {
                        name: nameVal,
                        email: emailVal,
                        checked: oldContacts[i] !== undefined ? oldContacts[i].checked : true,
                        role: roleVal
                    };
                }).filter(c => c.name || c.email); // drop fully empty rows

                client.name = nameInput ? nameInput.value.trim() : client.name;
                client.contacts = newContacts;
                recipientsState.overrides = {};   // contact indexes may have shifted
                saveRecipientsState();
                recipientsState.manageMode = false;
                renderRecipients();
                showToast('Saved!');
            });
        }
```

- [ ] **Step 2: Verify role persistence through Save**

Reload `index.html`. Open Recipients, expand, click Manage.

1. **Flip existing row, Save.** Flip a contact's pill from To to CC. Click Save. Section returns to expanded view. The contact's pill in the expanded view shows CC.
2. **Reload page.** `recipientsState.clients[0].contacts.find(c => c.email === '<that email>').role` — `'cc'`.
3. **localStorage contains role.** `localStorage.getItem('definian_recipients')` — JSON shows `"role":"cc"` on that contact.
4. **Override clears on Save.** Set a per-send override in expanded view (flip a pill there). `recipientsState.overrides` non-empty. Enter Manage, click Save with no changes. `recipientsState.overrides` is `{}`.
5. **Done (no save) preserves override.** Set per-send override, enter Manage, click Done. `recipientsState.overrides` unchanged.
6. **Add contact in Manage saves with role='to'.** In Manage, click `+ Add contact`, fill name + email, Save. Reload. New contact has `role: 'to'`.
7. **Add contact + flip to CC + Save.** Click `+ Add contact`, type name + email, flip pill to CC, Save. Reload. New contact has `role: 'cc'`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(recipients): save persists contact role and clears overrides"
```

---

### Task 10: End-to-end verification (Chrome DevTools MCP preferred per CLAUDE.md)

**Files:** none (verification only)

This task runs the full edge-case matrix from the spec against the live page. Do not skip — these checks together prove the feature works against every scenario from the spec.

- [ ] **Step 1: Set up a clean test client**

In DevTools console:

```js
localStorage.setItem('definian_recipients', JSON.stringify({
  activeClientIndex: 0,
  clients: [{
    name: 'AWC',
    contacts: [
      { name: 'Jane Smith', email: 'jane@awc.com', checked: true },
      { name: 'Bob Lee', email: 'bob@awc.com', checked: true },
      { name: 'Maria Cruz', email: 'maria@awc.com', checked: true }
    ]
  }, {
    name: 'Other Client',
    contacts: [
      { name: 'Alex Yan', email: 'alex@other.com', checked: true }
    ]
  }]
}));
location.reload();
```

This simulates pre-V7 data (no `role` field).

- [ ] **Step 2: Verify migration**

```js
recipientsState.clients.forEach(cl => cl.contacts.forEach(c => console.log(cl.name, c.name, c.role)));
```

Expected: every contact prints with `role: 'to'`. No errors.

- [ ] **Step 3: Verify Manage default change persists**

1. Expand Recipients → Manage.
2. Flip Bob Lee and Maria Cruz to CC. Click Save.
3. Expand → pills show Jane=To, Bob=CC, Maria=CC.
4. `location.reload()`. Same state — pills persist.

- [ ] **Step 4: Verify Copy To and Copy CC**

In the expanded view with the state from Step 3:

1. Click Copy To. Toast: "To addresses copied!" Verify: `navigator.clipboard.readText().then(s => console.log(s))` → `jane@awc.com`.
2. Click Copy CC. Toast: "CC addresses copied!" Clipboard: `bob@awc.com; maria@awc.com`.

- [ ] **Step 5: Verify per-send override**

1. Click Jane's pill in expanded view → flips to CC. `recipientsState.overrides[0]` is `'cc'`.
2. Click Copy CC. Clipboard: `jane@awc.com; bob@awc.com; maria@awc.com`.
3. Click Copy To. **No toast** (no To recipients). Both buttons reflect this — Copy To disabled.
4. Click Jane's pill again → flips back to To. `recipientsState.overrides[0]` is now `'to'`. Persistent role still `'to'` — unchanged.

- [ ] **Step 6: Verify override clears on client switch**

1. Flip Bob's pill in expanded view (already CC, flip to To) — `overrides[1]` is `'to'`.
2. Switch dropdown to "Other Client".
3. Switch back to "AWC". `recipientsState.overrides` is `{}`. Bob's pill is back to CC (persistent default).

- [ ] **Step 7: Verify disabled states**

1. Uncheck Bob and Maria — only Jane (To) checked. Copy To enabled, Copy CC disabled (tooltip "No CC recipients selected").
2. Uncheck Jane too. Both buttons disabled.
3. Collapse the section. `Copy To ▸` and `Copy CC ▸` in the header — both disabled, matching expanded state.

- [ ] **Step 8: Verify collapsed quick-copy doesn't toggle expand**

1. Re-check Jane (To). Collapse section.
2. Click `Copy To ▸` in collapsed header. Toast appears. Section stays collapsed.

- [ ] **Step 9: Verify pill click doesn't toggle checkbox**

1. Expand. Click pill on any checked contact — pill flips, checkbox stays checked.

- [ ] **Step 10: Verify Manage Save clears overrides**

1. With override set in expanded view (`recipientsState.overrides` non-empty), enter Manage.
2. Click Save (no edits). `recipientsState.overrides` is `{}`.

- [ ] **Step 11: Verify Done (without save) preserves overrides**

1. Set an override. Enter Manage. Click Done.
2. `recipientsState.overrides` unchanged.

- [ ] **Step 12: Real-Outlook end-to-end (one final check)**

1. Open Outlook to a new email compose window.
2. With a mixed To/CC selection in the tool: click Copy To, paste into Outlook's To field. Click Copy CC, paste into Outlook's CC field.
3. Both fields resolve to the expected recipients with no manual fixup.

- [ ] **Step 13: No commit (verification only)**

If any step above fails, return to the relevant task to fix. Do not commit "verification done" as its own commit.

---

## Self-Review (completed by plan author)

**Spec coverage check:**

| Spec section | Task(s) |
|---|---|
| Persistent role on contact | Task 1 (model), Task 9 (save) |
| Override state `recipientsState.overrides` | Task 1 |
| Migration (`role: 'to'` backfill) | Task 1, verified Task 10 step 2 |
| `effectiveRole()` | Task 1 |
| Pill render — expanded view | Task 4 |
| Pill click (override) — expanded view | Task 4 |
| Copy To / Copy CC buttons — expanded | Task 5 |
| Collapsed header — two quick copies | Task 6 |
| Override clear on client switch | Task 7 |
| Pill render — manage mode | Task 8 |
| Pill click (default toggle) — manage mode | Task 8 |
| New contact defaults to `to` | Task 8 (template) + Task 9 (save) |
| Save persists role | Task 9 |
| Save clears overrides | Task 9 |
| Done (no save) preserves overrides | Task 8 step 5, Task 10 step 11 |
| Pill doesn't toggle checkbox | Task 4 step 3 (#5), Task 10 step 9 |
| Quick-copy doesn't toggle expand | Task 6 step 3 (#2), Task 10 step 8 |
| CSS uses existing variables | Task 2 |
| Badge unchanged | (no task — current code preserved) |
| Outlook end-to-end | Task 10 step 12 |

All spec requirements have at least one task.

**Placeholder scan:** None found.

**Type consistency:** `effectiveRole(idx)` signature used identically in Task 3, Task 4, Task 5, Task 6. `copyRecipients(role)` signature identical across Tasks 3, 5, 6. Pill data attribute is `data-role-index` in expanded view (Task 4), `data-role` in manage (Task 8) — different on purpose (expanded carries the index for `overrides` map keying; manage carries the unsaved role value).
