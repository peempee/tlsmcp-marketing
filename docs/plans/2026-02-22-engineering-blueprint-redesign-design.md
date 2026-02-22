# Engineering Blueprint Redesign — Design Document

**Date:** 2026-02-22
**Direction:** Technical & Precise ("Engineering Blueprint")
**Scope:** All 6 content pages in tlsmcp-site/

## Goal

Elevate the TLSMCP marketing site from a generic dark-SaaS template into a distinctive, memorable experience that conveys engineering precision and builds trust with DevOps engineers, platform teams, and CISOs — while improving conversion flow.

## Pages Affected

1. `tlsmcp/index.html` — Main landing page
2. `tlsmcp/mtls/index.html` — mTLS & Client Certificates
3. `tlsmcp/server-certs/index.html` — Server Certificate Automation
4. `tlsmcp/enterprise/index.html` — Enterprise Security & Compliance
5. `tlsmcp/how-it-works/index.html` — Technical Architecture
6. `score/index.html` — Cyphers Score

## Design Decisions

### 1. Typography System

**Headings:** Bricolage Grotesque (Google Fonts, weights 700-800)
- Variable-weight geometric grotesque with distinctive character
- Used for h1, h2, h3, h4

**Body:** IBM Plex Sans (Google Fonts, weights 400/500/600)
- Engineered for technical documentation by IBM
- Clean readability, engineering heritage reinforces brand

**Code/Labels:** JetBrains Mono (already loaded, weights 400/500)
- Retained for terminal blocks
- Extended to section labels and numbered coordinates

**Section label pattern:**
- Old: `── Client Certificates & mTLS` (teal, uppercase, Inter)
- New: `01 // CLIENT CERTIFICATES` (JetBrains Mono, with dim coordinate prefix)
- Each section gets a sequential number across the page

### 2. Background & Texture

**Dot-grid background:**
- Pure CSS using `radial-gradient` on body
- 1px dots, 32px spacing, ~6% opacity
- Creates "graph paper" / "engineering blueprint" feel

**Refined glow effects:**
- Keep teal radial glow on hero/CTA
- Tighten: smaller radius, more concentrated, lower opacity

**Section dividers:**
- Replace solid `border-top/bottom` with `1px dashed var(--border)` on section separators
- Cards retain solid borders

**Noise texture:**
- Subtle CSS noise overlay (2-3% opacity) on card backgrounds
- Pure CSS, no image files needed

### 3. Motion & Animation

All vanilla JS via Intersection Observer. Total ~3KB. No libraries.

**Scroll-triggered reveals:**
- Sections fade in + translateY(20px) on viewport entry
- Staggered delays (100ms between children) for grid items
- Trigger once only (no repeat on scroll back)
- CSS class `.revealed` added by JS; CSS handles transitions

**Terminal typing effect:**
- Lines appear sequentially with 150ms delay when terminal scrolls into view
- Blinking cursor follows last line
- CSS `@keyframes blink` for cursor

**Score counter animation:**
- "98" animates from 0 to 98 over ~1.5s using requestAnimationFrame
- Progress bars animate width simultaneously
- Triggered by Intersection Observer

**Hover enhancements:**
- Cards: subtle box-shadow glow (`0 0 20px rgba(45, 212, 168, 0.08)`) on hover
- Terminal blocks: `scale(1.01)` on hover
- Existing translateY(-2px) on cards retained

**Hero badge pulse:**
- Green dot gets `@keyframes pulse` animation
- Subtle scale + opacity cycle, 2s duration, infinite

### 4. Component Refinements

**Navigation:**
- Add monospace version tag `v1.0` next to logo (12px, dim, JetBrains Mono)

**Hero:**
- Dot-grid slightly more visible (8% opacity in hero region)
- Radial glow creates "illuminated blueprint" interaction with dots

**Cards:**
- Top-left corner coordinate label `[01]`, `[02]` in JetBrains Mono
- 11px, dim color, positioned absolutely top-right of card padding area

**Terminal blocks:**
- Subtle scan-line animation: horizontal gradient sweep (very slow, very subtle)
- CSS `@keyframes scanline` — a thin bright line sweeps top to bottom every 8s

**CTA section:**
- Subtle dashed border around CTA area (2px dashed, dim teal)
- "Blueprint action zone" feel

**Footer:**
- Add monospace build string: `build 2026.02 // www.cyphers.ai`
- 11px, JetBrains Mono, dim text-muted color

### 5. Color Refinements

New/modified CSS custom properties:

```css
--cyan: #7dd3fc;            /* NEW: secondary accent for informational elements */
--text-primary: #f0f4ff;    /* MODIFIED: brighter, bluer white for dot-grid contrast */
--dot-grid: rgba(45, 212, 168, 0.06);  /* NEW: dot-grid color */
```

**Teal usage discipline:**
- Bright `--teal` reserved for interactive elements and key data only
- `--teal-dim` used more for decorative elements (section label lines, card borders)
- `--cyan` for secondary informational highlights (sparingly)

## What Does NOT Change

- All existing content and copy
- HTML structure and semantic markup
- SEO meta tags, Open Graph, Twitter Cards, structured data
- URL structure and navigation links
- Responsive breakpoints (900px)
- Deployment configuration (netlify.toml, _headers, _redirects)
- sitemap.xml, robots.txt

## Implementation Approach

Since every page is self-contained with inline styles, changes must be applied to each page individually. The implementation order should be:

1. Start with the main landing page as the reference implementation
2. Extract the common changes (fonts, variables, backgrounds, animations JS) into a repeatable pattern
3. Apply to remaining 5 pages, adapting section numbers per page
4. Test responsive behavior on all pages
5. Verify no SEO regressions

## Risk Assessment

- **Low risk:** Typography and color changes — purely visual, no structural impact
- **Low risk:** CSS animations — additive, won't break existing layout
- **Low risk:** JS animations — progressive enhancement; site works without JS
- **Medium risk:** Section label pattern change — need to ensure consistency across all 6 pages
- **Mitigation:** Test each page after changes, verify in browser before committing
