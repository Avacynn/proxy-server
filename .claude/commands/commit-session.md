---
description: Stage, commit, and push only the files YOU edited in this Claude session.
---

Stage, commit, and push **only** the files you have edited in this Claude Code session using Edit, Write, or NotebookEdit. Identify them by reviewing your own tool-call history in this conversation — do not use `git add -A` or `git add .` since other sessions may be working in parallel.

Procedure:

1. List the exact file paths you've edited in this conversation. If none, respond with "no session changes" and stop — do nothing destructive.
2. Run `git status` to confirm those files actually have uncommitted changes (a file you edited may already be committed). Filter out any that are clean.
3. Stage them explicitly by path: `git add <path1> <path2> ...`
4. Write a concise commit message describing what changed, following the repo's commit style (check recent `git log --oneline -10` if unsure).
5. Commit, then push to the current branch's upstream.
6. Report back with the commit hash and a one-line summary.

Do not commit files you did not edit. Do not interactively confirm — go straight through. If push fails (e.g. needs upstream set), set upstream and retry once.
