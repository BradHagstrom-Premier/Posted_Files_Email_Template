# Client Email List Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a collapsible Recipients section to the form panel that lets consultants maintain per-client contact lists (stored in localStorage) and copy a semicolon-separated To: line for Outlook.

**Architecture:** Single-file HTML app — all changes go into `index.html`. The section is driven by a `recipientsState` JS object that mirrors the localStorage schema; every mutation calls `saveRecipientsState()` then `renderRecipients()`, which does a full innerHTML replacement of the section body and re-wires events. No interaction with `updatePreview()` — recipients are not part of the email output.

**Tech Stack:** Vanilla JS, inline CSS, localStorage, `navigator.clipboard.writeText()`

---

## File Map

| File | Change |
|---|---|
| `index.html` lines ~575–628 | Add CSS for recipients section (before `</style>`) |
| `index.html` lines ~772–774 | Add recipients section HTML (before closing `</div>` of `panel-form`) |
| `index.html` lines ~1302–1303 | Add all recipients JS (before `</script>`) |

---

### Task 1: CSS — Recipients section styles

**Files:**
- Modify: `index.html` — inside `<style>` block, before `</style>` (around line 628)

- [ ] **Step 1: Add CSS** — Insert the following block immediately before the `</style>` closing tag:

```css
        /* ───────────────────────────────────────────
           Recipients Section
           ─────────────────────────────────────────── */
        .recipients-section {
            margin-top: 28px;
            border-top: 1.5px solid var(--border);
            padding-top: 20px;
        }

        .recipients-header {
            display: flex;
            align-items: center;
            gap: 8px;
            cursor: pointer;
            user-select: none;
        }

        .recipients-header-title {
            font-size: 11px;
            font-weight: 700;
            text-transform: uppercase;
            letter-spacing: 1px;
            color: var(--blue);
            flex: 1;
        }

        .recipients-badge {
            font-size: 11px;
            color: var(--text-secondary);
            background: var(--blue-muted);
            padding: 2px 9px;
            border-radius: 20px;
            font-weight: 500;
        }

        .recipients-chevron {
            font-size: 11px;
            color: var(--text-secondary);
            transition: transform var(--transition);
        }

        .recipients-chevron.open {
            transform: rotate(180deg);
        }

        .btn-copy-quick {
            font-size: 11.5px;
            font-weight: 600;
            color: #fff;
            background: var(--blue);
            border: none;
            border-radius: var(--radius-sm);
            padding: 4px 10px;
            cursor: pointer;
            transition: background var(--transition);
            white-space: nowrap;
        }

        .btn-copy-quick:hover {
            background: var(--blue-light);
        }

        .recipients-body {
            margin-top: 14px;
        }

        .recipients-client-label {
            font-size: 12.5px;
            font-weight: 600;
            color: var(--text);
            margin-bottom: 6px;
            display: block;
        }

        .recipients-contact-list {
            margin: 10px 0 8px;
        }

        .recipients-contact-row {
            display: flex;
            align-items: center;
            gap: 8px;
            padding: 5px 0;
            border-bottom: 1px solid var(--border);
        }

        .recipients-contact-row:last-child {
            border-bottom: none;
        }

        .recipients-contact-name {
            font-size: 12.5px;
            color: var(--text);
            flex: 0 0 auto;
            min-width: 110px;
        }

        .recipients-contact-email {
            font-size: 11.5px;
            color: var(--text-secondary);
            flex: 1;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
        }

        .recipients-add-link,
        .recipients-manage-link {
            font-size: 11.5px;
            color: var(--blue);
            cursor: pointer;
            text-decoration: none;
            opacity: 0.8;
            transition: opacity var(--transition);
            background: none;
            border: none;
            padding: 0;
            font-family: inherit;
        }

        .recipients-add-link:hover,
        .recipients-manage-link:hover {
            opacity: 1;
            text-decoration: underline;
        }

        .recipients-actions {
            display: flex;
            align-items: center;
            gap: 10px;
            margin-top: 10px;
        }

        .btn-copy-recipients {
            flex: 1;
            padding: 8px;
            background: var(--blue);
            color: #fff;
            border: none;
            border-radius: var(--radius-sm);
            font-size: 13px;
            font-weight: 600;
            cursor: pointer;
            font-family: inherit;
            transition: background var(--transition);
        }

        .btn-copy-recipients:hover:not(:disabled) {
            background: var(--blue-light);
        }

        .btn-copy-recipients:disabled {
            opacity: 0.45;
            cursor: not-allowed;
        }

        /* Manage mode */
        .recipients-manage-header {
            display: flex;
            align-items: center;
            justify-content: space-between;
            margin-bottom: 14px;
        }

        .recipients-manage-title {
            font-size: 11px;
            font-weight: 700;
            text-transform: uppercase;
            letter-spacing: 1px;
            color: var(--blue);
        }

        .recipients-manage-contact-row {
            display: flex;
            gap: 5px;
            align-items: center;
            margin-bottom: 6px;
        }

        .recipients-manage-contact-row .form-input {
            font-size: 12px;
            padding: 6px 9px;
        }

        .btn-delete-contact {
            background: none;
            border: none;
            color: #b91c1c;
            font-size: 16px;
            cursor: pointer;
            padding: 0 2px;
            line-height: 1;
            flex-shrink: 0;
        }

        .btn-delete-client {
            padding: 6px 10px;
            background: #fef2f2;
            color: #b91c1c;
            border: 1px solid #fca5a5;
            border-radius: var(--radius-sm);
            font-size: 11.5px;
            cursor: pointer;
            font-family: inherit;
            transition: background var(--transition);
        }

        .btn-delete-client:hover {
            background: #fee2e2;
        }

        .btn-save-client {
            flex: 1;
            padding: 7px;
            background: var(--blue);
            color: #fff;
            border: none;
            border-radius: var(--radius-sm);
            font-size: 12.5px;
            font-weight: 600;
            cursor: pointer;
            font-family: inherit;
            transition: background var(--transition);
        }

        .btn-save-client:hover {
            background: var(--blue-light);
        }

        .btn-new-client {
            padding: 7px 10px;
            background: var(--bg);
            color: var(--text);
            border: 1.5px solid var(--border);
            border-radius: var(--radius-sm);
            font-size: 12.5px;
            cursor: pointer;
            font-family: inherit;
            transition: border-color var(--transition);
        }

        .btn-new-client:hover {
            border-color: var(--blue);
        }

        .recipients-empty {
            font-size: 12.5px;
            color: var(--text-secondary);
            text-align: center;
            padding: 14px 0;
        }
```

