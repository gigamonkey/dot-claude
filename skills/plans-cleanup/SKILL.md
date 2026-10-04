---
name: plans-cleanup
description: Use this skill to clean up the plans/ directory (in the working tree, or on a separate `plans` branch), or specific plans named as arguments; to abandon, defer, or revive a named plan; or to write a new plan. Triggers when the user asks to "clean up plans", "tidy the plans directory", "move done plans", "update next-steps", "abandon the X plan", "defer the X plan", "revive the X plan", "write a plan", or "make a plan" — cleanup never runs automatically after implementing a plan; wait for the user to approve the implementation and request cleanup.
version: 1.6.0
---

# Plans Cleanup Skill

Clean up `plans/` so it reflects only what remains to be done: completed
plans move to `plans/done/`, their loose ends get recorded, and
`plans/next-steps.md` is rewritten as a forward-looking statement — not a
history. If `plans/functionality.md` exists, it is brought up to date as
part of the cleanup. Also handles single-plan moves on request:
**abandon** (to `plans/abandoned/`), **defer** (to `plans/deferred/`),
and **revive** (back to `plans/`).

## Where plans live

Resolve where `plans/` is **once, at the start**, and use that for every
path below. There are two layouts:

1. **In the working tree** — `plans/` exists at the repo root. This is
   the common case; every `plans/...` path in this skill is literal.

2. **On a `plans` branch** — the working tree has no `plans/`, but the
   repo has a branch named `plans` (local `refs/heads/plans`, or only
   `origin/plans`). Repos that send PRs upstream keep plans on an orphan
   branch like this so planning documents never pollute a PR. The branch
   holds a `plans/` directory at its root and nothing else, and is edited
   through a worktree (see below).

Check in that order:

```bash
test -d plans && echo "layout 1: working tree"
git rev-parse --verify --quiet refs/heads/plans \
  || git rev-parse --verify --quiet refs/remotes/origin/plans   # layout 2
```

If neither exists, don't create anything unprompted: ask the user whether
to create `plans/` — in the working tree, or as a new orphan `plans`
branch if they don't want plans in the branches they PR from — before
writing anything. To create the branch:
`git worktree add --orphan -b plans "$WT"`, then add `plans/` inside it.

