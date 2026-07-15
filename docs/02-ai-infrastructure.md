# AI Infrastructure Architecture

**Document:** 02 — AI Infrastructure Architecture  
**Project:** AI-Powered ICS Dashboard  
**Version:** 1.0  
**Audience:** Solution Architect, Data Engineers, AI/ML Engineers, Security Architect, Client IT Lead

---

## Purpose

This document describes the full technical architecture for the AI-powered ICS Dashboard. It covers every layer from raw data source through to the browser-rendered dashboard, with enough detail to support the Architecture Decision Record (ADR) produced in Phase 0, to guide the infrastructure provisioning in Phase 0, and to drive development in Phases 1–4.

All components are designed to run within the client's Azure tenant. No data leaves the client's cloud boundary.

---

## Section 1 — Architecture Overview

### 1.1 Layered Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL DATA SOURCES                                │
│  GRC Platform (ServiceNow/Archer/etc)  │  TPRM Tool  │  HR/IAM  │  Document Store│
└────────────────────────┬───────────────────────────────────────────────────────────┘
                         │  REST APIs / Scheduled Exports / Event Webhooks
                         ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           INGESTION & ETL LAYER                                   │
│  Azure Data Factory (orchestration)  │  Python connectors  │  Doc ingestion pipeline│
│  Data validation gates  │  Schema normalisation  │  Change-event capture           │
└────────────────────────┬───────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           DATA PLATFORM LAYER                                     │
│  ┌─────────────────────┐  ┌──────────────────────┐  ┌───────────────────────┐   │
│  │ Relational DB        │  │ Vector Database       │  │ Blob / Document Store │   │
│  │ (Azure PostgreSQL)   │  │ (Azure AI Search /    │  │ (Azure Blob Storage)  │   │
│  │ Risks, Controls,     │  │  pgvector extension)  │  │ Raw PDFs, Excel,      │   │
│  │ RCSA, Processes,     │  │ Embeddings over policy│  │ previous reports      │   │
│  │ Assessments, Users   │  │ docs + risk summaries │  │                       │   │
│  └─────────────────────┘  └──────────────────────┘  └───────────────────────┘   │
└────────────────────────┬───────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           AI / ML SERVICES LAYER                                  │
│  LLM Agent (Azure OpenAI GPT-4o or Azure-hosted Anthropic Claude)                │
│  RAG Pipeline (LangChain / Azure AI Search)                                       │
│  TPRM Signal Engine (Rule engine + LLM classification)                            │
│  Trend Detection (Python / statsmodels time-series)                               │
│  RCSA Due-Date Monitor (Scheduled Azure Function)                                 │
│  1LoD Workflow Engine (Azure Durable Functions)                                   │
│  NL Query Interface (Text-to-SQL / semantic search)                               │
└────────────────────────┬───────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           API GATEWAY LAYER                                       │
│  Azure API Management (APIM)                                                      │
│  FastAPI application (containerised, Azure Container Apps)                        │
│  Azure AD OAuth2 / OIDC authentication                                            │
│  Rate limiting, caching (Redis), audit logging                                    │
└────────────────────────┬───────────────────────────────────────────────────────────┘
                         │  HTTPS
                         ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           FRONTEND LAYER                                          │
│  React SPA (TypeScript)                                                           │
│  Hosted on Azure Static Web Apps                                                  │
│  ICS Cockpit Dashboard — heatmaps, RCSA timeline, agent chat, workflow actions    │
│  Users: 1LoD business users, 2LoD risk managers                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Design Principles

1. **Data sovereignty:** All compute and storage resides within the client's Azure tenant in an EU region. No data is transmitted to external APIs.
2. **Separation of concerns:** The ingestion layer is decoupled from the storage layer, which is decoupled from the AI layer. Each can be updated independently.
3. **Event-driven where appropriate:** TPRM signals and RCSA status changes trigger downstream processing without polling. Polling is used only where webhooks are not available.
4. **Defence in depth:** Private Endpoints for all Azure services, RBAC on every resource, audit logging of all AI agent actions.
5. **Incremental delivery:** The data and API layers are designed to support a static dashboard first (Phase 2), with AI components layered on top (Phase 3), so value is delivered before all AI components are ready.

---

## Section 2 — Data Ingestion & ETL

### 2.1 GRC System Connectors

Each GRC system connector is a Python module that handles authentication, pagination, rate-limit backoff, and schema normalisation for a specific platform. Connectors are orchestrated by Azure Data Factory (ADF).

**Connector design pattern:**

```python
class GRCConnector(ABC):
    """Base class for all GRC system connectors."""
    
    def authenticate(self) -> None: ...
    def fetch_entity(self, entity_type: str, since: datetime) -> List[Dict]: ...
    def normalise(self, raw_record: Dict) -> NormalisedRecord: ...
    def validate(self, record: NormalisedRecord) -> ValidationResult: ...
    def upsert(self, session: Session, record: NormalisedRecord) -> None: ...
```

**Connector implementations required:**

| Connector | Platform | Fetch mode | Entities |
|-----------|----------|------------|---------|
| `ServiceNowConnector` | ServiceNow GRC | REST API polling (incremental by sys_updated_on) + webhook for critical events | All entities |
| `ArcherConnector` | RSA Archer | REST API polling (incremental by LastUpdated) | All entities |
| `MetricStreamConnector` | MetricStream | REST API polling | All entities |
| `OpenPagesConnector` | IBM OpenPages | REST API polling | All entities |
| `OneTrustConnector` | OneTrust | REST API polling | GDPR/privacy entities |
| `GenericCSVConnector` | Any (fallback) | Scheduled CSV drop to Blob Storage, ADF pickup | Any entity |

**Incremental ingestion:** All connectors use a watermark table (`etl_watermarks`) to track the last successful fetch timestamp per entity type. On each run, only records modified since the watermark are fetched.

**Full refresh:** A full refresh is triggered weekly for all entities to catch any records that may have been missed due to timestamp issues.

---

### 2.2 ADF Pipeline Architecture

