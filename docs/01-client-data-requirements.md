# Client Data Requirements

**Document:** 01 — Client Data Requirements  
**Project:** AI-Powered ICS Dashboard  
**Version:** 1.0  
**Audience:** Programme Manager, Data Engineers, ICS Domain Expert, Client GRC System Admin

---

## Purpose

This document defines every data source, data entity, field, format, and quality criterion that must be obtained from the client before development can begin in earnest. It is structured as a working instrument: use it in Phase 0 discovery workshops to drive conversations with the client's risk management team, IT team, and data protection officer.

No AI component, no dashboard feature, and no workflow can be built without first securing the data described here. Missing or poor-quality data is the single most common cause of timeline slippage on GRC modernisation engagements.

---

## Section 1 — GRC System Data

### 1.1 GRC Platform Identification

Financial services clients typically operate one of the following GRC platforms. Identify which platform(s) are in scope before requesting data exports, as field names and export mechanisms differ significantly.

| Platform | Common in FS? | API Capability | Notes |
|----------|--------------|----------------|-------|
| ServiceNow GRC (IRM) | High | REST API (Table API) | Most flexible; supports real-time webhooks |
| RSA Archer | High | REST API + Reports | Older architecture; batch export often preferred |
| MetricStream | Medium | REST API | Good API coverage |
| IBM OpenPages | Medium | REST API | Strong in banking; complex data model |
| OneTrust | Growing | REST API | Strong on GDPR/privacy module |
| Custom / Excel-based | Common in mid-market | Manual export | Requires normalisation effort |
| Fusion Risk Management | Niche | REST API | Strong on BCP/BCM |

**Action (Phase 0):** Client GRC System Admin to confirm platform(s), version numbers, and available API/export options within Week 1.

---

### 1.2 Risk Register

The Risk Register is the single most critical data entity. Every area view, heatmap, KPI card, and AI analysis derives from it.

**Entity name:** Risk Register  
**GRC Module:** Risk Management / Enterprise Risk Management (ERM)  
**Owner at client:** Chief Risk Officer / Head of Operational Risk  
**Format:** REST API (preferred) / CSV export / database view

**Required Fields:**

| Field Name | Data Type | Description | Mandatory |
|------------|-----------|-------------|-----------|
| risk_id | String (UUID or client code) | Unique risk identifier (e.g. RSK-2024-0142) | Yes |
| risk_title | String (max 200 chars) | Short descriptive title | Yes |
| risk_description | Text | Full narrative description | Yes |
| business_area | String (controlled vocab) | One of: Operations & Sourcing, Payments, IT & Security, Finance, Data & Privacy, Treasury | Yes |
| process_id | String | FK to Process Inventory | Yes |
| risk_category | String (controlled vocab) | Operational, Financial, Compliance, Reputational, Strategic, Technology | Yes |
| risk_sub_category | String | Optional granular sub-category | No |
| risk_owner | String (user ID or email) | Primary accountable party (1LoD) | Yes |
| risk_reviewer | String (user ID or email) | 2LoD reviewer | No |
| status | String (controlled vocab) | Open, Closed, In Remediation, Accepted, Escalated | Yes |
| inherent_likelihood | Integer (1–5) | Likelihood before controls | Yes |
| inherent_impact | Integer (1–5) | Impact before controls | Yes |
| inherent_rating | String | Derived: Critical/High/Medium/Low (or score 1–25) | Yes |
| residual_likelihood | Integer (1–5) | Likelihood after controls | Yes |
| residual_impact | Integer (1–5) | Impact after controls | Yes |
| residual_rating | String | Derived rating after controls | Yes |
| target_residual_likelihood | Integer (1–5) | Desired future state likelihood | No |
| target_residual_impact | Integer (1–5) | Desired future state impact | No |
| risk_appetite_breach | Boolean | Whether residual exceeds appetite threshold | Yes |
| regulatory_reference | String | DORA Art. X, GDPR Art. Y, CRR Art. Z etc. | No |
| linked_control_ids | Array[String] | FK list to Control Register | Yes |
| last_reviewed_date | Date | Date of last formal review | Yes |
| next_review_date | Date | Scheduled next review | Yes |
| created_date | Date | When risk was first logged | Yes |
| modified_date | Timestamp | Last modification timestamp | Yes |
| notes | Text | Free-text notes from 2LoD | No |
| tags | Array[String] | Searchable labels | No |

