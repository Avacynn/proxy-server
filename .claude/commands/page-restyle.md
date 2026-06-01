---
description: Restyle a page to the 2026 style guide aesthetic. Preserves copy and information architecture — translates existing components into style-guide-compliant equivalents.
argument-hint: <page path or name, e.g. server/pages/affiliate.html or learning-modules>
---

# Style-guide restyle — $ARGUMENTS

You are applying the **2026 style guide aesthetic** to the page at `$ARGUMENTS`. This is a **restyle, not a rewrite**. Copy and information architecture stay; visual language gets brought into line with the style guide.

## What changes vs. what stays

**Stays (preserve verbatim where reasonable):**
- All headings, sub-headings, body copy, list items, button labels, link text, microcopy.
- Section order and section count.
- Every link, form action, route reference, JSON-LD block, and `data-*` attribute.
- Real product info: SKUs, prices, legal/disclaimer text, citations.
- The page's purpose and what info it communicates.

**Changes (translate to the style guide):**
- Visual tokens — replace ad-hoc colors with `var(--primary-color)`, `var(--secondary-color)`, `var(--tri-color)`, the grey ink ramp from §1.
- Typography — Lora (serif) for headings, Nunito (sans) for body/labels/buttons. Strip any third typeface.
- Containers — old card patterns (especially the deprecated 6px brand-border card) become the §3 flat-card recipe.
- Buttons — anything button-shaped becomes a pill (§5): primary filled or ghost outlined.
- Section headers — adopt the eyebrow → Lora `<h2>` → optional "see all" pattern from §2.
- Section padding — `clamp(2.5rem, 5vw, 4rem) var(--container-padding-x);` is the floor.
- Hero — if the page has a top intro section, reshape it into the §6 hero pattern (full-bleed soft tri-stop gradient, Lora `<h1>` with one accent word, primary + ghost CTAs, optional right-column artifact).
- Final CTA — if the page ends with a call to action, give it the §7 dark gradient band treatment with inverted buttons.
- Motion — strip bouncy easings / parallax / arbitrary animations. Keep only the §8 trio (card hover lift, pulse dot, EKG sweep — the last reserved for clinical illustrative elements).

**Light copy edits are allowed only where required by the new shape:**
- A hero may need one extracted "accent word" to italicize in the `<h1>`.
- An eyebrow label may need to be invented for a section that had none.
- A button labeled "Click here" can be retitled to something descriptive.

Anything beyond that crosses into rewrite territory — use `/page-overhaul` instead, not this command.

## Mandatory reading order (parallel where possible)

1. **The style guide**: `.github/spec-style-guide.md`. Read it end-to-end. The tokens (§1), card recipe (§3), buttons (§5), hero (§6), final CTA (§7), and do/don't list (§9) are the contract.
2. **The canonical reference**: `server/pages/index.html` and its associated CSS in `client/public/styles.css`. This is the live embodiment of the style guide — your patterns should look like patterns from this page.
3. **The target page** ($ARGUMENTS). If the argument is a bare name (e.g. `affiliate`), resolve it to `server/pages/$ARGUMENTS.html`. Read the full file. Also read any page-specific CSS in `client/public/` (e.g. `affiliate.css`, `learning-modules.css`).
4. **Shared chrome**: `server/templates/header.html`, `server/templates/footer.html`, `server/templates/base.css`. Don't duplicate or redefine.

## How to plan the restyle

Before editing, internally produce a **mapping table**: current section → style guide pattern it should adopt.

| Current section | Current shape | Style-guide target |
| --- | --- | --- |
| Top intro band | Centered text + button | §6 hero pattern, full-bleed gradient |
| "Features" grid | 4 boxed cards with brand border | §3 flat-card grid, one `.is-featured` tile |
| Testimonial strip | Loose paragraphs | Pill-tagged quote cards or pull-quote section |
| Bottom CTA | Plain centered button | §7 full-bleed dark gradient band |
| ... | ... | ... |

Then work section-by-section in render order. Don't try to ship the whole file in one Edit — multiple smaller edits keep diffs reviewable.

## Style guide compliance — non-negotiable

- **Tokens only** for brand color. The grey ink ramp from §1 is the only allowed off-token greys.
- **Fonts** as wired in `base.css` — don't import a new family.
- **Card recipe verbatim** from §3 — `#fff` background, `1px solid #e3ecf3`, `var(--radius-lg, 12px)`, hover `translateY(-2px)` + `0 12px 28px rgba(3, 98, 156, 0.12)` shadow + `border-color: var(--primary-light)`. At most one `.is-featured` dark tile per grid.
- **Pill buttons only** (§5). No square buttons, no `border-radius: 4px;` legacy buttons.
- **Eyebrows**: `var(--secondary-color)`, uppercase, `letter-spacing: 0.12em`, `font-size: 0.75rem`, Nunito 800.
- **Heading widths**: `<h1>` `max-width: 18ch; text-wrap: balance;`.
- **Forbidden** (§9): nested cards, old 6px brand-border cards, large brand-color fills, bouncy easings, parallax, pulse-dot animations on non-clinical UI, third typefaces, new accent colors.

## Where the CSS goes

- If the page has a dedicated stylesheet in [client/public/](../../client/public/) (e.g. `affiliate.css`, `careplans.css`), edit it in place — strip the old patterns, add the style-guide-aligned rules.
- If page-specific styles live inline in a `<style>` block, edit them there and keep them scoped to a unique page class.
- Don't touch [server/templates/base.css](../../server/templates/base.css) tokens — they're shared and authoritative.
- Reuse `.nc-card` from `base.css` where the page just needs a flat container; use the elevated §3 `.card` recipe for grid tiles that match the homepage aesthetic.

## Deliverable

1. The restyled HTML (same path, same copy, new shape).
2. The restyled CSS (same path where one exists, otherwise scoped inline).
3. A short final message (≤6 bullets): mapping table summary (what each section became), any places where copy had to flex (e.g. an invented eyebrow), any links/integrations you confirmed are intact.

## Verification

- Diff-read: confirm copy is preserved word-for-word except where explicitly noted in the deliverable summary.
- `grep` the page's path across the repo — confirm no header/footer/sitemap link breaks.
- Open every link, every form action, every script src — confirm none were dropped during the restyle.
- Walk through the rendered file mentally: does the hero match §6? Do cards match §3? Does the final CTA match §7? If a section can't be mapped to a style guide pattern, justify it in the summary rather than inventing a new pattern.
- Validate tag balance.

Begin.
