# Deploy Guide: Monthly Pre-Bill Assembler

**Template:** PTPAC-61 — B5: Monthly Pre-Bill Assembler
**Pillar:** Keep
**Replaces:** Manual billing-cycle herding

Get value in under 15 minutes. Once a month, the workflow pulls every draft pre-bill out of Clio, groups them by responsible attorney, and emails each attorney a styled review list — with a Claude-generated priority tip ("start with the oldest one") and a link straight into Clio. A separate daily check then tracks how long each individual pre-bill has been sitting in draft — using Clio's own timestamp on the bill — and nudges attorneys once it's been outstanding a while, escalating straight to the managing partner if it stays outstanding much longer. Nobody has to chase anybody down by hand.

---

## Before you start: heads up on this one

**This never touches, edits, or finalizes a bill.** It only reads draft pre-bills from Clio and sends email reminders — every actual review, edit, and finalize action still happens inside Clio, by the attorney, exactly as it does today.

**Nudging is driven by each bill's own age, not a shared firm-wide deadline.** Draft pre-bills don't carry a due date in Clio — that only gets set once a bill is finalized and issued to a client. So instead of a single calendar day everyone converges on, this tracks how long each individual pre-bill has personally been sitting untouched (using the bill's own creation timestamp) and nudges based on that. Two bills for the same attorney can be at completely different stages if they were created on different days.

**This has two independent schedules in one workflow file.** The compile-and-route half runs once a month; the nudge half runs a daily check. Both are visible on the same canvas but don't depend on each other.

**The priority tip is operational only, never legal.** Claude looks at nothing but each bill's dollar amount and how long it's been in draft — it's explicitly instructed never to comment on case merits or strategy (ABA Op. 512), and a second, independent keyword scan double-checks its output afterward. If either check fails, or the Anthropic API call itself fails for any reason, the tip is silently omitted — the review email still sends normally with the bill list and Clio link, it just won't have the extra sentence at the top.

**Two Clio field names haven't been independently confirmed live:**
- `status=draft` as the query value on `bills.json` follows the same convention as the Invoice Reminder Ladder's (PAC-18) confirmed-working `status=overdue`, just with a different value.
- `created_at` (used to compute each bill's age) is a standard REST timestamp field name, but hasn't itself been confirmed against a live Clio bill response.

Check your first real compile run's results against what you see in Clio's own pre-bill list, and adjust either field name in the two `Fetch...` HTTP nodes if something doesn't match.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** This template calls it purely for audit logging — every message here is internal (attorney or partner), never a client, so opt-out checking doesn't apply, but every send is still recorded.
- An n8n instance (self-hosted or n8n Cloud)
- A Clio account with API access (reuse the OAuth2 credential from PAC-18/20/55/57/58/59 if already deployed)
- Every matter in Clio should have a responsible attorney assigned with a valid email — matters without one are silently excluded, since there's nobody to send the review to
- An [Anthropic API key](https://console.anthropic.com) — reuse the one from PAC-31/50/60/62 if already deployed. Used only for the review email's priority tip; if the call fails for any reason, the email still sends without that line.
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-61-monthly-pre-bill-assembler.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-61-monthly-pre-bill-assembler.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if PAC-18, PAC-20, PAC-55, PAC-57, PAC-58, or PAC-59 is already deployed. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide.
3. Anthropic credential: reuse your existing `Anthropic API key` credential if PAC-31, PAC-50, PAC-60, or PAC-62 is already deployed. Otherwise: **Credentials → New → Header Auth** → Header Name `x-api-key`, Header Value your key from console.anthropic.com → save as `Anthropic API key`.
4. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | The sender address for all emails this workflow sends |
| `FIRM_PARTNER_EMAIL` | The managing partner's email for laggard escalations — reuses the same variable PAC-18 already uses, so no new configuration if that template is deployed |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_PREBILL_REMINDER_DAYS` *(optional)* | Days after creation before a pre-bill gets a first reminder — defaults to `5` |
| `FIRM_PREBILL_URGENT_DAYS` *(optional)* | Days after creation before the tone escalates — defaults to `10` |
| `FIRM_PREBILL_ESCALATE_DAYS` *(optional)* | Days after creation before the managing partner gets looped in — defaults to `15` |
| `FIRM_PREBILL_ESCALATE_REPEAT_DAYS` *(optional)* | How often the partner escalation repeats after that, until resolved — defaults to `5` |
| `FIRM_CLIO_BILLING_URL` *(optional)* | A direct link into Clio's draft-bills view, included in every email — falls back to a generic `https://app.clio.com/bills?status=draft` URL if unset |

---

## Step 3 — Set the compile day (2 min)

Open **"Monthly Pre-Bill Compile Trigger"** and set the cron expression to whenever your firm actually starts its billing cycle — the default is the 25th of each month at 8am. This date is independent of the nudge phase; it doesn't need to line up with any of the day thresholds from Step 2.

---

## Step 4 — Activate

Toggle the workflow to **Active**. Both schedules start running on their own — no webhook URL to wire anywhere. The compile trigger fires once a month; the nudge trigger fires every day and checks each outstanding pre-bill's own age against your configured thresholds.

---

## Step 5 — Test

The workflow ships with pinned sample Clio data on both fetch nodes (two draft pre-bills with different ages, for two different attorneys), so you can test without waiting for a real schedule fire or having real draft bills in Clio yet.

**Compile & route path:**
1. Run **"Fetch Draft Pre-Bills From Clio"** → **"Group Draft Pre-Bills By Attorney"** and confirm you get one group per attorney, each with their own bill list.
2. Continue through **"Generate Priority Insight"** and **"Extract Priority Insight"** and confirm you get a short, non-legal `priority_insight` sentence back — something like "Start with the oldest bill first." Temporarily disconnect the Anthropic credential and re-run to confirm the workflow still completes and sends a normal email with no priority-tip callout, just to see the graceful-failure behavior once.
3. Continue through to **"Send Pre-Bill Review Email"** and confirm the styled HTML email lists every bill for that attorney with the right matter, client, and dollar amount, plus the priority tip in the teal callout box at the top.

**Nudge path:**
1. Run **"Fetch Outstanding Draft Pre-Bills"** → **"Compute Ages And Filter Due Pre-Bills"** and confirm `days_in_draft` is calculated correctly for each pinned bill, and that `tier` only gets set for bills that land on an exact threshold (a bill that's 3 days old with a 5-day reminder threshold correctly shows no tier — that's expected, not a bug).
2. To test each tier without waiting for a real bill to age into it, temporarily edit the pinned data's `created_at` values (or add a test row directly in Clio) to be exactly `FIRM_PREBILL_REMINDER_DAYS` days old, then `FIRM_PREBILL_URGENT_DAYS`, then `FIRM_PREBILL_ESCALATE_DAYS` — confirm each produces the right tier.
3. With a bill at the `overdue` tier, confirm the flow routes to **"Build Partner Escalation Email"** instead of the attorney nudge — check `FIRM_PARTNER_EMAIL` receives it, not the attorney, and that the styled email header reads "Escalation" in red.
4. Confirm an attorney with *both* a reminder-tier and an urgent-tier bill gets exactly one email listing both, not two separate emails, and that the header color reflects the more urgent tier present (amber for reminder-only, orange for urgent).

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Compile day, at least one draft pre-bill exists | Grouped by attorney, one styled review email per attorney with a Claude priority tip |
| Compile day, nothing in draft | Skipped — no emails, nothing to route |
| Claude's priority-tip call fails, is flagged, or gets caught by the keyword scan | Review email still sends normally, just without the priority-tip callout |
| A matter has no responsible attorney assigned | That bill is silently excluded from both compile and nudge — there's nobody to send it to |
| A pre-bill is younger than `FIRM_PREBILL_REMINDER_DAYS` | No nudge yet — this is the normal, expected state for most drafts on most days |
| A pre-bill hits exactly `FIRM_PREBILL_REMINDER_DAYS` or `FIRM_PREBILL_URGENT_DAYS` old | Attorney gets a direct reminder, tone reflecting the more urgent of any bills they have outstanding |
| A pre-bill reaches `FIRM_PREBILL_ESCALATE_DAYS` old (and every `FIRM_PREBILL_ESCALATE_REPEAT_DAYS` after) | Managing partner gets the escalation instead of the attorney, repeating until it's finalized |
| Every outstanding pre-bill was already finalized before the nudge check ran | Skipped — the good outcome, not a failure |

---

## Compliance note

This template performs scheduling and internal notification only — it never reads, summarizes, or comments on the substance of any bill, and it never contacts a client (ABA Op. 512). The priority tip is generated from dollar amounts and bill age only, is explicitly instructed never to comment on case merits or legal strategy, and is checked twice (Claude's own self-report, then an independent keyword scan) before it's allowed into an email. Every routing email and every escalation is logged to the Bar-Compliance Guardrail's audit sheet for record-keeping, though opt-out checking doesn't apply since nothing here reaches a client.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces the manual chasing that stalls the monthly billing cycle — a stalled cycle directly delays revenue |
| Compliance test | Scheduling and internal notification only, no legal work, no client contact |
| Searchability test | Ranks for "law firm pre-bill automation" |
| Deployability test | Reuses Clio, Anthropic, and SMTP credentials from earlier templates if already deployed — under 15 minutes |
| Upsell test | Quick-Win Build adds Slack/Teams nudges, a firm-wide billing-cycle dashboard, and automatic finalize-and-send once an attorney approves |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Slack or Microsoft Teams nudges alongside email, for firms where attorneys live in chat instead of their inbox
- A firm-wide billing-cycle dashboard showing every attorney's review status at a glance
- One-click finalize-and-send once an attorney approves their pre-bills, no separate trip into Clio required
- Custom nudge cadences per attorney based on their historical response time

[Book a call with Protomated](https://protomated.com/book) to get started.
