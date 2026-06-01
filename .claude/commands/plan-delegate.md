---
description: Decompose the just-planned architecture into per-subtask docs in temp_docs/<topic>/, then auto-spawn a fresh subagent per doc to plan and implement it in parallel.
---

You are the big-picture orchestrator for the work just scoped in this conversation — either an **architecture just designed** here, or an **audit report** to be turned into fixes (see *Source of work* below).

Do **not** implement the subtasks yourself. Your job is two-phase:

1. **Decompose** the planned work into isolated, delegatable subtasks and write one lightweight prompt doc per subtask into `temp_docs/<topic>/`.
2. **Delegate** by spawning one fresh subagent (via the Agent tool) per subtask doc, in parallel, telling each to plan-then-implement its slice.

Each subtask doc must be self-sufficient for a fresh agent with no prior conversation context. Do not write so much that you re-do the architecture work inside the docs, and do not write so little that the next session has to re-derive the plan.

---

## Source of work

The decomposable input is one of:

1. **An architecture just designed in this conversation** — decompose the plan directly.
2. **An audit report** — from a preceding `/audit` (in-conversation), or persisted on disk. Look for it, in this order: `temp_docs/<topic>/findings.json` + `AUDIT-REPORT.md`, then any audit path named in the conversation or `$ARGUMENTS` (e.g. a `summaries/<date>-<slug>-audit.md`). **Read it before decomposing.** `findings.json` carries each finding's `file`, `line`, and `recommendation` — that is the raw material for the subtask docs; do not re-derive a fix the audit already specified.
3. **A `/implement` triage hand-off** — `/implement` sketched the slices and passed you the topic + audit path; treat it as case (2).

When the source is an audit, **default `<topic>` to the audit's slug and reuse/extend that same dir** so the audit and its decomposition live together. A narrower slug is fine when you're only decomposing part of the audit — but record the parent audit's path in `00_overview.md` and account for **every** finding in the coverage map (below), so nothing silently drops from a large report.

---

## Argument

`$ARGUMENTS` — the topic slug (kebab-case, e.g. `inventory`, `leaderboard-revamp`), optionally an audit path to decompose. If empty, default to the source audit's slug (case 2/3 above) or infer it from the planning conversation, and announce the slug you picked before writing files.

---

## Phase 0 — Branch Guard (do this first)

Before writing any docs or implementing anything (including the trivial-prerequisite escape hatch in Phase 1), check the current branch:

```bash
git rev-parse --abbrev-ref HEAD
```

- If on `main` (or `master`): create and switch to a topic branch before any writes — `git checkout -b feature/<topic>`. Announce the branch you created. Any uncommitted in-progress edits come along onto the new branch automatically — that is safe, nothing is lost, so do not stop to ask.
- If already on a non-main branch, stay on it and proceed.

This guard runs once in the orchestrator. Do **not** have subagents create or switch branches — they share one working tree and parallel checkouts would race.

---

## Phase 1 — Write the Doc Set

### Output Location & Naming

- Directory: `temp_docs/<topic>/`.
- `00_overview.md` — index + shared context (decisions made, key existing files, cross-cutting conventions, list of subtask files in build order). **When decomposing an audit, also include a findings→slice coverage map:** which finding(s) (title + `file:line`) each `NN_*.md` covers, which findings are **pulled for a decision** (need owner / policy / legal input — not delegated), and which are **held for a dedicated session** (criticals, DB migrations, or anything too large or risky for a blind slice). Every finding in the report must land in exactly one bucket so nothing silently drops. Cite the parent audit path at the top.
- `01_<slug>.md`, `02_<slug>.md`, … — one per isolated subtask, numbered in dependency order.

If the directory already exists from a prior session, read existing docs first and extend/update rather than duplicating.

### What Each Subtask Doc Must Contain

Keep each doc short — roughly one screen. Include only what a stranger needs to plan their own implementation:

1. **What** — one or two sentences on the goal.
2. **Scope** — bullets: what is in scope; what is explicitly out of scope.
3. **Key files / entry points** — exact paths the next agent should read first. Do not paste large code; point at it.
4. **Notes from planning** — only the non-obvious decisions or constraints that came out of the architecture discussion and cannot be re-derived by reading the code (naming choices, wire-format decisions, ordering constraints, gotchas). **If the slice comes from audit findings, list the exact finding(s) it resolves — the `file:line` anchor plus the audit's `recommendation` text verbatim — so the agent fixes precisely what was diagnosed instead of re-discovering it.**
5. **Depends on** — list of earlier `NN_*.md` files in this set whose output must exist first (omit if none).
6. **UI / follow-ups deferred** — anything intentionally pushed to a later session.

Do not prescribe a full implementation, file-by-file diff, or step-by-step plan. The next session plans its own implementation. Sketch a code snippet only when a specific shape (struct field, message name, function signature) was decided during planning and must match across subtasks.

### How to Decompose

- One subtask = one coherent, independently testable slice. If two slices share a wire type or schema change, that shared change is its own earlier subtask, and downstream docs reference it by filename.
- Prefer ordering by dependency: data model → server logic → client store → UI → polish.
- If a subtask is trivial (under ~20 lines, no judgment calls) and is a hard prerequisite for everything else, you may implement it now instead of writing a doc — but state in `00_overview.md` that you did, and why.
- If a subtask is too large or ambiguous to delegate cleanly, split it further or call it out in `00_overview.md` as needing another planning pass.

