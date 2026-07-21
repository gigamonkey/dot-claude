# CLAUDE.md

## Plans Directory

The `plans/` directory contains implementation plans. The `plans/done/`
subdirectory holds plans that have already been implemented — **do not read
files in `plans/done/` unless explicitly asked to**. Those plans describe the
codebase as it was when they were written; the code has likely changed since
then and the plans may be misleading. If you need context from a completed
plan, the user will tell you.

## Git Worktree Workflow

Worktree branches are based on `origin/main`, not local `main`. **Always push
local `main` to `origin/main` before starting a worktree session** to avoid the
worktree missing unpushed commits.

When creating a worktree (via `EnterWorktree` or manually), the new branch is
named `worktree-<name>` and automatically tracks `origin/main` as its upstream.
**Always unset the upstream immediately** so that a plain `git push` doesn't
accidentally push to main:

```bash
git branch --unset-upstream
```

When ready to push, use `-u` so the local branch tracks the new remote branch
for all subsequent pushes:

```bash
git push -u origin HEAD
```

## Committing inside a yolo session

When running inside a yolo container (`yolo.py` — Claude Code in an ephemeral
Docker container), **commit on whatever branch is currently checked out**; do
**not** branch first just because that branch is `main`. The container and its
worktree/branch are already the unit of isolation, so the usual "don't commit
straight to the default branch" caution doesn't apply — branching adds a step
you'd only have to merge back.

You can tell you're in a yolo session deterministically: the env var
**`YOLO_SESSION`** is exported into every yolo container (claude sessions and
`yolo shell` alike). Its presence (`[ -n "$YOLO_SESSION" ]`) is the "am I in
yolo?" test; its **value names the session kind** — `worktree` for a worktree
session, `cwd` for a current-working-directory session.

Commit cadence depends on the session kind:

- **`worktree`**: commit at the end of each turn in which you wrote new code,
  without waiting to be asked. When asked to write a plan, you should definitely
  commit your first draft.

- **`cwd`**: wait for the user to tell you to commit.

## Merging

When asked to merge a branch, a clean textual merge is not the finish line.
**Always analyze the incoming changes for semantic conflicts** — changes that
merge cleanly but invalidate assumptions on the receiving branch. Review the
incoming diff (`git diff <pre-merge-head> <merge-commit> --stat`, then the
interesting files) and check it against what the current branch changed or
asserts:

- Code vs. code: renamed/refactored functions the branch's new code still
  calls the old way, changed behavior the branch depends on, duplicated fixes.

- Docs/plans vs. code: claims, file:line references, and "X doesn't exist yet"
  statements that the incoming code makes stale.

Report what was checked and fix what the merge invalidated (as a follow-up
commit, so the merge commit stays a pure merge).

## macOS `sed` in-place editing

On macOS, `sed -i ''` (with a space) fails — the empty string must be attached
directly to the flag:

```bash
sed -i'' 's/foo/bar/g' file.txt   # correct
sed -i '' 's/foo/bar/g' file.txt  # wrong on macOS
```

## npm Publishing

Use **npm Trusted Publisher** (OIDC) for publishing packages to npmjs.com. This
means GitHub Actions workflows use `id-token: write` permission and
`npm publish --provenance --access public` — no `NPM_TOKEN` secret needed.

Requirements for Trusted Publisher:

- `package.json` must have a `repository.url` matching the GitHub repo

- The package must be linked to the repo on npmjs.com under Trusted Publishers

- The workflow needs `permissions: { contents: read, id-token: write }`

## Markdown

Remember that `-`, `*`, and `+` at the beginning of a line start a list even if
the line is in the middle of a paragraph. To render any of those characters as
plain text when they occur at the beginning of a line they must be escaped with
`\`.

When writing Markdown include blank lines before and after lists and between
list items.
