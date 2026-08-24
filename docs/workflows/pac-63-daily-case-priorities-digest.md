# Deploy Guide: Daily Case Priorities Digest

**Template:** PTPAC-63 — K8: Morning Digest (lite)
**Pillar:** Keep
**Replaces:** Attorneys manually cross-referencing Clio's task list and calendar each morning to figure out what's due

Get value in under 15 minutes. Every morning, the workflow pulls three things out of Clio in parallel — every open task, every calendar entry (hearings, court dates, appointments) coming up, and the firm's user directory (used to resolve who to email) — and groups everything by attorney into three sections: **Priorities** (overdue tasks), **Deadlines** (upcoming calendar entries), and **Follow-ups** (tasks due today or this week). Each attorney gets one styled email with everything they need to know before their first coffee, pulled from both systems they already use — no manual cross-referencing required.

---

## Before you start: heads up on this one

**This is a lite preview, not the full LegalContext Morning Digest.** It only reads task due dates and calendar entries from Clio — it does not pull case status, billing alerts, or generate an AI summary. Every email ends with a short teaser pointing to the full LegalContext Morning Digest product, which does all of that.

**Bucketing is date math and a Clio "priority" flag, nothing more.** A task is a "Priority" if its due date is in the past, a "Follow-up" if it's due today or within the follow-up window, and a calendar entry is a "Deadline" if it falls within that same window. Clio's own `priority: high` flag adds a star to a task line. There's no AI judgment and no legal analysis of what any task or entry actually is (ABA Op. 512) — just a sort by date and a flag Clio already tracks.

**Three Clio resources, one credential.** Tasks (`tasks.json`) and calendar entries (`calendar_entries.json`) are fetched in parallel, each split into individual records, filtered, sorted chronologically, then merged before grouping — all with native n8n nodes (Split Out, Filter, Sort, Merge), not one big custom script. A third fetch pulls the firm's user directory (`users.json`), used only to resolve emails (see below). All three use the same Clio OAuth2 credential; no separate calendar integration to configure.

**Confirmed live: Clio will not return a User's email as a nested field on tasks.json.** Requesting `assignee{id,name,email}` is rejected outright with `"assignee{email} is not a valid field"` — Users, unlike Contacts, don't expose email through nested field selection on this endpoint. This template works around it by fetching the firm's user directory once per run (`Fetch Firm Users From Clio` — email *is* requestable there, since it's the primary resource) and resolving each task's `assignee.id` against it. Calendar attendees are checked for a direct email first (in case that resource behaves differently — unconfirmed either way) and fall back to the same lookup.

**Tasks and calendar entries whose assignee/attendee can't be resolved to an email, or that have no date, are silently excluded.** This includes tasks assigned to someone who isn't in the fetched user directory (e.g. a disabled or external user) — check Clio periodically for records stuck in that state.

**Confirmed live: `calendar_entries.json`'s `from`/`to` need a full ISO 8601 datetime, not a bare date.** A plain date like `2026-08-24` is rejected with `ArgumentError: An invalid argument was supplied: invalid xmlschema format`. Fixed by using `$now.startOf('day').toISO()` instead of `$now.toISODate()`. All three fetches have "Never Error" turned on and degrade gracefully — if the calendar endpoint isn't available on your Clio plan, or the user-directory fetch fails, the digest still sends with whatever it could resolve rather than failing outright.

**If your Clio account is on a regional subdomain (e.g. `eu.app.clio.com`), `CLIO_BASE_URL` must match exactly.** All three fetches use it — a US default against an EU account (or vice versa) will silently return nothing usable.

