# Posted Files Email Generator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file browser tool that generates Outlook-compatible HTML emails for Definian's conversion file posting notifications.

**Architecture:** Single `index.html` with embedded CSS and JS. One `generateEmailHTML()` function produces the complete Outlook-compatible XHTML string. Preview via `<iframe srcdoc>`. Clipboard via `ClipboardItem` API with `execCommand` fallback. Split-panel layout (form left, preview right) with 1024px responsive breakpoint.

**Tech Stack:** Vanilla HTML5, CSS3, JavaScript (ES2020+). Zero external dependencies. Works via `file://` protocol.

**Note:** This is a zero-dependency single HTML file — no test framework, no build step. Verification is manual: open in browser, check behavior. Each task includes specific verification steps.

---

## File Structure

- **Create:** `index.html` — the entire application (HTML structure, CSS styles, JavaScript logic)

That's it. One file. Supporting files (`README.md`, `.gitignore`, `CLAUDE.md`) already exist.

---

### Task 1: HTML Skeleton + Split-Panel CSS Layout

**Files:**
- Create: `index.html`

- [ ] **Step 1: Write the HTML document structure**

Create `index.html` with the full document skeleton: header bar, left panel (form area), right panel (preview area). No form fields yet — just the structural containers.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Posted Files Email Generator</title>
    <style>
        /* CSS Reset */
        *, *::before, *::after {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }

        :root {
            --definian-blue: #0D2C71;
            --definian-green: #00AB63;
            --bg-light: #f5f7fa;
            --bg-white: #ffffff;
            --text-primary: #1a1a2e;
            --text-secondary: #555555;
            --border-color: #d1d5db;
            --input-focus: #0D2C71;
        }

        html, body {
            height: 100%;
            font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
            background: var(--bg-light);
            color: var(--text-primary);
        }

        /* Header */
        .header {
            background: var(--definian-blue);
            color: #ffffff;
            padding: 12px 24px;
            font-size: 16px;
            font-weight: 600;
            letter-spacing: 0.3px;
            display: flex;
            align-items: center;
            gap: 10px;
        }

        .header-accent {
            width: 3px;
            height: 20px;
            background: var(--definian-green);
            border-radius: 2px;
        }

        /* Main Layout */
        .main {
            display: flex;
            height: calc(100vh - 48px);
            overflow: hidden;
        }

        .panel-form {
            width: 45%;
            padding: 24px;
            overflow-y: auto;
            border-right: 1px solid var(--border-color);
            background: var(--bg-white);
        }

        .panel-preview {
            width: 55%;
            display: flex;
            flex-direction: column;
            background: var(--bg-light);
        }

        /* Responsive: stack below 1024px */
        @media (max-width: 1024px) {
            .main {
                flex-direction: column;
                height: auto;
                overflow: auto;
            }
            .panel-form {
                width: 100%;
                border-right: none;
                border-bottom: 1px solid var(--border-color);
            }
            .panel-preview {
                width: 100%;
                min-height: 500px;
            }
        }
    </style>
</head>
<body>
    <div class="header">
        <div class="header-accent"></div>
        Posted Files Email Generator
    </div>
    <div class="main">
        <div class="panel-form" id="formPanel">
            <!-- Form fields go here (Task 2) -->
        </div>
        <div class="panel-preview" id="previewPanel">
            <!-- Subject line + preview iframe + copy button go here (Task 4) -->
        </div>
    </div>
    <script>
        // JavaScript goes here (Tasks 3-5)
    </script>
</body>
</html>
```

- [ ] **Step 2: Verify layout in browser**

Open `index.html` in Chrome. Verify:
- Blue header bar spans full width with green accent pip and white title text
- Two panels side by side — left ~45%, right ~55%
- Left panel has white background, right panel has light gray
- Resize window below 1024px — panels should stack vertically

---

### Task 2: Form Fields

**Files:**
- Modify: `index.html` (form panel section + add form CSS)

- [ ] **Step 1: Add form CSS styles**

Insert these styles before the closing `</style>` tag:

```css
/* Form Styles */
.form-group {
    margin-bottom: 20px;
}

