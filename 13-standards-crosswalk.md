# International Standards & Regulatory Crosswalk

This file maps the **concepts used throughout this reference** to the **formal international standards, national frameworks, and regulations** that define them. The rest of this guide is intentionally built on practitioner and vendor sources (Fowler, Newman, Kleppmann, GoF, Google SRE, AWS/Azure/GCP Well-Architected, OWASP) because they are the best source of trade-off language and failure modes. But an auditor, RFP, regulator, or enterprise architecture review will often name a **standard**, not a book — this file is the bridge.

> **Use standards the same way this guide uses every other source: as inputs to a decision, not as a substitute for judgment** ([`09` — Source Evaluation Notes](09-references.md#source-evaluation-notes)). Every standard assumes a context — a small internal tool and a regulated financial platform need different rigor. Standard numbers, editions, and regulatory status **change**; the version/date noted here reflects the verification date at the bottom of this file — confirm against the issuing body before a production or compliance decision.

---

## 1. Quality Models & Measurement (SQuaRE series)

| Standard | What it defines | Where it maps in this guide |
|---|---|---|
| **ISO/IEC 25010:2023** | The product quality model: **9 characteristics** — Functional Suitability, Performance Efficiency, Compatibility, Interaction Capability (Usability), Reliability, Security, Maintainability, Flexibility (Portability), and **Safety** (added 2023). | [`06` §2](06-quality-attributes-tradeoffs.md#2-the-quality-attributes-catalog) — the guide's "-ilities" catalog is realigned to this model; see the mapping table there. |
| **ISO/IEC 25012:2008** | Data quality model: 15 characteristics across *inherent* (accuracy, completeness, consistency, credibility, currentness…) and *system-dependent* (availability, portability, traceability…) views. | [`01` §8](01-architecture-principles.md#8-treat-data-ownership-as-an-architectural-decision) data ownership; [`04` §9](04-web-application-design.md#9-data--persistence) data/persistence. |
| **ISO/IEC 25023** | Measures for the 25010 quality characteristics — turns each characteristic into measurable metrics. | [`06` §2](06-quality-attributes-tradeoffs.md#2-the-quality-attributes-catalog) "Implementation guidance" (measurable scenarios), SLI definition in [`07` §5](07-security-reliability-operations.md#5-slos-slis-and-error-budgets). |
| **ISO/IEC 25040** | The quality **evaluation process** (SQuaRE) — a structured method for evaluating a system against a quality model. | [`06` §6](06-quality-attributes-tradeoffs.md#6-architecture-evaluation-methods) alongside ATAM. |
| **ISO/IEC 5055:2021 (CISQ Automated Source Code Quality Measures)** | The first ISO standard measuring **Reliability, Performance Efficiency, Security, and Maintainability directly from source code structure**, each weakness mapped to a CWE. Also underlies the **CISQ Automated Technical Debt** measure. | [`03` §10](03-software-design-principles.md#10-refactoring-code-smells--technical-debt) technical debt; [`06` §2.6](06-quality-attributes-tradeoffs.md#2-the-quality-attributes-catalog) maintainability. |

**Note on Safety:** ISO/IEC 25010:2023 added **Safety** (operational constraint, risk identification, fail-safe, hazard warning, safe integration) as its ninth characteristic. This guide's catalog previously lacked a Safety attribute; where a system has physical, financial, or life-safety consequences (medical, automotive, industrial control, aviation), consult the **domain-specific functional-safety standards** — **IEC 61508** (generic), **ISO 26262** (automotive), **DO-178C** (avionics software), **IEC 62304** (medical device software) — which go far beyond what this technology-agnostic guide covers.

---

## 2. Architecture Description & Life Cycle

| Standard | What it defines | Where it maps |
|---|---|---|
| **ISO/IEC/IEEE 42010:2022** | The conceptual model for **architecture description**: stakeholders → concerns → (grouped, since 2022, by **stakeholder perspectives**) → viewpoints → views → correspondence rules. The formal ancestor of every "views" framework. | [`01` §9.2](01-architecture-principles.md#92-the-c4-model) — C4, 4+1 (Kruchten), arc42, and Views & Beyond are concrete **viewpoint frameworks** that instantiate 42010's abstract model. |
| **ISO/IEC/IEEE 12207:2017** | Software life cycle processes (agreement, organizational, technical, and supporting processes across the full software life cycle). | [`03` §11](03-software-design-principles.md#11-development-process--collaboration) dev process; the process backbone Agile/CD practices realize concretely. |
| **ISO/IEC/IEEE 15288:2023** | System life cycle processes — the systems-engineering counterpart to 12207, covering the system (not just software) life cycle. | Same as above, at the system level; relevant when software is one component of a larger system (embedded, IoT, safety-critical). |
| **TOGAF (The Open Group)** / **Zachman Framework** | Enterprise-architecture methodologies (ADM phases; a classification taxonomy) — heavier-weight than 42010, aimed at whole-enterprise alignment rather than a single system. | [`01` §9.2](01-architecture-principles.md#92-the-c4-model) — noted as an option when enterprise-wide governance is required; caveat: high process overhead for a single product team. |

---

## 3. Software Testing

| Standard | What it defines | Where it maps |
|---|---|---|
| **ISO/IEC/IEEE 29119** (Parts 1–5: concepts, processes, documentation, techniques, keyword-driven testing) | A comprehensive international testing standard: vocabulary, test processes, documentation templates, and test-design techniques usable in any SDLC. | [`03` §9](03-software-design-principles.md#9-testing-principles) testing principles. |

> **Contested standard — present both sides.** Unlike most standards in this file, 29119 attracted a formal, organized practitioner objection (notably from the context-driven testing community and a "Stop 29119" petition) on the grounds that it over-formalizes testing with heavyweight documentation that doesn't fit exploratory/Agile practice, and that its working group underrepresented practicing testers. Treat 29119 as a **reference vocabulary and a documentation-heavy option for regulated contexts** (e.g., where an auditor expects a named process standard), not as a required practice for most teams — the guide's pyramid/trophy ([`03` §9.1](03-software-design-principles.md#91-the-test-pyramid-and-trophy)) and FIRST principles remain the primary guidance.

---

## 4. Security — Management, Application, and SDLC Standards

| Standard | What it defines | Where it maps |
|---|---|---|
| **NIST SSDF (SP 800-218)** ✓ already cited | Secure software development practice groups (Prepare/Protect/Produce/Respond). | [`07` §1](07-security-reliability-operations.md#1-secure-software-development-lifecycle-ssdf). |
| **ISO/IEC 27034** | Application security — security requirements, controls, and verification integrated into the SDLC (the ISO counterpart to SSDF/OWASP ASVS). | [`07` §1](07-security-reliability-operations.md#1-secure-software-development-lifecycle-ssdf); [`04` §7](04-web-application-design.md#7-web-application-security). |
| **ISO/IEC 27001 / 27002:2022** | 27001: certifiable **information security management system (ISMS)** requirements. 27002: the accompanying **control catalog** (reorganized into 4 themes in the 2022 revision). | [`07` §1](07-security-reliability-operations.md#1-secure-software-development-lifecycle-ssdf) and throughout §1–4 as the formal ISMS backbone behind the guide's security practices. |
| **NIST Cybersecurity Framework (CSF) 2.0** | Six functions — Govern, Identify, Protect, Detect, Respond, Recover — a widely adopted US-originated risk-based security framework, framework-agnostic like this guide. | [`07` §1–2](07-security-reliability-operations.md#1-secure-software-development-lifecycle-ssdf). |
| **NIST SP 800-53** | A detailed catalog of security and privacy controls (the basis for FedRAMP). | [`07` §1](07-security-reliability-operations.md#1-secure-software-development-lifecycle-ssdf); §7 compliance frameworks. |
| **CIS Controls** | A prioritized, practitioner-friendly control list (Center for Internet Security) — a lighter-weight alternative/complement to 800-53. | [`07` §1](07-security-reliability-operations.md#1-secure-software-development-lifecycle-ssdf). |
| **OWASP SAMM** / **BSIMM** | Maturity models for measuring and improving an organization's software-security practice over time (SAMM is prescriptive/open; BSIMM is descriptive/benchmarking). | [`07` §1](07-security-reliability-operations.md#1-secure-software-development-lifecycle-ssdf) as a way to phase secure-SDLC adoption. |
| **OWASP ASVS** ✓ already cited | Application Security Verification Standard — concrete verifiable requirements at three rigor levels. | [`04` §7](04-web-application-design.md#7-web-application-security). |
| **OWASP MASVS / MASTG** | Mobile Application Security Verification Standard (L1/L2/Resilience) and its testing guide. | [`12` §8](12-mobile-application-design.md#8-mobile-security). |

---

## 5. Threat Modeling

| Framework | Focus | Where it maps |
|---|---|---|
| **STRIDE** ✓ already cited | Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege — general-purpose threat categories. | [`07` §2](07-security-reliability-operations.md#2-threat-modeling). |
| **LINDDUN** | A privacy-specific threat-modeling methodology (Linkability, Identifiability, Non-repudiation, Detectability, Disclosure of information, Unawareness, Non-compliance) — the privacy counterpart to STRIDE. | [`07` §2](07-security-reliability-operations.md#2-threat-modeling); [`04` §12](04-web-application-design.md#12-privacy--data-minimization) privacy. |
| **PASTA** (Process for Attack Simulation and Threat Analysis) | A risk-centric, 7-stage threat-modeling methodology emphasizing business-impact analysis and attacker simulation. | [`07` §2](07-security-reliability-operations.md#2-threat-modeling) as a heavier alternative for high-stakes systems. |
| **MITRE ATT&CK** | A knowledge base of real-world adversary tactics and techniques — used to validate that threat models and detections cover observed attacker behavior. | [`07` §2](07-security-reliability-operations.md#2-threat-modeling), [`07` §6](07-security-reliability-operations.md#6-observability) detection engineering. |

---

## 6. Supply Chain Security

| Standard / framework | What it defines | Where it maps |
|---|---|---|
| **SLSA (Supply-chain Levels for Software Artifacts)** | Graduated build-integrity levels (provenance documentation → hardened, tamper-resistant build platforms), an OpenSSF project. | [`07` §4](07-security-reliability-operations.md#4-supply-chain-security). |
| **SPDX** / **CycloneDX** | The two dominant machine-readable **SBOM** formats (Linux Foundation / OWASP respectively); CISA guidance accepts either. | [`07` §4](07-security-reliability-operations.md#4-supply-chain-security). |
| **in-toto** | An attestation framework (JSON attestations covering builder identity, source commit, build parameters, artifact digest) commonly used to carry SLSA provenance. | [`07` §4](07-security-reliability-operations.md#4-supply-chain-security). |
| **US Executive Order 14028** | US federal directive driving SBOM and secure-build requirements for software sold to the US government — the regulatory push behind SLSA/SBOM adoption. | [`07` §4](07-security-reliability-operations.md#4-supply-chain-security). |
| **EU Cyber Resilience Act (CRA)** | EU regulation imposing security-by-design, vulnerability-handling, and SBOM-adjacent obligations on "products with digital elements" sold in the EU (phased obligations through 2027). | [`07` §4](07-security-reliability-operations.md#4-supply-chain-security); [`05` §11](05-desktop-application-design.md#11-packaging-distribution--updates) desktop distribution; [`12` §11](12-mobile-application-design.md#11-packaging-signing-store-submission--release) mobile release. |

---

## 7. Privacy

| Standard / regulation | What it defines | Where it maps |
|---|---|---|
| **ISO/IEC 29100** | A privacy framework — common privacy terminology and principles, vendor/jurisdiction-neutral. | [`04` §12](04-web-application-design.md#12-privacy--data-minimization). |
| **ISO/IEC 27701** | Extends the 27001 ISMS into a **Privacy Information Management System (PIMS)** — the formal management-system counterpart to GDPR-style obligations. | [`07`](07-security-reliability-operations.md) privacy-ops note. |
| **NIST Privacy Framework** | A voluntary, risk-based US framework structured analogously to the NIST CSF. | [`04` §12](04-web-application-design.md#12-privacy--data-minimization). |
| **GDPR (EU)** | Binding regulation: lawful basis, purpose limitation, data minimization, subject-access/erasure rights, breach notification, cross-border transfer rules. | [`04` §12](04-web-application-design.md#12-privacy--data-minimization); [`01` §8](01-architecture-principles.md#8-treat-data-ownership-as-an-architectural-decision) data retention/deletion. |
| **CCPA/CPRA (California)** | US state-level consumer privacy law with GDPR-like rights (access, deletion, opt-out of sale). | [`04` §12](04-web-application-design.md#12-privacy--data-minimization). |

---

## 8. Accessibility

| Standard / regulation | What it defines | Where it maps |
|---|---|---|
| **WCAG 2.2** ✓ already cited | Web Content Accessibility Guidelines — POUR principles at A/AA/AAA conformance levels; current stable version (2.2). | [`04` §5](04-web-application-design.md#5-accessibility-a11y); [`12` §10](12-mobile-application-design.md#10-accessibility) applies via platform accessibility APIs. |
| **EN 301 549** | The EU's harmonized ICT accessibility standard — the technical standard EU public-sector and (per the EAA) private-sector accessibility obligations point to; incorporates WCAG by reference and extends it to non-web ICT (native apps, documents, hardware). | [`04` §5](04-web-application-design.md#5-accessibility-a11y); [`12` §10](12-mobile-application-design.md#10-accessibility). |
| **European Accessibility Act (EAA)** | EU directive requiring accessibility for a range of products/services (e-commerce, banking, e-books, etc.), in force with compliance deadlines from **June 2025**. | [`04` §5](04-web-application-design.md#5-accessibility-a11y); [`12`](12-mobile-application-design.md). |
| **ISO 9241-210** | Human-centred design process for interactive systems — the design-process standard underlying usability/accessibility practice generally, not a conformance checklist. | [`01`](01-architecture-principles.md), [`05` §2](05-desktop-application-design.md#2-desktop-ui-architecture-patterns), [`12` §2](12-mobile-application-design.md#2-mobile-ui-architecture-patterns). |
| **US Section 508 / ADA** | US federal procurement accessibility requirement (508) and broader civil-rights-based accessibility case law (ADA) applied to digital products. | [`04` §5](04-web-application-design.md#5-accessibility-a11y). |

---

## 9. Service Management & Risk

| Standard | What it defines | Where it maps |
|---|---|---|
| **ISO/IEC 20000** | Certifiable IT service management system standard. | [`07` §5–9](07-security-reliability-operations.md#5-slos-slis-and-error-budgets) SLO/incident/delivery practices. |
| **ITIL 4** | A widely adopted (non-certifiable-for-orgs, practitioner-certifiable) IT service management practice framework. | Same as above — the practitioner-level counterpart to 20000. |
| **ISO 31000** | General risk-management principles and process, domain-agnostic. | [`06` §7](06-quality-attributes-tradeoffs.md#7-decision-making-frameworks) decision frameworks; [`07` §2](07-security-reliability-operations.md#2-threat-modeling) threat modeling. |
| **ISO/IEC 27005** | Information-security-specific risk management, aligned with 27001. | [`07` §1–2](07-security-reliability-operations.md#1-secure-software-development-lifecycle-ssdf). |

---

## 10. Named Compliance Frameworks (Domain/Jurisdiction-Specific)

These are not general engineering standards — they apply **only when the system falls in scope** (payments, US health data, service-organization trust reporting, US government cloud). See [`07`](07-security-reliability-operations.md) for the corresponding subsection.

| Framework | Scope |
|---|---|
| **PCI-DSS** | Payment card data — any system that stores, processes, or transmits cardholder data. |
| **HIPAA** | US protected health information (PHI) — covered entities and business associates. |
| **SOC 2** | A service organization's controls report (Security, Availability, Processing Integrity, Confidentiality, Privacy "trust services criteria") — commonly requested by enterprise customers of a SaaS vendor, not a government mandate. |
| **FedRAMP / StateRAMP** | US federal/state government cloud-service authorization, built on NIST SP 800-53 control baselines. |

---

## 11. Internationalization

| Standard | What it defines | Where it maps |
|---|---|---|
| **Unicode** / **CLDR (Common Locale Data Repository)** | Character encoding and the locale data (date/number/currency formats, pluralization rules, collation) that underlies correct i18n. | New i18n section, [`03` §12](03-software-design-principles.md#12-internationalization--localization-i18nl10n). |
| **ISO 639** / **ISO 3166** / **ISO 4217** / **ISO 8601** | Language codes, country codes, currency codes, and date/time format respectively — the building blocks of any locale identifier and locale-aware data exchange. | Same as above. |
| **W3C Internationalization** | Web-specific i18n guidance (HTML `lang`, BCP 47 tags, bidi text handling). | [`04`](04-web-application-design.md), [`03` §12](03-software-design-principles.md#12-internationalization--localization-i18nl10n). |

---

## How to Use This Crosswalk

1. **Find your concept** in the section headers above (quality, architecture description, testing, security, supply chain, privacy, accessibility, service management, compliance, i18n).
2. **Check whether a named standard applies to your context** — most engineering standards (25010, 42010, 5055) apply broadly; most regulatory frameworks (PCI-DSS, HIPAA, GDPR, EAA) apply only when your system is in scope (payment data, EU users, public-facing EU products, etc.).
3. **Use the Standards Conformance Triage checklist** ([`08`](08-checklists-and-templates.md)) to do this scoping quickly for a new system or review.
4. **Cite the standard by number/title/year** in ADRs and compliance documentation, but **verify current text and status against the issuing body** — this file summarizes; it does not reproduce paywalled standards text.

---

## Key Cross-References
- **Quality attributes:** [`06`](06-quality-attributes-tradeoffs.md). **Architecture principles:** [`01`](01-architecture-principles.md). **Code-level design:** [`03`](03-software-design-principles.md).
- **Web / desktop / mobile / game surfaces:** [`04`](04-web-application-design.md), [`05`](05-desktop-application-design.md), [`12`](12-mobile-application-design.md), [`10`](10-game-architecture.md).
- **Security, reliability, operations:** [`07`](07-security-reliability-operations.md). **Checklists & templates:** [`08`](08-checklists-and-templates.md). **Full source list:** [`09`](09-references.md).

*Last verified: 2026-07-15. Standard editions and regulatory status (especially EU AI Act/CRA phased obligations, WCAG minor revisions, and OWASP MASVS versioning) change — confirm against the issuing body for any compliance-relevant decision.*
