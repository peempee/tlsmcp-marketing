# Engineering Blueprint Redesign — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Transform all 6 TLSMCP marketing pages from generic dark-SaaS into a distinctive "engineering blueprint" aesthetic with new typography, dot-grid backgrounds, numbered section labels, scroll animations, and refined components.

**Architecture:** Each page is self-contained HTML with inline `<style>`. Changes are applied per-page: font link, CSS variables, CSS rules, HTML label text, and a `<script>` block before `</body>`. The main landing page is built first as reference, then replicated to the remaining 5 pages with per-page section number adjustments.

**Tech Stack:** Pure HTML/CSS/vanilla JS. Google Fonts (Bricolage Grotesque, IBM Plex Sans, JetBrains Mono). Intersection Observer API. No build tools, no libraries.

---

## Task 1: Update fonts, variables, and body background — Main Landing Page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/index.html` (lines 27-58)

**Step 1: Replace Google Fonts link**

Change line 27 from:
```html
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
```
To:
```html
<link href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:wght@600;700;800&family=IBM+Plex+Sans:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
```

**Step 2: Update `:root` variables**

Add these new variables after `--green: #2dd4a8;` (line 48):
```css
  --cyan: #7dd3fc;
  --dot-grid: rgba(45, 212, 168, 0.06);
```

Change `--text-primary` from `#e8edf5` to `#f0f4ff`.

**Step 3: Update body font-family**

Change line 53 from:
```css
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
```
To:
```css
  font-family: 'IBM Plex Sans', -apple-system, BlinkMacSystemFont, sans-serif;
```

**Step 4: Add dot-grid background to body**

After the existing body rule's `overflow-x: hidden;` (line 57), add:
```css
  background-image: radial-gradient(var(--dot-grid) 1px, transparent 1px);
  background-size: 32px 32px;
```

**Step 5: Update heading font-family**

Add a new rule after the body rule (after the closing `}` on line 58):
```css
h1, h2, h3, h4 {
  font-family: 'Bricolage Grotesque', 'IBM Plex Sans', sans-serif;
}
```

**Step 6: Verify in browser**

