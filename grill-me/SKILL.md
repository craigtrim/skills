---
name: grill-me
description: Interview the user relentlessly about a plan or design until reaching shared understanding, resolving each branch of the decision tree. Use when user wants to stress-test a plan, get grilled on their design, or mentions "grill me".
---

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time. **Exactly one question per turn.** No compound questions. No "Q4a / Q4b / Q4c" bundles. No "two questions for you" closers. No "and also..." tag-ons. If multiple sub-decisions feel related, ask the single highest-leverage one, post the Q&A, and save the rest for later turns. Counter-questions inside a recommendation are fine as long as the final ask is one decision.

If a question can be answered by exploring the codebase, explore the codebase instead.

A grill-me session lives in a single GitHub issue, regardless of how many Q&A rounds it contains. Create the umbrella issue at the start of the session. Each Q&A round after that is a new comment on the same issue, carrying the question, the recommended answer, the user's decision, the edit applied, and cross-references to other relevant issues. Do not file separate issues per Q&A.

The umbrella issue must carry the `grill-me` label. Zero assumptions: before applying the label, check whether it exists in the target repository (`gh label list --repo <owner/repo> --search grill-me`). If it does not exist, create it with `gh label create grill-me --repo <owner/repo> --description "Interactive design grill session, tracked across comments" --color 7057FF` before opening the issue.

The issue title must indicate the topic or the document under discussion (for example, `CH01 01-02 identifying the three failure modes` or `PRD draft v3 for auth migration`). The title must not contain the string `grill-me`; the label carries that signal.

If you aren't certain which GitHub repo to use, ask.