```
ADF Pipeline: ICS_GRC_Incremental_Ingest
├── Activity: Lookup watermark from etl_watermarks
├── Activity: Invoke GRCConnector for each entity type (ForEach, parallel)
│   ├── Web Activity: Fetch from GRC API
│   ├── Data Flow: Normalise + validate schema
│   ├── Sink: Write to PostgreSQL staging tables (prefix: stg_)
│   └── Web Activity: Update watermark on success
├── Activity: Run dbt transformations (staging → core tables)
└── Activity: Trigger downstream AI components (TPRM signal engine, trend detection)
```

**Schedule:** Incremental runs every 15 minutes for critical entities (Risk Register, RCSA status). Every 4 hours for less volatile entities (Control Register, Process Inventory). Full refresh: nightly at 02:00 UTC.

---

### 2.3 Document Ingestion Pipeline

Policy documents, previous RCSA reports, and regulatory texts are ingested into the vector database to power RAG.

**Pipeline steps:**

```
1. Document Upload
   └── User or admin uploads PDF/Word/Excel to Azure Blob Storage (container: ics-documents)

2. Blob Trigger → Azure Function: DocumentIngestionTrigger
   └── Invoked on new blob in ics-documents container

3. Parsing (Azure Function: DocumentParser)
   ├── PDF: PyMuPDF for text extraction, handling multi-column layouts
   ├── Word (.docx): python-docx
   ├── Excel (.xlsx): openpyxl — table extraction only (no chart content)
   └── Metadata extraction: document_title, document_type, version, date, author

4. Chunking (Azure Function: DocumentChunker)
   ├── Strategy: Recursive character text splitter
   ├── Chunk size: 800 tokens (approx. 600 words)
   ├── Overlap: 100 tokens (to preserve cross-chunk context)
   └── Preserve: section headers as metadata on each chunk

5. Embedding (Azure OpenAI: text-embedding-3-large)
   └── 3072-dimensional embeddings per chunk

6. Indexing (Azure AI Search or pgvector)
   ├── Index: ics-document-chunks
   ├── Fields: chunk_id, document_id, document_type, section_header, content, embedding (vector), metadata (JSON)
   └── Hybrid search: keyword (BM25) + vector (cosine) with RRF fusion

7. Document Registry (PostgreSQL: documents table)
   └── Record: document_id, filename, document_type, ingestion_date, chunk_count, status
```

**Supported document types:** PDF, DOCX, XLSX, PPTX (text slides only)  
**Maximum document size:** 50 MB  
**Re-ingestion:** If a document is re-uploaded (same filename, new version), old chunks are deleted and new chunks replace them.

---

### 2.4 TPRM Event-Driven Feed

Rather than batch polling, the TPRM feed uses an event-driven pattern where vendor record changes trigger immediate processing.

**Mechanism:**
- TPRM tool posts a webhook to the TPRM Ingest endpoint when a vendor record changes a monitored field
- If the TPRM tool does not support webhooks, ADF polls the TPRM API every 5 minutes and computes a diff against the last known state
- Change events are published to an Azure Service Bus topic: `tprm-vendor-change-events`
- The TPRM Signal Engine (Section 4.2) subscribes to this topic

**Change event schema:**
```json
{
  "event_id": "evt-2025-0892",
  "event_type": "VENDOR_CLASSIFICATION_CHANGED",
  "vendor_id": "VND-0234",
  "vendor_name": "Acme Cloud Services Ltd",
  "previous_value": "Standard",
  "new_value": "Critical",
  "changed_field": "vendor_classification",
  "changed_at": "2025-07-14T10:32:00Z",
  "changed_by": "user@client.com"
}
```

---

### 2.5 Data Validation and Quality Gates

Every ingestion run passes records through validation before they reach the core tables:

| Validation Rule | Failure Action |
|-----------------|---------------|
| Mandatory fields populated | Reject record; log to `etl_validation_failures`; alert data engineer |
| `process_id` resolves to valid process | Reject record; log warning |
| Date fields are valid dates and not in the future (for historical dates) | Reject with error |
| Controlled vocabulary values match enum | Reject with error |
| Integer ratings are in range 1–5 | Reject with error |
| Duplicate primary keys | Deduplicate (keep most recent by modified_date); log warning |
| Cross-entity consistency (risk rating = likelihood × impact) | Log warning; do not reject |

**Data Quality Dashboard:** A separate lightweight admin view shows validation failure rates, last ingest timestamps, and record counts per entity. This is for the Accenture team's operational use, not the client dashboard.

---

## Section 3 — Data Storage

### 3.1 Relational Database (PostgreSQL on Azure)

**Service:** Azure Database for PostgreSQL — Flexible Server (v16)  
**Tier:** General Purpose, 8 vCores, 32 GB RAM (scale up for production based on load testing)  
**Replication:** Zone-redundant high availability  
**Backup:** Geo-redundant backup, 35-day retention  
**Extensions required:** `pgvector` (optional — used if Azure AI Search is not selected as the vector store), `pg_trgm` (trigram search for fuzzy text matching), `uuid-ossp`

#### Core Schema (abbreviated DDL)