**Minimum Data Quality Criteria:**
- All mandatory fields populated for 100% of Open risks
- `inherent_*` and `residual_*` fields populated for at least 95% of Open risks
- `process_id` must resolve to a valid entry in the Process Inventory for 100% of records
- No duplicate `risk_id` values
- `status` values must match the controlled vocabulary exactly
- Historical records going back at least 2 complete RCSA cycles (typically 2 years)

---

### 1.3 Control Register

**Entity name:** Control Register  
**GRC Module:** Controls Management  
**Owner at client:** Head of Internal Controls / Compliance  
**Format:** REST API / CSV export

**Required Fields:**

| Field Name | Data Type | Description | Mandatory |
|------------|-----------|-------------|-----------|
| control_id | String | Unique control identifier (e.g. CTL-2024-0089) | Yes |
| control_title | String | Short title | Yes |
| control_description | Text | Full description of the control activity | Yes |
| control_type | String | Preventive, Detective, Corrective | Yes |
| control_category | String | Manual, Automated, IT-Dependent Manual | Yes |
| business_area | String | Same controlled vocab as Risk Register | Yes |
| process_id | String | FK to Process Inventory | Yes |
| control_owner | String | Person responsible for operating the control | Yes |
| control_operator | String | Person performing the control (if different) | No |
| linked_risk_ids | Array[String] | FK list to Risk Register | Yes |
| effectiveness_rating | String | Effective, Partially Effective, Ineffective, Not Assessed | Yes |
| last_test_date | Date | Most recent test/assessment date | Yes |
| next_test_date | Date | Scheduled next test | Yes |
| test_frequency | String | Monthly, Quarterly, Semi-Annual, Annual | Yes |
| testing_methodology | String | Self-assessment, Independent testing, Automated monitoring | No |
| key_control_indicator | Boolean | Whether this is a Key Control | Yes |
| regulatory_reference | String | Regulatory obligations this control addresses | No |
| evidence_location | String | Where evidence is stored (SharePoint URL, GRC attachment) | No |
| status | String | Active, Inactive, Under Review | Yes |
| created_date | Date | | Yes |
| modified_date | Timestamp | | Yes |

**Minimum Data Quality Criteria:**
- `effectiveness_rating` populated for all Active controls
- `last_test_date` and `next_test_date` populated for all Active controls
- `linked_risk_ids` populated for at least 90% of Active controls
- At least one control linked per High or Critical Open risk

---

### 1.4 RCSA Assessment Results

**Entity name:** RCSA Assessment Results  
**GRC Module:** Risk & Control Self-Assessment  
**Owner at client:** 2LoD Risk Management team  
**Format:** REST API / CSV / structured report export

**Required Fields:**

| Field Name | Data Type | Description | Mandatory |
|------------|-----------|-------------|-----------|
| assessment_id | String | Unique assessment run ID | Yes |
| assessment_cycle | String | e.g. Q1-2025, H1-2025, Annual-2024 | Yes |
| business_area | String | Area in scope for this assessment | Yes |
| process_id | String | Process in scope | Yes |
| assessment_status | String | Not Started, In Progress, 1LoD Complete, 2LoD Review, Completed, Overdue | Yes |
| start_date | Date | Assessment cycle start | Yes |
| due_date | Date | Deadline for 1LoD input | Yes |
| completion_date | Date | Actual completion date (null if not yet complete) | No |
| assigned_to | String (user ID) | 1LoD owner responsible for completing | Yes |
| reviewer | String (user ID) | 2LoD reviewer | Yes |
| overall_risk_rating | String | The resulting aggregate risk rating for this process | Yes |
| control_effectiveness_summary | String | Effective / Partially Effective / Ineffective | Yes |
| 1lod_comments | Text | Comments from 1LoD during self-assessment | No |
| 2lod_comments | Text | Review notes from 2LoD | No |
| findings_count | Integer | Number of findings raised during assessment | Yes |
| open_actions_count | Integer | Outstanding action items from this assessment | Yes |
| created_date | Date | | Yes |
| modified_date | Timestamp | | Yes |

