# Deploy Guide: Client Onboarding Sequence

**Template:** PTPAC-64 — K7: Client Onboarding Sequence
**Pillar:** Keep
**Replaces:** A Lawmatics-style onboarding journey

Get value in under 15 minutes. Every day, the workflow checks every open Clio matter and works out — purely from each matter's own open date — whether today is one of four onboarding-sequence days: **Day 0** (welcome + portal access), **Day 1** (what to expect), **Day 3** (document checklist), **Day 7** (how billing works). Whichever stage is due gets built, checked against your compliance guardrail, and sent. A rocky, silent first week is one of the biggest drivers of early client churn — this keeps every new client hearing from the firm on a predictable cadence without anyone having to remember to do it by hand.

---

## Before you start: heads up on this one

**This is a genuinely client-facing template — unlike PAC-61 and PAC-63, which only ever email internal staff.** Every send here goes through the full Bar-Compliance Guardrail flow: opt-out check, disclaimer/unsubscribe footer, and audit log. If a client has opted out, the guardrail suppresses the send and this workflow respects that — check `Skip — Client Opted Out` in your execution history if a client reports not receiving a stage email.

**There's no separate list tracking which stage has already been sent.** Each stage fires on an exact day-since-open match (0, 1, 3, 7) computed fresh from Clio's own `open_date` on every run — there's nothing to get out of sync, and nothing to migrate if you change the day thresholds later (a matter simply starts hitting the new days going forward).

**All four stages are plain text, not styled HTML.** The compliance guardrail only guarantees the opt-out footer is present in the plain-text `message_out` it returns — sending a separate HTML body without that same footer baked in would mean most email clients render the HTML (with no footer) instead of the compliant text version. Keeping this text-only avoids that risk entirely.

**One Clio field hasn't been independently confirmed live:** `client{primary_email_address}` as a nested field on `matters.json`. That field name is reused from how this catalog's Review Request Engine (NTC-17) already refers to Clio contact emails elsewhere — and PAC-63 confirmed live that a *User's* email is rejected as a nested field on `tasks.json`, but a matter's `client` is a *Contact*, which behaves differently. Check your first real run's output on `Fetch Open Matters From Clio` and adjust the field name if Clio rejects it or returns something else.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first** — this template cannot send a single email without it; every send is gated on the guardrail's `approved` response.
- An n8n instance (self-hosted or n8n Cloud)
- A Clio account with API access, matter read scope (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61/63 if already deployed)
- Every relevant Clio matter should have a client with a valid email on file — matters without one are silently excluded
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-64-client-onboarding-sequence.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-64-client-onboarding-sequence.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if PAC-18, PAC-20, PAC-55, PAC-57, PAC-58, PAC-59, PAC-61, or PAC-63 is already deployed. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide.
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | The contact address mentioned in the emails (e.g. "reach us at...") |
| `FIRM_FROM_EMAIL` | The sender address for the onboarding emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_CLIENT_PORTAL_URL` *(optional)* | Link to your client portal (Clio Connect, MyCase portal, etc.) — copy adapts gracefully if left unset |
| `FIRM_BILLING_EMAIL` *(optional)* | Contact address for billing questions — falls back to `FIRM_EMAIL` if unset |
| `FIRM_RESPONSE_TIME_DAYS` *(optional)* | Response-time expectation mentioned in the Day 1 email — defaults to `1-2` |
| `FIRM_PAYMENT_TERMS_DAYS` *(optional)* | Payment terms mentioned in the Day 7 email — defaults to `30` |
| `FIRM_ONBOARDING_CHECKLIST` *(optional)* | Newline-separated list of documents for the Day 3 email — defaults to a generic 4-item list |

---

## Step 3 — Set the daily check time (2 min)

Open **"When Onboarding Check Runs"** and adjust the cron expression if 9am (n8n server timezone) isn't the right time for your firm.

---

## Step 4 — Activate

Toggle the workflow to **Active**. It runs on its own from then on — no webhook URL to wire anywhere.

---

## Step 5 — Test

The workflow ships with pinned sample Clio data (six matters covering all four stage days, one already past the sequence window, and one with no client email on file), so you can test without waiting for a real schedule fire or having real matters at the right ages yet.

1. Run **"Fetch Open Matters From Clio"** → **"Compute Onboarding Stage For Each Matter"** and confirm: the matter opened 10 days ago and the matter with no client email are both excluded, and the other four each get the correct stage (`day0`, `day1`, `day3`, `day7`).
2. Continue through **"If Onboarding Stage Due Today"** → **"Build Onboarding Stage Email"** for each of the four and confirm the subject/body match the intended stage, with `matterLabel`, client name, and any optional variables filled in correctly.
3. Continue through **"Check Compliance Before Sending"** → **"If Onboarding Email Approved"** → **"Send Onboarding Stage Email"** and confirm the delivered email carries the guardrail's unsubscribe footer.
4. Add a row to the Guardrail's Opt-Outs sheet for one of the pinned client emails, re-run, and confirm that matter routes to **"Skip — Client Opted Out"** instead of sending.
5. Temporarily clear the pinned data on the fetch node down to an empty `data` array and re-run to confirm the workflow reaches **"Skip — No Onboarding Stage Due Today"** with no email sent.

**Testing against your real Clio account:** unpin `Fetch Open Matters From Clio`, connect a real Clio OAuth2 credential, and check the raw output before assuming anything is broken. If every matter gets excluded even though one should be at day 0, check whether `client.primary_email_address` actually appears in the response — this is the one unconfirmed field flagged above.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A matter is exactly 0, 1, 3, or 7 days since its Clio open date | The matching stage email is built and sent (unless opted out) |
| A matter is any other age | Excluded for today — no email, no error |
| A matter's client has no email on file, or the matter has no open date | Silently excluded from every stage |
| No open matter hits a sequence day today | Skipped — no emails, nothing to route |
| The client has opted out of firm emails | Guardrail suppresses the send; already logged to the audit sheet |
| The Clio matters fetch fails | Degrades to a quiet skip rather than failing the run |

---

## Compliance note

This template performs scheduling and templated operational notification only — portal access, process expectations, a document checklist, and how billing works. It never comments on the substance of the matter and never gives legal guidance (ABA Op. 512). Every send goes through the Bar-Compliance Guardrail for opt-out checking, the required disclaimer, and audit logging — this is a genuine client-facing template, not an internal-only notification.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces a Lawmatics-style onboarding journey and the manual follow-up firms do instead — a rocky, silent first week is a top driver of early client churn |
| Compliance test | Operational onboarding notifications only, fully wrapped in OPS1, no legal work |
| Searchability test | Ranks for "law firm client onboarding automation" |
| Deployability test | Reuses Clio and SMTP credentials from earlier templates if already deployed — under 15 minutes |
| Upsell test | Clear Quick-Win Build path: e-signature-triggered start (DocuSign envelope completed) instead of a daily poll, SMS alongside email, and firm-specific branching by practice area |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Trigger straight off a completed DocuSign engagement letter instead of polling Clio daily
- SMS touches alongside email for firms whose clients respond faster to text
- Practice-area-specific document checklists and timelines instead of one generic sequence
- A dashboard showing every client's onboarding progress at a glance

[Book a call with Protomated](https://protomated.com/book) to get started.
