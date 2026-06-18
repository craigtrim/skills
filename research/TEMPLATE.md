# Research Issue Template

Use this as the `--body` content for `gh issue create`. Title should be the topic stated as a researchable question or noun phrase (e.g., "Context rot in long-context LLMs: empirical evidence and mitigations").

```markdown
## Scope

One paragraph stating exactly what this issue researches and what it does not. If this issue cross-references another artifact (issue, video, paper, article), name it here with a link and one sentence on the relationship. Do not quote the source artifact at length.

## Research questions

- RQ1: ...
- RQ2: ...
- RQ3: ...

Three to six questions is typical. They should be answerable from literature, not opinion.

## Findings by sub-theme

### <Sub-theme 1, e.g., "Empirical evidence">

Synthesis paragraph naming the sources inline (Author, Year) and what they collectively show. Surface agreement and disagreement. No claims without a citation.

### <Sub-theme 2, e.g., "Mechanistic explanations">

...

### <Sub-theme 3>

...

## Annotated bibliography

Group by venue type. Each entry: full citation, canonical URL, and a one to three sentence summary grounded in the abstract.

### Peer-reviewed and preprint

1. **Author, A., Author, B., & Author, C. (Year).** *Exact Title.* Venue. <canonical URL>
   One to three sentence summary of what the paper shows and why it matters for this topic.

2. ...

### Surveys and benchmarks

1. ...

### Supplementary (blogs, model cards, primary-source non-academic)

Only include this section if the topic is genuinely too recent for peer-reviewed coverage, or if the source is itself the primary record (e.g., a lab model card). Otherwise omit.

1. ...

## Open questions and directions

- Where is the evidence thin?
- Where do sources disagree?
- What experiments or analyses would resolve the disagreement?

## Methodology note

Sources gathered via WebSearch and WebFetch on <YYYY-MM-DD>. Every citation was verified against its canonical landing page (arXiv abs, ACL Anthology, publisher DOI, etc.) during this research session. No citations were generated from model memory.
```

## Field-by-field rules

- **Title** — researchable phrase, no clickbait, no em dashes.
- **Scope** — explicit boundaries; this is what stops scope creep during follow-on work.
- **Research questions** — should be questions a literature review can answer, not opinions.
- **Findings by sub-theme** — synthesis, not a list. Every claim has an inline citation.
- **Annotated bibliography** — verified entries only; if you could not load the canonical page, drop the entry.
- **Methodology note** — always include; it tells future readers the citations are real.
