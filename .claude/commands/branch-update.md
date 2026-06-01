---
description: Update the CURRENT branch by merging another branch into it (default: main) to keep it caught up — the "stay fresh" direction. Handles dirty trees and git snags safely, resolves conflicts properly, verifies. Stays on the current branch; never leaves the repo tangled.
argument-hint: <source branch to merge FROM, e.g. main (default: main)>
---

# Update current branch ← $ARGUMENTS

Merge the **source** branch named in `$ARGUMENTS` INTO the **current** branch, to keep your branch caught up with upstream work. This is the "stay fresh" direction (main → feature). For the opposite — integrating your finished work into the target — use `/branch-merge`.

Be robust: dirty working trees, in-progress operations, missing branches, and diverged remotes are normal. Handle them cleanly. The cardinal rule is **never leave the repo in a worse or half-finished state than you found it** — if anything goes sideways, roll back to the snapshot and report.

---

## Resolve scope

- **Source** = `$ARGUMENTS` if given, else `main`.
- **Target** = the currently checked-out branch — it stays checked out the whole time.
- If source == current branch, stop — nothing to update.
- State both in one line: `Updating <current> with latest from <source>.`

---

## Step 0 — Snapshot the starting state

Before mutating anything, record a known-good rollback point and report it in one line:

- Current branch and its HEAD sha (`git rev-parse HEAD`).
- Whether the working tree is dirty (`git status --porcelain`).
- Whether a git operation is already in progress (see "Git snags" below).

Everything after this must be reversible back to this snapshot.

---

## Step 1 — Resolve any pre-existing git snags FIRST

Clear these *before* starting — starting a new operation on top of a broken state is what tangles the repo:

- **Operation already in progress** — a `.git/MERGE_HEAD`, `.git/rebase-merge/`, `.git/rebase-apply/`, `.git/CHERRY_PICK_HEAD`, or `.git/BISECT_LOG` exists. STOP. Don't start on top. Report what's in progress; offer to continue or abort it first.
- **Detached HEAD** — not on a branch, so there's no branch to update. STOP and report; offer to create/checkout a branch at the current commit.
- **Dirty working tree** — uncommitted changes (tracked or staged). Handle, don't just bail:
  1. Show what's dirty (`git status --short`).
  2. Default safe path: **auto-stash with a labeled stash**, `git stash push -u -m "claude-branch-update <current> <sha>"`, do the merge, then restore it at the end. Announce the stash and that you'll restore it.
  3. Offer to commit first via `/commit-session` if the changes are finished work.
  4. Never discard uncommitted work. Never `checkout -- .`, `reset --hard`, or `clean -f` to force a clean tree.
- **Untracked files that would be overwritten** by the merge — the `-u` stash covers most; if git still blocks, STOP and report the exact paths rather than forcing.

---

## Step 2 — Sync with the remote

- `git fetch` so the source comparison is against current upstream.
- Prefer the remote-tracking ref (e.g. `origin/<source>`) so you pull what's actually upstream, not a stale local copy of it.
- If the repo has **no remote** for the source, fall back to the local `<source>` branch and note it.
- **Preview:** `git log --oneline <current>..origin/<source>` — what's about to come in. If it's empty, the branch is already up to date; say so and stop.

---

## Step 3 — Update (stay on the current branch)

1. **Stay on the current branch** — do NOT checkout the source.
2. Merge: `git merge origin/<source>` (or `<source>` if local is the source of truth). Default to a real merge.
   - If the user prefers rebase (`/branch-update --rebase`, or "rebase instead"), use `git rebase origin/<source>`. Rebase rewrites local history — only on un-pushed or personal branches; warn before rebasing anything already shared.
3. **Conflicts — resolve, don't dodge.** Open each conflicted file, read both sides, integrate them. Never discard a side unless the user says which wins. Never `--no-verify`. Non-trivial or ambiguous → STOP and ask. Safe escapes: `git merge --abort` / `git rebase --abort` return the branch cleanly.

---

## Step 4 — Verify

- Run the project's build/test verification after merging — upstream changes can break your branch. Report failures rather than declaring success.

---

## Step 5 — Restore

- If you stashed in Step 1, restore it now: `git stash pop`.
  - **If the pop conflicts**, STOP and tell the user *exactly* where their work is (`git stash list`, the labeled entry) so nothing is lost. Do not drop the stash.

---

## Step 6 — Push (only if needed, confirm first)

- Updating a local branch usually needs no push. If the current branch tracks a remote and the user wants the update reflected there, **confirm before pushing**.
- If you rebased a branch that was already pushed, the push requires a force — **STOP and confirm explicitly**; never force-push silently, and warn loudly if the branch is shared.

---

## Recovery contract (if anything fails mid-way)

Leave the repo on the **starting branch** (you never left it) with the working tree restored to the snapshot:
- Abort any half-finished merge/rebase (`git merge --abort` / `git rebase --abort`).
- Restore the stash if one is outstanding (or, if it won't apply cleanly, leave it in `stash list` and point the user to it by name).
- Then report plainly what happened and the current state. Never report success on a partial result.

---

## Operating rules

- **Stay put.** Never leave the user on a different branch than they started on.
- **No destructive shortcuts.** No `reset --hard`, no silent force-push, no `clean -f`, no discarding conflict sides or stashes.
- **One operation at a time.** Never start a merge/rebase on top of an in-progress one.
- **Surface ambiguity.** Diverged history, messy conflicts, failing build, blocked stash pop → stop, restore to snapshot, report.

---

## End-of-turn summary

One or two sentences: what was pulled in, conflict/verification outcome, stash restored if applicable. Nothing else.
