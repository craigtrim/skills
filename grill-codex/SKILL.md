---
name: grill-codex
description: Run a relentless design grill where YOU pose each question and the `codex` CLI answers it, instead of a human. For each decision you write the question, options, and your recommendation, invoke codex headlessly to choose, record codex's answer as the decision, apply the edit, and move to the next. All Q&A lives as comments on a single GitHub issue. Use when the user says "grill-codex", "have codex grill <X>", or wants a plan or design stress-tested with Codex as the responder rather than answering it themselves.
---

# Grill Codex

Like grill-me, but the responder is the `codex` CLI, not the human. You drive both sides: ask, get codex's answer, record it, apply the edit, repeat. Walk the design tree one decision at a time, highest-leverage first.

## Setup (once per session)

1. Detect the repo: `gh repo view --json nameWithOwner -q .nameWithOwner`. If it errors or is ambiguous, ask which repo. Confirm `gh auth status` succeeds.
2. Confirm codex is callable: `command -v codex`. If absent, stop and tell the user.
3. Ensure the `grill-codex` label exists: `gh label list --repo <repo> --search grill-codex`. If missing: `gh label create grill-codex --repo <repo> --description "Design grill with Codex as responder, tracked across comments" --color 7057FF`.
4. Use ONE umbrella issue for the whole session. If the user points at an existing issue or doc ("grill-codex on 553"), host the rounds as comments on that issue and add the label to it. Otherwise create the issue; its title names the topic and must NOT contain the string "grill-codex" (the label carries that signal).

## Each round (exactly ONE question)

1. Pick the single highest-leverage unresolved decision. If it can be answered by reading the codebase, read it instead of asking.
2. Draft the round in the gold-standard format below: question, options, your recommendation, a folded counter-question, and empty Decision / Edit fields.
3. Ask codex **synchronously** for THIS one question and read its answer inline.
   **Deliver the prompt from a file, never as an inline double-quoted argument.**
   Write the round text to a temp file, then pass it with `"$(cat ...)"`:
   ```bash
   cat > /tmp/grill-codex-prompt.txt <<'EOF'
   <the round text + the relevant file snippets pasted inline>
   EOF
   codex exec --sandbox danger-full-access --skip-git-repo-check --color never \
     -C <repo-root> "$(cat /tmp/grill-codex-prompt.txt)" </dev/null
   ```
   Why the file, not an inline `"..."` string: a code-bearing prompt almost always
   contains backticks, `$(...)`, `${...}`, or `!`, and inside a double-quoted shell
   argument the shell evaluates those *before* codex ever sees them. The symptoms are
   a `parse error near` / `command not found`, or worse, a silently corrupted prompt
   where backtick spans are replaced by the output of running them as commands. The
   heredoc uses a **quoted delimiter** (`<<'EOF'`) so the body is taken literally, and
   `"$(cat file)"` injects that literal text as one argument with no re-scanning. This
   bites constantly when the prompt quotes Swift/TS/shell snippets; reach for the file
   first rather than fighting the escaping. (Single-quoting the whole prompt inline is
   not a fix: prompts contain apostrophes and single quotes of their own.)

   Three more things make this work and return in seconds (all learned the hard way):
   - **Long Bash `timeout` (e.g. 600000) is mandatory.** On the default ~120s
     timeout the harness detaches codex into the background and you fall into
     async-poll, which Claude handles badly. A long timeout makes the call block
     and return inline. Verify sync works once per session with a trivial prompt
     (`"Reply with one word: PONG"`); it should return in a few seconds.
   - **`</dev/null` is mandatory.** `codex exec` reads stdin even when the prompt
     is an argument; an open stdin hangs the run forever.
   - **Do NOT tell codex to "read the codebase."** Repo exploration is what turns a
     6-second answer into a multi-minute one. Paste the relevant file snippets and
     context INLINE in the prompt so codex answers directly. Keep the prompt
     self-contained.
   Match the user's codex setup (the user runs `--sandbox danger-full-access`).
   codex is only answering a question here, not changing files.
4. Fill **Decision** with codex's answer recorded faithfully (verbatim or lightly trimmed), naming the option it chose. Fill **Edit applied** with the concrete change to the plan/doc/issue, and add any cross-references.
5. Post the completed round as a comment on the umbrella issue.
6. Only AFTER reading codex's answer, pick the next single question. The answer to one question shapes the next, so never pre-write or batch future rounds. Stop when the design tree is resolved, nothing high-leverage remains, the user interjects, or codex and your recommendation diverge so widely that the split needs the user's call (see the divergence rule below).

## Gold-standard comment format

Model every round on this comment: https://github.com/Maryville-University-DLX/SaintsOutReach/issues/553#issuecomment-4623200542

```markdown
## Round N — <the single question?>

<1-2 sentences of context: why this is the fork>

**Options**
- **A. <name>.** <one line>
- **B. <name>.** <one line>
- **C. <name>.** <one line, only if a real third option exists>

**Recommendation: <X>.** <why, grounded in the code and constraints>

Folded counter-question: <the one nuance worth codex weighing>.

**Decision:** <codex's choice and its rationale, recorded faithfully>
**Edit applied:** <the concrete change made as a result>
```

## Rules

- Exactly one question per round and per comment. No compound questions, no "Q4a / Q4b" bundles, no "and also" tag-ons.
- All Q&A for a session lives on ONE issue, as comments. Never file a separate issue per question.
- Call codex SYNCHRONOUSLY: long Bash `timeout` plus `</dev/null`, read the answer inline. Never background-and-poll, and never wait on a completion notification for this.
- NEVER batch questions. The grill is an adaptive decision tree: ask one, read the answer, then decide the next. One sync codex call per question, with the relevant context inlined so the run stays fast.
- Apply Craig's voice (the `tonality` skill) to the prose YOU write: no em dashes, no en dashes, no dramatic tricolons. codex's recorded answer is a quote; leave it as written.
- Keep your recommendation honest, and do not defer to codex. codex is a foil whose job is to surface sharper questions and better discussion, not an oracle whose answer you adopt by default. Recommend the disciplined option even when codex differs, weigh its answer on the merits, and record plainly where it disagrees with you. Disagreement between you and codex is a feature of the grill, not a problem to smooth over.
- Halt on wide divergence. When codex's answer and your recommendation diverge so far that recording the decision would misrepresent the design, a genuine conflict rather than a minor preference, stop the grill and bring the disagreement to the user with both positions laid out. Do not paper over a deep split by silently picking a side.
- The human can interrupt, redirect, or override any round at any time.
