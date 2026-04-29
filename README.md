# Studio Darkroom — Case Study

> A focused media-workflow plugin for WordPress. Folder management, attachment curation, focal points, gallery output, and a diagnostics surface for site admins — built for sites that handle thousands of images.

🚧 **Pre-release.** Source code lives in a private repo. This page documents the architecture, engineering decisions, and screenshots.

---

## The problem

WordPress's default media library was designed in 2010 for sites with a few dozen attachments. It scales poorly past **~500 items**:

- The grid view has no nesting, no folders, no real filtering — only a flat list ordered by date
- "Edit alt text" is a five-click round trip per image; doing it for 200 images is unworkable
- There's no way to find duplicate uploads, oversized images, or images missing alt text without manual one-by-one inspection
- Themes that need image cropping (16:9 hero shots, 4:5 social posts) can't store per-image focal points natively, so faces end up cropped out of cover images
- Bulk operations are limited to "delete" — there's no bulk move, bulk tag, or bulk rate

Existing folder plugins solve **part** of the problem but bolt on a single layer (folders) without rethinking the workflow.

## What I built

A WordPress backend plugin that rebuilds the media workflow around the lifecycle: **organize → curate → enrich → publish → audit.**

### Folder organization
Drag-and-drop folders with arbitrary nesting. Per-folder sort order. Pinned folders. Stored as a custom WordPress taxonomy so every plugin that respects taxonomies sees folders for free — search, REST API, cache, export.

### Smart folders
Auto-generated views that update as the library changes:

| | |
|---|---|
| Missing Alt Text | Large Images |
| Unused Media | Duplicates (content-hash matched) |
| Recently Uploaded | SVG / PDF / Video |
| Favorites | Hero / Brand / Campaign tags |
| Rating 1–5 stars | |

These aren't separate tables — they're virtual views composed of `WP_Query` arguments, registered through a smart-folder registry so future features (or Pro-tier add-ons) can register their own without modifying core.

### Focal point system
Per-attachment focal coordinates plus aspect-ratio overrides (1:1, 16:9, 4:5). Themes consume via helper functions or the bundled gallery shortcode — no more cropped faces in cover images.

```
_studiodarkroom_focal           = {x: 0.5, y: 0.5}     // default
_studiodarkroom_focal_1x1       = {x: 0.5, y: 0.5}     // square
_studiodarkroom_focal_16x9      = {x: 0.5, y: 0.4}     // wide
_studiodarkroom_focal_4x5       = {x: 0.5, y: 0.6}     // portrait
```

CSS uses these as `object-position` values for `cover` / `contain` fits.

### Gallery module
Build presets in the admin once, output via shortcode. Manual selection or auto-sync to a folder. Swiper-powered carousels or grid layouts. Lightbox, responsive aspect ratios, focal points respected.

### Bulk operations
Move, delete, tag, pin, rate across selections. Multi-select grid with shift-click range, ctrl-click toggle.

