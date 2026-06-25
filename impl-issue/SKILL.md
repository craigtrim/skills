---
name: impl-issue
description: Implement a GitHub issue end to end in an isolated /tmp worktree, then open a PR. Use when the user wants to "implement issue N", "build issue N", "do issue N", or hand off a tracked issue for full implementation. Requires a GitHub issue number or URL as input. Not for filing a new issue (use create-issue) or planning (use prd-to-plan).
---

# Implement Issue

**Input (required):** a GitHub issue number or URL — `$ARGUMENTS`.

Take one tracked GitHub issue from "open" to "merged, with a test recipe in the
user's hands": read it, claim it, build it in an isolated worktree, write the smoke
coverage the change actually warrants, supply any migration scripts the change
needs, commit, open a pull request, and hand it to codex — using codex as your
second opinion whenever a question comes up along the way. Codex reviews the PR
and returns a merge verdict; on its say-so you merge and hand the user an
ultra-concise summary of what to test.

If no issue number or URL was supplied, STOP and ask for one. Do not infer an
issue from loose conversation. This skill implements an *existing, written*
issue; if the work is not yet captured as an issue, point the user at
`/create-issue` first.

## Non-negotiable: all work happens in a fresh `/tmp` worktree

This is the one rule that gets skipped, so it comes before everything else. You
do **not** edit, write, build against, or commit a single file in the main
checkout. First you create a brand-new git worktree under `/tmp`, then you do
*all* work inside it.

A branch in the existing checkout is **not** a substitute. "It's a small change"
is **not** an exception. Reusing an existing worktree is **not** allowed. The
isolated tree is the entire point — it cannot be allowed to collide with the
user's working copy or another in-flight task.

**Tripwire — check this before every Edit, Write, build, or commit:** if `$WT`
is not set to a `/tmp/...` path *you created this run* and verified, you have
skipped Step 2. Stop and go do it. Do not rationalize editing in place.

## Process

### 1. Detect the repo and resolve the issue

```bash
gh repo view --json nameWithOwner -q .nameWithOwner   # -> $REPO (owner/name)
gh auth status                                        # must succeed
```

If either fails (no `gh` auth, no git remote, no GitHub remote), STOP and tell
the user the working directory is not backed by a GitHub repo.

Read the full issue, including comments and cross-references:

```bash
gh issue view <N> --repo "$REPO" --comments
```

Build a concrete scope from the issue body and its discussion.

**Claim the issue before anything else.** The moment it resolves — before
grilling, before the worktree — stamp it so anyone watching knows it is taken:
assign yourself, mark it in-progress, and post a claim comment with a timestamp
and a run token.

```bash
gh issue edit <N> --repo "$REPO" --add-assignee @me
gh issue edit <N> --repo "$REPO" --add-label "in-progress" 2>/dev/null \
  || echo "(no 'in-progress' label in this repo — assignee + comment still mark the claim)"
gh issue comment <N> --repo "$REPO" --body \
  "🤖 Claimed by \`impl-issue\` at $(date -u '+%Y-%m-%dT%H:%M:%SZ') (run $$). Implementation in progress; a PR will follow and reference this issue."
```

This is a status marker, not authored prose: post it as written, do **not** route
it through `/tonality`. Post it once per run; if you are resuming a run that
already claimed the issue, do not re-post. If you abandon the run before a PR
exists, post a one-line comment releasing the claim and drop the assignee/label,
so the issue does not sit falsely claimed.

**The issue's self-assessment is not authoritative.** An issue that calls itself
mechanical, "a clean port", "follows the established pattern", or says outright
that it "needs no grill" is making a claim you have to verify, not a fact you can
accept. Issues are written before the work is understood, and the routine-looking
ones routinely hide undefined design forks: a contract with no upstream to mirror,
a behavior the port would silently change, an error path the boilerplate asserts
but the entity cannot reach. Never parrot "needs no grill"; go find out.

