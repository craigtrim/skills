---
name: git-commit
description: Write a commit for the current changes following Craig's conventions. Use when the user asks to commit staged or working-tree changes.
---

# Git Commit

**User Instructions:** $ARGUMENTS

## CRITICAL: No Co-Authored-By Lines

**NEVER** add `Co-Authored-By` lines, Claude attribution, or "Generated with Claude Code" footers to commits. No exceptions.

## Commit Format

One-line messages with conventional prefixes:
- `feat:` New feature
- `fix:` Bug fix
- `docs:` Documentation
- `refactor:` Code restructuring
- `chore:` Maintenance
- `test:` Tests

**Do NOT include:** file lists, bullet summaries, or multi-paragraph descriptions.

## Commit Rules

- **Atomic commits.** One logical change per commit.
- **Max ~15 files per commit.** Split by component if larger.
- **Never commit** until the user explicitly asks.
- **Do NOT push.** Commit locally only, never push to remote.

## Instructions

Follow the user's instructions in `$ARGUMENTS`. Check for changes before committing. If no changes exist, inform the user and stop.
