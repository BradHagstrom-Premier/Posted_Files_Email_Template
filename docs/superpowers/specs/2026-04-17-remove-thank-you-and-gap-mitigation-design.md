# Remove "Thank you." + Signature Gap Mitigation

**Date:** 2026-04-17
**Status:** Approved for implementation

## Problem

In emails sent from Fyzia Shaik's Outlook (using CodeTwo Signatures), a large vertical gap (~250px) appears between the generated email body and her appended signature. The same tool used from Brad Hagstrom's Outlook produces a much smaller gap (~50px) when the body is longer.

Observed examples:
- Fyzia, short Posted Files body → ~250px gap between "Thank you." and signature.
- Fyzia, slightly different Posted Files body → ~250px gap (unchanged).
- Brad, long Posted Files body with fallout callout → ~50px gap.

## Root-Cause Analysis

The gap is **not produced by the generated HTML**. The outer table ends at "Thank you." followed by an 8px spacer, then `</table></body>`. Nothing in the HTML accounts for 200+ pixels of blank space.

The variance between short-body (large gap) and long-body (small gap) emails points to CodeTwo / Outlook anchoring the signature at a fixed vertical position in the compose window. Short body leaves empty paragraphs between the paste target (top) and the signature (fixed lower position); long body pushes content past the anchor and the gap collapses.

CodeTwo's placement logic and Fyzia's Outlook compose template are the source of the reserved whitespace. Neither is controlled by this tool.

## Decision

Two independent actions, per Approach D from brainstorming:

### 1. Code change — `index.html`

In both email-generation functions, remove the "Thank you." row along with its two adjacent spacer rows.

**`generateEmailHTML()` (Posted Files mode)** — remove these three lines:
```
<tr><td height="20" ...>&nbsp;</td></tr>   ← 20px spacer above "Thank you."
<tr><td style="..." padding: 0;">Thank you.</td></tr>
<tr><td height="8" ...>&nbsp;</td></tr>    ← 8px trailing spacer
```

**`generateReconEmailHTML()` (Recon mode)** — remove the equivalent three lines (same structure).

After removal, the body table ends at its last content row (file locations sub-table, fallout/errors callout, or recon status lines). The signature's own top margin provides the natural break.

No branching on `activeMode`. Both modes get the same treatment.

### 2. Out-of-code action — Brad to raise with Fyzia

Separately from this code change, Brad asks Fyzia to inspect her Outlook setup:
- Does her Outlook compose window show reserved blank paragraphs above her signature before she starts typing?
- If yes, either edit her default Outlook compose template to remove them, or check CodeTwo for a "remove empty lines before signature" / "collapse whitespace" option.

This is the actual fix for the gap; the code change above happens regardless.

## Out of Scope

- **Paste-workflow tips in the UI** (Approach B — "After pasting, press Ctrl+End then Backspace..."). Not added at this time. Revisit only if Fyzia's Outlook/CodeTwo configuration cannot be adjusted.
- **Synthetic filler content** to push the body past CodeTwo's anchor. Fragile across different users' setups.
- **Any HTML restructure** of the outer table or conversion away from `align="left"`. Not the cause.

## Acceptance Criteria

- "Thank you." no longer appears in either the Posted Files email preview or the Recon Report email preview.
- Both emails still copy cleanly via the existing Copy Email button.
- Outlook rendering of both modes is visually clean: last content row flows into signature without visible orphaned whitespace attributable to our HTML.
- Verified manually by opening `index.html` in a browser and testing both tabs, including the Recon Clean/Issues toggle.

## Non-Goals

- No change to subject-line generation.
- No change to File Locations or Fallout Callout rendering.
- No change to clipboard/copy behavior.
