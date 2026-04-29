# Architecture

The engineering deep-dive companion to the [README](README.md). This document covers the patterns and decisions that shape the codebase — the dual-layer migration, the systems that make folders fast on large libraries, the license validation flow, and the security model.

> Source code is private. This page documents the architecture for prospective collaborators, employers, and technical peers.

---

## 1. Dual-layer PHP architecture

The codebase has two PHP layers running side-by-side:

```
┌──────────────────────────────────┐  ┌──────────────────────────────────┐
│  /app  (modern, namespaced)      │  │  /inc  (legacy, procedural)      │
│                                  │  │                                  │
│  StudioDarkroom\Folders\…        │  │  function snapmedia_…()          │
│  StudioDarkroom\Ajax\…           │  │  add_action(…, 'snapmedia_…')    │
│  StudioDarkroom\Security\…       │  │  global $studiodarkroom_…        │
│                                  │  │                                  │
│  PSR-4 autoloaded                │  │  Loaded via require_once         │
│  Class-based, dependency-aware   │  │  Snake-case helpers              │
└──────────────────────────────────┘  └──────────────────────────────────┘
                  ▲                                       ▲
                  │                                       │
                  └───────────────┬───────────────────────┘
                                  │
                       studiodarkroom.php
                       (plugin bootstrap; loads both)
```

### Why two layers

The modern `/app` layer is where new features land — namespaces, typed methods, dependency-aware classes.

The legacy `/inc` layer is where the original procedural code lives — admin-ajax handlers, capability checks, folder model helpers, the license client, crash guard, diagnostics. It's stable and battle-tested.

This was a deliberate decision. Two common failure modes exist for codebases in this position:

1. **Big-bang rewrite** — fork the codebase, rebuild from scratch, ship months later (or never)
2. **Endless deprecation** — drag legacy code along forever, never finish migrating

The chosen path is **gradual migration tied to feature work**. Touching a legacy cluster for a bug fix? Stay surgical. Adding a feature that needs to interact with the cluster? Migrate the cluster first, then add the feature. The legacy layer shrinks one cohesive unit at a time. No big-bang risk, no permanent technical debt.

### Practical rules

- **New features** land in `/app` under the appropriate namespace
- **Bug fixes in legacy code** stay in `/inc` if surgical; migrate the cluster if the fix would touch three+ files
- **Cross-layer calls** — `/app` calling `/inc` is fine (legacy is the substrate); `/inc` calling `/app` is rare and well-commented when it happens
- **Both layers share prefix conventions** — `studiodarkroom_*` for procedural, `StudioDarkroom\*` for namespaced

---

## 2. Directory layout

```
studiodarkroom/
├── app/                          # Modern, namespaced classes (PSR-4)
│   ├── Admin/                    # Admin screens, settings, role-preview matrix
│   ├── Ajax/                     # AJAX controllers (FolderAjaxController, etc.)
│   ├── Api/                      # REST API endpoints
│   ├── Core/                     # Plugin lifecycle, autoloader, container
│   ├── Focus/                    # Focal-point system
│   ├── Folders/                  # Folder taxonomy + smart-folder engine
│   ├── Media/                    # Attachment metadata, hash + duplicate detection
│   ├── Migration/                # Schema/data migrations between versions
│   ├── Modules/                  # Optional feature modules (Gallery, etc.)
│   ├── Security/                 # Capability checks, sanitization, CSRF
│   ├── Support/                  # Diagnostics, env reports, crash dispatcher
│   ├── Telemetry/                # Sentry integration, opt-in usage signals
│   ├── Tools/                    # Bulk operations, import/export
│   ├── UI/                       # Admin UI helpers, asset enqueueing
│   ├── Undo/                     # Trash + restore for folder deletes
│   ├── Uploads/                  # SVG sanitization, MIME hardening
│   └── WooCommerce/              # WC product gallery / image overrides
│
├── inc/                          # Legacy procedural layer (~30 files)
│   ├── capabilities.php
│   ├── crash-guard.php
│   ├── diagnostics.php
│   ├── folder-model.php
│   ├── lite-recovery.php
│   ├── class-studiodarkroom-license-client.php
│   ├── class-studiodarkroom-license-manager.php
│   └── …
│
├── assets/                       # CSS + JS, no build step
├── views/                        # Admin page templates
├── templates/                    # UI components (modals, partials)
├── blocks/                       # Gutenberg blocks
├── bricks-elements/              # Bricks-native elements
├── tests/                        # PHPUnit (unit + integration)
├── docs/                         # Public-facing documentation
├── languages/                    # .pot / .po / .mo
├── licenses/                     # Third-party license notices
├── composer.json
└── phpunit.xml.dist
```

