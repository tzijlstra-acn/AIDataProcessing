# Dashboard Feature Specification & Button Mapping

**Document:** 05 — Dashboard Feature Specification & Button Mapping  
**Project:** AI-Powered ICS Dashboard  
**Version:** 1.0  
**Audience:** Frontend Developers, Backend Developers, AI/ML Engineers, ICS Domain Expert (for validation)

---

## Purpose

This document is the implementation contract for every interactive element in the ICS Dashboard. For each button, tab, card, toggle, and input, it defines:
- What API call it triggers (or what client-side action it performs)
- The request payload
- The expected success state in the UI
- The loading state behaviour
- The error state behaviour

Developers should treat this as the authoritative source. Any deviation from this specification must be agreed with the Programme Manager and ICS Domain Expert and documented as a change.

---

## Master Interaction Table

| # | Element | Location | Trigger | API Call / Action | Notes |
|---|---------|----------|---------|-------------------|-------|
| N-01 | Tab: Overview | Top navigation | Click | None (client-side navigation to landing page view) | Displays KPI cards; triggers GET /api/overview on mount if data is stale |
| N-02 | Tab: Operations & Sourcing | Top navigation | Click | GET /api/areas/P-01 | |
| N-03 | Tab: Payments | Top navigation | Click | GET /api/areas/P-02 | |
| N-04 | Tab: IT & Security | Top navigation | Click | GET /api/areas/P-05 | |
| N-05 | Tab: Finance | Top navigation | Click | GET /api/areas/P-04 | |
| N-06 | Tab: Data & Privacy | Top navigation | Click | GET /api/areas/P-03 | |
| N-07 | Tab: Treasury | Top navigation | Click | GET /api/areas/P-06 | |
| O-01 | TPRM alert "Review risk →" link | Overview landing page, TPRM signal banner | Click | Client-side: navigate to TPRM Signals panel (or to P-01 area view with signal panel open) | If clicked, sets URL param ?tprm=open; signal panel auto-opens |
| O-02 | Area card | Overview landing page, area summary grid | Click | GET /api/areas/{area_id} | Navigates to area view |
| O-03 | Heatmap chip (risk dot) | Overview heatmap section | Click | GET /api/processes/{process_id} | Each dot/chip represents a risk positioned on the heatmap; click navigates to the process detail for that risk |
| O-04 | Heatmap Inherent toggle | Overview heatmap section | Click | No API call | Client-side: re-renders heatmap using inherent_likelihood / inherent_impact values already loaded; visual only |
| O-05 | Heatmap Residual toggle | Overview heatmap section | Click | No API call | Default state; reverts to residual values |
| O-06 | Risk by process bar (bar chart) | Overview, risk summary chart | Click on bar | GET /api/processes/{process_id} | Navigates to process detail |
| O-07 | KPI card (Open Risks, High/Critical, etc.) | Overview KPI row | Click | Client-side navigation to risk register view with pre-applied filter | E.g. clicking "High/Critical" → risk register filtered to rating=High,Critical |
| A-01 | Process card | Area view, process list | Click | GET /api/processes/{process_id} | Navigates to process detail view |
| A-02 | Risk table row | Area view, risk summary table | Click | GET /api/processes/{process_id} | Navigates to process detail and scrolls to the specific risk ID |
| A-03 | "Send all reminders" link | Area view, 1LoD task panel header | Click | POST /api/notifications/remind-all?area={area_id} | Sends reminders to all Pending 1LoD tasks in this area; requires 2LoD role |
| A-04 | Individual reminder send button (send icon) | Area view, 1LoD task list row | Click | POST /api/notifications/remind with {task_id} | Sends reminder to assignee of that specific task; requires 2LoD role |
| A-05 | Escalation button | Area view, 1LoD task list row | Click | POST /api/notifications/escalate with {task_id} | Escalates to assignee's manager; only active if reminder count ≥ 1 and days overdue ≥ 3; requires 2LoD role |
| P-01 | "Start ICS Assessment" button | Process detail view, header area | Click | POST /api/assessments/start with {process_id, cycle_id} | Only available if no In Progress assessment exists for this process/cycle; requires 1LoD (own process) or 2LoD |
| P-02 | "View full detail" button | Process detail view | Click | No API call | Client-side: expands all collapsed risk and control detail panels |
| P-03 | Risk table row expand | Process detail, risk table | Click | No API call (data already loaded) | Inline expand: reveals risk description, linked controls, last assessment notes, residual vs inherent comparison |
| P-04 | Control table row expand | Process detail, control table | Click | No API call (data already loaded) | Inline expand: reveals control description, last test result, next test date, test evidence link |
| P-05 | Risk detail "Edit" button | Risk inline expand panel | Click | Opens inline edit form; PUT /api/risks/{risk_id} on save | Only visible to 2LoD and (for own-process risks) 1LoD with restricted field set |
| P-06 | Risk edit: Save | Risk edit form | Click | PUT /api/risks/{risk_id} | |
| P-07 | Risk edit: Cancel | Risk edit form | Click | No API call | Closes form; reverts to read state |
| G-01 | Quick prompt: "Show all high-risk processes" | Agent chat panel | Click | POST /api/agent/chat | Pre-filled message; see Section below for full message text |
| G-02 | Quick prompt: "Upcoming RCSA due dates" | Agent chat panel | Click | POST /api/agent/chat | |
| G-03 | Quick prompt: "Draft Q3 gap summary" | Agent chat panel | Click | POST /api/agent/chat | |
| G-04 | Chat send button | Agent chat panel input | Click | POST /api/agent/chat | Also triggered by Enter key |
| G-05 | Chat input field | Agent chat panel | Enter key | POST /api/agent/chat | Same as G-04 |
| T-01 | TPRM signal: "Accept" button | TPRM signal review panel | Click | PUT /api/tprm/signals/{signal_id}/accept | Requires 2LoD role; creates draft risk in register |
| T-02 | TPRM signal: "Reject" button | TPRM signal review panel | Click | PUT /api/tprm/signals/{signal_id}/reject | Requires 2LoD role; prompts for rejection reason |
| T-03 | TPRM signal: "Modify before accepting" | TPRM signal review panel | Click | Opens edit form; PUT /api/tprm/signals/{signal_id}/accept on submit with modifications | Allows 2LoD to adjust suggested title, category, rating before creating risk |

