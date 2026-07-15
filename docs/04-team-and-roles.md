# Team Composition & Roles

**Document:** 04 — Team Composition & Roles  
**Project:** AI-Powered ICS Dashboard  
**Version:** 1.0  
**Audience:** Programme Manager, Engagement Lead, Accenture Resource Management, Client HR / Project Sponsor

---

## Purpose

This document defines the team structure for the engagement: Accenture roles, seniority levels, FTE allocations across phases, key responsibilities, and the client-side SMEs we require as counterparts. It also includes a RACI matrix for key project decisions and a staffing timeline.

---

## Accenture Team Roles

### Role 1: Programme Manager

| Attribute | Detail |
|-----------|--------|
| **Seniority** | Senior Manager (Level 7) or Managing Director for larger client |
| **Accenture Practice** | Finance & Risk / Accenture Technology — Delivery Management |
| **FTE Allocation** | 1.0 FTE, all phases |
| **Location** | On-site for governance meetings; remote otherwise |

**Key Responsibilities:**
- Overall accountability for delivery quality, timeline, and budget
- Weekly Steering Committee facilitation and reporting
- Client relationship management at project sponsor and C-suite level
- Scope change management: assess impact of scope changes on timeline and budget; present to Steering Committee
- Risk and issue management: maintain risk register, escalate appropriately
- Team performance management: address skill gaps, resource substitutions
- Phase gate management: confirm entry/exit criteria are met before proceeding
- Commercial: track effort actuals against plan; manage change orders

**Required Skills/Experience:**
- Minimum 8 years delivery management in financial services technology
- Prior experience on GRC, risk, or compliance modernisation engagements
- Accenture PM methodology (Accenture Delivery Methods) certified
- Excellent client stakeholder management at C-suite level
- Familiarity with Azure cloud delivery governance

---

### Role 2: ICS / GRC Domain Expert

| Attribute | Detail |
|-----------|--------|
| **Seniority** | Manager or Senior Manager (Level 6–7) |
| **Accenture Practice** | Finance & Risk — Operational Risk / Internal Controls |
| **FTE Allocation** | 1.0 FTE Phases 0–3; 0.5 FTE Phases 4–5 |

**Key Responsibilities:**
- Translate RCSA methodology, ICS framework, and 1LoD/2LoD operating model requirements into functional specifications that engineers can build to
- Validate the risk data model (PostgreSQL schema for risks, controls, assessments) against client's actual GRC methodology
- Quality-check the risk heatmap logic: confirm that the 5x5 likelihood/impact matrix, rating thresholds (Critical ≥15, High ≥9, Medium ≥4), and colour coding align with the client's approved risk appetite framework
- Review and approve AI agent test query set and evaluate answer quality in Phase 3
- Serve as SME for regulatory mapping: DORA, GDPR, CRR/CRD references in the risk register
- Validate that TPRM signal logic correctly maps vendor criticality changes to appropriate risk categories
- Interface with client's Head of Operational Risk and 2LoD risk management team throughout delivery
- Review user training materials for technical accuracy

**Required Skills/Experience:**
- 6+ years in operational risk, internal controls, or GRC within financial services
- Hands-on experience with at least one GRC platform (ServiceNow GRC, Archer RSA, or MetricStream)
- Deep understanding of RCSA methodology, 3LoD model, and internal control frameworks (COSO, ISO 31000)
- Knowledge of DORA, GDPR, and EBA operational resilience requirements
- Preferred: prior engagement delivering a GRC dashboard or ICS modernisation project

---

### Role 3: Solution Architect

| Attribute | Detail |
|-----------|--------|
| **Seniority** | Senior Manager / Architect (Level 7) |
| **Accenture Practice** | Technology Architecture — Cloud / Azure |
| **FTE Allocation** | 1.0 FTE Phases 0–2; 0.5 FTE Phases 3–5 |

**Key Responsibilities:**
- Own the end-to-end technical architecture (doc 02)
- Lead the Architecture Decision Record process in Phase 0
- Define all Azure resource configurations in Terraform
- Design the API contract (endpoint catalogue, request/response schemas, versioning strategy)
- Set coding standards, code review process, and CI/CD pipeline configuration
- Integration design: GRC API connector patterns, TPRM event bus, Azure AD SSO
- Oversee security architecture: Private Endpoints, NSG rules, RBAC, Key Vault
- Technical escalation point for Data Engineers, Backend, and AI/ML teams during implementation
- Production sizing recommendation (Phase 5)

