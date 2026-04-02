# Posted Files Email Generator

A self-contained, browser-based tool that generates professionally formatted, Outlook-compatible HTML emails for notifying AWC and Deloitte contacts when conversion files (FBDI loads, validation files, conversion logs) are posted to SharePoint.

## What It Does

Definian consultants post conversion files at the end of every test cycle. This tool replaces manually written plain-text emails with a consistent, visually structured format that renders correctly in Microsoft Outlook's Word-based HTML engine.

## How to Use

1. **Open `index.html`** in Chrome or Edge (double-click the file — no server needed)
2. **Fill in the form** on the left panel:
   - Conversion Name (e.g., "Item Master", "Purchase Orders")
   - Cycle (SIT1, SIT2, UAT, Mock 1–3, D-Cut, or custom)
   - Optional: Summary of Changes, Errors/Fallout Notes
   - File location labels and SharePoint URLs
3. **Watch the live preview** update in real-time on the right panel
4. **Copy the subject line** using the "Copy Subject" button
5. **Copy the email HTML** using the "Copy Email HTML" button
6. **Paste into Outlook** compose window — formatting will be preserved

## Technical Details

- **Zero dependencies** — single HTML file, no build step, no external libraries
- **Works offline** via `file://` protocol
- **Outlook-compatible output** — table-based layout, all inline styles, no CSS classes or `<style>` blocks
- **Brand colors** — Definian Blue (#0D2C71), Definian Green (#00AB63)

## Who This Is For

Definian consultants (Brad Hagstrom, Fyzia Shaik, and others) working on Oracle Cloud conversion projects for AWC.