**Minimum Data Quality Criteria:**
- At least 2 full historical RCSA cycles with completion records
- `due_date` populated for all assessments including future scheduled cycles
- `assessment_status` values must match controlled vocabulary
- `assigned_to` must resolve to a valid user in the User Directory

---

### 1.5 Process Inventory

**Entity name:** Process Inventory  
**GRC Module:** Business Process Management  
**Owner at client:** Business Process Management / COO function  
**Format:** REST API / CSV / Excel

**Required Fields:**

| Field Name | Data Type | Description | Mandatory |
|------------|-----------|-------------|-----------|
| process_id | String | Unique process identifier (e.g. PROC-001) | Yes |
| process_name | String | Full process name | Yes |
| process_description | Text | Description of what the process covers | Yes |
| business_area | String | Parent area | Yes |
| sub_area | String | Sub-grouping if applicable | No |
| process_owner | String (user ID) | 1LoD process owner | Yes |
| lod_designation | String | 1LoD, 2LoD | Yes |
| process_criticality | String | Critical, Important, Standard | Yes |
| regulatory_relevance | Array[String] | Which regulations are relevant: DORA, GDPR, CRR, MiFID II etc. | Yes |
| ict_service_flag | Boolean | Whether this process relies on a critical ICT service (DORA relevance) | Yes |
| tprm_relevant | Boolean | Whether third-party vendors are in scope for this process | Yes |
| rcsa_frequency | String | Quarterly, Semi-Annual, Annual | Yes |
| next_rcsa_due | Date | Next RCSA due date | Yes |
| active | Boolean | Whether process is currently in scope | Yes |

---

### 1.6 Risk Ratings (Inherent and Residual) — Historical Snapshots

Beyond the current-state risk ratings in the Risk Register, we require snapshots of risk ratings over time. This powers the trend detection component.

**Required:** Quarterly or semi-annual snapshots of `inherent_likelihood`, `inherent_impact`, `residual_likelihood`, `residual_impact` per risk, going back at least 2 years.

**Format:** Separate historical table or audit log extract from GRC system  
**Fields:** `risk_id`, `snapshot_date`, `inherent_likelihood`, `inherent_impact`, `residual_likelihood`, `residual_impact`, `assessed_by`

---

### 1.7 Control Test Results

**Entity name:** Control Test Results  
**GRC Module:** Controls Testing / SOX Testing / Internal Audit support  
**Owner at client:** Internal Controls / Internal Audit  
**Format:** REST API / CSV

**Required Fields:**

| Field Name | Data Type | Description | Mandatory |
|------------|-----------|-------------|-----------|
| test_id | String | Unique test record ID | Yes |
| control_id | String | FK to Control Register | Yes |
| test_date | Date | Date test was performed | Yes |
| tested_by | String | User who performed the test | Yes |
| test_result | String | Pass, Fail, Pass with Exceptions | Yes |
| exceptions_noted | Text | Description of exceptions/failures | No |
| evidence_reference | String | Document reference or URL | No |
| finding_id | String | FK to Findings & Remediation if test failed | No |
| test_period_start | Date | Period under test | Yes |
| test_period_end | Date | | Yes |

---

### 1.8 Control Owners

Cross-reference table linking controls to their operational owners and their manager chain. Used for notification routing.

**Required Fields:** `control_id`, `owner_user_id`, `owner_name`, `owner_email`, `manager_user_id`, `manager_email`, `business_area`, `department`

---

### 1.9 Assessment Schedules (Forward Calendar)

The RCSA timeline view requires forward-looking scheduled dates. This is distinct from historical RCSA results.

