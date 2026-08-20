# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

The personal website at **https://damian.perera.io** (GitHub Pages, custom domain). The entire
production site is **one hand-authored static file**: `web/public/index.html` (~1500 lines, with all CSS
in a single inline `<style>`, all SVG inline, and all JS in an inline `<script>`). Edit that file directly.

### The React shell is vestigial — do not build features in it

`web/` is a Create React App project used **only as a static-file bundler**. `web/src/index.js` has its
React bootstrap commented out (only `reportWebVitals()` runs), so `npm start` renders nothing useful.
`react-scripts build` simply copies `web/public/` verbatim into `web/build/`, which is what Pages serves.
Ignore `web/src/`. There are no real tests (`web/src/App.test.js` is CRA boilerplate).

## Commands

```bash
# Preview the real site (serve the static files, then open the URL):
cd web/public && python3 -m http.server 8899          # then http://localhost:8899/
# Production build (only meaningful as a copy step for Pages):
cd web && npm run build                               # -> web/build/
```

- **Do not use `npm start`** (CRA dev server) — the React app is inert.
- The `file://` preview cannot resolve the site's absolute `/assets/...` paths; always serve
  `web/public` over HTTP when driving it in a browser.

## Deploy

`.github/workflows/deploy.yml` runs on **push to `master`**: `npm run build` in `web/`, then deploys
`web/build/` to GitHub Pages (environment `damian.perera.io`). **A push to `master` is a live production
deploy — confirm with the user before pushing.** There is no committed `CNAME`; the custom domain lives
in the repo's Pages settings.

## Site architecture (the big picture)

A single-page "chalkboard" scrollytelling experience. Near-black slate background, chalk-white strokes,
one **amber** accent (`--amber: #e8b04b`), **Caveat** (handwriting) for the wordmark + diagram labels and
**Inter** for everything else. Six full-height scenes: `#who`, `#how`, `#skills`, `#career`, `#projects`,
`#contact`, plus the `#top` landing.

**Two-layer model (critical for SEO/accessibility):**
1. **Content layer** — real semantic HTML text (headings, paragraphs, lists, links). This is the
   crawlable, no-JS, reduced-motion site and must carry every fact. Never put information only in SVG.
2. **Chalk layer** — inline SVG diagrams, `aria-hidden`, layered on top. Decorative; they animate.

**Animation** (inline `<script>`, ~line 1330+): GSAP is **self-hosted** at
`web/public/assets/vendor/gsap/gsap.min.js` and driven by **IntersectionObserver, not ScrollTrigger** —
ScrollTrigger caches pixel offsets at setup that go stale once web fonts reflow the page. Each
`.scene[data-scene-index]` gets a paused GSAP timeline that plays when it scrolls into view. Shapes
(`.chalk-draw`, with `pathLength="1"`) draw via `stroke-dashoffset` 1→0; text (`.chalk-text`) writes on
via `clip-path`. Reduced-motion / no-GSAP leaves everything at CSS defaults (fully drawn).

**Section snapping:** CSS scroll-snap — `proximity` on mobile/tablet (never traps tall content),
`mandatory` on desktop (`@media (min-width: 900px)`). Snap centering depends on `scroll-padding-top: 0`
plus the scene's asymmetric padding; changing either shifts where snapped content lands.

**SEO** lives in the `<head>`: meta + Open Graph + Twitter + JSON-LD `@graph`
(Person / WebSite / ProfilePage), plus `sitemap.xml` and `robots.txt`. Keep it valid and truthful.

## Hard constraints (the user cares strongly about these)

- **Do not touch `web/public/impressum/`** — a separate legal-notice page with its own favicons/manifest.
- **Content must trace to the résumé; invent nothing.** The résumé JSON is provided at `/tmp` during
  sessions. Do not publish the phone number or a city — country only ("Germany", never "Munich").
- **No em-dashes (`—`) anywhere.** Plain hyphens are fine. Reader-facing copy is first-person, spare, and
  **tech-agnostic** — do not name specific tools (Kafka/gRPC/etc.) in prose or in the OG/Twitter/meta
  previews; that breadth belongs only in the `keywords` meta and JSON-LD `knowsAbout`.
- Amber is used **sparingly** — roughly one accent per scene.

## SVG chalk-diagram gotchas (learned the hard way)

- **Arrowheads:** SVG `marker-end` paints instantly, so the draw script sets `marker-end="none"` on load
  and restores it (via `data-marker-end`) only after each line finishes drawing. Markers `#arrowhead`
  (chalk) and `#arrowhead-amber` exist in the shared defs.
- **Use `stroke-linecap: butt`, not `round`.** Round caps render a stray dot at a hidden dashed line's
  start point (visible before that line animates).
- **Rotated (vertical) labels:** `clip-path` on a rotated `<text>` evaluates in the element's *rotated
  local* frame, so an `inset()` reveal goes sideways. For a bottom-up write-on, give the text a
  `clip-path="url(#…)"` whose `<clipPath clipPathUnits="userSpaceOnUse">` rect **grows in width** (local
  x-axis) — see the `data-clip-rect` handling in the draw script.
- **Amber on strokes:** `.chalk-amber` sets `stroke: amber !important` (it does not change `fill`); pair
  it with `marker-end="url(#arrowhead-amber)"` when an arrow should be fully amber.
- The How and Skills diagrams have separate **`.diagram-desktop` / `.diagram-mobile`** SVG variants —
  edit both.
