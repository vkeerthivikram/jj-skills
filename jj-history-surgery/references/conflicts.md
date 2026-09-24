# Jujutsu conflicts, deep dive (verified against jj 0.45.1)

Companion to jj-history-surgery/SKILL.md section 9. All marker formats, error
strings, and workflows below were reproduced live on jj 0.45.1 unless noted.

## 1. The model: conflicts are first-class

jj does not abort a rebase or merge on conflicting content. It stores the
conflict *inside* the commit (per-file, as a list of sides) and finishes the
operation. Consequences:

- Rewrites that conflict still succeed; descendants still get rebased.
- When a conflicted commit is checked out into a working copy, the file is
  materialized with **conflict markers** (below). Snapshotting such a file
  back is fine - jj parses the markers again.
- `jj log` labels the commit `(conflict)` and uses the `×` graph glyph;
  `jj status` prints `Warning: There are unresolved conflicts at these paths:`
  plus one line per path like `f.txt    2-sided conflict`.
- When a command creates conflicts, jj prints the recovery recipe itself:

```text
New conflicts appeared in 1 commits:
  vmykqzry 5bf4b1de (conflict) add b.txt (with fixes)
Hint: To resolve the conflicts, start by creating a commit on top of
  the conflicted commit:
  jj new vmykqzry
Then use `jj resolve`, or edit the conflict markers in the file directly.
Once the conflicts are resolved, you can inspect the result with `jj diff`.
Then run `jj squash` to move the resolution into the conflicted commit.
```

Sides are numbered: side #1 = first parent (`:ours`), side #2 = second parent
(`:theirs`). A 2-sided conflict with a common base is a normal 3-way merge;
add/add or delete/modify cases have no base section.

## 2. Marker anatomy (exact 0.45.1 format)

A conflicted file contains blocks like this (real output, line "two" edited
to TWO-A on side A and TWO-B on side B, base = "base P"):

```text
one
<<<<<<< conflict 1 of 1
%%%%%%% diff from: vxlyunpw 8f40d1d3 "base P"
\\\\\\\        to: sxrxmmpm 1f59de24 "side A"
-two
+TWO-A
+++++++ swzuktzz 296ee7ae "side B"
TWO-B
>>>>>>> conflict 1 of 1 ends
three
```

How to read it:

- `<<<<<<< conflict N of M` ... `>>>>>>> conflict N of M ends` delimit one
  conflict hunk; a file can contain several (numbered).