Open `tlsmcp-site/tlsmcp/index.html` in browser. Check:
- Headings render in Bricolage Grotesque (geometric, distinctive)
- Body text renders in IBM Plex Sans (clean, technical)
- Subtle teal dot-grid visible on background
- Text is brighter white (#f0f4ff)
- No layout breakage

**Step 7: Commit**

```bash
git add tlsmcp/index.html
git commit -m "feat(landing): update typography and add dot-grid background"
```

---

## Task 2: Update section labels to numbered monospace — Main Landing Page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/index.html`

**Step 1: Update `.section-label` CSS**

Replace the existing `.section-label` and `.section-label::before` rules (lines 214-230) with:
```css
.section-label {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 2px;
  color: var(--teal);
  margin-bottom: 16px;
}
.section-label .num {
  color: var(--text-muted);
  font-size: 12px;
}
```

**Step 2: Update all section label HTML**

Replace each section label's inner HTML with numbered format. The 12 content sections on the landing page:

1. `<div class="section-label"><span class="num">01</span> // The Foundation</div>`
2. `<div class="section-label" style="justify-content: center;"><span class="num">02</span> // Two Problems, One Platform</div>`
3. `<div class="section-label"><span class="num">03</span> // Client Certificates &amp; mTLS</div>`
4. `<div class="section-label"><span class="num">04</span> // Server Certificate Automation</div>`
5. `<div class="section-label" style="justify-content: center;"><span class="num">05</span> // Core Capabilities</div>`
6. `<div class="section-label"><span class="num">06</span> // From Enforcement to Assurance</div>`
7. `<div class="section-label" style="justify-content: center;"><span class="num">07</span> // How It Works</div>`
8. `<div class="section-label" style="justify-content: center;"><span class="num">08</span> // Enterprise Ready</div>`
9. `<div class="section-label" style="justify-content: center;"><span class="num">09</span> // Scope Clarity</div>`
10. `<div class="section-label" style="justify-content: center;"><span class="num">10</span> // Common Questions</div>`
11. `<div class="section-label" style="justify-content: center;"><span class="num">11</span> // Ready to Start?</div>`
12. `<div class="section-label" style="justify-content: center;"><span class="num">12</span> // The <span class="cyphers-bracket">[</span> cyphers <span class="cyphers-bracket">]</span> Platform</div>`

**Step 3: Verify in browser**

Check that all section labels render in JetBrains Mono with dim number prefix and `//` separator.

**Step 4: Commit**

```bash
git add tlsmcp/index.html
git commit -m "feat(landing): numbered monospace section labels"
```

---

## Task 3: Section dividers, hero refinements, nav version tag — Main Landing Page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/index.html`

**Step 1: Update section dividers to dashed**

Find all CSS rules with `border-top: 1px solid var(--border)` and `border-bottom: 1px solid var(--border)` on section-level elements (`.premise`, `.problem`, `.posture`, `.faq`, `.product-family`, `.scope` section styles, and inline styles on server-certs/enterprise sections). Change `solid` to `dashed` on the section separators.

Specifically update these CSS rules:
- `.premise` (line 296-298): change both borders to `1px dashed var(--border)`
- `.problem` (line 248-250): change both borders to dashed
- `.posture` (line 518): change both borders to dashed
- `.faq` (line 737): change both borders to dashed
- `.product-family` (line 851-852): change both borders to dashed
- Also update inline `style` attributes on server-certs section (line 1367) and enterprise section (line 1584)
- Footer top border (line 907): change to dashed
- Nav bottom border (line 73): keep solid (nav is structural, not decorative)

**Step 2: Refine hero glow**

Update `.hero::before` (lines 146-156). Change:
```css
  width: 600px;
  height: 600px;
  background: radial-gradient(circle, var(--teal-glow) 0%, transparent 70%);
```
To:
```css
  width: 500px;
  height: 500px;
  background: radial-gradient(circle, rgba(45, 212, 168, 0.12) 0%, transparent 60%);
```

**Step 3: Add hero badge pulse animation**

Add this CSS after the hero-badge rules (after line 175):
```css
@keyframes pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.6; transform: scale(1.4); }
}
.hero-badge::before {
  animation: pulse 2s ease-in-out infinite;
}
```

**Step 4: Add nav version tag**

In the nav HTML, after the logo `<a>` tag (line 1088), add:
```html
<span style="font-family: 'JetBrains Mono', monospace; font-size: 11px; color: var(--text-muted); margin-left: 10px;">v1.0</span>
```

**Step 5: Verify in browser**

- Dashed borders on section dividers
- Tighter, more focused hero glow
- Pulsing green dot on hero badge
- `v1.0` monospace tag next to logo

**Step 6: Commit**

```bash
git add tlsmcp/index.html
git commit -m "feat(landing): dashed dividers, hero refinements, nav version tag"
```

---

## Task 4: Card component refinements — Main Landing Page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/index.html`

**Step 1: Add card hover glow**

Update `.cap-card:hover` rule (lines 467-471). Add box-shadow:
```css
.cap-card:hover {
  border-color: var(--border-light);
  background: var(--bg-card-hover);
  transform: translateY(-2px);
  box-shadow: 0 0 30px rgba(45, 212, 168, 0.06);
}
```

Do the same for `.ent-card:hover` (line 635):
```css
.ent-card:hover { border-color: var(--border-light); background: var(--bg-card-hover); box-shadow: 0 0 20px rgba(45, 212, 168, 0.06); }
```

**Step 2: Add coordinate labels to capability cards**

Update `.cap-card` CSS to add relative positioning (it already has `position: relative`). Add:
```css
.cap-card .card-coord {
  position: absolute;
  top: 16px;
  right: 20px;
  font-family: 'JetBrains Mono', monospace;
  font-size: 11px;
  color: var(--text-muted);
  opacity: 0.5;
}
```

Then add `<span class="card-coord">[01]</span>`, `[02]`, `[03]` to the three `.cap-card` elements in the HTML (inside each card, as first child).

**Step 3: Add terminal hover scale**

Add to the `.terminal` CSS rule (after line 823):
```css
.terminal { transition: transform 0.3s ease; }
.terminal:hover { transform: scale(1.01); }
```

**Step 4: Verify in browser**

- Cards show subtle teal glow on hover
- `[01]` `[02]` `[03]` visible in top-right of capability cards
- Terminal blocks scale slightly on hover

**Step 5: Commit**

```bash
git add tlsmcp/index.html
git commit -m "feat(landing): card hover glow, coordinate labels, terminal hover"
```

---

## Task 5: Terminal scanline, CTA dashed border, footer build string — Main Landing Page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/index.html`

**Step 1: Add terminal scanline animation**

Add these CSS rules after the terminal styles (after line 845):
```css
@keyframes scanline {
  0% { top: -2px; }
  100% { top: 100%; }
}
.terminal::after {
  content: '';
  position: absolute;
  left: 0;
  right: 0;
  height: 1px;
  background: linear-gradient(90deg, transparent, rgba(45, 212, 168, 0.15), transparent);
  animation: scanline 8s linear infinite;
  pointer-events: none;
}
```

Also add `position: relative;` to the `.terminal` rule if not already present.

**Step 2: Add CTA dashed border**

Update `.cta-section` CSS (line 785-798). Add after existing rules:
```css
.cta-section .container {
  border: 1px dashed rgba(45, 212, 168, 0.2);
  border-radius: 16px;
  padding: 60px 40px;
  position: relative;
}
```

**Step 3: Add footer build string**

In the footer HTML, in `.footer-bottom` (line 1836), add before the copyright span:
```html
<span style="font-family: 'JetBrains Mono', monospace; font-size: 11px; color: var(--text-muted);">build 2026.02 // www.cyphers.ai</span>
```

Move the copyright to be after the build string, or place the build string on the right side. The cleanest approach: replace the footer-bottom content with three items:
```html
<div class="footer-bottom">
  <span style="font-family: 'JetBrains Mono', monospace; font-size: 11px;">build 2026.02</span>
  <span>&copy; 2025 Cyphers. All rights reserved.</span>
  <div class="footer-legal">
    <a href="https://www.cyphers.ai">Privacy Policy</a>
    <a href="https://www.cyphers.ai">Terms of Service</a>
  </div>
</div>
```

**Step 4: Verify in browser**

- Subtle horizontal line sweeps through terminal blocks
- CTA section has dashed border container
- Footer shows monospace build string

**Step 5: Commit**

```bash
git add tlsmcp/index.html
git commit -m "feat(landing): terminal scanline, CTA border, footer build string"
```

---

## Task 6: Add animation JavaScript — Main Landing Page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/index.html`

**Step 1: Add CSS for animation states**

Add these CSS rules in the `<style>` block:
```css
/* ── SCROLL ANIMATIONS ── */
.reveal {
  opacity: 0;
  transform: translateY(20px);
  transition: opacity 0.6s ease, transform 0.6s ease;
}
.reveal.revealed {
  opacity: 1;
  transform: translateY(0);
}
.reveal-children > * {
  opacity: 0;
  transform: translateY(20px);
  transition: opacity 0.5s ease, transform 0.5s ease;
}
.reveal-children.revealed > * {
  opacity: 1;
  transform: translateY(0);
}

/* ── TERMINAL TYPING ── */
.terminal-body.typing .line {
  opacity: 0;
  transition: opacity 0.15s ease;
}
.terminal-body.typing .line.visible {
  opacity: 1;
}
@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}
.terminal-cursor {
  display: inline-block;
  width: 8px;
  height: 16px;
  background: var(--teal);
  animation: blink 1s step-end infinite;
  vertical-align: text-bottom;
  margin-left: 4px;
}

/* ── SCORE ANIMATION ── */
.score-bar-fill {
  width: 0 !important;
  transition: width 1.5s ease-out;
}
.score-bar-fill.animated {
  /* width set by JS */
}
```

**Step 2: Add `reveal` classes to HTML sections**

Add `class="reveal"` to each `<section>` element and `class="reveal-children"` to grid containers:
- `.capabilities-grid` → add `reveal-children`
- `.enterprise-grid` → add `reveal-children`
- `.steps-grid` → add `reveal-children`
- `.scope-grid` → add `reveal-children`
- `.product-grid` → add `reveal-children`
- Each `<section>` (or the section's `.container` child) → add `reveal`

**Step 3: Wrap terminal lines in spans**

Wrap each line in terminal-body with `<span class="line">...</span>` for the typing effect. Add `<span class="terminal-cursor"></span>` after the last line.

**Step 4: Add JavaScript before `</body>`**

```html
<script>
/* Scroll-triggered reveals */
const revealObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.classList.add('revealed');
      if (entry.target.classList.contains('reveal-children')) {
        Array.from(entry.target.children).forEach((child, i) => {
          child.style.transitionDelay = `${i * 100}ms`;
        });
      }
      revealObserver.unobserve(entry.target);
    }
  });
}, { threshold: 0.15 });

document.querySelectorAll('.reveal, .reveal-children').forEach(el => {
  revealObserver.observe(el);
});

/* Terminal typing effect */
const termObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const body = entry.target.querySelector('.terminal-body');
      if (!body || body.dataset.typed) return;
      body.dataset.typed = 'true';
      const lines = body.querySelectorAll('.line');
      lines.forEach((line, i) => {
        line.style.opacity = '0';
        setTimeout(() => { line.style.opacity = '1'; }, i * 150);
      });
      termObserver.unobserve(entry.target);
    }
  });
}, { threshold: 0.3 });

document.querySelectorAll('.terminal').forEach(el => {
  termObserver.observe(el);
});

/* Score counter animation */
const scoreObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const numEl = entry.target.querySelector('.score-number');
      if (!numEl || numEl.dataset.animated) return;
      numEl.dataset.animated = 'true';
      const target = 98;
      const duration = 1500;
      const start = performance.now();
      const tick = (now) => {
        const elapsed = now - start;
        const progress = Math.min(elapsed / duration, 1);
        const eased = 1 - Math.pow(1 - progress, 3);
        numEl.innerHTML = Math.round(eased * target) + '<span>/100</span>';
        if (progress < 1) requestAnimationFrame(tick);
      };
      requestAnimationFrame(tick);

      entry.target.querySelectorAll('.score-bar-fill').forEach(bar => {
        const w = bar.dataset.width || bar.style.width;
        bar.style.width = '0%';
        requestAnimationFrame(() => {
          bar.style.width = w;
          bar.classList.add('animated');
        });
      });
      scoreObserver.unobserve(entry.target);
    }
  });
}, { threshold: 0.3 });

const scoreCard = document.querySelector('.score-card');
if (scoreCard) scoreObserver.observe(scoreCard);
</script>
```

**Step 5: Store target widths as data attributes on score bars**

On each `.score-bar-fill`, add a `data-width` attribute matching the current inline `style="width: X%"`:
- `<div class="score-bar-fill fill-teal" style="width: 100%" data-width="100%"></div>`
- `<div class="score-bar-fill fill-teal" style="width: 98%" data-width="98%"></div>`
- `<div class="score-bar-fill fill-amber" style="width: 95%" data-width="95%"></div>`
- `<div class="score-bar-fill fill-green" style="width: 99%" data-width="99%"></div>`

**Step 6: Verify in browser**

- Scroll down: sections fade in as they enter viewport
- Grid items stagger their appearance (100ms delays)
- Terminal lines appear one at a time when scrolled into view
- Score number counts up from 0 to 98
- Progress bars animate their width

**Step 7: Commit**

```bash
git add tlsmcp/index.html
git commit -m "feat(landing): scroll reveals, terminal typing, score counter animations"
```

---

## Task 7: Apply all changes to mTLS page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/mtls/index.html`

**Step 1: Apply font link, variables, body, heading changes**

Same as Task 1 Steps 1-5. Apply to `mtls/index.html`.

**Step 2: Apply section label CSS update**

Same as Task 2 Step 1. Replace `.section-label` rules.

**Step 3: Update section label HTML with page-specific numbers**

The 9 content sections on the mTLS page:
1. `01 // The Foundation`
2. `02 // Client Certificate Issuance`
3. `03 // Architecture`
4. `04 // Certificate Lifecycle`
5. `05 // Why TLSMCP for mTLS`
6. `06 // Revocation`
7. `07 // Use Cases`
8. `08 // FAQ`
9. `09 // Get Started`

**Step 4: Apply section divider, hero, nav, card, terminal, CTA, footer changes**

Same patterns as Tasks 3-5 applied to this page's HTML.

**Step 5: Add animation CSS and JavaScript**

Same as Task 6. Add `reveal` classes to sections, wrap terminal lines, add script block. This page has terminal blocks (issuance, revocation) — apply typing effect to those.

**Step 6: Verify in browser, commit**

```bash
git add tlsmcp/mtls/index.html
git commit -m "feat(mtls): apply engineering blueprint redesign"
```

---

## Task 8: Apply all changes to Server Certs page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/server-certs/index.html`

Apply the same pattern as Task 7. Section numbers for this page (7 sections):
1. `01 // The Problem`
2. `02 // The Solution`
3. `03 // Provider Agnostic`
4. `04 // Before &amp; After`
5. `05 // Capabilities`
6. `06 // FAQ`
7. `07 // Get Started`

This page has a terminal block (cert status) and a before/after timeline — apply typing effect to terminal. Apply reveal animations to all grids.

```bash
git add tlsmcp/server-certs/index.html
git commit -m "feat(server-certs): apply engineering blueprint redesign"
```

---

## Task 9: Apply all changes to Enterprise page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/enterprise/index.html`

Section numbers (8 sections):
1. `01 // Compliance &amp; Standards`
2. `02 // Role-Based Access Control`
3. `03 // Multi-Tenant Architecture`
4. `04 // Audit &amp; Observability`
5. `05 // Deployment Options`
6. `06 // Network Isolation`
7. `07 // Enterprise FAQ`
8. `08 // Get Started`

This page has terminal blocks (RBAC, network policy) — apply typing effect. Enterprise page has `--amber-glow` variables — keep those, add new variables alongside.

```bash
git add tlsmcp/enterprise/index.html
git commit -m "feat(enterprise): apply engineering blueprint redesign"
```

---

## Task 10: Apply all changes to How-It-Works page

**Files:**
- Modify: `tlsmcp-site/tlsmcp/how-it-works/index.html`

Section numbers (10 sections):
1. `01 // Getting Started`
2. `02 // Core Architecture`
3. `03 // Connection Flow`
4. `04 // Certificate Lifecycle`
5. `05 // Configuration`
6. `06 // Deployment Patterns`
7. `07 // Integration`
8. `08 // Operations`
9. `09 // Technical FAQ`
10. `10 // Ready to Deploy?`

This page has multiple terminal/code blocks — apply typing effect to all.

```bash
git add tlsmcp/how-it-works/index.html
git commit -m "feat(how-it-works): apply engineering blueprint redesign"
```

---

## Task 11: Apply all changes to Score page

**Files:**
- Modify: `tlsmcp-site/score/index.html`

Section numbers (5 sections):
1. `01 // Four Dimensions`
2. `02 // How It Works`
3. `03 // The Difference`
4. `04 // Improve Your Score`
5. `05 // Find Out Where You Stand`

This page likely has its own score visualization — ensure the score counter animation works here too. This page may have additional score bars or metrics that need `data-width` attributes.

```bash
git add score/index.html
git commit -m "feat(score): apply engineering blueprint redesign"
```

---

## Task 12: Cross-page verification and final commit

**Files:**
- All 6 pages (read-only verification)

**Step 1: Open each page in browser and check:**

For each page:
- [ ] Fonts rendering correctly (Bricolage Grotesque headings, IBM Plex Sans body, JetBrains Mono labels)
- [ ] Dot-grid background visible
- [ ] Section labels show numbered monospace format
- [ ] Dashed section dividers
- [ ] Scroll animations trigger once on scroll
- [ ] Terminal typing effect works
- [ ] Card hover glow and coordinate labels
- [ ] CTA dashed border
- [ ] Footer build string
- [ ] Nav version tag
- [ ] Responsive layout at 900px breakpoint — nothing broken
- [ ] No SEO content changes

**Step 2: Check for consistency**

- Section numbers are sequential per page (not global)
- Hero badge pulse animation present on all pages with hero badges
- Footer is identical across all pages
- Nav is identical across all pages

**Step 3: Final commit if any fixes needed**

```bash
git add -A
git commit -m "fix: cross-page consistency cleanup for blueprint redesign"
```
