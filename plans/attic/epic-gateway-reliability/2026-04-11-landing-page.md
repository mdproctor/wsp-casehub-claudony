# Claudony Landing Page Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Jekyll 4 landing page site in `docs/` served via GitHub Pages at `mdproctor.github.io/claudony`, with bioluminescent colony visual identity, seven landing sections, blog at `/blog/`, and getting-started guide at `/guide/`.

**Architecture:** Jekyll site lives in `docs/` (following project convention from sparge/cc-praxis). Global layout wraps all pages via `_layouts/default.html`. Landing page has its own `landing.html` layout. CSS is split into `main.css` (global), `landing.css` (landing sections), and `post.css` (blog/docs). Existing project documentation in `docs/` is excluded from Jekyll via `_config.yml`.

**Tech Stack:** Jekyll 4.3, jekyll-feed, Outfit + JetBrains Mono (Google Fonts), vanilla JS (minimal), inline SVG illustrations.

---

## File Map

```
docs/
├── _config.yml                    ← Jekyll config, baseurl, excludes
├── Gemfile                        ← jekyll + jekyll-feed
├── _layouts/
│   ├── default.html               ← nav + footer shell, font loading
│   ├── landing.html               ← extends default, landing-specific head
│   ├── post.html                  ← extends default, blog post
│   └── doc.html                   ← extends default, guide/docs
├── _posts/                        ← 12 migrated blog posts (Jekyll format)
├── assets/
│   ├── css/
│   │   ├── main.css               ← variables, reset, nav, footer, buttons
│   │   ├── landing.css            ← all landing page section styles
│   │   └── post.css               ← blog listing, post, doc page styles
│   └── js/
│       └── main.js                ← nav scroll class, smooth anchor scroll
├── blog/
│   └── index.html                 ← blog listing (all posts, reverse chron)
├── guide/
│   └── index.md                   ← getting-started guide
├── blog-archive/                  ← old non-Jekyll blog posts (excluded)
└── index.html                     ← landing page (layout: landing)
```

---

### Task 1: Initialize Jekyll project

**Files:**
- Create: `docs/Gemfile`
- Create: `docs/_config.yml`
- Rename: `docs/blog/` → `docs/blog-archive/` (git mv)
- Create dirs: `docs/_layouts/`, `docs/_posts/`, `docs/assets/css/`, `docs/assets/js/`, `docs/blog/`, `docs/guide/`

- [ ] **Step 1: Rename old blog directory to avoid collision with Jekyll blog listing**

```bash
cd /Users/mdproctor/claude/claudony
git mv docs/blog docs/blog-archive
```

- [ ] **Step 2: Create the directory structure**

```bash
mkdir -p docs/_layouts docs/_posts docs/assets/css docs/assets/js docs/blog docs/guide
```

- [ ] **Step 3: Write Gemfile**

Create `docs/Gemfile`:
```ruby
source "https://rubygems.org"

gem "jekyll", "~> 4.3"
gem "jekyll-feed", "~> 0.12"
```

- [ ] **Step 4: Write `_config.yml`**

Create `docs/_config.yml`:
```yaml
title: Claudony
description: >-
  Run Claude Code on one machine. Access every session from any device.
  Sessions persist. One controller to orchestrate them all.
baseurl: "/claudony"
url: "https://mdproctor.github.io"

markdown: kramdown
highlighter: rouge
permalink: /blog/:year/:month/:day/:title/

plugins:
  - jekyll-feed

exclude:
  - Gemfile
  - Gemfile.lock
  - DESIGN.md
  - BUGS-AND-ODDITIES.md
  - retro-issues.md
  - SESSION-HANDOFF-2026-04-04.md
  - adr/
  - ideas/
  - research/
  - superpowers/
  - blog-archive/

defaults:
  - scope:
      path: ""
      type: "posts"
    values:
      layout: "post"
  - scope:
      path: "guide"
    values:
      layout: "doc"
```

- [ ] **Step 5: Install gems and verify Jekyll builds**

```bash
cd docs
bundle install
bundle exec jekyll build --baseurl ""
```

Expected: `Destination: docs/_site` created with no errors. Zero posts, empty site is fine.

- [ ] **Step 6: Commit scaffold**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/Gemfile docs/_config.yml docs/_layouts docs/_posts docs/assets docs/blog docs/guide
git commit -m "feat(site): initialize Jekyll scaffold

Gemfile, _config.yml, directory structure. Renames old docs/blog/
to docs/blog-archive/ to free the path for the Jekyll blog listing.

Refs #(open or create issue for landing page)
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Global CSS and default layout

**Files:**
- Create: `docs/assets/css/main.css`
- Create: `docs/_layouts/default.html`

- [ ] **Step 1: Write `main.css`**

Create `docs/assets/css/main.css`:
```css
/* ── RESET & BASE ── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --void:    #05050f;
  --deep:    #09091a;
  --surface: #0e0e22;
  --violet:  #a060ff;
  --green:   #00ffaa;
  --magenta: #cc44ff;
  --text:    #e8e8f8;
  --dim:     rgba(232,232,248,0.45);
  --dimmer:  rgba(232,232,248,0.25);
  --font:    'Outfit', system-ui, sans-serif;
  --mono:    'JetBrains Mono', monospace;
}

html { scroll-behavior: smooth; }

body {
  background: var(--void);
  color: var(--text);
  font-family: var(--font);
  font-size: 16px;
  line-height: 1.6;
  overflow-x: hidden;
}

a { color: inherit; text-decoration: none; }

/* ── NAV ── */
nav.site-nav {
  position: sticky; top: 0; z-index: 100;
  display: flex; align-items: center; justify-content: space-between;
  padding: 16px 48px;
  background: rgba(5,5,15,0.85);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border-bottom: 1px solid rgba(160,96,255,0.12);
  transition: background 0.2s;
}

.nav-logo {
  font-family: var(--font);
  font-size: 18px; font-weight: 300; letter-spacing: 5px;
  text-transform: uppercase; color: var(--text);
}
.nav-logo span { color: var(--green); font-weight: 700; }

.nav-links { display: flex; gap: 32px; align-items: center; }

.nav-links a {
  font-size: 13px; letter-spacing: 1px; text-transform: uppercase;
  color: var(--dim); transition: color 0.15s; font-weight: 400;
}
.nav-links a:hover { color: var(--text); }

.nav-github {
  padding: 7px 18px;
  border: 1px solid rgba(160,96,255,0.35);
  border-radius: 50px; color: var(--violet) !important;
  transition: background 0.15s;
}
.nav-github:hover { background: rgba(160,96,255,0.1); }

/* ── SHARED SECTION UTILITIES ── */
section { position: relative; overflow: hidden; }
.container { max-width: 1100px; margin: 0 auto; padding: 0 48px; }

.section-divider {
  height: 1px;
  background: linear-gradient(90deg, transparent, rgba(160,96,255,0.18), transparent);
  margin: 0 48px;
}

.section-label {
  display: flex;
  font-size: 11px; letter-spacing: 3px; text-transform: uppercase;
  color: var(--violet); font-weight: 500; margin-bottom: 16px;
}
.section-label.centered { justify-content: center; }

.section-h2 {
  font-size: clamp(28px, 3.5vw, 40px); font-weight: 300;
  line-height: 1.2; margin-bottom: 20px; color: #fff;
}
.section-h2 strong { font-weight: 700; }
.section-h2 em { font-style: normal; color: var(--green); }
.section-h2.centered { text-align: center; }

.section-body { font-size: 16px; color: var(--dim); line-height: 1.8; font-weight: 300; }

/* ── BUTTONS ── */
.btn-primary {
  display: inline-flex; align-items: center; gap: 8px;
  padding: 13px 30px;
  background: linear-gradient(135deg, #7a2fff, #9040ff);
  color: #fff; font-size: 14px; font-weight: 600;
  border-radius: 50px; border: none; cursor: pointer;
  box-shadow: 0 0 30px rgba(120,0,255,0.3), 0 0 80px rgba(120,0,255,0.08);
  transition: box-shadow 0.2s, transform 0.15s;
  letter-spacing: 0.3px; font-family: var(--font);
}
.btn-primary:hover {
  box-shadow: 0 0 40px rgba(120,0,255,0.5);
  transform: translateY(-1px); color: #fff;
}

.btn-ghost {
  font-size: 14px; color: var(--dim); background: none; border: none;
  cursor: pointer; letter-spacing: 0.3px; font-family: var(--font);
  text-decoration: underline; text-underline-offset: 4px;
  text-decoration-color: rgba(255,255,255,0.2);
}

/* ── STATUS DOTS ── */
.dot {
  display: inline-block;
  width: 6px; height: 6px; border-radius: 50%; flex-shrink: 0;
}
.dot-green  { background: var(--green);  box-shadow: 0 0 6px var(--green);  animation: dotpulse 2s ease-in-out infinite; }
.dot-violet { background: var(--violet); box-shadow: 0 0 6px var(--violet); animation: dotpulse 2.5s ease-in-out infinite; }
.dot-dim    { background: rgba(255,255,255,0.2); }

@keyframes dotpulse { 0%,100%{opacity:1} 50%{opacity:0.3} }

/* ── AMBIENT ORB (reusable) ── */
.orb {
  position: absolute; border-radius: 50%; pointer-events: none;
}

/* ── FOOTER ── */
footer.site-footer {
  padding: 48px;
  border-top: 1px solid rgba(160,96,255,0.1);
  display: flex; align-items: center; justify-content: space-between;
  flex-wrap: wrap; gap: 20px;
}
.footer-logo {
  font-size: 16px; font-weight: 300; letter-spacing: 4px;
  text-transform: uppercase; color: var(--dimmer);
}
.footer-logo span { color: var(--green); font-weight: 700; }
.footer-links { display: flex; gap: 28px; }
.footer-links a { font-size: 13px; color: var(--dimmer); letter-spacing: 0.5px; }
.footer-links a:hover { color: var(--text); }
.footer-copy { font-size: 12px; color: rgba(255,255,255,0.15); }

/* ── RESPONSIVE ── */
@media (max-width: 768px) {
  nav.site-nav { padding: 14px 24px; }
  .container { padding: 0 24px; }
  .nav-links a:not(.nav-github) { display: none; }
  footer.site-footer { padding: 32px 24px; flex-direction: column; text-align: center; }
  .footer-links { justify-content: center; flex-wrap: wrap; }
  .section-divider { margin: 0 24px; }
}
```