**Required Skills/Experience:**
- Azure Solutions Architect Expert certification (AZ-305) or equivalent experience
- 8+ years in cloud architecture, specifically Azure
- Experience designing and deploying Azure PaaS services: Container Apps, PostgreSQL Flexible Server, Azure Functions, Service Bus, API Management
- Understanding of Azure AI Services (OpenAI, AI Search)
- Prior experience with FastAPI, Python microservices architecture
- Security architecture: Azure Private Endpoints, NSG, Key Vault, Azure AD app registration
- Preferred: experience with LangChain or Azure AI SDK

---

### Role 4: Data Engineers (x2)

| Attribute | Detail |
|-----------|--------|
| **Seniority** | Consultant or Senior Consultant (Level 4–5) |
| **Accenture Practice** | Data & AI — Data Engineering |
| **FTE Allocation** | 2.0 FTE Phases 1–2; 1.0 FTE Phases 3–5 (one engineer transitions to AI support) |

**Key Responsibilities (both engineers, divided by specialisation):**

*Data Engineer 1 — GRC Connectors and ETL:*
- Build and maintain GRC platform connectors (connector pattern per doc 02 Section 2.1)
- Design and implement the Azure Data Factory pipeline architecture
- Implement the watermark-based incremental ingestion pattern
- Implement data validation gate (validation rules, rejection logging, data quality alerting)
- Build dbt models for staging-to-core transformation layer
- Monitor pipeline health; resolve ingestion failures
- Historical data migration: load 2 years of risk rating snapshots

*Data Engineer 2 — Database and Document Ingestion:*
- Design and deploy the PostgreSQL schema (DDL, indexes, constraints)
- Build and maintain the document ingestion pipeline (blob trigger → parse → chunk → embed → index)
- Manage Azure AI Search index (schema, scoring profile, indexer configuration)
- TPRM feed integration: vendor registry connector, change event detection
- Data quality reporting: automated quality checks, weekly data quality dashboards (admin view)
- Database performance: query optimisation, explain plans, index maintenance

**Required Skills/Experience:**
- Python (3.10+): pandas, SQLAlchemy, pydantic, requests
- Azure Data Factory: pipeline authoring, linked services, integration runtime
- PostgreSQL: schema design, query optimisation, dbt
- Azure services: Blob Storage, Service Bus, Functions
- dbt (data build tool): model development, testing, documentation
- For DE 2: Azure AI Search or OpenSearch; familiarity with embedding models
- Preferred: experience with GRC platform APIs (ServiceNow Table API, Archer REST API)

---

### Role 5: AI/ML Engineers (1.0–1.5 FTE)

| Attribute | Detail |
|-----------|--------|
| **Seniority** | Consultant or Senior Consultant (Level 4–5), at least one Senior |
| **Accenture Practice** | Data & AI — AI/ML Engineering |
| **FTE Allocation** | 0.5 FTE Phase 1 (document ingestion support); 1.5 FTE Phases 3–4; 1.0 FTE Phase 5 |

**Key Responsibilities:**
- Design and build the LangChain-based LLM agent (chat panel, tool use, RAG)
- Build the POST /api/agent/chat endpoint with SSE streaming
- Prompt engineering: iterate on system prompt quality with ICS Domain Expert
- Implement the TPRM Signal Engine: Service Bus subscriber, LLM classification, JSON structured output parsing
- Implement the trend detection algorithm (scipy linear regression on residual score history)
- Build the Natural Language Query interface (Text-to-SQL mode and semantic search mode)
- Agent quality evaluation: design and execute test query set, document accuracy metrics
- LLM safety controls: output validation, prompt injection prevention, context bounding per user role
- Document ingestion embedding pipeline (Phase 1 support)
- Post-Phase 3: monitor agent performance in production; iterate on prompt quality based on feedback

**Required Skills/Experience:**
- Python: LangChain (0.2+), OpenAI SDK, pydantic, FastAPI
- Azure OpenAI Service: deployment configuration, API usage, structured output mode
- RAG patterns: chunking strategies, hybrid search, retrieval evaluation
- Text-to-SQL: prompt design, SQL validation, guardrails
- Azure AI Search: index design, hybrid search configuration, semantic ranking
- Prompt engineering: system prompt design, few-shot examples, chain-of-thought
- Preferred: experience with Azure Durable Functions or event-driven architectures
- Preferred: familiarity with GRC or risk domain terminology

