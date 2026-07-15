# Implementation Roadmap

**Document:** 03 — Implementation Roadmap  
**Project:** AI-Powered ICS Dashboard  
**Version:** 1.0  
**Audience:** Programme Manager, Engagement Lead, Client Steering Committee

---

## Overview

The project is structured into six phases spanning 28 weeks (approximately 7 months). Phases 1 and 2, and Phases 3 and 4, overlap deliberately to allow development to begin on later components while earlier foundational work is completing. The critical dependency chain is:

```
Phase 0 (Discovery) → Phase 1 (Data Foundation) → Phase 2 (Core Dashboard)
                                                  ↘
                                                    Phase 3 (AI Layer) → Phase 4 (Workflow & Actions) → Phase 5 (Hardening & Rollout)
```

No phase starts before the preceding phase's entry criteria are met. Entry criteria are defined for each phase below.

---

## Phase 0 — Discovery & Setup (Weeks 1–4)

### Objectives
Establish the project foundation: understand the client's existing GRC landscape, validate data availability, agree the technical architecture, and provision the development environment. No code is written beyond infrastructure-as-code and skeleton project setup.

### Entry Criteria
- Client engagement letter signed
- Programme Manager and Solution Architect available
- Client project sponsor identified and accessible

### Activities

**Week 1: Kickoff and GRC System Assessment**
- Project kickoff workshop with client stakeholders (Risk Management, IT, DPO)
- Walkthrough of the Data Readiness Checklist (doc 01) with client GRC System Admin
- Live demo of the GRC platform's API: confirm available endpoints, authentication method, API limits
- Document GRC platform version and any upgrade plans
- Identify all modules in scope (Risk, Controls, RCSA, Findings)

**Week 2: Data Audit**
- Extract sample data sets (100–500 records) from each GRC entity for quality assessment
- Assess data quality against minimum criteria defined in doc 01
- Complete TPRM tool assessment: confirm vendor registry API, change event capability
- Assess Azure AD configuration: confirm tenant, groups, MFA status
- Assess document corpus: inventory all policy documents, identify gaps vs doc 01 Section 4

**Week 3: Architecture Design and Sign-off**
- Draft Architecture Decision Record (ADR) based on doc 02
- Present architecture to client IT Lead and CISO for review
- Key decisions to make: GRC connector type (API vs CSV), vector DB choice (AI Search vs pgvector), LLM provider (Azure OpenAI GPT-4o), data residency region
- Solution Architect finalises infrastructure-as-code (Terraform) based on approved architecture
- Draft DPIA with client DPO (required before AI agent processes personal data)

**Week 4: Environment Provisioning and Project Charter**
- Terraform apply: provision all Azure resources in non-production subscription
- Confirm CI/CD pipeline (Azure DevOps or GitHub Actions) with client IT
- Project Charter finalised: scope, timeline, governance, RACI, escalation path
- Steering Committee structure agreed (cadence, attendees, decision rights)
- Development team onboarding complete (access to client Azure, GRC sandbox, document repository)

### Deliverables

| Deliverable | Owner | Due |
|-------------|-------|-----|
| Data Readiness Report | Data Engineer + ICS Domain Expert | Week 4 |
| Architecture Decision Record (ADR) | Solution Architect | Week 3 |
| Infrastructure-as-Code (Terraform, all resources) | Solution Architect | Week 4 |
| Project Charter | Programme Manager | Week 4 |
| DPIA draft | Accenture + Client DPO | Week 4 |
| Development environment operational | Solution Architect + Data Engineer | Week 4 |

### Phase Exit Criteria
- Data Readiness Report accepted by client
- ADR signed off by client IT Lead and Accenture Solution Architect
- All development environment access provisioned and tested
- Project Charter signed by client project sponsor

---

## Phase 1 — Data Foundation (Weeks 5–12)

### Objectives
Build the data ingestion pipelines, database schema, and document ingestion pipeline. Load historical risk and control data. Produce a static (hardcoded/seeded) version of the dashboard to validate data quality and agree on the visual design with the client before connecting live APIs.

