# System Design: FinTech Consumer Content Hub
### AEM Component Architecture — MVP & Future State

---

## Overview

This repository contains the system design document for the FinTech Consumer Content Hub — a detailed AEM component architecture covering the MVP build and the phased evolution through Engagement, Personalization, and Delight.

Where the [platform PRD](https://github.com/thedmons/prd-content-hub) defines what to build and why, this document defines how it's built: the component model, content model, authoring pipeline, Dispatcher caching strategy, search and analytics integration, and the architectural decisions required to support future phases without requiring foundational rework.

📄 **[Read the full System Design →](./system-design-content-hub.md)**

---

## What This Demonstrates

**Technical depth** — I face new initiaves by working closely with engineering - not just directing what gets built, but understanding and documenting how it's built well enough to communicate with stakeholders, make tradeoff decisions, and evaluate implementation choices.

**Component model and JCR architecture** — The document includes a full AEM component library (structural, editorial, and shared components), page template hierarchy, and JCR content model with node structure — the kind of detail that enables a development team to build without ambiguity.

**Documented design decisions with tradeoffs** — Four major architectural decisions are explicitly documented with rationale and tradeoffs: single AEM instance vs. separate instance; tag-matching vs. ML for related content; AEM Tags vs. external taxonomy service; Dispatcher-level vs. application-level redirects. This reflects how I approach technical decisions — not just picking an option, but articulating what was considered and why the chosen path is right for this context.

**Future state architecture that informs present decisions** — The document maps the architectural changes required for each post-MVP phase (Engagement, Personalization, Delight), including new components, new integrations, and the evolution from rule-based to CDP-driven personalization. This framing ensures MVP architecture is designed with the future in mind rather than optimized only for the current state.

**Caching strategy and performance design** — The Dispatcher cache strategy section defines per-content-type TTLs and invalidation triggers — the kind of operational detail that connects product performance targets (LCP < 2.5s) to concrete infrastructure decisions.

---

## Document Metadata

| | |
|---|---|
| **Product** | Consumer Content Hub (fintech.com/stories) |
| **Document Type** | System Design |
| **Status** | Completed |
| **Date** | Q1 2022 |
| **Platform** | Adobe Experience Manager (AEM) |

---

## Related Artifacts

| Artifact | Description |
|---|---|
| [PRD: FinTech Consumer Content Hub](https://github.com/thedmons/prd-content-hub) | Platform PRD this system design implements — requirements, personas, success metrics, phased roadmap |
| [PRD: Visual Stories](https://github.com/thedmons/prd-visual-stories) | Feature PRD for the Phase 4 Delight feature built on this architecture |