**Required Fields:** `schedule_id`, `process_id`, `business_area`, `cycle_name`, `due_date`, `assigned_to`, `type` (RCSA / Control Test / External Audit / Regulatory Submission)

**Minimum:** At least 12 months of forward-scheduled assessments populated in the GRC system before go-live

---

### 1.10 Findings & Remediation Plans

**Entity name:** Findings & Remediation Plans  
**GRC Module:** Issue Management / Action Tracking  
**Owner at client:** Internal Audit / Risk Management  
**Format:** REST API / CSV

**Required Fields:**

| Field Name | Data Type | Description | Mandatory |
|------------|-----------|-------------|-----------|
| finding_id | String | Unique finding ID | Yes |
| finding_title | String | Short description | Yes |
| finding_description | Text | Full narrative | Yes |
| source | String | Internal Audit, RCSA, External Audit, Regulatory Inspection, Control Test | Yes |
| severity | String | Critical, High, Medium, Low | Yes |
| status | String | Open, In Progress, Closed, Overdue | Yes |
| linked_risk_ids | Array[String] | Risks this finding relates to | No |
| linked_control_ids | Array[String] | Controls this finding relates to | No |
| business_area | String | | Yes |
| process_id | String | | No |
| owner | String | Person responsible for remediation | Yes |
| target_close_date | Date | | Yes |
| actual_close_date | Date | | No |
| remediation_plan | Text | Planned actions | Yes |
| created_date | Date | | Yes |
| modified_date | Timestamp | | Yes |

---

### 1.11 Issue Tracker Integration

If the client maintains a separate issue tracker (Jira, ServiceNow Incidents, Azure DevOps) for IT-related risks or control failures, an export or read-only API connection is required.

**Minimum fields:** `issue_id`, `issue_title`, `severity`, `status`, `linked_risk_or_control_id`, `created_date`, `resolved_date`

---

## Section 2 — TPRM Data

### 2.1 Vendor Registry

The TPRM signal engine requires a live feed from the client's vendor registry. This is typically maintained in a dedicated TPRM tool (ProcessUnity, Prevalent, OneTrust, or a module within the main GRC platform) or in a procurement system (SAP Ariba, Coupa).

**Entity name:** Vendor Registry  
**Owner at client:** Procurement / Third-Party Risk team  
**Format:** REST API / CSV export (daily refresh minimum)

**Required Fields:**

| Field Name | Data Type | Description | Mandatory |
|------------|-----------|-------------|-----------|
| vendor_id | String | Unique vendor ID | Yes |
| vendor_name | String | Legal entity name | Yes |
| vendor_short_name | String | Trading name / common name | No |
| vendor_classification | String | Critical, Important, Standard, Registered | Yes |
| previous_classification | String | Prior classification (to detect changes) | Yes |
| criticality_rationale | Text | Why classified at this level | No |
| country_of_incorporation | String (ISO-3166) | | Yes |
| country_of_operations | Array[String] | Countries where services are delivered | No |
| service_category | String | ICT, Professional Services, Facilities, etc. | Yes |
| services_provided | Text | What the vendor delivers | Yes |
| sub_processor_flag | Boolean | Whether vendor processes client personal data | Yes |
| ict_service_flag | Boolean | Whether vendor is an ICT third party under DORA | Yes |
| dora_criticality | String | Critical, Important, Non-critical, Not assessed | No |
| contract_id | String | Reference to contract in contract management system | Yes |
| contract_expiry_date | Date | | Yes |
| contract_value_eur | Decimal | Annual contract value | No |
| relationship_owner | String (user ID) | Internal owner of the vendor relationship | Yes |
| business_areas_served | Array[String] | Which business areas use this vendor | Yes |
| processes_served | Array[String] | Which processes depend on this vendor | No |
| concentration_risk | Boolean | Whether removal of vendor would cause significant disruption | Yes |
| exit_plan_available | Boolean | Whether a formal exit plan exists | No |
| onboarding_date | Date | | Yes |
| last_assessment_date | Date | | Yes |
| next_assessment_due | Date | | Yes |
| status | String | Active, Offboarding, Inactive | Yes |

