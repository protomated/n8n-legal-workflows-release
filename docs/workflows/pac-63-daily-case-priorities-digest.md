# Deploy Guide: Daily Case Priorities Digest

**Template:** PTPAC-63 — K8: Morning Digest (lite)
**Pillar:** Keep
**Replaces:** Attorneys manually cross-referencing Clio's task list and calendar each morning to figure out what's due

Get value in under 15 minutes. Every morning, the workflow pulls two things out of Clio in parallel — every open task, and every calendar entry (hearings, court dates, appointments) coming up — and groups everything by attorney into three sections: **Priorities** (overdue tasks), **Deadlines** (upcoming calendar entries), and **Follow-ups** (tasks due today or this week). Each attorney gets one styled email with everything they need to know before their first coffee, pulled from both systems they already use — no manual cross-referencing required.

---

## Before you start: heads up on this one

**This is a lite preview, not the full LegalContext Morning Digest.** It only reads task due dates and calendar entries from Clio — it does not pull case status, billing alerts, or generate an AI summary. Every email ends with a short teaser pointing to the full LegalContext Morning Digest product, which does all of that.

**Bucketing is date math and a Clio "priority" flag, nothing more.** A task is a "Priority" if its due date is in the past, a "Follow-up" if it's due today or within the follow-up window, and a calendar entry is a "Deadline" if it falls within that same window. Clio's own `priority: high` flag adds a star to a task line. There's no AI judgment and no legal analysis of what any task or entry actually is (ABA Op. 512) — just a sort by date and a flag Clio already tracks.

**Two Clio resources, two branches, one credential.** Tasks (`tasks.json`) and calendar entries (`calendar_entries.json`) are fetched in parallel, each split into individual records, filtered down to the ones with someone to notify and a date, sorted chronologically, then merged before grouping — all with native n8n nodes (Split Out, Filter, Sort, Merge), not one big custom script. Both use the same Clio OAuth2 credential; no separate calendar integration to configure.

**Tasks and calendar entries with no assignee/attendee email, or no date, are silently excluded.** There's nobody to send them to, or no date to sort them by. Check Clio periodically for records stuck in that state.

**A few Clio field names haven't been independently confirmed live:**
- `due_at` and `priority` on `tasks.json`'s GET response — these are the field names this catalog already uses when *creating* Clio tasks (see NTC-16, PAC-59), but reading them back on this endpoint hasn't itself been confirmed.
- `attendees` on `calendar_entries.json`, and the exact shape Clio expects for the `from`/`to` date-range query parameters.

Check your first real run's output against both `Fetch Open Tasks From Clio` and `Fetch Upcoming Calendar Entries From Clio`, and adjust field names in either node if something doesn't match. Both fetches have "Never Error" turned on and degrade gracefully — if the calendar endpoint isn't available on your Clio plan, the digest still sends with just the task sections.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** This template calls it purely for audit logging — every digest is internal (attorney only), never a client, so opt-out checking doesn't apply, but every send is still recorded.
- An n8n instance (self-hosted or n8n Cloud)
- A Clio account with API access, including calendar read access (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61 if already deployed)
- Every relevant Clio task should have an assignee with a valid email, and every relevant calendar entry should have at least one attendee with a valid email — records without one are silently excluded
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-63-daily-case-priorities-digest.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-63-daily-case-priorities-digest.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if PAC-18, PAC-20, PAC-55, PAC-57, PAC-58, PAC-59, or PAC-61 is already deployed. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide. The same credential covers both the tasks fetch and the calendar fetch.
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | The sender address for the digest emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_DIGEST_FOLLOWUP_DAYS` *(optional)* | How many days ahead counts as "coming up this week" for follow-ups, and the calendar lookahead window for deadlines — defaults to `7` |
| `LEGALCONTEXT_UPSELL_URL` *(optional)* | Link used in the digest's LegalContext teaser — falls back to a generic `https://protomated.com/legalcontext` URL if unset |

---

## Step 3 — Set the morning schedule (2 min)

