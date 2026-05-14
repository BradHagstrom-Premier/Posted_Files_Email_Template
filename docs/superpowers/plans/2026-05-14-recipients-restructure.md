# Recipients Restructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make multi-client capability visible (header pill + popover), restructure manage mode (left rail) to prevent the "+ New Client looks like overwrite" bug, add a preview envelope verification strip, and fold in keyboard shortcuts + a11y fixes that touch the same surfaces.

**Architecture:** Single-file browser tool (`index.html`). Three new UI surfaces with three distinct jobs: header pill (orientation), envelope strip (verification), Recipients section (management). No data model change — `recipientsState.clients[]` already supports multi-client. All work is UI: new CSS, new render functions called from existing orchestrators, new event wiring.

**Tech Stack:** Vanilla HTML/CSS/JS. No build step, no dependencies, no test framework. Verification is manual via Chrome DevTools MCP (per `CLAUDE.md`). Spec: [`docs/superpowers/specs/2026-05-14-recipients-restructure-design.md`](../specs/2026-05-14-recipients-restructure-design.md).

---

## File Structure

Single file modified: **`index.html`**

Three zones change:
1. **`<style>` block** (inside `<head>`): new CSS rules for `.header-pill`, `.header-pill-popover`, `.envelope-strip`, `.recipients-rail`, `.recipients-help-pill`. No edits to email-output styles (those live inside JS template literals).
2. **`<body>` markup**:
   - `.header` (around line 927): add `<button class="header-pill" id="headerPill">` and `<button class="recipients-help-pill" id="helpPill">` siblings to `.header-badge`.
   - `.panel-preview` (around line 1076): add `<div class="envelope-strip" id="envelopeStrip">` between `.subject-bar` and `.preview-container`.
3. **`<script>` block**: new functions (`renderHeaderPill`, `renderPillPopover`, `renderEnvelopeStrip`, `exitManageMode`, `showDoneConfirm`, `wireTablist`), modified functions (`renderRecipients`, `renderManageMode` — refactored to include the rail inline, `loadRecipientsState`, `saveRecipientsState`, `wireRecipientsEvents`), new global keydown handlers.

Spec file `docs/superpowers/specs/2026-05-14-recipients-restructure-design.md` is the reference; never modified by this plan.

---

## TDD Adaptation (No Test Framework)

Each task follows: **(1) Define the expected behavior** (acceptance criteria), **(2) Capture current behavior** to confirm the gap, **(3) Implement**, **(4) Verify in browser**, **(5) Commit**.

Verification uses Chrome DevTools MCP via `evaluate_script` snippets that return truthy/falsy assertions. Open `index.html` in Chrome with the DevTools MCP attached. For state-dependent tasks, the snippets seed `localStorage` directly so verification is reproducible.

---

## Task 1: State Foundations & Expanded Persistence

**Files:**
- Modify: `index.html` — `recipientsState` initial object, `loadRecipientsState()`, `saveRecipientsState()`

**Acceptance:** Expanding the Recipients section, reloading the page, and reading `recipientsState.expanded` returns `true`. New keys `manageDirty` and `popoverOpen` exist and default to `false`.

- [ ] **Step 1: Capture current state shape**

