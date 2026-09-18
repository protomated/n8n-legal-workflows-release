# The Immigration Firm Starter Pack

Five free automations for the parts of immigration practice that generate the most calls and the most paperwork — new intake, filing deadlines, renewal reminders, case-status questions, and form population.

Immigration clients call more than clients in almost any other practice area, because USCIS timelines are opaque and the stakes are high. This free bundle gets new intake into your practice management system without retyping, tracks every RFE and appeal deadline automatically, flags your team when a client's status is coming up for renewal, answers routine status questions without an attorney interruption, and fills your firm's own forms from intake data.

## What's included

1. **Intake Form to PM Contact** (`clio-intake-sync.json`) — The moment a prospective client submits your website intake form, this writes a clean Contact and Matter into your practice management system automatically — no staff member has to retype it.
2. **Filing & Response Deadline Tracker** (`court-deadline-calculator.json`) — Calculates every downstream deadline — RFE responses, appeal windows, filing dates — from the dates your attorneys define, and drops each one into your calendar and task list.
3. **Renewal Alerts** (`dormant-client-follow-up.json`) — Flags past clients to your team when their status is coming up for renewal, so someone reaches out well before a lapse becomes urgent.
4. **Matter Status Bot** (`matter-status-bot.json`) — Answers routine "what's the status of my case" questions by text or email, straight from your practice management system's own data — no attorney interruption needed.
5. **Form-Fill from Intake Data** (`form-fill-from-intake-data.json`) — Fills your firm's own form templates with the client's intake details — no more retyping the same names, dates, and details into multiple documents.

Together, these five workflows replace a patchwork of manual data entry, spreadsheet-based deadline tracking, and constant phone-based status updates — recovering hours of staff time every week in a practice area where client anxiety and call volume run especially high.

## Getting started

1. Each item above is a separate, independent n8n workflow — install one, some, or all five, in any order.
2. Open your n8n canvas and paste (`Ctrl+V` / `Cmd+V`) the contents of the `.json` file for the automation you want to set up.
3. Follow that automation's setup guide in the `deploy-guides/` folder (same name, `.md` instead of `.json`) — it walks through credentials, required settings, and how to test it safely before going live.

Every workflow ships turned off, with sample test data already loaded, so you can test it end-to-end before connecting it to a real client.

## Compliance

Every automation in this pack is operational only — data entry, deadline calculation from firm-entered rules, reminders, status reporting, and field population. None of them give immigration advice, predict case outcomes, or exercise legal judgment on your behalf (per ABA Opinion 512).

## Want more?

Want these tuned to your firm's specific case types, USCIS form set, and renewal cadence? [Book a call with Protomated](https://protomated.com/book).
