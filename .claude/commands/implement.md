---
description: Implement the fixes from a preceding /audit or bug investigation. Triages scope first — does focused work directly, or hands off to /plan-delegate when the work is substantial and better split across agents.
argument-hint: <optional: which findings to implement, e.g. "the Critical items", "finding 2", or blank to use the latest audit/investigation>
---

# Implement — $ARGUMENTS

You are executing fixes for work **already diagnosed in this conversation** — typically the findings from a preceding `/audit`, or the root cause from a bug investigation. The diagnosis is done. Your job is to ship the fix carefully, **or** to recognize when the work is large enough that it should be decomposed and parallelized via `/plan-delegate` rather than done inline.

This command executes a known diagnosis. It is **not** an open-ended "go find and fix things" — if there's no audit and no investigation to draw on, it stops and asks.

---

## What you're implementing (source of work)

Resolve the work to do, in this order:

1. If `$ARGUMENTS` names specific findings or scope, that's the work (e.g. "the Critical and Important items", "finding 3", "the IDOR fix").
2. Otherwise, use the most recent **audit report** or **bug root-cause** established earlier in THIS conversation.
3. If neither exists — no audit, no investigation, no clear argument — **stop and ask** what to implement. Do not re-discover work from scratch.

Restate in one sentence what you're about to implement before touching anything.

---

## Phase 1 — Triage: direct vs delegate

Before writing code, decide which path this work takes. This judgment is the heart of the command.

**Implement directly (one agent, this session) when:**
- It's a single coherent change, or a small set of tightly related findings.
- One mental model covers it — a handful of files, one subsystem.
- The changes are sequential or coupled, so parallelism wouldn't help.
- A bug fix with a clear, localized root cause.

**Hand off to `/plan-delegate` when:**
- The work splits into multiple **independent** slices that could run in parallel.
- It spans several subsystems or distinct mental models.
- Blast radius is large (many files, cross-cutting), or total scope is too big to hold in one focused session.
- Slices are independently testable and have few ordering constraints between them.

When it's borderline, **prefer direct implementation** — fanning out agents carries real overhead and token cost. Delegate only when the parallelism genuinely pays for itself.

State your triage decision in one line, with the reason, then proceed to 2a or 2b.

---

## Phase 2a — Direct implementation

1. **Load context.** Read `CLAUDE.md` / `AGENTS.md` and any spec doc relevant to the subsystem you're touching. Honor every mandate they declare (testing, type-sync, file-size, load-spec-before-editing, verify-before-concluding, etc.).
2. **Plan internally.** A short plan — what files change, in what order, what the verification step is. Track it with `TodoWrite` and mark items done as you finish them; don't batch. No planning doc to disk.
3. **Implement.** Stay strictly within the diagnosed scope. Fix the finding(s) — no "while I'm here" extras, no speculative refactors, no new abstractions, no backwards-compat shims for code you're replacing in the same change. Match existing patterns in the files you touch.
4. **Verify.** Run the verification appropriate to what you changed — infer from the repo / CLAUDE.md (build, typecheck, tests). For UI or runtime behavior, exercise it; if you can't, say so explicitly rather than claiming success on a type-check alone.
5. **Report.** Files changed, verification result, anything noticed-but-left-alone (goes in the report, not the diff), anything deferred.

If the diagnosis turns out wrong or incomplete as you implement, **stop and report** rather than expanding scope to patch over it.

---

## Phase 2b — Hand off to /plan-delegate

If triage says delegate:

1. **Sketch the decomposition.** List the slices, their rough dependency order, and which subsystem each touches — a few bullets. `/plan-delegate` writes the real per-subtask docs; you're just showing the shape.
2. **Confirm before fanning out.** Spawning subagents is a large, token-heavy action. Present the decomposition, recommend `/plan-delegate`, and get a quick go-ahead — **unless** the user already told you to proceed autonomously in their `/implement` invocation.
3. **On approval, invoke `/plan-delegate`** (via the Skill tool) with a topic slug — and, when the work came from an audit, the audit artifact path (`temp_docs/<topic>/findings.json` / `AUDIT-REPORT.md`) so it decomposes the persisted findings instead of re-deriving them from conversation. It owns doc-writing and execution from there — spawning subagents directly, or running them through the Workflow tool when the slice set is large/parallel. Do not also implement slices yourself.

Never silently escalate. The user should know the work just went from a focused fix to a multi-agent run.

---

## Operating rules

- **Execute, don't re-investigate.** The diagnosis is the input; trust it unless it visibly breaks as you work.
- **Scope discipline.** Fix what was diagnosed. New problems you spot go in the report, not the diff.
- **Honor project mandates.** Whatever `CLAUDE.md` / spec docs require for the subsystem you touch.
- **No risky actions unless asked.** Don't commit, push, or run destructive operations — leave the diff for the user to review.

---

## End-of-turn summary

One or two sentences: what landed (or what was delegated, and into how many slices) and the verification result. Nothing else.
