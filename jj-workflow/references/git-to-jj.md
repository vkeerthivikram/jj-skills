# Git → jj translation table (jj 0.45.1)

For users migrating from Git. Read the conceptual shifts first — most Git confusion in jj comes from habits that have no jj equivalent, not from missing commands. Everyday workflow guidance lives in [SKILL.md](../SKILL.md).

## Conceptual shifts

| Git concept | jj equivalent | What changes |
|---|---|---|
| HEAD + dirty working tree | `@` (the working-copy commit) | Every jj command snapshots the working copy into `@` first. A "dirty" state is just a commit whose description is not final yet. |
| Staging area / index | none | Nothing to `add`. All working-copy changes are in `@` automatically. Partial selection happens at squash/split time, not commit time. |
| Commit hash | change ID + commit ID | The change ID is stable across rewrites (like a Gerrit Change-Id); the commit ID changes whenever content does. Refer to work by change ID. |
| Branch | bookmark | Same idea, new name. `main*` = unpushed local move; `main??` = conflicted; `main@origin` = remote-tracking view. |
| Checked-out branch | no such concept | `@` is what you edit. Base new work on a bookmark with `jj new main`; nothing is "checked out". |
| Upstream branch (tracking) | tracked remote bookmark | `jj bookmark track <name> --remote=<origin>`; fetch then propagates remote moves into the local bookmark. |
| `commit --amend` | just edit files | Editing `@` amends it by default. Amending is the normal state, not a repair. |
| Stash | not needed | Every intermediate state is already a commit. Set work aside with `jj new`, bring it back with `jj squash`/`jj edit`. |
| Reflog | `jj op log` (+ `jj evolog`) | `op log` records every operation on the repo; `evolog` records how one change evolved. `jj undo`/`jj redo`/`jj --at-op` navigate it. |
| Force-push safety opt-in (`--force-with-lease`) | always on | `jj git push` refuses whenever the remote moved since your last fetch. |

## Command translation

### Setup and inspection

| Git | jj (0.45.1) | Notes |
|---|---|---|
| `git init` | `jj git init [dir]` | `jj init` is removed ("You probably want `jj git init`"). Colocated (`.git` + `.jj`) is the default; `--no-colocate` opts out. |
| `git init --bare` | (still `git init --bare`) | Bare repos have no working copy, so jj does not manage them. |
| `git clone <url>` | `jj git clone <src> [dst]` | Also sets `trunk()` and tracks the remote's default bookmark (if the remote advertises one). |
| `git status` | `jj st` | Shows `@`, its parent, working-copy changes, conflicts, and conflicted bookmarks. |
| `git diff` | `jj diff` | Diff of `@` (the working copy) against its parent. |
| `git diff --staged` / `--cached` | — | No staging area; the working copy is already the commit. |
| `git diff <commit>` | `jj diff --from <rev>` | Or `jj diff -r <rev>` for a change's own diff. |
| `git diff <a> <b>` | `jj diff --from <a> --to <b>` | |
| `git log` | `jj log` | Default revset: `@`, ~2 ancestor levels of mutable heads, `trunk()`. `jj log -r ::` for everything. |
| `git log -p` | `jj log -p` | `--stat`, `-s` (summary) also work. |
| `git log <file>` | `jj log <path>` | Path filters use fileset syntax. |
| `git log --oneline <rev>..` | `jj log -r '<rev>..'` | Revsets (see the jj-revsets skill). |
| `git show <commit>` | `jj show <rev>` | Metadata + patch of one change. |
| `git blame <file>` | `jj file annotate <file>` | `-r <rev>` to start elsewhere. Shows the change ID per line. |
| `git gc` | `jj util gc` (or `git gc` colocated) | |
| `git reflog` | `jj op log` | Repo-level operation log; `jj evolog -r <rev>` for one change's rewrites. |

### Recording changes

| Git | jj (0.45.1) | Notes |
|---|---|---|
| `git add <file>` | — (not needed) | Snapshotting is automatic on every command. |
| `git add -p` | — (not needed for committing) | To split work between commits later, see `jj squash --interactive` / `jj split` (jj-history-surgery). |
| `git commit -m "msg"` | `jj describe -m "msg"` then `jj new` | Or in one step from a finished `@`: `jj commit -m "msg"` (= describe + new). |
| `git commit -a -m "msg"` | same as above | There is no `-a`; deletions and new files are captured automatically. |
| `git commit --amend -m` | `jj describe -m "msg"` | Editing files already amends `@`; describe updates the message. |
| `git commit --amend` (content) | just edit files | The next jj command snapshots the edits into `@`. |
| `git rebase --interactive` (reorder/squash) | `jj squash`, `jj split`, `jj rebase` | History restructuring — see the jj-history-surgery skill. |
| `git restore <file>` | `jj restore <path>` | Restores paths in `@` from the parent. `--from <rev>` picks a source. |
| `git restore --staged <file>` | — | No index to unstage. |
| `git reset --hard` (discard WC changes) | `jj restore` (no args) | Restores `@`'s content to its parent's; keeps the (now empty) change. |
| `git reset --hard HEAD~` | `jj abandon @` | Descendants rebased automatically; abandoning `@` yields a fresh empty `@`. |
| `git reset --soft HEAD~` | `jj squash` | Fold `@` into its parent. See jj-history-surgery. |
| `git revert <commit>` | `jj revert -r <commit> -B @` | Native inverse-change command; pick placement with `-o`/`-A`/`-B`. To undo a change's diff by rewriting it instead: `jj restore --changes-in <commit>` (surgery). |
| `git stash` | — (no stash) | Every state is a commit. Set the current work aside: `jj new`; pick it up again later: `jj edit <change>`. |
| `git stash pop` | `jj squash --from <change>` | Folds the set-aside change into `@`. |
| `git cherry-pick <commit>` | `jj squash --from <rev> --into @` (copy the diff) or `jj duplicate <rev>` + `jj rebase` (copy the commit) | See jj-history-surgery. |
| `git merge <branch>` | `jj new @ <branch>` | Merge = a commit with two parents; conflicts land in `@` with markers; `jj resolve` (with a merge tool) finishes them. jj also auto-merges descendants on rebase, so explicit merge commits are rarer. |

