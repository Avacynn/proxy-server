---
description: Rework a page's AdSense placements end-to-end — strip the old ad markup, then place nc-ad markers for maximum fill, pushed as aggressively as Google policy + the style guide allow (content-to-ad ratio, no deceptive/in-feed placement, mobile density ceiling).
argument-hint: <page path or name, e.g. server/pages/lab-values.html or vital-signs-reference>
---

# AdSense placement pass — $ARGUMENTS

This is the **third step** after `/page-overhaul` (content+visual rewrite) and `/page-restyle` (style translation). Those two leave the page in its final editorial shape. Your job here is the **ads, end to end**: completely rework the page's AdSense placements so they **monetize as aggressively as policy allows** — correct, intelligently positioned, the right slot for each spot, and filling *every* placement the page can legally carry — and, where a content section is wasting desktop horizontal space, **restructure that one section into a content + gutter split** so a rail ad has somewhere to live.

**The mandate is maximum fill, not minimum.** Default to *more* ads, not fewer — but every added slot lives under a hard compliance ceiling (see *The compliance ceiling* below). Aggression that crosses that ceiling isn't aggressive, it's a policy strike on the **owner's** account, which zeroes all revenue — so the ceiling is the one thing you never trade for another impression.

Assume the page's existing ad placements may be **wrong** — wrong alias, wrong position, hardcoded `<ins>`, a stray page-local loader, stacked/crowded, or sitting somewhere the style guide forbids. **Rip them all out and rebuild**, don't patch around them.

A page with a single centered content column inside a much wider container has **dead horizontal gutter** on desktop (e.g. a ~760px reading column inside a 1200px container). You are allowed — encouraged, conservatively — to claim that gutter for an ad by converting a content-heavy section into a two-column grid. See *Building an ad gutter* below. This is the **one** structural change this command may make; everything else about the page's shape stays exactly as the overhaul/restyle left it.

## What you change vs. what you leave alone

**You change:**
- Which `<!-- nc-ad: <alias> -->` markers the page has, and where they sit.
- Removal of any hardcoded `<ins class="adsbygoogle">`, any page-local `adsbygoogle.js` loader, and any orphaned ad markup left by previous edits.
- **The grid structure of a content section that has wasted desktop gutter** — wrapping its existing content in a content column and adding a sibling gutter column to host a rail ad (see *Building an ad gutter*). This is layout-only and content-preserving: you re-home the section's existing markup verbatim, you do not rewrite it.

**You leave alone (do NOT touch):**
- Editorial copy, headings, section order, section count, link/form/route references, JSON-LD, `data-*` — the overhaul/restyle settled those, and a gutter split must preserve every one of them verbatim. The only thing that may move is *which wrapper a section's content sits inside*.
- Components, cards, and the visual shape of everything that isn't getting a gutter. Don't restyle, don't re-token, don't touch a section that has no dead space.
- `.nc-ad` container CSS, the collapse rules, the loader in `head.html`, and `adSlots.js` rendering logic. These are centralized and correct. You only add a new alias to `SLOTS` if a placement genuinely needs one that isn't mapped (rare — 13 are mapped already; see inventory). (Note: the *grid CSS* for a gutter split is page structural CSS and is yours to add — see *Marker + wrapper mechanics*. It is not ad-container CSS.)
- The render pipeline (`generateBaseHTML.js`, suppression set) — read it for context, don't edit it unless the user asks.

## Mandatory reading order (parallel where possible)

1. **The AdSense spec** — `.github/spec-adsense.md`. This is the contract: the render path (markers → `adSlots.js` → `<ins>`), the **13 mapped aliases** and what each is for, the wrapper convention, suppression, the dev story, and the pre-merge checklist. Treat the slot inventory and best-practices list as binding.
2. **The style guide's ad section** — `.github/spec-style-guide.md` §12 "Advertising / ad placement". This is *where ads may and may not sit*, the `.nc-ad` recipe, density/breathing-room rules, and collapse behavior. Binding.
3. **The render source** — `server/rendering/adSlots.js`. Confirm the live `SLOTS` map (alias → slot ID, format, layout) and that the alias you plan to use exists. Markers only expand through the Express server path.
4. **The target page** ($ARGUMENTS). If the argument is a bare name (e.g. `vital-signs-reference`), resolve it to `server/pages/$ARGUMENTS.html`. Read it end-to-end so you know its section rhythm, length, whether it has a sidebar rail, where the hero ends, and where the final CTA band begins.
5. **A good reference page** — skim `server/pages/lab-values.html` and `server/pages/vital-signs-reference.html` for how markers sit inside the page's own wrapper divs (`.ads-column`, `.horizontal-ad`, `.horizontal-ad-container`, in-flow between `</section>` and the next block).