.form-label {
    display: block;
    font-size: 13px;
    font-weight: 600;
    color: var(--text-primary);
    margin-bottom: 6px;
    text-transform: uppercase;
    letter-spacing: 0.5px;
}

.form-label-optional {
    font-weight: 400;
    text-transform: none;
    letter-spacing: 0;
    color: var(--text-secondary);
    font-size: 12px;
}

.form-input,
.form-select,
.form-textarea {
    width: 100%;
    padding: 10px 12px;
    font-size: 14px;
    font-family: inherit;
    border: 1px solid var(--border-color);
    border-radius: 6px;
    background: var(--bg-white);
    color: var(--text-primary);
    transition: border-color 0.15s, box-shadow 0.15s;
}

.form-input:focus,
.form-select:focus,
.form-textarea:focus {
    outline: none;
    border-color: var(--input-focus);
    box-shadow: 0 0 0 3px rgba(13, 44, 113, 0.1);
}

.form-textarea {
    resize: vertical;
    min-height: 80px;
}

/* File Location Rows */
.file-row {
    display: flex;
    gap: 8px;
    margin-bottom: 8px;
    align-items: center;
}

.file-row-number {
    font-size: 13px;
    font-weight: 600;
    color: var(--text-secondary);
    min-width: 20px;
    text-align: center;
}

.file-row-label {
    flex: 2;
}

.file-row-url {
    flex: 3;
}

.file-row .form-input {
    padding: 8px 10px;
    font-size: 13px;
}

.btn-remove-row {
    background: none;
    border: none;
    color: #999;
    cursor: pointer;
    font-size: 18px;
    padding: 4px 8px;
    border-radius: 4px;
    transition: color 0.15s, background 0.15s;
}

.btn-remove-row:hover {
    color: #e53e3e;
    background: #fee;
}

.btn-add-row {
    background: none;
    border: 1px dashed var(--border-color);
    color: var(--definian-blue);
    padding: 8px 16px;
    font-size: 13px;
    font-weight: 500;
    cursor: pointer;
    border-radius: 6px;
    transition: border-color 0.15s, background 0.15s;
    width: 100%;
}

.btn-add-row:hover {
    border-color: var(--definian-blue);
    background: rgba(13, 44, 113, 0.03);
}

/* Cycle custom input toggle */
.cycle-custom-toggle {
    font-size: 12px;
    color: var(--definian-blue);
    cursor: pointer;
    text-decoration: underline;
    margin-top: 4px;
    display: inline-block;
}
```

- [ ] **Step 2: Add form HTML**

Replace the `<!-- Form fields go here (Task 2) -->` comment inside `.panel-form` with:

```html
<div class="form-group">
    <label class="form-label" for="conversionName">Conversion Name</label>
    <input type="text" class="form-input" id="conversionName"
           placeholder="e.g., Item Master, Purchase Orders, On-Hand Balances">
</div>

<div class="form-group">
    <label class="form-label" for="cycle">Cycle</label>
    <div id="cycleContainer">
        <select class="form-select" id="cycleSelect">
            <option value="">Select cycle...</option>
            <option value="SIT1">SIT1</option>
            <option value="SIT2">SIT2</option>
            <option value="UAT">UAT</option>
            <option value="Mock 1">Mock 1</option>
            <option value="Mock 2">Mock 2</option>
            <option value="Mock 3">Mock 3</option>
            <option value="D-Cut">D-Cut</option>
            <option value="__other__">Other (custom)...</option>
        </select>
        <input type="text" class="form-input" id="cycleCustom"
               placeholder="Enter custom cycle name" style="display:none;">
        <span class="cycle-custom-toggle" id="cycleToggle" style="display:none;">Back to dropdown</span>
    </div>