---

## 3. Folder system — data model

Folders are a **custom taxonomy** registered as `studiodarkroom_folder`. Every folder is a standard WordPress term, which means:

- Hierarchy and ordering live in `wp_terms` + `wp_term_taxonomy` (no custom tables)
- Folder ↔ attachment associations live in `wp_term_relationships`
- Every plugin that respects taxonomies sees folders for free — search, REST API, cache, export
- Export/import is a JSON dump of term tree + relationships — survives staging→production migrations

### Smart folders are virtual

Smart folders aren't terms. They're rendered by querying attachments with specific `WP_Query` arguments:

| Smart folder | Query strategy |
|---|---|
| Missing Alt Text | `meta_query` for empty `_wp_attachment_image_alt` |
| Large Images | `meta_query` for filesize > threshold |
| Unused Media | attachments with zero post-relationship |
| Duplicates | grouped by content-hash custom field |
| Recently Uploaded | `date_query` last 7 days |
| SVG / PDF / Video | `mime_type` filter |
| Favorites | `meta_query` `_studiodarkroom_starred=1` |
| Hero / Brand / Campaign | `meta_query` workflow tag |
| Rating 1–5 | `meta_query` rating field |

The smart-folder engine in `app/Folders/` exposes a registry so future features (or Pro-tier add-ons) can register their own smart folders without modifying core.

---

## 4. Focal-point system

Per-attachment focal coordinates are stored in attachment meta:

```
_studiodarkroom_focal           = {x: 0.5, y: 0.5}     // default focal
_studiodarkroom_focal_1x1       = {x: 0.5, y: 0.5}     // square override
_studiodarkroom_focal_16x9      = {x: 0.5, y: 0.4}     // wide override
_studiodarkroom_focal_4x5       = {x: 0.5, y: 0.6}     // portrait override
```

Themes consume via:

```php
studiodarkroom_get_focal( $attachment_id, '16:9' );
// Returns the override if set, falls back to the default focal,
// falls back to {0.5, 0.5}.
```

The cover/contain logic in CSS uses the focal coordinates as `object-position` values.

---

## 5. License validation flow

License state lives in three places:

```
Remote license server (Lemon Squeezy)   ← source of truth
        │
        │  HTTPS, signed responses
        ▼
inc/class-studiodarkroom-license-client.php
        │
        │  caches result in wp_options
        ▼
inc/class-studiodarkroom-license-manager.php
        │
        │  exposes state to feature gates
        ▼
Plugin features   (read state, gate behavior)
```

### State machine

```
unactivated  →  activate(key)   → active
active       →  revalidate()   → active | grace | expired
grace        →  revalidate()   → active | expired
expired      →  activate(key)  → active | invalid
invalid      → (terminal until user pastes new key)
```

### Grace period

When the remote server is unreachable (network failure, server downtime), the manager enters a **7-day grace period** before downgrading to `expired`. This keeps customer sites working through transient outages. The plugin only locks if the server has been unreachable for a full week — well past any normal operational hiccup.

### Local development bypass

`wp_get_environment_type()` returning `local`, `development`, or `staging` triggers a development bypass — administrators can run unlicensed for local work. Production stays strict.

---

## 6. Lite ↔ Pro safety contract

The plugin ships in two editions:

- **Lite** — free, limited feature surface (folder basics, smart folders, focal points)
- **Pro** — licensed, full feature surface (gallery, bulk ops, diagnostics, advanced smart folders)

The hard part isn't gating features — it's letting customers downgrade safely.

The **swap contract** guarantees:

- Downgrading Pro → Lite preserves all folder structure and attachment metadata
- Pro-only data (gallery presets, advanced workflow tags) is **frozen but not destroyed**
- Re-upgrading restores everything exactly as it was

This is what makes Lite safe to install on production. Customers can experiment with Lite, upgrade to Pro for evaluation, drop back to Lite if they don't want to renew, and **never lose their data**. That trust is what justifies the price tier.

---

## 7. Security model

### Capability gating

Six plugin-specific capabilities, mapped to WordPress roles:

```
studiodarkroom_view_library
studiodarkroom_edit_attachments
studiodarkroom_create_folders
studiodarkroom_delete_folders
studiodarkroom_run_bulk_ops
studiodarkroom_view_diagnostics
```

Administrators always have all six regardless of role mapping. Other roles get capabilities granted by the admin in **Settings → Roles**.

### Nonce + origin checks

Every AJAX handler verifies a plugin-specific nonce. REST endpoints use the standard WP nonce. Origin checks reject cross-site requests.

### Input sanitization