```sql
-- Areas (business areas)
CREATE TABLE areas (
    area_id         VARCHAR(10) PRIMARY KEY,   -- e.g. P-01
    area_name       VARCHAR(100) NOT NULL,
    area_code       VARCHAR(20),
    active          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Processes
CREATE TABLE processes (
    process_id          VARCHAR(20) PRIMARY KEY,   -- e.g. PROC-001
    process_name        VARCHAR(200) NOT NULL,
    process_description TEXT,
    area_id             VARCHAR(10) REFERENCES areas(area_id),
    process_owner_id    VARCHAR(100),              -- FK to users
    lod_designation     VARCHAR(10) CHECK (lod_designation IN ('1LoD','2LoD')),
    criticality         VARCHAR(20) CHECK (criticality IN ('Critical','Important','Standard')),
    rcsa_frequency      VARCHAR(20),
    next_rcsa_due       DATE,
    ict_service_flag    BOOLEAN DEFAULT FALSE,
    tprm_relevant       BOOLEAN DEFAULT FALSE,
    active              BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Risks
CREATE TABLE risks (
    risk_id                 VARCHAR(30) PRIMARY KEY,
    risk_title              VARCHAR(200) NOT NULL,
    risk_description        TEXT,
    process_id              VARCHAR(20) REFERENCES processes(process_id),
    area_id                 VARCHAR(10) REFERENCES areas(area_id),
    risk_category           VARCHAR(50),
    risk_sub_category       VARCHAR(100),
    risk_owner_id           VARCHAR(100),
    status                  VARCHAR(30) CHECK (status IN ('Open','Closed','In Remediation','Accepted','Escalated')),
    inherent_likelihood     SMALLINT CHECK (inherent_likelihood BETWEEN 1 AND 5),
    inherent_impact         SMALLINT CHECK (inherent_impact BETWEEN 1 AND 5),
    inherent_score          SMALLINT GENERATED ALWAYS AS (inherent_likelihood * inherent_impact) STORED,
    inherent_rating         VARCHAR(20) GENERATED ALWAYS AS (
                                CASE
                                  WHEN inherent_likelihood * inherent_impact >= 15 THEN 'Critical'
                                  WHEN inherent_likelihood * inherent_impact >= 9 THEN 'High'
                                  WHEN inherent_likelihood * inherent_impact >= 4 THEN 'Medium'
                                  ELSE 'Low'
                                END
                            ) STORED,
    residual_likelihood     SMALLINT CHECK (residual_likelihood BETWEEN 1 AND 5),
    residual_impact         SMALLINT CHECK (residual_impact BETWEEN 1 AND 5),
    residual_score          SMALLINT GENERATED ALWAYS AS (residual_likelihood * residual_impact) STORED,
    residual_rating         VARCHAR(20) GENERATED ALWAYS AS (
                                CASE
                                  WHEN residual_likelihood * residual_impact >= 15 THEN 'Critical'
                                  WHEN residual_likelihood * residual_impact >= 9 THEN 'High'
                                  WHEN residual_likelihood * residual_impact >= 4 THEN 'Medium'
                                  ELSE 'Low'
                                END
                            ) STORED,
    risk_appetite_breach    BOOLEAN DEFAULT FALSE,
    regulatory_reference    VARCHAR(200),
    last_reviewed_date      DATE,
    next_review_date        DATE,
    source                  VARCHAR(20) DEFAULT 'Manual' CHECK (source IN ('Manual','TPRM-Signal','AI-Suggested','IA-Finding')),
    created_at              TIMESTAMPTZ DEFAULT NOW(),
    updated_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Controls
CREATE TABLE controls (
    control_id              VARCHAR(30) PRIMARY KEY,
    control_title           VARCHAR(200) NOT NULL,
    control_description     TEXT,
    control_type            VARCHAR(20) CHECK (control_type IN ('Preventive','Detective','Corrective')),
    control_category        VARCHAR(30) CHECK (control_category IN ('Manual','Automated','IT-Dependent Manual')),
    process_id              VARCHAR(20) REFERENCES processes(process_id),
    area_id                 VARCHAR(10) REFERENCES areas(area_id),
    control_owner_id        VARCHAR(100),
    effectiveness_rating    VARCHAR(30) CHECK (effectiveness_rating IN ('Effective','Partially Effective','Ineffective','Not Assessed')),
    key_control_indicator   BOOLEAN DEFAULT FALSE,
    test_frequency          VARCHAR(20),
    last_test_date          DATE,
    next_test_date          DATE,
    status                  VARCHAR(20) CHECK (status IN ('Active','Inactive','Under Review')),
    created_at              TIMESTAMPTZ DEFAULT NOW(),
    updated_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Risk-Control mapping (many-to-many)
CREATE TABLE risk_control_map (
    risk_id         VARCHAR(30) REFERENCES risks(risk_id),
    control_id      VARCHAR(30) REFERENCES controls(control_id),
    PRIMARY KEY (risk_id, control_id)
);

-- RCSA Assessments
CREATE TABLE rcsa_assessments (
    assessment_id               VARCHAR(30) PRIMARY KEY,
    assessment_cycle            VARCHAR(20) NOT NULL,  -- e.g. Q1-2025
    process_id                  VARCHAR(20) REFERENCES processes(process_id),
    area_id                     VARCHAR(10) REFERENCES areas(area_id),
    assessment_status           VARCHAR(30) CHECK (assessment_status IN ('Not Started','In Progress','1LoD Complete','2LoD Review','Completed','Overdue')),
    start_date                  DATE,
    due_date                    DATE NOT NULL,
    completion_date             DATE,
    assigned_to                 VARCHAR(100),
    reviewer                    VARCHAR(100),
    overall_risk_rating         VARCHAR(20),
    control_effectiveness_summary VARCHAR(30),
    lod1_comments               TEXT,
    lod2_comments               TEXT,
    findings_count              INTEGER DEFAULT 0,
    open_actions_count          INTEGER DEFAULT 0,
    created_at                  TIMESTAMPTZ DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ DEFAULT NOW()
);

-- Risk rating history (for trend detection)
CREATE TABLE risk_rating_history (
    id                      BIGSERIAL PRIMARY KEY,
    risk_id                 VARCHAR(30) REFERENCES risks(risk_id),
    snapshot_date           DATE NOT NULL,
    inherent_likelihood     SMALLINT,
    inherent_impact         SMALLINT,
    residual_likelihood     SMALLINT,
    residual_impact         SMALLINT,
    assessed_by             VARCHAR(100),
    created_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Workflow tasks (1LoD pending input)
CREATE TABLE workflow_tasks (
    task_id             VARCHAR(30) PRIMARY KEY,
    task_type           VARCHAR(30) CHECK (task_type IN ('RCSA_Input','Control_Test','Risk_Review','Finding_Remediation','TPRM_Review')),
    related_entity_type VARCHAR(20),
    related_entity_id   VARCHAR(30),
    assigned_to         VARCHAR(100) NOT NULL,
    area_id             VARCHAR(10) REFERENCES areas(area_id),
    process_id          VARCHAR(20),
    due_date            DATE NOT NULL,
    status              VARCHAR(20) CHECK (status IN ('Pending','In Progress','Completed','Overdue','Escalated')),
    reminder_count      SMALLINT DEFAULT 0,
    last_reminder_sent  TIMESTAMPTZ,
    escalated_to        VARCHAR(100),
    escalated_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- TPRM signals (pending suggested risks from TPRM engine)
CREATE TABLE tprm_signals (
    signal_id               VARCHAR(30) PRIMARY KEY,
    vendor_id               VARCHAR(30) NOT NULL,
    vendor_name             VARCHAR(200),
    trigger_event           VARCHAR(50),
    trigger_detail          JSONB,
    suggested_risk_title    VARCHAR(200),
    suggested_risk_description TEXT,
    suggested_category      VARCHAR(50),
    suggested_inherent_likelihood SMALLINT,
    suggested_inherent_impact     SMALLINT,
    suggested_controls      JSONB,
    affected_processes      JSONB,
    status                  VARCHAR(20) CHECK (status IN ('Pending','Accepted','Rejected','Modified')),
    reviewed_by             VARCHAR(100),
    reviewed_at             TIMESTAMPTZ,
    created_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Users
CREATE TABLE users (
    user_id         VARCHAR(100) PRIMARY KEY,  -- Azure AD UPN
    display_name    VARCHAR(200),
    email           VARCHAR(200),
    department      VARCHAR(200),
    business_area   VARCHAR(100),
    lod_role        VARCHAR(20) CHECK (lod_role IN ('1LoD','2LoD','3LoD','ReadOnly','Admin')),
    manager_id      VARCHAR(100),
    active          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Agent audit log
CREATE TABLE agent_audit_log (
    id              BIGSERIAL PRIMARY KEY,
    session_id      VARCHAR(50),
    user_id         VARCHAR(100),
    action_type     VARCHAR(30) CHECK (action_type IN ('Chat','RiskUpdate','AssessmentStart','ReminderSent','EscalationSent','TPRMAccept','TPRMReject')),
    input_summary   TEXT,
    output_summary  TEXT,
    context         JSONB,
    duration_ms     INTEGER,
    tokens_used     INTEGER,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

**Key indexes:**
```sql
CREATE INDEX idx_risks_area_id ON risks(area_id);
CREATE INDEX idx_risks_process_id ON risks(process_id);
CREATE INDEX idx_risks_status ON risks(status);
CREATE INDEX idx_risks_residual_rating ON risks(residual_rating);
CREATE INDEX idx_controls_process_id ON controls(process_id);
CREATE INDEX idx_controls_effectiveness ON controls(effectiveness_rating);
CREATE INDEX idx_rcsa_due_date ON rcsa_assessments(due_date);
CREATE INDEX idx_rcsa_status ON rcsa_assessments(assessment_status);
CREATE INDEX idx_workflow_tasks_assigned_due ON workflow_tasks(assigned_to, due_date);
CREATE INDEX idx_tprm_signals_status ON tprm_signals(status);
```

---

### 3.2 Vector Database

**Primary option:** Azure AI Search (managed, hybrid search with BM25 + vector, no operational overhead)  
**Alternative:** pgvector extension on the same PostgreSQL instance (simpler if cost is constrained)

**Index design (Azure AI Search):**

```
Index: ics-document-chunks
Fields:
  chunk_id          (String, key)
  document_id       (String, filterable)
  document_type     (String, filterable, facetable)  -- Policy, Regulation, RCSA, Procedure
  document_title    (String, searchable)
  section_header    (String, searchable)
  content           (String, searchable)
  embedding         (Collection(Edm.Single), searchable, dimensions=3072, algorithm=hnsw)
  metadata          (Complex field: version, date, source_url, area_relevance)

