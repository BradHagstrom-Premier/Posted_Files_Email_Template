---
name: Posted Files Email Generator
description: Internal Definian tool for generating Outlook-compatible email notifications for Oracle Cloud conversions.
colors:
  brand-blue: "#0D2C71"
  brand-blue-hover: "#1A3D8F"
  brand-green: "#00AB63"
  brand-green-hover: "#009955"
  midnight: "#02072D"
  slate-divider: "#3C405B"
  blueprint-fog: "#D8D7EE"
  surface-bg: "#F0F2F5"
  surface-white: "#FFFFFF"
  text-primary: "#1A1A2E"
  text-secondary: "#6B7280"
  border-default: "#DDE1E7"
  error: "#DC2626"
typography:
  headline:
    fontFamily: "DM Sans, Segoe UI, system-ui, -apple-system, sans-serif"
    fontSize: "15px"
    fontWeight: 600
    lineHeight: 1.3
    letterSpacing: "0.2px"
  title:
    fontFamily: "DM Sans, Segoe UI, system-ui, -apple-system, sans-serif"
    fontSize: "14px"
    fontWeight: 700
    lineHeight: 1.35
    letterSpacing: "0.3px"
  body:
    fontFamily: "DM Sans, Segoe UI, system-ui, -apple-system, sans-serif"
    fontSize: "13.5px"
    fontWeight: 400
    lineHeight: 1.5
  label:
    fontFamily: "DM Sans, Segoe UI, system-ui, -apple-system, sans-serif"
    fontSize: "12.5px"
    fontWeight: 600
    letterSpacing: "0.15px"
  meta:
    fontFamily: "DM Sans, Segoe UI, system-ui, -apple-system, sans-serif"
    fontSize: "11px"
    fontWeight: 700
    letterSpacing: "1px"
rounded:
  sm: "5px"
  md: "8px"
  pill: "10px"
spacing:
  xs: "6px"
  sm: "12px"
  md: "20px"
  lg: "28px"
components:
  button-primary:
    backgroundColor: "{colors.brand-blue}"
    textColor: "{colors.surface-white}"
    rounded: "{rounded.md}"
    padding: "13px 36px"
  button-primary-hover:
    backgroundColor: "{colors.brand-blue-hover}"
  button-primary-sm:
    backgroundColor: "{colors.brand-blue}"
    textColor: "{colors.surface-white}"
    rounded: "{rounded.sm}"
    padding: "7px 15px"
  button-outline:
    backgroundColor: "transparent"
    textColor: "{colors.brand-blue}"
    rounded: "{rounded.md}"
    padding: "13px 20px"
  tab-active:
    backgroundColor: "{colors.brand-blue}"
    textColor: "{colors.surface-white}"
    rounded: "6px"
    padding: "8px 12px"
  tab-inactive:
    backgroundColor: "transparent"
    textColor: "{colors.text-secondary}"
    rounded: "6px"
    padding: "8px 12px"
  input:
    backgroundColor: "{colors.surface-white}"
    textColor: "{colors.text-primary}"
    rounded: "{rounded.sm}"
    padding: "10px 13px"
---

# Design System: Posted Files Email Generator

## 1. Overview

**Creative North Star: "The Consultant's Instrument"**

This tool is a precision instrument. It does exactly one job, does it without waste, and looks like it was built by people who take their craft seriously. The interface does not call attention to itself; it frames the work. Every element earns its presence — if a color, shadow, or animation cannot be justified by function, it does not belong here.

The split-panel layout mirrors the consultant's mental model: inputs on the left, immediate output on the right. The visual weight deliberately centers on that preview frame. The form is the means; the generated email is the end. Nothing in the UI competes with that hierarchy.

This system explicitly rejects the vocabulary of startup SaaS tools: no gradient-heavy heroes, no micro-celebration animations for routine actions, no brand-as-wallpaper. It also rejects the opposite failure mode, the generic enterprise portal, which mistakes gray monotony for professionalism. The Consultant's Instrument is disciplined, not drab. It uses Definian brand color with intention and confidence, not decoration.