</div>

<div class="form-group">
    <label class="form-label">Summary of Changes <span class="form-label-optional">(optional)</span></label>
    <textarea class="form-textarea" id="summaryChanges"
              placeholder="What changed from last posting, what's new, notable decisions..."></textarea>
</div>

<div class="form-group">
    <label class="form-label">File Locations</label>
    <div id="fileLocations">
        <div class="file-row" data-row="0">
            <span class="file-row-number">1.</span>
            <input type="text" class="form-input file-row-label" value="Conversion/Error Log" placeholder="Label">
            <input type="text" class="form-input file-row-url" placeholder="SharePoint URL">
        </div>
        <div class="file-row" data-row="1">
            <span class="file-row-number">2.</span>
            <input type="text" class="form-input file-row-label" value="Validation Files" placeholder="Label">
            <input type="text" class="form-input file-row-url" placeholder="SharePoint URL">
        </div>
        <div class="file-row" data-row="2">
            <span class="file-row-number">3.</span>
            <input type="text" class="form-input file-row-label" value="FBDI File(s)" placeholder="Label">
            <input type="text" class="form-input file-row-url" placeholder="SharePoint URL">
        </div>
    </div>
    <button type="button" class="btn-add-row" id="addFileRow">+ Add File Location</button>
</div>

<div class="form-group">
    <label class="form-label">Errors / Fallout Notes <span class="form-label-optional">(optional)</span></label>
    <textarea class="form-textarea" id="errorsFallout"
              placeholder="Missing items, known issues, records excluded..."></textarea>
</div>
```

- [ ] **Step 3: Add cycle toggle and file row JavaScript**

Add this JavaScript inside the `<script>` tag:

```javascript
// --- Cycle dropdown / custom toggle ---
const cycleSelect = document.getElementById('cycleSelect');
const cycleCustom = document.getElementById('cycleCustom');
const cycleToggle = document.getElementById('cycleToggle');

cycleSelect.addEventListener('change', () => {
    if (cycleSelect.value === '__other__') {
        cycleSelect.style.display = 'none';
        cycleCustom.style.display = '';
        cycleToggle.style.display = '';
        cycleCustom.focus();
    }
    updatePreview();
});

cycleToggle.addEventListener('click', () => {
    cycleSelect.value = '';
    cycleSelect.style.display = '';
    cycleCustom.style.display = 'none';
    cycleCustom.value = '';
    cycleToggle.style.display = 'none';
    updatePreview();
});

function getCycleValue() {
    if (cycleSelect.style.display === 'none') {
        return cycleCustom.value.trim();
    }
    const val = cycleSelect.value;
    return val === '__other__' ? '' : val;
}

// --- File location rows ---
const fileLocations = document.getElementById('fileLocations');
const addFileRowBtn = document.getElementById('addFileRow');

function renumberFileRows() {
    const rows = fileLocations.querySelectorAll('.file-row');
    rows.forEach((row, i) => {
        row.querySelector('.file-row-number').textContent = (i + 1) + '.';
        // Show remove button only if more than 1 row
        const removeBtn = row.querySelector('.btn-remove-row');
        if (removeBtn) {
            removeBtn.style.display = rows.length > 1 ? '' : 'none';
        }
    });
}

function createFileRow(label = '', url = '') {
    const row = document.createElement('div');
    row.className = 'file-row';
    row.innerHTML = `
        <span class="file-row-number"></span>
        <input type="text" class="form-input file-row-label" value="${label}" placeholder="Label">
        <input type="text" class="form-input file-row-url" value="${url}" placeholder="SharePoint URL">
        <button type="button" class="btn-remove-row" title="Remove row">&times;</button>
    `;
    row.querySelector('.btn-remove-row').addEventListener('click', () => {
        row.remove();
        renumberFileRows();
        updatePreview();
    });
    row.querySelectorAll('input').forEach(input => {
        input.addEventListener('input', updatePreview);
    });
    return row;
}

