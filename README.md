# Hospital Pre-Authorization Automation

Redesigning the cashless insurance pre-authorization workflow at Indian private hospitals using AI and automation.
## Demo

[![Watch the demo](https://img.shields.io/badge/Loom-Watch%20Demo-orange)](https://www.loom.com/share/d274937a74ee444bbe21ea9efe74014c)
**Assignment writeup:** [Read the full document](https://docs.google.com/document/d/1OKAvxzBNgOzvMJcG-ldG1E8Cpe0jjL74hXunomw17Kg/edit?tab=t.0)

---

## The Problem

Every day, thousands of patients arrive at private hospital insurance desks hoping to use their health insurance for cashless treatment. What should be a smooth process is instead a broken, manual nightmare — for the coordinator, and for the patient.

The coordinator logs into a different portal for each TPA, copies the same patient data multiple times across different forms, chases the lab and radiology department over WhatsApp for reports, submits everything via email or portal, and then waits. Six hours on average. No real-time visibility. No alerts. Just periodically checking an inbox.

When the TPA responds — and it is usually with a query asking for more documents or clinical detail — the coordinator reads it, walks to the doctor, gets clarification, drafts a response from scratch, and starts again. Each query cycle adds 4 to 24 hours to the process.

In emergency cases, the family signs a liability form committing to pay the full bill if the TPA later denies the claim. They sit in a waiting room not knowing if they owe two lakh rupees or nothing.

The result: a first-pass approval rate of 40 to 60 percent. The majority of submissions require at least one rework cycle. Coordinators spend 2 to 3 hours per case on work that is almost entirely copy-paste and chasing.

**Why this problem can be solved now:**

Three things have converged to make this tractable in a way that wasn't possible before. NHCX — the National Health Claims Exchange — is now live as a unified API layer connecting hospitals to all 34+ insurers and TPAs in India. IRDAI has mandated a one-hour response time for pre-auth queries, creating regulatory pressure on TPAs to move faster. And large language models can now reliably extract structured medical codes from free-text clinical notes — the single hardest step to automate in this workflow. Together, these three create a window to rebuild the entire process.

---

## What We Built

A working prototype of the pre-authorization workflow built on n8n, demonstrating the full logical flow from admission to TPA response.

**Tools used:** n8n cloud · Claude Haiku (Anthropic API) · Google Sheets · Google Drive · Apps Script

### How the prototype works

**Phase 1 — Admission and ICD extraction**

A webhook trigger simulates an admission event from the HMS, receiving a patient payload with free-text diagnosis and procedure notes. Claude reads the clinical text and returns structured ICD-10 diagnosis codes, CPT procedure codes, and a clinical justification summary. The extracted data is written to a Google Sheet with status set to Pending Review. The coordinator reviews the ICD codes in the sheet and changes status to Approved. An Apps Script trigger detects this edit and fires a webhook back to n8n.

**Phase 2 — Checklist and document collection**

n8n runs a mock policy check and generates a claim ID. It then creates a new tab in the Google Sheet named after the claim ID — one isolated checklist per patient. The document checklist is written to this tab based on the procedure and TPA, with each required document listed along with its upload status. The coordinator uploads PDFs to a Google Drive folder and pastes the links into the checklist.

**Phase 3 — Document send and TPA response**

When the coordinator changes the action cell to Send, Apps Script batches all uploaded documents and fires a second webhook. n8n fetches each file from Drive, extracts the binary data, and passes the full case context to Claude. Claude drafts a professional response to the TPA query, listing the attached documents by name and flagging any pending documents as to follow shortly. The Google Sheet is updated with the draft response and response sent status. The action cell resets to prevent double submission.

**What is real vs mocked:**

| Component | Status |
|---|---|
| Claude ICD extraction | Live API call |
| Claude query response drafting | Live API call |
| Google Sheets and Drive | Live |
| HMS data | Mocked as webhook payload |
| NHCX API | Mocked — hardcoded responses |
| TPA query | Hardcoded for demo |

---

## The Full Vision

The prototype demonstrates the workflow logic. The production version would connect real systems at each layer.

### Data layer — HMS to FHIR

Hospital HMS systems emit patient data as HL7 v2 messages. Mirth Connect sits between the HMS and the workflow engine, translating these messages into FHIR-compliant bundles — structured JSON containing patient demographics, admission details, diagnosis as free text, and insurance policy information. This is the integration layer that makes the rest of the automation possible without requiring hospitals to replace their existing HMS.

### Intelligence layer — Claude

Claude handles the two steps in the workflow where unstructured text needs to become structured action. First, extracting ICD-10 and CPT codes from the doctor's clinical notes — replacing the most error-prone manual step with a reviewed AI suggestion. Second, drafting TPA query responses based on case context, attached documents, and the specific query text — compressing a 4 to 24 hour cycle into 10 to 15 minutes. A human review checkpoint sits before both outputs leave the system.

### Submission layer — NHCX

Instead of logging into a different portal for each TPA, the system makes a single API call to NHCX. The complete FHIR bundle — patient data, clinical codes, and all supporting documents base64-encoded as DocumentReference resources — travels as one structured payload. NHCX routes it to the correct TPA. Status updates arrive via webhook rather than inbox polling. The coordinator gets a push notification the moment the TPA responds rather than discovering it during a manual check.

### Document layer — LIS and RIS integration

In production, lab reports are pulled directly from the LIS and radiology reports from the RIS when they are ready, rather than requiring the coordinator to chase departments. For hospitals where these integrations are not yet in place, the system falls back to the manual upload flow demonstrated in the prototype. The coordinator experience is identical — the automation simply removes the manual steps progressively as integrations are established.

### Learning layer — query prediction

Every submission outcome is logged: which TPA, which procedure, which documents were submitted, whether the submission was queried, and what was asked for. Over time this builds a pattern layer. Before submission, the system flags which additional documents a specific TPA is likely to request for a specific procedure based on historical data — pre-empting queries before they happen. This is the mechanism that pushes first-pass approval rates toward 85 percent.

---

## Expected Impact

| Metric | Today | Target |
|---|---|---|
| First-pass approval rate | 40–60% | 85%+ |
| Coordinator time per case | 2–3 hours | 25–35 minutes |
| Query response turnaround | 4–24 hours | 10–15 minutes |
| TPA portals to manage | One per TPA | Single unified flow |

---

## Repository

```
## Repository

| File | Description |
|---|---|
| `n8n-workflow.json` | Full n8n workflow JSON — import directly into n8n |
| `Current Flow.png` | Current manual pre-authorization flow |
| `proposed flow.png` | Redesigned AI-assisted flow |
| `mock n8n flow.png` | n8n prototype workflow diagram |
| `n8n flow.png` | n8n canvas screenshot |
```
