# AI Leads Qualifier — Portfolio Project

An AI-powered lead qualification system built with n8n that automatically scores incoming leads as HOT, WARM, or COLD using Google's Gemini AI. Qualified leads are logged to Google Sheets, and hot leads trigger instant email alerts. A daily scheduled digest reminds the team of uncontacted hot leads.

---

## What It Does

### Flow 1 — Lead Intake & AI Qualification (Webhook-triggered)

When a lead submits a form:

1. **Tally form** submits lead data via webhook
2. **tallyParser** — reshapes Tally's field array into flat usable fields
3. **emailLookUp** — checks Google Sheets for duplicate email
4. **ifNoDuplicate** — branches:
   - Duplicate found → responds with `{ success: false }` and stops
   - No duplicate → continues to AI qualification
5. **httpQualifyLeadAI** — sends lead details to **Gemini 2.5 Pro** with structured JSON response schema
6. **responseParser** — strips markdown fences and parses the AI response
7. **classificationTag** — maps AI fields to clean variables
8. **leadsList** — appends full lead record to Google Sheets (`LEADS` tab)
9. **webhookRespondHot** — responds to the webhook with classification result
10. **ifHotLead** — if HOT → sends instant **Gmail alert** with reason and suggested action
11. **Error path** — if Gemini fails → `errorTag` → logs to Google Sheets (`ERRORS` tab)

### Flow 2 — Daily Follow-up Reminder (Scheduled)

Every day at **8:00 AM**:

1. **Schedule Trigger** fires
2. **checkLeadsList** — reads all rows from the `LEADS` sheet
3. **If** — filters for HOT leads where `Contacted` is empty
4. **Code in JavaScript** — aggregates uncontacted HOT leads into a digest
5. **followUpReminder** — sends a Gmail digest listing all leads needing follow-up

---

## Tech Stack

| Tool | Purpose |
|---|---|
| [n8n](https://n8n.io) | Workflow automation engine (self-hosted) |
| [Tally](https://tally.so) | Lead capture form with webhook integration |
| [Gemini 2.5 Pro](https://ai.google.dev) | AI lead scoring and classification |
| [Google Sheets](https://sheets.google.com) | Lead database and error log |
| [Gmail](https://gmail.com) | Hot lead alerts and daily follow-up reminders |
| Docker + Nginx + SSL | Self-hosted infrastructure |

---

## Workflow Architecture

```
Tally Form (webhook)
  └── tallyParser
        └── leadsData
              └── emailLookUp (Google Sheets)
                    └── ifNoDuplicate
                          ├── Duplicate → Respond to Webhook (success: false)
                          └── No Duplicate → httpQualifyLeadAI (Gemini 2.5 Pro)
                                ├── [Error] → errorTag → errorList (Google Sheets ERRORS tab)
                                └── responseParser
                                      └── classificationTag
                                            └── leadsList (Google Sheets LEADS tab)
                                                  └── webhookRespondHot
                                                        └── ifHotLead
                                                              └── HOT → hotLeadsEmail (Gmail)

Schedule Trigger (daily 8AM)
  └── checkLeadsList (Google Sheets)
        └── If (HOT + not contacted)
              └── Code in JavaScript (aggregate)
                    └── followUpReminder (Gmail digest)
```

---

## AI Classification Logic

The Gemini prompt classifies leads using confidence scoring:

| Score | Classification |
|---|---|
| 0% – 40% | COLD |
| 41% – 79% | WARM |
| 80% – 100% | HOT |

Gemini returns structured JSON with enforced schema (`responseMimeType: application/json`):

```json
{
  "classification": "HOT",
  "confidence_level": 92,
  "reason_for_confidence": "High budget, urgency signals, specific pain point",
  "suggested_action": "Call within 2 hours"
}
```

---

## Google Sheets Structure

### LEADS tab

| Column | Description |
|---|---|
| Timestamp | Date of submission |
| Name | Lead's full name |
| Email address | Lead's email |
| Company | Lead's company |
| Budget (monthly) | Stated budget |
| Message | Lead's message |
| Classification | HOT / WARM / COLD |
| Confidence level | AI confidence score (0–100) |
| Reason for confidence | AI reasoning |
| Suggested action | AI recommended next step |
| Contacted | Mark "yes" when followed up |

### ERRORS tab

| Column | Description |
|---|---|
| Timestamp | When the error occurred |
| Error code | API error code |
| Error Status | HTTP status |
| Error message | Error description |
| Error name | Error type |
| Error details | Full stack trace |

---

## Environment Variables

All credentials are stored as n8n credentials in the UI. No secrets are hardcoded in the workflow.

The Gemini API key is stored as a **Header Auth** credential:
- **Name:** `x-goog-api-key`
- **Value:** your Gemini API key

---

## Required Credentials

| Service | Type | Notes |
|---|---|---|
| Gemini API | Header Auth (`x-goog-api-key`) | Use Gemini 2.5 Pro for best results |
| Google Sheets | OAuth2 | Needs read/write access to the spreadsheet |
| Gmail | OAuth2 | Needs send scope |

---

## Tally Form Fields

The form collects these fields (labels must match exactly):

| Tally Label | Mapped to |
|---|---|
| Full name | `body.name` |
| Email | `body.email` |
| Company name | `body.company` |
| Monthly budget | `body.budget` |
| What do you need help with? | `body.message` |

---

## VPS & Infrastructure Setup

See the [main repository README](../README.md) for full VPS, Docker, Nginx, and SSL setup instructions.

---

## Testing

Test with a curl mimicking Tally's webhook format:

```bash
curl -X POST https://automation.jheilabs.com/webhook-test/lead-qualifier \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "FORM_RESPONSE",
    "data": {
      "fields": [
        { "label": "Full name", "value": "Juan dela Cruz" },
        { "label": "Email", "value": "juan@example.com" },
        { "label": "Company name", "value": "Acme Corp" },
        { "label": "Monthly budget", "value": "50000" },
        { "label": "What do you need help with?", "value": "We need full CRM automation urgently. Board approved budget." }
      ]
    }
  }'
```

Click **Listen for test event** on the Webhook node before running the curl.

---

> Built as a portfolio project demonstrating AI-powered lead qualification with real-world integrations and automated follow-up workflows.