Run in DevTools console with the tool loaded:
```js
({
  hasManageDirty: 'manageDirty' in recipientsState,
  hasPopoverOpen: 'popoverOpen' in recipientsState,
  expandedPersistsAcrossReload: (() => {
    const raw = JSON.parse(localStorage.getItem('definian_recipients') || '{}');
    return 'expanded' in raw;
  })(),
})
```
Expected: all three `false` (these keys don't exist yet).

- [ ] **Step 2: Update `recipientsState` initial value**

Locate `let recipientsState = {` (currently line 1608). Replace the block with:

```js
let recipientsState = {
    activeClientIndex: 0,
    expanded: false,
    manageMode: false,
    manageDirty: false,
    popoverOpen: false,
    clients: [],
    overrides: {}   // { [contactIndex]: 'to' | 'cc' } — transient, never persisted
};
```

- [ ] **Step 3: Update `loadRecipientsState()` to read `expanded`**

Inside the `if (raw) {` block in `loadRecipientsState`, after the migration `forEach` loop, add:

```js
if (typeof saved.expanded === 'boolean') {
    recipientsState.expanded = saved.expanded;
}
```

- [ ] **Step 4: Update `saveRecipientsState()` to persist `expanded`**

Replace the body of `saveRecipientsState()` with:

```js
localStorage.setItem(RECIPIENTS_KEY, JSON.stringify({
    activeClientIndex: recipientsState.activeClientIndex,
    expanded: recipientsState.expanded,
    clients: recipientsState.clients
}));
```

`manageDirty` and `popoverOpen` are explicitly NOT persisted — they're session-only.

- [ ] **Step 5: Verify behavior**

Reload the tool. Run in DevTools console:
```js
recipientsState.expanded = true;
saveRecipientsState();
location.reload();
// After reload:
recipientsState.expanded   // expect: true
recipientsState.manageDirty   // expect: false
recipientsState.popoverOpen   // expect: false
JSON.parse(localStorage.getItem('definian_recipients')).expanded   // expect: true
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(recipients): persist expanded; add manageDirty and popoverOpen flags"
```

---

## Task 2: Convert Section Header to `<button>` (Keyboard + A11y)

**Files:**
- Modify: `index.html` — `renderRecipients()` HTML template for the section header, `.recipients-header` CSS, click handler in `wireRecipientsEvents()`

**Acceptance:** Tabbing into the page focuses the Recipients header at the appropriate point; Space or Enter toggles expand/collapse; visible focus ring appears using the existing `:focus-visible` blue halo.

- [ ] **Step 1: Capture current behavior**

In DevTools console with tool loaded:
```js
const el = document.getElementById('recipientsToggle');
({
  tag: el.tagName,                     // expect: "DIV" (the bug we're fixing)
  hasTabindex: el.hasAttribute('tabindex'),
  isFocusable: (() => { el.focus(); return document.activeElement === el; })(),
})
```
Expected: `tag: "DIV"`, `hasTabindex: false`, `isFocusable: false`.

- [ ] **Step 2: Change the element to `<button>` in `renderRecipients()`**

Find the template at the end of `renderRecipients()`:
```js
section.innerHTML = `
    <div class="recipients-header" id="recipientsToggle">
```

Replace with:
```js
section.innerHTML = `
    <button type="button" class="recipients-header" id="recipientsToggle" aria-expanded="${recipientsState.expanded}" aria-controls="recipientsBody">
```

Then locate the matching closing `</div>` for that header (immediately before `${bodyHTML}`):
```js
        <span class="${chevronClass}" aria-hidden="true">&#9660;</span>
    </div>
```
Change `</div>` to `</button>`.

Also wrap `bodyHTML` so it has the `recipientsBody` id for `aria-controls`. Locate every `bodyHTML = \`<div class="recipients-body">` construction inside `renderRecipients()` (there are two: empty-state and normal-state) and add `id="recipientsBody"` to each:
```js
bodyHTML = `
<div class="recipients-body" id="recipientsBody">
```

- [ ] **Step 3: Update `.recipients-header` CSS to a button reset**

Locate `.recipients-header {` (around line 638) and add to that rule (do not replace existing properties):

```css
.recipients-header {
    /* ...existing properties... */
    background: none;
    border: none;
    width: 100%;
    text-align: left;
    font: inherit;
    color: inherit;
    cursor: pointer;
}
.recipients-header:focus-visible {
    outline: none;
    box-shadow: 0 0 0 3px rgba(13, 44, 113, 0.18);
    border-radius: 6px;
}
```

- [ ] **Step 4: Verify Space and Enter toggle the section**

In DevTools, after the tool reloads:
```js
const el = document.getElementById('recipientsToggle');
({
  tag: el.tagName,                                       // expect: "BUTTON"
  ariaExpanded: el.getAttribute('aria-expanded'),        // expect: "false"
  isFocusable: (() => { el.focus(); return document.activeElement === el; })(),  // expect: true
})
// Then press Space with the header focused — section should expand.
// Verify aria-expanded flips to "true" after expand.
```

Native `<button>` behavior handles Space/Enter automatically — no extra keydown handler needed. The existing click handler in `wireRecipientsEvents()` already toggles `recipientsState.expanded`; it will fire on Space/Enter too because buttons synthesize click events for both keys.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(recipients): section header is a button with aria-expanded"
```

---

## Task 3: Rebalance Collapsed Copy Buttons + Chevron Contrast

**Files:**
- Modify: `index.html` — `.btn-copy-quick` CSS, `.recipients-chevron` CSS

**Acceptance:** Both collapsed-state quick-copy buttons render as secondary style (transparent background, Definian Blue text, 1.5px border). Chevron uses `--text-primary` color instead of `--text-secondary`.

- [ ] **Step 1: Capture current behavior**

In DevTools console with tool expanded and a client active:
```js
recipientsState.expanded = false;
renderRecipients();
const to = document.getElementById('recipientsCopyQuickTo');
const cc = document.getElementById('recipientsCopyQuickCc');
const chevron = document.querySelector('.recipients-chevron');
({
  toBg: getComputedStyle(to).backgroundColor,
  ccBg: getComputedStyle(cc).backgroundColor,
  chevronColor: getComputedStyle(chevron).color,
})
```
Capture the values. Expected after the fix: `toBg` and `ccBg` both `rgba(0, 0, 0, 0)` (transparent); chevron color matches `--text-primary` (`rgb(26, 26, 46)`).

- [ ] **Step 2: Locate and replace `.btn-copy-quick` rule**

Search for `.btn-copy-quick {`. Replace the existing rule (and any `.btn-copy-quick--secondary` variant, if present) with:

```css
.btn-copy-quick {
    background: transparent;
    color: var(--blue);
    border: 1.5px solid var(--blue);
    border-radius: var(--radius-sm);
    padding: 5px 10px;
    font-family: inherit;
    font-size: 11.5px;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.15s ease-out, color 0.15s ease-out;
}
.btn-copy-quick:hover:not(:disabled) {
    background: rgba(13, 44, 113, 0.08);
}
.btn-copy-quick:active:not(:disabled) {
    transform: scale(0.97);
}
.btn-copy-quick:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}
```

- [ ] **Step 3: Bump chevron contrast**

Search for `.recipients-chevron {`. Change the `color` property from `var(--text-secondary)` to `var(--text-primary)`. Leave all other properties intact.

- [ ] **Step 4: Verify in browser**

Reload. Seed localStorage with a client that has contacts (if none exists). Collapse the section. Run:
```js
const to = document.getElementById('recipientsCopyQuickTo');
const chevron = document.querySelector('.recipients-chevron');
({
  toBg: getComputedStyle(to).backgroundColor,        // expect: "rgba(0, 0, 0, 0)"
  toBorder: getComputedStyle(to).borderColor,         // expect: "rgb(13, 44, 113)"
  toColor: getComputedStyle(to).color,                // expect: "rgb(13, 44, 113)"
  chevronColor: getComputedStyle(chevron).color,      // expect: "rgb(26, 26, 46)"
})
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(recipients): collapsed quick-copy buttons drop to secondary; chevron gains contrast"
```

---

## Task 4: Header "Sending to:" Pill — Markup + Render

**Files:**
- Modify: `index.html` — `.header` markup, new CSS for `.header-pill`, new `renderHeaderPill()` function, calls from `renderRecipients()` and the existing `loadRecipientsState()` flow

**Acceptance:** Pill renders in the app header reflecting the active client. With 0 clients: `+ Add recipients` (dashed outline). With named client + N contacts: `Sending to: AWC · N ▾`. Updates whenever `recipientsState` mutates.

- [ ] **Step 1: Add pill markup to `.header`**

Locate the `.header` block (around line 927):
```html
<div class="header">
    <div class="header-pip"></div>
    <span class="header-title">Posted Files Email Generator</span>
    <span class="header-badge">Definian</span>
</div>
```

Replace with:
```html
<div class="header">
    <div class="header-pip"></div>
    <span class="header-title">Posted Files Email Generator</span>
    <button type="button" class="header-pill" id="headerPill" aria-haspopup="menu" aria-expanded="false">
        <span class="header-pill-text">Loading…</span>
        <span class="header-pill-chevron" aria-hidden="true">&#9662;</span>
    </button>
    <span class="header-badge">Definian</span>
</div>
```

- [ ] **Step 2: Add `.header-pill` CSS**

Add to the `<style>` block, after the existing `.header-badge` rule:

```css
.header-pill {
    margin-left: auto;
    margin-right: 10px;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    background: rgba(255, 255, 255, 0.18);
    color: white;
    border: none;
    border-radius: 6px;
    padding: 5px 10px;
    font: 600 11.5px / 1 'DM Sans', 'Segoe UI', system-ui, -apple-system, sans-serif;
    cursor: pointer;
    transition: background-color 0.15s ease-out;
}
.header-pill:hover { background: rgba(255, 255, 255, 0.28); }
.header-pill[aria-expanded="true"] {
    background: white;
    color: var(--blue);
}
.header-pill[aria-expanded="true"] .header-pill-chevron { transform: rotate(180deg); }
.header-pill-chevron {
    font-size: 10px;
    opacity: 0.7;
    transition: transform 0.15s ease-out;
}
.header-pill.is-empty {
    background: transparent;
    border: 1.5px dashed rgba(255, 255, 255, 0.45);
    padding: 3.5px 10px;  /* compensate for border */
}
.header-pill.is-empty:hover { background: rgba(255, 255, 255, 0.10); }
.header-pill.is-warn .header-pill-chevron { color: #FCD34D; opacity: 1; }
.header-pill:focus-visible {
    outline: none;
    box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.45);
}
@media (max-width: 1024px) {
    .header-pill-text .pill-prefix { display: none; }
}
```

The `:focus-visible` halo uses white-tinted shadow because the pill sits on a blue header (the normal blue halo would be invisible).

- [ ] **Step 3: Add `renderHeaderPill()` function**

Insert immediately before `function renderRecipients() {`:

```js
function renderHeaderPill() {
    const pill = document.getElementById('headerPill');
    if (!pill) return;
    const textEl = pill.querySelector('.header-pill-text');
    const hasClients = recipientsState.clients.length > 0;
    pill.classList.remove('is-empty', 'is-warn');
    if (!hasClients) {
        pill.classList.add('is-empty');
        textEl.textContent = '+ Add recipients';
        return;
    }
    const client = recipientsState.clients[recipientsState.activeClientIndex];
    const name = client && client.name ? client.name : '(Untitled)';
    const checkedCount = client ? client.contacts.filter(c => c.checked).length : 0;
    const isWarn = !client.name || checkedCount === 0;
    if (isWarn) pill.classList.add('is-warn');
    const countSuffix = client && client.name && checkedCount > 0 ? ` · ${checkedCount}` : '';
    textEl.innerHTML = `<span class="pill-prefix">Sending to: </span>${escapeHtml(name)}${countSuffix}`;
}
```

- [ ] **Step 4: Wire pill render into the existing render flow**

Inside `renderRecipients()`, near the top after the guard `if (!section) return;`, add:
```js
renderHeaderPill();
```

Also call it once at initial load. Locate the top-level invocation `loadRecipientsState();` (currently around line 1646). Immediately after that line, add:
```js
renderHeaderPill();
```
The script tag sits at the end of `<body>` so the DOM is ready at this point.

- [ ] **Step 5: Verify pill state across all variants**

In DevTools console:
```js
// Variant: no clients
localStorage.setItem('definian_recipients', JSON.stringify({ clients: [], activeClientIndex: 0, expanded: false }));
location.reload();
// After reload, inspect:
document.getElementById('headerPill').textContent  // expect contains: "+ Add recipients"
document.getElementById('headerPill').classList.contains('is-empty')  // expect: true

// Variant: client with contacts
localStorage.setItem('definian_recipients', JSON.stringify({
  clients: [{ name: 'AWC', contacts: [
    { name: 'A', email: 'a@x.com', role: 'to', checked: true },
    { name: 'B', email: 'b@x.com', role: 'cc', checked: true }
  ]}],
  activeClientIndex: 0, expanded: false
}));
location.reload();
document.getElementById('headerPill').textContent.replace(/\s+/g, ' ').trim()
// expect: "Sending to: AWC · 2 ▾"

// Variant: client with 0 contacts (warn state)
localStorage.setItem('definian_recipients', JSON.stringify({
  clients: [{ name: 'AWC', contacts: [] }], activeClientIndex: 0, expanded: false
}));
location.reload();
document.getElementById('headerPill').classList.contains('is-warn')  // expect: true
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(recipients): header pill displays active client and count"
```

---

## Task 5: Header Pill Popover (Client Switcher)

**Files:**
- Modify: `index.html` — new `renderPillPopover()`, click handlers, document-level outside-click + Esc handler

**Acceptance:** Clicking the pill opens a 240px popover listing all clients (active row highlighted) plus `+ New client` and `Manage clients...` links. Clicking outside, pressing Esc, or selecting an option closes the popover. Selecting a different client updates `activeClientIndex` and re-renders everything. Arrow Up/Down navigates rows; Enter selects.

- [ ] **Step 1: Add popover markup placeholder**

Inside `.header`, immediately after the `<button class="header-pill">` element (before `<span class="header-badge">`), add:
```html
<div class="header-pill-popover" id="headerPillPopover" role="menu" hidden></div>
```

- [ ] **Step 2: Add `.header-pill-popover` CSS**

Add after the existing `.header-pill` rules:

```css
.header-pill-popover {
    position: absolute;
    top: 44px;
    right: 16px;
    z-index: 100;
    width: 240px;
    background: white;
    border-radius: 8px;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.18);
    padding: 6px;
    font: 500 13px / 1.4 'DM Sans', 'Segoe UI', system-ui, sans-serif;
    color: var(--text-primary);
}
.header-pill-popover[hidden] { display: none; }
.popover-client-row {
    display: flex;
    align-items: center;
    padding: 8px 10px;
    border-radius: 5px;
    cursor: pointer;
    position: relative;
    outline: none;
}
.popover-client-row:hover, .popover-client-row:focus { background: var(--surface-bg); }
.popover-client-row.is-active { font-weight: 700; }
.popover-client-row.is-active::before {
    content: '';
    position: absolute;
    left: 0;
    top: 6px;
    bottom: 6px;
    width: 4px;
    background: var(--blue);
    border-radius: 2px;
}
.popover-client-row .count-suffix { margin-left: auto; color: var(--text-secondary); font-weight: 400; font-size: 11.5px; }
.popover-divider { height: 1px; background: var(--border-default); margin: 6px 0; }
.popover-action {
    display: block;
    width: 100%;
    text-align: left;
    background: none;
    border: none;
    padding: 8px 10px;
    border-radius: 5px;
    color: var(--blue);
    font: 600 13px / 1.4 inherit;
    cursor: pointer;
}
.popover-action:hover, .popover-action:focus { background: rgba(13, 44, 113, 0.08); outline: none; }
```

Note: `.header` already has `position: relative` (verify with `getComputedStyle(document.querySelector('.header')).position`) — if not, add `position: relative` to `.header`.

- [ ] **Step 3: Add `renderPillPopover()` function**

Insert immediately after `renderHeaderPill()`:

```js
function renderPillPopover() {
    const pop = document.getElementById('headerPillPopover');
    if (!pop) return;
    if (!recipientsState.popoverOpen) {
        pop.hidden = true;
        document.getElementById('headerPill').setAttribute('aria-expanded', 'false');
        return;
    }
    pop.hidden = false;
    document.getElementById('headerPill').setAttribute('aria-expanded', 'true');

    const clients = recipientsState.clients;
    const active = recipientsState.activeClientIndex;
    const rowsHTML = clients.length === 0
        ? `<div style="padding:12px 10px;color:var(--text-secondary);font-size:12px">No clients yet.</div>`
        : clients.map((cl, i) => {
            const name = cl.name || '(Untitled)';
            const count = cl.contacts.filter(c => c.checked).length;
            const activeCls = i === active ? ' is-active' : '';
            return `<div class="popover-client-row${activeCls}" role="menuitem" tabindex="0" data-client-index="${i}">
                ${escapeHtml(name)}<span class="count-suffix">${count}</span>
            </div>`;
        }).join('');

    pop.innerHTML = `
        ${rowsHTML}
        <div class="popover-divider"></div>
        <button type="button" class="popover-action" id="popoverNewClient" role="menuitem">+ New client</button>
        <button type="button" class="popover-action" id="popoverManage" role="menuitem">Manage clients…</button>
    `;

    // Wire row clicks
    pop.querySelectorAll('.popover-client-row').forEach(row => {
        row.addEventListener('click', () => {
            const idx = parseInt(row.dataset.clientIndex, 10);
            recipientsState.activeClientIndex = idx;
            recipientsState.overrides = {};
            recipientsState.popoverOpen = false;
            saveRecipientsState();
            renderRecipients();
        });
        row.addEventListener('keydown', (e) => {
            if (e.key === 'Enter') { e.preventDefault(); row.click(); }
        });
    });
    document.getElementById('popoverNewClient').addEventListener('click', () => {
        recipientsState.clients.push({ name: '', contacts: [] });
        recipientsState.activeClientIndex = recipientsState.clients.length - 1;
        recipientsState.manageMode = true;
        recipientsState.manageDirty = false;
        recipientsState.popoverOpen = false;
        recipientsState.expanded = true;
        saveRecipientsState();
        renderRecipients();
        // focus the new client's name field
        setTimeout(() => {
            const nameInput = document.getElementById('manageClientName');
            if (nameInput) nameInput.focus();
        }, 0);
    });
    document.getElementById('popoverManage').addEventListener('click', () => {
        recipientsState.manageMode = true;
        recipientsState.manageDirty = false;
        recipientsState.popoverOpen = false;
        recipientsState.expanded = true;
        renderRecipients();
        document.getElementById('recipientsSection').scrollIntoView({ behavior: 'smooth', block: 'start' });
    });

    // Focus the active row (or first action if no clients)
    const activeRow = pop.querySelector('.popover-client-row.is-active') || pop.querySelector('.popover-client-row') || pop.querySelector('.popover-action');
    if (activeRow) activeRow.focus();
}
```

- [ ] **Step 4: Wire pill click to toggle popover, plus outside-click/Esc**

Inside `loadRecipientsState();` initialization area (near the existing `renderHeaderPill();` call), add:

```js
document.getElementById('headerPill').addEventListener('click', (e) => {
    e.stopPropagation();
    if (recipientsState.clients.length === 0) {
        // No clients yet — go straight to manage mode for new client
        recipientsState.clients.push({ name: '', contacts: [] });
        recipientsState.activeClientIndex = 0;
        recipientsState.manageMode = true;
        recipientsState.manageDirty = false;
        recipientsState.expanded = true;
        saveRecipientsState();
        renderRecipients();
        setTimeout(() => {
            const nameInput = document.getElementById('manageClientName');
            if (nameInput) nameInput.focus();
        }, 0);
        return;
    }
    recipientsState.popoverOpen = !recipientsState.popoverOpen;
    renderPillPopover();
});

document.addEventListener('click', (e) => {
    if (!recipientsState.popoverOpen) return;
    const pop = document.getElementById('headerPillPopover');
    const pill = document.getElementById('headerPill');
    if (pop.contains(e.target) || pill.contains(e.target)) return;
    recipientsState.popoverOpen = false;
    renderPillPopover();
});

document.addEventListener('keydown', (e) => {
    if (e.key !== 'Escape') return;
    if (recipientsState.popoverOpen) {
        recipientsState.popoverOpen = false;
        renderPillPopover();
        document.getElementById('headerPill').focus();
    }
});
```

Also extend `renderHeaderPill()` so that when the pill is re-rendered, it calls `renderPillPopover()` to keep the popover in sync. Add to the end of `renderHeaderPill()`:
```js
renderPillPopover();
```

- [ ] **Step 5: Verify popover behavior**

In DevTools console:
```js
// Seed two clients
localStorage.setItem('definian_recipients', JSON.stringify({
  clients: [
    { name: 'AWC', contacts: [{ name:'A', email:'a@x.com', role:'to', checked:true }] },
    { name: 'Beta', contacts: [] }
  ],
  activeClientIndex: 0, expanded: false
}));
location.reload();

// Click pill, check popover
document.getElementById('headerPill').click();
({
  popoverVisible: !document.getElementById('headerPillPopover').hidden,         // expect: true
  rowCount: document.querySelectorAll('.popover-client-row').length,             // expect: 2
  ariaExpanded: document.getElementById('headerPill').getAttribute('aria-expanded'),  // expect: "true"
})

// Switch to Beta
document.querySelector('[data-client-index="1"]').click();
recipientsState.activeClientIndex   // expect: 1
recipientsState.popoverOpen          // expect: false
```

Also manually: click pill, press Esc, popover closes and pill regains focus. Click pill, click outside the popover, popover closes.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(recipients): pill popover switches clients, opens new/manage"
```

---

## Task 6: Manage Mode Left Rail (Multi-Client Visibility)

**Files:**
- Modify: `index.html` — `renderManageMode()` function, new CSS for `.recipients-rail`

**Acceptance:** Manage mode shows a left-side rail listing every client. The active client's row has a 4px Definian Blue left-edge fill. Clicking any row switches the editor to that client without losing in-memory edits. `+ New client` and `Delete client` move to the rail bottom. On viewports <1024px, the rail collapses to a horizontal pill scroller above the editor.

- [ ] **Step 1: Capture current behavior**

```js
recipientsState.manageMode = true;
renderRecipients();
({
  hasRail: !!document.querySelector('.recipients-rail'),     // expect: false
  clientCountVisible: document.querySelectorAll('.recipients-manage-contact-row').length === recipientsState.clients[recipientsState.activeClientIndex].contacts.length,  // is showing active only
})
```

- [ ] **Step 2: Add `.recipients-rail` CSS**

Add to the `<style>` block, near the other `.recipients-manage-*` rules:

```css
.recipients-manage-layout {
    display: flex;
    gap: 14px;
    align-items: flex-start;
}
.recipients-rail {
    flex: 0 0 180px;
    border-right: 1px solid var(--border-default);
    padding-right: 14px;
    display: flex;
    flex-direction: column;
    gap: 2px;
}
.recipients-rail-row {
    position: relative;
    background: none;
    border: none;
    text-align: left;
    padding: 8px 10px 8px 14px;
    border-radius: 5px;
    cursor: pointer;
    font: 500 13px / 1.3 inherit;
    color: var(--text-primary);
    display: flex;
    align-items: center;
    outline: none;
}
.recipients-rail-row:hover, .recipients-rail-row:focus { background: var(--surface-bg); }
.recipients-rail-row.is-active { font-weight: 700; }
.recipients-rail-row.is-active::before {
    content: '';
    position: absolute;
    left: 0;
    top: 6px;
    bottom: 6px;
    width: 4px;
    background: var(--blue);
    border-radius: 2px;
}
.recipients-rail-row .rail-count {
    margin-left: auto;
    color: var(--text-secondary);
    font-weight: 400;
    font-size: 11.5px;
}
.recipients-rail-divider { height: 1px; background: var(--border-default); margin: 6px 0; }
.recipients-rail-new {
    background: none;
    border: 1.5px dashed var(--border-default);
    border-radius: 5px;
    padding: 7px 10px;
    color: var(--blue);
    font: 600 12.5px inherit;
    cursor: pointer;
    text-align: left;
}
.recipients-rail-new:hover { background: rgba(13, 44, 113, 0.06); border-color: var(--blue); }
.recipients-rail-delete {
    margin-top: 6px;
    background: none;
    border: 1px solid var(--border-default);
    border-radius: 5px;
    padding: 6px 10px;
    color: var(--error);
    font: 600 12px inherit;
    cursor: pointer;
    text-align: center;
}
.recipients-rail-delete:hover { background: #FEF2F2; border-color: var(--error); }
.recipients-manage-editor { flex: 1; min-width: 0; }
@media (max-width: 1024px) {
    .recipients-manage-layout { flex-direction: column; }
    .recipients-rail {
        flex: 0 0 auto;
        flex-direction: row;
        gap: 6px;
        border-right: none;
        border-bottom: 1px solid var(--border-default);
        padding-right: 0;
        padding-bottom: 10px;
        overflow-x: auto;
    }
    .recipients-rail-row { flex: 0 0 auto; }
    .recipients-rail-divider { display: none; }
}
```

- [ ] **Step 3: Refactor `renderManageMode()`**

Replace the entire body of `renderManageMode(section)` with:

```js
function renderManageMode(section) {
    renderHeaderPill();  // keep pill in sync while in manage mode
    const clients = recipientsState.clients;
    const activeIdx = recipientsState.activeClientIndex;
    const client = clients[activeIdx] || null;

    // Editor (right side)
    let contactRowsHTML = '';
    if (client) {
        contactRowsHTML = client.contacts.map((c, i) => {
            const role = c.role || 'to';
            return `
            <div class="recipients-manage-contact-row" data-row="${i}">
                <input type="text" class="form-input manage-contact-name" value="${escapeHtml(c.name)}" placeholder="Name" style="flex:2;">
                <input type="text" class="form-input manage-contact-email" value="${escapeHtml(c.email)}" placeholder="email@client.com" style="flex:3;">
                <button type="button" class="role-pill role-pill--${role}" data-role="${role}" aria-pressed="${role === 'cc' ? 'true' : 'false'}" title="Click to flip default To/CC">${role === 'to' ? 'To' : 'CC'}</button>
                <button type="button" class="btn-delete-contact" data-row="${i}" title="Remove contact" aria-label="Remove contact">&times;</button>
            </div>`;
        }).join('');
    }

    const clientNameVal = client ? escapeHtml(client.name) : '';

    // Rail (left side)
    const railRowsHTML = clients.map((cl, i) => {
        const name = cl.name || '(Untitled)';
        const count = cl.contacts.length;
        const activeCls = i === activeIdx ? ' is-active' : '';
        return `<button type="button" class="recipients-rail-row${activeCls}" role="option" aria-selected="${i === activeIdx ? 'true' : 'false'}" data-rail-index="${i}">
            <span class="rail-name">${escapeHtml(name)}</span>
            <span class="rail-count">${count}</span>
        </button>`;
    }).join('');

    const showDelete = clients.length > 1;

    section.innerHTML = `
        <div class="recipients-manage-header">
            <span class="recipients-manage-title">Manage Clients</span>
            <button type="button" class="recipients-manage-link" id="recipientsDoneBtn">&#8592; Done</button>
        </div>
        <div class="recipients-body" id="recipientsBody">
            <div class="recipients-manage-layout">
                <div class="recipients-rail" role="listbox" aria-label="Clients">
                    ${railRowsHTML}
                    <div class="recipients-rail-divider"></div>
                    <button type="button" class="recipients-rail-new" id="railNewClient">+ New client</button>
                    ${showDelete ? `<button type="button" class="recipients-rail-delete" id="railDeleteClient">Delete client</button>` : ''}
                </div>
                <div class="recipients-manage-editor">
                    <label class="recipients-client-label" for="manageClientName">Client name</label>
                    <input type="text" class="form-input" id="manageClientName" value="${clientNameVal}" placeholder="Client name" style="width:100%;margin-bottom:14px;">
                    <label class="recipients-client-label">Contacts</label>
                    <div id="manageContactRows">${contactRowsHTML}</div>
                    <button type="button" class="recipients-add-link" id="manageAddContact" style="margin-bottom:14px;display:inline-block;">+ Add contact</button>
                    <div style="display:flex;gap:6px;">
                        <button type="button" class="btn-save-client" id="manageSaveBtn">Save</button>
                    </div>
                </div>
            </div>
        </div>`;

    wireRecipientsEvents();
}
```

- [ ] **Step 4: Update `wireRecipientsEvents()` to handle rail clicks and new buttons**

Inside `wireRecipientsEvents()`, locate the existing handlers for `manageDeleteClient` and `manageNewClientBtn` (around line 1898-1920). Remove those handlers (they're gone from the DOM). In their place, add rail-row handler and rail-new/rail-delete handlers:

```js
// Rail row clicks — switch active client without leaving manage mode
document.querySelectorAll('.recipients-rail-row').forEach(row => {
    row.addEventListener('click', () => {
        // Capture in-memory edits to the previous client before switching
        const prevClient = recipientsState.clients[recipientsState.activeClientIndex];
        if (prevClient) {
            const nameInput = document.getElementById('manageClientName');
            if (nameInput) prevClient.name = nameInput.value.trim();
            document.querySelectorAll('.recipients-manage-contact-row').forEach(r => {
                const i = parseInt(r.dataset.row, 10);
                const nameEl = r.querySelector('.manage-contact-name');
                const emailEl = r.querySelector('.manage-contact-email');
                const roleBtn = r.querySelector('.role-pill');
                if (prevClient.contacts[i]) {
                    prevClient.contacts[i].name = nameEl.value.trim();
                    prevClient.contacts[i].email = emailEl.value.trim();
                    prevClient.contacts[i].role = roleBtn.dataset.role;
                }
            });
        }
        recipientsState.activeClientIndex = parseInt(row.dataset.railIndex, 10);
        renderRecipients();
    });
});

const railNew = document.getElementById('railNewClient');
if (railNew) {
    railNew.addEventListener('click', () => {
        recipientsState.clients.push({ name: '', contacts: [] });
        recipientsState.activeClientIndex = recipientsState.clients.length - 1;
        recipientsState.manageDirty = true;
        renderRecipients();
        setTimeout(() => {
            const nameInput = document.getElementById('manageClientName');
            if (nameInput) nameInput.focus();
        }, 0);
    });
}

const railDelete = document.getElementById('railDeleteClient');
if (railDelete) {
    railDelete.addEventListener('click', () => {
        if (recipientsState.clients.length <= 1) return;
        recipientsState.clients.splice(recipientsState.activeClientIndex, 1);
        recipientsState.activeClientIndex = Math.max(0, recipientsState.activeClientIndex - 1);
        recipientsState.manageDirty = true;
        renderRecipients();
    });
}
```

- [ ] **Step 5: Verify rail behavior**

In DevTools console:
```js
// Seed two clients
localStorage.setItem('definian_recipients', JSON.stringify({
  clients: [
    { name: 'AWC', contacts: [{ name:'A', email:'a@x.com', role:'to', checked:true }, { name:'B', email:'b@x.com', role:'cc', checked:true }] },
    { name: 'Beta', contacts: [{ name:'C', email:'c@x.com', role:'to', checked:true }] }
  ],
  activeClientIndex: 0, expanded: true
}));
location.reload();
recipientsState.manageMode = true;
renderRecipients();
({
  railRowCount: document.querySelectorAll('.recipients-rail-row').length,       // expect: 2
  activeRailIdx: document.querySelector('.recipients-rail-row.is-active').dataset.railIndex,  // expect: "0"
  editorClientName: document.getElementById('manageClientName').value,           // expect: "AWC"
  newClientBtnVisible: !!document.getElementById('railNewClient'),               // expect: true
  deleteBtnVisible: !!document.getElementById('railDeleteClient'),               // expect: true (2 clients)
})

// Switch to Beta via rail
document.querySelector('[data-rail-index="1"]').click();
document.getElementById('manageClientName').value  // expect: "Beta"
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(recipients): manage mode shows all clients in a left rail"
```

---

## Task 7: Dirty-State Done Guard

**Files:**
- Modify: `index.html` — `wireRecipientsEvents()` to track dirty state on manage-mode inputs, intercept Done click, render inline confirm

**Acceptance:** Editing any input in manage mode sets `recipientsState.manageDirty = true`. Clicking Done with `manageDirty: true` shows an inline confirm (`Save & Close` / `Discard`) inside the manage header. `Save` clears `manageDirty`. Esc with `manageDirty: true` triggers the same confirm.

- [ ] **Step 1: Add dirty-tracking listeners in `wireRecipientsEvents()`**

Locate `wireRecipientsEvents()`. Inside the function, after the rail handlers from Task 6, add:

```js
// Mark dirty on any manage-mode input change
document.querySelectorAll('.recipients-manage-contact-row input, #manageClientName').forEach(input => {
    input.addEventListener('input', () => { recipientsState.manageDirty = true; });
});
document.querySelectorAll('.recipients-manage-contact-row .role-pill, .recipients-manage-contact-row .btn-delete-contact, #manageAddContact').forEach(btn => {
    btn.addEventListener('click', () => { recipientsState.manageDirty = true; });
});
```

- [ ] **Step 2: Intercept Done click with dirty guard**

Locate the existing `recipientsDoneBtn` handler in `wireRecipientsEvents()` (around line 1910-1920). Replace its body with:

```js
const doneBtn = document.getElementById('recipientsDoneBtn');
if (doneBtn) {
    doneBtn.addEventListener('click', () => {
        if (recipientsState.manageDirty) {
            showDoneConfirm();
            return;
        }
        exitManageMode();
    });
}
```

- [ ] **Step 3: Add `showDoneConfirm()` and `exitManageMode()` helpers**

Insert immediately above `renderManageMode`:

```js
function exitManageMode() {
    const client = recipientsState.clients[recipientsState.activeClientIndex];
    if (client && !client.name.trim() && client.contacts.length === 0) {
        recipientsState.clients.splice(recipientsState.activeClientIndex, 1);
        recipientsState.activeClientIndex = Math.max(0, recipientsState.clients.length - 1);
    }
    recipientsState.manageMode = false;
    recipientsState.manageDirty = false;
    saveRecipientsState();
    renderRecipients();
}

function showDoneConfirm() {
    const header = document.querySelector('.recipients-manage-header');
    if (!header) return;
    header.innerHTML = `
        <span class="recipients-manage-title" style="color:var(--text-secondary)">Discard unsaved changes?</span>
        <button type="button" class="btn-save-client" id="confirmSaveClose" style="margin-right:6px;">Save &amp; Close</button>
        <button type="button" class="recipients-manage-link" id="confirmDiscard" style="color:var(--error);">Discard</button>
    `;
    document.getElementById('confirmSaveClose').addEventListener('click', () => {
        // Trigger save (commits in-memory state to localStorage), then exit
        const saveBtn = document.getElementById('manageSaveBtn');
        if (saveBtn) saveBtn.click();
        exitManageMode();
    });
    document.getElementById('confirmDiscard').addEventListener('click', () => {
        recipientsState.manageDirty = false;
        // Reload state from localStorage to discard in-memory edits
        loadRecipientsState();
        recipientsState.manageMode = false;
        renderRecipients();
    });
}
```

- [ ] **Step 4: Update Esc handler to route to dirty guard**

Locate the document-level keydown handler added in Task 5. Extend its `if (e.key !== 'Escape')` block:

```js
document.addEventListener('keydown', (e) => {
    if (e.key !== 'Escape') return;
    if (recipientsState.popoverOpen) {
        recipientsState.popoverOpen = false;
        renderPillPopover();
        document.getElementById('headerPill').focus();
        return;
    }
    if (recipientsState.manageMode) {
        if (recipientsState.manageDirty) {
            showDoneConfirm();
        } else {
            exitManageMode();
        }
        return;
    }
    if (recipientsState.expanded) {
        recipientsState.expanded = false;
        saveRecipientsState();
        renderRecipients();
    }
});
```

- [ ] **Step 5: Ensure `manageSaveBtn` clears dirty flag**

Locate the existing `manageSaveBtn` handler in `wireRecipientsEvents()`. After it commits state with `saveRecipientsState()`, add:
```js
recipientsState.manageDirty = false;
```

- [ ] **Step 6: Verify dirty guard**

```js
// Enter manage mode with a client
localStorage.setItem('definian_recipients', JSON.stringify({
  clients: [{ name: 'AWC', contacts: [{ name:'A', email:'a@x.com', role:'to', checked:true }] }],
  activeClientIndex: 0, expanded: true
}));
location.reload();
recipientsState.manageMode = true;
renderRecipients();

// Edit name → mark dirty
const nameInput = document.getElementById('manageClientName');
nameInput.value = 'AWC2';
nameInput.dispatchEvent(new Event('input'));
recipientsState.manageDirty  // expect: true

// Click Done → inline confirm appears
document.getElementById('recipientsDoneBtn').click();
({
  confirmShown: !!document.getElementById('confirmSaveClose'),  // expect: true
  saveCloseLabel: document.getElementById('confirmSaveClose').textContent.trim(),  // expect: contains "Save"
  discardLabel: document.getElementById('confirmDiscard').textContent.trim(),       // expect: "Discard"
})

// Click Discard
document.getElementById('confirmDiscard').click();
({
  manageMode: recipientsState.manageMode,   // expect: false
  nameAfterDiscard: recipientsState.clients[0].name,  // expect: "AWC" (original)
})
```

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(recipients): Done guard warns on unsaved manage-mode changes"
```

---

## Task 8: Preview Envelope Strip

**Files:**
- Modify: `index.html` — `.panel-preview` markup, new CSS for `.envelope-strip`, new `renderEnvelopeStrip()` function, calls from `renderRecipients()`

**Acceptance:** With an active client + contacts, a thin strip renders between `.subject-bar` and `.preview-container` showing `To: N · CC: N · Client`. With no clients, the strip is hidden. With a client + zero contacts, the strip shows `No recipients yet · ClientName`. Clicking the strip scrolls to and expands the Recipients section.

- [ ] **Step 1: Add envelope strip markup**

Inside `.panel-preview` (around line 1076), between `</div>` of `.subject-bar` and `<div class="preview-container">`, insert:
```html
<button type="button" class="envelope-strip" id="envelopeStrip" hidden aria-label="Recipients">
    <span class="envelope-strip-text"></span>
</button>
```

- [ ] **Step 2: Add `.envelope-strip` CSS**

Add to `<style>` block, near `.subject-bar` rules:

```css
.envelope-strip {
    display: block;
    width: 100%;
    background: var(--surface-white);
    border: none;
    border-top: 1px solid var(--border-default);
    border-bottom: 1px solid var(--border-default);
    padding: 7px 16px;
    font: 600 12.5px / 1 'DM Sans', 'Segoe UI', system-ui, sans-serif;
    color: var(--text-primary);
    text-align: left;
    cursor: pointer;
    font-variant-numeric: tabular-nums;
    transition: background-color 0.15s ease-out;
}
.envelope-strip:hover { background: var(--surface-bg); }
.envelope-strip[hidden] { display: none; }
.envelope-strip .label { color: var(--text-secondary); font-weight: 400; }
.envelope-strip .sep { color: var(--text-secondary); margin: 0 8px; }
.envelope-strip:focus-visible {
    outline: none;
    box-shadow: inset 0 0 0 3px rgba(13, 44, 113, 0.18);
}
```

- [ ] **Step 3: Add `renderEnvelopeStrip()` function**

Insert immediately after `renderHeaderPill()`:

```js
function renderEnvelopeStrip() {
    const strip = document.getElementById('envelopeStrip');
    if (!strip) return;
    const textEl = strip.querySelector('.envelope-strip-text');
    const clients = recipientsState.clients;
    if (clients.length === 0) {
        strip.hidden = true;
        return;
    }
    const client = clients[recipientsState.activeClientIndex];
    const name = client && client.name ? client.name : '(Untitled)';
    const toCount = client ? client.contacts.filter((c, i) => c.checked && effectiveRole(i) === 'to').length : 0;
    const ccCount = client ? client.contacts.filter((c, i) => c.checked && effectiveRole(i) === 'cc').length : 0;
    if (toCount + ccCount === 0) {
        strip.hidden = false;
        textEl.innerHTML = `<span class="label">No recipients yet</span><span class="sep">·</span>${escapeHtml(name)}`;
        strip.setAttribute('aria-label', `No recipients yet for ${name}. Click to manage.`);
        return;
    }
    strip.hidden = false;
    textEl.innerHTML = `<span class="label">To:</span> ${toCount}<span class="sep">·</span><span class="label">CC:</span> ${ccCount}<span class="sep">·</span>${escapeHtml(name)}`;
    strip.setAttribute('aria-label', `Recipients: ${toCount} to, ${ccCount} cc, ${name}. Click to manage.`);
}
```

- [ ] **Step 4: Wire strip click and call from render flow**

After the existing pill click wiring (added in Task 5), add:
```js
document.getElementById('envelopeStrip').addEventListener('click', () => {
    recipientsState.expanded = true;
    saveRecipientsState();
    renderRecipients();
    document.getElementById('recipientsSection').scrollIntoView({ behavior: 'smooth', block: 'start' });
});
```

Inside `renderRecipients()`, after `renderHeaderPill();`, add:
```js
renderEnvelopeStrip();
```

Also ensure strip renders on initial load. Below the existing `renderHeaderPill();` call near `loadRecipientsState();`, add:
```js
renderEnvelopeStrip();
```

- [ ] **Step 5: Verify strip state variants**

```js
// No clients
localStorage.setItem('definian_recipients', JSON.stringify({ clients: [], activeClientIndex: 0, expanded: false }));
location.reload();
document.getElementById('envelopeStrip').hidden  // expect: true

// Client with mixed roles
localStorage.setItem('definian_recipients', JSON.stringify({
  clients: [{ name: 'AWC', contacts: [
    { name:'A', email:'a@x.com', role:'to', checked:true },
    { name:'B', email:'b@x.com', role:'to', checked:true },
    { name:'C', email:'c@x.com', role:'cc', checked:true }
  ]}],
  activeClientIndex: 0, expanded: false
}));
location.reload();
({
  hidden: document.getElementById('envelopeStrip').hidden,                                // expect: false
  text: document.getElementById('envelopeStrip').textContent.replace(/\s+/g, ' ').trim(), // expect contains: "To: 2 · CC: 1 · AWC"
})

// Client with zero contacts
localStorage.setItem('definian_recipients', JSON.stringify({
  clients: [{ name: 'AWC', contacts: [] }], activeClientIndex: 0, expanded: false
}));
location.reload();
document.getElementById('envelopeStrip').textContent.replace(/\s+/g, ' ').trim()
// expect contains: "No recipients yet · AWC"
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(recipients): preview envelope strip shows To/CC counts"
```

---

## Task 9: Global Keyboard Shortcuts

**Files:**
- Modify: `index.html` — new global keydown handler installed once at script init

**Acceptance:** `Ctrl+Enter` (or `Cmd+Enter` on macOS) from any field triggers Copy Email. `Ctrl+Shift+C` triggers Copy Subject. `Ctrl+Shift+T` triggers Copy To recipients (no-op + warn toast if no client). `Ctrl+Shift+Y` triggers Copy CC. Shortcuts do not fire when focus is in a `<textarea>` for the Enter key (so multi-line input still works for Recon status text).

- [ ] **Step 1: Identify the existing copy entry points**

In DevTools console:
```js
({
  hasCopyEmail: typeof document.getElementById('copyEmail').click === 'function',
  hasCopySubject: typeof document.getElementById('copySubject').click === 'function',
  hasCopyRecipients: typeof copyRecipients === 'function',
})
```
Expected: all three `true`. Shortcuts will call these existing handlers, not duplicate logic.

- [ ] **Step 2: Add the global shortcuts handler**

After the existing document-level keydown handler (added in Task 5/7 for Escape), add a new listener:

```js
document.addEventListener('keydown', (e) => {
    const cmd = e.ctrlKey || e.metaKey;
    if (!cmd) return;

    // Ctrl+Enter → Copy Email (skip if focus is in textarea to allow newlines)
    if (e.key === 'Enter' && !e.shiftKey) {
        if (document.activeElement && document.activeElement.tagName === 'TEXTAREA') return;
        e.preventDefault();
        document.getElementById('copyEmail').click();
        return;
    }

    if (!e.shiftKey) return;

    if (e.key === 'C' || e.key === 'c') {
        e.preventDefault();
        document.getElementById('copySubject').click();
    } else if (e.key === 'T' || e.key === 't') {
        e.preventDefault();
        if (typeof copyRecipients === 'function') copyRecipients('to');
    } else if (e.key === 'Y' || e.key === 'y') {
        e.preventDefault();
        if (typeof copyRecipients === 'function') copyRecipients('cc');
    }
});
```

- [ ] **Step 3: Verify shortcuts**

In DevTools console:
```js
// Trigger Copy Email programmatically via keyboard
let copyEmailFired = false;
const orig = document.getElementById('copyEmail').onclick;
document.getElementById('copyEmail').addEventListener('click', () => { copyEmailFired = true; }, { once: true });
document.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', ctrlKey: true, bubbles: true }));
copyEmailFired  // expect: true
```

Also manually: focus an input, press Ctrl+Enter → toast should appear ("Email HTML copied!"). Press Ctrl+Shift+C → subject toast. Press Ctrl+Shift+T with a client active → To copied toast.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(shortcuts): Ctrl+Enter copies email; Ctrl+Shift+C/T/Y for subject and recipients"
```

---

## Task 10: Tablist Arrow-Key Navigation

**Files:**
- Modify: `index.html` — keydown handlers on `.mode-tabs` and `.status-toggle` containers

**Acceptance:** Focusing a mode tab and pressing `←` or `→` switches to the other tab and moves focus. Same for `.status-toggle` (Clean / Issues Found). Tab key still moves focus out of the tablist (standard tablist behavior).

- [ ] **Step 1: Locate the tab elements**

In DevTools console:
```js
({
  postedTab: document.getElementById('tabPostedFiles'),
  reconTab: document.getElementById('tabReconReport'),
  statusClean: document.getElementById('statusClean'),
  statusIssues: document.getElementById('statusIssues'),
})
```
Confirm all four are real elements.

- [ ] **Step 2: Add tablist keydown handlers**

After the global shortcuts handler from Task 9, add:

```js
function wireTablist(idA, idB) {
    const a = document.getElementById(idA);
    const b = document.getElementById(idB);
    if (!a || !b) return;
    [a, b].forEach(el => {
        el.addEventListener('keydown', (e) => {
            if (e.key !== 'ArrowLeft' && e.key !== 'ArrowRight') return;
            e.preventDefault();
            const other = el === a ? b : a;
            other.click();
            other.focus();
        });
    });
}
wireTablist('tabPostedFiles', 'tabReconReport');
wireTablist('statusClean', 'statusIssues');
```

- [ ] **Step 3: Verify arrow nav**

```js
// Focus posted tab, press right arrow
document.getElementById('tabPostedFiles').focus();
document.getElementById('tabPostedFiles').dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowRight', bubbles: true }));
activeMode  // expect: "recon"
document.activeElement.id  // expect: "tabReconReport"
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(a11y): arrow-key navigation on mode tabs and status toggle"
```

---

## Task 11: Help Pill (Shortcuts Reference)

**Files:**
- Modify: `index.html` — header markup, new CSS for `.recipients-help-pill` and its popover

**Acceptance:** A small `?` pill renders in the header chrome between the "Sending to:" pill and the V7 badge. Clicking it opens a small popover listing the keyboard shortcuts. Esc or outside-click closes it.

- [ ] **Step 1: Add help pill markup**

Inside `.header`, between the `header-pill-popover` div and the `header-badge` span, add:
```html
<button type="button" class="recipients-help-pill" id="helpPill" aria-haspopup="dialog" aria-expanded="false" title="Keyboard shortcuts">?</button>
<div class="recipients-help-popover" id="helpPopover" role="dialog" aria-label="Keyboard shortcuts" hidden>
    <h4>Keyboard Shortcuts</h4>
    <dl>
        <dt>Ctrl + Enter</dt><dd>Copy Email</dd>
        <dt>Ctrl + Shift + C</dt><dd>Copy Subject</dd>
        <dt>Ctrl + Shift + T</dt><dd>Copy To recipients</dd>
        <dt>Ctrl + Shift + Y</dt><dd>Copy CC recipients</dd>
        <dt>← / →</dt><dd>Switch tabs / toggle</dd>
        <dt>Space / Enter</dt><dd>Toggle Recipients</dd>
        <dt>Esc</dt><dd>Close popovers / manage</dd>
    </dl>
</div>
```

- [ ] **Step 2: Add help-pill CSS**

```css
.recipients-help-pill {
    background: rgba(255, 255, 255, 0.12);
    color: white;
    border: none;
    border-radius: 50%;
    width: 22px;
    height: 22px;
    font: 700 12px / 1 'DM Sans', system-ui, sans-serif;
    cursor: pointer;
    margin-right: 8px;
    transition: background-color 0.15s ease-out;
}
.recipients-help-pill:hover { background: rgba(255, 255, 255, 0.22); }
.recipients-help-pill:focus-visible {
    outline: none;
    box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.45);
}
.recipients-help-popover {
    position: absolute;
    top: 44px;
    right: 16px;
    z-index: 99;
    width: 280px;
    background: white;
    border-radius: 8px;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.18);
    padding: 14px 16px;
    color: var(--text-primary);
    font: 400 12.5px / 1.5 'DM Sans', system-ui, sans-serif;
}
.recipients-help-popover[hidden] { display: none; }
.recipients-help-popover h4 {
    margin: 0 0 8px 0;
    font-size: 12.5px;
    font-weight: 700;
}
.recipients-help-popover dl {
    margin: 0;
    display: grid;
    grid-template-columns: auto 1fr;
    gap: 4px 12px;
}
.recipients-help-popover dt {
    font-weight: 600;
    color: var(--text-primary);
    font-variant-numeric: tabular-nums;
}
.recipients-help-popover dd {
    margin: 0;
    color: var(--text-secondary);
}
```

- [ ] **Step 3: Wire help pill toggle**

Inside the script, near where the header pill click handler was wired in Task 5 (immediately after the `headerPill` click listener and before the `document.addEventListener('click', ...)` outside-click handler), add:
```js
document.getElementById('helpPill').addEventListener('click', (e) => {
    e.stopPropagation();
    const pop = document.getElementById('helpPopover');
    const wasOpen = !pop.hidden;
    pop.hidden = wasOpen;
    document.getElementById('helpPill').setAttribute('aria-expanded', wasOpen ? 'false' : 'true');
});

document.addEventListener('click', (e) => {
    const pop = document.getElementById('helpPopover');
    const pill = document.getElementById('helpPill');
    if (pop.hidden) return;
    if (pop.contains(e.target) || pill.contains(e.target)) return;
    pop.hidden = true;
    pill.setAttribute('aria-expanded', 'false');
});
```

Also extend the Esc handler (Task 7) to close the help popover. Add at the top of the Esc handler, before the popoverOpen check:
```js
const helpPop = document.getElementById('helpPopover');
if (helpPop && !helpPop.hidden) {
    helpPop.hidden = true;
    document.getElementById('helpPill').setAttribute('aria-expanded', 'false');
    document.getElementById('helpPill').focus();
    return;
}
```

- [ ] **Step 4: Verify help pill**

```js
document.getElementById('helpPill').click();
({
  popoverVisible: !document.getElementById('helpPopover').hidden,   // expect: true
  shortcutCount: document.querySelectorAll('#helpPopover dt').length,  // expect: 7
})
document.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));
document.getElementById('helpPopover').hidden  // expect: true
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(header): add ? pill with keyboard shortcuts reference"
```

---

## Task 12: Role-Pill aria-pressed (Final A11y Polish)

**Files:**
- Modify: `index.html` — both render paths that produce `.role-pill` (in `renderRecipients()` and `renderManageMode()`)

**Acceptance:** Every `.role-pill` button has `aria-pressed="true"` when role is `cc` and `aria-pressed="false"` when role is `to`. The attribute updates whenever the role flips.

- [ ] **Step 1: Capture current state**

```js
recipientsState.expanded = true;
recipientsState.manageMode = false;
renderRecipients();
[...document.querySelectorAll('.role-pill')].map(p => p.getAttribute('aria-pressed'))
// expect (before fix): array of nulls
```

- [ ] **Step 2: Add `aria-pressed` to expanded-view role pills**

In `renderRecipients()`, locate the contact row HTML template where `.role-pill` is constructed:
```js
<button type="button" class="role-pill role-pill--${role}" data-role-index="${i}" title="Click to flip To/CC for this email">${role === 'to' ? 'To' : 'CC'}</button>
```
Add `aria-pressed`:
```js
<button type="button" class="role-pill role-pill--${role}" data-role-index="${i}" aria-pressed="${role === 'cc' ? 'true' : 'false'}" title="Click to flip To/CC for this email">${role === 'to' ? 'To' : 'CC'}</button>
```

- [ ] **Step 3: Verify (manage-mode role pills already updated in Task 6)**

Confirm Task 6's `renderManageMode` template includes `aria-pressed="${role === 'cc' ? 'true' : 'false'}"` on the `.role-pill` button. If not (due to merge), add it now.

Test:
```js
renderRecipients();
[...document.querySelectorAll('.role-pill')].map(p => ({
  role: p.classList.contains('role-pill--cc') ? 'cc' : 'to',
  pressed: p.getAttribute('aria-pressed')
}))
// expect: every row where role is 'cc' has pressed: "true"; every 'to' has "false"
```

Also update the click handler that flips roles (look in `wireRecipientsEvents` for `data-role-index`). After mutation, the render re-runs via `renderRecipients()`, so the attribute is regenerated. No handler change needed.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(a11y): role-pill exposes aria-pressed state"
```