Scoring profile:
  Hybrid: 60% vector score + 40% BM25 keyword score (Reciprocal Rank Fusion)

Semantic ranking:
  Enabled (Azure AI Search semantic ranker) for top-K reranking
```

---

### 3.3 Blob / Document Store

**Service:** Azure Blob Storage  
**Container structure:**

```
ics-documents/
├── policies/           ← Internal policy documents
├── procedures/         ← SOPs and procedure manuals
├── regulations/        ← Regulatory texts
├── rcsa-reports/       ← Historical RCSA reports
├── audit-findings/     ← IA findings
└── staging/            ← Raw GRC exports (temporary)

ics-exports/            ← Dashboard exports (Excel/PDF reports generated for users)
ics-evidence/           ← Control test evidence files
```

**Access:** All containers use Azure Private Endpoints. No public blob access. SAS tokens generated per-session with 1-hour expiry for evidence file access.

---

## Section 4 — AI/ML Components

### 4.1 LLM Agent (Chat Panel)

**Purpose:** Powers the conversational chat panel in the dashboard sidebar. Allows 2LoD risk managers and 1LoD users to ask natural language questions about the risk register, request gap summaries, understand regulatory context, and get explanations of findings.

**Model:** Azure OpenAI GPT-4o (deployed within client Azure tenant via Azure OpenAI Service resource) or Claude claude-sonnet-5 deployed via Azure AI Studio if Anthropic partnership is in place. The architecture is model-agnostic; swap the provider by changing the API endpoint configuration.

**Architecture:** LangChain-based RAG agent with tool use.

```
User message
    │
    ▼
Intent Classification (lightweight LLM call)
    ├── "Factual question about risk data" → SQL lookup tool
    ├── "Policy/regulation question" → Document RAG tool  
    ├── "Analysis request (trends, gaps)" → Structured analysis tool
    └── "Action request (start assessment, send reminder)" → Workflow tool

Tool execution
    ├── SQLQueryTool: converts natural language → parameterised SQL → executes → returns structured data
    ├── DocumentRAGTool: embeds query → vector search → retrieve top-5 chunks → re-rank → return passages
    ├── StructuredAnalysisTool: runs pre-defined Python analysis functions (e.g. gap_analysis, trend_summary)
    └── WorkflowTool: validates permissions → calls workflow API endpoint

LLM synthesis
    └── Combine tool results + system prompt → generate response
    
Response + citations
    └── Return markdown-formatted answer with source references