---

## Section 1 — Navigation

### N-01: Overview Tab

**Trigger:** Click on "Overview" tab in top navigation bar  
**Action:** Client-side routing to `/overview`  
**Data loading:** On mount, if `overview` data is not in cache or cache is > 5 minutes old, dispatch `GET /api/overview`

**Loading state:**
- Tab switches immediately (no delay)
- KPI cards display skeleton loaders (grey animated rectangles) while awaiting response
- Heatmap area displays a spinner in the centre

**Success state:**
- KPI cards populate with live counts
- Area summary cards render with stats
- Heatmap renders with current residual risk data (default mode)
- TPRM alert banner appears at top if `tprm_pending_signals > 0`

**Error state:**
- Toast notification (top-right): "Could not load dashboard data. Showing last cached data." (if cache exists)
- If no cache: KPI cards display "--" with a warning icon; error message: "Data temporarily unavailable. Try refreshing."
- Retry button appears after 3 seconds of error state

**URL:** `/overview`

---

### N-02 to N-07: Area Tabs

**Trigger:** Click on area name tab (e.g. "Operations & Sourcing")  
**Action:** Client-side routing to `/areas/{area_id}`  
**API call:** `GET /api/areas/{area_id}`

**Loading state:**
- Active tab indicator moves immediately to clicked tab
- Area view content area: skeleton loaders for process cards (3–4 card skeletons), risk table (5-row skeleton), 1LoD task list (3-row skeleton)
- Tab title shows a subtle spinner icon while data is loading