---

### Role 6: Backend Developers (x2)

| Attribute | Detail |
|-----------|--------|
| **Seniority** | Consultant or Senior Consultant (Level 4–5) |
| **Accenture Practice** | Technology — Full Stack / Backend Development |
| **FTE Allocation** | 0.5 FTE Phase 1 (API skeleton); 2.0 FTE Phases 2–4; 1.0 FTE Phase 5 |

**Key Responsibilities:**

*Backend Developer 1 — Core API:*
- Build and maintain all REST API endpoints (FastAPI)
- Implement Azure AD authentication middleware and JWT validation
- Implement RBAC enforcement layer (role extraction, row-level filtering)
- Redis caching layer: cache management, TTL strategy, cache invalidation on data updates
- API performance optimisation
- OpenAPI specification documentation (auto-generated by FastAPI + manual descriptions)
- Unit and integration tests for all endpoints (pytest)

*Backend Developer 2 — Workflow and Notifications:*
- Build Azure Durable Functions for RCSA lifecycle orchestration
- Implement 1LoD task engine: task creation, reminder scheduling, escalation logic
- Build notification service: email templates (Azure Communication Services), Teams webhook
- Implement POST /api/assessments/start, /api/notifications/remind, /escalate endpoints
- Implement PUT /api/risks/{id} update endpoint with audit logging
- End-to-end workflow testing
- Production monitoring: Application Insights custom events, alert rules

**Required Skills/Experience:**
- Python: FastAPI, SQLAlchemy (async), pydantic, pytest, asyncio
- PostgreSQL: query writing, ORM patterns (SQLAlchemy), async DB access
- Azure: Container Apps, Functions/Durable Functions, Service Bus, Redis, Key Vault
- REST API design: versioning, pagination, filtering, error handling patterns
- Azure AD: OAuth2/OIDC, JWT, app roles
- For Backend Dev 2: Azure Communication Services or SendGrid; Microsoft Teams webhooks or Power Automate

---

### Role 7: Frontend Developers (1.0–1.5 FTE)

| Attribute | Detail |
|-----------|--------|
| **Seniority** | Consultant or Senior Consultant (Level 4–5); at least one senior |
| **Accenture Practice** | Technology — Frontend / Full Stack |
| **FTE Allocation** | 0.5 FTE Phase 1 (static dashboard); 1.5 FTE Phases 2–3; 1.0 FTE Phase 4; 0.5 FTE Phase 5 |

**Key Responsibilities:**
- Build and maintain the React SPA dashboard (TypeScript)
- Implement all interactive elements per the feature specification in doc 05
- Connect dashboard to live API endpoints with proper loading/error/success state handling
- Implement the risk heatmap component (5x5 matrix, inherent/residual toggle, click navigation)
- Implement the RCSA timeline view
- Implement the agent chat panel with SSE streaming (typing indicator, response rendering)
- Implement TPRM signal alert and review panel
- Implement all workflow action buttons (Send reminder, Start assessment, Accept TPRM signal)
- Responsive design (desktop-primary; tablet-compatible)
- Accessibility: WCAG 2.1 AA compliance
- Azure Static Web Apps deployment configuration

**Required Skills/Experience:**
- React 18 (TypeScript): hooks, context, component patterns
- State management: Zustand or React Query (TanStack Query) for server state
- REST API integration: fetch/axios, error handling, loading states
- SSE (Server-Sent Events): EventSource API
- Data visualisation: D3.js or Recharts for heatmap and charts
- Azure Static Web Apps: deployment, configuration routes, authentication integration
- WCAG 2.1 AA: semantic HTML, ARIA labels, keyboard navigation
- Preferred: experience with enterprise dashboards / data-heavy UIs

---

### Role 8: Security Architect

| Attribute | Detail |
|-----------|--------|
| **Seniority** | Manager or Senior Manager (Level 6–7) |
| **Accenture Practice** | Accenture Security — Cloud Security |
| **FTE Allocation** | 0.5 FTE Phase 0 (architecture review); 0.25 FTE Phases 1–4; 0.75 FTE Phase 5 (pen test oversight) |

