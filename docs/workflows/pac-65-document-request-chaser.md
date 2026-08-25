# Deploy Guide: Document Request Chaser

**Template:** PTPAC-65 — K5: Document Request Chaser
**Pillar:** Keep
**Replaces:** Manual paralegal follow-up chasing outstanding client documents

Get value in under 15 minutes. Every day, the workflow pulls three things from Clio in sequence — the firm's user directory, every open matter, and every pending task — and filters down to outstanding document requests (a configurable Clio task type). Each one's client is resolved from the matters lookup, and its days-overdue is computed fresh from its own due date and matched against three escalation tiers: **1-2 days overdue** (friendly reminder), **7-8 days** (firmer follow-up), **14-15 days** (urgent, plus a direct internal alert to the responsible staff member). Mark the Clio task complete once the document arrives, and every future reminder stops automatically — nothing else to update.

---

## Before you start: heads up on this one

**This is client-facing** — every reminder goes through the full Bar-Compliance Guardrail flow: opt-out check, disclaimer/unsubscribe footer, and audit log. The urgent-tier internal staff alert is the one exception — it never reaches a client, so it bypasses the guardrail entirely (same pattern as PAC-18's day-60 partner escalation).

**A document request is just a Clio task, with one convention.** Create a task named for what's needed (e.g. "Photo ID from client"), set a due date, assign it to whoever's chasing it, and set its task type to **Document Request** (or whatever you set `FIRM_DOCUMENT_REQUEST_TASK_TYPE` to). Mark it complete in Clio the moment the document actually arrives — that's the entire "stop reminding" mechanism. There's no separate list or sheet to keep in sync.

**Three Clio integrations, wired in sequence, not parallel.** `Fetch Firm Users From Clio` runs first, passes straight through into `Fetch Open Matters From Clio`, which passes straight through into `Fetch Outstanding Document Tasks From Clio` — this isn't cosmetic. PAC-64 confirmed live that a disconnected side-fetch (referenced only by name in code, with no real connection into the graph) is **not reliably executed by n8n** on every run or every testing method. Wiring every fetch as a genuine sequential dependency fixes that for good.

**Confirmed live: Clio rejects a doubly-nested field request.** The natural way to get a task's client would be `tasks.json`'s `matter{...,client{...}}` — but that fails outright with `InvalidFields: "matter} is not a valid field"`. Clio's field selector only supports one level of nesting; `client{...}` nested inside `matter{...}` nested inside the tasks fetch is two levels deep. So the client is resolved a different way: `Fetch Open Matters From Clio` pulls every matter with `client{id,name,primary_email_address}` nested just **one** level (exactly the pattern PAC-64 already confirmed works on `matters.json`), and `Compute Days Overdue And Escalation Tier` looks up each task's matter id against it — the same lookup-table approach already used for resolving the assignee against the user directory.

**Escalation windows are 2 days wide** (1-2, 7-8, 14-15) on purpose — same as PAC-18's invoice ladder — so a reminder isn't silently skipped if the daily cron runs a day late (server restart, holiday, etc.).

**A few Clio fields haven't been independently confirmed live on this exact combination:** `task_type{id,name}` and `assignee{id,name}` as nested fields on `tasks.json`. Each individual piece is confirmed elsewhere in this catalog (PAC-59 uses task fields), but not together on one call. Check your first real run's output on `Fetch Outstanding Document Tasks From Clio` and adjust field names if Clio rejects either.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first** — this template cannot send a client reminder without it.
- An n8n instance (self-hosted or n8n Cloud)
- A Clio account with API access, task and user-directory read scope (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61/63/64 if already deployed)
- A Clio task type set up for document requests (default expected name: **Document Request**)
- Every outstanding document tracked as a task of that type, with a due date, assigned to a real Clio user, on a matter whose client has a valid email
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-65-document-request-chaser.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-65-document-request-chaser.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide — and make sure the app has **Tasks**, **Matters**, and **Users** read scopes granted *before* you authorize it (PAC-64 hit a `ForbiddenError` when a scope was added after the credential was already connected — reconnect if you add a scope later).
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | The generic contact address used when a task's assignee can't be resolved |
| `FIRM_FROM_EMAIL` | The sender address for both client reminders and staff escalations |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_CLIENT_PORTAL_URL` *(optional)* | Link to your client portal for uploads — copy adapts gracefully if left unset |
| `FIRM_DOCUMENT_REQUEST_TASK_TYPE` *(optional)* | The exact Clio task type name to treat as a document request — defaults to `Document Request` |
| `FIRM_PARALEGAL_EMAIL` *(optional)* | Fallback address for the urgent-tier staff escalation if the task's assignee can't be resolved — falls back to `FIRM_EMAIL` if unset |

---

## Step 3 — Set the daily check time (2 min)

Open **"When Document Chaser Runs"** and adjust the cron expression if 8am (n8n server timezone) isn't the right time for your firm.

---

## Step 4 — Activate

Toggle the workflow to **Active**. It runs on its own from then on — no webhook URL to wire anywhere.

---

## Step 5 — Test

The workflow ships with pinned sample Clio data on all three Clio-facing nodes: a two-user firm directory, five open matters (one with no client email on file), and six document tasks covering all three tiers plus exclusion cases (wrong task type, an unresolvable client, and one 5-days-overdue task that doesn't land on any tier).

1. Run **"Fetch Outstanding Document Tasks From Clio"** → **"Split Out Task Records"** → **"Filter Document Request Tasks With Due Date"** and confirm the wrong-task-type task is dropped.
2. Continue through **"Compute Days Overdue And Escalation Tier"** and confirm each surviving task's `client_email` was correctly resolved from the matters lookup (empty for the one whose matter has no client email on file).
3. Continue through **"Filter Tasks Due For A Reminder Today"** and confirm the 5-days-overdue task and the no-client-email task are both dropped, and the remaining three carry `tier: reminder`, `second_reminder`, and `urgent` respectively.
4. Continue through **"Check If Any Reminder Due Today"** → **"If Any Reminder Due Today"** and confirm it fans out to both **"Build Document Reminder Email"** and **"If Tier Is Urgent"**.
5. For each of the three surviving tasks, confirm **"Build Document Reminder Email"** produces the right tone for its tier, with the assignee's contact email substituted in correctly.
6. Confirm only the `urgent`-tier task reaches **"Build Staff Escalation Email"** → **"Send Staff Escalation Email"** — the other two should hit **"Skip — No Staff Escalation Needed"**.
7. Continue the client-email path through **"Check Compliance Before Sending"** → **"If Reminder Approved"** → **"Send Document Reminder Email"** and confirm the delivered email carries the guardrail's unsubscribe footer.
8. Add a row to the Guardrail's Opt-Outs sheet for one of the pinned client emails, re-run, and confirm that task routes to **"Skip — Client Opted Out"** instead of sending (the staff escalation, if applicable, still fires independently — opting out of client emails doesn't suppress internal staff alerts).
9. Temporarily clear the pinned data on all three Clio-facing nodes down to empty `data` arrays and re-run to confirm the workflow reaches **"Skip — Nothing To Chase Today"** with no email sent.

**Testing against your real Clio account:** unpin all three Clio-facing nodes, connect a real credential, and create one real "Document Request" task due a day or two ago so it lands on the `reminder` tier immediately. Check each node's raw output before assuming anything is broken — see the field-name caveats above.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A document request task is 1-2, 7-8, or 14-15 days overdue | The matching-tier reminder is built and sent to the client (unless opted out) |
| A document request task is any other age (including not yet due) | Excluded for today — no email, no error |
| A task isn't the configured document-request type, or has no due date | Excluded before tier computation even runs |
| A task's matter doesn't resolve to a client with an email on file (not in the open-matters fetch, no email set) | Silently excluded from every tier |
| A task's assignee can't be resolved (unassigned, disabled user, or the directory fetch failed) | Reminder still sends — falls back to the generic firm contact address instead of naming staff |
| No document request hits a tier today | Skipped — no emails, nothing to route |
| The client has opted out of firm emails | Guardrail suppresses that client email; already logged to the audit sheet. The urgent-tier staff escalation is unaffected — it's not a client message |
| A task reaches the urgent tier (14-15 days overdue) | The client reminder still sends, and a separate internal email alerts the responsible staff member directly |
| The document arrives and the task is marked complete in Clio | It no longer appears in the `status=pending` fetch — all future reminders stop automatically |

---

## Compliance note

This template performs scheduling and templated operational follow-up only — chasing a document that was already requested, never any comment on the matter's substance or legal guidance (ABA Op. 512). Every client-facing reminder goes through the Bar-Compliance Guardrail for opt-out checking, the required disclaimer, and audit logging. The urgent-tier staff alert is purely internal and never reaches a client.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces the manual paralegal follow-up (checking what's outstanding, drafting reminder emails, escalating stuck requests) that eats roughly 1-2 hours a week per matter load |
| Compliance test | Operational document-chasing only, fully wrapped in OPS1, no legal work |
| Searchability test | Ranks for "client document reminder automation" |
| Deployability test | Reuses Clio and SMTP credentials from earlier templates if already deployed — under 15 minutes |
| Upsell test | Clear Quick-Win Build path: SMS reminders alongside email, a client-facing upload portal, and automatic task creation from an intake checklist instead of manual task setup |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- SMS reminders alongside email for clients who respond faster to text
- A branded upload portal instead of "reply with it attached"
- Automatic document-request task creation from a practice-area intake checklist, so nothing has to be set up by hand per matter
- A dashboard showing every outstanding document request and its age at a glance

[Book a call with Protomated](https://protomated.com/book) to get started.
