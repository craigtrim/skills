# Tonality

![Claude Code Skill](https://img.shields.io/badge/Claude_Code-Skill-7057FF)
![Applies to](https://img.shields.io/badge/applies_to-prose-2ea043)
![Status](https://img.shields.io/badge/status-active-success)

Enforce Craig Trim's authorial voice in any prose Claude writes or edits.

## What it does

The skill loads a fixed set of voice rules before Claude produces prose for a human reader. After writing, Claude scans its own output and fixes any violation before handing the text back.

## When it applies

Documentation, READMEs, articles, correspondence, book manuscripts, commit messages, PR descriptions, and GitHub issues. It does not apply inside fenced code blocks or to verbatim quotes from outside sources.

## Rules

1. **No em dashes.** Rewrite with a period, comma, colon, parentheses, or semicolon, whichever matches the relationship between the clauses.
2. **No en dashes.** Use `to` for ranges, a hyphen for compound modifiers, and a slash where one is conventional.
3. **No dramatic tricolons.** Three short parallel units used for rhetorical punch get broken up. A true list of three items is fine.

The rules are additive. New rules append under `## Rules` in [`SKILL.md`](./SKILL.md) and older ones stay intact unless explicitly retired.

## Usage

Invoke `/tonality`, or let Claude apply it automatically whenever it generates prose for Craig.