- The `%%%%%%%` section is a unified diff **base -> side #1**; the header
  names the commits (change id, commit id, description). Seven backslashes
  (`\\\\\\\`) continue the header line.
- The `+++++++` section holds side #2's literal content (its commit named in
  the header).
- When sides are not nameable (e.g. conflicts in a commit *description*,
  visible in `jj evolog -p`), the generic headers `+++++++ side #1`,
  `%%%%%%% diff from: base`, `\\\\\\\        to: side #2` are used instead.
- Markers are lowercase (`conflict 1 of 1`, not Git's `=======`/`-------`).
  Do not add Git-style separators when editing; just produce the content you
  want and delete the marker lines.

To resolve by hand: open the file, keep/combine the content you want, remove
all `<<<<<<<`, `%%%%%%%`, `\\\\\\\`, `+++++++`, `>>>>>>>` lines, save. The
next jj command snapshots the resolution.

## 3. Finding conflicts

```sh
jj status                      # conflicted paths at @, plus bookmark conflicts
jj resolve --list              # paths + side count at @ (-r to target another)
jj log -r 'conflicts()'        # every conflicted commit in the repo
jj next --conflict             # jump @ to the next conflicted descendant
jj prev --conflict             # ... or previous conflicted ancestor
```

`jj resolve --list` on a clean revision fails with
`Error: No conflicts found at this revision` (exit code 2) - useful in scripts: a failing
`jj resolve --list` means "clean", a listing means "conflicted".

## 4. Resolution workflows

### 4.1 Resolve at @ (the common case after a conflicted rebase/merge)

```sh
jj resolve --list                      # what and where
jj resolve --tool :ours  f.txt         # keep side #1 wholesale
jj resolve --tool :theirs f.txt        # keep side #2 wholesale
jj resolve f.txt                       # external 3-way merge tool (configured)
# or: edit f.txt markers directly, save
jj resolve --list                      # must now error "No conflicts found"
```

- `:ours` / `:theirs` are builtin tools choosing side #1 / side #2.
- Only conflicts expressible as a 3-way merge can be handled by `jj resolve`;
  marker editing always works.
- External tools are invoked per conflicted file, one by one; exit the tool
  unchanged to stop resolving. Tools are configured under `merge-tools`
  (`jj help -k config`); pass `--tool <name>` to select one.
- Restrict to paths with filesets: `jj resolve <filesets>`. Default revision
  is @; `-r <rev>` elsewhere (rarely what you want, since resolution is
  itself a change).

### 4.2 Fix a conflicted commit in the middle of a stack

The conflicted commit is usually an ancestor, not @. Use jj's own hinted
recipe (verified end-to-end):

```sh
jj new <conflicted-rev> -m "resolve ..."   # child of the conflicted commit
jj resolve --tool :ours <path>             # or edit markers directly
jj squash --into <conflicted-rev>          # push the resolution into it
```

Descendants rebase again automatically; some may need their own pass if the
resolution changes content they touched ("New conflicts appeared in ..." is
printed again - iterate).

### 4.3 Conflicts that clear themselves

If a later rewrite makes both sides agree (e.g. `jj restore --from`, a rebase
that reunites the sides, or abandoning one side), the conflict disappears -
jj prints `Existing conflicts were resolved or abandoned from N commits.`

### 4.4 Pushing with conflicts

```text
Error: Won't push commit f181ca8a since it has conflicts
Hint: Rejected commit: oorrlwys f181ca8a feature | (conflict) (empty) merge AB
```

`jj git push --allow-conflicts` overrides. Default behavior protects remotes
from marker soup; resolve first whenever the conflict is real.

## 5. Bookmark conflicts (`name??`)

### 5.1 How they arise

A *tracked* bookmark that moved both locally and on the remote becomes
conflicted when the two movements are merged (usually by `jj git fetch`).
Concurrent operations in one repo can do the same. After a fetch where the
remote's `main` moved while your local `main` also moved, `jj bookmark list`
shows the three-way state (verified):

```text
main (conflicted):
  - sknvqnlr 3ca57429 add a.txt           (former position, acts as base)
  + nzwwvouq a4849f58 add c.txt           (local move)
  + vmnzmtmq 3934bd43 remote-side move    (remote's move)
  @origin (behind by 2 commits): vmnzmtmq 3934bd43 remote-side move
Hint: Some bookmarks have conflicts. Use `jj bookmark set <name> -r <rev>` to resolve.
```

### 5.2 Symptoms

- `jj log` shows `main??` on **each** candidate target (double question mark).
- Using the bare name as a revset fails:

```text
Error: Name `main` is conflicted
Hint: Use commit ID to select single revision from: a4849f5896b1, 3934bd43a96c
Hint: Use `bookmarks(main)` to select all revisions
```

- `jj status` lists conflicted bookmarks alongside file conflicts.

### 5.3 Resolution

1. **Unify the histories first** if both sides matter (recommended):
   rebase the local side onto the remote side
   (`jj rebase -s <local-side> -o <remote-side>`, or merge with
   `jj new <side1> <side2>`). Once one candidate is a descendant of the
   other, the bookmark conflict collapses to the descendant (verified).
2. **Or pick a winner outright:**

```sh
jj bookmark move main --to <rev>      # exact flag verified on 0.45.1
jj bookmark set main -r <rev>         # equivalent; the form jj itself hints
```

Moving a bookmark backwards or sideways is refused by default
(`Error: Refusing to move bookmark backwards or sideways: main` /
`Hint: Use --allow-backwards to allow it.`).
3. **Remote-side conflict** (`main@origin` shown conflicted): just
   `jj git fetch` again - pulling re-reads the remote and resolves it,
   propagating to the local bookmark.
4. Then `jj git push --bookmark main` - it succeeds once the local bookmark is
   unconflicted and the remote matches jj's record.

### 5.4 Push safety (why there is no force flag)

Before moving a remote bookmark, `jj git push` checks that the remote's
actual position matches jj's last-seen record (like
`git push --force-with-lease`, but immune to stale-lease races), that the
local bookmark is not conflicted, and that existing remote bookmarks are
tracked. On mismatch it refuses; **the documented remedy is
`jj git fetch --remote <name>` and resolving the resulting conflict**, not
any force option. Rewriting a pushed bookmark is additionally guarded by
immutability (the pushed target becomes `trunk()`); after a push jj may print
`Warning: The working-copy commit became immutable; a new commit has been
created on top of it.`

### 5.5 Tracking notes

Fetched bookmarks may arrive **untracked**. Verified on 0.45.1: `jj git
clone` tracks the default remote bookmark (and sets `trunk()`) only when the
remote advertises a default branch — its HEAD must point at a branch that
actually exists. With a dangling HEAD (e.g. bare repo defaulting to
`master` while only `main` was pushed), every remote bookmark lands
untracked. Untracked
remote bookmarks do not propagate to a local bookmark on fetch. Check with
`jj bookmark list --all` and fix with:

```sh
jj bookmark track main --remote origin    # "Started tracking 1 remote bookmarks."
```

Untracked remote bookmark *targets* are also part of the default immutable
set, so tracking affects what you may rewrite.

## 6. Divergent changes (change-ID conflicts)

A change ID with more than one visible commit is **divergent** - the change
was rewritten in two concurrent operation branches (e.g. two commands run at
the same `--at-op`, two workspaces/machines before sync). Symptoms:

- `jj log -r 'divergent()'` lists them; lines carry `(divergent)`.
- Referencing the plain change ID fails:

```text
Error: Change ID `sxrxmmpm` is divergent
Hint: Use change offset to select single revision: sxrxmmpm/0, sxrxmmpm/1
Hint: Use `change_id(sxrxmmpm)` to select all revisions
Hint: To abandon unneeded revisions, run `jj abandon <commit_id>`
```

Recovery:

```sh
jj log -r 'change_id(<id>)' --no-graph    # inspect all versions + commit ids
jj converge -r 'change_id(<id>)'          # try automatic unification
```

`jj converge` (new in recent jj, present in 0.45.1) replaces the versions
with one merged revision, rebases descendants, and moves bookmarks. Caveats
verified on 0.45.1:

- It refuses non-interactively when it cannot pick a description
  (`Could not determine which description to use. Error: Could not converge
  change`); without `--no-interactive` it prompts instead.
- In testing it also hit `Internal error: Newly-created commit ... already
  exists` when both versions had identical content. Fallback that jj itself
  suggests: keep one version and `jj abandon <commit_id>` the other(s).
- Related: `jj rebase --keep-divergent` preserves divergence when a rebase
  would otherwise collapse an identical duplicate.

## 7. Triage checklist

1. `jj status` - what is conflicted (files at @, bookmarks)?
2. `jj log -r 'conflicts()'` - where in the stack?
3. Files: `jj resolve --list`, then `:ours`/`:theirs`/marker edit; mid-stack
   via `jj new <rev>` + `jj squash --into <rev>` (section 4.2).
4. Bookmarks: `jj bookmark list`, unify histories or `jj bookmark move
   --to`/`set -r`, `jj git fetch` for remote-side, then push.
5. Divergence: `jj log -r 'divergent()'`, `jj converge` or abandon one side.
6. Anything still wrong: `jj undo` / `jj op restore <op-id>`.