- [ ] **Step 2: Write `_layouts/default.html`**

Create `docs/_layouts/default.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{% if page.title %}{{ page.title }} — {% endif %}{{ site.title }}</title>
  <meta name="description" content="{% if page.description %}{{ page.description }}{% else %}{{ site.description }}{% endif %}">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@200;300;400;600;700&family=JetBrains+Mono:wght@400;600&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="{{ '/assets/css/main.css' | prepend: site.baseurl }}">
  <link rel="stylesheet" href="{{ '/assets/css/landing.css' | prepend: site.baseurl }}">
  <link rel="stylesheet" href="{{ '/assets/css/post.css' | prepend: site.baseurl }}">
  {% feed_meta %}
</head>
<body>

<nav class="site-nav">
  <a href="{{ '/' | prepend: site.baseurl }}" class="nav-logo">CLAUDONY<span>.</span></a>
  <div class="nav-links">
    <a href="{{ '/#how' | prepend: site.baseurl }}">How it works</a>
    <a href="{{ '/#features' | prepend: site.baseurl }}">Features</a>
    <a href="{{ '/#install' | prepend: site.baseurl }}">Get started</a>
    <a href="{{ '/blog/' | prepend: site.baseurl }}">Blog</a>
    <a href="https://github.com/mdproctor/claudony" class="nav-github" target="_blank" rel="noopener">GitHub ↗</a>
  </div>
</nav>

{{ content }}

<footer class="site-footer">
  <div class="footer-logo">CLAUDONY<span>.</span></div>
  <div class="footer-links">
    <a href="{{ '/guide/' | prepend: site.baseurl }}">Docs</a>
    <a href="{{ '/blog/' | prepend: site.baseurl }}">Blog</a>
    <a href="https://github.com/mdproctor/claudony" target="_blank" rel="noopener">GitHub</a>
    <a href="https://github.com/mdproctor/claudony/releases" target="_blank" rel="noopener">Releases</a>
  </div>
  <div class="footer-copy">Free and open source</div>
</footer>

<script src="{{ '/assets/js/main.js' | prepend: site.baseurl }}"></script>
</body>
</html>
```

- [ ] **Step 3: Write `assets/js/main.js`**

Create `docs/assets/js/main.js`:
```js
// Add scrolled class to nav for optional styling
const nav = document.querySelector('nav.site-nav');
if (nav) {
  window.addEventListener('scroll', () => {
    nav.classList.toggle('scrolled', window.scrollY > 40);
  }, { passive: true });
}
```

- [ ] **Step 4: Verify build with nav and footer**

```bash
cd docs
bundle exec jekyll serve --baseurl "" --open-url
```

Open `http://localhost:4000`. Verify: nav renders at top with logo and links, footer renders at bottom, fonts load (Outfit visible), dark `#05050f` background.

- [ ] **Step 5: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/assets/css/main.css docs/_layouts/default.html docs/assets/js/main.js
git commit -m "feat(site): add global CSS, default layout, nav and footer

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Landing CSS (all section styles)

**Files:**
- Create: `docs/assets/css/landing.css`

- [ ] **Step 1: Write `landing.css`**

