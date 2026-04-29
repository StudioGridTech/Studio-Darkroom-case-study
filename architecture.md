# Architecture

A high-level look at how Studio Darkroom is designed — the patterns that shape the codebase, the engineering decisions worth talking about, and the principles the project optimizes for.

> Source code is private. This page covers the *thinking*, not the *recipe* — implementation specifics live in the private repo.

---

## A modular, layered system

Studio Darkroom is a backend-first WordPress plugin. The codebase is structured so each system can evolve independently — folders, smart filtering, focal points, the gallery module, the licensing layer, and the diagnostics surface all live as separate concerns with clear boundaries.

The plugin runs on a **dual-layer pattern**: a modern, namespaced layer where new features land, and a stable legacy layer where the original code continues to operate. Migration between the two is gradual and tied to feature work — never a big-bang rewrite, never an endless deprecation drag. Each cluster moves when the cost of touching it without modernization exceeds the cost of modernizing it.

This is the kind of decision that's easy to get wrong. Two failure modes are common in codebases at this size:

- **The big-bang rewrite** — fork the codebase, rebuild from scratch, ship months later (or never)
- **The endless deprecation** — drag legacy code along forever, accumulating debt, never finishing the migration

The chosen path avoids both. Legacy code keeps working until it has a reason to change; when it changes, it modernizes. The tech debt curve trends down naturally.

---

## Folders that scale

The folder system is built around a simple insight: **WordPress already has the right primitive — taxonomies — for hierarchical organization with shared metadata.** The plugin uses that primitive instead of inventing custom database tables.

The benefit isn't just that there's less code to write. It's that every other plugin that respects WordPress taxonomies — search, REST API, caching layers, export tools — sees folders for free. Migrating a site between hosts is a JSON export of standard term data. There's nothing exotic to back up. There's nothing exotic to break.

Smart folders take a different approach. Rather than physically grouping attachments, they're virtual views composed at query time. The "Missing Alt Text" view, the "Duplicates" view, the "Recently Uploaded" view — none of these are stored. They're computed. As the library changes, the views update automatically without anyone running a sync.

The smart-folder system is itself extensible — future Pro-tier features and sister plugins can register their own without modifying core.

---

## A safety contract between Lite and Pro

The plugin ships in two editions: a free Lite tier with the core organization features, and a licensed Pro tier with the advanced surface — gallery output, bulk operations, advanced smart folders, diagnostics.

The hard part isn't gating features. It's letting customers move between editions without losing work.

The **swap contract** guarantees three things:

- Downgrading Pro → Lite preserves all folder structure and attachment metadata
- Pro-only data isn't deleted; it's frozen, waiting
- Re-upgrading restores everything exactly as it was

This is what makes Lite safe to install on production sites. Customers can experiment with Lite, upgrade to Pro for evaluation, drop back to Lite if they don't want to renew, and never lose their data. That trust is what justifies the price tier.

---

## A security mindset

The plugin handles file uploads, taxonomies, attachment metadata, and AJAX endpoints — every category of attack surface a WordPress plugin can have. The security model is layered:

- **Capability-based access control.** Plugin features are gated by plugin-specific WordPress capabilities, mappable per-role by site administrators. Features check capabilities at the entry point of every action, not at the UI layer.
- **Nonce verification on every state-changing request.** AJAX, REST, and form submissions all verify a plugin-scoped nonce. Cross-site requests are rejected at the boundary.
- **Sanitization on input, escaping on output.** Standard WordPress functions, applied consistently. No clever shortcuts.
- **SVG hardening.** SVG uploads are disabled by default. When enabled per-role, every uploaded SVG passes through a hardened sanitizer that strips inline scripts, external references, and event handlers before the file reaches the uploads directory.
- **MIME validation against extension.** Every upload is checked against its actual file content, not just its extension. Belt and suspenders.

These aren't novel security techniques. They're the basics done consistently. Most plugin vulnerabilities aren't sophisticated — they're a missing nonce, an un-sanitized input, an over-trusted capability. The architecture's job is making "the right thing" the default path so individual handlers don't have to remember.

---

## A graceful failure surface

WordPress's default error handler renders "There has been a critical error" — a single sentence that gives the user nothing to act on. Worse, fatal errors during early hooks happen before WordPress even initializes its handler, leaving visitors looking at raw PHP output.

The plugin registers its own fatal-error handler that catches early-hook fatals before they leak, captures a scrubbed environment snapshot, optionally forwards to a configured error-tracking service, and renders a graceful admin notice. The Diagnostics page surfaces the same data plus role-preview matrices, environment checks, license status, and a copy-report button that produces a sharable Markdown snapshot for support tickets.

The design goal is simple: when something breaks, the customer should be able to send a useful report without installing a debugger or opening DevTools.

---

## Performance, where it actually matters

Most WordPress performance work is premature optimization. The architecture treats performance as a discipline applied to **specific hot paths**, not a blanket rule.

Three hot paths get explicit attention:

- **Folder listing + per-folder counts** — the most-called query in the admin UI. Cached aggressively, invalidated precisely on the events that would make the cache wrong.
- **Smart folder queries** — particularly the meta-query-based ones, which are inherently slow on large libraries. Counts are cached short-term; result sets are paginated rather than fully materialized.
- **Admin asset loading** — plugin CSS and JS are scoped to plugin pages only. The post editor and block editor don't pay for assets they don't need.

Other code paths take the straightforward approach. Optimization happens in response to measurement, not preemptively.

---

## Extensibility through modules

Optional features are registered as modules with clear lifecycle hooks — enabled-check, bootstrap, dependency declaration. Modules that aren't enabled don't load; their code doesn't even reach the autoloader.

This is the extensibility surface for Pro-tier add-ons and future sister plugins. New features can plug in without modifying core. Disabled features cost zero runtime.

---

## Internationalization that respects context

Standard WordPress translations work via gettext. That's fine for site-facing strings, but the plugin needs something more flexible: an admin UI language **independent of the site locale.** Agency sites where the staff speaks one language and the public site is published in another are common; tying admin language to site locale forces a compromise.

The plugin layers a custom translation system on top of standard gettext that lets administrators select an admin UI language separately from the site locale. The plugin currently ships English, Spanish, and French.

---

## Design principles

Five constraints the architecture is consistently optimized for:

1. **Performance-first** — caching, paginated queries, screen-scoped asset loading. The admin UI is built to stay responsive at the scale the plugin has been validated against (currently libraries in the low thousands of attachments) and to keep that responsiveness as testing pushes the bar higher.
2. **Backward compatibility** — legacy code continues to work; migration is gradual, never big-bang.
3. **Modularity** — features are isolated with clear boundaries. Disabled modules don't load.
4. **UI isolation** — all CSS scoped under plugin-specific classes; all JS scoped to plugin pages. The plugin doesn't bleed styles or behavior into post admin or the block editor.
5. **Trust through the swap contract** — Lite ↔ Pro transitions never lose user data, even when downgrading or letting a license expire.

---

## What this document doesn't cover

Implementation specifics — file paths, class signatures, capability names, cache keys, exact timing parameters, internal data structures — are intentionally omitted. They live in the private repo.

For licensing inquiries, evaluation copies, or technical conversations with prospective collaborators: [studiodarkroom.com](https://studiodarkroom.com).