**Key Characteristics:**
- Single-font system (DM Sans) with strong weight and size contrast providing all hierarchy
- Definian Blue as the primary authority color; Definian Green reserved for success and confirmation states
- Flat surfaces at rest; structure revealed only through state changes
- Uppercase tracked labels as section delineators, never for body content
- Transitions are fast and directional (0.12s–0.25s ease-out), reflecting the tool's no-nonsense character

## 2. Colors: The Command Palette

Two primary colors carry all intent; neutrals hold the structure.

### Primary
- **Command Blue** (`#0D2C71`): The authority color. Used on the application header, all primary buttons, active tab states, form section labels, focused borders, and interactive links. Appears on approximately 15–20% of the surface. Its weight signals that something is actionable or significant.
- **Command Blue Hover** (`#1A3D8F`): The only hover treatment for blue elements. Never used at rest.

### Secondary
- **Signal Green** (`#00AB63`): Success and confirmation. Used exclusively on the "Clean" status toggle state, the toast notification, and any positive-outcome indicator in email output. Never used decoratively. Its rarity makes it mean something.
- **Signal Green Hover** (`#009955`): Hover deepening for green surfaces only.

### Neutral
- **Deep Text** (`#1A1A2E`): Primary reading text. Slightly navy-tinted rather than pure black, harmonizing with the blue palette without softening authority.
- **Muted Text** (`#6B7280`): Secondary labels, hints, form hints, column headers. Recedes without disappearing.
- **Border Default** (`#DDE1E7`): All structural dividers, input strokes, panel separations. Quiet and consistent.
- **Surface Background** (`#F0F2F5`): Application canvas. The grey field that holds both panels and makes the white form panel read as elevated.
- **Surface White** (`#FFFFFF`): Form panel and preview frame backgrounds. The working surface.
- **Error Red** (`#DC2626`): Delete/remove affordances only. Not a full status color; confined to destructive interactions.

### Brand Secondary (From Definian color guide; not yet in active use)
- **Midnight Charter** (`#02072D`): Deep navy for potential future dark-surface contexts. Appropriate as a section background or dramatic header variant.
- **Slate Divider** (`#3C405B`): For use as dividers or borders against Midnight Charter backgrounds only.
- **Blueprint Fog** (`#D8D7EE`): A warm-cool gray from the Definian palette. Candidate for replacing the generic `#F0F2F5` canvas in a future refinement pass.

### Named Rules
**The Two-Voice Rule.** Blue speaks authority; Green speaks success. These roles are not interchangeable. A "Copy Email" button is blue because copying is an action, not a success. A clean recon result is green because it is an outcome. Never use Green for primary actions or Blue for status confirmation.

**The No-Tint Rule.** Blue muted (`rgba(13, 44, 113, 0.08)`) appears as a hover background and focus ring only. It is a state indicator, not a design surface. Do not use it as a background at rest.

## 3. Typography

**Display / Body Font:** DM Sans (400, 500, 600, 700), fallback: Segoe UI, system-ui, -apple-system, sans-serif

**Character:** A single humanist sans that works at every scale through weight contrast. DM Sans at 700 reads with authority; at 400 it recedes cleanly. No second typeface is needed or wanted. The system's entire hierarchy is built from one variable.

### Hierarchy
- **Headline** (600, 15px, line-height 1.3, tracking 0.2px): Application header title. Single use.
- **Title** (700, 14px, line-height 1.35, tracking 0.3px): Primary action buttons ("Copy Email", "Copy File Locations") and generated subject line display. High-weight, action-weight.
- **Body** (400, 13.5px, line-height 1.5): Form inputs, select values, textarea content. The reading weight.
- **Label** (600, 12.5px, tracking 0.15px): Form field labels, tab text, status toggle text. Medium-weight bridge between section structure and body content.
- **Meta** (700, 11px, uppercase, tracking 1px): Form section titles ("CONVERSION DETAILS", "FILE LOCATIONS"). Tracked uppercase is reserved strictly for structural section delineation.
- **Hint** (400, 11.5px): Form label hints and supplementary context. Recedes behind Label.

