# ProjectARC — Autonomous Revenue Cycle

> **"Payers deny. ARC fights back."**

An autonomous medical billing denial-resolution and revenue-recovery system built on **UiPath Maestro Case**, combining LLM agents, RPA, human-in-the-loop checkpoints, and asynchronous outcome monitoring into a single self-routing case.

**UiPath AgentHack 2026 — Track 1: UiPath Maestro Case**

---

## Table of Contents

1. [The Business Problem](#the-business-problem)
2. [What ProjectARC Does](#what-projectarc-does)
3. [Solution Architecture](#solution-architecture)
4. [The Four Resolution Paths](#the-four-resolution-paths)
5. [Agents](#agents)
6. [RPA Processes](#rpa-processes)
7. [Human-in-the-Loop Checkpoints](#human-in-the-loop-checkpoints)
8. [Shared Patterns](#shared-patterns)
9. [UiPath Components Used](#uipath-components-used)
10. [Technology Stack](#technology-stack)
11. [Mock Payer API](#mock-payer-api)
12. [Test Data](#test-data)
13. [Setup & Run Instructions](#setup--run-instructions)
14. [Design Principles](#design-principles)
15. [Roadmap](#roadmap)
16. [License](#license)

---

## The Business Problem

US hospitals lose an estimated **$262 billion every year** to denied insurance claims. Industry-wide, **5–10% of all submitted claims are denied** on first pass — and the process to recover that revenue is overwhelmingly manual.

When a claim is denied, a hospital's revenue cycle team must:

- Read the denial code (CARC) and determine *why* the claim was rejected
- Decide which of several **completely different** remediation paths applies — a coding error is fixed nothing like a clinical-necessity dispute, which is fixed nothing like an eligibility lapse or a duplicate-billing error
- Gather the right supporting evidence — corrected codes, physician clinical notes, updated insurance information — often from different systems
- Route to the right human for sign-off — a coder, a physician, a patient access representative — based on the denial type
- Resubmit through the correct channel and track the outcome
- Do all of this **before a hard timely-filing deadline** (typically 90–365 days). Once that window closes, the revenue is permanently written off.

Today every one of these steps is done manually, across spreadsheets, payer portals, and phone calls, while deadlines quietly expire in the background. **There is no orchestration layer that understands "this type of denial needs this fix, this agent, and this human."**

This is one of the largest preventable revenue leaks in the entire US healthcare system, and it persists because no existing tool treats a denied claim as what it actually is: **a dynamic, branching case** that should route itself to the right combination of AI, automation, and human judgment.

---

## What ProjectARC Does

ProjectARC turns every denied claim into a **managed, self-routing case** on UiPath Maestro Case. The moment a claim is denied:

1. **Classifies** the denial — an LLM agent reads the EOB and CARC code and determines the denial category
2. **Routes** the case automatically to the correct one of four resolution paths
3. **Builds the fix** — corrected billing codes, a physician-ready clinical appeal letter, a patient outreach message, or a billing-correction decision — using a purpose-built agent for that path
4. **Holds for a human** at the exact point where liability, clinical judgment, or compliance requires it
5. **Submits** the resolution to the payer via RPA, with **explicit success/failure handling**
6. **Monitors the outcome asynchronously** — waiting, polling the payer, and resolving to approved, denied, or still-pending
7. **Notifies stakeholders** by email at every meaningful outcome — submission failure, final denial, or approval

Nothing in ARC is a single fixed workflow. It runs four structurally different processes under one case, branching dynamically based on what the denial actually is and how it unfolds.

---

## Solution Architecture

Every claim enters through a shared front-end, then diverges:

```
                  ┌─────────────────────────────────┐
                  │   Claim Denied — EOB Received    │
                  └────────────────┬────────────────┘
                                   │
                  ┌────────────────▼────────────────┐
                  │   Denial Classification Agent    │   (LLM)
                  │   Reads EOB → outputs denial_path│
                  └────────────────┬────────────────┘
                                   │
                  ┌────────────────▼────────────────┐
                  │       Denial Path Router         │   (Exclusive Gateway)
                  └──┬──────────┬──────────┬─────────┘
                     │          │          │         │
         ┌───────────▼──┐ ┌─────▼─────┐ ┌──▼──────┐ ┌▼────────────┐
         │   Path A     │ │  Path B   │ │ Path D  │ │   Path E    │
         │ Coding Error │ │ Clinical  │ │Eligibil-│ │ Duplicate / │
         │              │ │ Necessity │ │  ity    │ │ Technical   │
         └──────────────┘ └───────────┘ └─────────┘ └─────────────┘
```

All four paths run as branches of one UiPath Maestro Case. The router reads the `denial_path` produced by the Denial Classification Agent and dispatches the case to the correct branch automatically — no human decides where a case goes.

---

## The Four Resolution Paths

### Path A — Coding Error

Handles denials caused by incorrect billing codes (wrong modifier, unbundled codes, diagnosis-procedure mismatch).

**Flow:**
```
Denial Path Router
   → Coding Correction Agent (LLM proposes corrected codes)
   → HITL: Review Corrected Codes (Medical Coder approves)
   → API Call: SubmitToPayer
   → Gateway: Check API Call Status
        ├─ API Failed → Send Path A failure email → Terminate
        └─ API Succeeded → Inform Claim Creator
               → Timer Event (wait)
               → Appeal Status Polling Sequence
                     ├─ pending & poll_count < 3 → poll again
                     ├─ denied   → Send denial confirmation email (with reason) to stakeholder
                     └─ approved → Send approval confirmation email (with details) to stakeholder
```

**Why a human is required:** A medical coder must approve corrected codes before resubmission — submitting incorrect codes carries real compliance and fraud risk.

---

### Path B — Clinical Necessity

Handles denials where the payer deems a service not medically necessary — the most evidence-intensive path.

**Flow:**
```
Denial Path Router
   → Clinical Evidence Agent (LLM drafts an evidence-based appeal letter)
   → HITL: Clinical Evidence Review
   → Gateway: which button did the reviewer click?
        ├─ "Request Changes" → route back to Clinical Evidence Agent (regenerate)
        └─ "Approve & Sign"  → API Call: SubmitToPayer
               → Gateway: Check API Call Status
                    ├─ API Failed → Send Path B failure email → Terminate
                    └─ API Succeeded → Timer Event (wait)
                          → Appeal Status Polling Sequence
                                ├─ pending & poll_count < 3 → poll again
                                ├─ denied   → Send denial confirmation email (with reason)
                                └─ approved → Send approval confirmation email (with details)
```

**Why a human is required:** A physician must review and sign the appeal letter — a physician signature is a legal requirement for clinical-necessity appeals. The Request-Changes loop lets the physician send the draft back for regeneration rather than accepting it as-is.

---

### Path D — Eligibility

Handles denials caused by coverage lapses or coordination-of-benefits issues — the only path where resolution depends on information the patient holds.

**Flow:**
```
Denial Path Router
   → Patient Outreach Agent (LLM drafts a HIPAA-compliant outreach message)
   → HITL: Review Patient Outreach Content
   → Gateway: splits into three branches
        ├─ "Regenerate"    → route back to Patient Outreach Agent (regenerate content)
        ├─ "Edit Manually" → RPA drafts the outreach content and saves it as a mail draft
        │                     for manual editing
        └─ "Send"          → Send the outreach message to the patient as an email
                              → End
```

**Why a human is required:** A patient access representative must review the message before it is sent — patient communication must comply with HIPAA's minimum-necessary standard. The three-way gateway gives the reviewer real choice: regenerate, hand-edit, or send as-is.

**Note:** The end of Path D (patient email sent) is the current terminal point. A planned enhancement uses UiPath Document Understanding to extract the patient's email reply and automatically resume the case for resubmission.

---

### Path E — Duplicate / Technical

Handles denials flagged as duplicates — which may be true clearinghouse duplicates, or technical billing errors (missing bilateral modifiers, NPI conflicts, technical/professional component splits) that merely *look* like duplicates.

**Flow:**
```
Denial Path Router
   → Billing Review Agent (LLM diagnoses the issue and decides if resubmission is needed)
   → Gateway: requires_resubmission?
        ├─ false → Send "No Duplicate Resubmission" email → Terminate
        └─ true  → API Call: SubmitToPayer
               → Gateway: Check API Call Status
                    ├─ API Failed → Send Path E failure email → Terminate
                    └─ API Succeeded → Timer Event (wait)
                          → Appeal Status Polling Sequence
                                ├─ pending & poll_count < 3 → poll again
                                ├─ denied   → Send denial confirmation email (with reason)
                                └─ approved → Send approval confirmation email (with details)
```

**Why this path is distinctive:** It is the only path where ARC may correctly determine that **no payer action is required at all** — a true duplicate is resolved internally by voiding, with no resubmission. This proves the system does not blindly escalate every case; it matches the action to the actual problem.

---

## Agents

ProjectARC uses **five LLM agents**, built in UiPath Agent Builder, each handling reasoning over unstructured information:

| Agent | Path | Responsibility |
|-------|------|----------------|
| **Denial Classification Agent** | All | Reads the EOB and CARC code, outputs the denial path (A/B/D/E) that drives routing |
| **Coding Correction Agent** | A | Identifies the specific coding error and proposes corrected CPT/ICD codes |
| **Clinical Evidence Agent** | B | Reads physician clinical notes and drafts an evidence-based medical-necessity appeal letter |
| **Patient Outreach Agent** | D | Drafts a HIPAA-compliant patient outreach message requesting updated insurance information |
| **Billing Review Agent** | E | Diagnoses whether a "duplicate" denial is a true duplicate or a disguised technical error, and decides whether resubmission is required |

---

## RPA Processes

Built in UiPath Studio, each handling deterministic actions (no LLM reasoning):

| Process | Purpose |
|---------|---------|
| **SubmitToPayer** | Makes the HTTP POST to the payer system to submit an appeal/resubmission; returns acknowledgment and status |
| **Appeal Status Polling** | Polls the payer system for the current claim status (pending / approved / denied) |
| **Draft Outreach (Path D)** | Saves the patient outreach content as an editable mail draft when the reviewer chooses to edit manually |

---

## Human-in-the-Loop Checkpoints

Every human checkpoint in ARC exists because of a real liability, clinical, or compliance requirement — none are token approvals.

| Checkpoint | Path | Who | Why a human is required |
|------------|------|-----|--------------------------|
| Review Corrected Codes | A | Medical Coder | Resubmitting incorrect codes carries compliance and fraud risk |
| Clinical Evidence Review | B | Physician | A physician signature is legally required for clinical-necessity appeals |
| Review Patient Outreach | D | Patient Access Rep | Patient communication must meet HIPAA's minimum-necessary standard |

Each checkpoint offers **real choices**, not just approve/reject:
- Path B reviewer can **Request Changes** to regenerate the appeal letter
- Path D reviewer can **Regenerate**, **Edit Manually**, or **Send**

---

## Shared Patterns

Three patterns are reused across multiple paths, giving the system consistency:

**1. API Submission with Failure Handling (Paths A, B, E)**
Every payer submission is followed by an explicit status gateway. If the API call fails, a path-specific failure email is sent to stakeholders and the workflow terminates gracefully — no silent failures.

**2. Asynchronous Outcome Monitoring (Paths A, B, E)**
After a successful submission, the case waits (Timer Event), then enters a polling sequence. While the status is `pending` and the poll count is under the threshold, it loops and polls again. When the payer resolves the claim, it branches to a final outcome.

**3. Stakeholder Email Notifications (Paths A, B, E)**
Every terminal outcome notifies stakeholders by email:
- **Denied** → a denial confirmation email explaining *why* the appeal was denied
- **Approved** → an approval confirmation email containing the claim approval details

---

## UiPath Components Used

| Component | How ARC uses it |
|-----------|-----------------|
| **UiPath Maestro Case** | Core orchestration — case definition, denial path router, exclusive gateways, timer events, human tasks, polling loops |
| **UiPath Agent Builder** | Hosts all five LLM agents |
| **UiPath Studio (RPA)** | Payer API submission, status polling, draft-mail creation |
| **UiPath Orchestrator** | Task Catalog human-task forms, process publishing, queues, scheduling, credentials |

---

## Technology Stack

- **Orchestration:** UiPath Maestro Case (UiPath Automation Cloud)
- **Agents:** UiPath Agent Builder
- **LLM reasoning engine:** Claude (Anthropic)
- **RPA:** UiPath Studio
- **Mock Payer Portal:** FastAPI (Python) — simulates payer claim-status, appeal, and authorization behavior
- **Email notifications:** integrated stakeholder email at each terminal outcome

---

## Mock Payer API

A FastAPI service (`mock_payer_api.py`) simulates a payer's claims system so the solution can be demonstrated reliably and repeatably. Key endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/claim/{claim_id}/status` | GET | Poll a claim's status; auto-progresses after 3 polls based on path recovery rates |
| `/claim/{claim_id}/appeal` | POST | Submit an appeal/resubmission; resets the claim to `pending` and returns an acknowledgment |
| `/claim/status/override` | POST | Demo control — force a claim to a specific status (used live during the demo) |
| `/claims/all` | GET | Dashboard — all claim states plus a summary of total recovered amount |

The API holds 20 synthetic claims in memory, each with a starting status, payer, denial path, and amount.

---

## Test Data

The project includes a complete synthetic dataset (no real PHI):

- **20 EOBs** spanning all denial types, each with claim ID, patient, payer, date of service, CARC code, billed amount, recommended fix, and timely-filing deadline
- **Clinical notes** for clinical-necessity claims, used by the Clinical Evidence Agent to draft appeal letters
- **Configuration** mapping CARC codes to denial paths, payer filing deadlines, and escalation thresholds

---

## Setup & Run Instructions

### Prerequisites
- UiPath Automation Cloud account with Maestro and Agent Builder access
- UiPath Studio (Desktop)
- Python 3.9+ (for the mock payer API)

### 1. Start the Mock Payer API
```bash
cd mock-api
pip install fastapi uvicorn
uvicorn mock_payer_api:app --reload --port 8000
```
Confirm it's live at `http://localhost:8000` and explore endpoints at `http://localhost:8000/docs`.

### 2. Publish the RPA Processes
Open each Studio process (SubmitToPayer, Appeal Status Polling, Draft Outreach), confirm the API endpoint points to your running mock API, and publish to your Orchestrator folder.

> **Localhost note:** If the robot runs in the cloud, it cannot reach your laptop's `localhost`. Run the processes via a robot installed on your own machine, or expose the API publicly (e.g. via a tunneling tool) and update the endpoint.

### 3. Publish the Agents
In Agent Builder, confirm each agent's prompt, model, and input/output schema, then publish each one.

### 4. Configure & Publish the Maestro Case
- Confirm the case variables, the Denial Path Router conditions, and all gateway conditions
- Confirm each human task is linked to its Task Catalog form
- Publish the case to Orchestrator

### 5. Run a Case
Start a new case instance with a denied-claim payload. Watch it classify, route, build the fix, pause for human approval, submit, poll, and notify stakeholders.

---

## Design Principles

**Right tool for each job.** Agents are used where reasoning over unstructured information is required (classification, appeal drafting, patient communication, billing diagnosis). RPA is used where the action is deterministic (API submission, status polling, draft creation). ARC never uses an LLM where a rule or API call would do — and Path E's no-resubmission branch is deliberate proof of that discipline.

**Genuine human-in-the-loop.** Every human checkpoint maps to a real liability, clinical, or compliance requirement — and each gives the reviewer real choices, not just a rubber stamp.

**Graceful failure.** Every API submission is followed by an explicit status check. Failures send a notification and terminate cleanly rather than leaving a case stuck.

**Asynchronous by design.** Cases genuinely pause and resume on external outcomes (payer decisions), rather than simulating waits with timers that always succeed.

**Quantifiable impact.** Every path resolves to a stakeholder notification with a clear outcome — approved (revenue recovered), denied (with reason), or no-resubmission-needed.

---

## Roadmap

- **UiPath Document Understanding** as a front-of-pipeline layer — ingesting raw payer EOB PDFs instead of structured input
- **Document Understanding for Path D** — automatically extracting the patient's email reply and resuming the case for resubmission
- **Path C — Authorization Issue** — RPA auth-lookup followed by agent-driven remediation (simple correction vs retro-authorization with coordinator review)
- **Live payer portal RPA** in place of the mock API, for production-grade submission and polling
- **Escalation Agent** with ROI-based legal-vs-write-off recommendation for the Revenue Cycle Director
- **Expanded CARC code coverage** across all paths

---

## License

This project is released under the MIT License.

---

*ProjectARC — Autonomous Revenue Cycle. Built for UiPath AgentHack 2026.*
*"Payers deny. ARC fights back."*
