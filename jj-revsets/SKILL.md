---
name: jj-revsets
description: The Jujutsu (jj) revset language - symbols (@, change/commit ID prefixes, bookmark@remote), operators (parent/child +/-, :: and .. ranges, | & ~), functions (files(), author(), description(), mine(), heads(), latest(), mutable(), trunk(), conflicts()...), string and date patterns, and revset-aliases configuration. Use this skill whenever a revset needs to be composed or debugged - jj log -r / jj rebase -r / jj squash --from filters, 'show commits not yet pushed', 'find commits by message, author, or path', customizing the default jj log set, or diagnosing 'revset resolved to no/multiple revisions' errors. Also covers fileset expressions for path arguments. Everyday jj usage is jj-workflow; history rewriting is jj-history-surgery.
---

# Jujutsu revsets (verified against jj 0.45.1)

A revset is an expression that evaluates to a **set of commits**. Nearly every jj
command accepts one. Some commands (edit, describe, new, bookmark move --to)
require the set to contain exactly one commit; passing zero or multiple commits
is an error. All semantics below were live-verified on jj 0.45.1 unless noted.

## 1. Commands that accept revsets

- Any number of revisions: `jj log -r X`, `jj evolog -r X`, `jj git push -r X`
- Exactly one revision: `jj show X`, `jj describe X -m ...`, `jj edit X`,
  `jj new X [Y ...]` (multiple args = merge), `jj duplicate X`,
  `jj split -r X [filesets]`, `jj restore X` (positional is the target)
- Filter flags: `jj diff -r X [filesets]`, `jj squash --from X [--into Y]`,
  `jj rebase (-r | -s | -b) X -d Y`, `jj restore --from Y [--into X]`,
  `jj absorb --from X [--into Y]`, `jj file show/list/annotate -r X`
- Bookmarks: `jj bookmark create NAME -r X`, `jj bookmark move NAME --from X --to Y`
  (`jj git push -b` takes bookmark NAMES, not revsets)
- Config knobs that take revsets: `revsets.log`, `revsets.short-prefixes`,
  `revsets.op-diff-changes-in`, `revsets.log-graph-prioritize`

Prefer `-r "@-"` style double-quoted revsets in scripts so shell metacharacters
never leak into the expression. Output formatting is a separate concern:
customize columns with `-T` templates; the language reference is
`jj help -k templates` (running `jj log -T` with no value errors and, as a
side effect, lists the built-in template aliases).

## 2. Symbols

| Symbol | Meaning |
|---|---|
| `@` | Working-copy commit of the current workspace |
| `<workspace>@` | Working-copy commit of another workspace (verified: `default@`, `sbox3-ws2@`) |
| `<name>@<remote>` | Remote-tracking bookmark/tag, e.g. `main@origin` (verified) |
| full/prefix commit ID | Single commit; prefix must be unique among visible commits |
| full/prefix change ID | Visible commit with that change ID; prefix must be unique and non-divergent |
| `<change-id>/<offset>` | Hidden or divergent instance of a change ID; offset 0 is the most recent (verified for a hidden commit) |

Resolution priority for a bare symbol: **tag > bookmark > commit/change ID**
(verified: with tag `v1` on A and bookmark `v1` on E, symbol `v1` resolves to A;
`bookmarks(v1)` gives E). Force an interpretation with `change_id(x)` or
`commit_id(x)` in scripts.

Quoting: quotes make jj treat the text as a symbol instead of an expression.
`jj log -r "\"x-\""` resolves the bookmark named `x-`; unquoted `x-` parses as
parents of symbol `x` and fails with `Revision x does not exist` plus a hint to
quote. Creating a trailing-dash bookmark also needs the inner quotes:
`jj bookmark create "\"x-\"" -r @`. Mid-name dashes (`feat-x`) need no quoting.

## 3. Operators (binding order, strongest first)

1. `f(x)` function call
2. `x-` parents, `x+` children (may be empty; `none()-` is `{}`)
3. `p:x` string/date pattern or pattern alias
4. Ranges: `x::` `x..` `::x` `..x` `x::y` `x..y` `::` `..`
5. `~x` negation (complement within the search space)
6. `x & y` intersection, `x ~ y` difference
7. `x | y` union (weakest)