addFileRowBtn.addEventListener('click', () => {
    fileLocations.appendChild(createFileRow());
    renumberFileRows();
    // Focus the new label input
    const rows = fileLocations.querySelectorAll('.file-row');
    rows[rows.length - 1].querySelector('.file-row-label').focus();
    updatePreview();
});

// Add remove buttons to the initial 3 rows
fileLocations.querySelectorAll('.file-row').forEach(row => {
    const removeBtn = document.createElement('button');
    removeBtn.type = 'button';
    removeBtn.className = 'btn-remove-row';
    removeBtn.title = 'Remove row';
    removeBtn.innerHTML = '&times;';
    removeBtn.addEventListener('click', () => {
        row.remove();
        renumberFileRows();
        updatePreview();
    });
    row.appendChild(removeBtn);
});
renumberFileRows();

// --- Wire up all inputs to live preview ---
document.querySelectorAll('#conversionName, #summaryChanges, #errorsFallout, #cycleCustom').forEach(el => {
    el.addEventListener('input', updatePreview);
});
fileLocations.querySelectorAll('input').forEach(el => {
    el.addEventListener('input', updatePreview);
});

function updatePreview() {
    // Placeholder — implemented in Task 4
}
```

- [ ] **Step 4: Verify form in browser**

Open `index.html` in Chrome. Verify:
- All five field groups render with labels
- Cycle dropdown shows all options; selecting "Other" swaps to text input; "Back to dropdown" restores it
- File location rows show 3 pre-labeled rows with remove buttons
- "Add File Location" button appends new rows with auto-numbering
- Removing a row renumbers remaining rows
- Cannot remove the last remaining row (remove button hidden)

---

### Task 3: Email Template Generation Function

**Files:**
- Modify: `index.html` (add `generateEmailHTML()` function in script)

- [ ] **Step 1: Write the generateEmailHTML function**

Add this function in the `<script>` tag, before the `updatePreview()` placeholder:

```javascript
function getFileLocations() {
    const rows = fileLocations.querySelectorAll('.file-row');
    const locations = [];
    rows.forEach(row => {
        const label = row.querySelector('.file-row-label').value.trim();
        const url = row.querySelector('.file-row-url').value.trim();
        if (label || url) {
            locations.push({ label: label || 'Untitled', url });
        }
    });
    return locations;
}

function escapeHtml(str) {
    const div = document.createElement('div');
    div.textContent = str;
    return div.innerHTML;
}

function generateSubjectLine() {
    const name = document.getElementById('conversionName').value.trim();
    const cycle = getCycleValue();
    const namePart = name || '[Conversion Name]';
    const cyclePart = cycle || '[Cycle]';
    return `Wave 2C ${namePart} Files Posted - ${cyclePart}`;
}

