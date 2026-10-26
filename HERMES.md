---
title: Wiki Schema
created: 2026-10-15
updated: 2026-10-15
type: schema
tags: [meta, schema]
---

# Wiki Schema

## Domain
Personal blog and knowledge base: development, homelab, IT, AI, creativity, and project ideas. This wiki serves as a persistent, compounding knowledge store for technical notes, project documentation, and research.

## Conventions
- File names: lowercase, hyphens, no spaces (e.g., `nixos-configuration.md`)
- Every wiki page starts with YAML frontmatter (see below)
- Use `[[wikilinks]]` to link between pages (minimum 2 outbound links per page)
- When updating a page, always bump the `updated` date
- Every new page must be added to `index.md` under the correct section
- Every action must be appended to `log.md`
- **Provenance markers:** On pages that synthesize 3+ sources, append `^[raw/articles/source-file.md]`
  at the end of paragraphs whose claims come from a specific source.

## Frontmatter
All wiki pages must include:

```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | summary | idea | todo
tags: [from taxonomy below]
sources: [raw/articles/source-name.md]
# Optional quality signals:
confidence: high | medium | low        # how well-supported the claims are
contested: true                        # set when the page has unresolved contradictions
contradictions: [other-page-slug]      # pages this one conflicts with
---
```

### raw/ Frontmatter
Raw sources get a small frontmatter block for drift detection:

```yaml
---
source_url: https://example.com/article   # original URL, if applicable
ingested: YYYY-MM-DD
sha256: <hex digest of the raw content below the frontmatter>
---
```

## Tag Taxonomy
- **Categories:** blog, creativity, IT, AI, development, homelab, nixos, infrastructure, networking, security, automation, documentation
- **Content Types:** idea, project, tutorial, guide, reference, comparison, analysis, notes, draft
- **Status:** active, stalled, completed, archived, experimental
- **Work:** todo, improvement, fix, housekeeping
- **Meta:** meta, schema, index, log

Rule: every tag on a page must appear in this taxonomy. Add new tags here first.

## Page Thresholds
- **Create a page** when an entity/concept appears in 2+ sources OR is central to one source
- **Add to existing page** when a source mentions something already covered
- **DON'T create a page** for passing mentions, minor details, or things outside the domain
- **Split a page** when it exceeds ~200 lines — break into sub-topics with cross-links
- **Archive a page** when its content is fully superseded — move to `_archive/`, remove from index

## Entity Pages
One page per notable entity (tools, technologies, projects). Include:
- Overview / what it is
- Key facts and dates
- Relationships to other entities (`[[wikilinks]]`)
- Source references

## Concept Pages
One page per concept or topic. Include:
- Definition / explanation
- Current state of knowledge
- Open questions or debates
- Related concepts (`[[wikilinks]]`)

## Comparison Pages
Side-by-side analyses. Include:
- What is being compared and why
- Dimensions of comparison (table format preferred)
- Verdict or synthesis
- Sources

## Idea Pages
For project ideas and drafts. Include:
- Problem statement
- Proposed solution
- Status (active, stalled, completed)
- Related ideas and dependencies

**Scope rule:** `ideas/` is for **new services/projects that don't exist on the server yet**.
Improvements, fixes and housekeeping for things **already implemented** go in `todo/` instead.

**Lifecycle:** when an idea is implemented on the server, its knowledge moves to `docs/`
(per-service page or the relevant reference page, incl. a short "Decision history" paragraph
if there was a real trade-off) and the idea file is **deleted**. Update any wikilinks pointing
at it and log the deletion. Git history preserves the original draft.

## Todo Pages
One file per improvement/fix item under `todo/` (kebab-case, lowercase). Include:
- Status (Not started / In progress / Done — Done items move to `_archive/` after verification)
- Priority (Critical / High / Medium / Low)
- Description of the current state and the gap
- Task checklist
- Detailed notes (implementation guidance, caveats, acceptance test)
- Related wiki pages
`todo/index.md` is the hub: one line per item with status + priority + link. When an item is
completed, tick the checklist, bump `updated`, and link the resulting docs/ or service page.

## Update Policy
When new information conflicts with existing content:
1. Check the dates — newer sources generally supersede older ones
2. If genuinely contradictory, note both positions with dates and sources
3. Mark the contradiction in frontmatter: `contradictions: [page-name]`
4. Flag for user review in the lint report