### Entry Criteria
- Phase 0 exit criteria met
- At least 60% of Critical items in Data Readiness Checklist received and validated
- GRC sandbox API credentials issued to Accenture

### Activities

**Weeks 5–6: Database Schema and GRC Connectors**
- Deploy PostgreSQL schema (Section 3.1 of doc 02)
- Build GRC connector for the client's specific platform (ServiceNow/Archer/etc.)
- Implement incremental ingestion pattern (watermark table, delta fetch)
- Implement data validation gate (Section 2.5 of doc 02)
- Initial data load: Risk Register, Control Register, Process Inventory into staging tables

**Weeks 7–8: ETL Maturation and Historical Data**
- Build ADF pipelines for all remaining entities (RCSA, Findings, Assessments, Users)
- Load 2 years of historical risk rating snapshots (for trend detection in Phase 3)
- Build dbt transformation layer (staging → core tables)
- Data quality report: validation failure rates, missing data by entity
- TPRM connector: Vendor Registry and Assessment Results

**Weeks 9–10: Document Ingestion Pipeline**
- Deploy Azure Blob Storage containers and document ingestion Azure Function
- Build document parser (PDF, Word, Excel)
- Build chunking and embedding pipeline (Azure OpenAI text-embedding-3-large)
- Index first batch of documents into Azure AI Search
- Test hybrid search quality with sample queries

**Weeks 11–12: Static Dashboard and Design Validation**
- Deploy static dashboard (hardcoded data matching real client data structure)
- Present to client risk management team: validate visual design, navigation, heatmap logic
- Collect feedback; agree final design before live API connection
- Performance baseline: validate DB query times on full data set

### Deliverables

| Deliverable | Owner | Due |
|-------------|-------|-----|
| Working GRC data pipelines (all entities) | Data Engineer | Week 10 |
| Data quality report (per entity) | Data Engineer + ICS Domain Expert | Week 10 |
| Document ingestion pipeline operational | Data Engineer + AI/ML Engineer | Week 11 |
| Static dashboard (design validation) | Frontend Developer | Week 11 |
| Client design sign-off | Programme Manager + ICS Domain Expert | Week 12 |

### Phase Exit Criteria
- All Critical data entities loaded and passing quality validation
- Static dashboard reviewed and signed off by client
- ADF pipelines running on schedule without failures for 5 consecutive days

---

## Phase 2 — Core Dashboard (Weeks 9–16)

*Note: Phase 2 starts in Week 9, overlapping with Phase 1 completion. The frontend and backend teams work in parallel with the data team.*

### Objectives
Connect the dashboard to live API endpoints. Deliver a fully functional read-only dashboard showing real data: KPI cards, risk heatmaps, RCSA timeline, area and process drill-down, control effectiveness indicators. No AI features yet.

### Entry Criteria
- Core PostgreSQL schema deployed and seeded with production-quality data (from Phase 1)
- Static dashboard design approved (Phase 1 output)

### Activities

**Weeks 9–10: API Foundation**
- Deploy FastAPI application skeleton (Container Apps)
- Implement Azure AD authentication middleware
- Implement RBAC role extraction from JWT claims
- Build GET /api/overview endpoint
- Build GET /api/areas and GET /api/areas/{id} endpoints
- Build GET /api/processes/{id} endpoint
- Redis caching layer for overview and area endpoints (5-minute TTL)

**Weeks 11–12: Dashboard Connected to Live Data**
- Connect frontend to live API (replace hardcoded data)
- KPI cards rendering live counts
- Area cards with live risk summary stats
- Risk heatmap with real data (inherent and residual toggle)
- Area-level risk table

**Weeks 13–14: Process Detail and RCSA Timeline**
- Build GET /api/risks and GET /api/controls endpoints with full filter support
- Process detail view: risk table, control table, expandable panels
- Build GET /api/rcsa/timeline endpoint
- RCSA timeline view: upcoming due dates, status indicators
- Overdue RCSA highlighting