function generateEmailHTML() {
    const name = document.getElementById('conversionName').value.trim();
    const cycle = getCycleValue();
    const summary = document.getElementById('summaryChanges').value.trim();
    const errors = document.getElementById('errorsFallout').value.trim();
    const files = getFileLocations();

    const namePart = escapeHtml(name || '[Conversion Name]');
    const cyclePart = escapeHtml(cycle || '[Cycle]');

    const tdStyle = 'font-family: Arial, sans-serif; font-size: 14px; color: #333333; line-height: 1.5;';
    const linkStyle = 'color: #1a73e8; text-decoration: underline; font-family: Arial, sans-serif; font-size: 14px;';
    const headerStyle = 'font-family: Arial, sans-serif; font-size: 15px; font-weight: bold; color: #0D2C71;';

    // Build file locations HTML
    let filesHtml = '';
    files.forEach((file, i) => {
        const labelHtml = escapeHtml(file.label);
        const linkOrText = file.url
            ? `<a href="${escapeHtml(file.url)}" style="${linkStyle}">${labelHtml}</a>`
            : `<span style="font-family: Arial, sans-serif; font-size: 14px; color: #333333;">${labelHtml}</span>`;
        filesHtml += `<tr>
            <td style="${tdStyle} padding: 3px 0;">${i + 1}.&nbsp;&nbsp;${linkOrText}</td>
        </tr>`;
    });

    // Build summary section (omit entirely if empty)
    let summarySection = '';
    if (summary) {
        const summaryLines = escapeHtml(summary).replace(/\n/g, '<br />');
        summarySection = `
        <tr><td height="16" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>
        <tr>
            <td style="${headerStyle} padding: 0;">Summary of Changes</td>
        </tr>
        <tr><td height="8" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>
        <tr>
            <td style="${tdStyle} padding: 0;">${summaryLines}</td>
        </tr>`;
    }

    // Build errors/fallout section (omit entirely if empty)
    let errorsSection = '';
    if (errors) {
        const errorsLines = escapeHtml(errors).replace(/\n/g, '<br />');
        errorsSection = `
        <tr><td height="16" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>
        <tr>
            <td style="${headerStyle} padding: 0;">Errors / Fallout Notes</td>
        </tr>
        <tr><td height="8" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>
        <tr>
            <td style="padding: 0;">
                <table width="100%" cellpadding="0" cellspacing="0" border="0" role="presentation">
                    <tr>
                        <td width="3" bgcolor="#0D2C71" style="font-size: 1px; line-height: 1px;">&nbsp;</td>
                        <td width="12" style="font-size: 1px; line-height: 1px;">&nbsp;</td>
                        <td bgcolor="#f0f4fa" style="${tdStyle} padding: 10px 12px;">
                            ${errorsLines}
                        </td>
                    </tr>
                </table>
            </td>
        </tr>`;
    }

    return `<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>${escapeHtml(generateSubjectLine())}</title>
</head>
<body style="margin: 0; padding: 0; background-color: #ffffff;">
    <table width="600" align="center" cellpadding="0" cellspacing="0" border="0" role="presentation" style="margin: 0 auto;">
        <!-- Header accent bar -->
        <tr>
            <td bgcolor="#0D2C71" height="4" style="font-size: 1px; line-height: 1px;">&nbsp;</td>
        </tr>
        <tr><td height="20" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>

        <!-- Greeting -->
        <tr>
            <td style="${tdStyle} padding: 0;">Hi All,</td>
        </tr>
        <tr><td height="12" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>

        <!-- Status line -->
        <tr>
            <td style="${tdStyle} padding: 0;">Wave 2C ${namePart} ${cyclePart} files have been posted to SharePoint for pre-load validation.</td>
        </tr>

        <!-- Summary of Changes (conditional) -->
        ${summarySection}

        <!-- File Locations -->
        <tr><td height="16" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>
        <tr>
            <td style="${headerStyle} padding: 0;">File Locations</td>
        </tr>
        <tr><td height="8" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>
        <tr>
            <td style="padding: 0;">
                <table width="100%" cellpadding="0" cellspacing="0" border="0" role="presentation">
                    ${filesHtml}
                </table>
            </td>
        </tr>

        <!-- Errors / Fallout (conditional) -->
        ${errorsSection}

        <!-- Closing -->
        <tr><td height="20" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>
        <tr>
            <td style="${tdStyle} padding: 0;">Thank You.</td>
        </tr>
        <tr><td height="8" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>
    </table>
</body>
</html>`;
}
```

- [ ] **Step 2: Verify function produces valid output**

Open browser console, type `generateEmailHTML()` and verify:
- Returns a string starting with the XHTML Transitional doctype
- Contains `<table width="600" align="center">`
- Contains no CSS classes, no `<style>` blocks, no `<div>` elements
- All styles are inline
- With empty Summary and Errors fields, those sections are completely absent from output

---

### Task 4: Preview Panel + Live Preview Wiring

**Files:**
- Modify: `index.html` (preview panel HTML, preview CSS, update `updatePreview()`)

- [ ] **Step 1: Add preview panel CSS**

Insert before the closing `</style>` tag:

```css
/* Preview Panel */
.subject-bar {
    padding: 16px 20px;
    background: var(--bg-white);
    border-bottom: 1px solid var(--border-color);
}

