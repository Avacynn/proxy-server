---
description: Audit a page's content for factual and clinical accuracy — flag incorrect, outdated, unsupported, or internally inconsistent statements for follow-up. Read-only; web-verifies uncertain claims.
argument-hint: <page path or name, e.g. lab-values or server/pages/acid-base-balance.html>
---

# Fact-check — $ARGUMENTS

Read-only **content accuracy** audit of the page named by `$ARGUMENTS`. This is a nursing **education** site: readers act on this content clinically, so a wrong drug dose, an inverted lab range, or a quiz whose "correct" answer is actually wrong is a patient-safety problem, not a typo. Your job is to find every statement that is **wrong, outdated, unsupported, internally inconsistent, or imprecise enough to mislead** — and flag it for human follow-up. Do **not** fix anything.

If `$ARGUMENTS` is empty, ask which page to check and stop. Do not pick a target for them.

---

## Scope resolution

Resolve to the actual **rendered content a reader sees**, not just one source file.

- **HTML pages:** a bare name resolves to `server/pages/$ARGUMENTS.html`; otherwise use the given path. Read the full file, then skim the route handler (`server/routes/pages.js`, `pageRoutes.js`) for any data injected at render time.
- **DB-backed pages** (care plans, EKG rhythms, NCLEX questions): the visible facts live in MongoDB plus a rendering template. Read the template (`server/rendering/*.js`) **and** representative data records. For a single rhythm/careplan, read its record; for an index page, sample several.
- **nclex-app content:** `nclex-app/` source and its question data.
- Shared chrome (header/footer) is out of scope unless it carries a clinical or factual claim.
- **Respect disabled content** (CLAUDE.md): the drug-guide, pathophysiology, and procedures routes are legally gated and not mounted. Audit only what is actually rendered/active; if the target is a disabled page, say so up front and note that findings are against unshipped content.

State the resolved scope and which content surfaces you checked in one sentence before findings.

---

## What counts as a claim

Anything a reader could take as fact and act on:

- **Pharmacology:** drug names, doses, routes, frequencies, max doses, contraindications, interactions, side effects, antidotes, nursing considerations
- **Reference ranges:** lab/diagnostic values **with units**, critical values, normal vitals, I&O, growth/developmental norms
- **Pathophysiology / mechanism:** "X causes Y", "Z is a sign of W", cause-and-effect and classification statements
- **Dosage calculation:** formulas and **every worked numeric example**
- **EKG rhythm criteria:** rate, regularity, P-wave morphology, PR interval, QRS width/morphology, defining features
- **Procedures / skills:** step sequences, ordering, and their rationales (sterile technique, priority of action)
- **Exam facts:** NCLEX/NGN format, length, timing, scoring, item types, pass standard
- **Quiz/test-item correctness:** is the keyed "correct" answer actually correct? Are the distractor rationales sound?
- **Numbers & claims:** statistics, pass-rate claims, content counts ("10,000+ questions", "all 8 client-need categories"), pricing, "free" claims, licensure/credential statements, legal disclaimers
- **Internal consistency:** the same fact stated two different ways across hero / cards / FAQ / body

Skip non-claims: styling, voice, layout, and marketing tone that isn't a factual assertion. That's `/page-overhaul` territory, not this.

---

## Accuracy dimensions

**Clinical correctness** — doses, ranges, mechanisms, and procedure steps match established clinical standards.

**Recency** — guidelines change; treat guideline-dependent facts as suspect and web-verify them. Common movers: CPR ratios/depths and ACLS algorithms, sepsis bundles, ACC/AHA blood-pressure categories, lipid/A1c targets, stroke treatment windows, immunization schedules, and the **NGN exam format** (overhauled April 2023 — variable length, scoring model, new item types).

**Math** — recompute **every** worked example end-to-end, independently. Flag wrong answers *and* correct answers reached by an incorrect method or formula.