Create `docs/assets/css/landing.css`:
```css
/* ── HERO ── */
#hero {
  min-height: 92vh;
  display: flex; flex-direction: column; justify-content: center;
  padding: 80px 0 60px;
}

.hero-ambient {
  position: absolute; inset: 0; pointer-events: none;
  background:
    radial-gradient(ellipse at 70% 10%, rgba(160,60,255,0.18) 0%, transparent 50%),
    radial-gradient(ellipse at 15% 80%, rgba(0,255,140,0.08) 0%, transparent 40%),
    radial-gradient(ellipse at 50% 50%, rgba(100,0,200,0.05) 0%, transparent 60%);
}

.hero-inner {
  position: relative; z-index: 2;
  display: grid; grid-template-columns: 1fr 420px; gap: 60px;
  align-items: center;
}

.hero-eyebrow {
  font-size: 11px; letter-spacing: 3px; text-transform: uppercase;
  color: var(--violet); font-weight: 500; margin-bottom: 20px;
  display: flex; align-items: center; gap: 12px;
}
.hero-eyebrow::before {
  content: '';
  display: inline-block; width: 28px; height: 1px;
  background: linear-gradient(90deg, var(--violet), transparent);
}

.hero-h1 {
  font-size: clamp(36px, 5vw, 56px); font-weight: 300;
  line-height: 1.12; letter-spacing: 0.5px;
  margin-bottom: 22px; color: #fff;
}
.hero-h1 strong { font-weight: 700; }
.hero-h1 em { font-style: normal; color: var(--green); }

.hero-sub {
  font-size: 17px; line-height: 1.75; color: var(--dim);
  margin-bottom: 36px; max-width: 480px; font-weight: 300;
}

.hero-cta { display: flex; gap: 14px; align-items: center; flex-wrap: wrap; }

.hero-status {
  position: relative; z-index: 2;
  display: flex; gap: 20px; margin-top: 48px; flex-wrap: wrap;
}
.status-pill {
  display: flex; align-items: center; gap: 8px;
  font-size: 12px; color: var(--dimmer);
  font-family: var(--mono); letter-spacing: 0.5px;
}

/* ── PROBLEM ── */
#problem { padding: 100px 0; background: var(--deep); }

.problem-inner {
  display: grid; grid-template-columns: 1fr 1fr; gap: 80px;
  align-items: center;
}

.pain-cards { display: flex; flex-direction: column; gap: 14px; }

.pain-card {
  padding: 18px 20px;
  background: rgba(255,255,255,0.03);
  border: 1px solid rgba(255,255,255,0.07);
  border-radius: 10px;
  display: flex; gap: 14px; align-items: flex-start;
}
.pain-icon { font-size: 20px; flex-shrink: 0; margin-top: 2px; }
.pain-title { font-size: 14px; font-weight: 600; color: var(--text); margin-bottom: 4px; }
.pain-desc { font-size: 13px; color: var(--dimmer); line-height: 1.6; }

/* ── HOW IT WORKS ── */
#how { padding: 100px 0; }

.how-ambient {
  position: absolute; inset: 0; pointer-events: none;
  background: radial-gradient(ellipse at 50% 50%, rgba(100,0,200,0.07) 0%, transparent 70%);
}

.how-header { text-align: center; margin-bottom: 64px; }
.how-header p { color: var(--dim); font-size: 15px; font-weight: 300; max-width: 460px; margin: 0 auto; }

.architecture {
  display: grid; grid-template-columns: 1fr 300px 1fr; gap: 0;
  align-items: center;
}
.arch-col { display: flex; flex-direction: column; gap: 16px; }
.arch-col.right { align-items: flex-end; text-align: right; }

.arch-node {
  padding: 16px 20px; max-width: 270px;
  background: rgba(160,96,255,0.06);
  border: 1px solid rgba(160,96,255,0.2);
  border-radius: 10px;
}
.arch-node.green-node {
  background: rgba(0,255,140,0.05);
  border-color: rgba(0,255,140,0.2);
}
.arch-node.dim-node {
  background: rgba(255,255,255,0.02);
  border-color: rgba(255,255,255,0.08);
}
.arch-node-title { font-size: 13px; font-weight: 600; color: var(--text); margin-bottom: 4px; }
.arch-node.green-node .arch-node-title { color: var(--green); }
.arch-node.dim-node .arch-node-title { color: var(--dimmer); }
.arch-node-desc { font-size: 12px; color: var(--dimmer); line-height: 1.5; }

/* ── FEATURES ── */
#features { padding: 100px 0; background: var(--deep); }

.features-header { text-align: center; margin-bottom: 56px; }

.features-grid {
  display: grid; grid-template-columns: repeat(2, 1fr); gap: 20px;
}

.feature-card {
  padding: 28px;
  background: rgba(255,255,255,0.02);
  border: 1px solid rgba(255,255,255,0.07);
  border-radius: 12px;
  transition: border-color 0.2s, background 0.2s;
}
.feature-card:hover {
  border-color: rgba(160,96,255,0.3);
  background: rgba(160,96,255,0.04);
}
.feature-card.green:hover  { border-color: rgba(0,255,140,0.3);  background: rgba(0,255,140,0.04); }
.feature-card.magenta:hover { border-color: rgba(204,68,255,0.3); background: rgba(204,68,255,0.04); }

.feature-icon {
  width: 40px; height: 40px; border-radius: 10px; margin-bottom: 16px;
  display: flex; align-items: center; justify-content: center; font-size: 18px;
  background: rgba(160,96,255,0.1); border: 1px solid rgba(160,96,255,0.2);
}
.feature-card.green   .feature-icon { background: rgba(0,255,140,0.08);  border-color: rgba(0,255,140,0.2); }
.feature-card.magenta .feature-icon { background: rgba(204,68,255,0.08); border-color: rgba(204,68,255,0.2); }

.feature-title { font-size: 16px; font-weight: 600; margin-bottom: 8px; color: var(--text); }
.feature-desc  { font-size: 14px; color: var(--dimmer); line-height: 1.65; }

/* ── GET STARTED ── */
#install { padding: 100px 0; }

.install-ambient {
  position: absolute; inset: 0; pointer-events: none;
  background:
    radial-gradient(ellipse at 30% 50%, rgba(0,255,140,0.06) 0%, transparent 50%),
    radial-gradient(ellipse at 75% 40%, rgba(160,96,255,0.08) 0%, transparent 45%);
}

.install-header { text-align: center; margin-bottom: 56px; }

.steps {
  display: grid; grid-template-columns: repeat(3, 1fr); gap: 24px;
  margin-bottom: 48px;
}

.step {
  padding: 28px 24px;
  background: rgba(255,255,255,0.02);
  border: 1px solid rgba(255,255,255,0.07);
  border-radius: 12px; position: relative;
}
.step::before {
  content: attr(data-n);
  position: absolute; top: -14px; left: 24px;
  font-family: var(--mono); font-size: 11px; font-weight: 600;
  color: var(--violet); background: var(--void);
  padding: 2px 10px; border: 1px solid rgba(160,96,255,0.3); border-radius: 20px;
  letter-spacing: 1px;
}
.step-title { font-size: 15px; font-weight: 600; color: var(--text); margin-bottom: 10px; }
.step-body  { font-size: 13px; color: var(--dimmer); line-height: 1.65; }

.code-block {
  background: rgba(0,0,0,0.4);
  border: 1px solid rgba(160,96,255,0.2);
  border-radius: 10px; padding: 20px 24px;
  font-family: var(--mono); font-size: 13px;
  color: var(--green); line-height: 1.8;
  max-width: 640px; margin: 0 auto;
}
.code-block .comment { color: rgba(255,255,255,0.25); }
.code-prompt { color: var(--dimmer); user-select: none; }

.install-cta { text-align: center; margin-top: 40px; }

/* ── BLOG PREVIEW ── */
#blog { padding: 100px 0; background: var(--deep); }

.blog-header {
  display: flex; align-items: flex-end; justify-content: space-between;
  margin-bottom: 40px;
}
.blog-all { font-size: 13px; color: var(--violet); }
.blog-all:hover { text-decoration: underline; text-underline-offset: 3px; }

.blog-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; }

.blog-card {
  padding: 24px;
  background: rgba(255,255,255,0.02);
  border: 1px solid rgba(255,255,255,0.07);
  border-radius: 10px; transition: border-color 0.2s;
}
.blog-card:hover { border-color: rgba(160,96,255,0.3); }
.blog-date    { font-size: 11px; color: var(--dimmer); font-family: var(--mono); margin-bottom: 10px; }
.blog-title   { font-size: 15px; font-weight: 600; color: var(--text); line-height: 1.4; margin-bottom: 10px; }
.blog-excerpt { font-size: 13px; color: var(--dimmer); line-height: 1.6; }
.blog-card a  { display: block; }
.blog-card a:hover .blog-title { color: var(--violet); }

/* ── RESPONSIVE (landing) ── */
@media (max-width: 1024px) {
  .hero-inner { grid-template-columns: 1fr; }
  .hero-inner svg { display: none; }
  .architecture { grid-template-columns: 1fr; gap: 32px; }
  .arch-col.right { align-items: flex-start; text-align: left; }
}

@media (max-width: 768px) {
  #hero { padding: 60px 0 40px; }
  .problem-inner { grid-template-columns: 1fr; gap: 48px; }
  .features-grid { grid-template-columns: 1fr; }
  .steps { grid-template-columns: 1fr; }
  .blog-grid { grid-template-columns: 1fr; }
  .blog-header { flex-direction: column; align-items: flex-start; gap: 12px; }
}
```

- [ ] **Step 2: Verify the CSS file is valid (no syntax errors)**

```bash
cd docs
bundle exec jekyll build --baseurl "" 2>&1 | grep -i error
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/assets/css/landing.css
git commit -m "feat(site): add landing page CSS — all section styles

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Landing layout and index.html hero section

**Files:**
- Create: `docs/_layouts/landing.html`
- Create: `docs/index.html` (hero section)

- [ ] **Step 1: Write `_layouts/landing.html`**

Create `docs/_layouts/landing.html`:
```html
---
layout: default
---
{{ content }}
```

- [ ] **Step 2: Write `index.html` with front matter and hero section**

Create `docs/index.html`:
```html
---
layout: landing
title: Home
---