**Key Responsibilities:**
- Phase 0: Review and sign off the security architecture (Private Endpoints, NSG rules, RBAC design, secrets management)
- Threat modelling: assess the LLM agent and TPRM signal engine for prompt injection, data exfiltration, privilege escalation risks
- Azure Defender for Cloud: configure security policies, remediate findings
- Phase 5: Coordinate and scope external penetration test; review findings; validate remediation
- DPIA contribution: data flow mapping, security controls for LLM processing of personal data
- Azure Key Vault governance: key rotation policy, access policy review
- Security training for development team: secure coding practices for Python/Azure

**Required Skills/Experience:**
- Azure security architecture: Private Endpoints, NSG, Azure Defender for Cloud, Key Vault
- OWASP Top 10 and API security (OWASP API Security Top 10)
- Identity security: Azure AD, OAuth2/OIDC, Conditional Access
- Penetration test management: scoping, finding classification, remediation validation
- Preferred: LLM/AI security: prompt injection, OWASP LLM Top 10
- Preferred: CISSP, CCSP, or Azure Security Engineer (AZ-500) certification

---

### Role 9: Change Management / Adoption Lead

| Attribute | Detail |
|-----------|--------|
| **Seniority** | Senior Consultant or Manager (Level 5–6) |
| **Accenture Practice** | Accenture Song / Talent & Organisation — Change Management |
| **FTE Allocation** | 0.25 FTE Phase 0–2 (awareness communications); 0.5 FTE Phases 3–4; 1.0 FTE Phase 5 |

**Key Responsibilities:**
- Develop and execute a change management plan addressing the shift from existing GRC tools to the new dashboard
- Stakeholder analysis: map all affected user groups (1LoD process owners across 6 areas, 2LoD risk managers, senior leadership)
- Communications plan: project announcements, milestone communications, go-live communications
- Resistance identification: conduct pulse surveys or short interviews with 1LoD users ahead of UAT to identify adoption barriers
- Training programme: design and deliver role-based training sessions (1LoD session ~30 minutes; 2LoD session ~60 minutes)
- Training materials: recorded walkthrough videos, written quick-reference guides, FAQ document
- UAT facilitation support: brief UAT participants, gather feedback systematically
- Go-live support: on-site or on-call during first two weeks post-launch (hypercare)
- Adoption metrics: define and track adoption KPIs (login frequency, agent chat usage, reminder response rates)

**Required Skills/Experience:**
- Prosci ADKAR or equivalent change management methodology
- Experience managing change for enterprise technology deployments in financial services
- Facilitation: workshop design and delivery, training delivery
- Communication design: ability to write clearly for non-technical audiences
- Preferred: experience with GRC or risk tool rollouts where business users have low technical affinity

---

## Client-Side SMEs Required

The Accenture team cannot deliver this project without active, available counterparts at the client. The following client-side resources must be committed before Phase 0 begins.

| Role | Who at Client | Time Commitment | Key Contributions |
|------|--------------|-----------------|-------------------|
| **GRC System Admin** | IT / GRC Platform team | 2–3 days/week Phase 0–1; 1 day/week thereafter | API access, GRC data exports, sandbox environment access, data model documentation |
| **IT / Azure Lead** | Client IT / Cloud team | 1–2 days/week Phase 0; 1 day/week thereafter | Azure subscription access, network config (Private Endpoints, NSG), CI/CD permissions, Azure AD app registration |
| **Risk Management SME (2LoD)** | Head of Operational Risk or delegate | 3 days/week Phases 0–1; 2 days/week Phases 2–3; intensive UAT Phase 5 | Data model validation, heatmap logic sign-off, RCSA methodology input, AI agent test queries, UAT participation |
| **Data Protection Officer (DPO)** | Client DPO or privacy counsel | 1 day/week Phases 0–1 (DPIA); review in Phase 3 | DPIA review and sign-off, data minimisation guidance, consent/legal basis confirmation for LLM processing |
| **Project Sponsor** | CFO / CRO / COO (depending on client structure) | Steering Committee attendance bi-weekly; escalation availability | Phase gate approvals, scope change decisions, budget sign-off, organisational escalation path |
| **1LoD Champion** | Business manager from one of the 6 areas | 2–3 days intensive in Phase 4–5 | UAT for 1LoD user journeys, training feedback, adoption champion post go-live |
| **Communications Lead** | Client communications / internal comms | 1 day/week Phase 5 | Internal communications for go-live announcement |

---

## RACI Matrix