- `sanitize_text_field` for plain strings
- `sanitize_key` for option keys + slugs
- `esc_url_raw` for URLs on save, `esc_url` on render
- `wp_kses_post` for rich text fields
- Folder names: stripped to printable Unicode, length-capped at 200 chars

### SVG hardening

SVG uploads disabled by default. When enabled per-role, every uploaded SVG passes through `enshrined/svg-sanitize` before reaching the uploads directory. Inline scripts, external references, and event handlers are stripped.

### MIME validation

`finfo` checks every upload's actual MIME type against the file extension, rejecting mismatches. Belt + suspenders against `image.jpg.php` style attacks.

---

## 8. Crash guard + diagnostics

WordPress's default error handler renders a "There has been a critical error" page — useless for debugging. Worse, fatal errors during early hooks happen before WordPress even initializes its handler, so visitors see a raw PHP stack trace.

The plugin registers its own fatal-error handler that:

1. Catches early-hook fatals before WP's screen renders
2. Captures a scrubbed environment snapshot (no PII)
3. Forwards to Sentry (production only, opt-in)
4. Renders a graceful admin notice instead of a white screen of death

The Diagnostics page surfaces:

- WordPress + PHP + plugin versions
- Active theme + active plugins
- License status + tier + expiration
- Recent failures (last 50, scrubbed of PII)
- Role-preview matrix (capability check across all roles)
- A **"Copy diagnostics report"** button that produces a sharable Markdown snapshot for support tickets

---

## 9. Performance considerations

### Folder query hot path

Listing folders + counting attachments per folder is the most-called query in the admin UI. Optimizations:

- Folder counts cached in a transient with a 5-minute TTL
- Cache invalidated on folder create/delete/move and attachment add/remove
- Bulk-rebuild path skips invalidation per-attachment and bumps the cache version once at the end

### Smart folder queries

Smart folders that hit `meta_query` (Missing Alt Text, Rating, Workflow Tag) are slow on large libraries. Mitigation:

- Each smart folder caches its result count for 60 seconds
- Listing the folder uses a paginated query, not a full result set
- A future optimization will move the most-expensive smart folders to a denormalized index table

### Asset loading

Admin assets are only enqueued on plugin pages, not site-wide. The check uses a screen-detection helper so the plugin doesn't slow down post admin or the block editor.

---

## 10. I18n strategy

WordPress's gettext system is the substrate, but the plugin layers a **token dictionary** on top:

- Token map: `'studio.media.alt_missing' => 'Missing alt text'`
- A custom `gettext` filter hook intercepts every `__()` call and consults the dictionary first
- Falls back to standard `.mo` lookup if the token isn't in the dictionary
- The dictionary is keyed per-language; admin UI language can be set independently of the site locale

This lets the plugin ship admin UI translations (en/es/fr) without requiring users to override the site locale globally — useful for agency sites where the staff speaks Spanish but the public-facing site is in English.

---

## 11. Module system

Optional features are registered as **modules** under `app/Modules/`. The module pattern:

```
class GalleryModule {
    public static function isEnabled(): bool;     // Lite/Pro/license check
    public static function bootstrap(): void;     // Wire hooks
    public static function dependencies(): array; // Other modules required
}
```

Module discovery walks the modules directory at boot, calls `isEnabled()` on each, and bootstraps enabled ones in dependency order. Disabled modules consume zero runtime — their files aren't even loaded.

This is the extensibility surface for Pro-tier add-ons (or future Studio Grid sister plugins) that want to plug into StudioDarkroom without modifying core.

---

## 12. Testing

`/tests` contains PHPUnit suites split into `unit/` (pure functions, no WordPress) and `integration/` (full WordPress test environment). The CI pipeline runs both suites against PHP 8.1, 8.2, and 8.3 on every push.

Coverage is currently strongest on:

- Folder model
- License state machine
- Smart-folder query builder
- Capability checks

The admin UI layer is covered by manual QA in the release pass — a Playwright suite is planned.

---

## Design principles

Five constraints the architecture is optimized for:

1. **Performance-first** — caching, paginated queries, screen-scoped asset loading. Admin UI must stay responsive on libraries with 50,000+ attachments.
2. **Backward compatibility** — legacy procedural code continues to work; migration is incremental, not big-bang.
3. **Modularity** — features are isolated modules with clear dependency declarations. Disabled modules don't load.
4. **UI isolation** — all CSS scoped under plugin-specific classes; all JS scoped to plugin pages. The plugin doesn't bleed styles or behavior into post admin or the block editor.
5. **Trust through the swap contract** — Lite ↔ Pro transitions never lose user data, even when downgrading or letting a license expire.