Same-precedence infix operators parse left to right: `x ~ y & z` is `(x ~ y) & z`.
Parenthesize everything you do not want reassociated by this table.

### Range semantics

| Expr | Set | Equivalent |
|---|---|---|
| `x::` | descendants of x, including x | `x::visible_heads()` (no hidden mentioned) |
| `x..` | commits NOT ancestors of x | `~::x` |
| `::x` | ancestors of x, including x | `root()::x` |
| `..x` | ancestors of x, excluding the root commit | `::x ~ root()` |
| `x::y` | ancestry path: descendants of x that are ancestors of y | `x:: & ::y` (git `--ancestry-path`) |
| `x..y` | ancestors of y that are not ancestors of x | `::y ~ ::x` (same as git `x..y`) |
| `::` | all visible commits | `all()` |
| `..` | all visible commits except the root | `~root()` |

`x` and `y` may be arbitrary revsets. `x..y` does NOT require x and y to be
related; it simply subtracts `::x` from `::y`.

### Gotcha 1: `x::y` vs `x..y`

On graph `root() <- A <- {B, C}` (B and C siblings), verified live:

- `B::C` = `{}` (empty). C is not a descendant of B.
- `B..C` = `{C}`. Git-style subtraction keeps C.
- `B::D` = `{D, B}` (includes start B, excludes sibling C); `B..D` = `{D, C}`.

### Gotcha 2: `..` does not distribute over `|` on the left

`(C | B)..` = `C.. & B..`, NOT `C.. | B..`. Verified on the sandbox graph with
heads D and E: `(C|B)..` = `{D, E}` while `C.. | B..` = `{B, C, D, E}`.
Read `(A|B)..` as: not an ancestor of A AND not an ancestor of B.

### Gotcha 3: precedence surprises

`x | y & z` is `x | (y & z)`. A range swallows the tightest adjacent unions on
its left: write `(A|B)..`, never `A|B..`. Postfix `-`/`+` bind tighter than
everything except calls: `f(x)-` is `(f(x))-`, and a symbol ending in `-` must
be quoted (section 2).

Full worked-example tables for every operator: references/functions.md.

## 4. The functions you will actually use

| Function | One-line semantics |
|---|---|
| `x-` / `x+` | parents / children of x (may be empty) |
| `parents(x, [n])` / `children(x, [n])` | same, at depth n (`parents(x,3)` = `x---`) |
| `ancestors(x, [n])` / `descendants(x, [n])` | `::x` / `x::`, limited to n levels **including x** (verified: `ancestors(@, 2)` = @ plus its parents) |
| `first_parent(x)` / `first_ancestors(x)` | follow only first parents (verified on a merge) |
| `reachable(srcs, domain)` | commits reachable from srcs without leaving domain; `reachable(@, mutable())` = your current stack (verified) |
| `connected(x)` | `x::x`; fills in the graph between members of x |
| `heads(x)` / `roots(x)` | members of x that are not ancestors / descendants of other members |
| `latest(x, [n])` | n newest in x by committer timestamp (default 1) |
| `fork_point(x)` | `heads(::x1 & ::x2 & ...)`; nearest common ancestor(s) |
| `trunk()` | head of main/master/trunk on origin or upstream; **falls back to `root()`** if none exists (verified) |
| `mutable()` / `immutable()` | `~::(immutable_heads() | root())` and its complement |
| `description(p)` / `subject(p)` | match full description / first line (see newline gotcha, section 8) |
| `author(p)` / `mine()` | author name-or-email match / author email = current user email |
| `files(fileset)` | commits modifying paths matching a fileset (directory prefix matches recursively; verified `files("src")` does NOT match `srcfoo.txt`) |
| `empty()` | no file changes; includes root() and merges without user modifications (verified: a conflicted auto-merge counted as empty) |
| `conflicts()` | commits with conflicted files (verified) |
| `present(x)` / `coalesce(a, b, ...)` | tolerate missing refs; first argument that is non-empty |
| `exactly(x, n)` | error unless x has exactly n commits; returns x |

Complete reference with every function and worked examples:
references/functions.md.

## 5. String and date patterns

String-matching functions take a pattern; the default kind is **glob**:

- `exact:"s"` — full-string equality
- `glob:"p*"` — globset wildcards (`*` does not cross `/`)
- `regex:"^p"` — substring-matching regex
- `substring:"s"` — contains
- append `-i` for case-insensitive: `substring-i:"FEATURE C"` (verified)