Open **"When Morning Digest Runs"** and set the cron expression to whenever your firm wants the digest waiting in their inbox — the default is every day at 7am. Restrict it to weekdays (e.g. `0 7 * * 1-5`) if a weekend digest isn't useful for your firm.

---

## Step 4 — Activate

Toggle the workflow to **Active**. It runs on its own from then on — no webhook URL to wire anywhere.

---

## Step 5 — Test

The workflow ships with pinned sample Clio data on the trigger and both fetch nodes (six tasks and three calendar entries across two attorneys, covering overdue, due-today, follow-up, out-of-window, and missing-assignee/attendee cases), so you can test without waiting for a real schedule fire.

1. Run **"Fetch Open Tasks From Clio"** → **"Split Out Task Records"** → **"Filter Tasks With Assignee And Due Date"** and confirm the no-assignee task (id 9006) is dropped.
2. Run **"Fetch Upcoming Calendar Entries From Clio"** → **"Split Out Calendar Entries"** → **"Filter Calendar Entries With Attendee And Date"** and confirm the no-attendee entry (id 7003) is dropped.
3. Continue both branches through their **Sort** nodes, into **"Merge Tasks And Calendar Entries"**, then **"Group Priorities Deadlines And Follow-ups By Attorney"** — confirm each attorney's Priorities, Deadlines, and Follow-ups arrays contain the right records.
4. Continue through **"If Any Attorney Has Digest Items"** → **"Split Digest By Attorney"** and confirm one item per attorney.
5. Continue through to **"Send Morning Digest Email"** and confirm the styled HTML email shows the right sections (only the ones with items), a star on the high-priority task, the right calendar entry times, and the LegalContext teaser at the bottom.
6. Temporarily clear the pinned data on both fetch nodes down to empty `data` arrays and re-run to confirm the workflow reaches **"Skip — Nothing To Digest Today"** with no email sent.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| At least one attorney has an overdue task, an upcoming calendar entry, or a follow-up task | One styled digest email per attorney, sections only shown for buckets that have items |
| No open task is overdue or due within the window, and no calendar entry falls within the window | Skipped — no emails, nothing to route |
| A task or calendar entry has no assignee/attendee email, or no date | Silently excluded from every digest — there's nobody to send it to, or no date to sort it by |
| A task's due date is further out than `FIRM_DIGEST_FOLLOWUP_DAYS` | Excluded — this is a morning digest, not a full task list |
| The calendar entries endpoint fails or isn't available on your Clio plan | Digest still sends, with just the Priorities and Follow-ups sections |
| A task is flagged `priority: high` in Clio | Its digest line gets a star, in both the HTML and plain-text email |

---

## Compliance note

This template performs scheduling and internal notification only — it reads task due dates, a Clio priority flag, and calendar entry times, and never summarizes, analyzes, or comments on the substance of any matter (ABA Op. 512). It never contacts a client; every recipient is an attorney at the firm. Every digest send is logged to the Bar-Compliance Guardrail's audit sheet for record-keeping, though opt-out checking doesn't apply since nothing here reaches a client.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Saves attorneys the daily manual cross-reference of Clio's task list and calendar to figure out what's overdue, due today, and coming up — 20-30 minutes a day for a typical caseload, 2+ hours a week |
| Compliance test | Scheduling and internal notification only, no legal work, no client contact |
| Searchability test | Ranks for "daily case priorities digest for lawyers" |
| Deployability test | Reuses one Clio credential and SMTP from earlier templates if already deployed — under 15 minutes |
| Upsell test | Teases the full LegalContext Morning Digest — case status, billing alerts, and an AI-summarized action list, not just a sort of tasks and calendar entries |

---

## Want the full digest?

This lite template only sorts Clio task due dates and calendar entries by date. **LegalContext Morning Digest** adds:
- Case status pulled directly from your matter data, not just tasks and calendar
- Billing and trust account alerts alongside your priorities
- An AI-summarized action list — the "so what" for each item, not just a list
- Delivery to Slack or Microsoft Teams alongside email

[See LegalContext](https://protomated.com/legalcontext) to get started.
