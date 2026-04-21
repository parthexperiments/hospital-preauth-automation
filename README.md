# Hospital Pre-Authorization Automation

AI-assisted pre-authorization workflow for Indian private hospitals — built with n8n, Claude API, Google Sheets, and Apps Script. Redesigns the manual TPA cashless claim process end to end.

---

## Problem

Insurance coordinators at private hospitals spend 2–3 hours per cashless admission on manual, fragmented work — logging into multiple TPA portals, re-entering patient data, chasing documents over WhatsApp, waiting 6+ hours for responses, and drafting query replies from scratch. First-pass approval rates sit at 40–60%, meaning most cases require at least one rework cycle.

This workflow was designed after visiting CK Birla Hospital, Jaipur and speaking with the Insurance Desk Coordinator there.

---

## What This Builds

An end-to-end automated pre-authorization workflow that:

- Extracts ICD-10 codes from free-text clinical notes using Claude
- Generates a per-patient document checklist based on procedure and TPA
- Tracks document uploads via Google Drive
- Drafts TPA query responses using Claude with attached documents listed
- Updates a Google Sheet as the coordinator interface throughout

---

## Flow Overview

### Phase 1 — Admission
```
Admission webhook → Claude ICD extraction → Sheet1 written (Pending Review)
→ Coordinator approves → Apps Script fires preauth-approved webhook
```

### Phase 2 — Checklist
```
Mock policy check → Claim ID generated → NHCX checklist tab created
→ Document rows written per procedure + TPA rules
```

### Phase 3 — Document Send
```
Coordinator uploads PDFs to Drive → Sets action to Send
→ Apps Script fires document-send webhook
→ Drive files fetched → Claude drafts TPA response
→ Sheets updated: Response Sent, action reset
```

---

## Diagrams

### Current State
![Current state flow](diagrams/current-state-flow.png)

### Redesigned Flow
![Redesigned flow](diagrams/redesigned-flow.png)

### n8n Prototype
![n8n prototype](diagrams/n8n-prototype-flow.png)

---

## Tech Stack

| Layer | Tool |
|---|---|
| Workflow automation | n8n cloud |
| AI — ICD extraction + query drafts | Claude Haiku (Anthropic API) |
| Coordinator interface | Google Sheets |
| Document storage | Google Drive |
| Event trigger | Google Apps Script |
| HMS data (mocked) | n8n webhook payload |
| NHCX API (mocked) | Hardcoded JSON responses |

---

## Repo Structure

```
hospital-preauth-automation/
├── README.md
├── workflows/
│   ├── preauth-main-workflow.json      # Phase 1 + 2: admission to checklist
│   └── preauth-document-flow.json      # Phase 3: document send to response
└── diagrams/
    ├── current-state-flow.png
    ├── redesigned-flow.png
    └── n8n-prototype-flow.png
```

---

## Setup

### Prerequisites
- n8n cloud account
- Anthropic API key
- Google account with Sheets and Drive access

### Steps

1. Import both JSON files from `workflows/` into n8n
2. Set your Anthropic API key in both Claude nodes
3. Connect your Google Sheets and Google Drive credentials in n8n
4. Create a Google Sheet with these columns in Sheet1:
   ```
   patient_name | age | policy_number | tpa | procedure_raw | diagnosis_raw |
   icd_code | procedure_code | clinical_justification | status | claim_id |
   tpa_response | draft_response
   ```
5. Update the Sheet ID in all Google Sheets nodes to your sheet's ID
6. Add the Apps Script code to your sheet (Extensions → Apps Script) — see below
7. Activate both workflows in n8n
8. Test by sending a POST to your `preauth-trigger` webhook URL

### Apps Script

Paste this into your Google Sheet's Apps Script editor and set the trigger to `onEdit → From spreadsheet → On edit`:

```javascript
function onEdit(e) {
  handleApproval(e);
  handleDocumentSend(e);
}

function handleApproval(e) {
  const sheet = e.source.getActiveSheet();
  const range = e.range;
  if (sheet.getName() !== 'Sheet1') return;
  if (range.getColumn() !== 10) return;
  if (range.getValue() !== 'Approved') return;

  const row = range.getRow();
  const data = sheet.getRange(row, 1, 1, 13).getValues()[0];

  const payload = {
    row_number: row,
    patient_name: data[0], age: data[1], policy_number: data[2],
    tpa: data[3], procedure_raw: data[4], diagnosis_raw: data[5],
    icd_code: data[6], procedure_code: data[7], clinical_justification: data[8],
    status: data[9], claim_id: data[10], tpa_response: data[11], draft_response: data[12]
  };

  UrlFetchApp.fetch('YOUR_N8N_URL/webhook/preauth-approved', {
    method: 'POST', contentType: 'application/json',
    payload: JSON.stringify(payload), muteHttpExceptions: true
  });
}

function handleDocumentSend(e) {
  const sheet = e.source.getActiveSheet();
  const range = e.range;
  const sheetName = sheet.getName();
  if (!sheetName.startsWith('NHCX')) return;
  if (range.getColumn() !== 6) return;
  if (range.getValue() !== 'Send') return;

  const row = range.getRow();
  const claimId = sheet.getRange(row, 1).getValue();
  const lastRow = sheet.getLastRow();
  const allData = sheet.getRange(2, 1, lastRow - 1, 9).getValues();

  const documents = allData
    .filter(r => r[0] === claimId && r[3] === 'Uploaded' && r[4] !== '' && r[8] === '')
    .map(r => ({
      claim_id: r[0], patient_name: r[1], document_name: r[2],
      status: r[3], drive_link: r[4],
      file_id: r[4].match(/\/d\/([a-zA-Z0-9_-]+)/)?.[1] || '',
      response_sent: r[8]
    }));

  const mainSheet = e.source.getSheetByName('Sheet1');
  const mainData = mainSheet.getDataRange().getValues();
  const claimRow = mainData.find(r => r[10] === claimId);

  const payload = {
    claim_id: claimId,
    patient_name: claimRow?.[0] || '',
    tpa: claimRow?.[3] || '',
    diagnosis_raw: claimRow?.[5] || '',
    procedure_raw: claimRow?.[4] || '',
    icd_code: claimRow?.[6] || '',
    clinical_justification: claimRow?.[8] || '',
    tpa_query: claimRow?.[11] || '',
    documents: documents
  };

  UrlFetchApp.fetch('YOUR_N8N_URL/webhook/document-send', {
    method: 'POST', contentType: 'application/json',
    payload: JSON.stringify(payload), muteHttpExceptions: true
  });
}
```

Replace `YOUR_N8N_URL` with your n8n instance URL.

### Test Payload

Send this POST to `YOUR_N8N_URL/webhook/preauth-trigger` to test:

```json
{
  "patient_name": "Rajesh Sharma",
  "age": 54,
  "policy_number": "SH-123456",
  "tpa": "Star Health",
  "procedure_raw": "Coronary Angiography",
  "diagnosis_raw": "Patient presented with chest pain, history of hypertension and diabetes, ECG showing ST changes, planned for coronary angiography"
}
```

---

## What's Mocked vs Real

| Component | Status |
|---|---|
| Claude ICD extraction | Real API call |
| Claude query response drafting | Real API call |
| Google Sheets read/write | Real |
| Google Drive file fetch | Real |
| HMS data | Mocked as webhook payload |
| NHCX policy check | Mocked — hardcoded response |
| NHCX claim submission | Mocked — random claim ID generated |
| TPA query response | Hardcoded for demo |

In production, Mirth Connect would translate HL7 v2 HMS messages into FHIR bundles, and NHCX APIs would handle eligibility checks and claim submission.

---

## Target Impact

| Metric | Today | With automation |
|---|---|---|
| First-pass approval rate | 40–60% | 85%+ |
| Coordinator time per case | 2–3 hours | 25–35 minutes |
| Query response turnaround | 4–24 hours | 10–15 minutes |
| TPA portals to manage | One per TPA | Single workflow |