**Weeks 15–16: Polish, Filters, and Performance**
- Cross-area risk view (Overview tab — all areas summary)
- Heatmap scope toggle (by area, by process)
- Risk register sortable/filterable table
- Performance testing: all API endpoints respond in < 500ms under simulated load (50 concurrent users)
- Accessibility review (WCAG 2.1 AA minimum)

### Deliverables

| Deliverable | Owner | Due |
|-------------|-------|-----|
| Live dashboard v1.0 (read-only, real data, no AI) | Backend + Frontend | Week 16 |
| All core API endpoints tested (unit + integration) | Backend Developer | Week 16 |
| Performance test report | Backend Developer + DevOps | Week 16 |

### Phase Exit Criteria
- Dashboard v1.0 demo'd to client and accepted
- All API endpoints return correct data validated by ICS Domain Expert
- Performance targets met under simulated load

---

## Phase 3 — AI Layer (Weeks 13–20)

*Note: Phase 3 starts in Week 13, overlapping with Phase 2 completion.*

### Entry Criteria
- Document ingestion pipeline producing quality search results (Phase 1 output)
- At least one area's risk and process data fully loaded in DB

### Activities

**Weeks 13–14: LLM Agent Foundation**
- Azure OpenAI Service resource deployed (GPT-4o, text-embedding-3-large)
- Build agent base: LangChain agent with tool use
- Implement SQLQueryTool (text-to-SQL with read-only guardrails)
- Implement DocumentRAGTool (hybrid search against AI Search)
- Build POST /api/agent/chat endpoint with SSE streaming
- Agent chat panel in frontend with streaming response rendering

**Weeks 15–16: Agent Quality and Context Injection**
- System prompt engineering: iterate on prompt quality with ICS Domain Expert
- Context injection: current view, area risks, process risks injected per session
- Conversation history management (rolling 10-turn window)
- Agent audit logging (agent_audit_log table)
- Quick-prompt suggestions (hardcoded starting prompts on chat panel)

**Weeks 17–18: TPRM Signal Engine**
- Build TPRM event Azure Function (Service Bus subscriber)
- Implement TPRMSignalEngine class (Section 4.2 of doc 02)
- LLM-based risk suggestion generation and JSON parsing
- Build GET /api/tprm/signals endpoint
- TPRM signal review UI: alert banner on Overview, review panel
- Accept/reject signal workflow: PUT /api/tprm/signals/{id}/accept and /reject

**Weeks 19–20: Trend Detection and RCSA Monitor**
- Build trend detection scheduled Azure Function (weekly run)
- Visualise trend arrows on risk cards (increasing/decreasing/stable)
- Build RCSA due-date monitor Azure Function (daily run)
- Overdue RCSA alert visual in dashboard
- TPRM integration end-to-end test with client TPRM test environment

### Deliverables

| Deliverable | Owner | Due |
|-------------|-------|-----|
| Dashboard v2.0 with AI agent chat panel | Backend + Frontend + AI/ML | Week 18 |
| TPRM signal engine operational | AI/ML Engineer | Week 19 |
| Trend detection operational | AI/ML Engineer | Week 20 |
| Agent quality evaluation report (sample query set, accuracy assessment) | AI/ML Engineer + ICS Domain Expert | Week 20 |
| DPIA finalised and signed | Client DPO + Accenture | Week 18 |

### Phase Exit Criteria
- AI agent correctly answers 80%+ of a defined test query set (assessed by ICS Domain Expert)
- TPRM signal engine successfully generates draft risk from test vendor classification change
- DPIA signed by client DPO

---

## Phase 4 — Workflow & Actions (Weeks 17–24)

*Note: Phase 4 starts in Week 17, overlapping with Phase 3.*

### Entry Criteria
- Dashboard v2.0 accepted (Phase 3)
- Notification infrastructure (Azure Communication Services) provisioned

### Activities