---

## Task 13: State Matrix Verification (Manual)

**Files:** none — pure verification

**Acceptance:** Every row of the spec's §7 state matrix renders correctly. Both tabs (Posted Files, Recon Report) behave identically with respect to chrome (pill, strip).

- [ ] **Step 1: Run the empty-state walkthrough**

```js
localStorage.removeItem('definian_recipients');
location.reload();
```
Verify:
- [ ] Pill reads `+ Add recipients` (dashed outline)
- [ ] Envelope strip is hidden
- [ ] Section badge says "No clients yet" message in expanded state
- [ ] Clicking pill enters manage mode for a fresh empty client, focuses the name field

- [ ] **Step 2: Run the bug repro scenario**

```js
localStorage.setItem('definian_recipients', JSON.stringify({
  clients: [
    { name: 'AWC', contacts: [
      { name:'Jason Lewis', email:'jaslewis@woodmark.com', role:'to', checked:true },
      { name:'Patty Sinnott', email:'psinnott@woodmark.com', role:'to', checked:true },
      { name:'Fyzia Shaik', email:'fyzia.shaik@definian.com', role:'cc', checked:true }
    ]},
    { name: 'New Jersey', contacts: [{ name:'Test', email:'t@nj.com', role:'to', checked:true }] }
  ],
  activeClientIndex: 0, expanded: true
}));
location.reload();
recipientsState.manageMode = true;
renderRecipients();
```
Verify:
- [ ] Rail shows AWC and New Jersey, AWC active (blue left-edge fill)
- [ ] Click `+ New client` — rail grows to 3 rows, new one is active and empty, AWC and New Jersey still visible
- [ ] Click AWC in the rail — editor switches back to AWC's contacts

