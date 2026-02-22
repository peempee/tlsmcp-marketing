# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Marketing website for **TLSMCP** — an mTLS proxy and certificate lifecycle automation product by **Cyphers** (www.tlsmcp.com). The site is a pure static HTML/CSS site deployed on Netlify.

## Repository Layout

There are two layers in this workspace:

- **Root-level HTML files** (`tlsmcp-landing.html`, `tlsmcp-mtls.html`, etc.) — working/source copies of each page
- **`tlsmcp-site/`** — the deployable Netlify site with its own git repo (`github.com/peempee/tlsmcp-site.git`). This is the directory that gets published.

Inside `tlsmcp-site/`, pages use clean URL structure with `index.html` inside directories:

```
tlsmcp-site/
  index.html                       → Main landing page (/)
  mtls/index.html                  → mTLS & Client Certificates (/mtls/)
  server-certs/index.html          → Server Certificate Automation (/server-certs/)
  enterprise/index.html            → Enterprise Security (/enterprise/)
  how-it-works/index.html          → Technical Architecture (/how-it-works/)
  score/index.html                 → Cyphers Score (/score/)
```

When editing pages, changes should go into `tlsmcp-site/` (the deployed directory). The root-level HTML files may be slightly out of sync.

## Deployment

- **Host**: Netlify, publishing from `tlsmcp-site/` root (`publish = "."` in `netlify.toml`)
- **Domain**: www.tlsmcp.com
- **No build step** — Netlify serves the static files directly
- `_redirects` maps old `.html` filenames to clean URLs (301 redirects)
- `_headers` sets security headers (CSP, HSTS, X-Frame-Options) and cache policy for `/fonts/*` and `/assets/*`
- `netlify.toml` configures HTTPS enforcement and duplicate security headers

## Architecture Notes

- **No build tools, no JS framework** — every page is a self-contained HTML file with inline `<style>` blocks
- **Design system**: Dark theme using CSS custom properties defined in each page's `:root` block. Key variables: `--bg-primary: #0a0f1a`, `--teal: #2dd4a8`, `--text-primary: #e8edf5`
- **Fonts**: Inter (body) and JetBrains Mono (code) loaded from Google Fonts
- **Navigation**: Fixed top nav with `backdrop-filter: blur(20px)`, duplicated across pages (no shared components)
- **Container**: `max-width: 1140px` centered layout, consistent across all pages
- Since styles and nav are duplicated per page, changes to shared elements (nav, footer, color variables, fonts) must be applied to every HTML file individually

## SEO & Discoverability

Each page includes full SEO meta (description, keywords, canonical URL), Open Graph tags, and Twitter Card meta. The site also provides:
- `sitemap.xml` — all pages with priority and change frequency
- `robots.txt` — allows all crawlers including AI bots (GPTBot, ClaudeBot, etc.), blocks Bytespider
- `llms.txt` — structured product information for LLM consumption

## Git

The git repo lives inside `tlsmcp-site/`. The parent directory (`Claude_NEW/`) is not a git repository. Run git commands from `tlsmcp-site/`.