<!-- 1. HERO -->
<section id="hero">
  <div class="hero-ambient"></div>
  <div class="orb" style="width:400px;height:400px;top:-100px;right:-80px;background:radial-gradient(circle,rgba(140,0,255,0.14),transparent 70%)"></div>
  <div class="orb" style="width:300px;height:300px;bottom:0;left:-60px;background:radial-gradient(circle,rgba(0,255,140,0.07),transparent 70%)"></div>

  <div class="container">
    <div class="hero-inner">
      <div class="hero-text">
        <div class="hero-eyebrow">Remote colony intelligence</div>
        <h1 class="hero-h1">
          Your Claude sessions,<br>
          <strong>alive</strong> and <em>everywhere</em>
        </h1>
        <p class="hero-sub">
          Run Claude Code on one machine. Access every session from any browser, any device. Sessions persist when you disconnect. One controller to orchestrate them all.
        </p>
        <div class="hero-cta">
          <a href="#install" class="btn-primary">⬡ Deploy your colony</a>
          <button class="btn-ghost" onclick="document.getElementById('how').scrollIntoView({behavior:'smooth'})">See how it works ↓</button>
        </div>
        <div class="hero-status">
          <div class="status-pill"><span class="dot dot-green"></span>3 sessions active</div>
          <div class="status-pill"><span class="dot dot-violet"></span>Controller online</div>
          <div class="status-pill"><span class="dot dot-dim"></span>2 dormant</div>
        </div>
      </div>

      <!-- Colony network illustration -->
      <svg viewBox="0 0 420 440" width="420" height="440" aria-hidden="true" style="display:block">
        <defs>
          <radialGradient id="hubGrad" cx="50%" cy="50%" r="50%">
            <stop offset="0%"   stop-color="#a060ff" stop-opacity="0.5"/>
            <stop offset="60%"  stop-color="#7a2fff" stop-opacity="0.15"/>
            <stop offset="100%" stop-color="#a060ff" stop-opacity="0"/>
          </radialGradient>
          <radialGradient id="nodeG" cx="50%" cy="50%" r="50%">
            <stop offset="0%"   stop-color="#00ffaa" stop-opacity="0.4"/>
            <stop offset="100%" stop-color="#00ffaa" stop-opacity="0"/>
          </radialGradient>
          <radialGradient id="nodeM" cx="50%" cy="50%" r="50%">
            <stop offset="0%"   stop-color="#cc44ff" stop-opacity="0.35"/>
            <stop offset="100%" stop-color="#cc44ff" stop-opacity="0"/>
          </radialGradient>
          <filter id="glow" x="-50%" y="-50%" width="200%" height="200%">
            <feGaussianBlur stdDeviation="4" result="blur"/>
            <feMerge><feMergeNode in="blur"/><feMergeNode in="SourceGraphic"/></feMerge>
          </filter>
          <filter id="softglow" x="-100%" y="-100%" width="300%" height="300%">
            <feGaussianBlur stdDeviation="8" result="blur"/>
            <feMerge><feMergeNode in="blur"/><feMergeNode in="SourceGraphic"/></feMerge>
          </filter>
        </defs>

        <!-- Hub ambient -->
        <circle cx="210" cy="220" r="150" fill="url(#hubGrad)" opacity="0.6"/>

        <!-- Controller hub rings -->
        <circle cx="210" cy="220" r="52" fill="rgba(120,0,255,0.12)" stroke="rgba(160,96,255,0.2)" stroke-width="1" filter="url(#softglow)"/>
        <circle cx="210" cy="220" r="36" fill="rgba(120,0,255,0.18)" stroke="rgba(160,96,255,0.4)" stroke-width="1.5"/>
        <circle cx="210" cy="220" r="22" fill="rgba(140,0,255,0.3)"  stroke="rgba(160,96,255,0.7)" stroke-width="1.5" filter="url(#glow)"/>
        <circle cx="210" cy="220" r="9"  fill="#a060ff" filter="url(#glow)">
          <animate attributeName="opacity" values="1;0.5;1" dur="2.2s" repeatCount="indefinite"/>
        </circle>
        <text x="210" y="270" text-anchor="middle" font-family="Outfit,sans-serif" font-size="11" fill="rgba(160,96,255,0.7)" font-weight="300" letter-spacing="2">CONTROLLER</text>

        <!-- Organic connection paths -->
        <path d="M210,220 Q260,150 330,95"  fill="none" stroke="rgba(0,255,140,0.25)"  stroke-width="1.5"/>
        <path d="M210,220 Q145,130 85,80"   fill="none" stroke="rgba(0,255,140,0.2)"   stroke-width="1.5"/>
        <path d="M210,220 Q310,220 385,200" fill="none" stroke="rgba(160,96,255,0.3)"  stroke-width="1.5"/>
        <path d="M210,220 Q280,310 340,360" fill="none" stroke="rgba(204,68,255,0.15)" stroke-width="1" stroke-dasharray="4,5"/>
        <path d="M210,220 Q130,300 75,360"  fill="none" stroke="rgba(204,68,255,0.12)" stroke-width="1" stroke-dasharray="4,5"/>

        <!-- Active node: top-right -->
        <circle cx="330" cy="95" r="32" fill="url(#nodeG)" opacity="0.5"/>
        <circle cx="330" cy="95" r="18" fill="rgba(0,255,140,0.12)" stroke="rgba(0,255,140,0.5)" stroke-width="1.5" filter="url(#glow)"/>
        <circle cx="330" cy="95" r="7"  fill="#00ffaa" filter="url(#glow)">
          <animate attributeName="opacity" values="1;0.4;1" dur="1.8s" repeatCount="indefinite"/>
        </circle>
        <text x="330" y="69" text-anchor="middle" font-family="Outfit,sans-serif" font-size="11" fill="rgba(0,255,140,0.8)">api-dev</text>

        <!-- Active node: top-left -->
        <circle cx="85" cy="80" r="28" fill="url(#nodeG)" opacity="0.4"/>
        <circle cx="85" cy="80" r="17" fill="rgba(0,255,140,0.1)" stroke="rgba(0,255,140,0.45)" stroke-width="1.5" filter="url(#glow)"/>
        <circle cx="85" cy="80" r="7"  fill="#00ffaa" filter="url(#glow)">
          <animate attributeName="opacity" values="1;0.4;1" dur="2.4s" repeatCount="indefinite"/>
        </circle>
        <text x="85" y="55" text-anchor="middle" font-family="Outfit,sans-serif" font-size="11" fill="rgba(0,255,140,0.8)">frontend</text>

        <!-- Active node: right -->
        <circle cx="385" cy="200" r="26" fill="url(#nodeG)" opacity="0.35"/>
        <circle cx="385" cy="200" r="16" fill="rgba(0,255,140,0.08)" stroke="rgba(0,255,140,0.4)" stroke-width="1.5" filter="url(#glow)"/>
        <circle cx="385" cy="200" r="6"  fill="#00ffaa" opacity="0.9" filter="url(#glow)">
          <animate attributeName="opacity" values="0.9;0.3;0.9" dur="3s" repeatCount="indefinite"/>
        </circle>
        <text x="385" y="230" text-anchor="middle" font-family="Outfit,sans-serif" font-size="11" fill="rgba(0,255,140,0.7)">research</text>

        <!-- Dormant node: bottom-right -->
        <circle cx="340" cy="360" r="22" fill="url(#nodeM)" opacity="0.25"/>
        <circle cx="340" cy="360" r="13" fill="rgba(204,68,255,0.06)" stroke="rgba(204,68,255,0.25)" stroke-width="1.2"/>
        <circle cx="340" cy="360" r="5"  fill="rgba(204,68,255,0.4)"/>
        <text x="340" y="383" text-anchor="middle" font-family="Outfit,sans-serif" font-size="11" fill="rgba(204,68,255,0.4)">docs</text>
        <text x="340" y="395" text-anchor="middle" font-family="JetBrains Mono,monospace" font-size="8" fill="rgba(204,68,255,0.3)">dormant</text>

        <!-- Dormant node: bottom-left -->
        <circle cx="75" cy="360" r="20" fill="url(#nodeM)" opacity="0.2"/>
        <circle cx="75" cy="360" r="12" fill="rgba(204,68,255,0.05)" stroke="rgba(204,68,255,0.2)" stroke-width="1"/>
        <circle cx="75" cy="360" r="4"  fill="rgba(204,68,255,0.35)"/>
        <text x="75" y="383" text-anchor="middle" font-family="Outfit,sans-serif" font-size="11" fill="rgba(204,68,255,0.35)">infra</text>
        <text x="75" y="395" text-anchor="middle" font-family="JetBrains Mono,monospace" font-size="8" fill="rgba(204,68,255,0.25)">dormant</text>
      </svg>
    </div>
  </div>
</section>

<div class="section-divider"></div>
```

- [ ] **Step 3: Verify hero renders**

```bash
cd docs
bundle exec jekyll serve --baseurl ""
```

Open `http://localhost:4000`. Verify:
- Hero fills viewport height
- Colony SVG visible on right (disappears on mobile)
- Status pills visible with pulsing dots
- "Deploy your colony" button has violet glow
- Ambient orbs visible in background

- [ ] **Step 4: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/_layouts/landing.html docs/index.html
git commit -m "feat(site): add landing layout and hero section

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: Problem and How it works sections

**Files:**
- Modify: `docs/index.html` (append two sections)

- [ ] **Step 1: Append problem and how-it-works sections to `index.html`**