**Your Clio OAuth2 app needs Tasks, Calendar, and Users read scopes — granted *before* you authorize the credential.** All three showed up as `ForbiddenError: "User is forbidden from taking that action"` when the Users scope wasn't enabled on the app. Checking a scope's box in Clio's app config (`developers.clio.com/apps/...`) does **not** retroactively apply to a credential you already authorized — the scope is baked into the token at the moment you consent. If you add or change a scope after the fact, you must reconnect/reauthorize the credential in n8n (Credentials → your Clio credential → sign in again) to get a token that actually carries it.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** This template calls it purely for audit logging — every digest is internal (attorney only), never a client, so opt-out checking doesn't apply, but every send is still recorded.
- An n8n instance (self-hosted or n8n Cloud)
- A Clio account with API access, including calendar and user-directory read access (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61 if already deployed)
- Every relevant Clio task should be assigned to an enabled firm user with a valid email, and every relevant calendar entry should have at least one attendee that resolves to one — records that don't are silently excluded
- Know your Clio region — check whether your account is on `app.clio.com`, `eu.app.clio.com`, or another regional subdomain (visible in your browser's address bar when logged into Clio)
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-63-daily-case-priorities-digest.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-63-daily-case-priorities-digest.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if PAC-18, PAC-20, PAC-55, PAC-57, PAC-58, PAC-59, or PAC-61 is already deployed. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide. The same credential covers all three fetches (tasks, calendar entries, and the user directory).
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

The workflow ships with pinned sample Clio data on the trigger and all three fetch nodes (six tasks and three calendar entries across two attorneys, plus a two-user firm directory, covering overdue, due-today, follow-up, out-of-window, and missing-assignee/attendee cases), so you can test without waiting for a real schedule fire.

1. Run **"Fetch Open Tasks From Clio"** → **"Split Out Task Records"** → **"Filter Tasks With Assignee And Due Date"** and confirm the no-assignee task (id 9006) is dropped.
2. Run **"Fetch Upcoming Calendar Entries From Clio"** → **"Split Out Calendar Entries"** → **"Filter Calendar Entries With Attendee And Date"** and confirm the no-attendee entry (id 7003) is dropped.
3. Run **"Fetch Firm Users From Clio"** on its own and confirm it returns the two sample users with their emails.
4. Continue both fetch branches through their **Sort** nodes, into **"Merge Tasks And Calendar Entries"**, then **"Group Priorities Deadlines And Follow-ups By Attorney"** — confirm each attorney's Priorities, Deadlines, and Follow-ups arrays contain the right records, with emails resolved from the user directory rather than from the tasks/calendar data itself.
5. Continue through **"If Any Attorney Has Digest Items"** → **"Split Digest By Attorney"** and confirm one item per attorney.
6. Continue through to **"Send Morning Digest Email"** and confirm the styled HTML email shows the right sections (only the ones with items), a star on the high-priority task, the right calendar entry times, and the LegalContext teaser at the bottom.
7. Temporarily clear the pinned data on all three fetch nodes down to empty `data` arrays and re-run to confirm the workflow reaches **"Skip — Nothing To Digest Today"** with no email sent.

**Testing against your real Clio account:** unpin the three fetch nodes (click the pin icon on each), connect a real Clio OAuth2 credential, and seed one overdue task and one upcoming calendar entry assigned to yourself so there's something to see. Run the workflow and check each fetch node's raw output before assuming anything is broken — an empty or `Skip`-routed result can mean either "genuinely nothing due" or "a field name mismatch dropped everything silently." If a fetch node's output shows an `error` object instead of a `data` array, open it and read the `message` — that's Clio telling you exactly which field it rejected.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| At least one attorney has an overdue task, an upcoming calendar entry, or a follow-up task | One styled digest email per attorney, sections only shown for buckets that have items |
| No open task is overdue or due within the window, and no calendar entry falls within the window | Skipped — no emails, nothing to route |
| A task or calendar entry has no assignee/attendee at all, or no date | Silently excluded from every digest — there's nobody to send it to, or no date to sort it by |
| A task or calendar entry has an assignee/attendee, but their id doesn't match anyone in the user directory fetch | Silently excluded — e.g. assigned to a disabled or external user not returned by `users.json` |
| A task's due date is further out than `FIRM_DIGEST_FOLLOWUP_DAYS` | Excluded — this is a morning digest, not a full task list |
| The calendar-entries or user-directory fetch fails, or the calendar endpoint isn't available on your Clio plan | Digest still sends with whatever resolved successfully, rather than failing outright |
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