**Roles:**
- PM = Programme Manager (Accenture)
- GRC = ICS/GRC Domain Expert (Accenture)
- SA = Solution Architect (Accenture)
- DE = Data Engineers (Accenture)
- AI = AI/ML Engineer (Accenture)
- BE = Backend Developer (Accenture)
- FE = Frontend Developer (Accenture)
- SEC = Security Architect (Accenture)
- CM = Change Management Lead (Accenture)
- SP = Client Project Sponsor
- RM = Client Risk Management SME
- IT = Client IT/Azure Lead
- DPO = Client Data Protection Officer

| Decision / Deliverable | PM | GRC | SA | DE | AI | BE | FE | SEC | CM | SP | RM | IT | DPO |
|-----------------------|----|----|----|----|----|----|----|----|----|----|----|----|-----|
| Architecture ADR sign-off | A | C | R | I | I | I | I | C | - | I | I | C | - |
| Data model sign-off (DB schema) | I | A | C | R | I | I | - | - | - | - | C | I | - |
| GRC connector approach selection | I | C | A | R | - | - | - | - | - | - | I | C | - |
| AI model selection (GPT-4o vs Claude) | I | C | A | - | R | - | - | C | - | C | I | I | I |
| Heatmap logic and thresholds | I | A | I | - | - | - | C | - | - | - | R | - | - |
| TPRM signal logic sign-off | I | A | C | I | R | - | - | - | - | - | C | - | - |
| DPIA sign-off | I | C | C | - | C | - | - | C | - | - | - | - | A/R |
| Phase gate approvals (exit criteria) | A | C | C | I | I | I | I | I | I | R | C | C | - |
| UAT sign-off | A | C | I | - | - | I | I | - | C | R | R | - | - |
| Go-live approval | A | C | C | - | - | - | - | C | C | R | C | C | I |
| Scope change decisions | A | C | C | I | I | I | I | I | C | R | C | I | - |
| Training materials sign-off | I | C | - | - | - | - | - | - | A/R | I | C | - | - |
| Production sizing approval | I | - | A | C | - | C | - | - | - | R | - | C | - |

*R = Responsible, A = Accountable, C = Consulted, I = Informed*

---

## Staffing Timeline

The table below shows which roles are active in each phase, and the FTE level (H = 1.0 FTE, M = 0.5 FTE, L = 0.25 FTE, - = not engaged).

| Role | Phase 0 (W1–4) | Phase 1 (W5–12) | Phase 2 (W9–16) | Phase 3 (W13–20) | Phase 4 (W17–24) | Phase 5 (W21–28) |
|------|----------------|-----------------|-----------------|------------------|------------------|------------------|
| Programme Manager | H | H | H | H | H | H |
| ICS / GRC Domain Expert | H | H | H | H | M | M |
| Solution Architect | H | H | H | M | M | M |
| Data Engineer 1 (GRC/ETL) | L | H | M | L | L | - |
| Data Engineer 2 (DB/Docs) | L | H | M | M | L | L |
| AI/ML Engineer | - | L | L | H | H | M |
| Backend Developer 1 (API) | - | L | H | H | H | M |
| Backend Developer 2 (Workflow) | - | L | M | M | H | M |
| Frontend Developer 1 | - | M | H | H | M | L |
| Frontend Developer 2 | - | - | M | H | H | M |
| Security Architect | M | L | L | L | L | M |
| Change Management Lead | - | L | L | L | M | H |
| **Total Accenture FTE** | **~3.5** | **~7.5** | **~8.0** | **~8.0** | **~8.5** | **~6.0** |

**Total estimated Accenture effort:** ~200 FTE-weeks across all phases.  
**Peak team size:** 10–11 Accenture individuals active simultaneously in Phases 2–4.

---

## Skill Development Notes

For Accenture staff new to GRC domain or financial services:
- ICS Domain Expert to run a 2-hour domain onboarding session in Week 1 for all technical team members covering: what an RCSA is, how the 3LoD model works, what risk heatmaps represent, and key regulatory obligations (DORA, GDPR) in scope
- All team members to complete Accenture's mandatory "AI Ethical and Responsible Use" training before starting AI/ML work
- Data Engineers without prior GRC platform experience to complete vendor's API documentation walkthrough with the client GRC System Admin in Week 1

---

*End of document. Next: [05-dashboard-feature-spec.md](05-dashboard-feature-spec.md)*