**Weeks 17–18: Notification Infrastructure and 1LoD Task Engine**
- Azure Communication Services Email: configure sender domain, email templates
- Teams webhook integration
- Build workflow_tasks table management service
- Build POST /api/notifications/remind, /remind-all, /escalate endpoints
- Manual "Send all reminders" and individual reminder buttons in frontend

**Weeks 19–20: Durable Functions — RCSA Lifecycle Orchestration**
- Implement RCSA Assessment Durable Function workflow (Section 4.5 of doc 02)
- Auto-reminder scheduling (D-14, D-3 before due date)
- Auto-escalation trigger when D-3 passed with no action
- Build POST /api/assessments/start endpoint
- "Start ICS Assessment" button in process detail view

**Weeks 21–22: Risk Update Workflow and 1LoD Input Form**
- Build PUT /api/risks/{id} endpoint
- Risk update form in UI (inline edit panel): status update, notes, residual rating update
- 2LoD review and comment workflow on RCSA submissions
- Findings creation from RCSA: ability to raise finding from within the assessment view

**Weeks 23–24: RBAC Enforcement and Notification Polish**
- Full RBAC enforcement across all endpoints (1LoD restriction to own area/process)
- Role-based UI rendering (hide actions unauthorised for current user role)
- Notification audit trail view for 2LoD (who was reminded, when, how many times)
- End-to-end workflow test: complete RCSA cycle from assignment through 2LoD sign-off
- Integration test: Teams notifications delivered correctly

### Deliverables

| Deliverable | Owner | Due |
|-------------|-------|-----|
| Dashboard v3.0 — full interactive system | All teams | Week 24 |
| Notification system operational (email + Teams) | Backend Developer | Week 22 |
| RCSA lifecycle workflow operational | Backend Developer | Week 22 |
| RBAC fully enforced | Backend Developer + Solution Architect | Week 24 |
| End-to-end integration test report | All teams | Week 24 |

### Phase Exit Criteria
- Complete RCSA lifecycle tested end-to-end in staging environment
- RBAC validated: 1LoD user cannot access unauthorised areas; 2LoD has full access
- Notification delivery confirmed for both email and Teams channels

---

## Phase 5 — Hardening & Rollout (Weeks 21–28)

*Note: Phase 5 starts in Week 21, overlapping with Phase 4's final sprint.*

### Entry Criteria
- Dashboard v3.0 functional in staging environment
- All Phase 4 exit criteria met

### Activities

**Weeks 21–23: Security Review and Penetration Test**
- Security Architect conducts internal security review: network topology, RBAC, secrets management, LLM prompt security
- External penetration test (client-commissioned, Accenture Security coordinates scope)
- Remediate all Critical and High findings from pen test
- Azure Defender for Cloud posture review: resolve any policy violations

**Weeks 22–24: Performance and Load Testing**
- Define load test scenarios (100 concurrent users, peak RCSA submission period)
- Execute load test with Azure Load Testing service
- Optimise: add indexes, tune Redis TTLs, right-size Container Apps replicas
- Document performance test results and production sizing recommendation

**Weeks 24–26: User Acceptance Testing (UAT)**
- UAT participants: 5–10 2LoD risk managers, 5–10 1LoD process owners from each area
- UAT scripts covering all key user journeys (from doc 05)
- Daily defect triage; Critical and High defects fixed within 48 hours
- ICS Domain Expert validates risk heatmap logic and RCSA data accuracy against source GRC system
- Sign-off: UAT exit criteria signed by client Risk Management Lead

**Weeks 25–28: Training, Documentation, and Go-Live**
- User training: recorded video walkthroughs (1LoD session ~30 mins; 2LoD session ~60 mins)
- Admin training: runbook for data pipeline monitoring, adding new documents to RAG corpus
- Operational runbook: incident response, escalation contacts, SLA for bug fixes
- Production environment provisioned (separate Azure subscription, production-grade SKUs)
- Production data load: full GRC data migration to production DB
- Go-live: DNS cutover, monitored hypercare period (2 weeks with Accenture team on standby)
- Hypercare ends: transition to Accenture Managed Services or client IT Ops

