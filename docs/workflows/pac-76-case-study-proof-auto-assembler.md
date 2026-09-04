# Deploy Guide: Case-Study Proof Auto-Assembler

**Template:** PTPAC-76 — G8: Case-Study Proof Auto-Assembler
**Audience:** Internal Protomated tool — **not** a law firm template
**Replaces:** A freelance case-study copywriter, weeks of manual data-pulling

---

## This one is different from everything else in this catalog

Every other workflow in this repo is a lead-magnet template a law firm deploys on their own n8n instance. This one isn't — it tracks Protomated's own client deals and references an internal reviewer (Dele) by name. It exists to solve a purely internal problem: the single highest-leverage conversion unlock in the GTM Roadmap (a published case study) has no path to production today, because assembling one means a partner blocking time for interviews, data-pulling, and copywriting weeks after a Build closes — so it never gets prioritized.

It's still built to this catalog's usual bar (harness-tested code, pinned test data, sticky-note documentation, `active: false` until configured) because that discipline is worth keeping regardless of audience.

**No CRM API dependency, on purpose.** An earlier version of this template triggered off a Salesmate webhook. That integration has since gone offline, and rather than re-couple to whichever CRM Protomated happens to be using this quarter, this version triggers off a plain Google Sheet the team already has to touch anyway — someone flips a deal's `stage` to `Closed Won`, and this picks it up on the next daily run. No API token to keep alive, nothing to break when the CRM changes again.

---

## Before you start: the most important thing to understand

**This workflow never publishes anything.** It only ever produces a DRAFT and routes it for two separate human approvals — Dele's internal review, and the client's own sign-off plus a quote. Nothing goes out the door without both. There's no code path here that posts, emails externally as final copy, or marks anything "published" — that last step is always a manual edit to the ledger sheet once both approvals are in hand.

**This has two independent parts on one canvas:**
1. **Draft assembly** — runs daily against a Deals Tracker sheet; whenever the team marks a Build/Deep Assessment deal Closed Won, pulls before/after metrics from a second results tracker, drafts a one-pager, logs it, and emails Dele plus the client contact.
2. **Stale review reminder** — runs daily, independently, and nudges Dele about any draft that's sat in "pending review" too long.

**The before/after metrics are never pulled automatically from a client's own systems.** Protomated doesn't have blanket API access into every client's Clio instance, so someone on the team enters response-time, no-show-rate, and hours-saved numbers into a Google Sheet during the engagement. If that row doesn't exist yet when a deal closes, this workflow tells Dele instead of drafting anything — it never guesses or fabricates a number.

**It correctly handles more than one deal closing on the same day.** The matching step deliberately processes the whole day's newly-closed deals as a batch rather than assuming there's only ever one — each deal gets matched to its own results-tracker row independently, not silently defaulted to whichever deal happened to be first.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — the client-facing sign-off request goes through it
- A Google account with Sheets access
- An email account with SMTP access

---

## Step 1 — Create the three tracking sheets (5 min)

Create a new Google Sheet (or several) with these tabs:

**Deals Tracker** tab — this IS the trigger. Updated by hand whenever a deal's stage changes:
```
deal_id | client_name | pipeline | stage | contact_name | contact_email
```
Flip `stage` to exactly `Closed Won` when a Build or Deep Assessment deal closes — that's what this workflow watches for.

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

## Step 2 — Import the workflow and add credentials (3 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-76-case-study-proof-auto-assembler.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-76-case-study-proof-auto-assembler.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `DELE_EMAIL` | Where review requests and stale-review reminders go |
| `FIRM_FROM_EMAIL` | Sender address for all emails this template sends |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `DEALS_TRACKER_SHEET_ID` | The Deals Tracker Sheet's ID from Step 1 |
| `RESULTS_TRACKER_SHEET_ID` | The Results Tracker Sheet's ID from Step 1 |
| `CASE_STUDY_LEDGER_SHEET_ID` | The Case Studies Sheet's ID from Step 1 |
| `STALE_REVIEW_DAYS` *(optional)* | How many days a draft can sit before Dele gets nudged — defaults to 5 |

---

## Step 4 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expression on **"Check For Newly Closed Deals Daily"** (default 9am) and **"Check For Stale Case Study Reviews Daily"** if a different time suits.

---

## Step 5 — Test

The workflow ships with pinned sample data: a Deals Tracker with two deals already Closed Won (Smith & Associates, Roe Family Law — both with metrics already in the tracker) and one still in Discovery (Major Estate Planning, correctly ignored), plus a Case Studies ledger with Roe Family Law sitting 14 days pending review.

**Draft assembly branch:**
1. Run **"Read Deals Tracker"** → **"Compute Which Deals Just Closed Won"** and confirm `is_new_closed_won_deal: true` for deals `9001` and `8850`, and `false` for `8700` (wrong stage).
2. Continue through **"Filter Newly Closed Deals"** — confirm exactly 2 items survive.
3. Continue through **"Read Client Results Tracker"** → **"Find Matching Results Entry"** and confirm **both** deals resolve `is_found: true` with their **own** metrics — deal `9001` shows `response_time_before_hours: 18`, deal `8850` shows `24`. This is the key thing to verify: each deal must keep its own numbers, not both showing the first deal's data.
4. Continue through **"Compute Case Study Headline Stat"** and confirm deal `9001`'s headline is `"88.9% faster response time"`.
5. Continue through to **"Log Draft Case Study For Review"**, **"Send Review Request To Dele"**, and **"Build Client Sign-Off Request Email"** — confirm both deals produce independent, correctly-addressed emails.
6. Re-run from **"Read Deals Tracker"** a second time and confirm both deals now show `is_new_closed_won_deal: false` — the dedupe is working.
7. Temporarily clear the pinned data on **"Read Client Results Tracker"** to an empty array, re-run from **"Find Matching Results Entry"**, and confirm both deals route to **"Notify Dele Metrics Not Yet Logged"** instead of drafting anything.

**Stale review reminder branch:**
8. Run **"Read Case Study Review Ledger"** → **"Filter Entries Still Pending Review"** — confirm only Roe Family Law survives (Major Estate Planning is already `published`).
9. Continue through **"Compute Days Pending..."** — confirm `days_pending: 14, needs_reminder: true`.

**Testing against your real setup:** unpin all three Sheets nodes, connect real credentials, and flip a real (ideally test) deal's `stage` to `Closed Won` in your actual Deals Tracker, then manually run the daily branch (Execute step — don't wait for the schedule).

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A Deals Tracker row is flipped to `Closed Won` in a case-study pipeline | Checked for a Results Tracker entry and drafted if one exists |
| Multiple deals close the same day | Each is matched to its own metrics and processed independently |
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
| Deployability test | Three Google Sheets and SMTP — under 15 minutes, no CRM API dependency to keep alive |
| Upsell test | N/A — internal tool |
