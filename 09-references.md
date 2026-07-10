# References & Authoritative Sources

This reference synthesizes established, widely recognized sources in software architecture and engineering. The guide summarizes and integrates these sources; it does not reproduce them wholesale. Use them as **inputs to decisions, not absolute rules** — every source assumes a context.

*Last verified: 2026-07-10.*

---

## Architecture & Design

- **Martin Fowler** — *Software Architecture Guide*; *Patterns of Enterprise Application Architecture (PoEAA)*; *Microservices* (with James Lewis); articles on *PresentationDomainDataLayering*, *MonolithFirst*, *Strangler Fig / Patterns of Legacy Displacement*, *Micro Frontends*, *Serverless Architectures*, *Feature Toggles*, *GUI Architectures*. https://martinfowler.com/architecture/
- **Robert C. Martin (Uncle Bob)** — *Clean Architecture*; *Clean Code*; *Agile Software Development: Principles, Patterns, and Practices*; SOLID. https://en.wikipedia.org/wiki/SOLID
- **Eric Evans** — *Domain-Driven Design*; **Vaughn Vernon** — *Implementing Domain-Driven Design*.
- **Gang of Four** (Gamma, Helm, Johnson, Vlissides) — *Design Patterns*.
- **Neal Ford & Mark Richards** — *Fundamentals of Software Architecture*; *Software Architecture: The Hard Parts* (First/Second Laws of Software Architecture).
- **Neal Ford, Rebecca Parsons, Patrick Kua** — *Building Evolutionary Architectures* (fitness functions).
- **Alistair Cockburn** — *Hexagonal Architecture (Ports & Adapters)*. **Jeffrey Palermo** — *Onion Architecture*.
- **Simon Brown** — *The C4 Model*. https://c4model.com
- **Gregor Hohpe** — *The Software Architect Elevator*; *Enterprise Integration Patterns*.
- **Sam Newman** — *Building Microservices*; *Monolith to Microservices*.
- **Unmesh Joshi** — *Patterns of Distributed Systems*.
- **Melvin Conway** — *Conway's Law*; **Skelton & Pais** — *Team Topologies*.
- **Craig Larman** — *Applying UML and Patterns* (GRASP).
- **Meilir Page-Jones** — *connascence*; **Sandi Metz** — *Practical Object-Oriented Design* (AHA / "the wrong abstraction").
- **John Ousterhout** — *A Philosophy of Software Design* (deep vs shallow modules).
- **David Parnas** — *On the Criteria To Be Used in Decomposing Systems into Modules* (information hiding).

*Used for:* monolith vs microservices, layering, structural patterns, distributed-systems trade-offs, legacy modernization, UI architecture, evolutionary architecture, DDD, and code-level design principles.

---

## Architecture Patterns & Documentation

- **Azure Cloud Design Patterns:** https://learn.microsoft.com/en-us/azure/architecture/patterns/
- **microservices.io Pattern Language:** https://microservices.io/patterns/
- **Architecture Decision Records (ADR):** https://adr.github.io/
- **C4 Model:** https://c4model.com/

*Used for:* cache-aside, queues, CQRS, saga, transactional outbox, circuit breaker, rate limiting, deployment stamps, database-per-service, ADR rationale, and C4 context/container/component/code views.

---

## Cloud & Well-Architected Frameworks

- **Microsoft Azure Well-Architected Framework:** https://learn.microsoft.com/en-us/azure/well-architected/
- **AWS Well-Architected Framework:** https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html
- **Google Cloud Architecture / Well-Architected Framework:** https://cloud.google.com/architecture/framework

*Used for:* quality-attribute framing; the six pillars (operational excellence, security, reliability, performance efficiency, cost optimization, sustainability); cloud workload review structure; cross-pillar trade-offs.

---

## Process, Delivery & Reliability

- **Adam Wiggins / Heroku** — *The Twelve-Factor App*. https://12factor.net/
- **DORA Capabilities Catalog:** https://dora.dev/devops-capabilities/
- **Nicole Forsgren, Jez Humble, Gene Kim** — *Accelerate* (DORA metrics).
- **Jez Humble, David Farley** — *Continuous Delivery*.
- **Google** — *Site Reliability Engineering (SRE)* (SLI/SLO/SLA, error budgets, incident response, simplicity). https://sre.google/sre-book/table-of-contents/

