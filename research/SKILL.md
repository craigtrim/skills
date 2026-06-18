---
name: research
description: Research a topic with abundant academic sources (zero generative-memory citations) and file the findings as a GitHub issue labeled "research". Use when user wants to research a topic, gather academic literature, build a citation-rich annotated bibliography, or file a research issue. Also use when user references an existing issue/video/article and wants a net-new research issue that cross-references it.
---

# Research

Turn a topic into a citation-rich GitHub issue. Zero generative-memory citations. Academic sources dominate; blog posts and subjective sources are only acceptable for genuinely modern topics where peer-reviewed work does not yet exist.

## Operating principles

- **Zero assumptions.** If a fact is not in a source you fetched, do not write it.
- **No generative memory for citations.** Every citation must come from a `WebSearch` or `WebFetch` call performed in this session. Never invent authors, dates, titles, DOIs, arXiv IDs, or page numbers. If you cannot verify a citation, drop it.
- **Academic-first.** Prefer peer-reviewed venues (ACL, NeurIPS, ICML, EMNLP, NAACL, ICLR, AAAI, IEEE, ACM, Nature, Science, journal articles) and arXiv preprints. Blog posts, vendor docs, and Substack are acceptable supplements only when (a) the topic is too recent for peer-reviewed coverage, or (b) the author is the primary source (e.g., a model card from the lab that built the model).
- **Abundance.** Aim for at least 10–15 distinct sources for a normal topic, more for broad ones. Coverage matters more than count, but err toward more.
- **Verify before citing.** For each candidate source, run `WebFetch` against the canonical URL (arXiv abs page, ACL Anthology page, journal landing page, publisher DOI). Pull the title, authors, year, venue, and abstract from the page itself. Quote the abstract sparingly when summarizing.

## Process

### 1. Capture the topic and source context

Get the topic from the user. If the user references an existing artifact (GitHub issue, video timestamp, article, paper), `WebFetch` it first so the research has the correct frame. Note any explicit scope boundaries ("specifically this section", "only the part about X").

The user's framing is the brief; do not expand scope on your own. If the brief is ambiguous, ask **one** inline clarifying question in prose (never the questionnaire tool) and continue.

### 2. Detect the target repo

The session's working directory usually identifies the repo (`git remote get-url origin` or `gh repo view --json nameWithOwner`). If you cannot determine a repo, ask the user inline which repo to file the issue in. Do not guess.

### 3. Build the source set

Use `WebSearch` and `WebFetch` in parallel. Good entry points:

- `WebSearch` queries scoped to scholarly domains: `arxiv.org`, `aclanthology.org`, `aclweb.org`, `proceedings.neurips.cc`, `proceedings.mlr.press`, `openreview.net`, `dl.acm.org`, `ieeexplore.ieee.org`, `link.springer.com`, `jstor.org`, `nature.com`, `science.org`, `pubmed.ncbi.nlm.nih.gov`, `semanticscholar.org`.
- Targeted searches: `"<topic>" site:arxiv.org`, `"<topic>" survey`, `"<topic>" benchmark`, `"<topic>" filetype:pdf site:aclanthology.org`.
- Citation chasing: when you find a strong paper, fetch its abstract page and look at related-work links; then search for those titles directly.

For each candidate, `WebFetch` the canonical landing page and confirm:

- Title (exact)
- Authors (full list, or first three + "et al." if long)
- Year
- Venue / publisher
- URL (canonical, not a search redirect)
- A one to three sentence summary in your own words, grounded in the abstract or paper body

Discard any source you cannot verify.

### 4. Synthesize

Group sources by sub-theme (e.g., "Empirical evidence of degradation", "Architectural explanations", "Mitigations / training-time fixes", "Evaluation methodology"). For each group, write a short synthesis paragraph that names the sources and what they collectively show. Do not editorialize beyond what the sources support.

Surface tensions explicitly: where do sources disagree? Where is evidence thin? Where would more work help?

### 5. Ensure the `research` label exists

Before filing, check whether the repo has a `research` label and create it if not. Do this without asking:

```bash
gh label list --repo <owner/repo> --search research --json name --jq '.[].name' | grep -qx research \
  || gh label create research --repo <owner/repo> --description "Academic literature review / research synthesis" --color 0E8A16
```

### 6. File the issue

Use `gh issue create` with the template in [TEMPLATE.md](TEMPLATE.md). If the user pointed to a source issue, cross-reference it in the body using GitHub's `#<num>` or full URL form, with one or two sentences explaining the relationship. **Do not quote the source material at length and do not reproduce copyrighted text** — the new issue stands on its own academic legs.

After creation, print the issue URL and a one-line summary of the source set (e.g., "Filed #142 — 14 sources across 4 sub-themes; arXiv (8), ACL Anthology (3), NeurIPS (2), blog (1)").

## What to avoid

- **No fabricated citations.** If you did not `WebFetch` it, do not cite it.
- **No "according to recent research" without a source.** Every claim attaches to a citation.
- **No paraphrased-but-uncited synthesis.** If a sentence depends on a source's finding, name the source inline.
- **No copying long quotes** from the source artifact the user referenced (issue, video, article). Cross-reference and build on it; do not reproduce it.
- **No scope creep.** Stay inside the section/topic the user specified.

## Format and style notes

The issue body is technical prose, not an article for craigtrim.com. Still: no em dashes, no rhetorical tricolons, no "uncomfortable truth"-style framing. Keep claims tight and source-bound.

See [TEMPLATE.md](TEMPLATE.md) for the issue body skeleton.