- [ ] **Step 2: Open `index.html` in the browser and confirm no visual regressions** — the rest of the form should look identical to before. The new CSS adds no visible elements yet.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add recipients section CSS"
```

---

### Task 2: HTML skeleton

**Files:**
- Modify: `index.html` — form panel, between `</div><!-- /#reconFields -->` and the closing `</div>` of `panel-form` (around line 772–774)

- [ ] **Step 1: Insert HTML** — Find this exact block in `index.html`:

```html
            </div><!-- /#reconFields -->

        </div>
```

Replace it with:

```html
            </div><!-- /#reconFields -->

            <!-- Recipients Section -->
            <div class="recipients-section" id="recipientsSection">
                <!-- Rendered by renderRecipients() -->
            </div>

        </div>
```

- [ ] **Step 2: Verify in browser** — No visual change expected (the div is empty). Open DevTools and confirm `#recipientsSection` exists in the DOM.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add recipients section HTML placeholder"
```

---

### Task 3: State layer

**Files:**
- Modify: `index.html` — JS section, before `</script>` (around line 1303)

- [ ] **Step 1: Add state layer JS** — Insert the following block immediately before `</script>`:

```javascript
    /* ═══════════════════════════════════════════════
       Recipients — State
       ═══════════════════════════════════════════════ */
    const RECIPIENTS_KEY = 'definian_recipients';

    let recipientsState = {
        activeClientIndex: 0,
        expanded: false,
        manageMode: false,
        clients: []
    };

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
            }
        } catch {
            recipientsState.clients = [];
            recipientsState.activeClientIndex = 0;
        }
    }

    function saveRecipientsState() {
        localStorage.setItem(RECIPIENTS_KEY, JSON.stringify({
            activeClientIndex: recipientsState.activeClientIndex,
            clients: recipientsState.clients
        }));
    }

    loadRecipientsState();
