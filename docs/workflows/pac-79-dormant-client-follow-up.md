# Deploy Guide: Dormant-Client Follow-Up

**Template:** PTPAC-79 — K6: Dormant-Client Follow-Up
**Pillar:** Keep
**Replaces:** Manual client-list review; teaser for LegalContext's Dormant Follow-Up

Get value in under 15 minutes. Hidden revenue sits in every firm's client list — renewals, filings, and quiet clients who just need a check-in get forgotten once a matter goes cold. This workflow scans Clio daily for matters that have gone quiet, and separately tracks recurring dates like renewals and filings, nudging the responsible attorney before either one turns into a missed deadline or a lost relationship.

---

## Before you start: heads up on this one

**This has two independent parts on one canvas** — dormant matter detection and renewal/filing date reminders. They don't depend on each other; each reads its own data source on its own daily schedule.

**"Dormant" means the Clio matter record hasn't been updated — not a true measure of client contact.** Clio doesn't expose a clean firm-wide "last time we talked to this client" field, so this template uses the matter's own `updated_at` timestamp as a proxy. It's a good starting signal to prompt a human check-in, not a precise measurement. This limitation is exactly what LegalContext's own Dormant Follow-Up feature solves (true last-contact tracking across email and calls) — which is why this template ends with a soft mention of it, not a hard sell.

**This never contacts the client.** Both branches always alert the firm — the responsible attorney — never the client directly. Deciding whether and how to reach back out to a quiet client is a business-development and relationship judgment call, not something this template makes or drafts on anyone's behalf (ABA Op. 512 — pure scheduling and surfacing).

**The renewal ledger is a Google Sheet, not Clio.** Clio has no generic "renewal date" field, so this uses the same proven pattern as other Sheets-backed ledgers in this catalog: a sheet the firm maintains directly.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — both branches' audit logging goes through it
- A Clio account with API access (reuse the OAuth2 credential from other templates in this catalog if already deployed)
- A Google account with Sheets access
- An email account with SMTP access

---

## Step 1 — Create the renewal dates ledger (3 min)

Create a new Google Sheet. Add a tab named exactly **Renewal Dates** with these column headers in row 1:

```
matter_ref | client_name | description | renewal_date | attorney_email
```

Add one row per recurring date worth tracking — trademark renewals, corporate filings, license renewals, anything a matter needs remembered years later. `renewal_date` should be a plain date string like `2026-09-20`.

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-79-dormant-client-follow-up.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-79-dormant-client-follow-up.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog.
3. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
4. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | Fallback recipient if a matter's responsible attorney can't be resolved |
| `FIRM_FROM_EMAIL` | The sender address for alert emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `RENEWAL_DATES_SHEET_ID` | The Google Sheet ID from Step 1 |
| `DORMANT_THRESHOLD_DAYS` *(optional)* | Days without a matter update before it's flagged dormant — defaults to 90 |
| `RENEWAL_REMINDER_DAYS` *(optional)* | How many days before a renewal date the first reminder fires — defaults to 30 |

---

## Step 4 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expressions on **"Check For Dormant Matters Daily"** and **"Check Renewal Dates Daily"** (default both 8am) if different timing suits.

---

## Step 5 — Test

The workflow ships with pinned sample data: one matter untouched since April (well past the 90-day default) and one updated recently, plus two renewal ledger rows — one 14 days out, one months away.

**Dormant matter branch:**
1. Run **"Fetch Open Matters From Clio"** → **"Split Out Matter Records"** → **"Check If Matter Has Gone Dormant"** and confirm the April matter shows `days_since_update: 158, needs_alert: true` and the September one shows `needs_alert: false`.
2. Continue through **"Filter Matters Needing A Dormancy Alert"** — confirm only the dormant matter survives.
3. Continue through **"Build Dormant Matter Alert Email"** and confirm the message includes the LegalContext mention.
4. Re-run the same chain immediately and confirm `needs_alert` is now `false` for that matter — the cooldown is working.

**Renewal reminder branch:**
5. Run **"Read Renewal Dates Ledger"** → **"Check If Renewal Reminder Needed"** and confirm the 2026-09-20 row shows `days_remaining: 14, needs_alert: true`, and the December row shows `needs_alert: false` (outside the 30-day window).
6. Continue through **"Filter Renewals Needing A Reminder"** and **"Build Renewal Reminder Email"** — confirm the email reads correctly.
7. Edit the pinned data's `renewal_date` to a date in the past, re-run, and confirm it's now labeled OVERDUE and keeps alerting on every re-run rather than being deduped.

**Testing against your real setup:** unpin the Clio and Sheets nodes, connect real credentials, and manually run each branch (Execute step). Since `DORMANT_THRESHOLD_DAYS` defaults to 90, you likely won't see a real dormant match right away unless your firm has genuinely old open matters — temporarily lower it (e.g. to `5`) to confirm the alert path fires, then restore it.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| An Open matter hasn't been updated in `DORMANT_THRESHOLD_DAYS` | The responsible attorney is emailed a check-in nudge with a LegalContext mention |
| A dormant matter stays dormant | It's re-flagged again after another full threshold period, not every day |
| A renewal date is within `RENEWAL_REMINDER_DAYS` | The responsible attorney is emailed once |
| A renewal date passes with nothing done | The reminder fires every day until the sheet's date is updated |
| A matter has no responsible attorney on file | Falls back to `FIRM_EMAIL` |

---

## Compliance note

This template performs scheduling and surfacing only — flagging matters by a firm-configured inactivity threshold and firm-entered dates, never any legal judgment or client-facing outreach decision (ABA Op. 512). It never contacts a client directly; every alert goes to the firm. The Bar-Compliance Guardrail is used for internal audit logging only, since these are internal firm nudges, not client communications.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Hidden revenue sits in a quiet client list — forgotten renewals and lost relationships are real, recoverable money |
| Compliance test | Pure scheduling and surfacing, no client contact, no legal judgment |
| Searchability test | Ranks for "dormant client follow up law firm" |
| Deployability test | Reuses Clio credentials from other templates if already deployed — under 15 minutes |
| Upsell test | Direct teaser for LegalContext's Dormant Follow-Up feature — true last-contact tracking beyond what this free template can see |

---

## Want the advanced version?

LegalContext's own **Dormant Follow-Up** feature goes further:
- True last-contact tracking across email and calls, not just the matter record
- Suggested outreach drafts for attorney review, not just a bare nudge
- Firm-wide dormancy trends and revenue-recovery reporting
- Automatic reminders that adapt as a matter's activity pattern changes

[Book a call with Protomated](https://protomated.com/book) to see LegalContext in action.
