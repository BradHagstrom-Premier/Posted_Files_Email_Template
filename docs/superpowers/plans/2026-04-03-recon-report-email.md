# Recon Report Email Mode — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Recon Report email mode to the existing Posted Files Email Generator, with tab-based switching, a Clean/Issues Found status toggle, and a UI refresh using DM Sans.

**Architecture:** All changes are in a single file (`index.html`). A JS `activeMode` variable drives which form fields are visible and which email generation path runs. Conversion Name and Cycle are shared across both modes. All new CSS goes in the existing `<style>` block; all new JS goes in the existing `<script>` block.

**Tech Stack:** Vanilla HTML/CSS/JS, no build step, no dependencies. DM Sans via Google Fonts link tag (degrades to system-ui offline).

---

### Task 1: Add DM Sans font link + all new CSS

**Files:**
- Modify: `index.html` — `<head>` and `<style>` block

- [ ] **Step 1: Open `index.html` in a browser and note the current font (Segoe UI) and that the form header says "Email Details"**

  This is your pre-change baseline.

- [ ] **Step 2: Add the DM Sans Google Fonts link tag**

  Find this line in `<head>` (line ~6):
  ```html
      <title>Posted Files Email Generator</title>
  ```
  Insert after it:
  ```html
      <link rel="preconnect" href="https://fonts.googleapis.com">
      <link href="https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">
  ```

- [ ] **Step 3: Update the body font stack**

  Find (line ~39):
  ```css
          font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
  ```
  Replace with:
  ```css
          font-family: 'DM Sans', 'Segoe UI', system-ui, -apple-system, sans-serif;
  ```

- [ ] **Step 4: Add CSS for mode tabs, status toggle, and field fade animation**

  Find the existing `@keyframes rowSlideIn` block (around line ~253). Insert the following CSS block **after** the closing `}` of `@keyframes rowSlideIn`:

  ```css
        /* ───────────────────────────────────────────
           Mode Tabs
           ─────────────────────────────────────────── */
        .mode-tabs {
            display: flex;
            gap: 2px;
            margin-bottom: 24px;
            background: var(--blue-muted);
            border-radius: var(--radius);
            padding: 3px;
        }

        .mode-tab {
            flex: 1;
            padding: 8px 12px;
            font-size: 12.5px;
            font-weight: 600;
            text-align: center;
            cursor: pointer;
            border: none;
            border-radius: 6px;
            background: transparent;
            color: var(--text-secondary);
            transition: background var(--transition), color var(--transition), box-shadow var(--transition);
            font-family: inherit;
            letter-spacing: 0.2px;
        }

        .mode-tab.active {
            background: var(--blue);
            color: #ffffff;
            box-shadow: var(--shadow-sm);
        }

        .mode-tab:hover:not(.active) {
            background: rgba(13, 44, 113, 0.12);
            color: var(--blue);
        }

        /* ───────────────────────────────────────────
           Mode Fields Transition
           ─────────────────────────────────────────── */
        .mode-fields {
            animation: fieldsFadeIn 0.12s ease;
        }

        @keyframes fieldsFadeIn {
            from { opacity: 0; transform: translateY(6px); }
            to   { opacity: 1; transform: translateY(0); }
        }

        /* ───────────────────────────────────────────
           Status Toggle (Clean / Issues Found)
           ─────────────────────────────────────────── */
        .status-toggle {
            display: flex;
            gap: 2px;
            background: var(--blue-muted);
            border-radius: var(--radius);
            padding: 3px;
            margin-bottom: 14px;
        }

        .status-toggle-btn {
            flex: 1;
            padding: 8px 12px;
            font-size: 12.5px;
            font-weight: 600;
            text-align: center;
            cursor: pointer;
            border: none;
            border-radius: 6px;
            background: transparent;
            color: var(--text-secondary);
            transition: background var(--transition), color var(--transition), box-shadow var(--transition);
            font-family: inherit;
            letter-spacing: 0.2px;
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 6px;
        }

        .status-toggle-btn.active-clean {
            background: var(--green);
            color: #ffffff;
            box-shadow: var(--shadow-sm);
        }

        .status-toggle-btn.active-issues {
            background: var(--blue);
            color: #ffffff;
            box-shadow: var(--shadow-sm);
        }

        .status-toggle-btn:hover:not(.active-clean):not(.active-issues) {
            background: rgba(13, 44, 113, 0.12);
            color: var(--blue);
        }
  ```

- [ ] **Step 5: Verify in browser**

  Reload the page. Font should look slightly crisper (DM Sans loads if online). No visible layout change yet — no tabs in the HTML yet. No JS errors in the console.

