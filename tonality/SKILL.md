---
name: tonality
description: Enforce Craig Trim's authorial voice in any prose Claude writes or edits for Craig. Apply whenever generating or revising prose intended for human readers (documentation, READMEs, articles, correspondence, book manuscripts, commit messages, PR descriptions, GitHub issues). Skip for fenced code blocks and verbatim quotes from outside sources.
---

# Tonality

Craig's authorial voice. These rules are non-negotiable in any prose Claude produces for Craig. They apply across all repositories and all written outputs intended for human readers. They do **not** apply to fenced code blocks or to verbatim quotes from outside sources (publisher emails, third-party material, cited passages).

## Rules

### 1. No em dashes

Never use the em dash character (`—`, U+2014) in prose. Also never use the typewriter substitute `--` as an em-dash stand-in.

Rewrite instead. Options:
- A period and a new sentence.
- A comma, when the second clause is a continuation rather than an aside.
- A colon, when the second clause expands or defines the first.
- Parentheses, when the material is genuinely parenthetical.
- A semicolon, when joining two independent but related clauses.

Pick the option that matches the relationship between the two clauses. Do not reach for parentheses every time just because they are safe.

### 2. No en dashes

Never use the en dash character (`–`, U+2013) in prose. This includes number ranges, score lines, and compound modifiers. Use:
- The word `to` for ranges: `pages 40 to 80`, not `pages 40–80`.
- A hyphen for compound modifiers when one is genuinely needed.
- A slash where a slash is conventional (`I/O`, `client/server`).

### 3. No dramatic tricolons

A dramatic tricolon is three short parallel units used for rhetorical punch. Common forms:
- Three short sentences in a row, each fragment-length, building to a payoff. ("Not researchers. Not beginners. Engineers.")
- Three parallel noun phrases or verb phrases stacked for emphasis. ("Unbounded inputs, stochastic outputs, drifting models.")
- Three-beat sentence endings where the last beat is meant to land hard.

The pattern is fine in technical enumeration (a true list of three items where the count is incidental). It is not fine when the three-beat rhythm is doing the persuasive work. Reach for it once and a reader notices the craft. Reach for it twice in a chapter and the prose starts to feel performed.

When you catch yourself building a tricolon for effect, break the rhythm: vary the lengths, drop one of the three, or convert the list into a single sentence with a different structure.

## How to apply

- Before producing any prose artifact for Craig, re-read these rules.
- After producing prose, scan your own output for em dashes (`—`), en dashes (`–`), `--` substitutes, and three-beat rhythmic stacks. Fix any you find before handing back to Craig.
- If you are editing existing prose that contains violations (including text copied from outside sources Craig is rewriting), silently fix them as part of the edit rather than preserving them.
- These rules are additive. Craig will extend this skill over time. When new rules arrive, append them under `## Rules` with the same structure (name, prohibition, what to do instead) and keep older rules intact unless explicitly retired.

## Notes

### Recommending a skill in a repository's `Skills/` directory

The `Skills/` directory is a recommendation system. Each developer may recommend three to five unique skills, and a skill earns a recommendation only when both of these hold:

1. The developer uses the skill on a daily basis.
2. The developer finds the skill useful.

Do not clutter the directory with skills you think might be useful. The range is three to five per developer, so if many skills meet both bars, recommend only the strongest few.

Recommendations are attributed to the developer who made them. Craig recommends `grill-me` and `research`.