So before you build, walk the design forks yourself. For each decision the issue
leaves open or assumes settled, confirm it against the actual code. If a fork
genuinely changes what you build and the issue has not resolved it, resolve it
first; do not guess, and do not punt to the user a question the design can answer.
When the forks are non-trivial, run `/grill-codex` on the issue: pose each fork,
let codex answer, and record the Q&A as comments on the issue. A single focused
question to the user is right only when the answer is genuinely theirs to give
(scope, priorities, product intent); design mechanics are yours to settle.

After a grill, do **both**: keep the comment trail as the audit record, and fold
the resolved decisions into the issue body. The issue body is a living document,
not a frozen brief, and you own what goes into it. Codex is a strong second
opinion, not the final authority; you decide. Do not ask the user whether to
update the body. Update it.

### 2. Create the isolated `/tmp` worktree — and verify it before going further

**This step is mandatory and gated.** Run the block below as one unit. It
resolves every value itself (repo root, default branch, a unique slug), creates
the worktree under `/tmp`, and then *proves* you landed in an isolated tree. You
set only `N` and `SUMMARY`.

```bash
# --- you set these two ---
N=<issue-number>
SUMMARY="<short-kebab-summary>"            # e.g. paginate-variants

# --- everything else resolves itself ---
REPO_ROOT=$(git rev-parse --show-toplevel)
DEFAULT_BRANCH=$(git -C "$REPO_ROOT" symbolic-ref --short refs/remotes/origin/HEAD | sed 's@^origin/@@')
git -C "$REPO_ROOT" fetch origin

BASE=$(basename "$REPO_ROOT")
SLUG=$(printf 'issue-%s-%s' "$N" "$SUMMARY" | tr '[:upper:]' '[:lower:]' | tr -c 'a-z0-9' '-' | sed 's/-\{2,\}/-/g; s/^-//; s/-$//')
WT="/tmp/$BASE-$SLUG"

# never collide with an existing branch or an in-progress worktree — bump the slug
i=1
while git -C "$REPO_ROOT" show-ref --verify --quiet "refs/heads/$SLUG" || [ -e "$WT" ]; do
  i=$((i+1)); SLUG="$SLUG-$i"; WT="/tmp/$BASE-$SLUG"
done

git -C "$REPO_ROOT" worktree add -b "$SLUG" "$WT" "origin/$DEFAULT_BRANCH"

# --- gate: prove isolation, or ABORT and do not edit anything ---
case "$WT" in /tmp/*) ;; *) echo "ABORT: worktree is not under /tmp"; exit 1;; esac
TOP=$(git -C "$WT" rev-parse --show-toplevel)
[ "$TOP" = "$WT" ]        || { echo "ABORT: $WT is not a worktree root"; exit 1; }
[ "$TOP" != "$REPO_ROOT" ] || { echo "ABORT: landed in the main checkout, not an isolated worktree"; exit 1; }
cd "$WT"
echo "WORKTREE READY: $WT (branch $SLUG, base $DEFAULT_BRANCH)"
```

**Do not proceed unless the last line is `WORKTREE READY: /tmp/...`.** If you see
`ABORT` (or no `WORKTREE READY` line), fix the worktree before touching a single
file — never fall back to editing in the main checkout.

From here on, **every path you read, edit, write, build, or commit lives under
`$WT`**; the original checkout at `$REPO_ROOT` stays untouched. Keep `$WT` (and
`$REPO_ROOT`, `$DEFAULT_BRANCH`, `$SLUG`) in mind for the rest of the run — later
steps reference them.

**Never reuse, touch, or clean up another worktree that is already in progress.**
Other `/tmp/$BASE-*` trees may belong to other in-flight tasks; leave them alone.
The slug-bump loop above already guarantees you get a fresh one of your own.

### 3. Implement the issue

Every file you read, create, or edit lives under `$WT`. If you catch yourself
about to open a path under `$REPO_ROOT`, that is the tripwire — go back to Step 2.

