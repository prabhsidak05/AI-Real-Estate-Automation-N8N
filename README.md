# AI Real Estate Automation Platform

An end-to-end automation system built in **n8n** that streamlines the complete real estate customer journey — from lead capture to booking confirmation and post-visit feedback — using AI-driven decision making, human approval steps, and scheduled/webhook-triggered workflows.

**Domain:** Real Estate & Property Management
**Tools used:** n8n, Google Sheets, Gmail, Google Calendar, Google Drive & Docs, Google Maps / OpenStreetMap APIs, OpenAI, Google Forms + Apps Script

---

## 1. Problem Statement

Real estate agencies manage property listings, customer inquiries, site visits, legal documentation, and follow-ups across multiple platforms. As the number of properties and customers grows, manually managing leads and coordinating property visits becomes increasingly difficult, often resulting in delayed responses and lost business opportunities.

This project builds an AI-powered Real Estate Automation Platform that streamlines property management and customer engagement — capturing leads from multiple sources, recommending suitable properties based on customer preferences, scheduling site visits, managing property documents, automating follow-ups, and collecting customer feedback after visits, while giving agents visibility into sales performance.

### Business Context
Agencies today juggle spreadsheets, WhatsApp chats, and phone calls to manage the customer journey. This doesn't scale — leads get missed, visits get double-booked, documents get scattered, and there's no systematic feedback loop to catch problems before they cost a sale.

### Stakeholders
| Stakeholder | Interest |
|---|---|
| Agency Owner / Manager | Sales performance visibility, revenue, team productivity |
| Sales Agents | Fewer manual tasks, better-qualified leads, visit scheduling help |
| Customers / Leads | Fast responses, relevant property suggestions, easy scheduling |
| Property Owners / Landlords | Listing visibility, timely updates on interest |
| Legal/Admin Team | Accurate, digitally tracked documentation |
| Marketing Team | Lead source performance data |

### Pain Points Addressed
- Leads from multiple channels weren't centralized → slow or missed responses
- Manual property matching wasted agent time and often mismatched customer needs
- Site visit scheduling via phone/WhatsApp caused double-bookings and no-shows
- Documents were scattered across emails and drives
- No systematic feedback loop → lost insight into why deals fell through
- No consolidated dashboard for sales performance or customer preference trends

### Objectives
1. Automate property inquiry capture and centralization
2. Recommend properties intelligently based on customer preference (budget, location, type)
3. Simplify and automate site visit scheduling with reminders
4. Digitize document handling and booking approvals
5. Automate feedback collection and generate sales/customer analytics

---

## 2. System Architecture

At the center of the system is a shared **data layer** (Google Sheets acting as a lightweight CRM) with six tables: `Leads`, `Properties`, `Visits`, `Bookings`, `Feedback`, and `Dashboard`. Five independent n8n workflows read from and write to this shared data layer, each handling one stage of the customer journey.

```
                    ┌─────────────────────────────────────────┐
                    │              DATA LAYER                  │
                    │   Leads | Properties | Visits | Bookings │
                    │        | Feedback | Dashboard            │
                    └───────────────▲───────────────▲──────────┘
                                    │               │
        ┌───────────────────────────┼───────────────┼───────────────────────────┐
        │                           │               │                           │
┌───────▼────────┐        ┌─────────▼───────┐ ┌─────▼──────────┐      ┌─────────▼────────┐
│ WF1: Lead       │──────▶│ WF2: Qualify &   │ │ WF3: Visit      │─────▶│ WF4: Document      │
│ Collection      │ exec  │ Recommend        │ │ Scheduling      │ exec │ Booking            │
│ (Webhook)       │       │ (Exec Wkflw /    │ │ (Exec Wkflw)    │      │ (Webhook)          │
└─────────────────┘       │  Webhook)        │ └─────────────────┘      └─────────┬──────────┘
                          └─────────┬────────┘                                    │
                                    │                                             ▼
                                                              ┌───────────────────────────┐
                                                              │ WF5: Feedback Dashboard    │
                                                              │ (Cron ×2 + Webhook)        │
                                                              └───────────────────────────┘
```

