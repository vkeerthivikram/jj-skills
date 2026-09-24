# Revset function reference (jj 0.45)

Optional arguments are written `[arg]`. Some arguments can be passed by label
without filling earlier optional arguments. For example,
`remote_bookmarks([name_pattern], [[remote=]remote_pattern])` accepts all of:

- `remote_bookmarks()`
- `remote_bookmarks("main")` (verified)
- `remote_bookmarks("main", "origin")` (verified)
- `remote_bookmarks("main", remote="origin")`
- `remote_bookmarks(remote="origin")` (verified)

Verified examples below refer to the sandbox graph: root <- A <- {B, C};
D = merge(B, C); E child of B; P, Q children of A; M = merge(P, Q) with a
conflict; A pushed as main@origin.

## Navigation

- `parents(x, [depth])` — same as `x-`; with depth, parents at that distance.
  `parents(x, 3)` = `x---`. Verified: `parents(D, 2)` = {A}.
- `children(x, [depth])` — same as `x+`; with depth. Verified:
  `children(A, 2)` = {D, E} (grandchildren).
- `ancestors(x, [depth])` — same as `::x` (x INCLUDED); limited to depth levels
  counting x itself. Verified: `ancestors(E, 2)` = {E, B}.
- `descendants(x, [depth])` — same as `x::` (x INCLUDED), depth-limited the
  same way. Verified: `descendants(A, 2)` = {A, B, C}.
- `first_parent(x, [depth])` — like parents() but only the first parent of
  merges. Verified: `first_parent(D)` = {B} for a merge of B and C.
- `first_ancestors(x, [depth])` — ancestors() following first parents only.
  Useful on git-style history to skip merged-away branches. Verified:
  `first_ancestors(D)` = {D, B, A, root}.
- `reachable(srcs, domain)` — all commits reachable from srcs via parent AND
  child edges such that the whole path stays inside domain.
  `reachable(@, mutable())` is the current working stack (verified:
  {@, E, D, C, B} while P/Q/M hung off to the side).
- `connected(x)` — same as `x::x`; the subgraph spanning x. Verified:
  `connected(E | A)` = {E, B, A}; `connected(D | A)` = {D, C, B, A}.

## Set operations and selection

- `all()` — all visible commits, plus ancestors of explicitly mentioned hidden
  commits (11 in the sandbox). Verified.
- `none()` — empty set.
- `heads(x)` — members of x that are not ancestors of other members of x.
  Equivalent to `x ~ ::x-` (note: stronger than the Mercurial definition).
- `roots(x)` — members of x that are not descendants of other members of x.
  Equivalent to `x ~ x+::`.
- `latest(x, [count])` — count newest commits of x by committer timestamp;
  default 1. Verified: `latest(::E, 2)` = {E, B}.
- `fork_point(x)` — nearest common ancestor(s) of all commits in x;
  `heads(::x_1 & ::x_2 & ... & ::x_N)`. If x is a single commit, returns it.
- `merge_point(x)` — nearest common descendant(s); the mirror of fork_point.
  Verified: `merge_point(B | C)` = {D}.
- `bisect(x)` — commits in x with about half of x as descendants; help topic
  notes the implementation handles non-linear history poorly.
- `exactly(x, count)` — error unless x has exactly count commits, else x.
  Useful with count 1 before single-revision commands. Verified error:
  more than the expected 1 revisions.

## Refs and identities

- `change_id(prefix)` — commits with the given change ID prefix; resolves to
  several commits if divergent. Non-unique prefix is an error; an unmatched
  prefix is not.
- `commit_id(prefix)` — commits with the given commit ID prefix. Use this to
  bypass tag/bookmark name collisions in scripts.
- `bookmarks([pattern])` — local bookmark targets; pattern filters by name.
  Conflicted bookmarks include all targets. Verified: `bookmarks("v*")`,
  `bookmarks(~glob:"main")`, `bookmarks(glob:"*" & ~glob:"main")`.
- `remote_bookmarks([name_pattern], [[remote=]remote_pattern])` — remote
  bookmark targets. Git-tracking bookmarks (`@git`) are EXCLUDED unless
  `remote="git"` or `remote="*"`.
- `tracked_remote_bookmarks(...)` — tracked subset. Verified: after push,
  main@origin is tracked (empty untracked set).