- [ ] **Step 3: Wrong-client safety scenario**

With multiple clients seeded as above:
- [ ] Click pill, popover lists all
- [ ] Switch from AWC to New Jersey
- [ ] Verify pill text changes immediately, envelope strip text changes immediately, Recipients section badge changes
- [ ] Verify the email body preview (right pane) did NOT change

- [ ] **Step 4: Cross-mode parity**

- [ ] In Posted Files mode: pill + strip render correctly
- [ ] Click Recon Report tab: pill + strip still render identically
- [ ] Click Clean / Issues toggle: no effect on pill or strip

- [ ] **Step 5: Keyboard-only run**

Starting from a fresh page reload with AWC seeded:
- [ ] Tab into form, reach Recipients header, press Space → expands
- [ ] Tab to `Manage` button, press Enter → enters manage mode
- [ ] Tab to rail row, arrow keys navigate, Enter switches
- [ ] Press Esc → manage mode exits (or dirty confirm if edited)
- [ ] Press Ctrl+Enter from anywhere → Copy Email toast fires

- [ ] **Step 6: prefers-reduced-motion**

In DevTools, emulate `prefers-reduced-motion: reduce`:
```js
// In DevTools Rendering tab: Emulate CSS media feature prefers-reduced-motion: reduce
```
- [ ] Verify popover open is instant (no fade)
- [ ] Verify rail row hover state changes immediately