```

- [ ] **Step 2: Verify in browser console** — Open DevTools console and run:

```javascript
console.log(recipientsState);
// Expected: { activeClientIndex: 0, expanded: false, manageMode: false, clients: [] }
```

Then run `saveRecipientsState()` and check localStorage:

```javascript
saveRecipientsState();
console.log(localStorage.getItem('definian_recipients'));
// Expected: '{"activeClientIndex":0,"clients":[]}'
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add recipients state layer with localStorage"
```

---

### Task 4: renderRecipients() — normal views

**Files:**
- Modify: `index.html` — JS section, immediately after the state layer from Task 3, before `</script>`

- [ ] **Step 1: Add render function** — Insert after `loadRecipientsState();` and before `</script>`:

```javascript
    /* ═══════════════════════════════════════════════
       Recipients — Render
       ═══════════════════════════════════════════════ */
    function renderRecipients() {
        const section = document.getElementById('recipientsSection');
        if (!section) return;

        if (recipientsState.manageMode) {
            renderManageMode(section);
            return;
        }

        const client = recipientsState.clients[recipientsState.activeClientIndex] || null;
        const checkedContacts = client ? client.contacts.filter(c => c.checked) : [];
        const hasClients = recipientsState.clients.length > 0;

        // Build collapsed header (always shown)
        let badgeHTML = '';
        let quickCopyHTML = '';
        if (hasClients && client) {
            badgeHTML = `<span class="recipients-badge">${escapeHtml(client.name)} &middot; ${checkedContacts.length}</span>`;
            quickCopyHTML = `<button type="button" class="btn-copy-quick" id="recipientsCopyQuick">Copy &#9658;</button>`;
        }

        const chevronClass = recipientsState.expanded ? 'recipients-chevron open' : 'recipients-chevron';

        let bodyHTML = '';
        if (recipientsState.expanded) {
            if (!hasClients) {
                bodyHTML = `
                <div class="recipients-body">
                    <div class="recipients-empty">
                        No clients yet.
                        <button type="button" class="recipients-add-link" id="recipientsSetup">Set up your first client</button>
                    </div>
                </div>`;
            } else {
                // Client dropdown
                const options = recipientsState.clients.map((cl, i) =>
                    `<option value="${i}" ${i === recipientsState.activeClientIndex ? 'selected' : ''}>${escapeHtml(cl.name)}</option>`
                ).join('');

                // Contact list
                let contactsHTML = '';
                if (!client.contacts.length) {
                    contactsHTML = `<div class="recipients-empty" style="text-align:left;padding:8px 0;">No contacts yet.</div>`;
                } else {
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
                }

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
            }
        }

        section.innerHTML = `
            <div class="recipients-header" id="recipientsToggle">
                <span class="recipients-header-title">Recipients</span>
                ${badgeHTML}
                ${quickCopyHTML}
                <span class="${chevronClass}" aria-hidden="true">&#9660;</span>
            </div>
            ${bodyHTML}`;

        wireRecipientsEvents();
    }

    function escapeHtml(str) {
        return String(str)
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;');
    }
```

- [ ] **Step 2: Add renderManageMode() stub** — Insert immediately after `renderRecipients()`:

```javascript
    function renderManageMode(section) {
        // Implemented in Task 5
        section.innerHTML = '<div class="recipients-body"><p style="font-size:12px;color:var(--text-secondary);">Manage mode — coming soon</p></div>';
    }
```

- [ ] **Step 3: Add wireRecipientsEvents() stub** — Insert immediately after `renderManageMode()`:

```javascript
    function wireRecipientsEvents() {
        // Implemented in Task 6
    }
```

- [ ] **Step 4: Call renderRecipients() on load** — Add this line immediately after `loadRecipientsState();`:

```javascript
    renderRecipients();