### Operating Rules for Phase 1

1. **Read before writing.** Inspect the files each subtask touches so paths, struct names, and existing patterns in the docs are correct. Do not invent APIs, paths, or types.
2. **Do not start implementation work**, refactors, or "while I'm here" cleanups during Phase 1.
3. **Do not restate the full architecture inside every doc** — that belongs in `00_overview.md`, referenced by the others.

---

## Phase 2 — Execute the Subtasks

After all docs are written, pick an execution engine, then run the slices. Either way you do **not** implement slices yourself — each runs in a fresh agent with no memory of this conversation.

### Pick the engine

**Prefer the Workflow tool when the run is large / parallel / write-heavy:**
- 3 or more slices that can run concurrently, or
- independently-testable slices where you want per-slice verification, live progress (`/workflows`), and resumability, or
- the run is big enough that fire-and-forget `Agent` spawns would be hard to track.

A slash command whose instructions tell you to call Workflow is valid opt-in for the tool, so taking this path needs no extra user prompt.

**Fall back to plain `Agent` spawns when the run is small:**
- 1–2 slices, trivial slices, or a tightly sequential chain where parallelism wouldn't pay for the orchestration overhead.

State which engine you picked and why in one line.

### Respect Dependencies (both paths)

If subtask docs declare a **Depends on** section, do not run dependents in the same batch as their prerequisites. Run the prerequisite layer first, wait for completion, then the next layer. Within a layer, run everything in parallel. If everything is independent, run all slices in one parallel batch.

### Path A — Workflow engine (preferred for large runs)

Invoke the **Workflow** tool. Pass the topic slug and the list of subtask doc paths via `args` (an actual JSON array, not a stringified one). The script should:

1. **One agent per subtask doc**, each given the self-contained prompt from the template below.
2. **Pipeline implement → self-verify per slice** (`pipeline()`) so each slice verifies the moment its implementation lands, with no global barrier. Pass a `schema` so each agent returns structured output — e.g. `{status, filesChanged, testsPassing, deferred, briefIssues}` — so you don't parse free text.
3. **Encode the dependency layers** above: a `parallel()` barrier per layer, layers sequentially; one batch if all independent.
4. **Working tree:** by default let all agents share the one working tree (plan-delegate already enforces slice independence, so this matches today's behavior plus pipelining/verification). Only pass `isolation: 'worktree'` when slices have unavoidable file overlap — and note that worktree-isolated changes must then be integrated back after the workflow returns, so treat that as the more involved path and say so up front.

Have the script `return` the array of per-slice structured results so you can build the final report from it.

### Path B — Plain Agent spawns (small runs)

Spawn one subagent per `NN_<slug>.md` file (excluding `00_overview.md`) **in a single message with multiple Agent tool calls in parallel**. Use `subagent_type: "general-purpose"`. Apply the dependency-layer rule above.

### Subtask Prompt (both paths)

Every slice's agent runs with no memory of this conversation, so its prompt must be fully self-contained — the `agent()` prompt in Path A, the `Agent` call's `prompt` in Path B. Use this template, filling in the bracketed parts:

```
You are picking up an isolated subtask from a larger plan. The orchestrator has written a self-contained brief for you at:

  temp_docs/<topic>/NN_<slug>.md

Also read temp_docs/<topic>/00_overview.md for shared context (decisions, conventions, cross-cutting constraints).

Your job:
1. Read those two files first. Then read the key files they point at.
2. Plan your implementation (briefly — this is one slice, not the whole feature).
3. Implement it. Follow the repo's CLAUDE.md mandates (server-mandate, type-sync, testing, file-size, spec-sync, verify-before-concluding).
4. Run the relevant verification commands from CLAUDE.md rule 5 for what you changed.
5. Return a concise summary: files changed, tests added/passing, anything you deferred, anything that surprised you about the brief (so the orchestrator can refine it).

Do not touch files outside the scope declared in your brief. Do not start adjacent subtasks — those have their own briefs and their own agents. If your brief seems wrong or incomplete, stop and report back rather than guessing.
```

Set each agent's label to a 3–5 word summary of the subtask — the `Agent` call's `description` in Path B, the `agent()` `label` in Path A (e.g. "Wire model + migration", "Server quest handler", "Client inventory store").

### After the Slices Return

Collect each slice's result — structured output in Path A, summary text in Path B. Produce a final report to the user:

- One bullet per subtask: ✅ done / ⚠️ partial / ❌ failed, with the one-line summary.
- Any slice that flagged its brief as wrong/incomplete — call this out explicitly so the user can decide whether to re-plan that slice.
- Suggested next step (e.g. "all green, ready to review the diff" or "01 and 03 done, 02 needs another planning pass before its dependents can run").

Do not yourself implement fixes for failed subtasks — surface them and let the user direct the next move.

---

## End-of-Turn Summary

One or two sentences: how many docs written, how many slices ran and via which engine (Workflow vs. Agent spawns), overall outcome. Nothing else.