**Triggering Logic for TPRM Signal Engine:**  
A change event must be captured when any of the following fields change: `vendor_classification`, `ict_service_flag`, `dora_criticality`, `sub_processor_flag`, `status`. These events are the input to the automated risk suggestion engine.

---

### 2.2 Vendor Assessment Results

**Required Fields:** `assessment_id`, `vendor_id`, `assessment_date`, `assessment_type` (Annual Due Diligence, Event-Driven, Onboarding), `overall_score`, `overall_rating` (High/Medium/Low Risk), `domain_scores` (JSON object: Information Security, Financial Stability, Business Continuity, Compliance, ESG), `findings_count`, `critical_findings_count`, `assessor`, `status` (Draft/Completed/Approved)

---

### 2.3 SLA Breach History

**Purpose:** Past SLA breaches with a vendor are relevant risk context when auto-generating a draft risk entry.

**Required Fields:** `breach_id`, `vendor_id`, `breach_date`, `service_category`, `sla_metric`, `target_value`, `actual_value`, `duration_hours`, `business_impact`, `status` (Open/Resolved), `root_cause`

---

### 2.4 DORA ICT Third-Party Register

Under the Digital Operational Resilience Act (DORA), the client (if an EU financial entity) must maintain a register of all ICT third-party service providers. This register has specific required fields per RTS Article 28.

**Required Fields (DORA RTS on registers of information):** `ict_tpp_name`, `ict_tpp_lei`, `service_type`, `data_classification` (Critical / Non-critical), `substitutability`, `jurisdiction_of_law`, `service_start_date`, `contractual_arrangements_ref`, `sub-contracting_chain`

---

## Section 3 — Organisational Data

### 3.1 Organisational Hierarchy

The dashboard's area/process navigation and notification routing depends on a clean org hierarchy.

**Entity name:** Organisational Hierarchy  
**Owner at client:** HR / COO  
**Format:** REST API (preferred, from HR system: Workday, SAP SuccessFactors, Oracle HCM) / CSV export

**Required Fields:**

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| org_unit_id | String | Unique ID |
| org_unit_name | String | Full name |
| org_unit_type | String | Division / Business Area / Department / Team |
| parent_org_unit_id | String | FK to parent (null for top level) |
| head_of_unit | String (user ID) | Senior leader of this unit |
| level | Integer | Depth in hierarchy |
| active | Boolean | |
| cost_centre | String | Finance cost centre reference |

**Depth required:** At minimum from top-level Financial Entity → Business Area → Department. Sub-department level is desirable.

---

### 3.2 1LoD / 2LoD Designation per Process

**Entity name:** Line-of-Defence Designation  
**Owner at client:** Chief Risk Officer / Head of Operational Risk  

For every process in the Process Inventory, the system needs to know:
- Who is the 1LoD owner (operates the process, performs self-assessment)
- Who is the 2LoD reviewer (Risk Management function, reviews and challenges)
- Whether any processes have 3LoD involvement (Internal Audit) relevant to the dashboard

**Required Fields:** `process_id`, `lod1_owner_user_id`, `lod1_owner_name`, `lod1_owner_email`, `lod1_delegate_user_id`, `lod2_reviewer_user_id`, `lod2_reviewer_name`, `lod2_reviewer_email`, `lod2_team`

---

### 3.3 User Directory

Used for: display names in the dashboard, notification routing, RBAC.

**Source:** Azure Active Directory (preferred — direct AD integration) or export from HR system  
**Owner at client:** IT / Identity & Access Management team

**Required Fields:** `user_id` (UPN/email), `display_name`, `first_name`, `last_name`, `email`, `department`, `job_title`, `business_area`, `lod_role` (1LoD / 2LoD / 3LoD / Read-only / Admin), `manager_user_id`, `active` (Boolean), `last_login_date`

**Note:** GDPR/data minimisation applies. Only fields necessary for the dashboard's function should be ingested. Do not pull payroll, performance, or personal data beyond what is listed.

---

### 3.4 Reporting Calendar