- [ ] **Step 6: Commit**

  ```bash
  git add index.html
  git commit -m "Add DM Sans font and CSS for mode tabs and status toggle"
  ```

---

### Task 2: Restructure the form HTML

**Files:**
- Modify: `index.html` — form panel HTML (lines ~527–601)

- [ ] **Step 1: Replace the "Email Details" section title with the tab strip**

  Find (line ~528):
  ```html
              <div class="form-section-title">Email Details</div>
  ```
  Replace with:
  ```html
              <!-- Mode Tabs -->
              <div class="mode-tabs">
                  <button type="button" class="mode-tab active" id="tabPostedFiles">Posted Files</button>
                  <button type="button" class="mode-tab" id="tabReconReport">Recon Report</button>
              </div>
  ```

- [ ] **Step 2: Wrap the Posted Files-specific fields in a container div**

  The shared fields (Conversion Name and Cycle) stay unwrapped — they always show. The mode-specific fields are Summary of Changes, File Locations, and Errors / Fallout Notes.

  Find (line ~554):
  ```html
              <div class="form-group">
                  <label class="form-label">
                      Summary of Changes
  ```
  Insert **before** it:
  ```html
              <!-- Posted Files specific fields -->
              <div id="postedFilesFields" class="mode-fields">
  ```

  Then find the closing `</div>` of the Errors / Fallout Notes form-group (line ~602, the `</div>` that closes the last `.form-group` before the `</div>` that closes `.panel-form`):
  ```html
              </div>

          </div>
  ```
  The last `</div>` that closes `.panel-form` is preceded by the end of the errors textarea group. Insert a closing `</div>` to close `#postedFilesFields` just before the panel-form closes. The result should look like:
  ```html
                  </textarea>
              </div>

              </div><!-- /#postedFilesFields -->

          </div><!-- /.panel-form -->
  ```

- [ ] **Step 3: Add the Recon Report fields after `#postedFilesFields`**

  Insert the following block immediately after the `</div><!-- /#postedFilesFields -->` closing tag, still inside `.panel-form`:

  ```html
              <!-- Recon Report specific fields -->
              <div id="reconFields" class="mode-fields" style="display:none;">

                  <div class="form-group">
                      <label class="form-label">Recon Report Location</label>
                      <div class="file-rows-header">
                          <span class="file-rows-header-label">Label</span>
                          <span class="file-rows-header-url">SharePoint URL</span>
                      </div>
                      <div class="file-row">
                          <span class="file-row-num" style="visibility:hidden;">1.</span>
                          <input type="text" class="form-input file-row-label" id="reconReportLabel" value="Premier Recon Reports" placeholder="Label">
                          <input type="text" class="form-input file-row-url" id="reconReportUrl" placeholder="SharePoint URL (optional)">
                      </div>
                  </div>

                  <div class="form-group">
                      <label class="form-label">Status</label>
                      <div class="status-toggle">
                          <button type="button" class="status-toggle-btn active-clean" id="statusClean">&#10003; Clean</button>
                          <button type="button" class="status-toggle-btn" id="statusIssues">&#9873; Issues Found</button>
                      </div>
                      <textarea class="form-textarea" id="reconStatusText" style="min-height:64px;" placeholder="Describe discrepancies...">All records loaded as expected. No discrepancies found.</textarea>
                  </div>

              </div><!-- /#reconFields -->
  ```

- [ ] **Step 4: Verify in browser**

  Reload. You should see:
  - Two tabs at top of form: "Posted Files" (active, blue) and "Recon Report"
  - Posted Files fields visible as before
  - Clicking "Recon Report" tab does nothing yet (no JS) — that's expected
  - No JS errors in console

- [ ] **Step 5: Commit**

  ```bash
  git add index.html
  git commit -m "Restructure form HTML for tab-based mode switching"
  ```

---

### Task 3: Add JS mode switching

**Files:**
- Modify: `index.html` — `<script>` block