### Named Rules
**The Uppercase Ceiling Rule.** Uppercase tracked type exists only for section-level delineators (`.form-section-title`). It must never appear on interactive elements, body content, or column data. One use, one meaning.

## 4. Elevation

This system is flat by default. Surfaces do not layer against each other through shadow; they layer through color contrast (white panel against grey canvas). Shadows are structural and state-driven, not ambient or decorative.

A surface earns a shadow when it needs to announce a change of state or establish a distinct plane of interaction. The preview frame always carries `shadow-md` because it is the primary output surface — not the form, not the buttons, the email. The copy bar carries a negative-direction shadow to seal the preview panel from below.

### Shadow Vocabulary
- **Ambient Low** (`0 1px 3px rgba(0,0,0,0.06)`): Resting interactive elements (small copy buttons, active tabs). Barely perceptible; says "this is clickable" without adding visual weight.
- **Structural Medium** (`0 4px 12px rgba(0,0,0,0.08)`): The preview iframe, elevated panels, hover state on primary buttons. The standard "this surface matters" signal.
- **Action High** (`0 8px 24px rgba(0,0,0,0.10)`): Primary action hover only (Copy Email button). Reinforces the decisiveness of the primary action without being theatrical.
- **Toast** (`0 6px 20px rgba(0, 171, 99, 0.30)`): Green-tinted shadow for the toast notification. The only colored shadow in the system; the green glow makes the success message unmissable.

### Named Rules
**The Flat-By-Default Rule.** Surfaces are flat at rest. A shadow appears only when a surface needs to announce state (hover, focus elevation, fixed positioning). Applying `shadow-md` to a card that is not interactive is prohibited.

## 5. Components

### Buttons

Two sizes, two variants. No other button types exist in this system.

**Primary (Copy Email):**
- **Shape:** Gently rounded (8px radius)
- **Background:** Command Blue (`#0D2C71`), `shadow-md` at rest
- **Text:** White, 14px, 700, tracking 0.3px
- **Padding:** 13px 36px
- **Hover:** Background lifts to `#1A3D8F`, shadow escalates to `shadow-lg`, no transform
- **Active:** `scale(0.98)` for tactile press feedback

**Primary Small (Copy Subject):**
- **Shape:** Slightly tighter (5px radius)
- **Padding:** 7px 15px
- **Text:** 12px, 600
- Everything else matches primary

**Outline (Copy File Locations):**
- **Shape:** 8px radius
- **Border:** 2px solid Command Blue
- **Background:** Transparent at rest; blue-muted on hover
- **Text:** Command Blue, 14px, 700
- **Active:** `scale(0.98)`

### Mode Tabs / Status Toggles

The tab strip and status toggle use identical structure: a blue-muted pill container with full-bleed background, 3px internal padding, and 6px-radius inner tabs.

- **Container:** `background: rgba(13, 44, 113, 0.08)`, `border-radius: 8px`, `padding: 3px`
- **Active Tab:** Command Blue background, white text, `shadow-sm`
- **Inactive Tab:** Transparent, muted text; hover applies 12% blue tint and blue text
- **Status "Issues" active:** Command Blue (same as tab) — issues require attention
- **Status "Clean" active:** Signal Green — only green active state in the entire UI

### Form Inputs / Selects / Textareas

- **Shape:** 5px radius (precise, not rounded)
- **Border:** 1.5px solid `#DDE1E7` at rest
- **Background:** Surface White
- **Focus:** Border shifts to Command Blue; `box-shadow: 0 0 0 3px rgba(13, 44, 113, 0.08)`
- **Placeholder:** `#B0B7C3` (deliberately lighter than muted text — placeholders do not compete with labels)
- **Select:** Custom chevron via SVG background-image; `appearance: none`
- **Textarea:** Resizable vertically only; `min-height: 76px`