```

**System prompt (abridged):**
```
You are an ICS Risk Advisor assistant embedded in an Internal Control System dashboard 
for a financial services organisation. Your users are risk managers (2LoD) and business 
process owners (1LoD). 

You have access to:
- The organisation's risk and control register
- RCSA assessment history and status
- Policy documents, procedures, and regulatory guidance
- Workflow tools to initiate actions (subject to user permissions)

When answering questions about risk data, always cite the risk ID or control ID.
When referencing regulations, cite the specific article.
When asked to take an action, confirm with the user before executing.
Never fabricate risk ratings, dates, or regulatory citations.
If you are unsure, say so clearly and suggest the user consult the relevant register directly.

Current context:
- User: {user_name} | Role: {user_role} | Current view: {current_view}
- Today's date: {today}
```

**Context injection:** Up to 15,000 tokens of structured context is injected per request:
- If the user is viewing an area: top 10 risks for that area by residual score
- If viewing a process: all risks and controls for that process
- Always: upcoming RCSA due dates, open high/critical risks, current TPRM signals

**Token budget management:** Input context is trimmed progressively (remove oldest chat history first, then reduce structured context) if the total would exceed the model's context window.

**Streaming:** Responses stream via Server-Sent Events (SSE) to the frontend for a real-time typing effect.

---

### 4.2 TPRM Risk Suggestion Engine

**Purpose:** When a vendor's classification changes to Critical, or a new Critical/Important vendor is onboarded, automatically generate a draft risk entry for the risk manager to review.

**Trigger:** Azure Service Bus message from the TPRM event feed (Section 2.4)

**Processing flow:**

```python
class TPRMSignalEngine:
    
    def process_event(self, event: VendorChangeEvent) -> TPRMSignal:
        
        # 1. Rule-based pre-filter
        if not self.is_signal_generating_event(event):
            return None  # e.g. minor field change, not classification-relevant
        
        # 2. Fetch vendor context
        vendor = self.vendor_repo.get(event.vendor_id)
        affected_processes = self.process_repo.get_by_vendor(event.vendor_id)
        existing_vendor_risks = self.risk_repo.get_by_vendor(event.vendor_id)
        sla_breaches = self.sla_repo.get_history(event.vendor_id, months=12)
        
        # 3. Deduplication check
        if self.signal_already_exists(event.vendor_id, event.changed_field):
            return None
        
        # 4. LLM classification and risk drafting
        prompt = self.build_tprm_prompt(vendor, affected_processes, event, sla_breaches)
        llm_response = self.llm.generate(prompt)
        
        # 5. Parse structured output
        suggested = self.parse_risk_suggestion(llm_response)
        
        # 6. Persist signal for review
        signal = TPRMSignal(
            vendor_id=vendor.vendor_id,
            trigger_event=event.event_type,
            suggested_risk_title=suggested.title,
            suggested_risk_description=suggested.description,
            suggested_category=suggested.category,
            suggested_inherent_likelihood=suggested.inherent_likelihood,
            suggested_inherent_impact=suggested.inherent_impact,
            suggested_controls=suggested.controls,
            affected_processes=[p.process_id for p in affected_processes],
            status='Pending'
        )
        self.signal_repo.save(signal)
        
        # 7. Push notification to relevant 2LoD users
        self.notifier.notify_tprm_signal(signal)
        
        return signal
```

**LLM prompt design for TPRM signal:**
The prompt includes: vendor name, classification change, services provided, affected processes, SLA breach history, and any existing vendor-linked risks. The model is instructed to output structured JSON with: `risk_title`, `risk_description`, `risk_category`, `inherent_likelihood` (1–5), `inherent_impact` (1–5), `risk_rationale`, `suggested_controls` (array of control titles and types), `regulatory_references`.

**Output parsing:** JSON mode enforced (Azure OpenAI `response_format: {"type": "json_object"}`). Schema validated against Pydantic model before persisting.

---

### 4.3 Trend Detection

**Purpose:** Identify processes or areas where residual risk scores are trending upward over time, and surface this as an insight on the dashboard.

**Algorithm:**

```python
def detect_trends(risk_id: str, lookback_quarters: int = 4) -> TrendResult:
    # Fetch quarterly residual scores
    history = risk_rating_history_repo.get(risk_id, lookback_quarters)
    
    if len(history) < 3:
        return TrendResult(risk_id=risk_id, trend='Insufficient data')
    
    scores = [h.residual_likelihood * h.residual_impact for h in history]
    
    # Linear regression on residual scores over time
    from scipy.stats import linregress
    x = range(len(scores))
    slope, intercept, r_value, p_value, std_err = linregress(x, scores)
    
    # Classify trend
    if p_value > 0.1:
        trend = 'Stable'  # Not statistically significant
    elif slope > 0.5:
        trend = 'Increasing'
    elif slope < -0.5:
        trend = 'Decreasing'
    else:
        trend = 'Stable'
    
    return TrendResult(
        risk_id=risk_id,
        trend=trend,
        slope=slope,
        r_squared=r_value**2,
        current_score=scores[-1],
        prior_score=scores[-2],
        direction_delta=scores[-1] - scores[-2]
    )
```

**Scheduled execution:** Azure Function on a weekly schedule (Sunday 03:00 UTC). Results written to `risk_trends` table. Dashboard reads from this table — trend arrows on risk cards are populated from this pre-computed data.

---

### 4.4 RCSA Due-Date Monitor

**Purpose:** Identifies RCSA assessments that are approaching their due date or are already overdue. Triggers workflow tasks and notifications.

**Schedule:** Azure Function running every day at 07:00 UTC.

**Logic:**
```python
def monitor_rcsa_due_dates():
    today = date.today()
    
    # Find overdue assessments
    overdue = assessment_repo.get_by_status_and_date(
        statuses=['Not Started', 'In Progress', '1LoD Complete'],
        due_before=today
    )
    for assessment in overdue:
        assessment.status = 'Overdue'
        workflow_engine.update_task_status(assessment.task_id, 'Overdue')
        notifier.send_overdue_alert(assessment)
    
    # Find assessments due in ≤ 14 days (send reminder if no reminder in last 7 days)
    upcoming = assessment_repo.get_by_status_and_due_range(
        statuses=['Not Started', 'In Progress'],
        due_before=today + timedelta(days=14)
    )
    for assessment in upcoming:
        task = workflow_task_repo.get_by_assessment(assessment.assessment_id)
        if task.last_reminder_sent is None or (today - task.last_reminder_sent.date()).days >= 7:
            notifier.send_reminder(assessment)
            workflow_task_repo.record_reminder(task.task_id)
