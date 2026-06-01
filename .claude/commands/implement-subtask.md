---
description: Plan and implement a single subtask doc (read doc + sibling overview, plan internally, implement straight through, verify, report). Accepts a path argument or uses the IDE-opened file.
---

You are executing one isolated subtask brief — typically a `temp_docs/<topic>/NN_<slug>.md` file produced by `/plan-delegate`, but could be any single-slice implementation doc.

Run straight through — do not pause between planning and implementation. The user has already decided to ship this slice; your job is to execute it carefully, not to gate it.

---

## Argument

`$ARGUMENTS` — optional path to the subtask doc. If empty, fall back to discovery (see step 1).

---

## Procedure

### 1. Locate and read the subtask doc

Resolve the doc path in this order:

1. If `$ARGUMENTS` is non-empty, treat it as the path.
2. Otherwise, check the most recent `<ide_opened_file>` tag in the conversation — that's the file currently open in the user's editor. (Opening a file in the IDE sends only the *path* in this tag; the contents are not auto-attached, so you must `Read` it.)
3. Otherwise, look for `@`-attached files in the user's prompt that match `*subtask*`, `*_*.md` under `temp_docs/`, or a single-doc brief shape. If exactly one candidate, use it.
4. If none of the above resolves to a single doc, stop and ask the user which file to execute. Do not guess.

Once located, **Read** the doc fully before proceeding.

- If a sibling `00_overview.md` (or equivalently-named shared-context file) exists in the same directory, read it for cross-cutting decisions, conventions, and constraints.
- Read the repo's root convention files if present: `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, or the closest `CLAUDE.md` up the tree. Honor any mandates they declare (testing, type-sync, file-size, verify-before-concluding, etc.).
- Read the key files / entry points the doc points at — do not paraphrase them, open them.
- Check the doc for a **Depends on** section. If it lists prerequisite docs, verify their output actually exists in the codebase (grep for the named symbols/files). If a prerequisite isn't done, stop and report.

### 2. Plan (internal)

- Build a short plan: what files change, in what order, what the verification step will be.
- Use `TodoWrite` to track the plan as discrete tasks. Mark each completed as you finish — don't batch.
- Do not write a planning doc to disk. The brief is the plan; the todos are the breakdown.

### 3. Implement

- Stay strictly within the scope declared in the brief. Do not start adjacent subtasks, do not refactor "while you're here", do not add features the brief didn't ask for.
- Match existing repo patterns (naming, error handling, imports, test layout). Reference files the brief points at as the canonical pattern.
- If the brief specified a struct field, message name, function signature, or wire format, use that exact shape — downstream subtasks may depend on it.
- If you discover the brief is wrong, contradicted by the code, or missing a critical piece, **stop and report** rather than guessing. Do not silently expand scope to patch over an incomplete brief.

### 4. Verify

- Run the verification commands appropriate to what you touched. Infer from the repo:
  - Go changes → `go build ./...` and any relevant `go test` packages.
  - TypeScript/Vue changes → `tsc --noEmit` or `npm run typecheck`, plus relevant `npm test` / `vitest`.
  - Schema/migration changes → migration up/down dry-run if the repo supports it.
  - If the repo's root convention file (CLAUDE.md/AGENTS.md) names specific verification commands, prefer those.
- Do not declare success on type-check alone if the brief implies runtime behavior. For UI changes, say explicitly that runtime verification was not performed unless you actually exercised it.

### 5. Report

Concise summary, one screen max:

- **Files changed:** bulleted list of paths.
- **Verification:** what you ran and what passed/failed.
- **Out of scope but noticed:** anything you saw that's broken/relevant but intentionally left alone (helps the orchestrator).
- **Brief issues:** anything in the doc that was wrong, ambiguous, or incomplete — so the orchestrator can refine future briefs.
- **Deferred:** anything the brief explicitly punted to a later doc that's now ready to pick up.

---

## Operating Rules

1. **One slice only.** Do not start work declared by another subtask doc, even if it looks easy.
2. **No new abstractions.** If the brief asks for three similar lines, write three similar lines. Premature abstraction across subtask boundaries breaks the dependency model.
3. **Honor server-authoritative / type-sync / file-size mandates** if the repo's CLAUDE.md declares them.
4. **Don't touch the brief itself** unless asked — it's the orchestrator's artifact. Surface problems in your report.
5. **No backwards-compat hacks** for code you're replacing in this same subtask. If the brief says "migrate three components to the store", delete the old local state cleanly — don't leave shims.

---

## End-of-Turn Summary

One or two sentences: what slice landed, verification result. Nothing else.
