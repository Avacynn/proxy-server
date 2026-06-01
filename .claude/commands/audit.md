---
description: Audit a named system or subsystem across cleanliness, coherence, organization, security, performance, dead code, and architectural integrity. Read-only review — no fixes.
---

Audit `$ARGUMENTS` — the named system, subsystem, module, layer, or pipeline to review.

If `$ARGUMENTS` is empty, ask the user what to audit and stop. Do not pick a target for them.

This is a **read-only** review. Identify findings; do not apply fixes, refactors, or "while I'm here" cleanups. The user directs follow-up work after reading the report.

---

## Scope resolution

The named target may be:

- A directory (e.g. `client/src/network/`)
- A feature spanning many files (e.g. "auth", "bot AI", "websocket layer")
- A pipeline (data flow from one boundary to another — e.g. "join flow", "tick → broadcast")
- A single module, store, class, or service

Map the target end-to-end **before** reviewing: every file, every entry point, every callsite — including how it's invoked by the rest of the codebase. Use the Agent tool with `subagent_type=Explore` for broad searches when the system spans many files; otherwise grep/find directly.

If the named target is ambiguous, list the candidate interpretations in one short message and ask the user to disambiguate before proceeding. Do not guess.

State the resolved scope in one sentence before diving into findings.

---

## Audit dimensions

Review the target against these dimensions. Not every dimension applies to every system — skip the irrelevant ones rather than padding the report with weak findings.

**Clean**
- Naming is consistent and intention-revealing
- No leftover scaffolding, commented-out code, dead branches, debug logs
- Reads top-to-bottom without surprises; intent matches implementation

**Coherent**
- One mental model holds across the whole system; no contradictory patterns
- Conventions match the surrounding codebase
- Public surface tells a coherent story; private internals don't leak

**Organized**
- Files grouped by concern; no god-files; module boundaries are obvious
- Folder structure tracks the architecture, not historical accidents

**Modularized**
- High cohesion within modules; loose coupling across modules
- Each module has a single clear job and a clean interface
- Dependencies flow one direction; no cycles

**Separation of concerns**
- UI ≠ state ≠ business logic ≠ network/IO
- Each layer owns its own job; no leaky abstractions
- Cross-cutting concerns (logging, auth, telemetry) live in shared layers, not sprinkled

**Isolated**
- The system can be reasoned about without spelunking unrelated code
- External dependencies are explicit, minimal, and injected — not assumed globally
- Side effects are contained at clear boundaries
- Tests can target the system without standing up the world

**Unified**
- One canonical way to do each thing; no parallel half-finished implementations
- Shared types, constants, and helpers live in one place — not duplicated
- No "old + new" coexisting paths that should have collapsed

**Pipeline integrity**
- Data flows end-to-end with no silent drops or shape mismatches
- Error paths are real (propagated, logged, surfaced) — not swallowed
- Backpressure, ordering, idempotency, and retries are correct where required
- Boundary contracts (wire formats, schemas) match on both sides

**Optimized / Performance**
- Hot paths don't allocate, scan O(N), or do redundant work per frame/tick
- Caches, pools, and batching are used where they pay off
- No accidental N+1, no quadratic loops on growing collections
- UI updates are throttled when bound to high-frequency state
- See project-specific perf rules (CLAUDE.md, spec docs) if present

**Security**
- Authn/authz boundaries are enforced server-side, not just in UI
- Input validation at trust boundaries; output encoding where it matters
- No secrets in code, logs, error messages, or client bundles
- Standard OWASP-class issues: injection, XSS, SSRF, CSRF, path traversal, deserialization, broken access control, IDOR, CORS misconfig
- Rate-limiting / abuse vectors on public endpoints

**Bug check**
- Off-by-ones, race conditions, missing nil/undefined checks
- Unhandled promise rejections, unhandled errors, panics that escape
- Type-vs-runtime mismatches; assumptions that don't hold under load or partial state
- TODO/FIXME comments that flag real outstanding issues
- Concurrency hazards: shared mutable state, missing locks, goroutine leaks

**Dead code**
- Unused exports, unreachable branches, orphaned files
- Feature flags, shims, or compatibility layers that have outlived their purpose
- Legacy paths no caller hits
- Tests for code that no longer exists

**Best practices**
- Language / framework idioms (Go error wrapping, Vue reactivity rules, Three.js render-loop discipline, TypeScript strictness, etc.)
- Anything the project's `CLAUDE.md` or spec docs explicitly require — consult them if the target is project-internal
- Standard patterns the wider community would expect for this kind of system

---

## Procedure

1. Resolve scope. State it in one sentence.
2. Enumerate the files. For multi-file targets, batch Read calls in parallel.
3. Cross-reference with callsites and dependencies to understand the pipeline and blast radius.
4. Note findings as you go — don't try to hold everything in working memory.
5. Pick an execution engine (see below) sized to the target, then run the review.
6. Produce the report below.

---

## Execution engine

Match the engine to the target's size. State which you picked in one line.

**Inline (default — a single module/store/service or a handful of files):** review directly in this session. Map scope, batch Read calls in parallel, note findings as you go. Use `subagent_type=Explore` for broad scope-mapping searches when the system spans many files. A workflow adds orchestration latency that small targets don't earn.

