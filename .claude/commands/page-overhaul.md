---
description: Full content+visual overhaul of a page. Free liberties to rename, reword, restructure; style strictly per the project style guide.
argument-hint: <page path or name, e.g. server/pages/index.html or affiliate-new>
---

# Full page overhaul — $ARGUMENTS

You are doing a **complete overhaul redesign** of the page identified by `$ARGUMENTS`. Preserve only the high-level intent and information of the page — everything else is on the table.

## What "high-level info" means

- The page's **purpose** (what user goal it serves) and the **must-have facts** (product names, prices, legal disclaimers, route paths, form fields with real backend wiring, schema-required SEO metadata).
- The page's **inbound/outbound links** and any **route handlers / form actions / scripts** that the server or other pages depend on.

**You may freely change:** section order, section count, headings, sub-headings, copy/voice, component composition (cards vs. lists vs. split-grids), iconography choice, image placement, CTA wording, microcopy, eyebrow labels, taglines, what's above vs. below the fold, even what gets cut entirely if it's not pulling its weight.

You may **not** change: real product SKUs/prices, real backend route names, form `name=` attributes that submit to live endpoints, JSON-LD structured data field meanings, or anything that would break a live integration. When unsure whether a thing is structural, grep for it before touching it.

## Mandatory reading order (do this first, in parallel where possible)

1. **The style guide — non-negotiable**: `.github/spec-style-guide.md`. This defines tokens, card recipe, hero pattern, CTA pattern, buttons, motion. Treat every rule in §3, §5, §6, §7, §9 as binding.
2. **The canonical reference page**: `server/pages/index.html`. This is the live embodiment of the style guide. Mirror its patterns — section bands, full-bleed gradient hero, flat 1px-bordered cards with hover lift, one dark `.is-featured` tile per grid, dark final CTA band.
3. **The target page itself** ($ARGUMENTS). If the argument is a bare name (e.g. `affiliate-new`), resolve it to `server/pages/$ARGUMENTS.html`. Read the full file end-to-end.
4. **Shared chrome**: `server/templates/header.html`, `server/templates/footer.html`, `server/templates/base.css`. The overhaul lives between these — don't duplicate header/footer markup, don't redefine the design tokens.
5. **The route handler** (skim only): `server/routes/pages.js`. Confirm what the route expects this page to render and what data it passes in. Don't break the contract.
6. **CLAUDE.md** for project context — this is a nursing education platform. Voice should be clean, clinical, modern-editorial. Not corporate-cheesy, not gamified, not bro-y. Trust signals matter (citations, real licensure, free where it says free).

## How to plan the overhaul

Before writing any HTML/CSS, internally produce:

1. **An information audit** of the current page — list every distinct piece of info the page communicates, grouped by importance. Decide what survives, what gets reworded, what gets cut.
2. **A new section outline** — section-by-section, in render order, with: section purpose, eyebrow label, `<h2>` heading, body copy approach, primary component pattern (e.g. "3-card grid with one `.is-featured` tile", "split hero", "pull-quote band", "FAQ accordion"), CTA if any.
3. **The hero** — every overhauled page gets a proper hero in the style guide §6 shape (full-bleed soft gradient band, eyebrow → Lora `<h1>` with one accent word → lede → primary + ghost CTAs → trust strip → right-column artifact). The right-column artifact should be *relevant to this page's topic*, not a copy of the EKG monitor unless the page is about EKG.
4. **The final CTA** — every page ends with the §7 dark gradient band. Wording must be specific to this page's goal, not generic.

## Style guide compliance — non-negotiable rules

- **Tokens only**: `var(--primary-color)`, `var(--secondary-color)`, `var(--tri-color)`, etc. No new hex values for brand colors. The grey ink ramp from §1 is allowed.
- **Fonts**: Lora (serif) for headings, Nunito (sans) for body/labels/buttons. No third typeface. Italic only on the hero `<h1>` accent word.
- **Cards**: the recipe in §3 verbatim — flat `#fff`, `1px solid #e3ecf3`, `border-radius: var(--radius-lg, 12px)`, `translateY(-2px)` + shadow on hover, `border-color: var(--primary-light)` on hover. At most **one** `.is-featured` dark tile per grid.
- **Buttons**: pill shape only (§5). Primary = filled blue with shadow. Ghost = transparent with tinted border. Inverted on dark bands.
- **Eyebrows**: `var(--secondary-color)`, uppercase, `letter-spacing: 0.12em`, `font-size: 0.75rem`, Nunito 800. One per section header.
- **Section padding floor**: `padding: clamp(2.5rem, 5vw, 4rem) var(--container-padding-x);`
- **Section heading widths**: `<h1>` capped at `max-width: 18ch; text-wrap: balance;`. `<h2>` similarly tight.
- **Forbidden** (§9): old 6px brand-border cards, nested cards, large brand-color fills outside `.is-featured`/CTA band, bouncy motion, parallax, EKG-sweep/pulse animations on non-clinical UI.

## Where the CSS lives

- Page-specific CSS goes in `client/public/styles.css` (the global stylesheet served on every page) **only if** the patterns are reusable. Page-specific one-offs go in a `<style>` block at the top of the HTML file, scoped under a unique page class (e.g. `body.affiliate-new-page .hero { ... }`).
- Do **not** redefine tokens that already exist in `server/templates/base.css` or `client/public/styles.css`. Reuse.
- Reuse `.nc-card` from `base.css` where a flat container is enough; use the elevated `.card` recipe from §3 for the homepage-grade tiles.

## Deliverable

1. The fully rewritten HTML file at its original path (overwrite in place — this is a git repo, the diff is the record).
2. Any new CSS, scoped correctly per the rule above.
3. A short final message (≤6 bullets) summarizing: sections kept / sections cut / sections added / new copy direction / any integration points you confirmed are intact.

## Verification before you finish

- `grep` for the page's route in `server/routes/pages.js` and confirm the response shape still works.
- `grep` for the page's path across the repo (header links, footer links, sitemap, internal links) and confirm no link is broken by a rename. If the page slug changes, that's a bigger refactor and you should stop and ask the user first.
- Visually trace through the rendered file: does it match the §6 hero pattern, the §3 card recipe, the §7 final CTA pattern? If a section doesn't fit one of the established patterns, justify it in the summary.
- Type-check / lint nothing — these are HTML/CSS files. But validate that opening/closing tags balance.

## Tone & voice for any new copy you write

- Clinical confidence without coldness. Authoritative, not salesy.
- Short sentences. Real verbs.
- No emoji in copy. No exclamation marks except in genuine alerts.
- When in doubt, ask: would a charge nurse find this credible? Would a nursing student find it intimidating? Aim for the first, avoid the second.

Begin.