```

- [ ] **Step 5: Verify in browser** — Reload the page. The Recipients section should appear at the bottom of the form panel with the header "RECIPIENTS" and a down chevron. With no clients configured, only the collapsed header is visible. Confirm no console errors.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add renderRecipients() with collapsed/expanded/empty states"
```

---

### Task 5: renderManageMode()

**Files:**
- Modify: `index.html` — replace the stub `renderManageMode()` from Task 4

- [ ] **Step 1: Replace the stub** — Find and replace the stub:

```javascript
    function renderManageMode(section) {
        // Implemented in Task 5
        section.innerHTML = '<div class="recipients-body"><p style="font-size:12px;color:var(--text-secondary);">Manage mode — coming soon</p></div>';
    }
```

With the full implementation:

```javascript
    function renderManageMode(section) {
        const client = recipientsState.clients[recipientsState.activeClientIndex] || null;

        let contactRowsHTML = '';
        if (client) {
            contactRowsHTML = client.contacts.map((c, i) => `
                <div class="recipients-manage-contact-row" data-row="${i}">
                    <input type="text" class="form-input manage-contact-name" value="${escapeHtml(c.name)}" placeholder="Name" style="flex:2;">
                    <input type="text" class="form-input manage-contact-email" value="${escapeHtml(c.email)}" placeholder="email@client.com" style="flex:3;">
                    <button type="button" class="btn-delete-contact" data-row="${i}" title="Remove contact" aria-label="Remove contact">&times;</button>
                </div>`).join('');
        }

        const clientNameVal = client ? escapeHtml(client.name) : '';
        const hasMultipleClients = recipientsState.clients.length > 1;

        section.innerHTML = `
            <div class="recipients-manage-header">
                <span class="recipients-manage-title">Manage Clients</span>
                <button type="button" class="recipients-manage-link" id="recipientsDoneBtn">&#8592; Done</button>
            </div>
            <div class="recipients-body">
                <label class="recipients-client-label" for="manageClientName">Client name</label>
                <div style="display:flex;gap:6px;margin-bottom:14px;">
                    <input type="text" class="form-input" id="manageClientName" value="${clientNameVal}" placeholder="Client name" style="flex:1;">
                    ${hasMultipleClients ? `<button type="button" class="btn-delete-client" id="manageDeleteClient">Delete</button>` : ''}
                </div>
                <label class="recipients-client-label">Contacts</label>
                <div id="manageContactRows">${contactRowsHTML}</div>
                <button type="button" class="recipients-add-link" id="manageAddContact" style="margin-bottom:14px;display:inline-block;">+ Add contact</button>
                <div style="display:flex;gap:6px;">
                    <button type="button" class="btn-save-client" id="manageSaveBtn">Save</button>
                    <button type="button" class="btn-new-client" id="manageNewClientBtn">+ New Client</button>
                </div>
            </div>`;

        wireRecipientsEvents();
    }
```

- [ ] **Step 2: Verify in browser console** — Temporarily set manage mode and re-render:

```javascript
recipientsState.manageMode = true;
recipientsState.clients = [{ name: 'AWC', contacts: [{ name: 'Patty Smith', email: 'patty@awc.gov', checked: true }] }];
recipientsState.activeClientIndex = 0;
renderRecipients();
```

The manage mode view should render with the client name input pre-filled "AWC" and one contact row. Reset afterward:

```javascript
recipientsState.manageMode = false;
recipientsState.clients = [];
renderRecipients();
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add renderManageMode() with inline client/contact editor"
```

---

### Task 6: Copy Recipients

**Files:**
- Modify: `index.html` — add `copyRecipients()` after `wireRecipientsEvents()` stub

- [ ] **Step 1: Add copyRecipients()** — Insert immediately after the `wireRecipientsEvents()` stub:

```javascript
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

- [ ] **Step 2: Verify in browser console** — Add a test client with contacts:

```javascript
recipientsState.clients = [{ name: 'AWC', contacts: [
    { name: 'Patty Smith', email: 'patty.smith@awc.gov', checked: true },
    { name: 'Linh Truong', email: 'linh.truong@awc.gov', checked: true }
]}];
recipientsState.activeClientIndex = 0;
copyRecipients();
// Toast shows "Recipients copied!" and clipboard contains:
// patty.smith@awc.gov; linh.truong@awc.gov
```

Paste into a text editor to confirm the output. Reset after:

```javascript
recipientsState.clients = [];
renderRecipients();
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add copyRecipients() with clipboard write and toast"
```

---

### Task 7: Event wiring

**Files:**
- Modify: `index.html` — replace the `wireRecipientsEvents()` stub

- [ ] **Step 1: Replace the stub** — Find and replace:

```javascript
    function wireRecipientsEvents() {
        // Implemented in Task 6
    }
```

With:

```javascript
    function wireRecipientsEvents() {
        // Expand/collapse toggle (click on header, but not on child buttons)
        const toggle = document.getElementById('recipientsToggle');
        if (toggle) {
            toggle.addEventListener('click', (e) => {
                if (e.target.closest('button')) return;
                recipientsState.expanded = !recipientsState.expanded;
                renderRecipients();
            });
        }

        // Quick copy (collapsed state)
        const quickCopy = document.getElementById('recipientsCopyQuick');
        if (quickCopy) {
            quickCopy.addEventListener('click', (e) => {
                e.stopPropagation();
                copyRecipients();
            });
        }

        // Client select dropdown
        const clientSelect = document.getElementById('recipientsClientSelect');
        if (clientSelect) {
            clientSelect.addEventListener('change', () => {
                recipientsState.activeClientIndex = parseInt(clientSelect.value, 10);
                saveRecipientsState();
                renderRecipients();
            });
        }

        // Contact checkboxes
        const contactList = document.querySelector('.recipients-contact-list');
        if (contactList) {
            contactList.addEventListener('change', (e) => {
                const checkbox = e.target.closest('input[type="checkbox"][data-contact-index]');
                if (!checkbox) return;
                const idx = parseInt(checkbox.dataset.contactIndex, 10);
                const client = recipientsState.clients[recipientsState.activeClientIndex];
                if (client && client.contacts[idx]) {
                    client.contacts[idx].checked = checkbox.checked;
                    saveRecipientsState();
                    renderRecipients();
                }
            });
        }

        // Copy Recipients button (expanded state)
        const copyBtn = document.getElementById('recipientsCopy');
        if (copyBtn) {
            copyBtn.addEventListener('click', copyRecipients);
        }

        // "+ Add contact" shortcut in expanded view (enters manage mode)
        const addContactShortcut = document.getElementById('recipientsAddContact');
        if (addContactShortcut) {
            addContactShortcut.addEventListener('click', () => {
                recipientsState.manageMode = true;
                renderRecipients();
                // Auto-click the add contact button in manage mode
                const manageAdd = document.getElementById('manageAddContact');
                if (manageAdd) manageAdd.click();
            });
        }

        // Enter manage mode
        const manageBtn = document.getElementById('recipientsManageBtn');
        if (manageBtn) {
            manageBtn.addEventListener('click', () => {
                recipientsState.manageMode = true;
                renderRecipients();
            });
        }

        // Empty state "Set up" CTA
        const setupBtn = document.getElementById('recipientsSetup');
        if (setupBtn) {
            setupBtn.addEventListener('click', () => {
                // Add a blank client, enter manage mode
                recipientsState.clients.push({ name: '', contacts: [] });
                recipientsState.activeClientIndex = recipientsState.clients.length - 1;
                recipientsState.manageMode = true;
                saveRecipientsState();
                renderRecipients();
            });
        }

        // Manage mode — Done (discard)
        const doneBtn = document.getElementById('recipientsDoneBtn');
        if (doneBtn) {
            doneBtn.addEventListener('click', () => {
                // If active client has no name and no contacts, remove it (cleanup from abandoned new-client)
                const client = recipientsState.clients[recipientsState.activeClientIndex];
                if (client && !client.name.trim() && client.contacts.length === 0) {
                    recipientsState.clients.splice(recipientsState.activeClientIndex, 1);
                    recipientsState.activeClientIndex = Math.max(0, recipientsState.clients.length - 1);
                    saveRecipientsState();
                }
                recipientsState.manageMode = false;
                renderRecipients();
            });
        }

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

        // Manage mode — Delete contact row (event delegation)
        const contactRows = document.getElementById('manageContactRows');
        if (contactRows) {
            contactRows.addEventListener('click', (e) => {
                const btn = e.target.closest('.btn-delete-contact');
                if (!btn) return;
                btn.closest('.recipients-manage-contact-row').remove();
            });
        }

        // Manage mode — New client
        const newClientBtn = document.getElementById('manageNewClientBtn');
        if (newClientBtn) {
            newClientBtn.addEventListener('click', () => {
                recipientsState.clients.push({ name: '', contacts: [] });
                recipientsState.activeClientIndex = recipientsState.clients.length - 1;
                saveRecipientsState();
                renderRecipients();
            });
        }

        // Manage mode — Delete client
        const deleteClientBtn = document.getElementById('manageDeleteClient');
        if (deleteClientBtn) {
            deleteClientBtn.addEventListener('click', () => {
                recipientsState.clients.splice(recipientsState.activeClientIndex, 1);
                recipientsState.activeClientIndex = Math.max(0, recipientsState.clients.length - 1);
                saveRecipientsState();
                recipientsState.manageMode = false;
                renderRecipients();
                showToast('Client deleted.');
            });
        }
    }
