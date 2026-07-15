# AI-Powered ICS Dashboard — Strategy & Implementation Repository

**Client Engagement Type:** Financial Services — Internal Control System Modernisation  
**Accenture Practice:** Finance & Risk / Technology Strategy  
**Repository Purpose:** Strategy documents, architecture specifications, and implementation guide for an AI-powered ICS Cockpit

---

## What This Repository Is

This repository contains the full strategy and technical blueprint for building an **enterprise-grade AI-powered Internal Control System (ICS) Cockpit** for a financial services client. The dashboard replaces or augments an existing GRC platform UI with a purpose-built, intelligence-driven interface that:

- Surfaces Risk and Control Self-Assessment (RCSA) progress, risk ratings, and control effectiveness in real time across six business areas
- Embeds an AI agent that can answer natural language questions about the risk register, draft gap summaries, and flag trends
- Automates Third-Party Risk Management (TPRM) signal processing to generate draft risk entries when vendor classifications change
- Orchestrates 1LoD/2LoD workflows: reminders, escalations, and assessment initiations directly from the dashboard

This is not a prototype or proof-of-concept repository. Every document here is written to production-delivery standard and is intended to be handed directly to the engagement team as working project artefacts.

---

## The Three Pillars

| Pillar | Description | Document |
|--------|-------------|----------|
| **Client Data Requirements** | Every data source, field, format, and quality criterion needed from the client before a single line of code is written. Structured by GRC data, TPRM data, org data, and document corpus. | [docs/01-client-data-requirements.md](docs/01-client-data-requirements.md) |
| **AI Infrastructure Architecture** | Full technical architecture from data ingestion through to the AI/ML services layer, API gateway, and frontend. Includes database schema sketches, vector search design, LLM agent design, and security/compliance controls. | [docs/02-ai-infrastructure.md](docs/02-ai-infrastructure.md) |
| **Implementation Roadmap** | Six-phase delivery plan (28 weeks), Gantt-style timeline table, and a project-level risk register with mitigations. | [docs/03-roadmap.md](docs/03-roadmap.md) |

Supporting documents:

| Document | Description |
|----------|-------------|
| [docs/04-team-and-roles.md](docs/04-team-and-roles.md) | Role definitions, FTE allocations per phase, RACI matrix, and what the client needs to provide in terms of their own SMEs |
| [docs/05-dashboard-feature-spec.md](docs/05-dashboard-feature-spec.md) | Every interactive element in the dashboard mapped to its API call, loading state, error state, and success state |
| [templates/ics_dashboard.html](templates/ics_dashboard.html) | Reference HTML dashboard (combined single-file build for review purposes) |

---

## Dashboard Overview

The ICS Cockpit is structured around a multi-level navigation model:

```
Overview (KPI landing page)
  └── Area view  (e.g. Operations & Sourcing, Payments, IT & Security...)
        └── Process view  (e.g. Procurement, AP Processing, Identity Management...)
              └── Risk / Control detail panels
```

### Six Business Areas Covered

| Area ID | Area Name | Typical Processes |
|---------|-----------|-------------------|
| P-01 | Operations & Sourcing | Procurement, Supplier Management, Accounts Payable |
| P-02 | Payments | SEPA/SWIFT Execution, Nostro Reconciliation, FX Settlements |
| P-03 | Data & Privacy | Data Classification, GDPR Compliance, Data Retention |
| P-04 | Finance | Financial Reporting, Month-end Close, Tax Compliance |
| P-05 | IT & Security | Access Management, Incident Response, Change Management, BCP |
| P-06 | Treasury | Liquidity Risk, Investment Management, Cash Forecasting |

### Key KPI Metrics (Landing Page)

- **Open Risks** — total count of risks in Open status across all areas
- **High/Critical Risks** — count of risks rated High or Critical (inherent or residual)
- **Control Gaps** — controls rated Ineffective or Partially Effective
- **Overdue RCSAs** — RCSA assessments past their due date with no completion record
- **1LoD Pending Input** — workflow tasks awaiting first-line-of-defence action

---

## Quick-Reference: Table of Contents

1. [Client Data Requirements](docs/01-client-data-requirements.md)
   - GRC System Data (Risk Register, Control Register, RCSA Results, Process Inventory, etc.)
   - TPRM Data (Vendor Registry, Assessment Results, DORA ICT list)
   - Organisational Data (Org hierarchy, 1LoD/2LoD designation, reporting calendar)
   - Document Corpus (policies, procedures, regulatory frameworks for RAG)
   - Access & Integration Requirements
   - Data Readiness Checklist

2. [AI Infrastructure Architecture](docs/02-ai-infrastructure.md)
   - Layered architecture overview
   - Data ingestion & ETL design
   - Database schema (relational + vector)
   - AI/ML components (LLM agent, TPRM engine, trend detection, workflow engine)
   - REST API endpoint catalogue
   - Security & compliance controls
   - Technology stack summary

