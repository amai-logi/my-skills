---
name: frontend
description: "Build frontend interfaces across all runtimes: React/web apps, Google Apps Script dialogs/sidebars, and static HTML. Always applies the Logitech design language. Use this skill when the user asks to build UI components, pages, dialogs, or any user-facing interface."
---

# Frontend Skill

## When to Skip

- Backend-only API with no UI
- Mobile-only app — go to `mobile`
- The user is only modifying existing frontend code and doesn't need the full skill workflow

## When to Use

- Building web UI (React, static HTML)
- Building Apps Script dialogs/sidebars
- Any user-facing interface that needs the design language

---

Build production-grade frontend interfaces that follow the Logitech design language exactly. Every screen, dialog, and component must look like it belongs to the same product — regardless of runtime.

## Step 0: Load the Design Language

**Before writing any frontend code**, read the design language file:

```
/Users/antoine/Code/claude-skills/design-language.md
```

This is the single source of truth for colors, typography, spacing, borders, shadows, buttons, forms, tables, badges, icons, layout, and motion. Follow it exactly — no creative interpretation, no alternative fonts, no custom color choices.

## Step 1: Identify the Runtime

Ask or infer from context:

| Runtime | When to use |
|---------|-------------|
| **React** | Full web applications, SPAs, dashboards with complex state |
| **Apps Script HTML** | Google Workspace tools — dialogs, sidebars, simple forms within Sheets/Docs |
| **Static HTML** | Landing pages, simple tools, prototypes, single-file interfaces |

Each runtime has different constraints but the **visual output must be identical**.

## Step 2: Apply Runtime-Specific Patterns

### React

- Define design tokens as CSS variables in a root stylesheet or theme file
- Use semantic token names from the design language (`--bg-page`, `--logi-teal`, `--text-primary`, etc.)
- Components are functional React with standard hooks
- Use Vite as the build tool unless the project specifies otherwise
- File structure: `components/`, `pages/`, `styles/`

```css
/* styles/tokens.css — generated from design-language.md */
:root {
  --bg-page: #f3f1ec;
  --card: #ffffff;
  --border: #d7d7d7;
  --logi-teal: #00fdcf;
  --logi-dark-green: #0d3a38;
  /* ... all tokens from design language */
}
```

### Apps Script HTML

- All CSS is inline in `<style>` tags within `HtmlService.createHtmlOutput()`
- No build tools, no imports, no npm
- Load Inter from Google Fonts via `<link>` tag
- Keep HTML self-contained — one file per dialog/sidebar
- Use `google.script.run` for server communication
- Max dialog dimensions: respect `setWidth()` / `setHeight()` constraints

```html
<!DOCTYPE html>
<html>
<head>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
      background: #f3f1ec;
      color: #1b1b1b;
      font-size: 0.85rem;
    }
    /* ... design language tokens as plain CSS */
  </style>
</head>
<body>
  <!-- content -->
</body>
</html>
```

### Static HTML

- Single `.html` file with embedded `<style>` and `<script>`
- Load Inter from Google Fonts
- CSS variables defined in `:root`
- No framework dependencies
- Works when opened directly in a browser

## Step 3: Component Library

Every runtime must implement these core components following the design language specs exactly:

### Buttons
- **Primary (CTA):** `--logi-teal` background, `--logi-dark-green` text, pill radius, teal glow on hover
- **Ghost/Outline:** transparent, `--text-secondary`, `1.5px solid --border`, dark green border on hover
- All buttons: `border-radius: 100px`, `font-size: 0.85rem`, `padding: 0.7rem 1.75rem`, `transition: all 0.15s`

### Forms
- Inputs: `1.5px solid var(--input-border)`, `border-radius: 8px`, `padding: 0.6rem 0.85rem`
- Focus: dark green border + teal ring shadow `0 0 0 3px rgba(0,253,207,0.2)`
- Labels: `0.8rem`, weight `600`
- Two-column layout: `grid-template-columns: 1fr 1fr; gap: 1rem`

### Cards & Sections
- White background, `1px solid var(--border)`, `border-radius: 16px`, `padding: 1.75rem 2rem`
- Section header: icon circle (38px, teal-light bg) + title + subtitle, bottom border separator

