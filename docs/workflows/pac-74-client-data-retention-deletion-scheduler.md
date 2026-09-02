# Deploy Guide: Client Data Retention & Deletion Scheduler

**Template:** PTPAC-74 — OPS9: Client Data Retention & Deletion Scheduler
**Pillar:** Ops
**Replaces:** Manual retention tracking, legal-hold spreadsheets

Get value in under 15 minutes. When a matter closes in Clio, this workflow applies your firm's own retention policy, schedules a review date, and escalates reminders to the responsible attorney as that date approaches — with a full audit trail of what was kept, extended, or confirmed for deletion, and why. It never deletes anything itself, ever.

---

## Before you start: the most important thing to understand

**This workflow cannot delete client data, under any circumstance — there is no code path in it that does.** "Confirm Deletion" is a label on an email link that records an attorney's authorization in a spreadsheet. Actually removing the file from Clio and any document management system is always a separate, manual step the firm carries out afterward. This is deliberate, not a limitation: automating the actual destruction of client records is a decision this template will never make for you (ABA Op. 512 — pure scheduling and record-keeping, never a legal or destructive determination).

**This has three independent parts on one canvas:**
1. **Daily scheduling** — finds every Closed matter in Clio, applies your retention policy, and logs one ledger row per matter (only ever once, even if it runs every day for years).
2. **Daily escalation** — re-reads the ledger and reminds the responsible attorney at 90/30/7 days before the retention date, then every single day once it's overdue and still undecided — same "never go silent about something this important" philosophy as PAC-67's Statute of Limitations Watchdog.
3. **Decision webhook** — records whichever of the three action links (Confirm Deletion / Extend Retention / Place Legal Hold) the attorney clicked, and stops all future reminders for that matter accordingly.

**The retention ledger is a Google Sheet, not Clio** — same reasoning as PAC-68's referral ledger: retention tracking needs a persistent, human-reviewable record the firm can open directly, and doesn't map onto a Clio field already confirmed reliable in this catalog.

**Your firm's retention policy lives in one code node, not a config file.** Open `Check If Retention Needs Scheduling` and edit the `RETENTION_POLICY_YEARS` map to match your actual policy by practice area. This is the one thing every firm must customize before going live — the defaults in this template are placeholder examples, not a real policy recommendation.

**Confirmed live:** `close_date` and `practice_area{id,name}` both read correctly from `Fetch Closed Matters From Clio` — though on real test matters with no practice area or responsible attorney assigned in Clio, those fields correctly come back blank and the template falls back to `DEFAULT_RETENTION_YEARS` and `FIRM_EMAIL` respectively, exactly as designed. Google Sheets also confirmed to return date columns back as the exact plain-text string they were written as (no reformatting).

**Confirmed live, the hard way: every column header in the Retention sheet must be spelled exactly right, or that field silently has nowhere to go.** A sheet missing the `status`/`last_action`/`last_action_at` headers caused every ledger row to come back with an empty `status` on read — which made `Filter Ledger Entries Still Scheduled` in Branch B drop every single row and never send a reminder, with no error anywhere to point at the cause. Copy the column list in Step 1 exactly; don't rely on whatever a spreadsheet UI suggests auto-adding.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — reminder logging goes through it
- A Clio account with API access, matter and user-directory read scope (reuse the OAuth2 credential from other templates in this catalog if already deployed)
- A Google account with Sheets access
- An email account with SMTP access

---

## Step 1 — Create the retention ledger (3 min)

Create a new Google Sheet. Add a tab named exactly **Retention** with these column headers in row 1:

```
matter_id | matter_ref | matter_description | practice_area | client_name | close_date | retention_years_applied | retention_deadline | attorney_name | attorney_email | status | last_action | last_action_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-74-client-data-retention-deletion-scheduler.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-74-client-data-retention-deletion-scheduler.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog.
3. Google Sheets credential: **Credentials → New → Google Sheets OAuth2** → sign in with the account that owns the ledger sheet.
4. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 3 — Set your firm's actual retention policy (5 min)

Open **"Check If Retention Needs Scheduling"** and edit `RETENTION_POLICY_YEARS`:

```js
const RETENTION_POLICY_YEARS = {
  'Personal Injury': 7,
  'Family Law': 7,
  'Estate Planning': 10,
  // add every practice area your firm handles, with its actual retention period in years
};
```

Any practice area not listed uses `DEFAULT_RETENTION_YEARS` (an n8n Variable, defaulting to 7 if unset).

---

## Step 4 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | Fallback recipient if a matter's responsible attorney can't be resolved |
| `FIRM_FROM_EMAIL` | The sender address for reminder emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `RETENTION_LEDGER_SHEET_ID` | The Google Sheet ID from Step 1 |
| `RETENTION_DECISION_WEBHOOK_URL` | This workflow's own Production webhook URL — you'll copy this in Step 5, below, right after you activate |
| `DEFAULT_RETENTION_YEARS` *(optional)* | Fallback retention period in years for any practice area not listed in Step 3 — defaults to 7 |
| `RETENTION_EXTENSION_YEARS` *(optional)* | How many years "Extend Retention" pushes the deadline out by — defaults to 1 |

---

## Step 5 — Activate and wire the decision webhook (3 min)

1. Toggle the workflow to **Active**.
2. Open **"When Retention Decision Is Made"**, copy its **Production URL**, and set it as `RETENTION_DECISION_WEBHOOK_URL` in Settings → Variables (Step 4).
3. Adjust the cron expressions on **"Check For Newly Closed Matters Daily"** (default 6am) and **"Send Retention Deadline Reminders Daily"** (default 7am) if different times suit your firm.

---

## Step 6 — Test

The workflow ships with pinned sample data across all Clio- and Sheets-facing nodes.

**Scheduling branch:**
1. Run **"Fetch Closed Matters From Clio"** → **"Split Out Closed Matter Records"** → **"Filter Closed Matters With Close Date"** and confirm all 3 pinned closed matters survive.
2. Continue through **"Check If Retention Needs Scheduling"** and confirm each computes the right retention deadline (close date + the matching practice area's years, or the default for the unmapped "General Consultation" matter).
3. Re-run the same node a second time and confirm `needs_scheduling` is now `false` for all three — already scheduled.

**Reminder branch:**
4. Run **"Read Retention Ledger"** → **"Filter Ledger Entries Still Scheduled"** and confirm the `legal_hold` pinned row is dropped, leaving 2.
5. Continue through **"Compute Retention Escalation Tier"** and confirm both remaining rows show a tier and `needs_alert: true`.
6. Continue through **"Build Retention Reminder Email"** and confirm the email includes all three action links, each carrying the correct `matter_id`.

**Decision branch:**
7. Run **"When Retention Decision Is Made"** (pinned as `action=extend` for matter `41010`) → **"Parse Retention Decision Request"** and confirm it parses correctly.
8. Continue through **"Read Retention Ledger For Decision"** → **"Find Matching Row And Compute Update"** and confirm the retention deadline extends by 1 year and `action_label` reads correctly.
9. Edit the pinned webhook data's `action` to `confirm`, then to `hold`, re-running each time, and confirm the correct status and message for each.
10. Edit the pinned webhook data's `matter_id` to something not in the ledger (e.g. `99999`) and confirm it routes to **"Respond Matter Not Found Page"**.

**Testing against your real setup:** unpin every Clio- and Sheets-facing node, connect real credentials, close a real test matter in Clio, and manually run the scheduling branch (Execute step from `Check For Newly Closed Matters Daily` — don't wait for 6am). A real retention deadline will land years in the future, so to test the reminder branch without waiting: open the Sheet, edit that row's `retention_deadline` cell to a few days from today, and set `attorney_email` to an inbox you can check — then manually run the reminder branch (Execute step from `Send Retention Deadline Reminders Daily`) and click one of the three links in the actual delivered email to confirm the full round trip.

**If a real run produces rows with an empty `status` and Branch B never alerts on anything:** double-check every column header in your Sheet against the exact list in Step 1 — a single missing or misspelled header (most commonly `status`) silently drops that field with no error, which is exactly what happened during this template's own live testing.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A matter closes in Clio | Logged to the ledger once, with a retention date computed from your policy |
| A matter is already logged | Never logged again, even if the daily scan keeps finding it |
| A ledger entry reaches 90, 30, or 7 days before its retention date | An escalation email fires once for that tier |
| A ledger entry is overdue and still unresolved | The reminder fires on **every single daily run** until a decision is made |
| The attorney clicks "Confirm Deletion" | Ledger status becomes `confirmed_for_deletion`; reminders stop; **nothing is actually deleted by this workflow** |
| The attorney clicks "Extend Retention" | The retention date moves out by `RETENTION_EXTENSION_YEARS`; reminders continue on the new date |
| The attorney clicks "Place Legal Hold" | Ledger status becomes `legal_hold`; reminders stop indefinitely |
| A decision link is malformed, expired, or references a matter no longer in the ledger | A plain error page is shown; nothing in the ledger changes |

---

## Compliance note

This template performs scheduling, computation, and record-keeping only — applying a firm-configured retention period to a firm-confirmed close date, never any legal judgment about what must be retained or destroyed (ABA Op. 512). It never deletes client data or files under any circumstance; every retention decision requires the responsible attorney's explicit click-through, and actually carrying out a deletion remains entirely manual, outside this workflow. The Bar-Compliance Guardrail is used for audit logging on reminders only.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Indefinite data retention is a compliance and breach-exposure liability; this replaces manual tracking and legal-hold spreadsheets |
| Compliance test | Pure scheduling and record-keeping against a firm-configured policy, no automated deletion, ever |
| Searchability test | Ranks for "law firm client data retention policy" |
| Deployability test | Reuses Clio and SMTP credentials from earlier templates if already deployed — under 15 minutes |
| Upsell test | Clear Done-for-You path: managed retention policy configuration, automatic execution coordination with IT/DMS vendors, and a firm-wide retention compliance dashboard |

---

## Want the advanced version?

Protomated builds the done-for-you **Done-for-You** implementation that adds:
- Retention policy configuration handled for you, matched to your jurisdiction and practice areas
- Coordination with your IT provider or document management system so confirmed deletions actually get carried out on schedule
- A firm-wide dashboard showing every matter's retention status at a glance
- State-specific retention rule guidance built into the policy setup

[Book a call with Protomated](https://protomated.com/book) to get started.