3. [Implementation Roadmap](docs/03-roadmap.md)
   - Phase 0: Discovery & Setup (Weeks 1–4)
   - Phase 1: Data Foundation (Weeks 5–12)
   - Phase 2: Core Dashboard (Weeks 9–16)
   - Phase 3: AI Layer (Weeks 13–20)
   - Phase 4: Workflow & Actions (Weeks 17–24)
   - Phase 5: Hardening & Rollout (Weeks 21–28)
   - Gantt table and project risk register

4. [Team Composition & Roles](docs/04-team-and-roles.md)
   - 9 Accenture roles with FTE allocation by phase
   - Required client-side SMEs
   - RACI matrix
   - Staffing timeline

5. [Dashboard Feature Specification](docs/05-dashboard-feature-spec.md)
   - Every button, tab, and interactive element mapped to its API call
   - Loading, error, and success states per element
   - State management and real-time update approach

---

## Roles and Project Timeline Summary

| Role | FTE | Peak Phases |
|------|-----|------------|
| Programme Manager | 1.0 | All phases |
| ICS / GRC Domain Expert | 1.0 | 0, 1, 2, 3 |
| Solution Architect | 1.0 → 0.5 | 0–2 full, 3–5 at 50% |
| Data Engineer | 2.0 | 1, 2 |
| AI/ML Engineer | 1.5 | 3, 4 |
| Backend Developer | 2.0 | 2, 3, 4 |
| Frontend Developer | 1.5 | 2, 3, 4 |
| Security Architect | 0.5 | 0, 5 |
| Change Management Lead | 0.5 | 4, 5 |

**Total project duration:** 28 weeks (7 months)  
**Estimated Accenture FTE-weeks:** ~200 FTE-weeks across all phases  
**Delivery model:** Hybrid (on-site for workshops and UAT; remote for development sprints)

---

## How to Use This Repository

### For the Engagement Lead / Programme Manager
Start with the [Roadmap](docs/03-roadmap.md) and [Team & Roles](docs/04-team-and-roles.md) documents to plan the project structure and resource request. Use the Data Readiness Checklist in [Client Data Requirements](docs/01-client-data-requirements.md) as the first client-facing worksheet in Phase 0.

### For the Solution Architect
The [AI Infrastructure](docs/02-ai-infrastructure.md) document is your primary reference. It contains the architecture decision context, technology stack, database schema sketches, API endpoint catalogue, and security controls. Use it as the basis for the Architecture Decision Record (ADR) produced in Phase 0.

### For Data Engineers
Begin with [Client Data Requirements](docs/01-client-data-requirements.md) to understand every data entity you need to ingest. Cross-reference with the ETL and storage sections in [AI Infrastructure](docs/02-ai-infrastructure.md) for the technical design.

### For Frontend / Full-Stack Developers
The [Dashboard Feature Specification](docs/05-dashboard-feature-spec.md) is your implementation contract. Every interactive element, API call, and state transition is documented. The reference HTML in [templates/ics_dashboard.html](templates/ics_dashboard.html) is the visual and structural reference.

### For AI/ML Engineers
Section 4 of [AI Infrastructure](docs/02-ai-infrastructure.md) describes each AI component in detail: LLM agent with RAG, TPRM signal engine, trend detection, and the natural language query interface. Start there before touching Phase 3 stories.

### For the ICS / GRC Domain Expert
Review [Client Data Requirements](docs/01-client-data-requirements.md) Sections 1 and 2 to validate that the data model captures all risk and control attributes your methodology requires. Validate the heatmap logic in [Dashboard Feature Specification](docs/05-dashboard-feature-spec.md).

---

## Repository Structure

```
AIDataProcessing/
├── README.md                          ← This file
├── docs/
│   ├── 01-client-data-requirements.md
│   ├── 02-ai-infrastructure.md
│   ├── 03-roadmap.md
│   ├── 04-team-and-roles.md
│   └── 05-dashboard-feature-spec.md
└── templates/
    └── ics_dashboard.html             ← Reference dashboard (single-file build)
```

---

## Document Status

| Document | Version | Status | Last Updated |
|----------|---------|--------|--------------|
| README.md | 1.0 | Final | July 2026 |
| 01-client-data-requirements.md | 1.0 | Final | July 2026 |
| 02-ai-infrastructure.md | 1.0 | Final | July 2026 |
| 03-roadmap.md | 1.0 | Final | July 2026 |
| 04-team-and-roles.md | 1.0 | Final | July 2026 |
| 05-dashboard-feature-spec.md | 1.0 | Final | July 2026 |

---

*Prepared by Accenture Finance & Risk / Technology Strategy practice. For internal engagement use only.*
