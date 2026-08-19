# Deploy Guide: Monthly Pre-Bill Assembler

**Template:** PTPAC-61 — B5: Monthly Pre-Bill Assembler
**Pillar:** Keep
**Replaces:** Manual billing-cycle herding

Get value in under 15 minutes. Once a month, the workflow pulls every draft pre-bill out of Clio, groups them by responsible attorney, and emails each attorney their own review list with a link straight into Clio. A separate daily check — active only on specific days near your billing deadline, not every day — re-checks who's still behind and nudges them, escalating straight to the managing partner once the deadline is reached. Nobody has to chase anybody down by hand.

---

## Before you start: heads up on this one

**This never touches, edits, or finalizes a bill.** It only reads draft pre-bills from Clio and sends email reminders — every actual review, edit, and finalize action still happens inside Clio, by the attorney, exactly as it does today.

**This has two independent schedules in one workflow file.** The compile-and-route half runs once a month; the nudge half runs a daily check that only takes action on specific days. Both are visible on the same canvas but don't depend on each other.

**"Draft" as the Clio bill status value hasn't been independently confirmed live** — it follows the same `status=` query parameter that the Invoice Reminder Ladder (PAC-18) already confirmed works against `bills.json`, just with a different value. Check your first real compile run's results against what you see in Clio's own pre-bill list, and adjust the value in both `Fetch Draft Pre-Bills From Clio` and `Fetch Outstanding Draft Pre-Bills` if it doesn't match.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** This template calls it purely for audit logging — every message here is internal (attorney or partner), never a client, so opt-out checking doesn't apply, but every send is still recorded.
- An n8n instance (self-hosted or n8n Cloud)
- A Clio account with API access (reuse the OAuth2 credential from PAC-18/20/55/57/58/59 if already deployed)
- Every matter in Clio should have a responsible attorney assigned with a valid email — matters without one are silently excluded, since there's nobody to send the review to
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-61-monthly-pre-bill-assembler.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-61-monthly-pre-bill-assembler.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if PAC-18, PAC-20, PAC-55, PAC-57, PAC-58, or PAC-59 is already deployed. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide.
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_BILLING_DEADLINE_DAY` | The day of the month pre-bills must be finalized by — pick **28 or lower** (days above 28 don't exist in every month and will roll into the next month unpredictably) |
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | The sender address for all emails this workflow sends |
| `FIRM_PARTNER_EMAIL` | The managing partner's email for laggard escalations — reuses the same variable PAC-18 already uses, so no new configuration if that template is deployed |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_CLIO_BILLING_URL` *(optional)* | A direct link into Clio's draft-bills view, included in every email — falls back to a generic `https://app.clio.com/bills?status=draft` URL if unset |

---

## Step 3 — Set the compile day (2 min)

Open **"Monthly Pre-Bill Compile Trigger"** and confirm the cron expression fires a few days *before* your `FIRM_BILLING_DEADLINE_DAY` — the default is the 25th at 8am. Attorneys need time to actually review before the nudge phase starts checking in.

---

## Step 4 — Activate

Toggle the workflow to **Active**. Both schedules start running on their own — no webhook URL to wire anywhere. The compile trigger fires once a month; the nudge trigger fires daily but only takes action on the specific offsets configured in `Compute Nudge Tier For Today` (3 days before the deadline, 1 day before, on the deadline, and two follow-ups after).

---

## Step 5 — Test

The workflow ships with pinned sample Clio data (two draft pre-bills, two different attorneys) on both fetch nodes, so you can test without waiting for a real schedule fire or having real draft bills in Clio yet.

**Compile & route path:**
1. Run **"Fetch Draft Pre-Bills From Clio"** → **"Group Pre-Bills By Attorney"** and confirm you get one group per attorney, each with their own bill list.
2. Continue through to **"Send Pre-Bill Review Email To Attorney"** and confirm the email lists every bill for that attorney with the right matter, client, and dollar amount.

**Nudge path (test each tier manually since it's date-gated):**
1. Run **"Compute Nudge Tier For Today"** as-is — on most days it'll return `is_nudge_day: false`, which is correct and means nothing else runs today.
2. To test the reminder/urgent/overdue branches without waiting for the actual calendar day, temporarily edit `FIRM_BILLING_DEADLINE_DAY` to a day 3 days from today, run it, confirm `tier: "reminder"`, then try 1 day from today for `urgent`, then today's actual day-of-month for `overdue`. Revert the variable afterward.
3. With `tier: "overdue"` forced, confirm the flow routes to **"Build Partner Escalation For Laggard"** instead of the attorney nudge — check `FIRM_PARTNER_EMAIL` receives it, not the attorney.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Compile day, at least one draft pre-bill exists | Grouped by attorney, one review email per attorney |
| Compile day, nothing in draft | Skipped — no emails, nothing to route |
| A matter has no responsible attorney assigned | That bill is counted in `unassigned_count` but excluded from routing — check execution history periodically |
| Nudge day, but every pre-bill was already finalized | Skipped — the good outcome, not a failure |
| Nudge day (3 or 1 days before deadline), attorney still has drafts | Attorney gets a direct reminder |
| Nudge day (on or after deadline), attorney still has drafts | Managing partner gets the escalation instead of the attorney |
| Any day that isn't a configured nudge offset | No API call made at all — the workflow does nothing until the next relevant day |

---

## Compliance note

This template performs scheduling and internal notification only — it never reads, summarizes, or comments on the substance of any bill, and it never contacts a client (ABA Op. 512). Every routing email and every escalation is logged to the Bar-Compliance Guardrail's audit sheet for record-keeping, though opt-out checking doesn't apply since nothing here reaches a client.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces the manual chasing that stalls the monthly billing cycle — a stalled cycle directly delays revenue |
| Compliance test | Scheduling and internal notification only, no legal work, no client contact |
| Searchability test | Ranks for "law firm pre-bill automation" |
| Deployability test | Reuses Clio and SMTP credentials from earlier templates if already deployed — under 15 minutes |
| Upsell test | Quick-Win Build adds Slack/Teams nudges, a firm-wide billing-cycle dashboard, and automatic finalize-and-send once an attorney approves |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Slack or Microsoft Teams nudges alongside email, for firms where attorneys live in chat instead of their inbox
- A firm-wide billing-cycle dashboard showing every attorney's review status at a glance
- One-click finalize-and-send once an attorney approves their pre-bills, no separate trip into Clio required
- Custom nudge cadences per attorney based on their historical response time

[Book a call with Protomated](https://protomated.com/book) to get started.