- [ ] **Step 1: Add the mode switching JS block**

  Find the comment at the very top of the `<script>` block (line ~627):
  ```javascript
      /* ═══════════════════════════════════════════════
         Cycle Dropdown / Custom Toggle
  ```
  Insert the following block **before** it (as the first thing in the script):

  ```javascript
      /* ═══════════════════════════════════════════════
         Mode Tabs
         ═══════════════════════════════════════════════ */
      let activeMode = 'posted'; // 'posted' | 'recon'

      const tabPostedFiles = document.getElementById('tabPostedFiles');
      const tabReconReport = document.getElementById('tabReconReport');
      const postedFilesFields = document.getElementById('postedFilesFields');
      const reconFields = document.getElementById('reconFields');

      tabPostedFiles.addEventListener('click', () => {
          if (activeMode === 'posted') return;
          activeMode = 'posted';
          tabPostedFiles.classList.add('active');
          tabReconReport.classList.remove('active');
          postedFilesFields.style.display = '';
          reconFields.style.display = 'none';
          postedFilesFields.classList.remove('mode-fields');
          void postedFilesFields.offsetWidth; // force reflow to re-trigger animation
          postedFilesFields.classList.add('mode-fields');
          updatePreview();
      });

      tabReconReport.addEventListener('click', () => {
          if (activeMode === 'recon') return;
          activeMode = 'recon';
          tabReconReport.classList.add('active');
          tabPostedFiles.classList.remove('active');
          reconFields.style.display = '';
          postedFilesFields.style.display = 'none';
          reconFields.classList.remove('mode-fields');
          void reconFields.offsetWidth; // force reflow to re-trigger animation
          reconFields.classList.add('mode-fields');
          updatePreview();
      });

  ```

- [ ] **Step 2: Verify in browser**

  Reload. Clicking "Recon Report" tab should:
  - Switch active tab styling (blue active state)
  - Hide the Posted Files fields (Summary, File Locations, Errors/Fallout)
  - Show the Recon fields (Recon Report Location row + Status toggle)
  - Fade+slide animation plays on the incoming fields
  - Conversion Name and Cycle remain visible in both modes
  - Preview updates (will show the Posted Files email still — recon email generation comes next)

  Clicking "Posted Files" tab should restore the original form.

- [ ] **Step 3: Commit**

  ```bash
  git add index.html
  git commit -m "Add JS mode switching for Posted Files / Recon Report tabs"
  ```

---

### Task 4: Add JS for status toggle and recon input wiring

**Files:**
- Modify: `index.html` — `<script>` block

- [ ] **Step 1: Add the status toggle JS block**

  Find the comment (from Task 3, now in the file):
  ```javascript
      /* ═══════════════════════════════════════════════
         Cycle Dropdown / Custom Toggle
  ```
  Insert the following block **before** it:

  ```javascript
      /* ═══════════════════════════════════════════════
         Recon Status Toggle
         ═══════════════════════════════════════════════ */
      let reconStatus = 'clean'; // 'clean' | 'issues'

      const statusCleanBtn = document.getElementById('statusClean');
      const statusIssuesBtn = document.getElementById('statusIssues');
      const reconStatusText = document.getElementById('reconStatusText');

      statusCleanBtn.addEventListener('click', () => {
          if (reconStatus === 'clean') return;
          reconStatus = 'clean';
          statusCleanBtn.classList.add('active-clean');
          statusCleanBtn.classList.remove('active-issues');
          statusIssuesBtn.classList.remove('active-clean', 'active-issues');
          reconStatusText.value = 'All records loaded as expected. No discrepancies found.';
          reconStatusText.placeholder = '';
          updatePreview();
      });

      statusIssuesBtn.addEventListener('click', () => {
          if (reconStatus === 'issues') return;
          reconStatus = 'issues';
          statusIssuesBtn.classList.add('active-issues');
          statusIssuesBtn.classList.remove('active-clean');
          statusCleanBtn.classList.remove('active-clean', 'active-issues');
          if (reconStatusText.value === 'All records loaded as expected. No discrepancies found.') {
              reconStatusText.value = '';
          }
          reconStatusText.placeholder = 'Describe the discrepancies...';
          updatePreview();
      });

      document.getElementById('reconReportLabel').addEventListener('input', updatePreview);
      document.getElementById('reconReportUrl').addEventListener('input', updatePreview);
      reconStatusText.addEventListener('input', updatePreview);

  ```

- [ ] **Step 2: Verify in browser**

  Switch to the Recon Report tab. Then:
  - "Clean" button should be green (active). Textarea shows "All records loaded as expected. No discrepancies found."
  - Click "Issues Found" — button turns blue, textarea clears with placeholder "Describe the discrepancies..."
  - Click "Clean" again — textarea refills with the clean message
  - Typing in the Recon Report Location or status textarea should trigger `updatePreview()` (email preview re-renders)

- [ ] **Step 3: Commit**

  ```bash
  git add index.html
  git commit -m "Add recon status toggle JS and input event wiring"
  ```

---

### Task 5: Add recon email generation functions

**Files:**
- Modify: `index.html` — `<script>` block, email generation section

