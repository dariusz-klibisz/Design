# AGENTS.md — Guidance for AI Coding Agents

This file tells AI coding agents how to read, search, and edit this repository.
Humans are welcome to read it too. For orientation, start with
[`README.md`](README.md) and the deep index at [`00-index.md`](00-index.md).

## What this repository is

- A **documentation knowledge base** on software & system architecture and design.
  There is **no application code, build, or test suite** here — only Markdown.
- It is **technology-agnostic** and **descriptive, not prescriptive.** Every entry is a
  tool with a context; the value is in the reasoning, so a reader can judge when each
  idea applies. The throughline: *everything is a trade-off.*

## Structure & conventions

- Files are numbered `00`–`13` (see the file map in [`README.md`](README.md)). The
  numbering is a stable ordering; **do not renumber or rename files** without updating
  every cross-reference.
- [`00-index.md`](00-index.md) is the canonical index: file table, *Decision
  Quick-Reference Matrix*, *Core Principle Matrix*, *Glossary*, and meta-principles.
- Most principles and patterns follow the **Standard Entry Format** (documented at the
  end of [`00-index.md`](00-index.md)):

  > Summary → Problem it addresses → Description / How it works → Benefits →
  > Costs & trade-offs → When to use / When not to use → Decision criteria →
  > Common mistakes / warning signs → Related patterns → Sources

- Heading hierarchy is consistent: `#` file title, `##` major numbered sections,
  `###` sub-sections, `####` for the Standard Entry Format fields.
- Cross-links use relative paths with anchor fragments, e.g.
  `[`02` §7](02-architecture-patterns.md#7-distributed-systems-resilience-patterns)`.

## Rules for editing

When modifying content in this repo:

1. **Preserve the Standard Entry Format and heading levels.** New principles/patterns
   should follow the same field structure so the docs stay scannable and chunk cleanly.
2. **Keep cross-links valid.** There are many internal anchor links (`file.md#heading`).
   If you rename or reword a heading, update every link that targets it. If you add a
   section, consider whether [`00-index.md`](00-index.md)'s tables/matrices should link
   to it.
3. **Update the index.** When adding, removing, or restructuring content, reflect it in
   [`00-index.md`](00-index.md) (file table, decision matrices, glossary) so navigation
   stays accurate.
4. **Cite real sources.** Authoritative references belong in
   [`09-references.md`](09-references.md). Do not invent citations or attribute claims to
   sources that don't support them. Update the `Last verified` date there if you revise
   sources.
5. **Stay technology-agnostic.** Avoid framework- or version-specific claims unless the
   surrounding text is explicitly marked as time-sensitive (e.g., Core Web Vitals
   thresholds, framework footprints). Prefer durable reasoning over current trivia.
   Engine/framework-specific guidance is allowed only in files whose header explicitly
   marks them as engine-specific and time-sensitive (currently
   [`11-godot-engine-notes.md`](11-godot-engine-notes.md)); such files must state the
   verified version and date, and must link back to the agnostic pattern they map.
   This repository must also stay **self-contained**: never link to files outside the
   repository (e.g. sibling repos or local paths).
6. **Match the existing tone.** Neutral, concise, technical. No marketing language, no
   emojis, no first-person voice.
7. **Frame everything as a trade-off.** If an entry presents only upsides, add the costs,
   when-not-to-use, and failure modes.

## What not to do

- Do not reproduce copyrighted sources wholesale; summarize and link instead.
- Do not break the index or leave dangling anchor links.
- Do not add code, tooling, or dependencies — this is a docs-only repository.
- Do not convert the prose into a rigid rulebook; preserve the "judge for your context"
  framing.

## Quick checklist before committing a docs change

- [ ] Standard Entry Format and heading levels preserved.
- [ ] All internal links and anchors still resolve.
- [ ] [`00-index.md`](00-index.md) updated if structure/content changed.
- [ ] Any new claims are backed by a source in [`09-references.md`](09-references.md).
- [ ] Tone is neutral, concise, technology-agnostic; trade-offs are stated.