Add after the closing `<div class="section-divider"></div>` of the hero:

```html
<!-- 2. PROBLEM -->
<section id="problem">
  <div class="container">
    <div class="problem-inner">
      <div>
        <div class="section-label">The problem</div>
        <h2 class="section-h2">Claude Code dies<br>when <em>you</em> disconnect</h2>
        <p class="section-body">
          Close the terminal. Switch devices. Laptop goes to sleep. Your session is gone — the context, the working state, everything. You're back to square one.
          <br><br>
          And there's no way to reach it from your iPad, your phone, or another machine. Claude Code is tethered to wherever it was started.
        </p>
      </div>
      <div class="pain-cards">
        <div class="pain-card">
          <div class="pain-icon">💀</div>
          <div>
            <div class="pain-title">Sessions don't survive disconnects</div>
            <div class="pain-desc">Close a terminal, lose everything. Every reconnect starts cold with no context.</div>
          </div>
        </div>
        <div class="pain-card">
          <div class="pain-icon">📱</div>
          <div>
            <div class="pain-title">No cross-device access</div>
            <div class="pain-desc">Claude Code runs on one machine. You can't reach it from an iPad, phone, or another laptop.</div>
          </div>
        </div>
        <div class="pain-card">
          <div class="pain-icon">🔀</div>
          <div>
            <div class="pain-title">No overview of all your sessions</div>
            <div class="pain-desc">Multiple projects, multiple terminals. No single place to see what's running and what it's doing.</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</section>

<div class="section-divider"></div>

<!-- 3. HOW IT WORKS -->
<section id="how">
  <div class="how-ambient"></div>
  <div class="orb" style="width:500px;height:500px;top:-100px;left:50%;transform:translateX(-50%);background:radial-gradient(circle,rgba(100,0,200,0.08),transparent 70%)"></div>
  <div class="container" style="position:relative;z-index:2">
    <div class="how-header">
      <div class="section-label centered">How it works</div>
      <h2 class="section-h2 centered" style="max-width:500px;margin-left:auto;margin-right:auto">
        A colony that <em>lives</em><br>independently of you
      </h2>
      <p>Two lightweight processes. Sessions in tmux. Connect from any browser.</p>
    </div>

    <div class="architecture">
      <!-- Left: your devices -->
      <div class="arch-col">
        <div class="arch-node green-node">
          <div class="arch-node-title">Browser / PWA</div>
          <div class="arch-node-desc">Any device with a browser. Install as a PWA for a full-screen native feel on iPad.</div>
        </div>
        <div class="arch-node green-node">
          <div class="arch-node-title">Controller Claude</div>
          <div class="arch-node-desc">A Claude instance with MCP tools. Create, read, and manage every session in the colony using natural language.</div>
        </div>
        <div class="arch-node green-node">
          <div class="arch-node-title">iTerm2 / Terminal</div>
          <div class="arch-node-desc">Open any session directly as a local iTerm2 window via the Agent.</div>
        </div>
      </div>

      <!-- Centre: architecture SVG -->
      <div style="display:flex;justify-content:center;align-items:center;padding:20px 0">
        <svg viewBox="0 0 260 360" width="260" height="360" aria-hidden="true">
          <defs>
            <marker id="arrowD" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
              <path d="M0,0 L0,6 L8,3 z" fill="rgba(160,96,255,0.5)"/>
            </marker>
            <marker id="arrowU" markerWidth="8" markerHeight="8" refX="2" refY="3" orient="auto">
              <path d="M8,0 L8,6 L0,3 z" fill="rgba(160,96,255,0.5)"/>
            </marker>
          </defs>
          <!-- Agent -->
          <rect x="70" y="10" width="120" height="56" rx="10" fill="rgba(0,255,140,0.06)" stroke="rgba(0,255,140,0.3)" stroke-width="1.5"/>
          <text x="130" y="36" text-anchor="middle" font-family="Outfit,sans-serif" font-size="13" font-weight="600" fill="rgba(0,255,140,0.9)">Agent</text>
          <text x="130" y="53" text-anchor="middle" font-family="Outfit,sans-serif" font-size="10" fill="rgba(0,255,140,0.5)">MCP · iTerm2 · local</text>
          <!-- REST arrows -->
          <line x1="130" y1="66"  x2="130" y2="136" stroke="rgba(160,96,255,0.4)" stroke-width="1.5" marker-end="url(#arrowD)"/>
          <line x1="122" y1="136" x2="122" y2="66"  stroke="rgba(160,96,255,0.3)" stroke-width="1" marker-end="url(#arrowU)" stroke-dasharray="4,3"/>
          <text x="145" y="106" font-family="JetBrains Mono,monospace" font-size="8" fill="rgba(160,96,255,0.4)">REST</text>
          <!-- Server -->
          <rect x="50" y="141" width="160" height="66" rx="10" fill="rgba(160,96,255,0.08)" stroke="rgba(160,96,255,0.4)" stroke-width="1.5"/>
          <circle cx="88" cy="161" r="6" fill="#a060ff" opacity="0.7">
            <animate attributeName="opacity" values="0.7;0.3;0.7" dur="2s" repeatCount="indefinite"/>
          </circle>
          <text x="130" y="167" text-anchor="middle" font-family="Outfit,sans-serif" font-size="14" font-weight="600" fill="rgba(160,96,255,0.95)">Server</text>
          <text x="130" y="184" text-anchor="middle" font-family="Outfit,sans-serif" font-size="10" fill="rgba(160,96,255,0.5)">WebSocket · REST API</text>
          <text x="130" y="198" text-anchor="middle" font-family="Outfit,sans-serif" font-size="10" fill="rgba(160,96,255,0.5)">Dashboard · Auth</text>
          <!-- tmux arrow -->
          <line x1="130" y1="207" x2="130" y2="264" stroke="rgba(160,96,255,0.4)" stroke-width="1.5" marker-end="url(#arrowD)"/>
          <text x="145" y="240" font-family="JetBrains Mono,monospace" font-size="8" fill="rgba(160,96,255,0.4)">tmux</text>
          <!-- tmux box -->
          <rect x="60" y="269" width="140" height="52" rx="10" fill="rgba(255,255,255,0.03)" stroke="rgba(255,255,255,0.12)" stroke-width="1.2"/>
          <text x="130" y="293" text-anchor="middle" font-family="JetBrains Mono,monospace" font-size="12" fill="rgba(255,255,255,0.6)">tmux</text>
          <text x="130" y="310" text-anchor="middle" font-family="Outfit,sans-serif" font-size="10" fill="rgba(255,255,255,0.3)">Sessions persist here</text>
          <!-- Browser WebSocket arc (left side) -->
          <path d="M50,174 Q5,174 5,110 Q5,46 70,30" fill="none" stroke="rgba(0,255,140,0.2)" stroke-width="1" stroke-dasharray="4,4" marker-end="url(#arrowD)"/>
          <text x="3" y="145" font-family="JetBrains Mono,monospace" font-size="7" fill="rgba(0,255,140,0.3)" transform="rotate(-90,3,145)">WebSocket</text>
        </svg>
      </div>

      <!-- Right: sessions -->
      <div class="arch-col right">
        <div class="arch-node">
          <div class="arch-node-title">Session — api-dev</div>
          <div class="arch-node-desc">A live tmux pane. Claude Code running your API project. Accessible from browser or iTerm2.</div>
        </div>
        <div class="arch-node">
          <div class="arch-node-title">Session — frontend</div>
          <div class="arch-node-desc">Another tmux pane. Completely independent. Persists whether you're connected or not.</div>
        </div>
        <div class="arch-node dim-node">
          <div class="arch-node-title">Session — dormant</div>
          <div class="arch-node-desc">Waiting. The process lives in tmux. Reconnect instantly whenever you need it.</div>
        </div>
      </div>
    </div>
  </div>
</section>

<div class="section-divider"></div>
```

- [ ] **Step 2: Verify sections render**

```bash
cd docs
bundle exec jekyll serve --baseurl ""
```

Open `http://localhost:4000`. Scroll past hero. Verify:
- Problem section has dark (`--deep`) background, 3 pain cards render correctly
- How it works shows 3-column layout with architecture SVG in centre
- SVG diagram shows Agent → Server → tmux stack with arrows

- [ ] **Step 3: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/index.html
git commit -m "feat(site): add problem and how-it-works sections

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 6: Features and Get started sections