.subject-label {
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: var(--text-secondary);
    margin-bottom: 6px;
}

.subject-line {
    display: flex;
    align-items: center;
    gap: 10px;
}

.subject-text {
    flex: 1;
    font-size: 14px;
    font-weight: 600;
    color: var(--text-primary);
    word-break: break-word;
}

.btn-copy {
    background: var(--definian-blue);
    color: #ffffff;
    border: none;
    padding: 8px 16px;
    font-size: 13px;
    font-weight: 500;
    border-radius: 6px;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.15s, transform 0.1s;
}

.btn-copy:hover {
    background: #0a2259;
}

.btn-copy:active {
    transform: scale(0.97);
}

.preview-frame-container {
    flex: 1;
    padding: 20px;
    overflow: auto;
}

.preview-frame {
    width: 100%;
    height: 100%;
    min-height: 400px;
    border: 1px solid var(--border-color);
    border-radius: 6px;
    background: #ffffff;
}

.copy-bar {
    padding: 12px 20px;
    background: var(--bg-white);
    border-top: 1px solid var(--border-color);
    display: flex;
    justify-content: center;
}

.btn-copy-email {
    background: var(--definian-blue);
    color: #ffffff;
    border: none;
    padding: 12px 32px;
    font-size: 14px;
    font-weight: 600;
    border-radius: 6px;
    cursor: pointer;
    transition: background 0.15s, transform 0.1s;
    width: 100%;
    max-width: 400px;
}

.btn-copy-email:hover {
    background: #0a2259;
}

.btn-copy-email:active {
    transform: scale(0.98);
}

