# Software & System Architecture and Design Reference

A unified, **technology-agnostic** reference covering system & software architecture
principles, architecture patterns & trade-offs, general software design principles,
web, desktop, and mobile application design, cross-cutting quality attributes, and
security/reliability/operations/delivery — plus a library of decision guides,
checklists, and templates. Every major concept is also cross-referenced to the
international standards (ISO/IEC/IEEE, NIST, OWASP) and regulations that formalize it.

This repository is **documentation, not code.** Each principle, pattern, and practice
is described so you can judge whether it fits a specific situation — not as a rule to
apply blindly. The unifying idea: *manage complexity, make change cheap, expect
failure, and align technical trade-offs with real business needs.*

> **Deep entry point:** [`00-index.md`](00-index.md) contains the full navigation,
> decision matrices, the "big picture" diagram, the universal meta-principles, and a
> glossary. Start there for anything beyond a quick orientation.

---

## File Map

| File | Scope |
|---|---|
| [`00-index.md`](00-index.md) | Navigation, glossary, decision matrices, the big picture, meta-principles |
| [`01-architecture-principles.md`](01-architecture-principles.md) | What architecture is, reversibility, quality principles, ADRs, C4, fitness functions, build/buy |
| [`02-architecture-patterns.md`](02-architecture-patterns.md) | Layering, distribution styles (monolith → microservices → serverless), communication, data, resilience, DDD |
| [`03-software-design-principles.md`](03-software-design-principles.md) | SOLID, DRY/KISS/YAGNI, coupling/cohesion/connascence, GRASP, design patterns, error handling, testing, refactoring |
| [`04-web-application-design.md`](04-web-application-design.md) | 12-Factor, frontend & state, rendering, Core Web Vitals, accessibility, API design, web security, authn/authz |
| [`05-desktop-application-design.md`](05-desktop-application-design.md) | UI patterns (MVC/MVP/MVVM/MVU), cross-platform vs native, frameworks, per-OS specifics, threading, packaging |
| [`06-quality-attributes-tradeoffs.md`](06-quality-attributes-tradeoffs.md) | NFR catalog, CAP/PACELC, Well-Architected pillars, ATAM, decision frameworks, essential vs accidental complexity |
| [`07-security-reliability-operations.md`](07-security-reliability-operations.md) | Secure SDLC/NIST SSDF, threat modeling, supply chain, SLOs/error budgets, observability, continuous delivery |
| [`08-checklists-and-templates.md`](08-checklists-and-templates.md) | ADR template, review checklists, quality-attribute scenarios, selection rubrics, readiness reviews |
| [`09-references.md`](09-references.md) | Authoritative sources by category, with how each informed the guide |
| [`10-game-architecture.md`](10-game-architecture.md) | Sim/presentation separation, fixed-tick pipeline, command pattern for replay, determinism, ECS vs node composition, state machines & AI architectures, data-driven content, procedural generation, saves, async-PvP & real-time netcode, performance patterns, frame budgets, asset pipeline, game testing |
| [`11-godot-engine-notes.md`](11-godot-engine-notes.md) | **Engine-specific, time-sensitive** — maps `10`'s patterns onto the Godot engine (verified against Godot 4.7) |
| [`12-mobile-application-design.md`](12-mobile-application-design.md) | UI patterns (MVVM/MVI/MVU), native vs cross-platform (Flutter/React Native/Kotlin Multiplatform), iOS/Android specifics, offline-first & sync, mobile security (OWASP MASVS/MASTG), performance, accessibility, store packaging & release |
| [`13-standards-crosswalk.md`](13-standards-crosswalk.md) | Maps this guide's concepts to formal ISO/IEC/IEEE standards and regulations (25010, 42010, 5055, 27001, WCAG, GDPR, and more) |

The `00`–`13` numbering is a stable ordering; treat the filenames as durable anchors
for cross-references.

---

## How to Navigate

Start with the **decision you need to make**, not a favorite pattern or technology.

- **Orienting / finding a section** → [`00-index.md`](00-index.md), especially its
  *Decision Quick-Reference Matrix* and *Glossary*.
- **Designing a new system** → [`06`](06-quality-attributes-tradeoffs.md) (decide what
  matters) → [`01`](01-architecture-principles.md) / [`02`](02-architecture-patterns.md)
  → [`03`](03-software-design-principles.md) → surface-specific
  [`04`](04-web-application-design.md) / [`05`](05-desktop-application-design.md).
- **Building a web app** → [`04`](04-web-application-design.md).
- **Building a desktop app** → [`05`](05-desktop-application-design.md).
- **Building a mobile app** → [`12`](12-mobile-application-design.md).
- **Building a game / real-time simulation** → [`10`](10-game-architecture.md); in the
  Godot engine specifically → [`11`](11-godot-engine-notes.md).
- **Running software in production** → [`07`](07-security-reliability-operations.md).
- **Turning a decision into a record** → [`08`](08-checklists-and-templates.md).
- **Mapping to a named standard/regulation (audit, RFP, compliance)** →
  [`13`](13-standards-crosswalk.md).

Every entry is self-contained, so you can also jump directly to any principle.

---

## For AI Coding Agents

This repo is structured for retrieval:

1. **Begin at [`00-index.md`](00-index.md).** Its *Decision Quick-Reference Matrix* maps
   a question to the exact file and section; the *Glossary* defines every acronym used.
2. **Files use a documented "Standard Entry Format"** (Summary → Problem → How it works →
   Benefits → Costs & trade-offs → When to use / not → Decision criteria → Common
   mistakes → Related → Sources). Heading levels are consistent, so sections chunk
   cleanly.
3. **Cross-links use anchor fragments** (e.g. `02-architecture-patterns.md#7-...`).
   Resolve them by file + heading.
4. **Sources live in [`09-references.md`](09-references.md).** Cite from there; do not
   invent citations.

Before editing this repository, read [`AGENTS.md`](AGENTS.md) for content conventions and
maintenance rules.