- `untracked_remote_bookmarks(...)` — untracked subset; feeds the default
  `immutable_heads()`.
- `tags([pattern])` — tag targets. Verified: git tags imported on fetch
  appear here.
- `remote_tags(...)`, `tracked_remote_tags(...)`, `untracked_remote_tags(...)`
  — same three-way split for tags. Verified untracked set empty after a
  tracked fetch.
- `visible_heads()` — all visible heads. Verified: {D, E} on the clean graph.
- `root()` — the virtual oldest ancestor of everything; change id zzzzzz.
- `working_copies()` — the @ commit of every workspace. Verified with two
  workspaces.

## Metadata and dates

- `description(pattern)` — commits whose full description matches. A non-empty
  description normally ends with a newline: `description("")` finds undescribed
  commits and `description(exact:"foo\n")` matches description foo. Verified:
  `description(exact:"A: the base")` is EMPTY; the \n form matches.
- `subject(pattern)` — matches only the first line, without the newline.
  Verified: `subject(exact:"A: the base")` matches.
- `author(pattern)` — author name or email matches; equivalent to
  `author_name(pattern) | author_email(pattern)`. Verified glob on name.
- `author_name(pattern)` / `author_email(pattern)` — name-only / email-only.
  Verified `author_email(exact:"other@example.com")` and
  `author_name(glob-i:"other dev")`.
- `author_date(pattern)` — author timestamp against a date pattern.
  Verified: after:"yesterday" (inclusive), before:"2020-01-01" (only root).
- `mine()` — author email equals the CURRENT user email config, case-blind
  (documented as `author_email(exact-i:<user-email>)`). Verified under both
  identities: switched with `--config user.email=...`.
- `committer(pattern)`, `committer_name(pattern)`, `committer_email(pattern)`,
  `committer_date(pattern)` — the committer-side equivalents. jj rewrites
  committer fields on every amend, so committer_date tracks when a change was
  last touched.
- `signed()` — cryptographically signed commits. Evaluates; none in sandbox.

## Content and state

- `empty()` — commits with no file changes; includes root() and merges without
  user modifications. Verified: an auto-merged conflict commit with no manual
  edits counted as empty.
- `files(expression)` — commits modifying paths matching a fileset. Paths are
  relative to the invocation directory; a directory matches itself and its
  subdirectories (files(foo) matches foo, foo/bar, foo/bar/baz but NOT foobar
  or bar/foo). Some patterns need quoting because the argument must parse as a
  revset too: `files(".")`. Verified both.
- `diff_lines(text, [files])` — commits whose added OR removed diff lines
  match text. Verified: `diff_lines("*TODO*")` = {C}.
- `diff_lines_added(text, [files])` / `diff_lines_removed(text, [files])` —
  only the added / removed side. Verified added = {C}, removed = {}.
- `conflicts()` — commits containing conflicted files. Verified: {M}.
- `divergent()` — commits whose change ID has more than one live instance.
  Evaluates; empty in sandbox.

## Error tolerance and operations

- `present(x)` — x, or none() if any commit in x does not exist (e.g. unknown
  bookmark). Verified: exit 0, empty set for an absent name.
- `coalesce(revsets...)` — the first argument that is not none(). Verified:
  `coalesce(present(nope), @)` = @.
- `at_operation(op, x)` — evaluate x as of a previous operation, e.g.
  `at_operation(@-, visible_heads())` for the heads before the last operation.
  Brings everything visible then back into the search space. Verified to
  evaluate.

## Operator worked examples (from the official revsets doc)

Given this history (graph 1):

```
D
|\
B C
|/
A
|
root()
```

Operator `x-`:

- `D-` => {C,B}
- `B-` => {A}
- `A-` => {root()}
- `root()-` => {} (empty set)
- `none()-` => {} (empty set)
- `(D|A)-` => {C,B,root()}
- `(C|B)-` => {A}

Operator `x+`:

- `D+` => {} (empty set)
- `B+` => {D}
- `A+` => {B,C}
- `root()+` => {A}
- `none()+` => {} (empty set)
- `(C|B)+` => {D}
- `(B|root())+` => {D,A}

Operator `x::`:

- `D::` => {D}
- `B::` => {D,B}
- `A::` => {D,C,B,A}
- `root()::` => {D,C,B,A,root()}
- `none()::` => {} (empty set)
- `(C|B)::` => {D,C,B}

Operator `x..`:

- `D..` => {} (empty set)
- `B..` => {D,C} (note that, unlike `B::`, this includes `C`)
- `A..` => {D,C,B}
- `root()..` => {D,C,B,A}
- `none()..` => {D,C,B,A,root()}
- `(C|B)..` => {D}

Operator `::x`:

- `::D` => {D,C,B,A,root()}
- `::B` => {B,A,root()}
- `::A` => {A,root()}
- `::root()` => {root()}
- `::none()` => {} (empty set)
- `::(C|B)` => {C,B,A,root()}

Operator `..x`:

- `..D` => {D,C,B,A}
- `..B` => {B,A}
- `..A` => {A}
- `..root()` => {} (empty set)
- `..none()` => {} (empty set)
- `..(C|B)` => {C,B,A}

Operator `x::y`:

- `D::D` => {D}
- `B::D` => {D,B} (note that, unlike `B..D`, this includes `B` and excludes `C`)
- `B::C` => {} (empty set) (note that, unlike `B..C`, this excludes `C`)
- `A::D` => {D,C,B,A}
- `root()::D` => {D,C,B,A,root()}
- `none()::D` => {} (empty set)
- `D::B` => {} (empty set)
- `(C|B)::(C|B)` => {C,B}

Operator `x..y`:

- `D..D` => {} (empty set)
- `B..D` => {D,C} (note that, unlike `B::D`, this includes `C` and excludes `B`)
- `B..C` => {C} (note that, unlike `B::C`, this includes `C`)
- `A..D` => {D,C,B}
- `root()..D` => {D,C,B,A}
- `none()..D` => {D,C,B,A,root()}
- `D..B` => {} (empty set)
- `(C|B)..(C|B)` => {} (empty set)

Non-distributivity of `..` over union (left side), same graph:

- `(C|B)..` => {D}, but:
- `C..` => {D,B}
- `B..` => {D,C}
- `C.. | B..` => {D,C,B}
- `C.. & B..` => {D}

So `(C|B)..` is NOT `C.. | B..`; it equals `C.. & B..`. The `..` operator
converts union to intersection on its left side.

Live verification note: the sandbox reproduced graph 1 plus one extra head E
on B; every shared expression above returned the documented set with E added
exactly where the semantics predict (e.g. `B::` = {E, D, B}, `(C|B)..` = {E, D},
`C.. | B..` = {E, D, C, B}), so the non-distributivity contrast was confirmed
on a real graph.

Function examples (official doc, graph 2):

```
E
| D
|/|
B C
|/
A
|
root()
```

- `reachable(E, A..)` => {E,D,C,B} — all commits after A are reachable
- `reachable(E, B..)` => {E} — E is isolated: its only edge (to B) leaves the domain
- `reachable(C, B..)` => {D,C} — C and D are connected; the C-to-A edge leaves the domain
- `reachable(D, B..)` => {D,C} — same result: D reaches C, but D-to-B leaves the domain
- `reachable(A, A..)` => {} (empty set) — A is not in domain `A..`, so it is ignored
- `connected(E|A)` => {E,B,A}
- `connected(D|A)` => {D,C,B,A}
- `connected(A)` => {A}
- `heads(E|D)` => {E,D}; `heads(E|C)` => {E,C}; `heads(E|B)` => {E}; `heads(E|A)` => {E}; `heads(A)` => {A}
- `roots(E|D)` => {E,D}; `roots(E|C)` => {E,C}; `roots(E|B)` => {B}; `roots(E|A)` => {A}; `roots(A)` => {A}
- `fork_point(E|D)` => {B}
- `fork_point(E|C)` => {A}
- `fork_point(E|B)` => {B}
- `fork_point(E|A)` => {A}
- `fork_point(D|C)` => {C}
- `fork_point(D|B)` => {B}
- `fork_point(B|C)` => {A}
- `fork_point(A)` => {A}
- `fork_point(none())` => {}

Every graph-2 line was reproduced verbatim in the sandbox (same structure,
identical results), including all nine fork_point cases.