/* Toast */
.toast {
    position: fixed;
    bottom: 24px;
    left: 50%;
    transform: translateX(-50%) translateY(20px);
    background: var(--definian-green);
    color: #ffffff;
    padding: 10px 24px;
    border-radius: 8px;
    font-size: 14px;
    font-weight: 500;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.3s, transform 0.3s;
    z-index: 1000;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.toast.show {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
}
```

- [ ] **Step 2: Add preview panel HTML**

Replace `<!-- Subject line + preview iframe + copy button go here (Task 4) -->` with:

```html
<div class="subject-bar">
    <div class="subject-label">Subject Line</div>
    <div class="subject-line">
        <span class="subject-text" id="subjectText">Wave 2C [Conversion Name] Files Posted - [Cycle]</span>
        <button type="button" class="btn-copy" id="copySubject">Copy Subject</button>
    </div>
</div>
<div class="preview-frame-container">
    <iframe class="preview-frame" id="previewFrame" sandbox="allow-same-origin" title="Email preview"></iframe>
</div>
<div class="copy-bar">
    <button type="button" class="btn-copy-email" id="copyEmail">Copy Email HTML</button>
</div>

<!-- Toast notification -->
<div class="toast" id="toast">Copied!</div>
```

- [ ] **Step 3: Implement updatePreview()**

Replace the `function updatePreview() { // Placeholder }` with:

```javascript
function updatePreview() {
    const subject = generateSubjectLine();
    document.getElementById('subjectText').textContent = subject;

    const html = generateEmailHTML();
    const iframe = document.getElementById('previewFrame');
    iframe.srcdoc = html;
}

// Initial render
updatePreview();
```

- [ ] **Step 4: Verify live preview**

Open `index.html` in Chrome. Verify:
- Preview iframe shows the email with default placeholder text
- Subject line bar shows "Wave 2C [Conversion Name] Files Posted - [Cycle]"
- Typing in Conversion Name updates both subject line and email body in real-time
- Selecting a cycle updates both subject line and email body
- Typing in Summary of Changes shows the section in the preview; clearing it removes it completely
- Typing in Errors/Fallout shows the callout section; clearing it removes it completely
- Adding/removing file location rows updates the preview

---

### Task 5: Clipboard Functionality + Toast

**Files:**
- Modify: `index.html` (add clipboard JS)

- [ ] **Step 1: Add toast and clipboard functions**

Add this JavaScript after the `updatePreview()` function:

```javascript
// --- Toast notification ---
function showToast(message = 'Copied!') {
    const toast = document.getElementById('toast');
    toast.textContent = message;
    toast.classList.add('show');
    setTimeout(() => toast.classList.remove('show'), 2000);
}

// --- Clipboard: copy subject line ---
document.getElementById('copySubject').addEventListener('click', async () => {
    const subject = generateSubjectLine();
    try {
        await navigator.clipboard.writeText(subject);
        showToast('Subject copied!');
    } catch {
        fallbackCopyText(subject);
    }
});

function fallbackCopyText(text) {
    const textarea = document.createElement('textarea');
    textarea.value = text;
    textarea.style.position = 'fixed';
    textarea.style.opacity = '0';
    document.body.appendChild(textarea);
    textarea.select();
    try {
        document.execCommand('copy');
        showToast('Subject copied!');
    } catch {
        showToast('Copy failed — select and Ctrl+C');
    }
    document.body.removeChild(textarea);
}

// --- Clipboard: copy email HTML ---
document.getElementById('copyEmail').addEventListener('click', async () => {
    const html = generateEmailHTML();
    try {
        const blob = new Blob([html], { type: 'text/html' });
        const item = new ClipboardItem({ 'text/html': blob });
        await navigator.clipboard.write([item]);
        showToast('Email HTML copied!');
    } catch {
        fallbackCopyHtml(html);
    }
});

function fallbackCopyHtml(html) {
    const container = document.createElement('div');
    container.innerHTML = html;
    container.style.position = 'fixed';
    container.style.opacity = '0';
    container.style.pointerEvents = 'none';
    document.body.appendChild(container);

    const range = document.createRange();
    range.selectNodeContents(container);
    const selection = window.getSelection();
    selection.removeAllRanges();
    selection.addRange(range);

    try {
        document.execCommand('copy');
        showToast('Email HTML copied!');
    } catch {
        showToast('Copy failed — try Ctrl+C from preview');
    }

    selection.removeAllRanges();
    document.body.removeChild(container);
}
```

- [ ] **Step 2: Verify clipboard functionality**

Open `index.html` in Chrome (must be `http://` or `https://` for ClipboardItem — use a simple local server: `python -m http.server 8000` or similar). Verify:
- "Copy Subject" copies plain text to clipboard, shows green toast "Subject copied!" for ~2 seconds
- "Copy Email HTML" copies rich HTML to clipboard, shows green toast "Email HTML copied!"
- Paste the email HTML into an Outlook compose window — verify tables, colors, links render correctly
- Also test by opening via `file://` — the fallback method should work

---

### Task 6: Polish + Final Verification

**Files:**
- Modify: `index.html` (minor polish)

- [ ] **Step 1: Invoke the humanizer skill**

Run `/humanizer-skill:humanizer` on the email template text content (greeting, status line, section headers, closing) to ensure natural tone.

- [ ] **Step 2: Invoke the frontend-design skill**

Run the `frontend-design` skill to review and refine the generator UI for polish and professional quality.

- [ ] **Step 3: Invoke the verification-before-completion skill**

Run through the full verification checklist from the handoff document.

- [ ] **Step 4: Commit all files**

Use `commit-commands:commit` to commit with a descriptive initial commit message.