**Files:**
- Modify: `docs/index.html` (append two sections)

- [ ] **Step 1: Append features and get-started sections to `index.html`**

Add after the closing `<div class="section-divider"></div>` of the how-it-works section:

```html
<!-- 4. FEATURES -->
<section id="features">
  <div class="container">
    <div class="features-header">
      <div class="section-label centered">What you get</div>
      <h2 class="section-h2 centered">Built for how you <em>actually</em> work</h2>
    </div>
    <div class="features-grid">
      <div class="feature-card green">
        <div class="feature-icon">⬡</div>
        <div class="feature-title">Sessions that never die</div>
        <div class="feature-desc">tmux is the source of truth. Close your browser, shut your laptop — sessions keep running. Reconnect from any device and pick up exactly where you left off.</div>
      </div>
      <div class="feature-card">
        <div class="feature-icon">🌐</div>
        <div class="feature-title">Any device, any browser</div>
        <div class="feature-desc">Full xterm.js terminal in the browser. Install as a PWA on iPad for a native full-screen experience. No app to install, no SSH keys to manage — just a URL.</div>
      </div>
      <div class="feature-card magenta">
        <div class="feature-icon">🧠</div>
        <div class="feature-title">Controller Claude</div>
        <div class="feature-desc">A Claude instance with MCP tools that sees your entire colony. Create sessions, read output, send commands — using natural language to manage your Claude fleet.</div>
      </div>
      <div class="feature-card">
        <div class="feature-icon">🔐</div>
        <div class="feature-title">Passkey auth + API keys</div>
        <div class="feature-desc">WebAuthn passkey login for browser sessions. API key auth for the Agent. Rate-limited, invite-only registration. Secure by default.</div>
      </div>
    </div>
  </div>
</section>

<div class="section-divider"></div>

<!-- 5. GET STARTED -->
<section id="install">
  <div class="install-ambient"></div>
  <div class="container" style="position:relative;z-index:2">
    <div class="install-header">
      <div class="section-label centered">Get started</div>
      <h2 class="section-h2 centered">Colony running in <em>three steps</em></h2>
    </div>
    <div class="steps">
      <div class="step" data-n="01">
        <div class="step-title">Download the binary</div>
        <div class="step-body">Grab the latest release for macOS from the GitHub releases page. A single native binary — no JVM required. Fast startup, low memory.</div>
      </div>
      <div class="step" data-n="02">
        <div class="step-title">Start the server</div>
        <div class="step-body">Run the server on your Mac. It manages tmux sessions and serves the web dashboard on port 7777. Open the URL in any browser to see your colony.</div>
      </div>
      <div class="step" data-n="03">
        <div class="step-title">Connect the agent</div>
        <div class="step-body">Start the agent locally to give your controller Claude MCP access to the full colony. Point it at the server URL.</div>
      </div>
    </div>
    <div class="code-block">
      <div><span class="comment"># Start the server (Mac Mini / always-on machine)</span></div>
      <div><span class="code-prompt">$ </span>./claudony server --bind 0.0.0.0</div>
      <div>&nbsp;</div>
      <div><span class="comment"># Start the agent (your MacBook)</span></div>
      <div><span class="code-prompt">$ </span>./claudony agent --server http://mac-mini.local:7777</div>
    </div>
    <div class="install-cta">
      <a href="https://github.com/mdproctor/claudony/releases" class="btn-primary" style="font-size:15px;padding:14px 36px">⬡ Download Claudony</a>
    </div>
  </div>
</section>

<div class="section-divider"></div>
```

- [ ] **Step 2: Verify sections render**

```bash
cd docs
bundle exec jekyll serve --baseurl ""
```

Open `http://localhost:4000`. Scroll to features and install sections. Verify:
- 2×2 feature card grid renders with accent-coloured icons
- Three numbered step cards visible (numbered badge at top-left)
- Code block shows dark background with green text
- "Download Claudony" button visible and links to GitHub releases

- [ ] **Step 3: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/index.html
git commit -m "feat(site): add features and get-started sections

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 7: Blog preview section and footer

**Files:**
- Modify: `docs/index.html` (append final sections — page is now complete)

- [ ] **Step 1: Append blog preview section to `index.html`**

Add after the closing `<div class="section-divider"></div>` of the get-started section:

```html
<!-- 6. BLOG PREVIEW -->
<section id="blog">
  <div class="container">
    <div class="blog-header">
      <div>
        <div class="section-label">Development diary</div>
        <h2 class="section-h2" style="margin-bottom:0">How we built it</h2>
      </div>
      <a href="{{ '/blog/' | prepend: site.baseurl }}" class="blog-all">All entries →</a>
    </div>
    <div class="blog-grid">
      {% for post in site.posts limit:3 %}
      <div class="blog-card">
        <a href="{{ post.url | prepend: site.baseurl }}">
          <div class="blog-date">{{ post.date | date: "%Y-%m-%d" }}</div>
          <div class="blog-title">{{ post.title }}</div>
          <div class="blog-excerpt">{{ post.excerpt | strip_html | truncatewords: 28 }}</div>
        </a>
      </div>
      {% else %}
      <div class="blog-card" style="grid-column:1/-1;text-align:center;color:var(--dimmer)">
        Blog posts coming soon.
      </div>
      {% endfor %}
    </div>
  </div>
</section>
```

- [ ] **Step 2: Verify blog preview renders (may show placeholder until posts are migrated)**

```bash
cd docs
bundle exec jekyll serve --baseurl ""
```

Open `http://localhost:4000`. Scroll to bottom. Verify:
- Blog section header and "All entries →" link visible
- Either shows "Blog posts coming soon" placeholder or post cards if posts exist
- Full page scrolls correctly end-to-end

- [ ] **Step 3: Commit — landing page HTML is now complete**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/index.html
git commit -m "feat(site): complete landing page — blog preview section added

All seven sections now present. Blog preview uses Liquid to pull
from _posts; shows placeholder until posts are migrated.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 8: Blog listing page, post layout, and post CSS

**Files:**
- Create: `docs/assets/css/post.css`
- Create: `docs/_layouts/post.html`
- Create: `docs/blog/index.html`

- [ ] **Step 1: Write `post.css`**