The RCSA timeline view and the RCSA Due-Date Monitor component require knowledge of all scheduled events.

**Entity name:** Reporting Calendar  
**Owner at client:** Risk Management / Finance / Regulatory Affairs  
**Format:** CSV or iCal export / manual import

**Required Fields:** `event_id`, `event_type` (RCSA Due Date, Control Test, Regulatory Submission, Board Report, Internal Audit, External Audit), `event_name`, `business_area`, `process_id` (optional), `due_date`, `responsible_party`, `frequency` (One-off, Monthly, Quarterly, Annual), `regulatory_basis` (e.g. DORA Art. 11, GDPR Art. 30)

**Minimum coverage required:** 18 months forward from go-live date

---

## Section 4 — Document Corpus (for AI Agent RAG)

The AI agent chat panel uses Retrieval-Augmented Generation (RAG) over a corpus of policy and regulatory documents. The quality of the agent's answers depends directly on the quality and completeness of this corpus.

### 4.1 Internal Policy Documents

| Document Type | Examples | Format | Priority |
|---------------|----------|--------|----------|
| Risk Management Policy | Enterprise Risk Management Policy v2.3 | PDF / Word | Critical |
| Control Framework | Internal Control Framework, SOX/J-SOX Controls Manual | PDF / Word | Critical |
| Operational Risk Policy | Op Risk Policy, RCSA Methodology Guide | PDF / Word | Critical |
| Information Security Policy | ISMS Policy, Acceptable Use Policy | PDF / Word | High |
| Data Protection Policy | GDPR Policy, Data Classification Standard | PDF / Word | High |
| Third-Party Risk Policy | TPRM Framework, Vendor Management Policy | PDF / Word | High |
| Business Continuity Policy | BCP Policy, Disaster Recovery Plan | PDF / Word | Medium |
| Compliance Policy | Compliance Framework, Regulatory Horizon Scan | PDF / Word | Medium |
| Escalation Procedures | Risk Escalation Procedure, Incident Response Procedure | PDF / Word | High |
| RCSA Procedure | Step-by-step RCSA guide for 1LoD | PDF / Word | Critical |

**Note on document versioning:** Provide current versions only. If multiple superseded versions exist, confirm with 2LoD which version is authoritative. The RAG system should not ingest superseded policies without a clear superseded flag.

---

### 4.2 Regulatory Frameworks Referenced

The agent must be able to answer questions that reference regulatory requirements. We will include the most relevant regulatory texts and Accenture-curated summaries.

| Regulation | Relevance | Source |
|------------|-----------|--------|
| DORA (EU 2022/2554) | Digital operational resilience; ICT risk management, third-party requirements | EUR-Lex |
| DORA RTS packages (5 RTSs) | Technical standards on ICT risk, incident reporting, testing, TPPM | EBA/ESMA/EIOPA |
| GDPR (EU 2016/679) | Data protection and privacy | EUR-Lex |
| CRR (EU 575/2013) / CRR II | Capital requirements, operational risk chapter | EUR-Lex |
| CRD V (EU 2019/878) | Governance, internal controls | EUR-Lex |
| EBA Guidelines on IRM (EBA/GL/2019/04) | Internal governance expectations | EBA website |
| EBA Guidelines on IRRBB (EBA/GL/2018/02) | Interest rate risk | EBA website |
| MiFID II / MiFIR | Where applicable for payments and treasury | EUR-Lex |
| BCBS Principles for Operational Resilience | International framework | BIS website |
| ECB Guide to internal models | Where applicable | ECB website |

**Action (Phase 0):** Legal / Compliance team at client to confirm which regulations are in scope and whether the client already holds sanitised internal summaries.

---

### 4.3 Previous RCSA Reports

Historical RCSA reports provide the AI agent with context about past assessments, trends, and commentary from previous cycles.

**Required:** Last 4 completed RCSA cycle reports per business area (up to 24 documents)  
**Format:** PDF (preferred) — exported from GRC system or provided as management reports  
**Sensitivity:** These documents likely contain confidential risk information. Classify as Confidential and handle under the data handling protocol in Section 5.