### Deliverables

| Deliverable | Owner | Due |
|-------------|-------|-----|
| Security review report + pen test remediation evidence | Security Architect | Week 23 |
| Performance test report + production sizing recommendation | Backend Developer + DevOps | Week 24 |
| UAT sign-off document | Client Risk Management Lead + Programme Manager | Week 26 |
| User training materials (video + written guide) | Change Management Lead | Week 27 |
| Admin and operational runbooks | Solution Architect + Backend Developer | Week 27 |
| Production go-live | All teams | Week 28 |

---

## Gantt Table

| Phase / Activity | W1 | W2 | W3 | W4 | W5 | W6 | W7 | W8 | W9 | W10 | W11 | W12 | W13 | W14 | W15 | W16 | W17 | W18 | W19 | W20 | W21 | W22 | W23 | W24 | W25 | W26 | W27 | W28 |
|------------------|----|----|----|----|----|----|----|----|----|----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
| **Ph0: Discovery** | ██ | ██ | ██ | ██ | | | | | | | | | | | | | | | | | | | | | | | | |
| GRC assessment | ██ | ██ | | | | | | | | | | | | | | | | | | | | | | | | | | |
| Architecture ADR | | | ██ | ██ | | | | | | | | | | | | | | | | | | | | | | | | |
| Env provisioning | | | | ██ | | | | | | | | | | | | | | | | | | | | | | | | |
| **Ph1: Data Foundation** | | | | | ██ | ██ | ██ | ██ | ██ | ██ | ██ | ██ | | | | | | | | | | | | | | | | |
| GRC connectors | | | | | ██ | ██ | | | | | | | | | | | | | | | | | | | | | | |
| ETL pipelines | | | | | | | ██ | ██ | ██ | ██ | | | | | | | | | | | | | | | | | | |
| Doc ingestion | | | | | | | | | ██ | ██ | ██ | | | | | | | | | | | | | | | | | |
| Static dashboard | | | | | | | | | | | ██ | ██ | | | | | | | | | | | | | | | | |
| **Ph2: Core Dashboard** | | | | | | | | | ██ | ██ | ██ | ██ | ██ | ██ | ██ | ██ | | | | | | | | | | | | |
| API foundation | | | | | | | | | ██ | ██ | | | | | | | | | | | | | | | | | | |
| Live dashboard v1 | | | | | | | | | | | ██ | ██ | ██ | ██ | | | | | | | | | | | | | | |
| Process detail / RCSA | | | | | | | | | | | | | ██ | ██ | | | | | | | | | | | | | | |
| Polish & perf | | | | | | | | | | | | | | | ██ | ██ | | | | | | | | | | | | |
| **Ph3: AI Layer** | | | | | | | | | | | | | ██ | ██ | ██ | ██ | ██ | ██ | ██ | ██ | | | | | | | | |
| LLM agent | | | | | | | | | | | | | ██ | ██ | ██ | ██ | | | | | | | | | | | | |
| TPRM engine | | | | | | | | | | | | | | | | | ██ | ██ | | | | | | | | | | |
| Trend detection | | | | | | | | | | | | | | | | | | | ██ | ██ | | | | | | | | |
| **Ph4: Workflows** | | | | | | | | | | | | | | | | | ██ | ██ | ██ | ██ | ██ | ██ | ██ | ██ | | | | |
| Notifications | | | | | | | | | | | | | | | | | ██ | ██ | | | | | | | | | | |
| RCSA workflow | | | | | | | | | | | | | | | | | | | ██ | ██ | | | | | | | | |
| Risk update / RBAC | | | | | | | | | | | | | | | | | | | | | ██ | ██ | ██ | ██ | | | | |
| **Ph5: Hardening** | | | | | | | | | | | | | | | | | | | | | ██ | ██ | ██ | ██ | ██ | ██ | ██ | ██ |
| Security / pen test | | | | | | | | | | | | | | | | | | | | | ██ | ██ | ██ | | | | | |
| Load testing | | | | | | | | | | | | | | | | | | | | | | ██ | ██ | ██ | | | | |
| UAT | | | | | | | | | | | | | | | | | | | | | | | | ██ | ██ | ██ | | |
| Training + go-live | | | | | | | | | | | | | | | | | | | | | | | | | | | ██ | ██ |