Create `docs/assets/css/post.css`:
```css
/* ── BLOG LISTING ── */
.blog-listing { padding: 80px 0; min-height: 70vh; }
.blog-listing-header { margin-bottom: 48px; }
.blog-listing-header .section-h2 { margin-bottom: 0; }

.post-list { display: flex; flex-direction: column; gap: 20px; max-width: 720px; }

.post-list-item {
  padding: 24px 28px;
  background: rgba(255,255,255,0.02);
  border: 1px solid rgba(255,255,255,0.07);
  border-radius: 10px; transition: border-color 0.2s;
}
.post-list-item:hover { border-color: rgba(160,96,255,0.3); }
.post-list-item a { display: block; }

.post-list-date  { font-size: 11px; color: var(--dimmer); font-family: var(--mono); margin-bottom: 8px; }
.post-list-title { font-size: 18px; font-weight: 600; color: var(--text); margin-bottom: 8px; line-height: 1.35; }
.post-list-item:hover .post-list-title { color: var(--violet); }
.post-list-excerpt { font-size: 14px; color: var(--dimmer); line-height: 1.6; }

/* ── BLOG POST ── */
.post-page { padding: 80px 0; }
.post-container { max-width: 720px; margin: 0 auto; padding: 0 48px; }

.post-header { margin-bottom: 48px; }
.post-date  { font-size: 12px; color: var(--dimmer); font-family: var(--mono); margin-bottom: 16px; }
.post-title {
  font-size: clamp(28px, 4vw, 40px); font-weight: 700;
  line-height: 1.2; color: #fff; margin-bottom: 0;
}

.post-body { font-size: 16px; color: var(--dim); line-height: 1.85; font-weight: 300; }
.post-body h2 { font-size: 22px; font-weight: 600; color: var(--text); margin: 48px 0 16px; }
.post-body h3 { font-size: 18px; font-weight: 600; color: var(--text); margin: 36px 0 12px; }
.post-body p  { margin-bottom: 20px; }
.post-body ul, .post-body ol { margin: 0 0 20px 24px; }
.post-body li { margin-bottom: 8px; }
.post-body strong { color: var(--text); font-weight: 600; }
.post-body em { color: var(--green); font-style: normal; }
.post-body a { color: var(--violet); text-decoration: underline; text-underline-offset: 3px; }
.post-body a:hover { color: var(--green); }
.post-body hr {
  border: none; border-top: 1px solid rgba(160,96,255,0.15);
  margin: 48px 0;
}
.post-body code {
  font-family: var(--mono); font-size: 0.875em;
  background: rgba(160,96,255,0.08); color: var(--green);
  padding: 2px 6px; border-radius: 4px;
}
.post-body pre {
  background: rgba(0,0,0,0.4);
  border: 1px solid rgba(160,96,255,0.2);
  border-radius: 10px; padding: 20px 24px;
  overflow-x: auto; margin-bottom: 24px;
}
.post-body pre code {
  background: none; padding: 0; font-size: 13px;
  color: var(--green); line-height: 1.7;
}
.post-body blockquote {
  border-left: 2px solid var(--violet);
  padding-left: 20px; margin: 24px 0;
  color: var(--dimmer); font-style: italic;
}

.post-nav {
  display: flex; justify-content: space-between;
  margin-top: 64px; padding-top: 32px;
  border-top: 1px solid rgba(160,96,255,0.1);
  font-size: 13px;
}
.post-nav a { color: var(--violet); }
.post-nav a:hover { text-decoration: underline; text-underline-offset: 3px; }

/* ── DOC PAGE ── */
.doc-page { padding: 80px 0; }
.doc-container { max-width: 720px; margin: 0 auto; padding: 0 48px; }
.doc-header { margin-bottom: 48px; }
.doc-title { font-size: clamp(28px, 4vw, 40px); font-weight: 700; color: #fff; }

.doc-body { font-size: 16px; color: var(--dim); line-height: 1.85; font-weight: 300; }
.doc-body h2 { font-size: 22px; font-weight: 600; color: var(--text); margin: 48px 0 16px; }
.doc-body h3 { font-size: 18px; font-weight: 600; color: var(--text); margin: 36px 0 12px; }
.doc-body p  { margin-bottom: 20px; }
.doc-body ul, .doc-body ol { margin: 0 0 20px 24px; }
.doc-body li { margin-bottom: 8px; }
.doc-body strong { color: var(--text); font-weight: 600; }
.doc-body code {
  font-family: var(--mono); font-size: 0.875em;
  background: rgba(160,96,255,0.08); color: var(--green);
  padding: 2px 6px; border-radius: 4px;
}
.doc-body pre {
  background: rgba(0,0,0,0.4);
  border: 1px solid rgba(160,96,255,0.2);
  border-radius: 10px; padding: 20px 24px;
  overflow-x: auto; margin-bottom: 24px;
}
.doc-body pre code {
  background: none; padding: 0; font-size: 13px;
  color: var(--green); line-height: 1.7;
}

@media (max-width: 768px) {
  .post-container, .doc-container { padding: 0 24px; }
}
```

- [ ] **Step 2: Write `_layouts/post.html`**

Create `docs/_layouts/post.html`:
```html
---
layout: default
---
<article class="post-page">
  <div class="post-container">
    <div class="post-header">
      <div class="post-date">{{ page.date | date: "%Y-%m-%d" }}</div>
      <h1 class="post-title">{{ page.title }}</h1>
    </div>
    <div class="post-body">
      {{ content }}
    </div>
    <div class="post-nav">
      {% if page.previous.url %}
        <a href="{{ page.previous.url | prepend: site.baseurl }}">← {{ page.previous.title }}</a>
      {% else %}
        <span></span>
      {% endif %}
      {% if page.next.url %}
        <a href="{{ page.next.url | prepend: site.baseurl }}">{{ page.next.title }} →</a>
      {% endif %}
    </div>
  </div>
</article>
```

- [ ] **Step 3: Write `blog/index.html`**

Create `docs/blog/index.html`:
```html
---
layout: default
title: Development Diary
---
<div class="blog-listing">
  <div class="container">
    <div class="blog-listing-header">
      <div class="section-label">Development diary</div>
      <h1 class="section-h2">How Claudony was built</h1>
    </div>
    <div class="post-list">
      {% for post in site.posts %}
      <div class="post-list-item">
        <a href="{{ post.url | prepend: site.baseurl }}">
          <div class="post-list-date">{{ post.date | date: "%Y-%m-%d" }}</div>
          <div class="post-list-title">{{ post.title }}</div>
          <div class="post-list-excerpt">{{ post.excerpt | strip_html | truncatewords: 35 }}</div>
        </a>
      </div>
      {% endfor %}
    </div>
  </div>
</div>
```

- [ ] **Step 4: Verify blog listing builds**

```bash
cd docs
bundle exec jekyll serve --baseurl ""
```

Open `http://localhost:4000/blog/`. Verify: blog listing page renders, "Development Diary" heading visible. Empty list is fine until Task 9.

- [ ] **Step 5: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/assets/css/post.css docs/_layouts/post.html docs/blog/index.html
git commit -m "feat(site): add blog listing, post layout, and post/doc CSS

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 9: Migrate existing blog posts

**Files:**
- Create: `docs/_posts/` (12 files)

The 12 existing posts in `docs/blog-archive/` have no Jekyll front matter. Each gets:
1. A YAML front matter block added at the top
2. The opening `# Title` line and `**Date:**` / `**Type:**` metadata stripped (captured in front matter)
3. A new filename in `_posts/` using the Jekyll date-title format

The title comes from the `# Title` line (strip leading `# ` and `RemoteCC — ` prefix, replacing with `Claudony — `).

- [ ] **Step 1: Create the post for "From SSH to PWA"**

Create `docs/_posts/2026-04-03-from-ssh-to-pwa.md`:
```markdown
---
layout: post
title: "Claudony — From SSH to PWA"
date: 2026-04-03
---

## What We Were Trying To Achieve
```
Then paste all content from `docs/blog-archive/2026-04-03-mdp01-from-ssh-to-pwa.md` starting from the first `##` heading (skip the `# Title`, `**Date:**`, `**Type:**`, and `---` lines at the top).

- [ ] **Step 2: Verify the first post builds and renders**

```bash
cd docs
bundle exec jekyll serve --baseurl ""
```

Open `http://localhost:4000/blog/`. Verify: post appears in listing. Click through to verify post page renders with styled body text, headings, and previous/next nav.

- [ ] **Step 3: Create remaining 11 posts**

Repeat the same pattern for each file. Front matter titles (strip `RemoteCC — ` prefix, add `Claudony — `):

| Source file | `_posts/` filename | title |
|---|---|---|
| `2026-04-03-mdp02-server-core-and-native.md` | `2026-04-03-server-core-and-native.md` | `Claudony — Server Core and Native Image` |
| `2026-04-03-mdp03-the-ipad-itch.md` | `2026-04-03-the-ipad-itch.md` | `Claudony — The iPad Itch` |
| `2026-04-04-mdp01-terminal-rendering-saga.md` | `2026-04-04-terminal-rendering-saga.md` | `Claudony — The Terminal Rendering Saga` |
| `2026-04-04-mdp02-wiring-the-control-plane.md` | `2026-04-04-wiring-the-control-plane.md` | `Claudony — Wiring the Control Plane` |
| `2026-04-05-mdp01-testing-what-wasnt-tested.md` | `2026-04-05-testing-what-wasnt-tested.md` | `Claudony — Testing What Wasn't Tested` |
| `2026-04-05-mdp02-locking-the-door.md` | `2026-04-05-locking-the-door.md` | `Claudony — Locking the Door` |
| `2026-04-06-mdp01-closing-the-gaps.md` | `2026-04-06-closing-the-gaps.md` | `Claudony — Closing the Gaps` |
| `2026-04-07-mdp01-two-silent-bugs-and-a-working-binary.md` | `2026-04-07-two-silent-bugs-and-a-working-binary.md` | `Claudony — Two Silent Bugs and a Working Binary` |
| `2026-04-07-mdp02-zero-configuration.md` | `2026-04-07-zero-configuration.md` | `Claudony — Zero Configuration` |
| `2026-04-07-mdp03-terminal-was-lying-about-cursor.md` | `2026-04-07-terminal-was-lying-about-cursor.md` | `Claudony — The Terminal Was Lying About the Cursor` |
| `2026-04-09-mdp01-the-compose-took-five-tries.md` | `2026-04-09-the-compose-took-five-tries.md` | `Claudony — The Compose Took Five Tries` |