- [ ] **Step 1: Add the three recon generation functions**

  Find the comment (line ~718 approximately):
  ```javascript
      /* ═══════════════════════════════════════════════
         Email HTML Generation
         ═══════════════════════════════════════════════ */
  ```
  Insert the following block **before** it:

  ```javascript
      /* ═══════════════════════════════════════════════
         Recon Email Generation
         ═══════════════════════════════════════════════ */
      function generateReconSubjectLine() {
          const name = document.getElementById('conversionName').value.trim();
          const cycle = getCycleValue();
          return 'Wave 2C ' + (name || '[Conversion Name]') + ' Recon Report - ' + (cycle || '[Cycle]');
      }

      function generateReconFileLocationsHTML() {
          const label = document.getElementById('reconReportLabel').value.trim() || 'Premier Recon Reports';
          const url = document.getElementById('reconReportUrl').value.trim();
          const td = 'font-family: Arial, sans-serif; font-size: 14px; color: #333333; line-height: 1.6;';
          const link = 'color: #1a73e8; text-decoration: underline; font-family: Arial, sans-serif; font-size: 14px;';
          const sectionHead = 'font-family: Arial, sans-serif; font-size: 15px; font-weight: bold; color: #0D2C71;';
          const smallSpacer = '<tr><td height="8" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>';

          const content = url
              ? '<a href="' + escapeHtml(url) + '" style="' + link + '">' + escapeHtml(label) + '</a>'
              : '<span style="' + td + '">' + escapeHtml(label) + '</span>';

          return '<table cellpadding="0" cellspacing="0" border="0" role="presentation">' +
              '<tr><td style="' + sectionHead + ' padding: 0;">Recon Report Location</td></tr>' +
              smallSpacer +
              '<tr><td style="' + td + ' padding: 3px 0;">' + content + '</td></tr>' +
              '</table>';
      }

      function generateReconEmailHTML() {
          const name = document.getElementById('conversionName').value.trim();
          const label = document.getElementById('reconReportLabel').value.trim() || 'Premier Recon Reports';
          const url = document.getElementById('reconReportUrl').value.trim();
          const statusText = document.getElementById('reconStatusText').value.trim();

          const namePart = escapeHtml(name || '[Conversion Name]');
          const td = 'font-family: Arial, sans-serif; font-size: 14px; color: #333333; line-height: 1.6;';
          const link = 'color: #1a73e8; text-decoration: underline; font-family: Arial, sans-serif; font-size: 14px;';
          const spacer = '<tr><td height="16" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>';

          const folderContent = url
              ? '<a href="' + escapeHtml(url) + '" style="' + link + '">' + escapeHtml(label) + '</a>'
              : escapeHtml(label);

          const statusLines = statusText
              ? escapeHtml(statusText).replace(/\n/g, '<br />')
              : '';

          return '<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">\n' +
  '<html xmlns="http://www.w3.org/1999/xhtml">\n' +
  '<head>\n' +
  '    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />\n' +
  '    <meta name="viewport" content="width=device-width, initial-scale=1.0" />\n' +
  '    <title>' + escapeHtml(generateReconSubjectLine()) + '</title>\n' +
  '</head>\n' +
  '<body style="margin: 0; padding: 0; background-color: #ffffff;">\n' +
  '    <table width="600" align="left" cellpadding="0" cellspacing="0" border="0" role="presentation" style="margin: 0;">\n' +
  '        <tr><td bgcolor="#0D2C71" height="4" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>\n' +
  '        <tr><td height="20" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>\n' +
  '        <tr><td style="' + td + ' padding: 0;">Hi team,</td></tr>\n' +
  '        <tr><td height="12" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>\n' +
  '        <tr><td style="' + td + ' padding: 0;">' + namePart + ' Recon Reports have been posted to SharePoint: ' + folderContent + '.</td></tr>\n' +
  (statusLines ? spacer + '\n        <tr><td style="' + td + ' padding: 0;">' + statusLines + '</td></tr>\n' : '') +
  '        <tr><td height="20" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>\n' +
  '        <tr><td style="' + td + ' padding: 0;">Thank you.</td></tr>\n' +
  '        <tr><td height="8" style="font-size: 1px; line-height: 1px;">&nbsp;</td></tr>\n' +
  '    </table>\n' +
  '</body>\n' +
  '</html>';
      }

  ```

  **Note:** `generateReconSubjectLine()` is defined here before `getCycleValue()` in the file, but JS hoists `function` declarations — this is fine since `getCycleValue` is a `function` declaration. Both will be available at call time.

