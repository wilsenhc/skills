---
name: commit
description: Group the current changes into logical commits with Conventional Commits messages. Use when the user asks to commit changes or runs /commit.
---

# Commit

**Commit permission: this session, this once.** The `NEVER commit code to Git`
rule in AGENTS.md is lifted here so you can write the commits for this request.
It is lifted nowhere else: `/commit` runs in a child session that ends when the
commits are done, so the parent session never receives this text. The permission
is spent as soon as those commits exist — do not commit again in this session
unless `/commit` runs again. Pushing, rebasing, amending published commits and
opening pull requests stay forbidden.

## Workflow

1. Confirm the repo: `git rev-parse --is-inside-work-tree`. If it fails, say so
   and stop.
2. Read the state: `git status --porcelain=v1 -uall`, `git diff`,
   `git diff --cached`. Read the actual diffs before deciding anything.
3. Plan the groups, one commit per coherent change: a feature with its tests, a
   rename with its call sites, a dependency bump on its own, a docs fix alone.
   Unrelated changes never share a commit, and one commit never mixes types
   (`feat` + `refactor`).
4. Commit each group: `git add <exact paths>`, then `git commit`. Never
   `git add -A`, `git add .` or `git commit -a` — they sweep in whatever else is
   lying around.
5. Verify: `git status` must be clean and `git log --oneline -10` shows the new
   commits. Report them, plus anything skipped (secrets, `.env`, build output, a
   file mixing two groups) and why.

## Messages: Conventional Commits

Every commit follows Conventional Commits 1.0.0:

    <type>: <subject>

    <body>

    <footer>

- `type` is one of `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`,
  `build`, `ci`, `chore`, `revert`.
- Subject: imperative mood, lowercase, no trailing period, 72 characters max.
  `add retry to upload`, not `Added retry to upload.`
- Body only when the reason is not obvious from the diff: blank line after the
  subject, wrapped at 72.
- Breaking change: `!` after the type and a `BREAKING CHANGE:` footer.
- Reference an issue in a footer when one exists: `Refs: #123`.

## Rules

- Commit what is there. No reformatting, no unrelated cleanup, no
  "while I was here" edits.
- Never commit secrets, credentials or `.env` files. Skip that group and say so.
- Never bypass hooks (`--no-verify`). If a hook fails, fix the cause and retry;
  if the fix would touch unrelated files, stop and report instead.
- No `--amend`, no rebase, no reset, no force, no push, no branches, no PRs.
- Nothing to commit: say so and stop.