```

- [ ] **Step 2: Verify expand/collapse in browser** — Reload. Click the Recipients header — it should expand to the empty state. Click again — it should collapse. No console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: wire recipients section event handlers"
```

---

### Task 8: End-to-end verification

No new code — this task confirms the full feature works as a user would use it.

- [ ] **Step 1: Add a client from empty state**
  1. Reload `index.html`
  2. Expand Recipients — see empty state with "Set up your first client"
  3. Click "Set up your first client" — manage mode opens with blank name field
  4. Type "AWC" in the client name field
  5. Click "+ Add contact", type "Patty Smith" and "patty.smith@awc.gov"
  6. Click "+ Add contact" again, type "Linh Truong" and "linh.truong@awc.gov"
  7. Click "Save" — toast shows "Saved!", returns to expanded view
  8. Verify: client dropdown shows "AWC", both contacts listed with checkboxes checked

- [ ] **Step 2: Copy Recipients**
  1. Click "Copy Recipients" — toast shows "Recipients copied!"
  2. Paste into a text editor — confirm output is `patty.smith@awc.gov; linh.truong@awc.gov`
  3. Uncheck Patty's checkbox — Copy button should update immediately (Linh only)
  4. Click "Copy Recipients" — paste confirms only `linh.truong@awc.gov`
  5. Uncheck both — "Copy Recipients" button should read "No contacts selected" and be disabled

- [ ] **Step 3: Quick copy in collapsed state**
  1. Re-check both contacts
  2. Collapse the section
  3. Verify the badge shows "AWC · 2"
  4. Click "Copy ▸" — toast shows "Recipients copied!" without expanding the section

- [ ] **Step 4: Add a second client**
  1. Expand, click "Manage"
  2. Click "+ New Client", type "Premier", add one contact: "Brad Hagstrom" / "brad.hagstrom@definian.com" (self as test)
  3. Click "Save" — returns to expanded showing "Premier"
  4. Switch dropdown back to "AWC" — contacts switch correctly

- [ ] **Step 5: Persistence check**
  1. Refresh the page (F5)
  2. Expand Recipients — both clients and all contacts should be restored from localStorage
  3. Active client should be whichever was last selected

- [ ] **Step 6: Delete a client**
  1. Select "Premier" in the dropdown
  2. Click "Manage"
  3. Click "Delete" — toast shows "Client deleted.", returns to expanded with only "AWC"

- [ ] **Step 7: Both email modes**
  1. Switch to the Recon Report tab — Recipients section should still be visible and functional
  2. Switch back to Posted Files — same

- [ ] **Step 8: Final commit**

```bash
git add index.html
git commit -m "feat: client email list — collapsible recipients section with localStorage"
```
