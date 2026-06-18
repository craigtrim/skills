# skills

![Claude Code Skills](https://img.shields.io/badge/Claude_Code-Skills-7057FF)
![Skills](https://img.shields.io/badge/skills-2-1f6feb)
![License](https://img.shields.io/badge/license-MIT-2ea043)

Craig Trim's Claude Code skills for stress-testing plans and designs.

## Skills

### grill-me

Interview the user relentlessly about a plan or design until reaching shared understanding. One question per turn, highest-leverage first, walking each branch of the decision tree. Every round carries a recommended answer. The whole session lives in a single GitHub issue under the `grill-me` label.

### grill-codex

The same relentless grill, but the `codex` CLI answers instead of the human. Claude drives both sides: it writes each question with options and a recommendation, invokes codex headlessly to choose, records the answer, applies the edit, and moves on. All Q&A lives as comments on one GitHub issue under the `grill-codex` label.