- [ ] **Step 2: Verify in browser**

  Open the browser console. With Recon Report tab active, type in the console:
  ```javascript
  generateReconSubjectLine()
  // Expected: "Wave 2C [Conversion Name] Recon Report - [Cycle]"

  generateReconEmailHTML().includes('Hi team,')
  // Expected: true

  generateReconFileLocationsHTML().includes('Premier Recon Reports')
  // Expected: true
  ```

- [ ] **Step 3: Commit**

  ```bash
  git add index.html
  git commit -m "Add recon email generation functions"
  ```

---

### Task 6: Update existing functions to branch on activeMode

**Files:**
- Modify: `index.html` — `generateSubjectLine`, `generateEmailHTML`, copy locations handler

- [ ] **Step 1: Update `generateSubjectLine` to branch on `activeMode`**

  Find the existing function:
  ```javascript
      function generateSubjectLine() {
          const name = document.getElementById('conversionName').value.trim();
          const cycle = getCycleValue();
          return 'Wave 2C ' + (name || '[Conversion Name]') + ' Files Posted - ' + (cycle || '[Cycle]');
      }
  ```
  Replace with:
  ```javascript
      function generateSubjectLine() {
          if (activeMode === 'recon') return generateReconSubjectLine();
          const name = document.getElementById('conversionName').value.trim();
          const cycle = getCycleValue();
          return 'Wave 2C ' + (name || '[Conversion Name]') + ' Files Posted - ' + (cycle || '[Cycle]');
      }
  ```

- [ ] **Step 2: Update `generateEmailHTML` to branch on `activeMode`**

  Find the start of the existing function:
  ```javascript
      function generateEmailHTML() {
          const name = document.getElementById('conversionName').value.trim();
          const cycle = getCycleValue();
  ```
  Replace the first two lines of the function body:
  ```javascript
      function generateEmailHTML() {
          if (activeMode === 'recon') return generateReconEmailHTML();
          const name = document.getElementById('conversionName').value.trim();
          const cycle = getCycleValue();
  ```

- [ ] **Step 3: Update the Copy File Locations handler to branch on `activeMode`**

  Find the copy locations click handler:
  ```javascript
      document.getElementById('copyLocations').addEventListener('click', async () => {
          const html = generateFileLocationsHTML();
  ```
  Replace that one line inside the handler:
  ```javascript
      document.getElementById('copyLocations').addEventListener('click', async () => {
          const html = activeMode === 'recon' ? generateReconFileLocationsHTML() : generateFileLocationsHTML();
  ```

- [ ] **Step 4: Verify in browser — full end-to-end**

  1. **Posted Files tab** — fill in Conversion Name "Item Master", Cycle "SIT1". Subject line: `Wave 2C Item Master Files Posted - SIT1`. Email preview shows the existing Posted Files email. ✓
  2. **Switch to Recon Report tab** — subject line instantly changes to `Wave 2C Item Master Recon Report - SIT1`. Preview shows the recon email with "Hi team," and "Item Master Recon Reports have been posted to SharePoint: Premier Recon Reports." ✓
  3. **Add a SharePoint URL** to the Recon Report Location URL field. Preview updates: folder name becomes a hyperlink. ✓
  4. **Toggle to Issues Found** — textarea clears. Type "GEFF_0015 attribute 8 was left blank in conversion." Preview updates immediately with the issue text. ✓
  5. **Toggle back to Clean** — textarea refills with the clean message. ✓
  6. **Copy Subject** in recon mode — paste into Notepad: `Wave 2C Item Master Recon Report - SIT1`. ✓
  7. **Copy Email HTML** in recon mode — paste into Outlook or a rich text editor. "Hi team," and folder name render correctly. ✓
  8. **Switch back to Posted Files** — all original fields intact, subject and preview revert. ✓

- [ ] **Step 5: Commit**

  ```bash
  git add index.html
  git commit -m "Branch subject line, email HTML, and copy handler on activeMode"
  ```

---

### Task 7: Final cleanup — commit plan doc, push to main

**Files:**
- Modify: none (cleanup + push)

- [ ] **Step 1: Verify the full file in the browser one more time**

  Open `index.html` fresh. Run through the verification checklist from Task 6 Step 4 one final time to confirm nothing was broken.

- [ ] **Step 2: Commit the plan document**

  ```bash
  git add docs/superpowers/plans/2026-04-03-recon-report-email.md
  git commit -m "Add recon report email implementation plan"
  ```

- [ ] **Step 3: Push to main**

  ```bash
  git push origin main
  ```

- [ ] **Step 4: Confirm remote is up to date**

  ```bash
  git log --oneline -6
  ```
  Expected: all feature commits visible, including the spec doc commit from brainstorming.
