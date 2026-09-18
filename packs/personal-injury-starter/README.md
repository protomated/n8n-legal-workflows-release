# The Personal Injury Firm Starter Pack

Four free automations built for how PI firms actually lose leads and revenue — missed calls, unqualified intake eating attorney time, untracked billable minutes, and settlements that end without a review request.

Personal injury intake moves fast, and every missed call is a call to the next firm. This free bundle catches the calls you miss, filters serious inquiries from tire-kickers before an attorney ever sees them, recovers the billable time that slips through informal client check-ins, and asks for a review the moment a case closes well.

## What's included

1. **Missed-Call Instant Text-Back** (`missed-call-text-back.json`) — When someone calls your firm and you miss it, they instantly receive a text and you get an email alert — before they dial the next lawyer.
2. **AI Intake Qualifier** (`ai-intake-qualifier.json`) — Scores every new inquiry against your firm's own practice areas and urgency signals, replies appropriately, and immediately alerts an attorney for the highest-fit leads.
3. **Unbilled-Time Catcher** (`unbilled-time-catcher.json`) — Cross-references your calendar against logged time entries so client calls and case-status check-ins that never made it onto an invoice get caught before the bill goes out.
4. **Review Request Engine** (`review-request-engine.json`) — Automatically requests a review from clients after a matter closes well, with built-in rating-gating so an unhappy client never lands on your public Google profile.

Together, these four workflows replace a patchwork of answering services, intake-screening subscriptions, and manual billing review — recovering both the leads that would have gone to a competitor and the billable hours that quietly disappear in a high-volume PI practice.

## Getting started

1. Each item above is a separate, independent n8n workflow — install one, some, or all four, in any order.
2. Open your n8n canvas and paste (`Ctrl+V` / `Cmd+V`) the contents of the `.json` file for the automation you want to set up.
3. Follow that automation's setup guide in the `deploy-guides/` folder (same name, `.md` instead of `.json`) — it walks through credentials, required settings, and how to test it safely before going live.

Every workflow ships turned off, with sample test data already loaded, so you can test it end-to-end before connecting it to a real client.

## Compliance

Every automation in this pack is operational only — routing, scoring by firm-configured rules, time-entry cross-referencing, and review requests. None of them evaluate case merits, give a legal opinion on a claim, or exercise legal judgment on your behalf (per ABA Opinion 512).

## Want more?

Want deeper AI-based intake scoring, PM systems beyond Clio, or authentication options for sensitive matters? [Book a call with Protomated](https://protomated.com/book).