Build exactly what the issue asks for, matching the surrounding code's idiom,
naming, and structure. Keep the changeset surgical: touch only the units the
issue actually affects, never a blanket sweep across peers. Run the repo's own
build / lint / typecheck as you go.

**Leave a traceability comment.** Every file you create or modify should carry a
short comment linking back to the underlying issue(s) — the full issue URL or the
`<owner/repo>#<N>` form, whichever the surrounding code already uses — so anyone
reading the line later can trace it to the decision that produced it. Place it
where it reads naturally (a new file's header, or beside the changed block in an
existing file), match the surrounding comment idiom, and keep it to one line. When
the work implements more than one issue, reference each. Do not repeat the same
link across every hunk — one anchor per file is enough; the goal is traceability,
not comment noise.

### 4. Write smoke coverage the change warrants

**If a `smoke-test` skill is available, that skill is the canonical standard;
defer to it and treat the rules below as the portable summary.** (One such skill
covers the inventory-manager Playwright and Lambda suites in depth, with full
detail in its `REFERENCE.md`.) Invoke it when writing or reviewing the smoke so
its suite-specific rules apply.

If the repo has a smoke suite and the change touches a critical or user-facing
path, add or extend a smoke test for it. A smoke test is a narrow, deterministic,
diagnostic proof that one critical path works through its real integration
points. When it fails, the failure should point at a real product, seed,
deployment, or environment problem, never at shared state, a wrong sleep, an
accidental first-row click, or a leaked fixture.

Make the coverage *relevant* to what you changed, and hold it to these rules:

- **Classify the test, pick exactly one class.** Shell/render, read-only seeded
  contract, mutating workflow, destructive/cleanup, or external/auth. Fixture
  strategy and assertion style follow from the class; do not mix classes in one
  scenario. Keep auth/external smokes isolated so an external dependency's
  throttling can't make unrelated product smoke look broken.
- **Own your fixtures.** For mutating tests, generate unique names/keys with a
  recognizable smoke prefix, store the returned IDs, assert against those IDs
  (not incidental row order), and delete what you create. Use fixed seed data
  only when the test is read-only or the seed is reset first.
- **Add an explicit seed-prerequisite check** with a crisp message that names
  the missing seed/migration. Never let a deep downstream `404` be the first
  signal.
- **Wait on domain events, not sleeps.** Wait on the exact save request/response,
  read back through the canonical endpoint, or wait for the committed value to
  appear. Avoid arbitrary timeouts; if forced, explain why and keep it local.
- **Assert strictly where the test owns the behavior.** Exact status code, exact
  entity ID/route/name, exact persisted field value, exact activity context.
  Justify any `first()`/`last()`/`nth()`, broad text search, bare visibility
  check, `>=` count in a mutation, or multiple accepted status codes for one
  happy path.
- **Cover cleanup in the sweep.** Cleanup is best-effort and idempotent but must
  not mask unexpected `401/403/409/500`, and must not be the only proof delete
  behavior works. Any new fixture prefix must join the sweep allow-list — never
  broad delete-all.

Reject: mutating fixed seed without resetting it; passing because a prior run
left the row complete; asserting transient UI state instead of persisted domain
state; sleep-and-hope; clicking the first row when the test created a specific
one; using cleanup tolerance to mask product behavior; failures that are
ambiguous between missing seed, stale deploy, cleanup leak, and regression.

If the repo has no smoke suite, or the change is purely internal with no
user-facing path, say so in the PR rather than inventing a hollow test.

### 5. Provide migration scripts when the change needs them

Not every issue needs one. If the change alters the database schema or requires
a data backfill, include the migration script in the PR, following the repo's
existing migration convention and location. State in the PR description how the
migration is applied and whether it must run before or after deploy. If no
migration is needed, say so explicitly in the PR so the reviewer isn't left
guessing.

### 6. Run all prose through /tonality