### Event Flow Between Workflows
1. **WF1** captures a lead via webhook → normalizes fields, routes by source, geocodes location, validates, and upserts into `Leads` → in parallel, sends a welcome email and calls **WF2**.
2. **WF2** scores the lead with AI, matches properties by budget/type/proximity, and emails a recommendation with a pre-filled "confirm interest" form link. When the customer responds, a second webhook (a separate entry point on the same WF2 canvas) looks up the lead and property, merges them, and hands off to **WF3**.
3. **WF3** cleans the requested date/time, checks calendar availability, books the visit, fetches directions, and sends confirmations to both customer and agent.
4. A booking-intent form (sent after the visit) triggers **WF4** via webhook, which merges lead and property data, generates an agreement document, routes it through a human approval step, and confirms or cancels the booking accordingly.
5. **WF5** runs three independent chains: a daily check that sends feedback requests for visits completed the day before, a webhook that processes incoming feedback with AI sentiment analysis and alerts the manager on negative results, and a weekly scheduled chain that aggregates Leads/Bookings/Feedback into a summary report.

---

## 3. Workflows

### WF1 — Lead Collection
**Trigger:** Webhook (from a Google Form via Apps Script)
Captures inbound leads, normalizes the fields, routes them by source, geocodes their preferred location, validates the record, and upserts it into the `Leads` sheet — then, in parallel, sends a welcome email to the customer and calls WF2 (Qualification & Recommendation) to continue the journey.

**Nodes:** Webhook → Edit Fields → Switch (route by lead source) → HTTP Request (OpenStreetMap Nominatim geocoding) → Edit Fields1 → Code in JavaScript (validation) → Append or Update Row in Sheet → *(parallel)* Send a Message (Gmail) **and** Call 'My workflow 2'

> Note: Geocoding uses the free **OpenStreetMap Nominatim API** rather than paid Google Maps Geocoding, avoiding any billing dependency for this step.

### WF2 — Qualification Recommendation
**Two independent entry points on one canvas:**

- **Chain A (from WF1):** When Executed by Another Workflow → Message a Model (AI lead scoring) → Parse AI (Code) → If (qualified?) → Get Row(s) in Sheet (Properties) → Code in JavaScript1 (location/budget/type matching algorithm) → Message a Model1 (AI-generated recommendation text) → Send a Message (Gmail, with pre-filled interest-confirmation form link) → Update Row in Sheet (lead status).
- **Chain B (interest confirmation):** Webhook (customer confirms interest via form) → Get Row(s) in Sheet1 (Leads) + Get Row(s) in Sheet2 (Properties) → Merge → Code in JavaScript → Call 'My workflow 3' (hands off to WF3).

### WF3 — Visit Scheduling
**Trigger:** When Executed by Another Workflow (from WF2)
Cleans and timezone-corrects the requested visit date/time, checks the agent's Google Calendar for availability, and — if free — books the event, fetches directions, and notifies both customer and agent before logging the visit.

**Nodes:** When Executed by Another Workflow → Date Clearance (Code, ISO/IST formatting) → Get Availability in a Calendar → If (slot free?) → Create an Event → HTTP Request (Google Maps Directions) → Customer Message (Gmail) → Agent Message (Gmail) → Append Row in Sheet (`Visits`)

### WF4 — Document Booking
**Trigger:** Webhook (booking-intent form)
Looks up the lead and property in parallel, merges them, creates a dedicated Google Drive folder, generates and updates an agreement document, moves it into the client folder, and routes the booking through a **human approval step** before branching on the outcome.

**Nodes:** Webhook → *(parallel)* Lead Data + Properties Data (Sheets lookups) → Merge → Create Folder → Create a Document → Update a Document → Move File → Send Message and Wait for Response (human approval) → If (approved?) → **true:** Append Row in Sheet + Send a Message (booking confirmed) | **false:** Append Row in Sheet1 + Send a Message1 (booking cancelled)

### WF5 — Feedback Dashboard
**Three independent entry points on one canvas:**

- **Chain A (weekly summary):** Schedule Trigger1 (weekly) → Get Feedback + Get Bookings + Get Leads → Merge → Code in JavaScript3 (KPI aggregation) → Send a Message2 (weekly report to manager).
- **Chain B (daily feedback request):** Schedule Trigger (daily) → Get Row(s) in Sheet (Visits) → Code in JavaScript (filter visits completed yesterday) → Send a Message (Gmail feedback request with pre-filled form link) → Update Row in Sheet (mark visit as feedback-requested).
- **Chain C (feedback processing):** Receive & Process Feedback (Webhook) → Get Row(s) in Sheet1 (Leads) → Message a Model (AI sentiment analysis) → Code in JavaScript1 (parse AI response) → Append Row in Sheet (`Feedback`) → If (negative sentiment?) → **true:** Send a Message1 (manager alert) → Code in JavaScript2 → Update Row in Sheet1 (dashboard KPIs)