**Workflow (preferred — dozens of files, or many independent dimensions/submodules):** invoke the **Workflow** tool. A slash command instructing a Workflow call is valid opt-in, so no extra user prompt is needed. Follow the find → dedup → adversarially-verify → synthesize shape:

1. **Find** — `parallel()` one agent per applicable audit dimension (or per submodule), each returning structured findings (`schema`: `{findings: [{severity, category, title, file, line, description, recommendation}]}` — `recommendation` is the concrete fix this finding implies; it's what `/plan-delegate` turns into a subtask, so never leave it blank). Pass the resolved scope + file list via `args` so agents don't each re-derive it.
2. **Dedup across dimensions** — a barrier: collect all finders, then in plain code collapse findings that fit two dimensions to one home (the "One home per finding" rule) before spending verification on them.
3. **Adversarially verify** — for each deduped finding, spawn skeptic agents prompted to *refute* it (default to refuted when uncertain); drop the ones a majority refute. This is the executable form of "Severity is honest," "Understand before flagging," and "don't invent findings" — it kills false positives and severity inflation before they reach the report.
4. **Synthesize** — `return` the survivors as structured data, then persist the hand-off per **Output & hand-off** below (write `temp_docs/<topic>/findings.json` + `temp_docs/<topic>/AUDIT-REPORT.md`) and build the Critical/Important/Nice-to-have report in-conversation from the same data.

Watch live progress with `/workflows`. Scale to the ask: a quick audit uses single-vote verification; "thorough"/"comprehensive" warrants 3–5 skeptic votes per finding and a completeness-critic pass ("what dimension, file, or callsite did we miss?").

---

## Report format

```
# Audit — <system>

**Scope:** <one sentence: what was reviewed, file count, entry points>

## Critical
Broken, insecure, or actively harmful. Each item:
**Short bold title** — [file:line](path#L42) — one-sentence explanation.

## Important
Wrong but not on fire: dead code paths, perf hotspots, leaky abstractions, missing validation, coupling issues. Same format.

## Nice-to-have
Polish: naming, organization, redundant helpers, minor idiom drift. Same format.

## Pipeline notes
Only if the system has a meaningful end-to-end flow — a short narrative of how data moves, where state lives, and where it could break. Skip if not applicable.

## Out of scope
Anything intentionally skipped and why. Omit if empty.

## Suggested next steps
1–3 bullets max — concrete follow-ups the user might want (a focused fix PR, a deeper investigation, a spec update). When the findings are numerous and cluster into **independent, parallelizable** groups (by subsystem or file), say so and name the hand-off: `/implement` (which triages and may fan out to `/plan-delegate`), or `/plan-delegate <topic>` directly — and point at the on-disk `temp_docs/<topic>/findings.json`. Omit if everything is already actionable from the findings list.
```

If the target is clean on a dimension, do not invent findings — silence is the correct signal. If everything is clean, say so plainly in one line and stop.

---

## Output & hand-off

The report above is the in-conversation deliverable. Whenever the audit is non-trivial and a fix pass is plausible — **always** on the Workflow path, and on the inline path when you'd recommend `/implement` or `/plan-delegate` — also persist the hand-off to disk, because conversation context gets summarized and the downstream commands need the exact anchors:

- `<topic>` = a kebab slug of the audited system suffixed `-audit` (e.g. `nclex-vue-audit`). State it in one line.
- `temp_docs/<topic>/findings.json` — the structured findings array, one object per finding: `{severity, category, title, file, line, description, recommendation}`. This is the machine-readable hand-off `/plan-delegate` consumes; `recommendation` carries the concrete fix so the next agent doesn't re-derive it.
- `temp_docs/<topic>/AUDIT-REPORT.md` — the human report (the format above), whose **By-Subsystem** grouping doubles as `/plan-delegate`'s slice-boundary hint.

Writing these two files is the audit's deliverable, **not** a code edit — it does not violate read-only. Do not touch any audited source file. `/plan-delegate` looks for these paths first (then any audit path you name in the report), and reuses this same `<topic>` dir for its decomposition so the audit and its fix plan live together.

---

## Rules

- **Read-only.** Do not edit, write, or stage anything. The deliverable is the report.
- **Quote real anchors.** Every finding cites a real file and line. No vague "somewhere in the codebase" or "consider improving X" without a pointer.
- **Be terse.** Findings are navigation aids; the user reads the linked code for detail. No re-explaining what the code already shows.
- **No padding.** Skip executive summaries, preambles ("This audit reviews…"), and conclusions ("In summary…"). Lead with findings.
- **One home per finding.** If something fits two categories, pick the more severe one and don't duplicate.
- **Severity is honest.** Don't promote a nice-to-have to Critical for emphasis — it dilutes the signal. Don't demote a real bug to Nice-to-have to be polite.
- **Understand before flagging.** If code looks weird, check git blame or `CLAUDE.md` / spec docs before calling it a defect — it may encode a constraint you missed.
- **Don't recommend rewrites.** Audits surface findings, not redesigns. If the system needs a redesign, say so in one line under Suggested next steps and let the user trigger a planning session.
