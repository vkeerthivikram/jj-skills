---
name: jj-history-surgery
description: Restructuring and rewriting history in Jujutsu (jj) - squash, split, absorb, rebase (-s/-b/-r and insert-after/before), duplicate, parallelize, amending work into older changes, plus first-class merge conflicts and resolution, and recovery via undo/redo, the operation log (jj op), and evolog. Use this skill whenever the user wants to amend or fixup a commit, move changes between commits, split or reorder a stack, edit an old change, rebase safely, resolve rebase/merge conflicts (including with :ours/:theirs), recover a lost or rewritten commit, undo any jj operation, inspect repo history at a past operation, or clean up a review stack before pushing. Everyday committing and pushing live in jj-workflow instead.
---

# Jujutsu history surgery (verified against jj 0.45.1)

Everything below was live-verified against jj 0.45.1 unless noted. Commands that
would open an editor are always shown in their non-interactive form (path args,
`-m`, `--use-destination-message`) because agents run non-interactively. When
piping output, pass `--no-pager`. To defuse any stray editor prompt, prefix with
`EDITOR=true`.

## 1. Mental model: rewriting is the normal workflow

In jj you do not tiptoe around history - you rewrite it constantly, and the tool
is built for that:

- A **change ID** (e.g. `vmykqzry`) is stable identity; the **commit ID**
  (e.g. `f7515d93`) changes on every rewrite. Address work by change ID.
- Rewriting a commit **rebases all descendants automatically** and **moves
  bookmarks** pointing at it. You never rebase the stack "after" an amend.
- If the working-copy commit (`@`) gets abandoned or emptied, jj creates a
  fresh empty `@` on top. This is normal, not data loss.
- Nothing is truly lost: every previous commit version is reachable via
  `jj evolog`, and every repo state via `jj op log` (section 10).
- A rebase that produces conflicting content does **not** stop: jj records a
  first-class conflict and finishes the operation (section 9).

**Before surgery, describe your changes** (`jj describe -m "..."`). Descriptions
make stacks identifiable in `jj log`, survive rewrites, and you will need them
to find changes again after restructuring.

## 2. Which tool when

| Goal | Command |
|---|---|
| Amend the latest commit (`@-`) with @'s changes | `jj squash` |
| Fix a specific older commit in my stack | `jj squash --into <rev> [-m ...]`, or `jj absorb` when unsure where it belongs |
| Move only some files into a target | `jj squash <paths> [--into <rev>]` |
| Auto-distribute fixup hunks into a stack | `jj absorb` |
| Break one commit into two | `jj split <paths>` (non-interactive) |
| Move a commit and its descendants | `jj rebase -s <rev> -o <dest>` |
| Move a whole branch relative to a destination | `jj rebase -b <rev> -o <dest>` (default selector) |
| Move just one commit; leave descendants behind | `jj rebase -r <rev> -o <dest>` |
| Reorder / insert a commit inside a stack | `jj rebase -r <rev> -A\|-B <target>` or `jj new -A\|-B <target>` |
| Resume editing an existing change | `jj new <rev>` + edit + `jj squash --into <rev>` (preferred), `jj edit <rev>` |
| Walk up/down a stack | `jj prev [N]` / `jj next [N]`, `--edit` to edit instead, `--conflict` to jump to a conflict |
| Undo a revision's own changes | `jj restore --changes-in <rev>` |
| Create an inverse change (git revert) | `jj revert -r <rev> -o <dest>` (placement via `-o`/`-A`/`-B`; `-B @` = before working copy) |
| Reorder many commits at once | `jj arrange` (interactive editor — scripted reorder: repeated `jj rebase -r ... -A/-B`) |
| Scrap changes (keep commit shell) | `jj restore` (bare, on @) |
| Delete a change outright | `jj abandon <rev>` |
| Copy a change (new change ID) | `jj duplicate <rev>` |
| Decouple commits into siblings | `jj parallelize <revs>` |
| Drop redundant merge-parent edges | `jj simplify-parents <revs>` |

## 3. squash - move changes into another revision

Default: move ALL of @'s changes into the parent.

```sh
jj squash                                  # @ -> @- ; emptied @ is abandoned
jj squash -m "combine: new description"    # non-interactive combined message
```

Target an older commit in the stack (`--into`/`--to`, source defaults to @):

```sh
jj squash --into vmykqzry -m "add b.txt (with fixes)"
```

