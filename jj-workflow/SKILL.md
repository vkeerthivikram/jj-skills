---
name: jj-workflow
description: Everyday version control with Jujutsu (jj) - automatic working-copy snapshots, describing and amending changes, jj log/status/diff, bookmarks, and syncing with Git remotes in colocated repos (jj git fetch/push with force-with-lease-style safety). Use this skill whenever work happens in a jj repository (.jj directory present) or the user mentions jj, Jujutsu, change IDs, working-copy snapshots, jj bookmarks, or asks how to commit, branch, or push with jj - including translating Git habits to jj. Not for history restructuring (see jj-history-surgery) or authoring revset expressions (see jj-revsets).
---

# Everyday jj workflow (jj 0.45.1)

## Mental model

In jj, the working copy IS a commit. `@` denotes the working-copy commit, and every jj command snapshots the working copy into `@` before it runs: new, modified, and deleted files are captured automatically. There is no staging area and no `git add` — if a file is in the working directory, it is in the commit. Every commit has two IDs: a **change ID** (stable across rewrites, like a Gerrit Change-Id) and a **commit ID** (changes whenever content changes). Consequence: you describe a change first, edit freely, and never "commit" — editing an already-described change amends it, which is the normal workflow, not a history-rewrite escape hatch.

Invariants to internalize before running anything:

- `@` = working-copy commit; `@-` = its parent; every command snapshots first.
- Editing `@` amends it: same change ID, new commit ID.
- Empty changes are normal and harmless. Describe early, even before editing.
- Bookmarks (jj's branches) move automatically when commits are rewritten and are deleted when their commit is abandoned.
- Commits at or below `trunk() | tags() | untracked remote bookmarks/tags` are immutable by default (overridable via `revset-aliases."immutable_heads()"`); rewriting them errors with "Commit ... is immutable" unless you pass `--ignore-immutable`.

## Orientation: what to run to answer a question

| Question | Command |
|---|---|
| What's in my working copy / any conflicts / conflicted bookmarks? | `jj st` (alias of `jj status`) |
| Where am I in history? | `jj log` |
| What exactly did I change (diff of `@`)? | `jj diff` |
| What did a given change do? | `jj diff -r <rev>` (patch) or `jj show <rev>` (metadata + patch) |

`jj log` with no arguments shows `@`, about two levels of ancestors of the mutable heads, and `trunk()` (the default `revsets.log` = `present(@) | ancestors(immutable_heads().., 2) | trunk()`). Use `jj log -r ::` for all revisions. The graph marks `@` for the working copy, a filled diamond for immutable commits, `○` for others.

Snapshotting is automatic, so `jj st` is also a commit: running it right after editing files shows them under "Working copy changes" of `@`. No `add` step ever intervenes.

## The daily loop

1. **Start a change by describing it — before editing.**
   ```
   jj new -m "Add login rate limiting"
   ```
   This creates a new empty child of `@` (default), describes it, and makes it the working copy. Describing first matters: the description stays attached to the change ID through every later amend and rebase, and an empty described change costs nothing.

2. **Edit files.** Every subsequent jj command snapshots your edits into `@`. There is nothing to stage or commit. Review with `jj st` and `jj diff`.

3. **Close the change by starting the next one.**
   ```
   jj new -m "Handle rate-limit headers"
   ```
   The previous change keeps its change ID and description; you get a fresh empty `@` on top of it.

Shortcut: `jj commit -m "msg"` finishes the *current* `@` in one step — it is exactly `jj describe -m "msg"` followed by `jj new`. Use it when the work is done and you are describing at the end anyway.

Amending is implicit: to add more work to the current change, just edit files. To fold work into an *older* change, use `jj squash` (see jj-history-surgery). To revisit a change later:

- `jj next` / `jj prev` — create a new empty `@` on the child/parent change (working-copy files update to match). `jj next --edit` / `jj prev --edit` edit the target change directly.
- `jj edit <change>` — set an existing change as the working copy and continue editing it.

Untracking files (build artifacts, .env): add the path to `.gitignore`, then `jj file untrack <path>`. The path must already be ignored; untrack removes it from the change while leaving the file on disk.

## Git interop

Init and clone are colocated by default in 0.45: the repo gets both `.jj/` and `.git/`, so plain `git` commands operate on the same directory.

```
jj git init [dir]        # new colocated repo (--no-colocate to opt out)
jj git clone <src> [dst] # clone; also sets trunk() and tracks the default bookmark
```

- `jj init` was removed; running it prints `Hint: You probably want 'jj git init'.`
- `jj git clone` prints `Setting the revset alias 'trunk()' to 'main@origin'.` and creates a tracked local bookmark for the remote's default branch. Caveat: this works only when the remote advertises a default branch (its HEAD points at a branch that exists). Otherwise every remote bookmark lands **untracked** and `trunk()` falls back to `main`/`master`/`trunk` name matching — track manually (see Bookmarks).
- `jj git colocation status|enable|disable` inspects or converts colocation after the fact.

In a colocated repo, jj automatically imports git ref changes and exports jj changes on every command — `jj git import` / `jj git export` exist but are no-ops unless something got out of sync (or the repo is non-colocated). Each bookmark's state in the colocated git repo appears as `name@git` (a pseudo-remote). Mixing jj and git commands is allowed, but keep the split: **use jj for anything that writes history**, read-mostly git commands for the rest. Three reasons and one escape hatch: jj keeps git in a detached-HEAD state (run `git switch` first if you must mutate via git), `git rebase` drops jj's change-ID commit header (`git commit --amend` keeps it), and interleaving raises bookmark conflicts / divergent change IDs. Escape hatch: mutating git commands show up in `jj op log` as "import git refs" operations and are undoable with `jj undo` / `jj op restore` like any other op. `git gc` is documented as safe (back up first); jj's own compaction is `jj util gc`.

## Bookmarks

Bookmarks are named pointers to commits, like Git branches. There is **no current/checked-out bookmark**: `@` is what you edit; to base work on a bookmark, run `jj new main`. A local bookmark `foo` has a remote counterpart `foo@origin` (like a remote-tracking branch), updated on every fetch/push of that bookmark. A *tracked* remote bookmark (like a Git "upstream") propagates its changes into the local bookmark on fetch.

`jj bookmark` aliases to `jj b`, and subcommands alias to one letter:

| Task | Command |
|---|---|
| Create | `jj b c <name> -r <rev>` (target defaults to `@`) |
| Create or update | `jj b s <name> -r <rev>` |
| Move (existing only) | `jj b m <name> --to <rev>` (defaults to `@`; refuses backwards/sideways without `-B`) |
| Advance nearest bookmark to a rev | `jj b a <name> --to <rev>` |
| Delete | `jj b d <name>` (propagate to the remote with `jj git push --deleted`) |
| List | `jj b l` (`--all` includes remotes, `-t` tracked only) |
| Track / untrack | `jj b t <name> --remote=origin` / `jj bookmark untrack <name> --remote=origin` |
| Forget locally, no remote deletion | `jj bookmark forget <name>` |

Markers in `jj log` / `jj b l`:

- `main*` — the local bookmark has moved relative to its remote counterpart; you probably want to push.
- `main??` — bookmark conflict (local and remote both moved). `jj st` prints the mitigation hint: resolve with `jj bookmark set <name> -r <rev>` after merging or rebasing the two sides.
- `main@origin` — last-seen position on the remote; read-only, updated by fetch/push.

Tracking is manual by default: only the bookmark tracked at `jj git clone` time and bookmarks you push become tracked. Fetched bookmarks stay untracked until `jj b t <name> --remote=<remote>`; opt in for all future bookmarks with config `remotes.<name>.auto-track-bookmarks = "*"`. Untracked remote bookmarks are immutable by default.

Bookmarks follow their commits: rewriting a commit moves its bookmarks along; abandoning a commit deletes its bookmarks.

## Syncing with a remote

### Fetch

```
jj git fetch                       # origin by default
jj git fetch --remote upstream     # named remote; also --branch, --tracked, --all-remotes
```

Fetch updates each `name@remote` record. For tracked bookmarks, remote changes propagate into the local bookmark; if both sides moved, the local bookmark becomes conflicted (`name??`) and `jj st` prints resolution hints. Commits no longer reachable on the remote are abandoned locally to match. After fetching new `main` work, rebase your stack onto it with `jj rebase -o main` (default selector `-b @`; repeat per outstanding branch).

Default remotes are configurable when origin isn't the right source or target (e.g. fork workflows with `upstream`/`origin`): `[git] fetch = "upstream"` and `git.push = "origin"` — `fetch` accepts a list (`jj config set --repo git.fetch upstream`).

### Push

Default: push **tracking bookmarks and tags** pointing into `remote_bookmarks(remote=<remote>)..@` — i.e. local moves of bookmarks the remote already knows, reachable from `@`. Plain `jj git push` will not create brand-new remote bookmarks; it warns `Refusing to create new remote bookmark <name>@origin` and suggests `jj bookmark track`. Deletions are not pushed by the default revset either — use `--deleted`.

| Option | Effect |
|---|---|
| `-b <name>` | Push one bookmark (glob patterns allowed); a new remote bookmark is auto-tracked |
| `-c <rev>` | Create/track a `push-<change-id>` bookmark per change — the PR/review workflow; re-run after amending to update it |
| `--named <name>=<rev>` | Push `<rev>` under a new bookmark `<name>` (auto-tracked) |
| `--all` / `--tracked` / `--deleted` | Every bookmark+tag / only tracked ones / propagate deletions |
| `--dry-run` | Print `[add to ...]` / `[move forward from X to Y]` lines without pushing |
| `--remote <name>` | Select the remote (not inferred from tracking config like git) |
| `-o <opt>` | Git push options (`-o ci.skip`, `-o merge_request.create ...`), repeated as needed |

Safety checks (like `git push --force-with-lease`, but always on):

1. **Remote moved since last fetch** → refused: `Warning: The following references unexpectedly moved on the remote ... Hint: Try fetching from the remote, then make the bookmark point to where you want it to be, and push again.` Remedy: `jj git fetch`, resolve the resulting conflict (merge the sides with `jj new <local> <name>@origin`, or pick a side with `jj bookmark set`), push again.
2. **Conflicted local bookmark** → refused; resolve with `jj bookmark set`.
3. **Empty descriptions** → refused (`Won't push commit ... since it has no description`) unless `--allow-empty-description`.
4. **Conflicted commits** → refused unless `--allow-conflicts`.

Make `jj git push --dry-run` the habit before a real push; it prints exactly what will move on the remote.

If you push `@` itself and it lands in the immutable set, jj prints `Warning: The working-copy commit became immutable; a new commit has been created on top of it.` and gives you a fresh `@` — expected, not an error.

## When things go wrong

`jj undo` undoes the last operation (repeat to walk further back); `jj redo` steps forward again. Every operation is recorded with its arguments in `jj op log`; inspect the repo as of a past operation with `jj --at-op <id> <command>`. For restructuring, recovering commits, or anything deeper, use the jj-history-surgery skill.

## Quick reference

| Task | Command |
|---|---|
| New repo | `jj git init [dir]` |
| Clone | `jj git clone <src> [dst]` |
| Status of `@` | `jj st` |
| History | `jj log` |
| Diff of `@` / of a change | `jj diff` / `jj diff -r <rev>` |
| Describe current change | `jj describe -m "msg"` |
| Start next change | `jj new -m "msg"` |
| Finish current change in one step | `jj commit -m "msg"` |
| Move to parent/child change | `jj prev` / `jj next` |
| Resume an existing change | `jj edit <change>` |
| Stop tracking a file | add to `.gitignore`, then `jj file untrack <path>` |
| Branch ops | `jj b c/s/m/d/l/t ...` (see Bookmarks) |
| Pull | `jj git fetch [--remote <name>]` |
| Push tracked bookmark moves | `jj git push` |
| Push a change for review | `jj git push -c @` |
| Push specific bookmark | `jj git push -b <name>` |
| Preview a push | `jj git push --dry-run ...` |
| Undo last operation | `jj undo` |

Useful global flags: `-R <path>` (operate on another repo), `--no-pager`, `--at-op <id>` (repo as of a past operation), `--ignore-working-copy` (no snapshot).

## Deeper topics

- Translating Git habits and commands: read `references/git-to-jj.md` in this skill.
- Revset-heavy queries (`jj log -r "..."`): use the jj-revsets skill.
- History restructuring (rebase, split, squash, amend into older commits, recover abandoned commits): use the jj-history-surgery skill.