**Units & ranges** — unit mismatches (mg/dL vs mmol/L), bad conversions, off-by-a-decimal, and swapped/inverted ranges (hypo- vs hyper-, low vs high cutoff).

**Terminology precision** — a wrong term that changes meaning: afferent/efferent, preload/afterload, sensitivity/specificity, agonist/antagonist, hyper-/hypo-. Imprecision that misleads is a finding, not nitpicking.

**Citations** — when content cites or links a source, `WebFetch` **every** cited URL and run a three-part check:

1. **No 404s / dead links** — the URL must resolve to live content. A 404, dead link, redirect-to-homepage, parked domain, or paywall/login stub where the claim should be is a finding.
2. **Claim is actually on the page** — confirm the specific statement the page attributes to that source is genuinely present in the fetched content. A reputable-looking domain is not enough; if the fetched page doesn't say what's cited, treat the claim as unsupported.
3. **Tier-1 clinical source** — the citation must point to a tier-1 authority (defined in Verification protocol). A blog, content farm, Quizlet/Course Hero, or AI-generated summary is a finding even when the underlying fact happens to be correct.

Flag dead links, hallucinated references, sources that don't support the claim, and non-tier-1 citations.

**Misleading omission** — a partial truth that's dangerous without the omitted caveat (a dose with no "max", a drug with no key contraindication). Flag it; don't rewrite it.

---

## Verification protocol