Fixup with file selection - move only the given paths (everything else stays
in the source, which is then NOT abandoned because it is not empty):

```sh
jj squash b.txt --into vmykqzry            # only b.txt moves
```

Behavior verified on 0.45.1:

- Source is abandoned when left empty and `--keep-emptied` (`-k`) is not set;
  `-k` keeps the emptied change (useful to preserve a placeholder commit).
- If the abandoned source and destination both have descriptions, jj wants a
  combined description - pass `-m` or `--use-destination-message` (`-u`) to
  stay non-interactive.
- `jj squash -r <rev>` squashes that revision into its parent; on a merge
  commit it fails: `Error: Cannot squash merge commits without a specified
  destination` + `Hint: Use --into to specify which parent to squash into`.
- `-i` (interactive hunk picking) exists; agents should use path args instead.
- Experimental `-o/-A/-B` options create a new commit from the moved changes.

## 4. absorb - automatic fixups into the stack

Like `hg absorb` / git autosquash, but automatic: each hunk in `--from`
(default `@`) moves into the closest ancestor in `--into` (default
`mutable()`) that last modified those lines. Ambiguous hunks stay put. A fully
absorbed, undescribed source is abandoned.

```sh
# fix a typo that belongs to some commit in your stack:
sed -i 's/alpha line/alpha line (fixed)/' a.txt
jj absorb                    # distributes each hunk to the right ancestor
jj op show -p                # review exactly what absorb rewrote
```

Verified output: `Absorbed changes into 2 revisions:` listing the destination
change IDs. Use absorb when a fix clearly belongs somewhere in the stack but
you are not sure where; use `squash --into` when you know the target.

## 5. split - one commit into two

Non-interactive form: pass filesets. Files matching go into the **selected**
(first) commit, which stays in the original position; the remainder becomes a
new child commit that keeps the original description; `@` ends up on the
remainder.

```sh
jj split x.txt -m "just x"     # selected = x.txt (first commit), rest = child
```

```text
L                 L'
|                 |
K (split)   =>    K" (remaining, new child)
|                 |
J                 K' (selected, keeps position/change ID)
                  |
                  J
```

- `-r <rev>` splits another commit than @.
- `-m` sets the selected part's description; the other part keeps the original.
- `-p/--parallel` makes the two parts siblings instead of parent/child.
- `-o/-A/-B <target>` extract the selected changes to a new location and leave
  the remainder in place.
- Splitting an empty commit is unsupported - use `jj new`.
- With no filesets, split opens a diff editor (interactive); always give paths.

## 6. rebase - move, reorder, retarget

One selector (what moves) + one destination (where to): `jj rebase <selector>
<destination>`. Selector defaults to `-b @`.

Selectors:

- `-s <rev>` - the revision **and its descendants**:

```text
O           N'        jj rebase -s M -o O :
|           |
| N         M'        M (and N below it) become
| |         |         children of O.
| M         O
| |    =>   |
| | L       | L
| |/        | |
| K         | K
|/          |/
J           J
```

- `-b <rev>` - the whole "branch" relative to the destination: the rev's
  ancestors back to (excluding) ancestors shared with the destination, plus all
  descendants; equivalent to `-s 'roots(<dest>..<rev>)' -o <dest>`:

```text
O           N'
|           |
| N         M'
| |         |
| M         | L'      jj rebase -b L -o O :
| |    =>   |/        L, M, N, K move;
| | L       K'        J does not (shared with
| |/        |         the destination).
| K         O
|/          |
J           J
```

- `-r <rev>` - only the named revisions; **descendants stay behind**, filling
  the hole onto the moved rev's parent(s). Multiple `-r`/set args keep
  internal dependencies:

```text
M          K'        jj rebase -r K -o M :
|          |
| L        M         K moves onto M;
| |   =>   |         its former child L is
| K        | L'      rebased onto K's old
|/         |/        parent J.
J          J
```

Destinations:

- `-o/--onto <revs>` - plain rebase; existing descendants of the target are
  untouched (all diagrams above).
- `-A/--insert-after <target>` - selected revs become children of the target,
  and the target's existing descendants are rebased ON TOP of the moved revs.
  Also used for reordering: `jj rebase -r M -A J` reverses a chain.
- `-B/--insert-before <target>` - selected revs land on the target's parents,
  and the target (with its descendants) is rebased on top of the moved revs.