### Header

- **Background:** Command Blue (`#0D2C71`)
- **Height:** 52px fixed
- **Accent bar:** 3px bottom edge, Signal Green occupying the leftmost 30%, fading to transparent. The only persistent green element in the chrome.
- **Green pip:** 4px × 22px, 2px radius, positioned left of the title. A calibrated Definian identity mark.
- **Badge:** Version/context chip, right-aligned, 12% white background, uppercase tracked 11px label.

### File Row (Signature Component)

Each file location row is a flex container with a numbered index (blue, tabular numeric, 12px 700), two inputs (label + URL), and a remove button.

- **Index number:** Command Blue, tabular-nums, right-aligned within its min-width column
- **Remove button (×):** `#C5C9D1` at rest; transitions to `#DC2626` text on `#FEF2F2` background on hover. Destructive intent revealed only on approach.
- **Add row button:** Dashed border (`1.5px dashed #DDE1E7`), blue text; hover fills with blue-muted. The dashed treatment signals "extensible" without demanding attention.
- **Row entry animation:** `rowSlideIn` — 0.2s ease, opacity 0→1, translateY(-6px)→0. Directional: rows enter from above, suggesting insertion.

### Toast

- **Background:** Signal Green (`#00AB63`)
- **Shadow:** Green-tinted (`0 6px 20px rgba(0, 171, 99, 0.3)`)
- **Position:** Fixed, bottom-center, `border-radius: 10px`
- **Entry:** translateY(+16px)→0, opacity 0→1, 0.25s ease-out
- **Text:** White, 13.5px, 600, tracking 0.2px

## 6. Do's and Don'ts

### Do:
- **Do** use Command Blue (`#0D2C71`) for all primary actions, structural headers, active states, and focused inputs. It is the single authority color.
- **Do** reserve Signal Green (`#00AB63`) exclusively for success states and confirmation feedback. Its rarity is its power.
- **Do** use uppercase tracked type (`font-weight: 700; text-transform: uppercase; letter-spacing: 1px`) only for structural section delineators. Never on buttons, data, or content.
- **Do** keep shadows flat at rest. A surface earns its shadow through state (hover, active, fixed panel). Static decorative shadows are prohibited.
- **Do** use `scale(0.96–0.98)` on active button press for tactile feedback. This is the only transform-based animation in the system.
- **Do** keep transitions at 0.12s–0.25s. The tool's character is quick and decisive, not flowing.
- **Do** use the Definian color guide values verbatim for any brand color: Blue `#0D2C71`, Green `#00AB63`, Midnight `#02072D`, Slate `#3C405B`, Cool Gray `#D8D7EE`.

### Don't:
- **Don't** use gradient text (`background-clip: text`). Never in this system.
- **Don't** introduce a second typeface. DM Sans covers the entire hierarchy.
- **Don't** use Signal Green on interactive elements. A button should never be green. Green answers questions; it does not ask them.
- **Don't** add decorative shadows to containers or cards at rest. The flat-by-default rule is not a default; it is a prohibition.
- **Don't** build anything that looks like a SaaS marketing tool: no gradient-heavy hero sections, no animated stat counters, no pastel success illustrations. This is a professional instrument.
- **Don't** use the generic SharePoint/enterprise portal gray palette. The Definian brand colors exist precisely to distinguish this tool from that register.
- **Don't** add motion for motion's sake. The existing animations (`rowSlideIn`, `fieldsFadeIn`, toast) each carry directional meaning. New animations must earn the same justification.
- **Don't** add a border-left stripe as a colored accent to any list item, card, or alert. Use full borders, background tints, or leading numbers instead.
- **Don't** use `align="left"` on the outer email body table. Outlook's Word renderer floats it, breaking CodeTwo signature layout. Use `width="600"` only.
