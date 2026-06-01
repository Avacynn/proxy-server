---
description: Merge the CURRENT branch into a target branch (default: main) — the "integrate my finished work" direction. Handles dirty trees and git snags safely, resolves conflicts properly, verifies, and pushes only after confirmation. Never force-pushes; never leaves the repo tangled.
argument-hint: <target branch to merge INTO, e.g. main (default: main)>
---

# Merge current branch → $ARGUMENTS

Merge the **current** branch into the **target** branch named in `$ARGUMENTS`. This is the "my work is done, integrate it" direction (feature → main). For the opposite — pulling the target's latest changes into your current branch to stay caught up — use `/branch-update`.

Be robust: dirty working trees, in-progress operations, missing branches, and diverged remotes are normal. Handle them cleanly. The cardinal rule is **never leave the repo in a worse or half-finished state than you found it** — if anything goes sideways, roll back to the snapshot and report.

---

## Resolve scope

- **Target** = `$ARGUMENTS` if given, else `main`.
- **Source** = whatever branch is currently checked out.
- If source == target, stop — you're already on the target, nothing to merge.
- State both in one line: `Merging <source> → <target>.`

---

## Step 0 — Snapshot the starting state

Before doing ANYTHING that mutates state, record a known-good rollback point and report it in one line:

- Current branch (the source) and its HEAD sha (`git rev-parse HEAD`).
- Whether the working tree is dirty (`git status --porcelain`).
- Whether a git operation is already in progress (see "Git snags" below).

Everything after this must be reversible back to this snapshot.

---

## Step 1 — Resolve any pre-existing git snags FIRST

Check for and clear these *before* starting the merge — starting a new operation on top of a broken state is what tangles the repo:

- **Operation already in progress** — a `.git/MERGE_HEAD`, `.git/rebase-merge/`, `.git/rebase-apply/`, `.git/CHERRY_PICK_HEAD`, or `.git/BISECT_LOG` exists. STOP. Do not start a merge on top. Report what's in progress and offer to continue or abort it first; let the user decide.
- **Detached HEAD** — not on a branch. STOP and report; there's no "source branch" to merge. Offer to create/checkout a branch at the current commit.
- **Dirty working tree** — uncommitted changes (tracked or staged). Handle, don't just bail:
  1. Show the user what's dirty (`git status --short`).
  2. Default safe path: **auto-stash with a labeled stash**, `git stash push -u -m "claude-branch-merge <source> <sha>"`, do the merge, then restore it onto the source branch at the end. Announce that you stashed and that you'll restore it.
  3. Offer the alternative of committing first via `/commit-session` if the changes are finished work.
  4. Never discard uncommitted work. Never `checkout -- .`, `reset --hard`, or `clean -f` to "make it clean."
- **Untracked files that would be overwritten** by the checkout/merge — the `-u` stash above covers most; if git still blocks, STOP and report the exact paths rather than forcing.

---

## Step 2 — Sync with the remote

- `git fetch` the relevant remotes so every comparison is against current upstream.
- If the repo has **no remote** or the target has **no upstream**, skip remote steps and note it — operate purely locally.
- **Show the merge shape:** `git log --oneline <target>..<source>` (what will land) and `git log --oneline <source>..<target>` (what target has that source lacks). Summarize in one or two lines.
- **Review gate.** If the target is shared/protected (especially `main`) and the change should be reviewed before landing, suggest a PR (or `/code-review ultra`) instead of a direct local merge. Confirm the user actually wants a direct merge before proceeding.

---

## Step 3 — Merge

1. Checkout the target: `git checkout <target>`. (If a stash from Step 1 is in play, it travels with you — it'll be restored onto the source at the end.)
2. Bring the target up to date with its remote if it tracks one: `git merge --ff-only origin/<target>`. If the local target has **diverged** from its remote (neither is an ancestor of the other), STOP and report — don't guess how to reconcile it, and don't force.
3. Merge the source in: `git merge <source>`. Default to a real merge; use `--no-ff` if the project keeps feature merges as distinct commits (check recent `git log` for the convention). Honor any extra git flags the user passed.
4. **Conflicts — resolve, don't dodge.** Open each conflicted file, understand both sides, integrate them. Never discard a side with `checkout --ours/--theirs` unless the user explicitly says which side wins. Never `--no-verify`. If a conflict is non-trivial or ambiguous, STOP and ask. If you need to back out, `git merge --abort` returns the target cleanly.

---

## Step 4 — Verify

- After a clean merge, run the project's build/test verification appropriate to what changed (infer from `CLAUDE.md` / the repo). A merge can compile on each side yet break combined — don't skip this.
- If verification fails: report it and **do not push**. Leave the merge for the user to inspect, or offer to `git merge --abort` if you haven't committed past it.

---

## Step 5 — Push (confirm first)

- Pushing the target — especially `main` — is a shared, hard-to-reverse action. **Confirm before pushing**, unless the user pre-authorized it in the `/branch-merge` invocation.
- Push with a plain `git push`. **Never force-push a shared branch.** If the push is rejected because the remote moved, STOP and report — do not force.

---

## Step 6 — Restore & return

- Return to the original branch: `git checkout <source>`.
- If you stashed in Step 1, restore it now: `git stash pop`.
  - **If the pop conflicts**, STOP and tell the user *exactly* where their work is (`git stash list`, the labeled entry) so nothing is lost. Do not drop the stash.
- Report: what merged, conflicts resolved, verification result, push status, and whether a stash was restored cleanly.

---

## Recovery contract (if anything fails mid-way)

Leave the repo on the **starting branch** with the working tree restored to the snapshot:
- Abort any half-finished merge (`git merge --abort`).
- `git checkout <source>`.
- Restore the stash if one is outstanding (or, if it can't apply cleanly, leave it in `stash list` and point the user to it by name).
- Then report plainly what happened and the current state. Never report success on a partial result.

---

## Operating rules

- **Helper, not owner.** The user reviews shared-branch changes. Confirm before any push; never assume blanket authorization.
- **No destructive shortcuts.** No `reset --hard`, no force-push, no `--no-verify`, no `clean -f`, no discarding a conflict side or a stash to make a problem disappear.
- **One operation at a time.** Never start a merge on top of an in-progress merge/rebase/cherry-pick.
- **Surface ambiguity.** Diverged history, messy conflicts, failing build, blocked stash pop → stop, restore to snapshot, report.

---

## End-of-turn summary

One or two sentences: what merged into what, conflict/verification outcome, push status, stash restored if applicable. Nothing else.