*Used for:* continuous delivery, trunk-based development, deployment automation, observability, SLOs, toil reduction, incident response, configuration, logs, disposability, and cloud-native app practices.

---

## Distributed Systems Theory

- **Martin Kleppmann** — *Designing Data-Intensive Applications* (consistency, distributed data).
- **Eric Brewer** — *CAP Theorem*; **Daniel Abadi** — *PACELC*.
- **Deutsch & Gosling** — *Fallacies of Distributed Computing*.
- **Gene Amdahl** — *Amdahl's Law*; **Neil Gunther** — *Universal Scalability Law*.
- **Zhamak Dehghani** — *Data Mesh*.
- **Fred Brooks** — *No Silver Bullet* (essential vs accidental complexity); *The Mythical Man-Month*.
- **Ward Cunningham** — *Technical Debt* metaphor.

---

## Security & Secure Development

- **OWASP Top 10:2025** (current release): https://owasp.org/Top10/2025/ — primary web application risk categories. Previous: **OWASP Top 10:2021** https://owasp.org/Top10/2021/
- **OWASP Application Security Verification Standard (ASVS):** https://owasp.org/www-project-application-security-verification-standard/
- **OWASP Cheat Sheet Series:** https://cheatsheetseries.owasp.org/
- **NIST Secure Software Development Framework (SSDF), SP 800-218:** https://csrc.nist.gov/Projects/ssdf

*Used for:* secure SDLC, verification levels, application security requirements, vulnerability prevention, supply-chain security, secure environments, vulnerability response, and web app risk categories.

> **OWASP Top 10 note (2025 vs 2021):** Broken Access Control remains #1. **Security Misconfiguration** rose to #2; **Software Supply Chain Failures** (A03) and **Mishandling of Exceptional Conditions** (A10) are new/reframed categories; the 2021 standalone **SSRF** is folded into Broken Access Control. Rankings evolve roughly every few years — verify against the OWASP source for production decisions.

---

## Web Application Design

- **Google web.dev — Web Vitals:** https://web.dev/articles/vitals — Core Web Vitals and thresholds.
- **web.dev Learn / Performance / Accessibility:** https://web.dev/learn/
- **W3C WCAG (Web Content Accessibility Guidelines):** https://www.w3.org/WAI/standards-guidelines/wcag/
- **Kent C. Dodds** — *Testing Trophy*; **Mike Cohn** — *Test Pyramid*.

*Used for:* rendering strategies, performance, Core Web Vitals, accessibility (POUR), responsive design, privacy, progressive web apps, and testing strategy.

> **Core Web Vitals note (verified 2026-06-25):** the current stable Core Web Vitals are **LCP ≤ 2.5 s**, **INP ≤ 200 ms**, and **CLS ≤ 0.1**, assessed at the **75th percentile** across mobile and desktop. INP replaced First Input Delay (FID) as a Core Web Vital in 2024. Metrics evolve on an annual cadence; verify against web.dev.

---

## Desktop Application Design

- **Microsoft — Windows App Design / Windows App SDK / WinUI / MSIX / Authenticode:** https://learn.microsoft.com/en-us/windows/apps/
- **Apple — Human Interface Guidelines / App Sandbox / notarization / Keychain:** https://developer.apple.com/design/human-interface-guidelines
- **GNOME Human Interface Guidelines:** https://developer.gnome.org/hig/
- **KDE Human Interface Guidelines:** https://develop.kde.org/hig/
- **freedesktop.org** — XDG Base Directory spec; Flatpak / Snap / AppImage.
- **Flatpak Sandbox Permissions:** https://docs.flatpak.org/en/latest/sandbox-permissions.html
- **Electron Security:** https://www.electronjs.org/docs/latest/tutorial/security
- **Tauri Security:** https://v2.tauri.app/security/
- **Qt:** https://doc.qt.io — Widgets vs Qt Quick/QML, deployment.
- **Flutter desktop, .NET MAUI, Avalonia, Uno** — respective official docs.
- **Martin Fowler** — *GUI Architectures* (MVC/MVP/MVVM): https://martinfowler.com/eaaDev/uiArchs.html

*Used for:* platform design conventions, desktop UX, webview-based app security, trust boundaries, IPC, sandboxing, permissions, portals, code signing, update risk, and platform integration.

> *Note:* the Apple HIG page serves a JavaScript-only shell to automated fetchers. It remains an authoritative reference; this guide reflects general platform-design knowledge rather than verbatim Apple text.

---

## Game Architecture

