# Real Estate Lead Qualification & AI Outreach — n8n Workflow

An end-to-end automation that captures real estate leads from a web form, uses an AI agent to score and classify them (Hot / Warm / Cold), routes them to the right team channel and outreach sequence, and handles failures and duplicate submissions gracefully.

Built to replace manual lead triage — where every enquiry gets the same generic response regardless of how ready the seller actually is — with intent-based scoring that tells the sales team who to call first and why.

---

## What it does

1. A seller fills out a property enquiry form (name, suburb, property type, ownership, timeline, asking price, prior agent contact).
2. The workflow checks Airtable for an existing record with that email to **prevent duplicate processing**.
3. A new lead is passed to an **AI Agent** (Groq / Llama 3.3 70B) that scores it against explicit business rules — timeline urgency, ownership status, suburb tier, and competitive context — and returns a structured classification: `Hot`, `Warm`, or `Cold`.
4. The lead is logged to **Airtable** as the single source of truth, along with the AI's reasoning and recommended action.
5. Based on classification, the team gets a **Slack alert** with the right urgency level, and the lead receives a **personalized outreach email**.
6. 48 hours later, Hot and Warm leads automatically receive a **follow-up email**, and their Airtable status updates.
7. If anything fails along the way — the AI Agent errors out, returns an unparseable response, or the lead is a duplicate — the workflow logs it and notifies the team instead of failing silently.

---

## Why intent-based scoring

Not every enquiry is ready to sell this month. Treating a "just exploring" browser the same as an owner who wants to list within two weeks wastes the agent's time on one and loses the other to a competitor who responded faster. This workflow encodes real qualifying logic — suburb demand tier, ownership clarity, timeline, and whether the lead has already spoken to other agents — into the classification, so the team's time and follow-up cadence match how ready each lead actually is.

---

## Architecture

```
Form Submission
      │
      ▼
Search Airtable (dedupe check by email)
      │
      ├── Duplicate found ──► Slack: duplicate alert (skip)
      │
      └── New lead
            │
            ▼
      AI Agent (Groq/Llama 3.3) — classify Hot/Warm/Cold
            │
            ├── Agent fails (API/technical error)
            │         └──► Slack alert + Airtable log (Status: Needs Manual Review)
            │
            └── Agent responds
                  │
                  ▼
            Parse & normalize classification
                  │
                  ├── Classification empty/unparseable
                  │         └──► Slack alert + Airtable log (Status: Needs Manual Review)
                  │
                  └── Valid classification
                        │
                        ▼
                  Airtable: create lead record
                        │
                        ▼
                  Switch by classification
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
      Hot             Warm            Cold
        │               │               │
  Slack (urgent)   Slack (24h)     Slack (nurture)
        │               │               │
  Outreach email   Outreach email  Outreach email
        │               │
   Wait 48hrs       Wait 48hrs
        │               │
  Follow-up email  Follow-up email
        │               │
  Airtable update  Airtable update
   (Status: Follow-Up sent)
```

---

## Key design decisions

**Single source of truth.** Every lead is written to Airtable exactly once at classification time, then updated in place as its status changes. Earlier versions of this workflow wrote to multiple Google Sheets tabs and Airtable in parallel — this was simplified to avoid systems disagreeing about lead state.

**Two distinct failure modes, handled separately.**
- *AI Agent hard failure* (API error, timeout) — caught via the node's error output, logged with `Classification: Error`.
- *AI Agent responds but classification can't be determined* — caught via an explicit empty-check after parsing, logged with `Status: Needs Manual Review`.

Splitting these matters: one is a technical/infrastructure problem, the other is a scoring edge case. Conflating them into one generic "something went wrong" branch would make debugging harder and lose the distinction between "the AI is down" and "this lead is genuinely ambiguous."

**Dedupe before scoring.** A duplicate submission is caught before it ever reaches the AI Agent, so the same lead can't be scored and outreached twice from a double form submission.

**Accurate timing.** Follow-up delays use real Wait node durations (48 hours) — not placeholder values left over from testing.

---

## Tech stack

| Component | Tool |
|---|---|
| Workflow engine | [n8n](https://n8n.io) |
| Form intake | n8n Form Trigger |
| AI classification | Groq (Llama 3.3 70B Versatile) via LangChain AI Agent node |
| Structured output | n8n Structured Output Parser |
| Data store | Airtable |
| Team notifications | Slack |
| Client outreach | Gmail |

Groq/Llama was chosen over a larger frontier model for this classification task specifically for cost and latency — lead scoring is a well-bounded task with explicit rules, not open-ended reasoning, so a fast, cheap model is the right fit here.

---

## Setup

1. Import `workflow.json` into your n8n instance.
2. Connect credentials for:
   - Airtable (Personal Access Token)
   - Slack (OAuth or Bot Token)
   - Gmail (OAuth2)
   - Groq (API key)
3. Update the Airtable `base` and `table` IDs to point at your own base. Expected schema:

   | Field | Type |
   |---|---|
   | Name | Single line text |
   | Email | Single line text |
   | Suburb | Single line text |
   | Property Type | Single select |
   | Ownership | Single line text |
   | Timeline | Single line text |
   | Asking Price | Single line text |
   | Other Agents | Single line text |
   | Classification | Single select (Hot / Warm / Cold / Error) |
   | AI Reason | Long text |
   | Recommended Action | Long text |
   | Status | Single select (New / Follow-Up sent / Booked / Needs Manual Review) |
   | Source | Single line text |

4. Update Slack `channelId` values to your own workspace channels.
5. Update the Calendly link and agent signature in the Gmail nodes to your own booking link and business details.
6. Adjust the classification rules inside the AI Agent's system prompt to match your market (suburb tiers, timeline thresholds) — the current rules are tuned for a Brisbane, Australia context.

---

## Known limitations / possible next steps

- **Cold-lead re-engagement is not yet automated.** The Slack copy references leads "re-qualifying after 90 days," but no scheduled check currently exists — this is planned as a separate, decoupled scheduled workflow (event-driven intake and time-driven re-engagement don't belong in the same trigger).
- **Duplicate submissions are logged but not merged.** A repeat enquiry from the same email is currently just flagged and skipped, rather than updating the original record's engagement signal.
- **No automated retry** on transient Slack/Gmail/Airtable API failures — currently just logged for manual follow-up.

---

## Screenshots

<img width="1877" height="752" alt="Screenshot 2026-07-02 164315" src="https://github.com/user-attachments/assets/ae7c0826-e3dd-49fe-92f7-f2b2489ea120" />