**Every line of prose you write for the PR — title, body, and any test comments
or doc text — must go through the `/tonality` skill.** This applies to all
human-readable prose; it does not apply to fenced code blocks or verbatim quotes
from outside sources. After `/tonality` returns, scan for `—`, `–`, `--`, and
three-beat rhythmic stacks as a final check before submitting.

### 7. Commit via the /git-commit skill

**Commit through the `/git-commit` skill — never hand-craft `git commit`.** It
enforces one-line conventional messages, no `Co-Authored-By` / Claude
attribution, and no push. Make atomic commits (one logical change each) inside
the worktree.

`/git-commit` operates on the current directory, so your working directory must
be `$WT` (Step 2 already `cd`'d you there). Before committing, confirm it:
`git rev-parse --show-toplevel` must print `$WT`, not `$REPO_ROOT`. If it prints
the main checkout, you are about to commit in the wrong tree — stop and `cd "$WT"`.

### 8. Open the PR

Push the branch and open a pull request against the default branch:

```bash
git -C "$WT" push -u origin "$SLUG"
gh pr create --repo "$REPO" --base <default-branch> --head "$SLUG" \
  --title "<title>" \
  --body "$(cat <<'EOF'
<body>
EOF
)"
```

Link the issue so it auto-closes on merge: include `Closes #<N>` in the body.

PR body must cover: what changed and why (tied to the issue), the smoke coverage
added (or why none), migration script details (or that none is needed), and any
deploy/seed prerequisite that must happen before the change can be verified in a
deployed environment. Keep it prose, run through `/tonality`, no Claude
attribution.

Print the PR URL when done.

### 9. Have codex review the PR and post comments

Once the PR is open, hand it to the `codex` CLI for a second-opinion review and
let codex post its findings as comments on the PR. Confirm codex is callable
first (`command -v codex`); if it is absent, say so in your summary and skip this
step rather than failing the run.

Invoke codex the way `grill-codex` does — the parts that matter:

```bash
codex exec --sandbox danger-full-access --skip-git-repo-check --color never \
  -C "$WT" \
  "<the review brief below, with the PR number and repo inlined>" </dev/null
```

- **Long Bash `timeout` (e.g. 600000) is mandatory.** On the default ~120s
  timeout the harness detaches codex into the background and you fall into
  async-poll. A long timeout makes the call block and return inline.
- **`</dev/null` is mandatory.** `codex exec` reads stdin even when the prompt is
  an argument; an open stdin hangs the run forever.

The review brief you pass to codex must set the bar explicitly. codex is a senior
reviewer, **not a linter**:

- **Surface only issues that matter.** Correctness bugs, security holes,
  data-loss or data-corruption risks, broken or unmet contracts, race
  conditions, wrong or missing migrations, RLS/tenant-isolation gaps, real
  regressions, smoke tests that would pass on a broken build. Each comment must
  name a concrete consequence ("this drops rows when X" / "this leaks across
  tenants when Y"), not a preference.
- **Do not nitpick.** No style, formatting, import ordering, naming bikeshedding,
  "consider renaming", "you could extract a helper", or any change whose only
  justification is taste. If a thing wouldn't survive the question "what breaks
  if we ship it as-is?", codex should not raise it.
- **Silence is a valid result.** If nothing of consequence is wrong, codex posts
  one short approving comment and stops. An empty review beats a manufactured
  one. Tell codex plainly: do not invent findings to look thorough.
- **Post the comments itself.** codex has the sandbox and `gh` to read the diff
  (`gh pr diff <num> --repo "$REPO"`) and post (inline review comments where a
  finding is line-specific, otherwise a single PR-level comment). Pass it the PR
  number and `$REPO` so it does not have to go hunting.

This review is codex's voice, not yours — leave its wording as written; do not run
its comments through `/tonality`. Note in your final summary that codex reviewed
the PR (and whether it raised anything) so the user knows it ran.

**End the brief by demanding a merge verdict.** codex's final line must be an
explicit `MERGE` or `DO NOT MERGE`, with one sentence of reason. This verdict —
not your own judgment — decides Step 10. If codex is absent (the `command -v
codex` check failed) there is no verdict: skip the merge, leave the PR open, and
tell the user it is awaiting their review.

### 10. Merge on codex's say-so, then hand the user a test recipe

**codex's verdict governs.** If codex said `MERGE` and CI (if any) is green,
merge the PR; if it said `DO NOT MERGE`, leave the PR open, summarize what codex
flagged, and stop — do not merge over an objection, and do not substitute your
own opinion for the verdict.

When you do merge, use the squash merge and delete the branch:

```bash
gh pr merge <num> --repo "$REPO" --squash --delete-branch
```

After a successful merge — and only then — give the user an **ultra-concise
summary of what changed, framed as something they can test**: the few concrete
things they should click, run, or check to confirm the change works in the
deployed/running app. Keep it to a tight bullet list, name real screens/routes/
commands, and skip the internals. This summary is for the user, so it goes
through `/tonality`. If the PR was left open (no merge), say so plainly instead
and do not write a test recipe.

## Guardrails

- **Requires an issue.** No issue number/URL in `$ARGUMENTS` → stop and ask.
- **Claim before building.** Right after the issue resolves, assign yourself,
  add an `in-progress` label (best effort, tolerate its absence), and post a
  timestamped claim comment with a run token — so the issue reads as taken before
  any grilling or worktree work. Post it as-is, not via `/tonality`. Release the
  claim (comment + drop assignee/label) if you abandon the run before a PR exists.
- **The issue's self-assessment is not authoritative.** "Mechanical", "needs no
  grill", "follows the pattern" are claims to verify, not facts to accept. Walk
  the design forks before building and resolve the ones that change what you
  build (via `/grill-codex` when non-trivial). Then keep the comment trail and
  fold the decisions into the living issue body. You decide what lands in the
  body, not codex; do not ask the user whether to update it.
- **Worktree is mandatory, and a branch is not enough.** You MUST create a fresh
  `/tmp/<repo>-<slug>` worktree (Step 2) and verify `WORKTREE READY` before you
  edit, write, build, or commit anything. Branching in place in the main checkout
  is the exact failure this skill exists to prevent — there is no "small change"
  exception. Never reuse or clean up another in-progress worktree; never create
  one under `.claude/` or inside the repo. If `$WT` is unset or not a verified
  `/tmp/...` path, you have not earned the right to touch a file yet.
- **Surgical changeset.** Touch only what the issue affects.
- **Traceability comment.** Every created or modified file carries a one-line
  comment linking back to the underlying issue(s) (issue URL or `<owner/repo>#<N>`),
  one anchor per file, in the surrounding comment idiom.
- **Never skip `/tonality`** on PR prose, even for a small change.
- **Never bypass `/git-commit`** with a raw `git commit`.
- **No Claude attribution** in commits, PR title, or PR body.
- **codex reviews the open PR** and posts its own comments — but only issues that
  matter. It is a senior reviewer, not a linter; no style/naming/taste nits, and
  silence is allowed when nothing is wrong. If `codex` is not installed, skip the
  review and say so.
- **codex's merge verdict governs.** Its review ends in an explicit `MERGE` /
  `DO NOT MERGE`. Merge (squash, delete branch) only on `MERGE`; on `DO NOT
  MERGE`, or when codex is absent, leave the PR open and say why. Never override
  the verdict with your own opinion.
- **After a merge, hand back a test recipe.** An ultra-concise, user-facing bullet
  list of the concrete things to click/run/check to confirm the change — real
  screens/routes/commands, no internals — run through `/tonality`. No merge → no
  recipe; say the PR is awaiting review instead.
- One invocation implements one issue, opens one PR, and — if codex says merge —
  merges it and hands back a test recipe.