---

### 4.4 Internal Audit Findings and Management Responses

The AI agent can draw on IA findings to enrich its analysis and provide historical context to 2LoD users.

**Required:** Last 2 years of IA report summaries (headline findings and management responses — not full working papers)  
**Format:** PDF / Word

---

### 4.5 Process Documentation

Where available, process maps and procedure documents (SOPs) provide the agent with factual grounding when answering process-level questions.

**Required (nice-to-have):** SOPs for the 20 most critical processes across all six areas  
**Format:** PDF / Word / Visio (Visio exports as PDF)

---

## Section 5 — Access & Integration Requirements

### 5.1 GRC Platform API Access

For each GRC platform module in scope:

| Requirement | Detail |
|-------------|--------|
| API authentication method | OAuth 2.0 (preferred) / API key / Basic Auth |
| API rate limits | Confirm with GRC system admin — affects polling frequency design |
| Read vs Write access | Read access required for all entities; Write access required for: risk status updates, RCSA status updates, TPRM signal acknowledgement |
| Sandbox/UAT environment | Required for development and testing — do not develop against production |
| API documentation | Client GRC system admin to provide Swagger/OpenAPI spec or vendor API docs |
| Network access | Whether API is accessible from Accenture development infrastructure or requires VPN/private network access |

---

### 5.2 SSO / SAML Integration

The dashboard must integrate with the client's identity provider for authentication.

**Requirements:**
- Identity Provider: Azure Active Directory (SAML 2.0 / OIDC)
- Confirm whether the client uses Azure AD, Okta, PingFederate, or another IdP
- App registration in client Azure AD tenant required
- Group-based access control: At minimum, three security groups must be created: `ICS-Dashboard-1LoD`, `ICS-Dashboard-2LoD`, `ICS-Dashboard-ReadOnly`
- MFA enforcement: Required for 2LoD users; recommended for 1LoD
- Session timeout: Align with client information security policy (typically 15–30 minutes)

---

### 5.3 Data Classification and GDPR Considerations

| Category | Classification | Handling |
|----------|---------------|----------|
| Risk Register (contains business process context) | Confidential | Access restricted to 1LoD/2LoD roles; encrypted at rest and in transit |
| Control Register | Confidential | As above |
| RCSA results and comments | Confidential | As above |
| Findings and remediation plans | Highly Confidential | 2LoD+ access only for sensitive findings |
| User directory fields (name, email, manager) | Personal Data under GDPR | Data minimisation; only ingest fields required for function; covered by DPIA |
| LLM prompts containing risk data | Confidential | LLM must run within client Azure tenant; no data transmitted to external LLM APIs. See AI Infrastructure doc Section 6. |
| Vendor names in TPRM data | Confidential / Commercially Sensitive | Access restricted |

**DPIA Requirement:** A Data Protection Impact Assessment (DPIA) is required before the AI agent processes any personal data. The Accenture team will assist in preparing the DPIA template; the client's DPO must review and sign off.

---

### 5.4 Data Retention Constraints

Confirm with client:
- How long risk and control data must be retained in the dashboard platform (typically 7–10 years for financial entities)
- Whether data must remain within EU data residency boundaries (required for GDPR + DORA)
- Backup and archival requirements
- Whether there are legal holds on any data that would affect deletion or modification

---

### 5.5 Network and Infrastructure Access

| Requirement | Notes |
|-------------|-------|
| Azure subscription | Client provides Azure subscription for all deployment; Accenture team provisioned as contributors |
| Azure region | Must be EU region (e.g. West Europe / North Europe) for data residency |
| Private Endpoint configuration | GRC API connector and data platform components must use Private Endpoints — no public internet exposure |
| Firewall rules | Client IT to whitelist Accenture development IPs for Phase 0/1 development access |
| CI/CD access | GitHub Actions or Azure DevOps pipeline access to client Azure environment |
| Monitoring | Azure Monitor, Application Insights — client IT team to provide workspace IDs |

---

## Data Readiness Checklist