---

## 4. Advanced Features Implemented

| Feature | Where Implemented |
|---|---|
| AI-powered decision making | WF2 (lead scoring, location-based property matching, recommendation generation), WF5 (sentiment analysis) |
| Human approval step(s) | WF4 — booking approval via Send and Wait for Approval before any booking is confirmed |
| Error handling & retry logic | Retry-enabled API nodes (Calendar, Directions, Sheets) across WF1–WF5 |
| Logging and audit trail | Google Sheets acts as the running record of every lead, visit, booking, and feedback event across the journey |
| Scheduled workflows (Cron) | WF5 (daily feedback requests, weekly sales summary via two separate Schedule Triggers) |
| Webhook-triggered workflows | WF1 (lead capture), WF2 (interest confirmation), WF4 (booking intent), WF5 (feedback submission) |
| Conditional branching and loops | IF/Switch nodes throughout — lead qualification, calendar availability, booking approval, sentiment routing |
| Parallel branching & data merging | WF1 (parallel Gmail + WF2 handoff), WF2 (parallel Leads + Properties lookups merged before WF3 handoff), WF4 (parallel Lead + Properties lookups merged before document generation), WF5 (three-way merge of Feedback/Bookings/Leads for weekly reporting) |

---

## 5. Tech Stack & Integrations

- **Automation engine:** n8n
- **Data store:** Google Sheets (Leads, Properties, Visits, Bookings, Feedback, Dashboard)
- **Communication:** Gmail (transactional emails, reminders, alerts)
- **Scheduling:** Google Calendar
- **Documents:** Google Drive & Google Docs
- **Location services:** OpenStreetMap Nominatim API for geocoding (free, no billing dependency), Google Maps Directions API for turn-by-turn directions to confirmed visits
- **AI:** OpenAI (lead scoring, property recommendation generation, sentiment analysis)
- **Lead & form capture:** Google Forms + Apps Script (webhook bridge into n8n)

---

## 6. Setup Instructions

1. Import each workflow JSON from `/workflows` into your n8n instance.
2. Create a Google Sheet with tabs: `Leads`, `Properties`, `Visits`, `Bookings`, `Feedback`, `Dashboard` (sample data provided in `/docs/demo-data`).
3. Set up credentials in n8n for: Google Sheets, Gmail, Google Calendar, Google Drive/Docs, Google Maps (Directions only), and OpenAI. No credential is needed for the Nominatim geocoding call.
4. Create the linked Google Forms (Lead Capture, Confirm Interest, Booking Intent, Feedback) and attach the Apps Script webhook bridges documented in `/docs/apps-script`.
5. Activate all five workflows in n8n, making sure each Webhook node's **Production URL** (not Test URL) is what the corresponding Apps Script points to.
6. Submit a test lead through the Lead Capture form to trigger the full chain end-to-end.

---

## 7. Repository Structure

```
ai-real-estate-automation-n8n/
├── README.md
├── workflows/
│   ├── WF1_Lead_Collection.json
│   ├── WF2_Qualification_Recommendation.json
│   ├── WF3_Visit_Scheduling.json
│   ├── WF4_Document_Booking.json
│   └── WF5_Feedback_Dashboard.json
├── docs/
│   ├── architecture-diagram.png
│   ├── event-flow-diagram.png
│   ├── demo-data/ (sample CSVs for each sheet)
│   └── workflow-docs/ (one-page doc per workflow)
├── screenshots/
└── demo-video-link.txt
```

> A sixth workflow (global error handling — Error Trigger → audit log → alert) is a natural next addition; see Future Improvements.

---

## 8. Future Improvements
- A dedicated global error-handling workflow (Error Trigger → audit log sheet → Slack/Gmail alert) wired into every workflow's Settings → Error Workflow
- WhatsApp chatbot as an additional lead-capture channel
- Real e-signature integration (e.g., DocuSign) for booking agreements
- Live Looker Studio dashboard instead of a static sheet
- Predictive analytics on property demand by area
- No-show detection for scheduled visits with automatic agent follow-up flags

---

## 9. Author
Built as part of a university project on business process automation using n8n.