1. **Knowledge first.** Settle well-established facts from clinical knowledge — don't web-search the boiling point of water.
2. **Web cross-check** (WebSearch/WebFetch) anything that is time-sensitive or guideline-dependent, a specific statistic, a cited URL, or where your confidence is below High. Ground every correction in a **tier-1 clinical source**: primary guidelines and standards bodies (AHA/ACC, CDC, WHO, USPSTF, NIH/NHLBI, FDA), official `.gov`/`.edu`, the major nursing & medical organizations (NCSBN for exam facts, ANA, AACN), peer-reviewed primary literature, and established drug/clinical references (Lexicomp, Micromedex, UpToDate, Davis's Drug Guide). Reputable secondary sources (Mayo Clinic, Cleveland Clinic, MedlinePlus, StatPearls) may corroborate but should not be the sole basis of a confident correction; blogs, content farms, Quizlet/Course Hero, and AI-generated summaries are never authoritative. **Actually fetch the page before relying on it** — no 404s, and confirm the page genuinely contains the claim, don't trust the domain alone. Record the source in the finding.
3. **Recompute** all math yourself.
4. **When uncertain, flag — don't guess.** If sources conflict or you can't confirm, mark it "unverified, needs human/SME review" and state what you checked. A flagged uncertainty is more useful than a confident wrong correction.
5. **Never invent a "correct" value** to look authoritative. Wrong fixes to clinical content are worse than honest flags.

---

## Confidence & follow-up

Every finding carries a **confidence** (High / Med / Low) and an explicit **follow-up disposition**:

- `Correct to: <value>` — when you're confident of the right answer (cite the source if web-verified).
- `Verify: <what a human/SME should confirm, against what source>` — when you're not.

The user runs this to get a follow-up queue, so make each disposition actionable.

---

## Procedure

1. Resolve scope; state it in one sentence.
2. Pull all rendered content (file + template + sampled data). Batch reads in parallel.
3. Extract claims as you read, noting the location of each.
4. Triage: knowledge-settled vs needs-web-verification. Batch the web checks.
5. Recompute every numeric example.
6. Pick an execution engine (see below) sized to the page, then run the checks.
7. Produce the report.

---

## Execution engine

Match the engine to the page's size. State which you picked in one line.

**Inline (default — short pages, few claims):** run the verification procedure above directly. Extract claims, fire independent web checks in parallel where possible, report. A workflow isn't worth its overhead for a handful of claims.

**Workflow (preferred — content-dense pages, or many checkable claims):** invoke the **Workflow** tool. A slash command instructing a Workflow call is valid opt-in, so no extra user prompt is needed. Verification agents have web access (WebSearch/WebFetch). Follow the extract → verify → clinical-double-check → completeness-critic → synthesize shape:

1. **Extract** — agent(s) pull the full checkable-claim list as structured output (`schema`: `{claims: [{text, file, line, type, confidence}]}`). For dense pages, fan extraction out per content block (per rhythm, per care-plan section, per quiz batch) so no block is skimmed. Pass the resolved scope + content surfaces via `args` so agents don't re-derive them.
2. **Verify** — `pipeline()` each claim needing web cross-check through an independent verification agent. Running N web round-trips concurrently instead of serially is the main performance win. Each returns `{claim, verdict: verified|refuted|unsupported|outdated, source, correction}`, grounded in a tier-1 source per the Verification protocol above (fetch before relying — no 404s, claim must actually be on the page). Recompute math in-agent, don't web-search it.
3. **Clinical double-check** — for clinically dangerous claims (doses, routes, thresholds, contraindications, interactions), spawn a second independent verifier against a *different* tier-1 source (e.g. official PI/Lexicomp vs. a standards-body guideline); on conflict, route to "Needs human / SME review" rather than picking. Patient-facing safety earns the redundancy.
4. **Completeness critic** — a final agent re-reads the content asking "what checkable claim did extraction miss?" — the executable form of catching skimmed claims on a second pass. Loop anything new it surfaces back through verify.
5. **Synthesize** — `return` the structured results and build the Critical/Important/Minor report (plus Verified clean and the SME-review queue) from them.

Watch live progress with `/workflows`. Scale to the ask: a quick check uses single-source verification; "thorough" warrants the clinical double-check on every dangerous claim plus the completeness pass.

---

## Report format

```
# Fact-check — <page>

**Scope:** <one sentence: content surfaces checked, where the facts are sourced from>

## Critical — clinically dangerous
Wrong drug/dose/route, inverted lab range, dangerous procedure error, a quiz item keyed to the wrong answer — anything that could cause harm if acted on.
**Bold claim summary** — [file:line](path#L42) — "quoted statement" → what's wrong. **Confidence:** High · **Follow-up:** Correct to <X> (source) / Verify <Y> against <source>.

## Important — wrong but not dangerous
Outdated guideline, wrong statistic, swapped terminology, unsupported or misrepresented citation, misleading omission. Same format.

## Minor — imprecise / needs citation
Ambiguous phrasing that could mislead, missing units, an uncited factual claim that should carry a source. Same format.

## Verified clean
One terse line per high-risk area you actively checked and found correct — e.g. "All 6 dosage-calc worked examples recompute correctly", "Lab ranges match standard adult reference values". This tells the user what was *covered*, not just what failed.

## Needs human / SME review
Every Low/Med-confidence flag collected in one place — the follow-up queue. Omit if empty.
```

If a dimension is clean, record it under **Verified clean** — do not fabricate findings to fill sections. If the whole page checks out, say so plainly and list what you verified.

---

## Rules

- **Read-only.** The deliverable is the report. Never edit clinical content — a wrong fix is worse than a flag.
- **Quote the real statement** and cite a real `file:line`. For DB content, cite the record (slug/id) and the rendering template.
- **Cite your source** for every web-verified correction. No source means lower confidence, not a confident claim.
- **Citations must survive a fetch.** WebFetch every cited/linked source: no 404s or dead links, the claim must actually appear on the fetched page, and it must be a tier-1 clinical source. A live-but-irrelevant or non-authoritative citation is a finding.
- **Recompute, don't eyeball,** every number.
- **Confidence is honest.** Don't inflate Low to High; don't bury a dangerous error under Minor.
- **One home per finding** — pick the higher severity; don't duplicate.
- **Stay on facts.** Style, voice, and layout belong to `/page-overhaul`, not here.
- **Understand before flagging.** If a statement looks wrong, check whether it's a recognized clinical nuance or a project constraint (CLAUDE.md, spec docs) before calling it an error.