**Success state:**
- Area header renders: area name, summary KPIs (open risks, high/critical count, next RCSA due)
- Process cards grid renders
- Risk table populates with top risks for this area (up to 10, sorted by residual score desc)
- 1LoD task list populates with Pending tasks for this area
- If no pending tasks: "No pending 1LoD tasks for this area." message

**Error state:**
- Error message in the main content area: "Unable to load {area_name} data. Please try again."
- Retry button
- Other area tabs remain accessible (each area is independently loaded)

**URL:** `/areas/P-01` (etc.)

---

## Section 2 — Overview (Landing Page) Elements

### O-01: TPRM Alert Banner "Review risk →"

**Visibility condition:** Only shown when `GET /api/overview` returns `tprm_pending_signals > 0`

**Content:** "TPRM Alert: {count} new risk suggestion(s) from third-party activity. [Review risk →]"  
The count and suggested risk title (if count = 1) are shown.

**Trigger:** Click on "Review risk →" link  
**Action:** Client-side navigation; opens TPRM Signals side panel

**Loading state:** None — opens immediately client-side  
**Success state:** TPRM signal panel slides in from the right; displays list of pending signals with vendor name, trigger event, suggested risk title  
**Error state:** If signal panel cannot load its data (`GET /api/tprm/signals` fails): "Could not load TPRM signals. Please try again."

---

### O-02: Area Card Click

**Location:** Overview page, area summary grid (6 cards, one per area)

**Card content:**
- Area name
- Open risks count (badge)
- High/Critical count
- Control effectiveness indicator (colour dot: green/amber/red)
- RCSA status chip
- Next RCSA due date

**Trigger:** Click anywhere on the card (not just the title)  
**Action:** Client-side navigation to `/areas/{area_id}`; triggers `GET /api/areas/{area_id}`

**Loading/Success/Error:** Same as N-02 to N-07 (navigating to area view).

**Hover state:** Card elevates (box-shadow increases); cursor becomes pointer; border highlights in Accenture purple.

---

### O-03: Heatmap Risk Chip Click

**Location:** 5x5 risk heatmap on Overview page

**Rendering:** Each cell in the heatmap shows a count badge. Clicking a cell expands a popover showing the list of risks in that likelihood/impact position. Clicking a specific risk in the popover navigates to the process detail for that risk.

**Trigger:** Click on a specific risk in the heatmap cell popover  
**Action:** `GET /api/processes/{process_id}` + navigate to `/processes/{process_id}` with scroll target = risk_id

**Loading state:** Process detail view shows skeleton while loading  
**Success state:** Process detail renders; page scrolls to the clicked risk's row and briefly highlights it (yellow background fades after 2 seconds)  
**Error state:** Toast: "Unable to load process detail. Please try again."

---

### O-04 / O-05: Heatmap Inherent / Residual Toggle

**Location:** Heatmap header, toggle button group: [Inherent] [Residual]

**Trigger:** Click on "Inherent" or "Residual" button  
**Action:** Client-side only — no API call  
**Data:** Both inherent and residual values for all risks are included in the `GET /api/overview` response, so the toggle is purely a client-side re-render

**State change:**
- Active button is highlighted (filled style)
- Heatmap cells re-render: risk dots move to their inherent_likelihood/inherent_impact positions (or residual positions)
- Cell colours re-render based on the score at the new position
- Legend updates to clarify "Showing: Inherent Risk" or "Showing: Residual Risk"
- No loading state (data already present)

---

### O-06: Risk by Process Bar Chart Click

**Location:** Overview page, horizontal bar chart showing risk count by process

**Trigger:** Click on a bar (process name)  
**Action:** Navigate to `/processes/{process_id}`; `GET /api/processes/{process_id}`

**Loading/Success/Error:** Same as process navigation (see P-01 onwards).

---

## Section 3 — Area View Elements

### A-01: Process Card Click

**Location:** Area view, process cards grid

**Card content:**
- Process name
- Process owner name
- Open risk count
- Control effectiveness rating (chip: Effective / Partially Effective / Ineffective / Not Assessed)
- RCSA status (chip: Not Started / In Progress / Completed / Overdue)
- Days until next RCSA due