Useful extras: `--skip-emptied` (drop commits that become empty due to the
move), `--simplify-parents`, `--keep-divergent`. `-o` may be repeated to create
a merge parent: `jj rebase -s L -o K -o M`.

If a working-copy commit is abandoned by a rebase, jj just makes a new empty @.

## 7. Inserting and resuming: new -A/-B, edit, next/prev

`jj new <rev>` starts a new empty child and makes it @. `jj new A B` creates a
merge (two parents). `-m` sets the description. Insertion forms rebase the
neighbors:

```text
jj new -A A : insert after A        jj new -B C : insert before C

    B   C                                C
     \ /                                 |
      C     =>      @                A   B    =>    @
      |           / |                 \ /          / \
      A          A  (B and C            A         (A and B become
                     rebase on @)                  parents of @)
```

`jj edit <rev>` points the working copy at an existing change - resume working
on an old change in place. The generally recommended pattern is instead:

```sh
jj new <rev>            # new child of the old change
# ...edit files...
jj squash --into <rev>  # move the fixes into the old change
```

Movement:

- `jj next [N]` / `jj prev [N]` - create a new empty @ on the child/ancestor
  N steps away. Fails with `Error: The working copy must not have any
  children` when @ is not a tip; add `--edit` (or use `jj edit`) to move onto
  an existing commit instead of creating a child.
- `--conflict` jumps to the next/previous conflicted revision.

## 8. restore, abandon, duplicate, parallelize, simplify-parents

```sh
jj restore                          # undo ALL changes in @ (keeps description)
jj restore --changes-in <rev>       # undo a revision's own changes, in place
jj restore --from <rev> <paths>     # take paths' content from another revision
```

`restore` (bare) is like abandon-but-keep-the-shell: the commit survives,
empty, with its description and metadata. `--changes-in <rev>` is the same
idea applied to any revision. Use `jj diffedit` for partial (hunk-level)
restores in a diff editor. All forms rebase affected descendants.

```sh
jj abandon <revs>        # drop changes; descendants rebase onto parents.
                         # bookmarks on them are DELETED (use --retain-bookmarks
                         # to move them to the parents instead)
jj duplicate <revs>      # copy with NEW change IDs, same descriptions,
                         # on the same parents; -o/-A/-B to place elsewhere
jj parallelize <revs>    # make them siblings; outside ancestry is preserved:
```

```text
3                 3
|                / \      jj parallelize 1::2 :
2      ->       1   2     0 stays an ancestor of both,
|                \ /      3 becomes their merge child.
1                 0
```

`jj simplify-parents <revs>` drops merge edges that are redundant (a parent
that is also an ancestor of another parent).

## 9. Conflicts (summary - details in references/conflicts.md)

Conflicts are first-class objects, not error states:

- A rebase/merge that conflicts **completes**; conflicted files materialize in
  the working copy with jj conflict markers; `jj log` marks such commits
  `(conflict)` (graph glyph `×`), `jj status`/`jj resolve --list` name paths.
- Resolve by editing the markers and saving, or with tooling:

```sh
jj resolve --list                    # conflicted paths at @
jj resolve --tool :ours  <path>      # keep side #1 (first parent)
jj resolve --tool :theirs <path>     # keep side #2 (second parent)
```

- For a conflicted commit mid-stack, jj's own printed hint is the recipe:
  `jj new <conflicted-rev>`, fix, then `jj squash --into <conflicted-rev>`.
- Find conflicts anywhere: `jj log -r 'conflicts()'`; jump to them with
  `jj next --conflict` / `jj prev --conflict`.
- Pushing conflicted commits is refused (`Won't push commit ... since it has
  conflicts`) unless `jj git push --allow-conflicts`.
- Bookmarks can conflict too (`name??` in log) - see references/conflicts.md.

## 10. Recovery: undo/redo, evolog, operation log

Nothing recorded in the op log is lost. Three layers of recovery:

1. **Undo/redo the last operation** (each jj command = one operation):

```sh
jj undo        # restores the state before the last operation
jj undo        # again -> walks one operation further back
jj redo        # reverses the previous undo (editor-like undo/redo)
```

2. **Per-change history** - every previous version of a change, with diffs and
   the operation that produced it. This is where you find pre-rewrite commit
   IDs to resurrect content from:

```sh
jj evolog -p               # evolution of @ with patches
jj evolog -p -r <change>   # old versions, e.g. after a bad squash/split
```