- [ ] **Step 7: localStorage migration sanity**

```js
// Old-shape state (no expanded key, no manageDirty)
localStorage.setItem('definian_recipients', JSON.stringify({
  clients: [{ name: 'AWC', contacts: [{ name:'A', email:'a@x.com', role:'to', checked:true }] }],
  activeClientIndex: 0
}));
location.reload();
// Verify no errors in console, pill renders, expanded defaults to false
recipientsState.expanded  // expect: false
recipientsState.manageDirty  // expect: false
```

- [ ] **Step 8: Final commit (verification record)**

If any issues surface during verification, fix them in the relevant earlier task's commits or create a follow-up commit:
```bash
git log --oneline -15
# Confirm clean history through Task 12.
```

No commit needed if verification passes cleanly. If a fixup is needed:
```bash
git add index.html
git commit -m "fix(recipients): <specific issue uncovered during verification>"
```

---

## Out of Scope (Tracked Separately)

The following are pending tasks that this plan does NOT address. They have separate workstreams in the task list:

- **#1** Differentiate twin segmented controls (`/impeccable distill`)
- **#4** Validate email format in manage mode (`/impeccable clarify`)
- **#5** Tighten font-size ladder (`/impeccable typeset`)
- **#6** Fix DESIGN.md self-violations: header gradient, toast radius, em dash (inline edits)
- **#9** Address minor observations: iPad portrait preview height, `--radius` vs `--radius-sm` consistency
- **#11** Final `/impeccable polish` pass

After Task 13, run `/impeccable critique` to confirm the heuristics score moved (target: 32→36+).
