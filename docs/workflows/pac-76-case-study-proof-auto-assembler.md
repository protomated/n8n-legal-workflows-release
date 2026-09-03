# Deploy Guide: Case-Study Proof Auto-Assembler

**Template:** PTPAC-76 — G8: Case-Study Proof Auto-Assembler
**Audience:** Internal Protomated tool — **not** a law firm template
**Replaces:** A freelance case-study copywriter, weeks of manual data-pulling

---

## This one is different from everything else in this catalog

Every other workflow in this repo is a lead-magnet template a law firm deploys on their own n8n instance. This one isn't — it runs on **Protomated's own Salesmate CRM**, references an internal reviewer (Dele) by name, and exists to solve a purely internal problem: the single highest-leverage conversion unlock in the GTM Roadmap (a published case study) has no path to production today, because assembling one means a partner blocking time for interviews, data-pulling, and copywriting weeks after a Build closes — so it never gets prioritized.

It's still built to this catalog's usual bar (harness-tested code, pinned test data, sticky-note documentation, `active: false` until configured) because that discipline is worth keeping regardless of audience. But nothing about its naming, framing, or setup steps is written for a law firm — it's written for whoever on the Protomated team maintains it.

---

## Before you start: the most important thing to understand

**This workflow never publishes anything.** It only ever produces a DRAFT and routes it for two separate human approvals — Dele's internal review, and the client's own sign-off plus a quote. Nothing goes out the door without both. There's no code path here that posts, emails externally as final copy, or marks anything "published" — that last step is always a manual edit to the ledger sheet once both approvals are in hand.

**This has two independent parts on one canvas:**
1. **Draft assembly** — fires when a Build/Deep Assessment deal moves to Closed Won in Salesmate. Pulls before/after metrics from a manually-maintained results tracker, drafts a one-pager, logs it, and emails Dele plus the client contact.
2. **Stale review reminder** — runs daily, independently, and nudges Dele about any draft that's sat in "pending review" too long.

**The before/after metrics are never pulled automatically from a client's own systems.** Protomated doesn't have blanket API access into every client's Clio instance, so someone on the team enters response-time, no-show-rate, and hours-saved numbers into a Google Sheet during the engagement. If that row doesn't exist yet when a deal closes, this workflow tells Dele instead of drafting anything — it never guesses or fabricates a number.

**Salesmate's exact webhook and API response shapes here are a best-effort reconstruction**, built from Salesmate's documented v4 API and the payload conventions already used in Protomated's other Salesmate automations (see `workflows.sample/`). Confirm field names against a real webhook delivery on first live test — see the troubleshooting note at the end.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — the client-facing sign-off request goes through it
- Admin access to Protomated's own Salesmate account, to add an automation rule
- A Salesmate API token
- A Google account with Sheets access
- An email account with SMTP access

---

## Step 1 — Create the two tracking sheets (5 min)

Create a new Google Sheet (or two) with these tabs:

**Results Tracker** tab — filled in by hand during the engagement, one row per deal:
```
deal_id | client_name | response_time_before_hours | response_time_after_hours | no_show_rate_before_pct | no_show_rate_after_pct | hours_saved_per_week
```

**Case Studies** tab — this workflow's own review ledger, written automatically:
```
deal_id | client_name | headline_stat | draft_summary | status | created_at | reviewed_at | client_signoff_at
```

Copy each Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-76-case-study-proof-auto-assembler.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-76-case-study-proof-auto-assembler.json). Do not activate it yet.
2. Salesmate credential: **Credentials → New → Header Auth** → header name `accessToken`, value = your Salesmate API token.
3. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
4. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `DELE_EMAIL` | Where review requests and stale-review reminders go |
| `FIRM_FROM_EMAIL` | Sender address for all emails this template sends |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `RESULTS_TRACKER_SHEET_ID` | The Results Tracker Sheet's ID from Step 1 |
| `CASE_STUDY_LEDGER_SHEET_ID` | The Case Studies Sheet's ID from Step 1 |
| `STALE_REVIEW_DAYS` *(optional)* | How many days a draft can sit before Dele gets nudged — defaults to 5 |

---

## Step 4 — Wire the Salesmate trigger and activate (5 min)

1. Toggle the workflow to **Active**.
2. Open **"When Deal Moves To Closed Won"**, copy its **Production URL**.
3. In Salesmate, create a Workflow Automation: trigger on **Deal stage changed**, condition **stage = Closed Won**, action **Call Webhook** → paste the URL from step 2. Scope the rule to the Build and Deep Assessment pipelines if Salesmate's automation builder allows it (this workflow re-checks the pipeline anyway, so it's not strictly required).
4. Adjust the cron expression on **"Check For Stale Case Study Reviews Daily"** (default 9am) if a different time suits.