3. **Whole-repo time travel** via the operation log (`jj op` = `jj operation`):

```sh
jj op log                          # every operation ever, with args
jj op show [-p] [<op-id>]          # what an operation changed (default: last)
jj op diff --from <op1> [--to <op2>]
jj --at-op <op-id> log             # read-only view of the repo at that op
jj op restore <op-id>              # rewind repo to that op (creates a NEW op,
                                   # does not erase history; undoable)
jj op revert [<op-id>]             # apply the inverse of one operation
jj op abandon / jj op integrate    # prune / adopt stray unintegrated ops
```

`jj --at-op` accepts any unambiguous op-ID prefix; inspection there is
read-only. Divergent changes (same change ID, two visible commits - possible
after concurrent operations) are shown as `(divergent)`; address one version
via `<change>/0` offsets or all via `change_id(<change>)`, then unify with
`jj converge` (see references/conflicts.md for caveats).

## 11. Safety rules

- **Immutable commits**: by default `::(trunk() | tags() |
  untracked_remote_bookmarks() | untracked_remote_tags())` cannot be rewritten.
  Attempts fail with `Error: Commit <id> is immutable`. Escape hatch (rarely
  correct): the global `--ignore-immutable` flag. Do not rewrite published
  history casually - rewrite your own mutable stack instead.
- **Fetch, don't force**: there is no force-push flag. `jj git push` behaves
  like `git push --force-with-lease`: if the remote bookmark moved
  unexpectedly, the push is refused and the remedy is `jj git fetch` +
  resolve the resulting conflict (references/conflicts.md). A pushed bookmark
  target becomes trunk and therefore immutable; jj then moves your @ off it
  automatically ("The working-copy commit became immutable").
- **Prefer recovery over prevention**: if a surgery goes wrong, `jj undo` (or
  `jj op restore`) before attempting to fix forward. Check `jj evolog` before
  re-doing work you think you lost.
- Push hygiene (descriptions, `--dry-run`, bookmark targeting) belongs to the
  jj-workflow skill; only its history-related edges live here: pushes are also
  refused for undescribed commits and (without `--allow-conflicts`) for
  conflicted commits.

## 12. Quick reference

| Task | Command |
|---|---|
| Amend parent with @ | `jj squash` |
| Amend specific older commit | `jj squash --into <rev> -m "..."` |
| Amend only some files | `jj squash <paths> [--into <rev>]` |
| Keep emptied source | `jj squash -k` |
| Absorb fixes automatically | `jj absorb` (+ `jj op show -p` to review) |
| Split by paths | `jj split <paths> -m "selected msg"` |
| Split into siblings | `jj split -p <paths>` |
| Extract part elsewhere | `jj split <paths> -o/-A/-B <target> -m "..."` |
| Move rev + descendants | `jj rebase -s <rev> -o <dest>` |
| Move branch (default) | `jj rebase -b <rev> -o <dest>` / `jj rebase -o <dest>` |
| Move rev only | `jj rebase -r <rev> -o <dest>` |
| Reorder/insert | `jj rebase -r <rev> -A <t>` / `-B <t>`, `jj new -A <t>` / `-B <t>` |
| Resume old change | `jj new <rev>` ... `jj squash --into <rev>`; or `jj edit <rev>` |
| Walk stack | `jj prev` / `jj next` (+`--edit`, `--conflict`) |
| Undo a revision's diff | `jj restore --changes-in <rev>` |
| Invert a change as a NEW change | `jj revert -r <rev> -B @` |
| Undo @'s diff | `jj restore` |
| Take file content from elsewhere | `jj restore --from <rev> <paths>` |
| Delete changes | `jj abandon <revs>` |
| Copy changes | `jj duplicate <revs>` |
| Decouple commits | `jj parallelize <revs>` |
| List conflicts | `jj resolve --list`, `jj log -r 'conflicts()'` |
| Pick a conflict side | `jj resolve --tool :ours\|:theirs <path>` |
| Last-op undo / redo | `jj undo` / `jj redo` |
| Change history | `jj evolog -p [-r <rev>]` |
| Repo history | `jj op log`, `jj op show -p`, `jj op diff` |
| Inspect past state | `jj --at-op <op-id> log` |
| Rewind repo | `jj op restore <op-id>` (new op; undoable) |
| Invert one op | `jj op revert <op-id>` |
| Unify divergent change | `jj converge -r 'change_id(<id>)'` |
