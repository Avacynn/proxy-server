---
description: Stage, commit, and push ALL uncommitted changes in the working tree to get the repo current and clean.
---

Stage, commit, and push **every** uncommitted change in the working tree — modified, deleted, and untracked files — so the repo ends up current and clean. Unlike `/commit-session`, this is not scoped to files you edited in this conversation.

Procedure:

1. Run `git status` and `git diff --stat` (plus `git diff --stat --cached` if anything is staged) to see exactly what will be committed. Include untracked files in your scan.
2. If the working tree is clean, respond with "nothing to commit" and stop.
3. Sanity-check the file list before staging:
   - Refuse to stage anything that looks sensitive: `.env`, `.env.*`, `*credentials*`, `*secret*`, private keys (`*.pem`, `*.key`, `id_rsa*`), `.aws/`, `.ssh/`.
   - Refuse to stage build artifacts the repo explicitly excludes (per CLAUDE.md when present, e.g. EKGpal forbids `main` binary, `dist/`, `build/`, `node_modules/`).
   - If any such files appear, stop and ask the user how to proceed instead of committing them.
4. Stage everything with `git add -A` (after the checks above pass).
5. Write a concise commit message that accurately describes the union of changes, following the repo's commit style (check `git log --oneline -10` if unsure). If the changes span multiple unrelated concerns, use a short summary line plus a brief bulleted body — do not split into multiple commits unless the user asks.
6. Commit, then push to the current branch's upstream. If push fails because upstream is not set, set upstream to `origin/<current-branch>` and retry once.
7. Report back with the commit hash, branch, and a one-line summary of what shipped.

Do not skip hooks (`--no-verify`), do not amend, do not force-push. Go straight through without interactive confirmation unless step 3 trips.