---

## Project Risk Register

The following are the top project-level risks. This is distinct from the ICS dashboard's risk register; these are delivery risks for the Accenture engagement team.

| # | Risk | Likelihood | Impact | Severity | Mitigation | Owner |
|---|------|-----------|--------|----------|------------|-------|
| R1 | **Client data quality is significantly worse than anticipated**, causing GRC connector failures, rejections in validation gates, and requiring manual data cleansing effort before Phase 2 can proceed | Medium | High | High | Phase 0 data audit identifies issues early; Data Readiness Report captures all gaps with remediation owners and deadlines; Data Engineers build flexible validation tolerances with warning-not-reject rules for non-critical fields; Static dashboard (Phase 1) de-risks Phase 2 timeline | Data Engineer + Client GRC Admin |
| R2 | **GRC system API access delayed or not available**, forcing reliance on CSV exports instead of real-time polling and degrading the dashboard's freshness | Medium | High | High | CSV fallback connector built in parallel; ETL pipeline designed to accept both API and CSV inputs; APIM caching reduces the impact of lower API refresh frequency; Phase 2 timeline has two-week buffer | Solution Architect + Client IT Lead |
| R3 | **DPIA not approved in time for Phase 3**, blocking the AI agent from processing user-identifying data and processing risk data via LLM prompts | Low-Medium | High | High | DPIA process started in Phase 0 (Week 3); Accenture provides template and drafting support; DPO is identified as a named stakeholder in Project Charter; Agent architecture is designed to minimise PII exposure (user_id rather than names in prompts) reducing DPIA scope | Programme Manager + Client DPO |
| R4 | **Key client SME (GRC domain expert or risk management lead) unavailable during UAT**, resulting in sign-off delays and timeline slip into Phase 5 | Medium | Medium | Medium | UAT participants and backups named in Project Charter; UAT scripts prepared 2 weeks in advance; Asynchronous review mechanism (screen recordings + written comments) available if SME availability is intermittent | Programme Manager + Client Risk Lead |
| R5 | **LLM agent answer quality insufficient for production use**, specifically for complex regulatory questions or gap analysis, leading to client rejection of the AI feature | Low-Medium | High | High | Agent quality evaluation process built into Phase 3 deliverables; 20-question test set defined by ICS Domain Expert at Phase 0; Prompt engineering iterations with domain expert in Weeks 15–16; Fallback option: deploy agent as "beta/preview" feature in v3.0 with explicit disclaimer, with quality improvement continuing post-go-live | AI/ML Engineer + ICS Domain Expert |

---

## Governance Structure

### Steering Committee
- **Frequency:** Bi-weekly
- **Attendees:** Client project sponsor, Client Risk Management Lead, Client CIO/IT Lead, Accenture Programme Manager, Accenture Engagement Lead
- **Purpose:** Phase gate sign-offs, escalation resolution, scope change decisions

### Delivery Team Stand-up
- **Frequency:** Daily (async on Fridays)
- **Attendees:** Full Accenture delivery team
- **Purpose:** Progress, blockers, dependencies

### Technical Review
- **Frequency:** Weekly
- **Attendees:** Solution Architect, Data Engineers, AI/ML Engineers, Backend and Frontend leads, Client GRC System Admin (optional)
- **Purpose:** Technical decisions, integration issues, architecture deviations

### Risk and Issue Log
Maintained by Programme Manager in shared project management tool (Azure DevOps Boards or Jira). Updated weekly. Reviewed at each Steering Committee.

---

*End of document. Next: [04-team-and-roles.md](04-team-and-roles.md)*