### SVG uploads (per-role opt-in)
Sanitized via [`enshrined/svg-sanitize`](https://github.com/darylldoyle/svg-sanitize) before reaching the uploads directory. Disabled by default; opt-in per role.

### Diagnostics surface
Environment checks, recent failure log, license status, role-preview matrix, shareable scrubbed crash reports. Built so support can debug a customer's site without asking them to install a debugger.

---

## Engineering decisions worth talking about

### Dual-layer PHP architecture

The codebase has been migrating from a procedural codebase (`/inc`) to a namespaced, class-based one (`/app`). Both layers run side-by-side. **New features land in `/app`; legacy code stays where it works.**

```
┌──────────────────────────────────┐  ┌──────────────────────────────────┐
│  /app  (modern, namespaced)      │  │  /inc  (legacy, procedural)      │
│  StudioDarkroom\Folders\…        │  │  function snapmedia_…()          │
│  PSR-4 autoloaded                │  │  Loaded via require_once         │
│  Class-based, dependency-aware   │  │  Snake-case helpers              │
└──────────────────────────────────┘  └──────────────────────────────────┘
```

This is the kind of decision that's easy to get wrong. Two common failures:

1. **Big-bang rewrite** — fork the codebase, rebuild from scratch, ship months later (or never).
2. **Endless deprecation** — drag legacy code along forever, never finish migrating.

The chosen path: **gradual migration tied to feature work**. Touching a legacy cluster for a bug fix? Stay surgical. Adding a feature that needs to interact with the cluster? Migrate the cluster first, then add the feature. The legacy layer shrinks one cohesive unit at a time, no big-bang risk, no permanent technical debt.

See [architecture.md](architecture.md) for the full pattern.

### License state machine

Licensing is one of the harder problems in commercial WordPress plugins. Network failures, server downtime, customer site offline — any of these will trigger a license check failure. The naive implementation locks the plugin and angry customers email support.

The state machine handles this gracefully:

```
unactivated  →  activate(key)   → active
active       →  revalidate()   → active | grace | expired
grace        →  revalidate()   → active | expired
expired      →  activate(key)  → active | invalid
invalid      → (terminal until user pastes new key)
```

When the remote license server is unreachable, the manager enters a **7-day grace period** before downgrading. This keeps customer sites working through transient outages — the plugin only locks if the server has been unreachable for a full week.

`wp_get_environment_type()` returning `local`, `development`, or `staging` triggers a development bypass — administrators can run unlicensed for local work. Production stays strict.

### Lite ↔ Pro safety contract

The plugin ships in two editions: free Lite (folder basics) and licensed Pro (gallery, bulk ops, advanced smart folders). The hard part isn't gating features — it's letting customers downgrade safely.

The **swap contract** guarantees:

- Downgrading Pro → Lite preserves all folder structure and attachment metadata
- Pro-only data (gallery presets, advanced workflow tags) is **frozen but not destroyed**
- Re-upgrading restores everything exactly as it was

This is what makes Lite safe to install on production. Customers can experiment with Lite, upgrade for evaluation, drop back if they don't want to renew, and **never lose their data**. That trust is what justifies the price tier.

### Crash guard

WordPress's default error handler renders a "There has been a critical error" page. Useless for debugging. Worse, fatal errors during early hooks happen before WordPress even initializes its handler, so visitors see a raw PHP stack trace.

The plugin registers its own fatal-error handler that:

1. Catches early-hook fatals before WP's screen renders
2. Captures a scrubbed environment snapshot (no PII)
3. Forwards to Sentry (production only, opt-in)
4. Renders a graceful admin notice instead of a white screen of death

The Diagnostics page surfaces those captured failures plus environment data, role-preview matrices, and a "Copy report" button that produces sharable Markdown for support tickets.

### Custom translation token dictionary

WordPress's gettext system handles standard strings well, but the plugin needs to support **admin UI language independent of site locale** — agency sites where the staff speaks Spanish but the public site is English.

The fix: a custom translation token dictionary that intercepts every `__()` call via the `gettext` filter hook, consults the dictionary first, and falls back to standard `.mo` lookup. The dictionary is keyed per-language; admin UI language is a per-user preference.

Three languages shipped (en, es, fr).

---

## Tech stack

- **PHP 8.1+** — typed properties, named arguments, modern syntax throughout
- **WordPress 6.4+** — minimum version that supports the REST API patterns used
- **Composer** for runtime dependencies (Sentry SDK, svg-sanitize, Swiper)
- **PHPUnit 9** for unit + integration tests
- **No frontend build step** — vanilla JS + CSS. No webpack, no React, no build pipeline.
- **Custom taxonomy** for folders so the data model is portable
- **WP REST API + admin-ajax** hybrid — REST for new endpoints, admin-ajax for legacy code paths

---

## Architecture

For the full technical breakdown — file map, key systems, license validation flow, security model, performance considerations — see [architecture.md](architecture.md).

For where the project is going — what's shipped, what's next, what won't ship — see [roadmap.md](roadmap.md).

---

## Status

**v1.0.0 (pre-release).** Currently in the 1.0.1 polish pass:

- Lemon Squeezy licensing + payment integration polish
- Security hardening across the AJAX surface
- Crash-guard expansion + Sentry tagging
- Internal QA across PHP 8.1 / 8.2 / 8.3 and WordPress 6.4 / 6.5 / 6.6 / 6.7

---

## Screenshots

> Screenshots TK — capturing during the v1.0.1 release pass.

```
./screenshots/01-folder-grid.png        # Main library view with folder sidebar
./screenshots/02-attachment-detail.png  # Attachment detail page with focal-point editor
./screenshots/03-smart-folders.png      # Smart-folder picker
./screenshots/04-gallery-preset.png     # Gallery preset builder
./screenshots/05-diagnostics.png        # Diagnostics page
./screenshots/06-bulk-ops.png           # Bulk operation toolbar
```

---

## About

Built by **Mark Hernandez** ([Studio Grid](https://github.com/studiogrid)). Part of the Studio Grid family alongside:

- **Studio Player** — multi-mode media player with native Bricks/Gutenberg/Divi/Elementor integrations and true cross-page continuous playback
- **Studio Media** — Bricks-native gallery, video, Lottie, and filter elements
- **StudioDashboard** — modern WordPress admin dashboard with built-in analytics

This is a private commercial plugin. Source code is not public. This page exists to document the engineering for prospective collaborators and employers.

For licensing inquiries: [studiodarkroom.com](https://studiodarkroom.com).