```

---

### 4.5 1LoD Workflow Engine

**Purpose:** Manages the lifecycle of tasks assigned to 1LoD process owners: pending RCSA inputs, control test due dates, finding remediations, and TPRM risk confirmations.

**Technology:** Azure Durable Functions (stateful orchestration for multi-step workflows with escalation timers)

**Workflow: RCSA Assessment Lifecycle**

```
State: Not Started
  → System creates WorkflowTask (type: RCSA_Input, assigned_to: process_owner)
  → Email + Teams notification sent to process_owner

State: Reminder Sent (Day -14)
  → DurableFunction timer fires
  → Notifier sends reminder email/Teams message
  → task.reminder_count++

State: Escalation (Day -3 if still Not Started)
  → DurableFunction timer fires
  → Escalation notification sent to process_owner + manager
  → task.status = Escalated

State: In Progress
  → Triggered when process_owner starts RCSA in dashboard (POST /api/assessments/start)

State: 1LoD Complete
  → Triggered when process_owner submits RCSA
  → 2LoD reviewer notified for review

State: Completed
  → Triggered when 2LoD reviewer approves
  → RCSA cycle record closed
```

**Notification channels:**
- Email: via Azure Communication Services Email
- Microsoft Teams: via Power Automate connector or Teams Webhook
- In-app: notification badge in the dashboard (SSE push to active users)

---

### 4.6 Natural Language Query Interface

**Purpose:** Allows users to type plain-language queries like "show me all high-risk processes in Operations" and receive structured results.

**Two-mode approach:**

**Mode 1: Text-to-SQL** (for well-scoped data queries)
```
User: "How many overdue RCSAs are in IT & Security?"
  │
  ▼
LLM: generate SQL
  │   SELECT COUNT(*) FROM rcsa_assessments a
  │   JOIN processes p ON a.process_id = p.process_id
  │   JOIN areas ar ON p.area_id = ar.area_id
  │   WHERE a.assessment_status = 'Overdue'
  │   AND ar.area_name = 'IT & Security'
  ▼
Execute against read-only DB connection (limited to SELECT only)
  │
  ▼
Format result: "There are 3 overdue RCSA assessments in IT & Security."
```

**Mode 2: Semantic Search** (for policy/document questions)
```
User: "What does our policy say about vendor concentration risk?"
  │
  ▼
Embed query using text-embedding-3-large
  │
  ▼
Hybrid search in Azure AI Search: top-5 chunks
  │
  ▼
LLM: synthesise answer from retrieved passages + cite source documents
```

**Safety guardrails for Text-to-SQL:**
- SQL is generated by the LLM but executed against a read-only database user (`ics_agent_readonly`)
- SQL is validated before execution: no DML (INSERT/UPDATE/DELETE/DROP), no schema-changing statements, no system table access, single statement only
- Maximum result set: 1000 rows
- Execution timeout: 10 seconds

---

## Section 5 — API Layer

### 5.1 API Framework and Hosting

**Framework:** FastAPI (Python 3.11+) — chosen for: async support, automatic OpenAPI docs, Pydantic validation  
**Hosting:** Azure Container Apps (serverless, auto-scales to zero when idle in non-prod)  
**Authentication:** Azure AD OAuth2 (OIDC) — Bearer token on all endpoints  
**API Gateway:** Azure API Management (APIM) — rate limiting, caching, API versioning, developer portal

### 5.2 Endpoint Catalogue

#### Overview & KPI Endpoints

```
GET /api/overview
Description: Returns all KPI metrics for the landing page dashboard
Auth: Any authenticated role
Response:
{
  "open_risks_total": 47,
  "high_critical_risks": 12,
  "control_gaps": 8,
  "overdue_rcsas": 3,
  "lod1_pending_input": 5,
  "tprm_pending_signals": 2,
  "areas": [
    {
      "area_id": "P-01",
      "area_name": "Operations & Sourcing",
      "open_risks": 9,
      "high_critical_risks": 3,
      "control_effectiveness": "Partially Effective",
      "rcsa_status": "In Progress",
      "next_rcsa_due": "2025-09-30"
    }, ...
  ],
  "last_updated": "2025-07-14T10:30:00Z"
}
```

#### Area Endpoints

```
GET /api/areas
Description: All areas with summary statistics
Auth: Any authenticated role
Query params: none
Response: Array of area summary objects (same structure as areas[] in /overview)

GET /api/areas/{area_id}
Description: Full area detail including process list, risk table, 1LoD task list
Auth: Any authenticated role
Path params: area_id (e.g. P-01)
Response:
{
  "area_id": "P-01",
  "area_name": "Operations & Sourcing",
  "summary_stats": { ... },
  "processes": [
    {
      "process_id": "PROC-001",
      "process_name": "Procurement",
      "process_owner": "Jane Smith",
      "open_risks": 4,
      "control_effectiveness": "Effective",
      "rcsa_status": "Completed",
      "next_rcsa_due": "2025-12-31"
    }, ...
  ],
  "top_risks": [ ... ],   // Top 5 risks by residual score for this area
  "lod1_tasks": [
    {
      "task_id": "TASK-0892",
      "task_type": "RCSA_Input",
      "process_name": "Accounts Payable",
      "assigned_to_name": "John Doe",
      "due_date": "2025-07-31",
      "status": "Pending",
      "days_until_due": 17,
      "reminder_count": 1
    }, ...
  ]
}
```

#### Process Endpoints

```
GET /api/processes/{process_id}
Description: Full process detail with all risks and controls
Auth: Any authenticated role; 1LoD users only see their own process(es) if role-restricted
Path params: process_id (e.g. PROC-001)
Response:
{
  "process_id": "PROC-001",
  "process_name": "Procurement",
  "area_id": "P-01",
  "process_owner": { "user_id": "...", "name": "Jane Smith", "email": "..." },
  "criticality": "Critical",
  "next_rcsa_due": "2025-12-31",
  "rcsa_status": "Completed",
  "regulatory_relevance": ["DORA", "GDPR"],
  "risks": [ { risk objects... } ],
  "controls": [ { control objects... } ],
  "current_assessment": { assessment object or null },
  "assessment_history": [ last 4 completed assessments ]
}
```

#### Risk and Control Endpoints

```
GET /api/risks
Description: Risk register with filters
Auth: Any authenticated role
Query params:
  area_id (string)
  process_id (string)
  status (string, multi-value: ?status=Open&status=Escalated)
  rating (string: Critical|High|Medium|Low)
  rating_type (string: inherent|residual, default: residual)
  owner_id (string)
  page (int, default 1)
  page_size (int, default 50, max 200)
  sort_by (string: residual_score|inherent_score|next_review_date)
  sort_dir (string: asc|desc)
