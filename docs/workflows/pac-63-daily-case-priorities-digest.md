# Deploy Guide: Daily Case Priorities Digest

**Template:** PTPAC-63 — K8: Morning Digest (lite)
**Pillar:** Keep
**Replaces:** Attorneys manually scanning Clio each morning to figure out what's due

Get value in under 15 minutes. Every morning, the workflow pulls every open task out of Clio, groups it by the assigned attorney, and sorts it into overdue, due today, and due this week. Each attorney gets one styled email with everything they need to know before their first coffee — no manual scanning through Clio's task list required.

---

## Before you start: heads up on this one

**This is a lite preview, not the full LegalContext Morning Digest.** It only reads task due dates from Clio and sorts them into three buckets — it does not pull case status, billing alerts, or generate an AI summary. Every email ends with a short teaser pointing to the full LegalContext Morning Digest product, which does all of that.

**Bucketing is pure date math, nothing else.** A task is "overdue" if its due date is in the past, "due today" if it matches today, and a "follow-up" if it falls within the follow-up window (7 days by default). There's no priority scoring, no AI judgment, and no legal analysis of what any task actually is (ABA Op. 512) — just a sort by date.

**Tasks with no assignee email or no due date are silently excluded.** There's nobody to send them to, or no date to sort them by. Check Clio periodically for tasks stuck in that state.

**`due_at` hasn't been independently confirmed on Clio's task GET response.** It's the field name this catalog already uses when *creating* Clio tasks (see NTC-16, PAC-59), but reading it back on `tasks.json` hasn't itself been confirmed live. Check your first real run's output and adjust the field name in "Fetch Open Tasks From Clio" if it comes back under a different key.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** This template calls it purely for audit logging — every digest is internal (attorney only), never a client, so opt-out checking doesn't apply, but every send is still recorded.
- An n8n instance (self-hosted or n8n Cloud)
- A Clio account with API access (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61 if already deployed)
- Every Clio task should have an assignee with a valid email — tasks without one are silently excluded, since there's nobody to send them to
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-63-daily-case-priorities-digest.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-63-daily-case-priorities-digest.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if PAC-18, PAC-20, PAC-55, PAC-57, PAC-58, PAC-59, or PAC-61 is already deployed. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide.
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | The sender address for the digest emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_DIGEST_FOLLOWUP_DAYS` *(optional)* | How many days ahead counts as "coming up this week" — defaults to `7` |
| `LEGALCONTEXT_UPSELL_URL` *(optional)* | Link used in the digest's LegalContext teaser — falls back to a generic `https://protomated.com/legalcontext` URL if unset |

---

## Step 3 — Set the morning schedule (2 min)

Open **"When Morning Digest Runs"** and set the cron expression to whenever your firm wants the digest waiting in their inbox — the default is every day at 7am. Restrict it to weekdays (e.g. `0 7 * * 1-5`) if a weekend digest isn't useful for your firm.

---

## Step 4 — Activate

Toggle the workflow to **Active**. It runs on its own from then on — no webhook URL to wire anywhere.

---

## Step 5 — Test

The workflow ships with pinned sample Clio data on both the trigger and fetch nodes (six tasks across two attorneys, covering overdue, due-today, follow-up, out-of-window, and no-assignee cases), so you can test without waiting for a real schedule fire.

1. Run **"Fetch Open Tasks From Clio"** → **"Group Tasks By Attorney And Bucket"** and confirm each attorney's tasks land in the right bucket, and that the task with no assignee and the task 27 days out are both excluded.
2. Continue through **"If Any Attorney Has Digest Items"** → **"Split Digest By Attorney"** and confirm you get one item per attorney with digest-worthy tasks.
3. Continue through to **"Send Morning Digest Email"** and confirm the styled HTML email shows the right sections (only the buckets that have items), the right task and matter details in each, and the LegalContext teaser at the bottom.
4. Temporarily clear the pinned data on "Fetch Open Tasks From Clio" down to an empty `data` array and re-run to confirm the workflow reaches **"Skip — Nothing To Digest Today"** with no email sent.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| At least one attorney has an overdue, due-today, or follow-up task | One styled digest email per attorney, sections only shown for buckets that have items |
| No open task is overdue, due today, or within the follow-up window | Skipped — no emails, nothing to route |
| A task has no assignee email, or no due date | Silently excluded from every digest — there's nobody to send it to, or no date to sort it by |
| A task's due date is further out than `FIRM_DIGEST_FOLLOWUP_DAYS` | Excluded — this is a morning digest, not a full task list |

---

## Compliance note

This template performs scheduling and internal notification only — it reads task due dates and groups them by date, and never summarizes, analyzes, or comments on the substance of any matter (ABA Op. 512). It never contacts a client; every recipient is an attorney at the firm. Every digest send is logged to the Bar-Compliance Guardrail's audit sheet for record-keeping, though opt-out checking doesn't apply since nothing here reaches a client.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Saves attorneys the daily manual scan through Clio's task list to figure out what's overdue, due today, and coming up — 20-30 minutes a day for a typical caseload, 2+ hours a week |
| Compliance test | Scheduling and internal notification only, no legal work, no client contact |
| Searchability test | Ranks for "daily case priorities digest for lawyers" |
| Deployability test | Reuses Clio and SMTP credentials from earlier templates if already deployed — under 15 minutes |
| Upsell test | Teases the full LegalContext Morning Digest — case status, billing alerts, and an AI-summarized action list, not just a task due-date sort |

---

## Want the full digest?

This lite template only sorts Clio task due dates. **LegalContext Morning Digest** adds:
- Case status pulled directly from your matter data, not just tasks
- Billing and trust account alerts alongside your priorities
- An AI-summarized action list — the "so what" for each item, not just a list
- Delivery to Slack or Microsoft Teams alongside email

[See LegalContext](https://protomated.com/legalcontext) to get started.
