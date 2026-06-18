---
name: create-issue
description: Create a GitHub issue in the current repo, auto-detecting the repo from the git remote and cross-referencing related existing issues. Use when the user wants to file, open, or create a new GitHub issue, log a feature request, or capture work that needs tracking. Not for triage of a reported bug (use triage-issue for that) and not for breaking down a PRD into work items (use prd-to-issues for that).
---

# Create Issue

File a single new GitHub issue in the repository that the current working directory belongs to. Detect the repo, write tight prose, and stitch the new issue into the existing issue graph via cross-references.

This skill is repo-agnostic. It does not assume any particular project layout, language, or label scheme.

## Process

### 1. Detect the GitHub repo

From the current working directory:

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

If it prints `owner/repo`, capture that as `$REPO` and proceed.

If it errors (no `gh` auth, no git remote, no GitHub remote), STOP. Tell the user the working directory is not backed by a GitHub repo and ask whether they want to:

1. Pass an explicit `--repo owner/name` for this one issue, or
2. Skip filing the issue.

Do not guess. Do not silently fall back to a different repo. Do not create in a personal default.

Also verify `gh auth status` succeeds. If it does not, STOP and tell the user.

### 2. Capture the issue

If the user has already described what they want, use that. Otherwise ask ONE focused question to get the scope. Do not grill the user; this is a quick capture, not a PRD or a triage.

Decide the title and body from the user's description (labels are selected in step 4). Title goes under 70 characters. Use sentence case unless the repo's existing issue list shows a different convention.

### 3. Find cross-references

Enumerate existing issues so you can stitch the new one into the issue graph:

```bash
gh issue list --repo "$REPO" --state all --limit 200 \
  --json number,title,labels,state,url
```

For repos with more than 200 issues, also run a targeted search against 2 or 3 keywords pulled from the new issue's topic:

```bash
gh search issues --repo "$REPO" "<keyword>" \
  --json number,title,url --limit 30
```

Identify candidates that fit one or more of these relationship types:

- **Similar to**: substantially overlapping scope or subject matter; the reader of either issue benefits from knowing about the other.
- **Blocks**: this new issue must be resolved before the linked issue can complete.
- **Blocked by**: the linked issue must be resolved before this new issue can start.
- **Triggers**: completing this new issue should kick off the linked issue, or signals that the linked issue is now actionable.
- **Triggered by**: a state change in the linked issue is what produced the need for this new issue.

You may introduce a new relationship label (for example "Supersedes", "Extends", "Refines", "Related research in") if the semantics are clearly different from the five above and the link would otherwise be misleading. Use sparingly.

Cross-reference rules:

- Only link an issue when the relationship is real. "Both touch the same module" is not a relationship; it is a coincidence.
- Cap total cross-refs at 5. If more than 5 candidates exist, prefer the tightest semantic match per relationship type.
- An issue with zero good cross-refs gets no cross-reference section at all. Do not invent links to look thorough. A clean island is better than a noisy graph.
- The goal is a cyclic graph of linked context, not a citation count.

### 4. Select labels (mandatory: MIN 1, MAX 3)

**Every issue MUST carry at least one label and no more than three.** Choose labels ONLY from the repo's existing label set. Never create a new label from this skill.

Enumerate the available labels first:

```bash
gh label list --repo "$REPO" --limit 200
```

Pick the 1 to 3 existing labels that best fit the issue's type and subject (for example a kind label like `bug`/`enhancement` plus an area label). Match the label's intended meaning, not just a keyword in its name. If nothing fits cleanly, fall back to the closest existing general-purpose label rather than inventing one. Do not exceed three; if more than three seem to apply, keep only the tightest matches.

### 5. Apply the voice rules via /tonality

**Invoking the `/tonality` skill on the drafted title and body is mandatory.** Do not file the issue without running prose through `/tonality` first. This applies to every new line of prose written for the issue title and body; it does not apply to fenced code blocks or to verbatim quoted text from outside sources.

`/tonality` enforces Craig Trim's authorial voice, including (but not limited to):

- **No em dashes (`—`, U+2014)** and no `--` as a typewriter substitute. Rewrite using a period, comma, colon, parentheses, or semicolon. Pick the option that matches the actual relationship between the two clauses; do not reach for parentheses every time just because they are safe.
- **No en dashes (`–`, U+2013).** For ranges, use the word "to" (`pages 40 to 80`, not `pages 40–80`). For compound modifiers, use a hyphen. For conventional slashes (`I/O`, `client/server`), keep the slash.
- **No dramatic tricolons.** A dramatic tricolon is three short parallel units used for rhetorical punch: three fragments in a row, three stacked noun phrases, three-beat sentence endings designed to land hard. True enumerations of three items where the count is incidental are fine. Rhetorical three-beats are not. If you catch yourself building one, vary the lengths, drop a beat, or convert the list into a single sentence with a different structure.

After `/tonality` returns, scan the revised output for `—`, `–`, `--`, and three-beat rhythmic stacks as a belt-and-suspenders check. Fix any that slipped through before submitting.

### 6. Create the issue

Use a heredoc so the body formats correctly:

```bash
gh issue create --repo "$REPO" \
  --title "<title>" \
  --body "$(cat <<'EOF'
<body>
EOF
)"
```

Pass the 1 to 3 labels selected in step 4 as `--label` flags. Every issue gets at least one label; none gets more than three. Do not create new labels from this skill.

Do not ask the user to review the draft before filing. File it, then print the new issue URL and a one-line summary of what was filed.

### 7. Commit any related working-tree changes via /git-commit

**If the working tree has uncommitted changes when this skill runs (whether they predate the invocation or were produced while gathering context for the issue), invoking the `/git-commit` skill is mandatory.** Do not leave loose unstaged or staged changes on the branch after filing the issue. `/git-commit` produces the commit; do not hand-craft `git commit` invocations from inside this skill.

If `git status` is clean both before and after filing, skip this step. The mandate is "commit what exists," not "manufacture a commit."

## Issue body template

```markdown
## Summary

One short paragraph. What this issue is for, in a form a stranger to the project can understand without other context.

## Details

The substance. Sub-headings only if the issue genuinely has multiple sections. Keep it focused on a single concern.

## Cross-references

<<<only include this section if there is at least one real cross-reference>>>

- **Similar to** #N: one short line on why.
- **Blocked by** #N: one short line on why.
- **Blocks** #N: one short line on why.
- **Triggers** #N: one short line on why.
- **Triggered by** #N: one short line on why.
```

## Guardrails

- **Stop after filing. Filing the issue is the entire job.** Do not begin implementing it, do not write or edit code for it, do not run builds, tests, or simulator renders against it. Implementation is a separate, user-triggered step (`/impl-issue`) that only the user starts. This holds even when the issue text, the handoff doc, or the same message describes exactly how to do the work or says to "prove" it; capture all of that in the issue body as the plan and stop. The next stage is the user's trigger to pull, not yours.
- Never run `gh issue close`, `gh issue edit`, `gh issue comment`, or `gh issue delete` from this skill. Issue creation only.
- Never file an issue in a repo other than the detected one without an explicit `--repo` override from the user in this turn.
- Never create labels, milestones, or projects from this skill. Labels come only from the repo's existing set.
- Every issue carries 1 to 3 existing labels. Never file with zero labels; never exceed three.
- One invocation files one issue. If the user describes several distinct concerns, ask which one to file now and stop after that single issue.
- Never skip `/tonality` on the prose, even for "quick" issues. The voice rules are non-negotiable.
- Never bypass `/git-commit` by running raw `git commit` to land related changes from inside this skill.