## The 13 mapped aliases — pick the right one for each spot

From `SLOTS` in `adSlots.js` (the spec is canonical). Choose by **placement intent**, not by page identity:

| Need | Use | Why |
| --- | --- | --- |
| Between two body sections, long-form content | `in-article` / `in-article-2` | fluid in-article format, reads as editorial-adjacent |
| Horizontal banner break in the flow | `horizontal` / `horizontal-2` | responsive display |
| Related-content block at the **end** of content (before final CTA) | `multiplex` / `multiplex-vertical` | autorelaxed grid, good as a content endcap |
| Desktop sidebar rail alongside scrollable content | `sidebar-right` / `sidebar-left` (`-2` variants for a second) | 320px-capped, sits in the column |
| Square unit in a narrow column / between cards | `square` / `square-2` | responsive square |
| Tall unit in a narrow vertical space | `vertical` | responsive vertical |

**Use distinct aliases for distinct placements.** If a page needs two horizontal banners, use `horizontal` and `horizontal-2`, not `horizontal` twice — each alias is one dashboard unit, and reusing one twice on a page muddies reporting and risks two identical slots competing. The `-2` variants exist exactly for this.

The two intentionally-unmapped dashboard orphans (`Horizontal1` `2023257941`, `square2` `3120717197`) stay out of `SLOTS` — don't reach for them.

## How to plan the placement pass

Before editing, internally produce a **placement table**:

| Spot in page (anchor) | Old ad there? | Style-guide verdict | Chosen alias | Rationale |
| --- | --- | --- | --- | --- |
| After hero, before §1 | (e.g. hardcoded `<ins>`) | **Forbidden** — remove | — | no ad between hero and first section |
| Between §2 and §3 (long body) | none | OK in-flow | `in-article` | fluid, mid-article |
| Desktop right rail | wrong alias `vertical` | OK in rail | `sidebar-right` | 320px-capped rail unit |
| End of content, before final CTA | none | OK endcap | `multiplex` | related-content block |
| ... | ... | ... | ... | ... |

Decide **count by page length** — push *to* the ceiling, not below it. These are floors to hit, not caps to stay under; a page only carries fewer when it's too thin to keep content dominant (see *The compliance ceiling*):
- **Short page (1–2 screens):** 1 manual slot — a `horizontal` immediately after the first content section — plus Auto Ads. No gutter (too short for a rail to earn its keep).
- **Medium page (3–5 sections):** 2–3 manual slots — e.g. an early `horizontal`, one `in-article` mid-flow, and a `multiplex` endcap above the final CTA. Add a single desktop gutter rail if a section has real dead space.
- **Long reference/guide page (6+ sections, multi-screen):** 4–6 placements — **both** `in-article` slots spread through the body, a `horizontal` / `horizontal-2` break, a `multiplex` (or `multiplex-vertical`) endcap, **plus 1–2 desktop gutter rails** on sections with dead horizontal space (see below). Use the `-2` alias variants so no single dashboard unit repeats on the page. Gutter rails count toward the total; the only spacing rule is a full content section between any two *in-flow* slots — rails in the gutter run parallel to content and don't count against that.

While reading the page, also flag **gutter candidates**: any full-width content section that is text-heavy or bullet-heavy, sits in a column much narrower than its container, and is tall enough that a sticky rail beside it stays in view as the reader scrolls. These are where a content + gutter split is worth it. A short section, a grid of cards that already fills the width, or anything above the fold near the hero is *not* a candidate.

