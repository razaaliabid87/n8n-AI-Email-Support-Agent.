# AI Email Support Agent

An n8n-based email automation system that ingests incoming Gmail messages (including PDF/TXT attachments), classifies and scores them with Gemini, runs a full sales-qualification and policy-decision pipeline, and executes outcomes through a custom job-queue/worker architecture — with drafted AI replies, CRM logging, and graceful, honest degradation for anything not yet automated.

## Why this exists

Most "AI email agent" demos are a trigger, one LLM call, and a canned reply. This project is built around the reality that inbound business email isn't uniform: it includes attachments that need extraction, senders that need reputation checks, departments that need different policies, and decisions that sometimes should *not* be automated. The system is designed to make that distinction explicitly, rather than force every email down the same path.

## Architecture

```
Gmail Trigger → Fetch full message → Spam gate
   │
   ▼
Attachment handling (PDF/TXT extraction, normalization, metadata)
   │
   ▼
Email Classification Engine (Gemini, structured output)
   → category, subcategory, priority, sentiment, confidence, requires_human
   │
   ▼
Enterprise Decision Engine
   → automation_allowed / route (rule-based: low confidence, Legal, critical
     Finance, sensitive IT, unknown category, etc. force human review)
   │
   ▼
Department Router (Automation Allowed / Not Allowed)
   → 9 departments: Sales, Support, HR, Finance, Marketing, IT,
     Operations, Meeting, General (+ IT/Finance/Legal/Spam/Unknown
     on the not-allowed side)
   │
   ├─→ [Sales branch — fully automated end-to-end]
   │      Sales Opportunity Analyzer (lead scoring, buying intent)
   │      → Environment / Sender Intelligence / Score Optimizer rules engines
   │      → Organization / Routing / Compliance policy engines
   │      → Execution Planner → Validator → Queue → Dispatcher
   │      → Job Lifecycle Manager → typed Workers:
   │           CRM Worker (Airtable create/update)
   │           Email Worker (AI-drafted reply → Gmail draft)
   │           Notify Sales / Notify Manager stubs
   │
   └─→ [All other departments — honest stub]
          → routed to a generic "not yet implemented" stub that reports
            SKIPPED status with the classification and routing reason
            attached, rather than silently dropping the email
```

## Key engineering decisions

**A rule-based decision layer sits between "AI classified it" and "we act on it."** The Enterprise Decision Engine explicitly blocks automation for low AI confidence, unknown categories, Legal correspondence, critical Finance transactions, and sensitive IT requests — regardless of what the classifier says. This is a deliberate human-in-the-loop boundary, not a missing feature: some categories of email should never be fully automated, and the system says so instead of trying.

**Sales gets a fully built pipeline; every other department gets an honest placeholder, not silence.** Only the Sales branch currently has a complete downstream pipeline (lead scoring, policy engines, execution queue, workers). The other eight departments — and the sub-categories on the "not allowed" side — route to a single reusable stub node that reports `SKIPPED, not yet implemented` along with the classification data, rather than the email simply vanishing. This was a deliberate build sequence: prove the full pattern on one department end-to-end before replicating it, and make the untouched paths visible rather than silent while that work is pending.

**A custom job-queue/worker architecture, not a straight linear chain.** Execution Planner → Validator → Queue → Worker Dispatcher → Job Lifecycle Manager → typed Workers is a deliberate scaffold: each qualifying email can spawn multiple jobs (CRM update, email draft, sales notification, manager notification), each tracked with its own execution ID, retry count, and state, rather than one monolithic action per email.

**AI-drafted replies go to Gmail Drafts, not auto-send.** For a sales inquiry, the system drafts a reply but stops short of sending it automatically — keeping a human in the loop for AI-generated customer-facing text, while still automating the drafting work.

**Retry and explicit failure paths on every external call.** Airtable and Gmail calls use `retryOnFail` with capped attempts and `continueErrorOutput`; 14 distinct `Stop and Error` nodes each guard a genuinely different failure point (spam, unsupported file type, classification failure, sales-analysis failure, individual Airtable/Gmail call failures) rather than one generic catch-all.

## Tech stack
- **Orchestration:** n8n (custom JS code nodes + LangChain nodes)
- **Classification & analysis:** Google Gemini (primary + fallback model pairs throughout)
- **Email:** Gmail (trigger, message fetch, draft creation)
- **CRM/state tracking:** Airtable
- **Auth:** n8n credential store (no inline API keys)

## Known limitations (honest, as of this version)
- **Only the Sales department has a fully automated downstream pipeline.** Support, HR, Finance, Marketing, IT, Operations, Meeting, and General currently route to a stub that reports "not yet implemented" — classified and visible, but not acted on.
- **Stub output isn't persisted yet.** The generic stub generates a `SKIPPED` status object, but it isn't currently written to Airtable or any other log — so there's no queryable record yet of "how many Support/HR/Finance emails came in unhandled." This is the next planned addition.
- **Attachment support is PDF and TXT only.** DOCX, XLSX, and image attachments currently fail the run via an explicit error rather than degrading gracefully — a known gap, not a silent one.
- **Notification workers (Notify Sales, Notify Manager) and several job types (Calendar, Audit, Analysis, Task) are stubs**, following the same "visible, not silent" pattern as the department routing.

## Setup requirements
- n8n instance with `@n8n/n8n-nodes-langchain` package installed
- Gmail OAuth credential with read + draft-creation scopes
- Airtable base with a leads/CRM table (see "CRM Worker" node for expected schema)
- Google Gemini API credential (stored in n8n credential manager)
## Screenshot
<img width="1690" height="367" alt="AI Email Support Agent Screenshot" src="https://github.com/user-attachments/assets/d71242cb-aeff-412b-8f1f-d380fa459c1d" />