Simulation, determinism & netcode:

- **Glenn Fiedler** — "Fix Your Timestep!", "Deterministic Lockstep",
  "Snapshot Interpolation", "Floating Point Determinism", and the
  networked-physics series — https://gafferongames.com/ (newer material at
  https://mas-bandwidth.com/).
- **Gabriel Gambetta** — *Fast-Paced Multiplayer* series (authoritative
  servers, client-side prediction & server reconciliation, entity
  interpolation, lag compensation) —
  https://www.gabrielgambetta.com/client-server-game-architecture.html
- **Tony Cannon** — GGPO rollback networking SDK (input prediction +
  speculative execution + re-simulation) — https://github.com/pond3r/ggpo
- **Blizzard Entertainment** — "Overwatch Gameplay Architecture and Netcode"
  (GDC 2017; production ECS + deterministic netcode case study).

Patterns, entity modeling & data-oriented design:

- **Robert Nystrom** — *Game Programming Patterns* (Game Loop, Update Method,
  Command, State, Observer, Event Queue, Component, Type Object, Double
  Buffer, Object Pool, Spatial Partition, Data Locality, Dirty Flag) —
  https://gameprogrammingpatterns.com/
- **Sander Mertens** — *ECS FAQ* (archetype vs sparse-set implementations,
  EC-vs-ECS distinction, data-oriented-design glossary, production users) —
  https://github.com/SanderMertens/ecs-faq
- **Adam Martin** — "Entity Systems are the future of MMOG development"
  (ECS rationale) — https://t-machine.org/
- **Scott Bilas** — "A Data-Driven Game Object System" (GDC 2002; the
  component/data-driven game-object lineage Adam Martin credits).
- **Richard Fabian** — *Data-Oriented Design* (free online book) —
  https://www.dataorienteddesign.com/dodbook/
- **Mike Acton** — "Data-Oriented Design and C++" (CppCon 2014 talk).
- **Jason Gregory** — *Game Engine Architecture* (engine subsystems, resource/
  asset pipelines, serialization; the standard industry textbook).

Game AI:

- **Steve Rabin (ed.)** — *Game AI Pro* series, all chapters free online
  (behavior selection overview, utility theory, behavior-tree starter kit,
  HTN planners, influence maps, context steering, flow-field crowds, MCTS,
  automated AI testing, autoplay agents for tuning, PCG overview) —
  http://www.gameaipro.com/
- **Damian Isla** — "Handling Complexity in the Halo 2 AI" (GDC 2005;
  behavior trees).
- **Jeff Orkin** — GOAP (Goal-Oriented Action Planning) applied in *F.E.A.R.*

Engine documentation (for [`11`](11-godot-engine-notes.md); time-sensitive):

- **Godot Engine** — official documentation, *Best practices*, *Physics
  interpolation*, *High-level multiplayer*, *Runtime file loading and saving*
  — https://docs.godotengine.org/en/stable/ (verified against Godot 4.7).
- **Juan Linietsky** — "Why isn't Godot an ECS-based game engine?"
  (godotengine.org blog, 2021; nodes-vs-ECS reasoning and the servers
  architecture).

*Used for:* sim/presentation separation, fixed-tick update pipelines, the
command pattern for deterministic replay, determinism requirements, ECS vs
scene-graph/node composition, AI decision architectures, data-driven content
and procedural generation, save persistence, async-PvP and real-time netcode
models, performance patterns, frame-budget/profiling and asset-pipeline
discipline, and game testing in [`10`](10-game-architecture.md); and the
engine mapping in [`11`](11-godot-engine-notes.md).

---

## Source Evaluation Notes

The strongest sources are **vendor/operator frameworks and standards bodies** (Well-Architected frameworks, NIST SSDF, OWASP ASVS, WCAG, Google SRE) because they are maintained and reflect production experience across many systems — but they are intentionally broad. **Practitioner books and pattern catalogs** (Fowler, Newman, Kleppmann, Evans, GoF) are best for trade-off language and failure modes.

Use sources as **inputs to decisions, not absolute rules**. Every source assumes a context: a small internal tool, a regulated financial platform, a global consumer web app, and a local-first desktop app each need a different level of rigor.

Specific quantitative figures in this reference (Core Web Vitals thresholds, availability "nines," framework binary sizes, OWASP rankings) reflect commonly published industry values at the verification date and may evolve — **always confirm against the current primary source for production decisions.**