**Place the first slot as high as the rules allow.** The hero rule means the earliest legal ad is *immediately after the first content section* — so put it there, don't let the first manual slot drift to mid- or late-page. Readers who never scroll to the bottom should still see one ad. Two things make this matter:
- On a page whose first content section *is* a card grid or a single dominant block, "after the first section" can still be fairly low — that's the floor the hero rule imposes, not a reason to push it lower. Use a `horizontal` break right below that first section, and if the page has a tall prose section, a **gutter rail rides highest of all** (it sits beside content from the top of that section and stays in view), so prefer a gutter over a late in-flow slot when "too low" is the worry.
- The genuinely-high placements (top-of-content, anchor) are **Auto Ads'** job, placed account-side — not something manual markers can or should do. If a page feels starved of ads up top, the lever is Auto Ads (and the known above-hero collapse issue in the spec), not pushing a manual `<ins>` into the hero.

## The compliance ceiling (what bounds "aggressive")

These are **Google's actual program/placement policies** — contract terms enforced by disabling ad serving or the **owner's account** (not law, but the account is the owner's, so a strike is the worst outcome on this site). They are the hard stop that "push more ads" must never cross. Within them, push hard; at them, stop.

- **There is no numeric cap — content-to-ad ratio is the real limit.** Google removed the old "3 ad units per page" rule years ago, so don't ration by count. The governing rule is that **publisher content must clearly dominate the page**: ads may not equal or exceed content, and a page with little/thin content may carry few or no ads. This is why density scales with *page length* below — a long guide can carry many ads; a thin page cannot, no matter how much you'd like to.
- **Auto Ads are already on the page — count them.** Every non-suppressed route runs account-side Auto Ads *on top of* your manual markers. Your manual slots are *additive*. When judging density, picture the page with Auto Ads filled too — manual + auto together must still leave content dominant.
- **Mobile viewport density ≤ ~30%.** Per the Better Ads Standards, ads must not occupy more than ~30% of a mobile viewport's height, and no ad may be a large sticky/overlay that obscures content. The fastest way to violate this is stacking manual slots near the top on mobile where Auto Ads also land — so concentrate aggression on **desktop gutters and in-flow breaks**, and let mobile lean on Auto Ads + the in-flow slots rather than piling on rails.
- **Distinguishable, never deceptive.** Every ad must read as *an ad*, clearly separable from content — no looks-like-content, in-feed, or near-nav/CTA placement (the §12 rules below are the specific list). Aggression is *more legitimate slots*, never *sneakier* ones.
- **Account-level consent is not this command's job, but don't fight it.** Personalized-ad eligibility in the EEA/UK/CH currently requires a certified CMP the site doesn't have (see `.github/spec-adsense.md` "Consent: legal vs. contractual" and `.github/defer-ledger.md` #1). Place slots normally; don't add EEA-specific assumptions or extra consent UI here.

## Placement rules — non-negotiable (style guide §12)

- **Never in the hero**, and **never between the hero and the first content section**.
- **Never touching a CTA** — keep at least one full content section between any ad and a `.btn-primary` / the §7 final CTA band. The end-of-content slot goes *above* the dark CTA band, never inside it.
- **Never stacked back-to-back** — one ad per break, spaced by at least one full content section. Watch the narrow breakpoint where an in-flow slot and a sidebar slot can collide into an ad wall.
- **Never nested inside a card / `.nc-card` / any bordered surface** — the ad is its own bare surface in the flow.
- **Never in-feed — no ad cell inside a card grid or list.** Do not drop an ad into the cells of a `.card` grid, a list of tiles, or any repeating-item feed, and do not style a `.nc-ad` to look like one of those items. An ad sharing a grid's visual rhythm reads as *part of* the content (a looks-like-content deception AdSense bans), and if the grid items are **clickable** (`<a class="card" href=…>`, the case-study/tile pattern) it also becomes an accidental-/encouraged-click trap — a policy strike. Treat a card grid as **one unit**: place ads *before or after* the whole grid, never among its items. The only "gridded-looking" ad allowed is a `multiplex` unit as an **endcap below** the grid — clearly its own block, not a cell within it.
- **Honor suppression** — if the page is in `adsenseSuppressedRoutes` (currently `/`, the homepage), it gets **no** markers; they'd be stripped anyway. Confirm against `generateBaseHTML.js` before placing.

## Building an ad gutter (structural rail)

This is the one structural move you may make: take a content section that wastes desktop horizontal space and split it into a **content column (~75%) + gutter column (~25%)**, with a sticky `sidebar-*` ad in the gutter. The gutter exists **only** to host the ad — it is an ads concern, not an editorial one, which is why it lives in this command and not `/page-restyle`.

**When to build one (build wherever a section genuinely qualifies):**

- The section is **long and text/bullet-heavy** with obvious empty horizontal space on desktop, and
- the page **scrolls well past** the section (so a sticky rail stays useful), and
- the content column stays **readable after the split** — never let it fall below ~60ch. The gutter comes out of the dead space, *not* out of the reading measure. If 75/25 would crush the text, don't split.
- **Up to 2 gutters on a long, multi-screen page** (one on a short/medium page). Build a rail on every qualifying section up to that cap — a second rail on a genuinely long page is fill, not a cage; the cap exists only so the page never reads as wall-to-wall rails. Separate the two rails by several screens of scroll, alternate their sides, and use distinct `sidebar-*` aliases (e.g. `sidebar-right` then `sidebar-left-2`) so no unit repeats. If more than two sections qualify, pick the two longest.

**Side — alternate across the site.** Put the gutter **left or right**, choosing the side *opposite* the section's own focal element (a callout, figure, or pull-quote), and vary the side from page to page so the site doesn't feel like every page has a right rail. Record the chosen side (and why) in the placement table.

**The canonical pattern** is the whole-page grid in [`server/pages/ekg-simulator.html`](../../server/pages/ekg-simulator.html) (`.page-grid` + `.sidebar-ad-left/right`, collapsing to one column at its breakpoint). Yours is the **section-scoped** version of the same idea — apply the grid to the one section, not the whole page. Recipe (scope the class to the page, e.g. `.dc-rail-section`):

```css
.rail-section {
  display: grid;
  grid-template-columns: minmax(0, 1fr) clamp(240px, 25%, 300px); /* content | gutter-right */
  gap: clamp(1.5rem, 3vw, 2.5rem);
  align-items: start;
}
.rail-section.rail-left {        /* gutter on the left instead */
  grid-template-columns: clamp(240px, 25%, 300px) minmax(0, 1fr);
}
.rail-section.rail-left > .rail-content { grid-column: 2; }
.rail-section.rail-left > .rail-gutter  { grid-column: 1; grid-row: 1; }
.rail-gutter { position: sticky; top: 90px; }   /* clears the sticky header */

/* HIDE ON MOBILE — chosen behavior. No reflow, no ad wall. */
@media (max-width: 900px) {       /* match the page's existing section breakpoint */
  .rail-section,
  .rail-section.rail-left { grid-template-columns: 1fr; }
  .rail-gutter { display: none; }
}
```

```html
<section class="… rail-section">          <!-- add rail-left for a left gutter -->
  <div class="rail-content">
    …the section's existing markup, verbatim…
  </div>
  <aside class="rail-gutter">
    <!-- nc-ad: sidebar-right -->          <!-- sidebar-left when rail-left -->
  </aside>
</section>
```

**Non-negotiables for the gutter:**

- **Content preserved verbatim.** You wrap the section's existing children in `.rail-content`; you do not edit, reorder, or restyle them. Diff must show only the new wrappers + the `<aside>`.
- **Mobile hides the gutter** (`display:none` at the breakpoint). Mobile keeps its in-flow + Auto Ads; the rail unit does not reflow into the flow.
- **Gutter width stays within the 320px sidebar cap** — `clamp(240px, 25%, 300px)` keeps the `.nc-ad--sidebar-*` 320px ceiling honored at every viewport.
- **Use a `sidebar-*` alias** (`sidebar-left` / `sidebar-right`; reach for the `-2` variants for a second rail). Match the alias to the side, and never repeat the same unit twice on one page.
- All §12 rules still apply inside the gutter: not touching a CTA, not nested in a card, sticky-positioned but spaced so it never abuts another ad.

## Marker + wrapper mechanics

- Drop the marker exactly as `<!-- nc-ad: <alias> -->` (lowercase, hyphenated). The regex is `/<!--\s*nc-ad:\s*([a-z0-9-]+)\s*-->/gi`.
- **Wrapper convention:** the renderer emits its own `.nc-ad` wrapper. Keep the page's *surrounding* layout wrapper divs and their classes (`.ads-column`, `.horizontal-ad`, `.horizontal-ad-container`, `.sidebar-ad-right`, etc.) so existing CSS still applies. When removing a hardcoded `<ins>`, replace **only** the `<ins>` and its inline `(adsbygoogle…).push({})` script with the marker — keep the wrapper. If you're adding a fresh in-flow slot with no existing wrapper, a bare marker between blocks is fine; `.nc-ad` already centers and spaces itself.
- An unknown alias renders an inert `<!-- nc-ad: unknown slot "x" -->` comment — so a typo silently produces no ad. Double-check every alias against `SLOTS`.
- **No page-local loader, no per-page ad-*container* CSS.** The one loader lives in `head.html`; `.nc-ad` container styles live in `client/public/styles.css`. Adding either on the page is a defect — strip it if you find one. **Exception:** the *grid/layout* CSS for a gutter split (the `.rail-section` recipe above) is page structural CSS, not ad-container CSS — put it in the page's own `<style>` block (scoped to a unique page class), the same place page-specific layout already lives. Don't touch `.nc-ad` itself.

## Deliverable

1. The edited HTML at its original path — old ad markup removed, new markers placed, page wrappers preserved. If you built a gutter, the section is wrapped per the recipe and its grid CSS is in the page's `<style>` block.
2. If (and only if) a placement needed an alias not in `SLOTS`, the new alias added to `SLOTS` in `adSlots.js` with a real dashboard slot ID — flag this prominently, since it implies the user must confirm the dashboard unit exists.
3. A short final message (≤6 bullets): the placement table summary (what was removed, what was added and where, which alias and why), whether you built a gutter (which section(s), which side, and why that side), the **final manual ad count vs. the page-length target** (did you hit the floor for this length tier?), and confirmation of the non-negotiables + ceiling (none in hero, none touching CTA, none stacked, none nested/in-feed, gutter content preserved verbatim + hidden on mobile, suppression honored, content still dominant + mobile density ≤ ~30%).

## Verification before you finish

- `grep "nc-ad:" $ARGUMENTS` — every alias listed must exist in `SLOTS` (no "unknown slot"). `grep "adsbygoogle"` — confirm **zero** hardcoded `<ins>` and **zero** page-local loaders remain.
- Re-walk §12: trace each placed slot against "where ads may sit / may NOT sit." Any slot that can't be justified by the table gets removed.
- **Re-check the ceiling** (the limit on aggression): with Auto Ads imagined filled *on top of* your manual slots, content still visibly dominates the page (content-to-ad ratio), and on a mobile viewport ads occupy ≤ ~30% of any single screen's height with no content-obscuring sticky/overlay. If either fails, pull the lowest-value slot until it passes — the ceiling always wins over the count target.
- Verify rendering **via the Express server** (`npm start` with `PROD` unset for dev), not Vite — markers only expand server-side, and dev mode emits `data-adtest="on"` test ads plus the `.nc-ad` debug overlay (dashed box + "ad slot: <alias>" label) so you can see every slot. **Never** click a live ad to test.
- **If you built a gutter:** resize the browser across the breakpoint. Confirm the gutter sits beside readable content on desktop (text not crushed, column ≥ ~60ch) and the gutter is fully hidden — not reflowed, not stacked — on mobile. Diff the section to confirm its content is byte-for-byte unchanged inside the new `.rail-content` wrapper.
- Validate tag balance — confirm you didn't orphan a wrapper `<div>` when removing an old `<ins>`, or leave the new `.rail-section` / `<aside>` unclosed.
- Note in the summary that this branch isn't live until merged to `main` with the **account owner's sign-off** (policy strikes land on the owner's account) — per the spec's pre-merge checklist.

Begin.