Bare `"B*"` is a glob; verified that `*` DOES match the trailing newline of a
description. Patterns compose: `bookmarks(~glob:"main")`,
`bookmarks(glob:"*" & ~glob:"main")` (both verified).

Date functions (`author_date`, `committer_date`) take:

- `after:"2026-09-24"`, `after:"2 days ago"`, `before:"yesterday 5pm"`
- forms accepted: `2024-02-01`, `2024-02-01T12:00:00-08:00`,
  `2024-02-01 12:00:00`, `2 days ago`, `yesterday 10:30`
- `after:` is inclusive, `before:` is exclusive; both verified

## 6. Built-in aliases and configuration

Built-ins (all verified to evaluate; check with `jj log -r "trunk()"`):

- `trunk()` — default bookmark of default remote, else `main`/`master`/`trunk`
  on `upstream`/`origin` (newest wins), else `root()`
- `builtin_log()` = `present(@) | ancestors(immutable_heads().., 2) | trunk()`
  — default for `revsets.log` (what plain `jj log` shows)
- `immutable_heads()` = `trunk() | tags() | untracked_remote_bookmarks() | untracked_remote_tags()` (alias of `builtin_immutable_heads()`; override THIS one)
- `immutable()` = `::(immutable_heads() | root())`; `mutable()` = `~immutable()`
  — do not redefine these two: the alias does not change actual immutability
- `visible()` = `::visible_heads()`; `hidden()` = `~visible()`, empty unless the
  revset itself mentions hidden revisions

Overriding (config file, double-quoted TOML works):

```toml
[revset-aliases]
"trunk()" = "main@origin"                  # pin trunk; must resolve to ONE commit
"immutable_heads()" = "builtin_immutable_heads() | (trunk().. & ~mine())"
"stack()" = "reachable(@, mutable())"      # custom alias (verified)
"grep:x" = "description(regex:x)"          # pattern alias: use as grep:"TODO"
HEAD = { definition = "@-", doc = "Parent of the working copy" }  # .definition/.doc table form (verified)
```

Alias functions overload by parameter count; a user alias shadows a builtin of
the same name. Custom log default:

```toml
[revsets]
log = "main@origin.."       # built on builtin_log() if you want the default shape
```

Warning (from official docs): when `revsets.short-prefixes` is not set it
defaults to `revsets.log`; changing `log` alone can lengthen the short ID
prefixes needed on the command line.

## 7. Recipes (each verified on jj 0.45.1)

Sandbox shape: root <- A(main@origin) <- {B, C}; D = merge(B, C); E child of B;
P, Q siblings on A; M = merge(P, Q) with a conflict; @ on top of D.

```shell
# Commits not pushed to ANY remote (before any push: everything incl. root)
jj log -r "remote_bookmarks().."

# Commits not on origin specifically (multiple remotes: fork, upstream)
jj log -r "remote_bookmarks(remote=origin).."

# What a plain `jj git push` would send (default push range, verified {@,D,C,B})
jj log -r "remote_bookmarks(remote=origin)..@"

# Your current stack only - not sibling branches (verified: {@,E,D,C,B})
jj log -r "reachable(@, mutable())"

# Commits touching a path since trunk (verified: {B})
jj log -r "trunk().. & files(\"src\")"

# Find by message (substring is usually what you want)
jj log -r "description(substring:\"feature\")"
jj log -r "subject(exact:\"A: the base\")"     # first line, no newline
jj log -r "description(exact:\"A: the base\\n\")"  # full description ends in \n

# Find by author
jj log -r "author(\"Other*\")"
jj log -r "mine()"                              # current user.email
jj log -r "mine() & author_date(after:\"yesterday\")"   # my work today

# Latest N commits (by committer timestamp)
jj log -r "latest(::@, 5)"

# Conflicted commits (verified: {M})
jj log -r "conflicts()"

# Empty, undescribed changes worth cleaning up (verified: {@} only)
jj log -r "mutable() & empty() & description(exact:\"\")"

# Divergent change IDs
jj log -r "divergent()"

# Context around a revision: both directions plus the commit itself (verified)
jj log -r "::$x | $x::"
jj log -r "connected($x | $x- | $x+)"          # x with parents and children

# Merge base / fork point helpers
jj log -r "fork_point($a | $b)"                # nearest common ancestor(s)
jj log -r "merge_point($a | $b)"               # nearest common descendant(s)

# Initial commits of the repo (git root commits)
jj log -r "root()+"

# Decorations only, like git log --simplify-by-decoration
jj log -r "tags() | bookmarks()"

# Added/removed text lines under a path
jj log -r "diff_lines_added(\"*TODO*\", \"src\")"
```