### Tables
- Wrapper: white card with `border-radius: 16px`, `overflow: hidden`
- Header: `--gray-950` background, uppercase `0.7rem`, `letter-spacing: 0.5px`
- Row hover: `--brand-cream` background

### Badges
- Pill shape, `padding: 0.2rem 0.65rem`, `font-size: 0.7rem`, weight `600`
- Approved: `#d6f5f0` bg / `#0d3a38` text
- In Review: `#FFF7ED` bg / `#C2410C` text
- Declined: `#FEE2E2` bg / `#B91C1C` text
- Draft: `#eeeeee` bg / `#8f8f8f` text

### Modals
- Overlay: `rgba(27,27,27,0.45)` + `backdrop-filter: blur(4px)`
- Card: white, `border-radius: 16px`, `padding: 2.5rem 2.25rem`, `max-width: 440px`
- Entrance animation: `scale(0.95) translateY(8px)` → identity, `0.2s ease-out`

### Toasts
- Fixed bottom center, `--logi-dark-green` background, white text
- Icon: `16px` SVG stroked in `--logi-teal`
- Animation: `translateY(20px) opacity:0` → `translateY(0) opacity:1`, `0.3s ease`

### Icons
- Always inline SVG — no icon libraries
- `fill: none`, `stroke: currentColor`, `stroke-width: 2`, `stroke-linecap: round`, `stroke-linejoin: round`
- Sizes: `14px` (row actions), `16px` (buttons), `18px` (section headers), `28px` (modals)

## Step 4: Layout

```
Page background: --bg-page (cream)
Max width: 860px (forms) or 1060px (data-heavy/dashboards)
Centered, padding: 2.5rem 1.5rem 4rem
```

For Apps Script dialogs/sidebars, adapt the layout to the constrained dimensions — no max-width needed, padding scales down.

### Responsive breakpoints (React/Static only)
- `768px`: stats 4→2 columns, header stacks, table scrolls horizontally
- `640px`: padding shrinks, H1 to 1.8rem, form rows 1 column

## Rules

**Cross-cutting guardrails:** Follow all rules in `guardrails.md` — security, cost, data integrity, process, resilience, and maintenance.

| Do | Don't |
|----|-------|
| Read `design-language.md` before every frontend task | Invent colors, fonts, or spacing |
| Use `--logi-teal` only for CTAs and active/focus states | Use teal as a general accent everywhere |
| Use `--brand-dark-green` text on teal backgrounds | Use white text on teal |
| Use cream `--bg-page` as page background | Use white or gray page backgrounds |
| Use Inter as the only font | Use any other font |
| Use inline SVG for icons | Install icon packages |
| Use `--radius-pill` (100px) for all buttons | Use square or small-radius buttons |
| Use dashed borders only for draft/incomplete items | Use dashed borders for complete items |
| Keep all CSS values in `rem` (except borders in `px`) | Use arbitrary `px` values for spacing |

## Handoff

### Input
- Component requirements from `software-architecture` design document
- API contract from `backend` skill (endpoints to call)
- `design-language.md` (mandatory — read before any frontend work)

### Output
- Frontend code (React components, Apps Script HTML, or static HTML)
- Styled to design language spec

### Next skills

| Next | What it receives |
|------|-----------------|
| `testing` | Frontend code to add component tests (Vitest) and E2E tests (Playwright) |
| `deployment` | Frontend Dockerfile or static files for deployment alongside the backend |

## Continuous Improvement

### Skill Health Check

At the end of every use, evaluate and report if any of the following apply:

- **Skill gap:** "This project needs [X] and no current skill covers it. Consider creating a `[name]` skill."
- **Skill split:** "This skill's [section] is growing complex enough to be its own skill." — e.g., if Apps Script HTML patterns diverge significantly from React patterns.
- **Skill overlap:** "This skill and `[other skill]` both cover [topic]. Consider consolidating."

Only report when genuinely applicable. Don't force observations.

### Learnings

Corrections and refinements discovered during use. When the user overrides a recommendation or a default doesn't fit, record it here so future uses benefit.

_(Empty — will accumulate over time)_