Response: { total, page, page_size, items: [risk objects] }

GET /api/risks/{risk_id}
Description: Single risk detail with full history
Response: risk object + rating_history array + linked_controls array + linked_findings array

PUT /api/risks/{risk_id}
Description: Update risk (status, ratings, notes)
Auth: 2LoD role required; 1LoD can update own risks in limited fields
Request body: Partial risk object (only updatable fields)
Response: Updated risk object + audit entry created

GET /api/controls
Description: Control register with filters
Query params: area_id, process_id, effectiveness_rating, status, key_control_indicator (bool)
Response: { total, page, page_size, items: [control objects] }
```

#### Heatmap Endpoint

```
GET /api/heatmap
Description: Returns data for rendering a 5x5 risk heatmap
Auth: Any authenticated role
Query params:
  mode (string: inherent|residual, default: residual)
  scope (string: area|process|all, default: all)
  scope_id (string: area_id or process_id, required if scope != 'all')
Response:
{
  "mode": "residual",
  "scope": "area",
  "scope_id": "P-01",
  "matrix": {
    "cells": [
      {
        "likelihood": 4,
        "impact": 5,
        "risk_count": 2,
        "risks": [
          { "risk_id": "RSK-001", "risk_title": "...", "process_id": "PROC-001" }
        ]
      }, ...
    ]
  },
  "thresholds": {
    "critical": { "min_score": 15 },
    "high": { "min_score": 9 },
    "medium": { "min_score": 4 },
    "low": { "min_score": 1 }
  }
}
```

#### RCSA Timeline Endpoint

```
GET /api/rcsa/timeline
Description: Upcoming RCSA due dates (used for timeline view)
Auth: Any authenticated role
Query params:
  from_date (date, default: today)
  to_date (date, default: today + 12 months)
  area_id (string, optional)
  status (string, multi-value)
Response: { assessments: [ assessment objects with due_date, status, assigned_to, area, process ] }
```

#### Agent Endpoints

```
POST /api/agent/chat
Description: Send a message to the LLM agent
Auth: Any authenticated role
Request body:
{
  "message": "Show me all high-risk processes in Operations & Sourcing",
  "session_id": "sess-abc123",   // For conversation continuity
  "context": {
    "current_view": "area",
    "current_area_id": "P-01",
    "current_process_id": null
  }
}
Response (streaming SSE):
  data: {"type": "token", "content": "Here"}
  data: {"type": "token", "content": " are"}
  ...
  data: {"type": "done", "message_id": "msg-xyz", "tokens_used": 842, "sources": [...]}
  
Notes:
  - Streaming response via SSE
  - If streaming not supported by client, use POST /api/agent/chat/sync for full response
  - All agent interactions logged to agent_audit_log
```

#### Workflow and Notification Endpoints

```
POST /api/assessments/start
Description: Initiate an RCSA assessment (creates or activates assessment record)
Auth: 1LoD (own process) or 2LoD
Request body: { "process_id": "PROC-001", "cycle_id": "Q3-2025" }
Response: { "assessment_id": "...", "status": "In Progress", "due_date": "..." }

POST /api/notifications/remind
Description: Send a manual reminder for a specific workflow task
Auth: 2LoD role required
Request body: { "task_id": "TASK-0892" }
Response: { "sent": true, "sent_to": "john.doe@client.com", "sent_at": "..." }

POST /api/notifications/remind-all
Description: Send reminders to all pending 1LoD tasks in an area
Auth: 2LoD role required
Query params: area_id (required)
Response: { "sent_count": 4, "skipped_count": 1, "results": [...] }

POST /api/notifications/escalate
Description: Escalate a specific workflow task to the assignee's manager
Auth: 2LoD role required
Request body: { "task_id": "TASK-0892" }
Response: { "escalated": true, "escalated_to": "manager@client.com" }
```

#### TPRM Signal Endpoints

```
GET /api/tprm/signals
Description: List pending TPRM-triggered risk suggestions
Auth: 2LoD role required
Query params: status (default: Pending), area_id
Response: { total, items: [signal objects] }

PUT /api/tprm/signals/{signal_id}/accept
Description: Accept TPRM signal → create draft risk in Risk Register
Auth: 2LoD role required
Request body: { "modifications": { optional field overrides } }
Response: { "risk_id": "RSK-new-001", "signal_status": "Accepted" }

PUT /api/tprm/signals/{signal_id}/reject
Description: Reject TPRM signal
Auth: 2LoD role required
Request body: { "reason": "..." }
Response: { "signal_status": "Rejected" }
```

---

## Section 6 — Security & Compliance

### 6.1 Data Residency

All Azure resources are deployed in `West Europe` (Netherlands) or `North Europe` (Ireland) region. No data is replicated outside the EU. Azure OpenAI Service resource is deployed in the same EU region. Cross-region replication (e.g. for geo-redundant backup) uses EU pairs only.

### 6.2 Network Security

```
Architecture: No public endpoints for any data services