## 8. Debugging zero/multiple revision errors

`Revision X does not exist` / empty result:

- Typo or absent bookmark: wrap speculative refs in `present(x)` or
  `coalesce(present(integration), trunk())`. Verified: `present(main)` with no
  main = empty set, exit 0.
- `description(exact:"foo")` matched nothing: descriptions END WITH `\n`.
  Use `subject(exact:"foo")` or `description(exact:"foo\n")`. Verified both ways.
- Hidden commit: abandoned/forgotten commits are invisible to `all()`; mention
  them by full commit ID or `<change-id>/0` to pull them (and their ancestors)
  back into the search space (verified).
- `files(.)` fails to parse: the argument is parsed as a revset first; quote it
  `files(".")` (verified parse error otherwise).
- Symbol ending in `-` (like `x-`): quote it, or jj parses the parent operator
  and reports unknown symbol `x`.
- No remotes yet: `remote_bookmarks()..` still evaluates (equals `~::none()`),
  returning every visible commit including root (verified).
- Everything looks mutable / trunk is the root commit: no remote bookmarks
  exist, so `trunk()` fell back to `root()`. Pin it via `[revset-aliases]`.

`more than the expected 1 revisions` / multiple targets for a one-commit command:

- Pin with `exactly(x, 1)` (verified error text: more than the expected 1
  revisions), or select a side with `heads()`, `latest(x, 1)`, `fork_point()`.
- Check for tag/bookmark priority collisions (section 2): a tag shadows your
  bookmark name; use `bookmarks(name)` to be explicit.
- Non-unique ID prefix: lengthen the prefix or use `commit_id()`/`change_id()`.

Wrong-but-nonempty results:

- `(A|B)..` is not `A.. | B..` (section 3, Gotcha 2).
- `x::y` silently empty when y is not a descendant of x; you wanted `x..y`.
- `mine()` matches by EMAIL ONLY under the current `user.email` config.

## 9. Filesets (path arguments)

Filesets select FILES; they appear as path arguments to diff, split, restore,
file list, and inside `files()` / `diff_lines()` in revsets.

- Bare `"path"` is a cwd-relative **prefix match**: file OR directory (recursively).
  Verified: `files("src")` matches `src/b.txt`, not `srcfoo.txt`.
- `cwd:` `file:`/`cwd-file:` (exact file), `glob:`/`cwd-glob:` (wildcards, `*`
  does not cross `/`; verified `glob:*.txt` does not match `src/b.txt`),
  `prefix-glob:`, and workspace-anchored `root:`, `root-file:`, `root-glob:`,
  `root-prefix-glob:` (verified `root-file:src/b.txt`). Append `-i` for
  case-insensitive globs.
- Operators mirror revsets: `~x` complement, `x & y`, `x ~ y`, `x | y`
  (verified: `jj diff -r B --stat "~src"`).
- `all()`, `none()`; config aliases under `[fileset-aliases]` with the same
  `.definition`/`.doc` table form.
- Quoting: inner quotes needed around whitespace/metacharacters and for `.`:
  `files(".")`. A bare pattern without operators may omit inner quotes.

Run `jj help -k filesets` for the full topic.

## 10. Related skills

- jj-workflow — everyday jj usage: snapshots, describing, squashing into parents
- jj-history-surgery — rewriting history: rebase, split, absorb, evolve repair

## Verification

Every operator result, function, pattern, recipe, and error message in this
skill was executed against jj 0.45.1 in a sandbox repository on 2026-09-24
(graph: root <- A <- {B, C}; D = merge(B,C); E on B; conflict merge M; remote
origin with main; git tag; hidden abandoned commit; second workspace). Where
the live behavior adds detail beyond the official docs (glob `*` crossing the
description newline, `empty()` including a conflicted auto-merge, and the
Did-you-mean quoting hint on trailing-dash symbols), the live behavior is stated.