If the working tree **is** the `plans` branch (it contains only
`plans/`), layout 1 applies for the plan files, but the code lives on
other branches: verify a plan's claims with `git grep <pattern> <branch>
-- <dir>`, `git show <branch>:<file>`, and `git log <branch>`, using the
branch the plan targets (the plan or its commit message usually names
it; otherwise the main branch).

### Working through a `plans` worktree (layout 2)

1. Look for an existing worktree of the branch in
   `git worktree list --porcelain` (the entry whose `branch` line is
   `refs/heads/plans`):

   - Listed and reachable (`test -d <path>/plans`): use it.

   - Listed but not reachable — typical inside a yolo container, where
     other checkouts on the host aren't mounted and show as `prunable`
     even though they exist — stop and tell the user: the plans worktree
     at `<path>` isn't visible from this session, so run the cleanup from
     a session on that worktree or the `plans` branch, or remove the
     worktree first. Do **not** `git worktree prune`, `git worktree add
     --force`, or commit to the branch with plumbing; all three desync
     the host's checkout.

   - Not listed: add a temporary one in the scratchpad and remove it
     once the commit is made:

     ```bash
     WT="$SCRATCHPAD/plans-wt"
     git worktree add "$WT" plans                            # local branch exists
     git worktree add --track -b plans "$WT" origin/plans    # only origin/plans exists
     ...
     git worktree remove "$WT"
     ```

2. Everywhere this skill says `plans/...`, read `"$WT"/plans/...`. Run
   `git mv`, `git add`, and `git commit` inside `$WT` (or with
   `git -C "$WT"`) so the commit lands on the `plans` branch. Commands
   that check the **code** — `git log`, `rg`, reading source — run in
   the original working tree, never in `$WT`.

## Writing a new plan

When the user asks you to "write a plan" or "make a plan" for some piece
of work, that means writing a plan file into the repo's `plans/`
directory (e.g. `plans/<plan-name>.md`, kebab-case, matching the naming
of plans already there) — not just producing a plan inline in the
conversation.

If `plans/` doesn't exist in the working tree, check for a `plans`
branch before asking to create anything — see "Where plans live".

## The invariant this skill maintains

- `plans/*.md` (top level) = plans **not yet implemented**, plus
  `next-steps.md`.

- `plans/done/` = implemented plans. They hold the analysis and history;
  nothing needs to be repeated elsewhere. (Per CLAUDE.md, done plans are
  not read in future sessions unless explicitly asked — so anything a
  future session must know has to live in `next-steps.md` or the plan
  still in `plans/`.)

- `plans/abandoned/` = plans that won't be implemented. Kept for the
  record, but out of play: not read during cleanup, not mentioned in
  `next-steps.md`.

- `plans/deferred/` = plans intentionally set aside for later. Also out
  of play — not read during cleanup, not mentioned in `next-steps.md` —
  until the user revives them.

- `plans/next-steps.md` = a brief statement of **what should happen
  next**: where things stand in a few sentences, then the remaining work.
  No changelog, no session-by-session updates, no landed-work narratives.
  It covers only plans at the top level of `plans/` — never abandoned or
  deferred ones.

- `plans/functionality.md` (optional — only some projects have it) = an
  outline of **user-visible functionality**, organized into sections with
  Markdown headers, where implemented items are checked (`- [x]`) and
  not-yet-implemented items are unchecked (`- [ ]`). When present, the
  checked items match what the code actually does and the unchecked items
  match what active plans call for.

## Scope: full sweep, specific plans, or a single move

The skill runs in one of three modes:

- **Full sweep** (no arguments): assess every plan in `plans/`, as
  described below.

- **Scoped** (arguments name one or more plans, e.g.
  `/plans-cleanup server-side-js-grading` or a session that just
  implemented a plan is asked to clean it up): assess and move **only the
  named plans**, but still update `next-steps.md` for what those moves
  change (record their residue, drop them from the remaining-plans list,
  prune any of their now-done items). If `plans/functionality.md` exists,
  update it for the named plans' functionality too (step 4). Don't
  rewrite unrelated sections or reassess other plans.

- **Abandon / defer / revive** (the user names a plan and the operation,
  e.g. "abandon the foo-bar plan", "defer the widget-enhancements plan",
  "revive the foo-bar plan"): move that one plan and update
  `next-steps.md` accordingly — see "Abandoning, deferring, and reviving
  plans" below. No status assessment, no full rewrite of `next-steps.md`.

The scoped mode exists so the session that **did the implementation** can
clean up its own plan while it still has the context — it knows better
than any later session where the implementation diverged from the plan
(see "Changes to the plan" below) and what residue is real.

However, **don't run this cleanup automatically right after implementing
a plan**. Wait until the user has reviewed and approved the
implementation — cleanup happens when they ask for it, not as an
automatic final step of implementation.

## Abandoning, deferring, and reviving plans

These are single-plan moves, done only when the user explicitly asks —
never decide on your own that a plan should be abandoned or deferred.
The distinction: **abandoned** = won't be implemented; **deferred** =
set aside, intended for later.

To abandon or defer a plan:

1. Move it (create the directory if it doesn't exist yet):

   ```bash
   git mv plans/<plan>.md plans/abandoned/<plan>.md   # or plans/deferred/
   ```

2. Update `next-steps.md`: remove the plan from the remaining-plans list
   and drop anything that exists only to support it (residue items,
   ordering dependencies that named it). If other remaining work referred
   to it ("X before <plan> because …"), rewrite those sentences so they
   stand without it. Do **not** add an "abandoned/deferred plans" section
   — `next-steps.md` stays silent about plans that are out of play.

3. If `plans/functionality.md` exists, remove **unchecked** items that
   exist only because of this plan — functionality it called for that
   isn't implemented and that no other active plan calls for. Keep any
   unchecked item another active plan also wants, and never touch checked
   items (they describe implemented functionality regardless of which
   plan drove it). If unsure whether an item came only from this plan,
   keep it. Remove a section header only if the removal leaves the
   section empty. (A revival re-adds what's removed here — the plan file
   itself remains the record of what it called for.) Report the change
   summary in numbers, as described in cleanup step 4.

4. If the reason for abandoning/deferring is known and non-obvious,
   append a one-line note at the bottom of the moved plan file (e.g.
   `**Abandoned YYYY-MM-DD:** superseded by <other-plan>.`) so a future
   reader — or a revival — has the context. Don't editorialize beyond
   what the user said.

To revive a plan:

1. Move it back from whichever directory it's in:

   ```bash
   git mv plans/abandoned/<plan>.md plans/<plan>.md   # or plans/deferred/
   ```

   If the plan isn't found in either directory, say so and stop — don't
   guess at near-matching names without confirming.

2. Read the revived plan and skim it against the current code — if it has
   sat for a while, its claims may be stale. Don't rewrite it; just flag
   anything obviously out of date to the user.

3. Update `next-steps.md`: add the plan to the remaining-plans list (one
   or two sentences, same style as the rest) and note any ordering
   dependencies with the other remaining plans. Remove the
   abandoned/deferred note from the plan file if one was added.

4. If `plans/functionality.md` exists, re-add unchecked items for
   user-visible functionality the revived plan calls for that isn't
   implemented and isn't already listed — same placement rules as
   cleanup step 4 (fit items into existing sections, match the file's
   style). If some of it has been implemented in the meantime, list
   those pieces checked. Report the change summary in numbers, as
   described in cleanup step 4.

Commit in the established style, e.g. `plans: defer widget-enhancements`
or `plans: revive foo-bar, update next-steps.md`.

## Procedure

### 1. Inventory and assess each plan

List `plans/*.md` (skip `done/`, `abandoned/`, and `deferred/`) — or, in
scoped mode, just the named plans. For each plan, determine its status by
**checking the codebase and git history, not the plan's own text** — a
plan never says it's done; the code does:

```bash
git log --oneline -20                       # recent work often names plans
rg -l <key-identifier-from-plan> website/   # do the described changes exist?
```

Verify a few of the plan's concrete claims (files it says it will create,
functions it will add, schema changes). In layout 2 these checks run
against the code in the original working tree, not the plans worktree.
Classify:

- **Complete** — the substance is implemented, even if small residue
  remains. Move it to `done/` and record the residue (step 2).

- **Partially done** — meaningful chunks remain. Keep it in `plans/`; if
  parts landed, it's fine to note that in one line in `next-steps.md`, but
  don't rewrite the plan.

- **Not started / in progress** — stays in `plans/`.

If a plan's status is genuinely ambiguous after checking the code, ask the
user rather than guessing.

### 2. Move complete plans

Before moving, compare what the plan describes against what was actually
implemented. If there are **notable differences** — steps done another
way, pieces dropped or added, a design decision reversed mid-flight —
append a section to the bottom of the plan file:

```markdown
## Changes to the plan

- **<what differed>** — <what was done instead, and why>.
```

Only notable differences: things a future reader of the done plan would
be misled without. Cosmetic drift (renamed variables, reordered steps)
doesn't qualify. If the implementation matched the plan, add nothing.
This is why the implementing session is the best one to run this skill
(scoped mode, above) — it knows the rationale for each deviation without
having to reconstruct it from diffs.

Then move it:

```bash
git mv plans/<plan>.md plans/done/<plan>.md
```

Also extract any **loose ends** — deferred verification,
follow-up tasks, "do this later" items, deviations from the plan that
future work must know about. These go into `next-steps.md` as a short
residue list (bullet points, terse). Everything else in the plan — its
analysis, alternatives considered, execution narrative — stays in the
moved file and is NOT copied into `next-steps.md`.

### 3. Rewrite next-steps.md

Rewrite it top to bottom as if written fresh today. Structure:

1. **A one-paragraph "where things stand"** — only what's needed to
   orient the remaining work. Convert any relative dates to absolute.

2. **Remaining plans** — for each plan still in `plans/`, one or two
   sentences saying what it does and, where relevant, what it's waiting
   on. Note **ordering dependencies** explicitly ("X before Y because
   …"). Reference the plan file by name; don't summarize its contents
   beyond that.

3. **Residue / loose ends** — the small follow-ups extracted in step 2
   plus any that were already listed and are still open. Drop any residue
   item that has since been done (verify in the code before dropping).

4. **Checklists that are still live** (e.g. a cutover checklist) survive,
   but prune completed items rather than annotating them ~~DONE~~.

Strip aggressively:

- "Update YYYY-MM-DD: …" paragraphs and rewrite-history preambles.

- Narratives of landed work ("X LANDED" sections) — the done plan file is
  the record; keep only its open residue bullets.

- Explanations of past decisions that no remaining work depends on. If a
  remaining task needs the context, one pointer to the done plan file
  (`see done/<plan>.md`) suffices.

The test: every sentence in the new `next-steps.md` should help someone
start the remaining work. If it only explains the past, cut it.

### 4. Update functionality.md (if present)

Check whether `plans/functionality.md` exists. If it doesn't, skip this
step — don't create it. If it does, it's an outline of user-visible
functionality organized into sections (`#`/`##`/`###` headers), with
implemented items checked (`- [x]`) and not-yet-implemented items
unchecked (`- [ ]`). Bring it up to date:

1. **Check off what now exists.** For each unchecked item, verify against
   the code (same evidence standard as step 1 — the code decides, not a
   plan's text) and flip `- [ ]` to `- [x]` for functionality that's now
   implemented. Leave genuinely unimplemented items unchecked.

2. **Add recently added functionality.** If work that landed recently
   (the plans just moved to `done/`, recent commits) introduced
   user-visible functionality not yet listed, add it as checked items.

3. **Add planned functionality.** If plans still in `plans/` call for
   specific user-visible functionality not yet listed, add it as
   unchecked items.

Place new items in the section whose header fits them; only add a new
section if nothing existing fits. Match the file's style — item
granularity, phrasing, nesting depth. Only **user-visible** functionality
belongs here: internal refactors, infrastructure, and residue-type
follow-ups stay in `next-steps.md`, not this file. Don't remove or
rewrite existing items beyond flipping their checkboxes; if an existing
item looks wrong (e.g. checked but the feature doesn't exist), flag it to
the user rather than silently deleting it.

In scoped mode, limit all three updates to functionality touched by the
named plans.

Whenever `functionality.md` changes — here or in an abandon, defer, or
revive — report a summary of the changes to the user in numbers: how
many items were checked off, how many added checked, how many added
unchecked, and how many removed (e.g. "functionality.md: 3 checked off,
2 added unchecked, 1 removed"). Omit zero counts.

### 5. Commit

One commit for the whole cleanup — including any `functionality.md`
update — in the established style:

```
plans: <what moved / what changed>, rewrite next-steps.md
```

e.g. `plans: content-migration-implementation is done — move to done/,
record residue`. In a yolo session, commit on the current branch. In
layout 2 the commit is made inside the plans worktree, so it lands on
the `plans` branch regardless of what the working tree has checked out.

## Cautions

- **Never delete a plan** — done plans move to `done/`, dropped plans to
  `abandoned/`, postponed plans to `deferred/`; only `next-steps.md`
  content gets cut.

- **Never abandon or defer a plan on your own initiative** — those moves
  happen only when the user names the plan and asks for them.

- **Don't clean up a plan you just implemented until the user approves
  the implementation** — the cleanup is user-initiated, not an automatic
  post-implementation step.

- Don't read files already in `plans/done/`, `plans/abandoned/`, or
  `plans/deferred/` to build the summary; the active plans and the
  current `next-steps.md` are the inputs. (The exception is a revive,
  which reads the one plan being revived.)

- When in doubt whether a residue item is still open, check the code; if
  still unsure, keep it (cheap) rather than silently dropping it.

- In layout 2, remove a worktree you added (`git worktree remove "$WT"`)
  once the commit is made, and never prune or force-add worktrees to get
  at a `plans` checkout that isn't reachable from the session.