Each file front matter:
```markdown
---
layout: post
title: "Claudony — [title from table above]"
date: [date from filename]
---
```
Then all content from the first `##` heading onwards (no other changes to body content).

- [ ] **Step 4: Verify all 12 posts appear in blog listing**

```bash
cd docs
bundle exec jekyll build --baseurl "" 2>&1 | grep -E "(Post|Error|Warning)"
bundle exec jekyll serve --baseurl ""
```

Open `http://localhost:4000/blog/`. Verify: 12 posts listed in reverse chronological order. Click a few to verify they render correctly.

- [ ] **Step 5: Verify blog preview on landing page shows 3 most recent posts**

Open `http://localhost:4000`. Scroll to blog section. Verify: 3 most recent post cards visible with title and excerpt.

- [ ] **Step 6: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/_posts/
git commit -m "feat(site): migrate 12 blog posts with Jekyll front matter

Strips RemoteCC branding, adds Claudony prefix to all titles.
Posts migrated from docs/blog-archive/ to docs/_posts/.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 10: Getting started guide page

**Files:**
- Create: `docs/_layouts/doc.html`
- Create: `docs/guide/index.md`

- [ ] **Step 1: Write `_layouts/doc.html`**

Create `docs/_layouts/doc.html`:
```html
---
layout: default
---
<div class="doc-page">
  <div class="doc-container">
    <div class="doc-header">
      <div class="section-label">{{ page.section | default: "Docs" }}</div>
      <h1 class="doc-title">{{ page.title }}</h1>
    </div>
    <div class="doc-body">
      {{ content }}
    </div>
  </div>
</div>
```

- [ ] **Step 2: Write `guide/index.md`**

Create `docs/guide/index.md`:
```markdown
---
layout: doc
title: Getting Started
section: Guide
description: Install and run Claudony — server, agent, and first session in minutes.
---

Claudony is a single binary with two modes: **server** and **agent**. The server runs on the machine that hosts your Claude Code sessions. The agent runs on your local machine and exposes an MCP endpoint for your controller Claude.

## Requirements

- macOS (Apple Silicon or Intel)
- [tmux](https://github.com/tmux/tmux) installed (`brew install tmux`)
- [Claude Code CLI](https://code.claude.com) authenticated

## Download

Grab the latest binary from [GitHub Releases](https://github.com/mdproctor/claudony/releases).

```bash
# Make it executable
chmod +x claudony
# Optional: move to PATH
mv claudony /usr/local/bin/claudony
```

## Start the Server

Run this on the machine that will host your sessions (Mac Mini, always-on MacBook, etc.):

```bash
claudony server --bind 0.0.0.0
```

On first run, the server prints an API key and opens the registration page at `http://localhost:7777/auth/register`. Register your passkey — this is how you log in from any browser.

**To persist sessions across server restarts**, set the encryption key env var so auth cookies survive:

```bash
QUARKUS_HTTP_AUTH_SESSION_ENCRYPTION_KEY=your-secret-32-chars \
  claudony server --bind 0.0.0.0
```

Default port is `7777`. Set `--port` to change it.

## Open the Dashboard

Open `http://your-server:7777/app/` in any browser. Log in with your passkey. You'll see the session dashboard — empty for now.

Install it as a PWA from your browser's menu for a full-screen native experience on iPad.

## Create Your First Session

From the dashboard, click **New Session**. Give it a name and a working directory. The server starts a new tmux pane and runs `claude` in it. You'll see the terminal appear in your browser.

## Start the Agent (optional)

The agent gives your controller Claude MCP tools to manage the colony. Run this on your local machine (the machine with iTerm2):

```bash
claudony agent --server http://your-server:7777
```

Then add the MCP endpoint to your controller Claude's config:

```json
{
  "mcpServers": {
    "claudony": {
      "url": "http://localhost:7778/mcp"
    }
  }
}
```

Your controller Claude can now `list_sessions`, `create_session`, `send_input`, `read_output`, and `open_in_terminal` for any session in the colony.

## Configuration

Key settings (can be passed as flags or env vars):

| Property | Default | Description |
|---|---|---|
| `remotecc.mode` | `server` | `server` or `agent` |
| `remotecc.port` | `7777` | HTTP port |
| `remotecc.bind` | `localhost` | Bind address (`0.0.0.0` for remote) |
| `remotecc.server.url` | `http://localhost:7777` | Agent → Server URL |
| `remotecc.default-working-dir` | `~/remotecc-workspace` | Default dir for new sessions |

## What's Next

- Read the [development diary](/claudony/blog/) to understand how it was built
- Open an issue on [GitHub](https://github.com/mdproctor/claudony) if something breaks
```

- [ ] **Step 3: Verify guide page builds and renders**

```bash
cd docs
bundle exec jekyll serve --baseurl ""
```

Open `http://localhost:4000/guide/`. Verify:
- "Guide" label and "Getting Started" title render
- Code blocks show dark background with green text
- Table renders correctly
- Links in nav ("Docs" in footer) resolve to this page

- [ ] **Step 4: Commit**

```bash
cd /Users/mdproctor/claude/claudony
git add docs/_layouts/doc.html docs/guide/index.md
git commit -m "feat(site): add getting started guide page

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 11: GitHub Pages setup and deploy

**Note:** GitHub Pages "Deploy from branch" uses Jekyll 3.9.x and is incompatible with Jekyll 4.x. We must use GitHub Actions to build and deploy instead.

**Files:**
- Modify: `.gitignore` (add `docs/_site/` and `docs/.jekyll-cache/`)
- Create: `.github/workflows/jekyll.yml`

- [ ] **Step 1: Add Jekyll build artifacts to `.gitignore`**

Add to `/Users/mdproctor/claude/claudony/.gitignore`:
```
docs/_site/
docs/.jekyll-cache/
docs/vendor/
```

- [ ] **Step 2: Create GitHub Actions workflow**

Create `.github/workflows/jekyll.yml` at the repo root:
```yaml
name: Deploy Jekyll site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true
          working-directory: docs
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Build with Jekyll
        run: bundle exec jekyll build --baseurl "${{ steps.pages.outputs.base_path }}"
        working-directory: docs
        env:
          JEKYLL_ENV: production
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: docs/_site

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

- [ ] **Step 3: Final local build — check for any warnings**

```bash
cd /Users/mdproctor/claude/claudony/.worktrees/feat-landing-page/docs
bundle exec jekyll build --baseurl "/claudony" 2>&1
```

Expected: Clean build with 12 posts, 0 errors.

- [ ] **Step 4: Enable GitHub Pages with GitHub Actions source**

In a browser:
1. Go to `https://github.com/mdproctor/claudony/settings/pages`
2. Source → **GitHub Actions** (not "Deploy from a branch")
3. Click Save — no branch or folder selection needed

- [ ] **Step 5: Commit and push**

```bash
cd /Users/mdproctor/claude/claudony/.worktrees/feat-landing-page
git add .gitignore .github/workflows/jekyll.yml
git commit -m "feat(site): add GitHub Actions deploy workflow for Jekyll 4

Uses actions/jekyll-build-pages compatible approach so Jekyll 4.x
builds correctly. GitHub Pages classic is pinned to Jekyll 3.9.x
and incompatible with our Gemfile.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
git push origin feat/landing-page
```

- [ ] **Step 6: Open PR and verify deployment**

```bash
gh pr create --title "feat(site): Claudony landing page" \
  --body "Jekyll 4 landing page site with bioluminescent colony design. Seven sections, blog, guide. Closes #N" \
  --repo mdproctor/claudony
```

Then monitor the Actions run:
```bash
gh run list --repo mdproctor/claudony --limit 5
```

Once merged to main and the Actions workflow runs, verify at:
```bash
open https://mdproctor.github.io/claudony/
```

Verify:
- Landing page loads with bioluminescent colony design
- Nav links work (How it works, Features, Get started, Blog)
- Colony SVG animation plays
- Blog at `/claudony/blog/` shows all 12 posts
- Guide at `/claudony/guide/` renders correctly
- Footer links resolve correctly
- Site is mobile-responsive (check with browser dev tools)

- [ ] **Step 7: Commit any fixes found during verification**

If any issues found (broken links, missing assets, layout problems), fix and push:
```bash
git add -p
git commit -m "fix(site): [describe what was broken]"
git push origin feat/landing-page
```