Use this table as the master tracking worksheet during Phase 0. Update status weekly and present at the Phase 0 steering committee.

| # | Data Source | Data Entity | Owner at Client | Format | Priority | Status | Notes |
|---|-------------|-------------|-----------------|--------|----------|--------|-------|
| 1 | GRC Platform | Risk Register (current state) | Head of Op Risk | REST API | Critical | Not Started | |
| 2 | GRC Platform | Risk Register (historical snapshots, 2yr) | Head of Op Risk | CSV export | Critical | Not Started | |
| 3 | GRC Platform | Control Register | Head of Internal Controls | REST API | Critical | Not Started | |
| 4 | GRC Platform | RCSA Assessment Results (historical) | 2LoD Risk team | REST API / CSV | Critical | Not Started | |
| 5 | GRC Platform | RCSA Assessment Results (forward schedule) | 2LoD Risk team | REST API | Critical | Not Started | |
| 6 | GRC Platform | Process Inventory | COO / BPM team | REST API / CSV | Critical | Not Started | |
| 7 | GRC Platform | Control Test Results | Internal Controls / IA | REST API / CSV | High | Not Started | |
| 8 | GRC Platform | Findings & Remediation Plans | IA / Risk Mgmt | REST API / CSV | High | Not Started | |
| 9 | GRC Platform | Assessment Schedules (18-month forward) | 2LoD Risk team | REST API | High | Not Started | |
| 10 | TPRM Tool | Vendor Registry | Third-Party Risk team | REST API | Critical | Not Started | Change events required |
| 11 | TPRM Tool | Vendor Assessment Results | Third-Party Risk team | REST API / CSV | High | Not Started | |
| 12 | TPRM Tool | SLA Breach History | Procurement | CSV | Medium | Not Started | |
| 13 | TPRM Tool | DORA ICT Third-Party Register | DORA Programme team | CSV / Excel | High | Not Started | |
| 14 | HR / IAM | Organisational Hierarchy | HR / COO | REST API | High | Not Started | |
| 15 | HR / IAM | 1LoD/2LoD Designation per Process | CRO / Risk team | CSV | Critical | Not Started | |
| 16 | Azure AD | User Directory | IT / IAM | Azure AD Graph API | Critical | Not Started | GDPR/data minimisation |
| 17 | Risk Mgmt | Reporting Calendar (18-month forward) | Risk Mgmt / Finance | CSV | High | Not Started | |
| 18 | Document Store | Risk Management Policy | Compliance | PDF | Critical | Not Started | |
| 19 | Document Store | Control Framework | Internal Controls | PDF | Critical | Not Started | |
| 20 | Document Store | RCSA Procedure / Methodology | 2LoD Risk team | PDF | Critical | Not Started | |
| 21 | Document Store | Operational Risk Policy | 2LoD Risk team | PDF | Critical | Not Started | |
| 22 | Document Store | Information Security Policy | CISO | PDF | High | Not Started | |
| 23 | Document Store | Data Protection Policy | DPO | PDF | High | Not Started | |
| 24 | Document Store | Third-Party Risk Policy | Third-Party Risk team | PDF | High | Not Started | |
| 25 | Document Store | Previous RCSA Reports (last 4 cycles) | 2LoD Risk team | PDF | High | Not Started | Confidential — DPIA required |
| 26 | Document Store | Internal Audit Findings (last 2yr) | IA | PDF | Medium | Not Started | |
| 27 | IT / Infrastructure | Azure subscription & region | IT | — | Critical | Not Started | |
| 28 | IT / IAM | Azure AD App Registration | IT / IAM | — | Critical | Not Started | SSO |
| 29 | IT | GRC API credentials (sandbox) | GRC Admin | API key / OAuth | Critical | Not Started | |
| 30 | Legal / Compliance | DPIA sign-off | DPO | — | Critical | Not Started | Required before AI agent processes personal data |

**Status values:** Not Started / In Progress / Received (Pending Validation) / Validated / Blocked

---

*End of document. Next: [02-ai-infrastructure.md](02-ai-infrastructure.md)*