### Branches and remotes

| Git | jj (0.45.1) | Notes |
|---|---|---|
| `git branch` | `jj b l` | `--all` shows remote bookmarks too. |
| `git branch <name>` | `jj b c <name> -r@` | Create at `@` (`-r` defaults to `@`). |
| `git branch -f <name> <rev>` | `jj b s <name> -r <rev>` | Set = create-or-move. Plain move refuses to go backwards/sideways without `-B`. |
| `git branch -m old new` | `jj b r old new` | `jj bookmark rename`. |
| `git branch -d <name>` | `jj b d <name>` | Deletion propagates to remotes on `jj git push --deleted`. `jj bookmark forget` drops it locally without remote deletion. |
| `git checkout <branch>` / `git switch <branch>` | `jj new <bookmark>` | New empty `@` on top of the bookmark. To edit an existing change instead: `jj edit <change>`. |
| `git checkout -b <name>` / `git switch -c` | `jj new -m "msg"` + `jj b c <name>` | Or `jj new main; ...work...; jj b c name`. |
| `git checkout <commit>` (detached) | `jj new <rev>` / `jj edit <rev>` | Detached editing is the default mode of work in jj. |
| `git tag <name> [<rev>]` | `jj tag set <name> [-r <rev>]` | Tags are understood by jj; `jj tag list` lists them; `jj git push` pushes tags in its default revset. (`jj tag delete/track/untrack` also exist.) |
| `git grep <pat>` | `jj file search --pattern=<pat>` | Also `git grep` works in colocated repos. `jj file list` ≙ `git ls-files`. |
| `git remote add <name> <url>` | `jj git remote add <name> <url>` | `jj git remote list` / `remove` / `rename` / `set-url`. |
| `git fetch [<remote>]` | `jj git fetch [--remote <name>]` | Updates `name@remote` records; tracked bookmarks propagate to local ones. Also `--branch`, `--tracked`, `--all-remotes`. |
| `git pull` | `jj git fetch` | Nothing extra needed: descendants and bookmarks update automatically; there is no separate merge step. |
| `git push` | `jj git push` | Pushes tracking bookmarks/tags in `remote_bookmarks(remote=<remote>)..@`. |
| `git push origin <branch>` | `jj git push -b <name>` | Auto-tracks a brand-new remote bookmark. |
| `git push -u origin <branch>` | `jj git push -b <name>` | Pushing tracks it — that IS the `-u`. |
| `git push --force-with-lease` | `jj git push` | Always on: push is refused if the remote moved since your last fetch. Remedy: `jj git fetch`, resolve, push. |
| `git push --force` | (no flag) | Fetch, resolve the bookmark conflict (`jj bookmark set`), push. Deliberately hard. |
| `git push --delete origin <b>` | `jj b d <name>` + `jj git push --deleted` | |
| (review/PR per-commit push) | `jj git push -c <rev>` | Creates and tracks a `push-<change-id>` bookmark per change. |

## Habits to drop

- **`git add` before meaning anything.** Files are already in `@`. Nothing can be "lost" by forgetting to add.
- **Committing late to "lock in" work.** Snapshots are free and continuous; describe when the change gets an identity, amend freely after.
- **Checking out branches.** You never switch branches; you move `@` (`jj new`/`jj edit`/`jj next`/`jj prev`).
- **Stashing.** `jj new` to park, `jj squash --from` to unpark. Nothing ever leaves history.
- **Rewriting-then-force-push fear.** Rewrites are tracked (change IDs, `jj evolog`) and push safety is built in; rewriting published work is still a coordination question, but not a data-loss risk.

## Where to go next

- Rebase, split, squash into older commits, duplicate, abandon, recover: **jj-history-surgery** skill.
- `-r "..."` expressions beyond `@`, `@-`, bookmarks, `trunk()`: **jj-revsets** skill.
- Everyday workflow, bookmarks, sync: [SKILL.md](../SKILL.md).
