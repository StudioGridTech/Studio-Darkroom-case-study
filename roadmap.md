# Roadmap

A high-level look at where Studio Darkroom is and where it's headed.

> Specific feature breakdowns and release timing for upcoming versions live in the private repo. This page covers the trajectory.

---

## Now — v1.0.1

The polish pass before public release. Focus areas:

- Security hardening across the AJAX and REST surfaces
- Lemon Squeezy licensing and payment integration
- Crash-guard and diagnostics expansion
- Performance work on the folder-listing hot path
- Internal QA across PHP 8.1 / 8.2 / 8.3 and WordPress 6.4 through 6.7
- Documentation, onboarding flow, and the public release checklist

This is the version that ships first. Everything below depends on it landing solid.

---

## Soon — v2

The next major version moves past "where do my files live" into **what happens to them once they're there**. The plugin already handles structure. v2 adds the layer that turns structure into a real workflow.

Themes the v2 release will cover:

- **Upload** — smarter handling of metadata that's already inside the files visitors send to the library
- **Organize** — curation primitives borrowed from professional photo workflows
- **Enrich** — AI-assisted accessibility and discoverability tooling
- **Protect** — visible companion to the existing AI-Protection layer

Each theme is its own discrete piece of work. None of them depend on each other; all of them sharpen the v1 foundation.

Specific feature names, ordering, and implementation details aren't being shared publicly until each piece ships. The shape will become visible at release.

---

## Later — beyond v2

Possibilities being scoped for future major versions, in rough order of interest:

- **Pipeline state management** — workflow stages (working / ready / published / archive) as a separate dimension from quality curation
- **Editing surface** — basic image editing (crop, exposure, color, denoise) without leaving WordPress
- **Hosted AI services** — managed credit packs as an alternative to bring-your-own-API-key
- **Studio Grid integrations** — deeper hooks between Studio Darkroom and sister plugins (Studio Player, Studio Media, StudioDashboard)
- **Statistics surface** — usage signals, top folders, attribution patterns
- **Pre-built templates** — drop-in configurations for common site types (photographer portfolio, agency case studies, podcast show pages)

These are the conversations happening internally. None are committed; some won't ship; one or two might land sooner than others depending on how customers actually use v1 and v2.

---

## Won't ship — explicit non-goals

Worth saying out loud so the conversation doesn't keep coming up:

- **Hosted SaaS** — the plugin runs on the customer's WordPress site. There's no central server, no cloud library, no multi-site analytics service. Not now, not later.
- **Mobile app** — out of scope. The admin UI is responsive enough for tablet review; phone-first media management isn't the audience.
- **Frontend page builder integration as a primary surface** — Studio Darkroom is a backend plugin. Page builder companion blocks/elements may exist; the core experience stays in `wp-admin`.

---

## Long-term commitment

Studio Darkroom is a multi-year project. The plan is to keep shipping incrementally — v1 establishes the structure, v2 adds the workflow, future versions extend the surface as customer needs become legible.

The Studio Grid family is built on the same long-arc thinking: each plugin is its own focused tool, designed to ship for years rather than as a one-time release.

For licensing inquiries, evaluation copies, or technical conversations: [studiodarkroom.com](https://studiodarkroom.com).
