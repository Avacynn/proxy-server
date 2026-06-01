---
description: Write a markdown summary of issues identified + fixes applied in this Claude Code session.
---

Write a single markdown document summarizing the review and/or fix work from **this Claude Code session**. Pull content from your own tool-call history and the conversation — do not re-scan the codebase to rediscover what you already know.

If the session lacks reviewable content (no analysis, no diagnoses, no fixes — just chitchat or trivial edits), respond `nothing to summarize` and stop.

## Output location

Summaries are written to a `summaries/` folder at the repo root, with a sortable timestamp prefix so listings are chronological top-to-bottom.

**Filename format:** `summaries/<YYYY-MM-DD-HHMMSS>-<slug>.md`

Get the timestamp with `date +%Y-%m-%d-%H%M%S` via Bash at write time — do not guess from session context.

## Argument

`$ARGUMENTS` is optional and may be one of:

- **A kebab-case topic slug** (e.g. `simulator-cleanup`) → write to `summaries/<timestamp>-<slug>.md`.
- **A relative or absolute path** ending in `.md` → write there verbatim (no timestamp injection).
- **Empty** → infer a slug from the session work and write to `summaries/<timestamp>-<slug>.md`. If no clear topic emerges, use `session` as the slug.

Create the `summaries/` directory if it doesn't exist. Announce the chosen path in your reply.

## Procedure

1. Locate the work product in this conversation: the diagnostic findings (numbered or otherwise), the fixes that landed, and any commit hashes / branch names produced.
2. Run `date +%Y-%m-%d-%H%M%S` to get the timestamp.
3. Pick the output path per the argument rules above.
4. Write the doc using the format below.
5. Report the path and a one-line gist. Don't restate the doc contents back to the user.

## Format

Use this skeleton. Drop any section that's empty rather than leaving a stub.

```
# <Topic> — Summary

<One paragraph: what was reviewed/changed and why. Mention the commit hash or branch if there is one.>

## Issues identified

If ≤8 items: a numbered list. Each item: short bold title, file:line anchor as a markdown link, one-sentence explanation.

If >8 items: group by category (Bugs / Dead code / Performance / Consolidation / etc.) and number within each group.

## Fixes applied

A table is best when there are many items:

| # | Status | (Slice/PR/Commit) | Change |

Status uses ✅ done, ⚠️ partial, ⏭ deferred. The middle column is optional — use it when fixes came in batches (e.g. plan-delegate slices) or distinct commits.

When there are only a few items, a bulleted list with status icons in front works fine.

## Files touched

Bulleted list of paths edited, each as a markdown link. Append a brief note (line-count delta, "added", "deleted", or what the change was) — but only if it's not already obvious from the title.

## Deferred / out of scope

Bullets for anything intentionally not done. Omit the section entirely if empty.

## Suggested next steps

1–3 bullets max. Things the user might want to do *after reading this doc*: manual verification, follow-up tasks, decisions to make. Omit the section if there's nothing actionable.
```

## Rules

- **Use clickable anchors.** `[filename.ext](relative/path.ext#L42)` for code references, `[useQuiz.ts](../client/src/composables/useQuiz.ts)` for files. Paths are relative to the doc's location (i.e. from `summaries/`).
- **Be terse.** A summary is a navigation aid, not a tutorial. Don't re-explain what the linked code already shows.
- **One home per item.** If a fix could fit two categories, pick one and don't duplicate.
- **Quote real artifacts.** Commit hashes, branch names, file paths, line numbers — pull them from the actual session, not estimates.
- **Don't pad.** No "executive summary," no preamble like "This document summarizes…", no "thanks for using…". Lead with the work.
- **Don't generate from a re-scan.** Use the conversation context you already have. The point of `/summary` is to capture *this session's* findings, not redo them.
- **Don't commit the doc yourself.** Writing the file is `/summary`'s only deliverable — do not run `git add`/`git commit` as part of *this* command. But the summary **is** a normal session artifact: when the user subsequently commits session work (e.g. `/commit-session`), include it like any other file you edited this session. Never read "don't commit" as "exclude the summary from a later commit" — writing a summary right before committing usually means the user wants it *in* that commit.