**Trigger:** Click anywhere on the card  
**Action:** `GET /api/processes/{process_id}`; navigate to `/processes/{process_id}`

**Loading state:** Process detail skeleton while `GET /api/processes/{process_id}` is in flight  
**Success state:** Process detail view renders fully  
**Error state:** Toast: "Unable to load process detail for {process_name}."

---

### A-02: Risk Table Row Click (Area View)

**Location:** Area view, risk summary table (top risks in area)

**Table columns:** Risk ID | Risk Title | Process | Category | Residual Rating | Owner | Status | Last Reviewed

**Trigger:** Click on any row  
**Action:** Navigate to `/processes/{process_id}` (derived from the risk's process_id); page scrolls to and highlights the risk row in the process detail risk table

**Anchor:** URL includes hash `#risk-{risk_id}` so the scroll position is restored on browser back navigation

---

### A-03: "Send All Reminders" Link

**Location:** Area view, 1LoD task panel header (top-right of the task list section)

**Visibility:** Only visible to users with 2LoD role. Hidden for 1LoD and ReadOnly users.

**Trigger:** Click  
**Pre-condition check (client-side):** If there are 0 pending tasks in the list, button is disabled with tooltip "No pending tasks to remind"

**Confirmation:** A confirmation modal appears: "Send reminders to all {count} pending 1LoD owners in {area_name}? This will send an email and Teams message to each assignee."  
Two buttons: [Send Reminders] [Cancel]

**API call on confirm:** `POST /api/notifications/remind-all?area={area_id}`

**Loading state:**
- Modal shows a spinner and "Sending..." text
- [Send Reminders] button disabled while in flight

**Success state:**
- Modal closes
- Toast: "Reminders sent to {count} assignees."
- Each task row in the list updates its `reminder_count` badge (increments by 1) and `last_reminder_sent` timestamp

**Error state:**
- Modal remains open
- Error message in modal: "Failed to send reminders. Please try again."
- [Send Reminders] button re-enabled

---

### A-04: Individual Reminder Button

**Location:** Area view, 1LoD task list, each row's action column (paper-plane send icon)

**Visibility:** Only visible to 2LoD role.

**Trigger:** Click on send icon  
**No confirmation modal** (individual send is low-impact; "Send all" has a confirmation)

**API call:** `POST /api/notifications/remind` with body `{ "task_id": "{task_id}" }`

**Loading state:**
- The send icon in the row spins (replaced with a spinner icon) while in flight
- Row is not disabled — other actions on other rows still available

**Success state:**
- Icon returns to normal state
- Row's `reminder_count` badge increments by 1
- Row's "Last reminded" field updates to "Just now"
- Small checkmark flash on the icon for 1 second

**Error state:**
- Icon returns to normal state
- Toast: "Failed to send reminder to {assignee_name}. Please try again."

---

### A-05: Escalation Button

**Location:** Area view, 1LoD task list, each row's action column (chevron-up / escalate icon)

**Visibility:** Only visible to 2LoD role. Only active (not greyed out) when: `task.reminder_count >= 1` AND (`task.days_overdue >= 3` OR user explicitly chooses to escalate).

**Trigger:** Click on escalation icon  
**Confirmation modal:** "Escalate this task to {manager_name} ({manager_email})? A notification will be sent to both {assignee_name} and their manager."  
Buttons: [Escalate] [Cancel]

**API call on confirm:** `POST /api/notifications/escalate` with body `{ "task_id": "{task_id}" }`

**Loading state:** Modal spinner while in flight  
**Success state:**
- Modal closes
- Toast: "Task escalated to {manager_name}."
- Task row status badge updates to "Escalated" (amber badge)
- Escalation icon replaced with a lock icon (no further escalations from UI — one escalation per task)

**Error state:**
- Modal remains open
- Error: "Escalation failed. Please try again or contact the system administrator."

---

## Section 4 — Process Detail View

### P-01: "Start ICS Assessment" Button

**Location:** Process detail view, action bar at the top right

**Visibility:**
- Visible to 1LoD (for their own process) and 2LoD
- Hidden if current assessment status is "In Progress" or "1LoD Complete" (assessment already running)
- Disabled with tooltip "No assessment cycle configured" if no upcoming cycle_id is found

**Trigger:** Click  
**Confirmation modal:** "Start a new RCSA assessment for {process_name} — {cycle_id} ({due_date})? This will assign the assessment to {assigned_to} and send them a notification."  
Buttons: [Start Assessment] [Cancel]

**API call on confirm:** `POST /api/assessments/start` with body:
```json
{
  "process_id": "PROC-001",
  "cycle_id": "Q3-2025"
}
```

**Loading state:** Modal spinner; button disabled  
**Success state:**
- Modal closes
- Toast: "Assessment started. {assigned_to} has been notified."
- Assessment status chip in process detail header updates from "Not Started" to "In Progress"
- Button text changes to "Assessment In Progress" and is disabled until cycle completes

**Error state:**
- Modal: "Failed to start assessment. This may be because an assessment is already in progress. Refresh the page and try again."

---

### P-02: "View Full Detail" Button

**Location:** Process detail view, below the process header summary

**Trigger:** Click  
**Action:** Client-side only — expands all collapsed sections:
- Risk detail panels (all risks expand to show description, linked controls, history)
- Control detail panels (all controls expand to show test evidence, findings)

**Toggle behaviour:**
- Button text changes to "Collapse Detail"
- Clicking again collapses all panels

**No loading state** — all data was loaded with the initial `GET /api/processes/{process_id}` call.

---

### P-03: Risk Table Row Expand (Process Detail)

**Location:** Process detail view, risk table, each row

**Row content (collapsed):** Risk ID | Risk Title | Category | Inherent Rating | Residual Rating | Owner | Status

**Trigger:** Click on row (or expand chevron)  
**Action:** Client-side inline panel expands below the row. Content:
- Risk description (full text)
- Regulatory reference (if any)
- Linked controls list (control IDs + titles + effectiveness ratings)
- Risk history: bar chart showing residual score over the last 4 quarters
- Last assessed date and by whom
- Next review date (highlighted red if overdue)
- Notes from last 2LoD review
- Edit button (for authorised roles)

**No API call** — all this data is included in the `GET /api/processes/{process_id}` response.

**Animation:** Expand panel slides down with a 200ms CSS transition.

---

### P-04: Control Table Row Expand (Process Detail)

**Location:** Process detail view, control table, each row

**Row content (collapsed):** Control ID | Control Title | Type | Category | Effectiveness | Last Tested | Next Test Due

**Trigger:** Click on row  
**Action:** Client-side inline panel expands. Content:
- Control description
- Control owner and operator
- Testing methodology
- Last test result: Pass / Pass with Exceptions / Fail (with date and tested-by)
- Exceptions noted (if any)
- Evidence link (opens in new tab if URL; displays reference string if not a URL)
- Next test date (highlighted amber if within 14 days; red if overdue)
- Finding linked to this control (if any open finding)

**No API call** — all data included in `GET /api/processes/{process_id}` response.

---

### P-05 / P-06 / P-07: Risk Edit Form

**Location:** Risk inline expand panel (P-03), accessible via "Edit" button

**Visibility of Edit button:**
- 2LoD role: can edit any risk
- 1LoD role: can edit their own process's risks, but only the following fields: `status` (limited options: Open → In Remediation only), `notes`, `next_review_date`
- ReadOnly: no edit button shown

**Trigger (P-05):** Click "Edit" button in risk expand panel  
**Action:** The read-only fields in the expand panel transform into editable form fields. Fields that are editable for the current user's role are active; others remain read-only.

**Editable fields (2LoD):**
- `status` (select dropdown)
- `residual_likelihood` (1–5 select)
- `residual_impact` (1–5 select)
- `risk_owner_id` (user search dropdown)
- `next_review_date` (date picker)
- `notes` (text area)

**Editable fields (1LoD, own process only):**
- `status`: can change Open → In Remediation only
- `notes` (text area)

**Trigger (P-06):** Click "Save" button  
**API call:** `PUT /api/risks/{risk_id}` with body containing only the changed fields (partial update):
```json
{
  "status": "In Remediation",
  "residual_likelihood": 3,
  "residual_impact": 3,
  "notes": "Mitigating actions underway as of 2025-07-15",
  "next_review_date": "2025-10-01"
}
```

**Loading state on save:**
- Save button shows spinner; disabled
- Form fields disabled while in flight

**Success state:**
- Form collapses back to read view
- Updated values are reflected immediately in the expand panel (optimistic UI is not used; values come from the API response)
- Toast: "Risk {risk_id} updated successfully."
- Parent risk table row updates its rating chips if residual_score changed
- `agent_audit_log` entry created server-side

**Error state:**
- Form remains open
- Error message below form: "Failed to save changes. Please try again."
- Save button re-enabled; changed values are preserved in form (not lost)

**Trigger (P-07):** Click "Cancel"  
**Action:** Form reverts to read state with original values; no API call; no data loss prompt if fields were modified (simple cancel, no confirmation needed for minor edits)

---

## Section 5 — Agent Chat Panel

The agent chat panel is always accessible via the side panel icon (or a dedicated "AI Assistant" tab). It persists its conversation history within the session.

### G-01: Quick Prompt — "Show all high-risk processes"

**Location:** Agent chat panel, quick-prompt suggestion buttons (shown when no conversation has started, or at the bottom of an ongoing conversation)

**Trigger:** Click on suggestion chip  
**Action:** Populates the message input with the predefined text and immediately submits (no extra user action needed)

**Message sent to API:**
```json
{
  "message": "Show me all processes with High or Critical residual risk ratings. List them by area, include the risk count and the top risk for each process.",
  "session_id": "{current_session_id}",
  "context": {
    "current_view": "overview",
    "current_area_id": null,
    "current_process_id": null
  }
}
```

**Expected agent behaviour:** Executes `SQLQueryTool` to retrieve processes with high/critical residual risks; formats as a structured list with area grouping; includes risk counts.

---

### G-02: Quick Prompt — "Upcoming RCSA due dates"

**Message sent:**
```json
{
  "message": "What RCSA assessments are due in the next 30 days? Show me the process name, area, assigned owner, due date, and current status. Flag any that are overdue.",
  "session_id": "{current_session_id}",
  "context": { "current_view": "overview" }
}
```

**Expected agent behaviour:** Executes `SQLQueryTool` against `rcsa_assessments` table filtering by due_date between today and today+30days; formats as a table with overdue items highlighted.

---

### G-03: Quick Prompt — "Draft Q3 gap summary"

**Message sent:**
```json
{
  "message": "Draft a Q3 gap summary for 2LoD review. Include: (1) Number of open risks by area and rating. (2) Controls rated Ineffective or Partially Effective by area. (3) Overdue RCSA assessments. (4) Any risks where residual rating exceeds risk appetite. Keep it concise for a management summary.",
  "session_id": "{current_session_id}",
  "context": { "current_view": "overview" }
}
```

**Expected agent behaviour:** Executes multiple `SQLQueryTool` calls; synthesises into a structured management summary format with sections; presents draft text that user can copy.

---

### G-04 / G-05: Chat Send (Button / Enter Key)

**Trigger:** Click "Send" button or press Enter key in the message input  
**Pre-condition:** Input field must not be empty (send button is disabled and Enter key does nothing if input is empty)

**API call:** `POST /api/agent/chat`

**Request body:**
```json
{
  "message": "{userInput}",
  "session_id": "{current_session_id}",
  "context": {
    "current_view": "{current_view}",
    "current_area_id": "{current_area_id_or_null}",
    "current_process_id": "{current_process_id_or_null}"
  }
}
```

**The API responds via SSE (Server-Sent Events). The frontend listens using the `EventSource` API (or a streaming fetch with `ReadableStream`).**

**Loading state:**
- Message input is cleared immediately and disabled while response is streaming
- Send button is disabled
- A "typing indicator" appears in the chat (three animated dots) while the first token has not yet arrived
- Once first token arrives, typing indicator is replaced by the streaming text

**Streaming state:**
- Response text renders word-by-word as tokens arrive
- Markdown is rendered progressively (bold, tables, code blocks)
- User can scroll up to read previous messages while streaming continues
- A "Stop" button appears during streaming to allow the user to abort the SSE stream

**Success state (stream complete):**
- Final `data: {"type": "done", ...}` event received
- Source citations (if any) appear below the response in a collapsed "Sources" section
- Message input re-enabled; send button re-enabled
- Conversation history updated in local state
- Session ID preserved for subsequent messages

**Error state:**
- If SSE connection fails or a `data: {"type": "error"}` event is received:
  - Error message appears in the chat: "I couldn't process your request. Please try again."
  - Input re-enabled immediately
- If timeout (> 30 seconds with no response): "The request timed out. The query may be too complex. Please try a more specific question."

**Multi-turn context:** `session_id` is generated client-side (UUID) at session start and passed on every request. The server-side agent uses this to retrieve conversation history from Redis (last 10 turns) and include it in the LLM context.

**Shift+Enter:** Inserts a newline in the message without submitting (for multi-line questions).

---

## Section 6 — TPRM Signal Review

### T-01: Accept TPRM Signal

**Location:** TPRM Signals panel (opened from O-01 banner) or TPRM Signals dedicated page

**Visibility:** Only 2LoD role can accept/reject. ReadOnly and 1LoD see signals but not action buttons.

**Trigger:** Click "Accept" button on a pending signal card

**Confirmation modal:**
```
Accept this TPRM Risk Suggestion?

Vendor: Acme Cloud Services Ltd
Trigger: Vendor classification changed: Standard → Critical
Suggested Risk: "Critical vendor dependency on Acme Cloud — potential concentration risk"
Category: Operational
Inherent Rating: High (L4 × I4)
Suggested Controls: [list]

This will create a new draft risk in the Risk Register. A risk owner will need to be assigned.

[Accept and Create Risk]  [Cancel]
```

**API call on confirm:** `PUT /api/tprm/signals/{signal_id}/accept` with body `{}` (or with modifications if T-03 was used)

**Loading state:** Modal spinner  
**Success state:**
- Modal closes
- Toast: "Risk RSK-XXXX created from TPRM signal. View in risk register."
- Toast includes a link to the newly created risk
- Signal card in the panel updates to "Accepted" status (green chip)
- Signal moves to a "Resolved" tab in the panel (no longer in "Pending" list)
- Overview KPI card "Open Risks" increments by 1

**Error state:**
- Modal: "Failed to create risk. Please try again or create the risk manually."

---

### T-02: Reject TPRM Signal

**Trigger:** Click "Reject" button  
**Prompt for reason:**
- Small inline form appears: text area "Reason for rejection (required for audit purposes)"
- [Confirm Rejection] [Cancel]

**API call on confirm:** `PUT /api/tprm/signals/{signal_id}/reject` with body `{ "reason": "{reason_text}" }`

**Loading/Success/Error:** Same pattern as T-01. Signal moves to "Rejected" tab.

---

### T-03: Modify Before Accepting

**Trigger:** Click "Modify & Accept" button  
**Action:** Opens a form pre-populated with the suggested risk fields. User can edit: `risk_title`, `risk_description`, `risk_category`, `inherent_likelihood`, `inherent_impact`, `suggested_controls`

**API call on submit:** `PUT /api/tprm/signals/{signal_id}/accept` with body:
```json
{
  "modifications": {
    "risk_title": "Modified title",
    "inherent_likelihood": 3,
    "inherent_impact": 4
  }
}
```

---

## Section 7 — State Management

### Client-Side vs Server-Driven State

| State | Where Managed | Cache TTL | Notes |
|-------|--------------|-----------|-------|
| Overview KPI data | React Query (server state) | 5 minutes | Stale-while-revalidate |
| Area data | React Query | 5 minutes | Refetched on tab click if stale |
| Process detail | React Query | 2 minutes | More volatile (risk updates, assessment status) |
| Risk register (filtered list) | React Query | 2 minutes | |
| Heatmap mode (inherent/residual) | Zustand (client state) | Session | Persisted to localStorage |
| Active tab / current view | URL params (React Router) | — | Bookmarkable, browser-back navigable |
| Agent chat history | Zustand | Session | Not persisted across browser sessions |
| Agent session_id | Zustand | Session | UUID generated on first chat interaction |
| User role / permissions | React context (from JWT) | Token lifetime | Decoded from Bearer token on app load |
| Notification toast queue | Zustand | — | Auto-dismiss after 5 seconds |
| Expanded row IDs (risk/control tables) | Zustand | Session | |

---

## Section 8 — Real-Time Updates

### SSE Channels (Server-Sent Events)

The dashboard opens a persistent SSE connection on app load to receive push updates for time-sensitive events.

**SSE endpoint:** `GET /api/events/stream` (auth: Bearer token in query param or header)

**Events pushed:**

| Event type | Payload | Dashboard action |
|-----------|---------|-----------------|
| `tprm_signal_new` | `{ signal_id, vendor_name, count }` | Show/update TPRM alert banner; increment signal count; show toast |
| `rcsa_overdue` | `{ assessment_id, process_name, area_name }` | Update RCSA status chip to "Overdue" if process is currently visible; increment "Overdue RCSAs" KPI |
| `risk_updated` | `{ risk_id, process_id, updated_by }` | If the risk is currently in the viewed process, invalidate React Query cache for that process and trigger refetch |
| `reminder_sent` | `{ task_id, sent_to }` | Update the task row's reminder count and timestamp if area view is active |
| `assessment_status_change` | `{ assessment_id, new_status }` | Update status chip in process/area views if currently visible |

**Reconnection:** The frontend `EventSource` auto-reconnects on disconnect. If SSE is unavailable (client network, proxy), the app falls back to polling `GET /api/overview` every 60 seconds.

---

## Section 9 — Responsive Behaviour

**Primary target:** Desktop (1920×1080 and 1440×900). The dashboard is designed for knowledge workers on laptops/desktops.

**Tablet (1024×768):** Supported in a simplified layout:
- Navigation tabs collapse to a hamburger/dropdown menu
- Area cards reduce from 3-column to 2-column grid
- Heatmap maintains 5x5 grid but with smaller cells; tooltips on hover replaced by tap-to-expand
- Agent chat panel docks below main content rather than as a side panel

**Mobile:** Not a supported primary target. A read-only summary view is acceptable on mobile (KPI cards only); action buttons (reminders, assessments) are desktop-only.

**Print/Export:** A "Print view" CSS class renders a clean, header/footer-free version for browser print. "Export to PDF" option generates a PDF server-side from the current view data (Azure Function generates using WeasyPrint or Puppeteer).

---

## Section 10 — Accessibility (WCAG 2.1 AA)

| Requirement | Implementation |
|-------------|---------------|
| Keyboard navigation | All interactive elements reachable by Tab; action buttons have visible focus rings; modal focus trapped |
| Screen reader support | ARIA labels on all icon-only buttons; role attributes on heatmap cells; live regions (`aria-live`) for toast notifications and streaming chat responses |
| Colour contrast | All text ≥ 4.5:1 contrast ratio; heatmap colour coding uses both colour and pattern/icon to avoid colour-only information encoding |
| Risk rating colours | Critical = red, High = orange, Medium = yellow, Low = green; all cells also have text labels for colour-blind users |
| Skip navigation | "Skip to main content" link as first focusable element |
| Form labels | All form inputs have explicit `<label>` elements; error messages associated via `aria-describedby` |
| Streaming chat | `aria-live="polite"` region for chat responses; "Response loading" announced to screen readers on send |

---

*End of document. Reference: [templates/ics_dashboard.html](../templates/ics_dashboard.html)*