---

## Step 5 — Test

The workflow ships with pinned sample data: a Closed Won "Build" deal for Smith & Associates (metrics already in the tracker) and a second deal, Roe Family Law, sitting in the Case Studies ledger for 14 days (well past the 5-day default).

**Draft assembly branch:**
1. Run **"When Deal Moves To Closed Won"** → **"Parse Closed Won Deal Webhook"** and confirm `is_valid_case_study_deal: true`.
2. Edit the pinned webhook data's `pipeline` to `"Retainer"`, re-run, confirm it now routes to **"Respond Not A Case Study Deal"**. Restore it to `"Build"` afterward.
3. Continue through **"Fetch Full Deal Details From Salesmate"** → **"Fetch Primary Contact From Salesmate"** → **"Read Client Results Tracker"** → **"Find Matching Results Entry"** and confirm `is_found: true` with all five metrics populated for deal `9001`.
4. Continue through **"Compute Case Study Headline Stat"** and confirm the headline is `"88.9% faster response time"` (the largest improvement of the three metrics, computed from `18h → 2h`).
5. Continue through **"Draft Case Study One-Pager"** → **"Log Draft Case Study For Review"** and confirm the row would append with `status: "pending_review"`.
6. Continue through to **"Send Review Request To Dele"** and **"Send Client Sign-Off Request Email"** and confirm both emails read correctly.
7. Temporarily clear the pinned data on **"Read Client Results Tracker"** to an empty array, re-run from **"Find Matching Results Entry"**, and confirm it routes to **"Notify Dele Metrics Not Yet Logged"** instead of drafting anything.

**Stale review reminder branch:**
8. Run **"Read Case Study Review Ledger"** → **"Filter Entries Still Pending Review"** and confirm only the Roe Family Law row survives (the Major Estate Planning row is already `published`).
9. Continue through **"Compute Days Pending And Check If Reminder Needed"** and confirm `days_pending: 14, needs_reminder: true`.
10. Continue through **"Build Stale Review Reminder Email"** and confirm it names Roe Family Law and 14 days.

**Testing against your real setup:** unpin all Salesmate/Sheets nodes, connect real credentials, and manually run the draft branch (Execute step) after moving a real test deal to Closed Won in Salesmate — or by manually POSTing a test payload to the webhook URL if you'd rather not touch a real deal. Confirm the actual Salesmate API response shapes match what **"Find Matching Results Entry"** expects (`data.title`, `data.primaryContact`, `data.firstName`/`lastName`/`email`) and adjust if Salesmate's real response differs from this reconstruction.

**If a real Salesmate response doesn't match:** open the raw output of "Fetch Full Deal Details From Salesmate" or "Fetch Primary Contact From Salesmate" in n8n's execution view and compare field names directly — Salesmate's actual v4 responses may nest fields differently than documented. Adjust "Find Matching Results Entry" to match once confirmed, the same way every other integration in this catalog gets corrected against real execution data.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A Build/Deep Assessment deal moves to Closed Won | The workflow checks for a Results Tracker entry and drafts a case study if one exists |
| No Results Tracker entry exists yet for that deal | Dele is emailed that metrics are missing; nothing is drafted |
| A draft is assembled | Logged to the Case Studies sheet as `pending_review`; Dele and the client contact are both emailed |
| The client contact has opted out of firm emails | The draft and Dele's review request still happen; only the client-facing sign-off request is suppressed |
| A draft sits in `pending_review` past `STALE_REVIEW_DAYS` | Dele gets a daily reminder until the sheet row is updated |
| A case study is approved and published | That's always a manual edit to the sheet — this workflow has no "publish" action |

---

## Compliance note

This is Protomated's own internal marketing operations tool — it never touches legal work, client matters, or any law firm's systems, so ABA Op. 512 doesn't apply to it directly. It only aggregates operational metrics (response time, no-show rate, hours saved) that a client has already agreed can be shared, and nothing publishes without that client's own explicit sign-off and quote. The Bar-Compliance Guardrail is used for the client-facing sign-off request (opt-out respected, disclaimer applied, send logged) and for internal audit logging on Dele's review request.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces a freelance case-study copywriter and weeks of manual data-pulling; directly serves the GTM Roadmap's Q3 milestone of publishing the first named case study |
| Compliance test | Internal tool, no legal work touched; client sign-off required before anything publishes |
| Searchability test | N/A — internal tool, not a lead-magnet template |
| Deployability test | One Salesmate token, two Sheets, and SMTP — under 15 minutes for whoever maintains Protomated's own n8n instance |
| Upsell test | N/A — internal tool |