Azure Virtual Network (ICS-Dashboard-VNet)
├── Subnet: ics-app-subnet         ← Container Apps, Function Apps
├── Subnet: ics-data-subnet        ← PostgreSQL, Redis, Service Bus (via Private Endpoints)
├── Subnet: ics-ai-subnet          ← Azure OpenAI, AI Search (via Private Endpoints)
└── Subnet: ics-integration-subnet ← ADF integration runtime

Private Endpoints:
  - Azure PostgreSQL → DNS: ics-postgres.privatelink.postgres.database.azure.com
  - Azure Redis Cache → DNS: ics-redis.privatelink.redis.cache.windows.net
  - Azure Blob Storage → DNS: icsstorage.privatelink.blob.core.windows.net
  - Azure AI Search → DNS: ics-search.privatelink.search.windows.net
  - Azure OpenAI → DNS: ics-openai.privatelink.openai.azure.com

Network Security Groups (NSGs):
  - App subnet: Allow HTTPS inbound from APIM only; deny all other inbound
  - Data subnet: Allow connections from App subnet only; deny all other inbound
  - All subnets: Block outbound internet except to Azure platform services via Service Endpoints
```

### 6.3 Identity & Access Management

**Authentication:** Azure AD (OIDC/OAuth2). JWT tokens validated on every API request.

**RBAC roles and permissions:**

| Permission | 1LoD | 2LoD | ReadOnly | Admin |
|------------|------|------|----------|-------|
| View all risks and controls | Own area only | All areas | All areas | All areas |
| View RCSA details | Own processes | All | All | All |
| Start RCSA assessment | Own processes | Any process | No | Any process |
| Update risk (limited fields) | Own process risks | Any risk | No | Any risk |
| Send reminders | No | Yes | No | Yes |
| Escalate tasks | No | Yes | No | Yes |
| Accept/reject TPRM signals | No | Yes | No | Yes |
| Access agent chat | Yes | Yes | Yes | Yes |
| View agent audit log | No | Own sessions | No | All |
| Admin functions | No | No | No | Yes |

**Row-level security:** 1LoD users are filtered at the API layer to their assigned processes/areas. This is enforced in the FastAPI dependency injection layer, not solely at the database level.

### 6.4 Audit Logging

Every state-changing action is recorded in `agent_audit_log` and Azure Monitor:
- All POST/PUT API calls: endpoint, user, request summary, response status, timestamp
- All LLM agent interactions: session ID, user, input/output summary (no full PII in log), token count
- All notification dispatches: task ID, recipient, channel, timestamp
- All TPRM signal accept/reject decisions
- All risk updates

Logs are retained for 90 days in Azure Monitor. For compliance archiving, logs are exported to Azure Blob Storage (cold tier) and retained per the client's data retention policy (minimum 7 years for financial entities).

### 6.5 LLM Prompt Security

- PII minimisation: User directory data (email, names) is referenced by user_id in prompts; display names are injected only where necessary for contextual responses
- Prompt injection prevention: User input is passed as a quoted literal parameter; never concatenated directly into system prompts
- Context bounding: LLM has access only to data the requesting user is authorised to see (enforced by context-building logic in the agent layer)
- No training: Azure OpenAI is configured with `opt-out-of-training: true` (default for Azure deployments)
- Output filtering: Azure OpenAI content filtering is enabled for all completions

---

## Section 7 — Technology Stack Summary

| Layer | Technology | Version / Tier | Notes |
|-------|-----------|----------------|-------|
| **Frontend** | React | 18.x | TypeScript; built with Vite |
| **Frontend hosting** | Azure Static Web Apps | Standard tier | Integrated with Azure AD auth |
| **API framework** | FastAPI | 0.110+ | Python 3.11 |
| **API hosting** | Azure Container Apps | Consumption plan | Auto-scales; min 1 instance in prod |
| **API gateway** | Azure API Management | Developer → Standard tier | Rate limiting, caching, dev portal |
| **Orchestration / ETL** | Azure Data Factory | v2 | Managed service; no infrastructure to maintain |
| **Event bus** | Azure Service Bus | Standard tier | TPRM change events |
| **Serverless functions** | Azure Functions | v4 (.NET isolated / Python) | RCSA monitor, document ingestion, TPRM engine |
| **Workflow orchestration** | Azure Durable Functions | v4 | Stateful 1LoD workflow orchestration |
| **Relational DB** | Azure Database for PostgreSQL — Flexible Server | v16, General Purpose 8 vCores | Primary operational store |
| **Cache** | Azure Cache for Redis | C2 Standard | API response caching; session storage |
| **Vector search** | Azure AI Search | Standard S1 | Hybrid BM25 + vector (pgvector as fallback) |
| **Blob storage** | Azure Blob Storage | LRS + ZRS | Document store; GRC export staging |
| **LLM** | Azure OpenAI Service (GPT-4o) | 2024-11-20 deployment | Within client tenant; EU region |
| **Embeddings** | Azure OpenAI (text-embedding-3-large) | Standard deployment | Document chunk embeddings |
| **RAG framework** | LangChain | 0.2+ | Agent and RAG pipeline |
| **Notifications** | Azure Communication Services (Email) | Standard | Transactional email |
| **Teams integration** | Microsoft Teams Webhook / Power Automate | — | In-app notifications |
| **Identity** | Azure Active Directory | — | SSO, RBAC, app registration |
| **Secret management** | Azure Key Vault | Standard tier | All credentials, API keys |
| **Container registry** | Azure Container Registry | Standard | Docker images for Container Apps |
| **CI/CD** | Azure DevOps Pipelines or GitHub Actions | — | Build, test, deploy pipelines |
| **IaC** | Terraform | 1.7+ | All Azure resources defined as code |
| **Monitoring** | Azure Monitor + Application Insights | — | Metrics, logs, alerts |
| **Security** | Azure Defender for Cloud | Standard | Threat protection, compliance posture |
| **Networking** | Azure Virtual Network + Private Endpoints | — | No public data service exposure |

---

*End of document. Next: [03-roadmap.md](03-roadmap.md)*
