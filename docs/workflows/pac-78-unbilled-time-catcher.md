# Deploy Guide: Unbilled-Time Catcher

**Template:** PTPAC-78 — B2: Unbilled-Time Catcher
**Pillar:** Keep
**Replaces:** LeanLaw, manual review

Get value in under 15 minutes. Firms typically bill only about 2.4 of an 8-hour day — activity with no matching time entry is money that simply evaporates. This workflow scans yesterday's Clio calendar against logged time entries, flags matters with activity but nothing billed, and puts a weekly dollar figure on the leak.

---

## Before you start: the most important thing to understand

**This workflow never creates a time entry or decides what's billable.** It only ever flags that something happened on a matter with no matching time entry that day. The attorney reviews and logs the actual entry themselves — with their own narrative and judgment about what's genuinely billable. There is no code path here that writes to Clio's time-entry data at all (ABA Op. 512 — surfacing a gap, never drafting or deciding billing content).

**Unbilled detection works at the matter level, not the minute level.** Matching a specific calendar event to a specific time entry down to the exact time of day isn't reliable from the Clio API alone, so this checks "did this matter have ANY time entry logged that day at all" — a defensible, honest approximation, not a precise audit.

**This has two independent parts on one canvas:**
1. **Daily unbilled activity check** — every morning, checks yesterday's calendar against yesterday's time entries, groups anything unbilled by responsible attorney, and emails each one a digest.
2. **Weekly revenue-impact summary** — re-runs the same check over the trailing 7 days and emails the firm a rough dollar estimate of the leak, using a configurable hourly rate.

**Coverage today is calendar entries only.** Logged calls and emails (Clio's Communications) are a natural extension, not built here yet — their exact API shape hasn't been confirmed against a live account the way calendar entries have elsewhere in this catalog.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — both branches' audit logging goes through it
- A Clio account with API access (reuse the OAuth2 credential from other templates in this catalog if already deployed)
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-78-unbilled-time-catcher.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-78-unbilled-time-catcher.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 2 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | Where the weekly summary goes, and fallback recipient for the daily digest if a matter has no responsible attorney |
| `FIRM_FROM_EMAIL` | Sender address for both branches' emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `DEFAULT_CALENDAR_DURATION_HOURS` *(optional)* | Estimated hours for a calendar entry with no clear start/end time — defaults to 0.5 |
| `DEFAULT_HOURLY_RATE` *(optional)* | Blended hourly rate used only for the weekly dollar estimate — defaults to 250 |

---

## Step 3 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expressions on **"Check For Unbilled Activity Daily"** (default 7am) and **"Check Weekly Unbilled Revenue Impact"** (default Monday 8am) if different timing suits.

---

## Step 4 — Test

The workflow ships with pinned sample data: two calendar entries yesterday (a 1-hour client meeting and a 15-minute call), and one time entry logged only for the second matter — meaning the first matter should be flagged as unbilled and the second should not.

**Daily branch:**
1. Run **"Fetch Prior Day Calendar Entries From Clio"** → **"Fetch Prior Day Time Entries From Clio"** → **"Fetch Open Matters For Unbilled Check"** → **"Find Unbilled Calendar Activity"** and confirm exactly one item survives — matter `2026-CH-050`, with `estimated_hours: 1` (matched from the actual start/end times).
2. Continue through **"Group Unbilled Activity By Attorney"** and confirm it groups under `attorney@smithlaw.com`.
3. Continue through **"Build Unbilled Time Digest Email"** and confirm the email lists the matter and estimated hours.

**Weekly branch:**
4. Run the equivalent weekly chain and confirm the same single unbilled item surfaces.
5. Continue through **"Compute Weekly Unbilled Revenue Impact"** and confirm `total_estimated_hours: 1, estimated_value: 250` (at the default $250/hr rate).
6. Continue through **"Build Weekly Unbilled Revenue Summary Email"** and confirm the message reads correctly.
7. Temporarily clear the pinned data on **"Fetch Weekly Calendar Entries From Clio"** to an empty `data: []`, re-run, and confirm the summary explicitly says there's nothing to flag rather than sending a zero-filled or broken message.

**Testing against your real setup:** unpin all Clio nodes, connect real credentials, and manually run each branch (Execute step — don't wait for the schedule). Confirm the `from_date`/`to_date` query parameters on the two "Fetch ... Time Entries From Clio" nodes actually filter as expected against a real response — this specific Clio endpoint's date-filtering behavior hasn't been independently confirmed live in this catalog yet, so check the raw output and adjust the parameter names if Clio's real API differs.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A matter has a calendar entry but no time entry that day | Flagged in the responsible attorney's daily digest |
| A matter has both a calendar entry and a time entry that day | Not flagged — assumed already billed |
| A calendar entry has no matter attached | Ignored — nothing to cross-reference it against |
| A calendar entry has no clear start/end time | Estimated at `DEFAULT_CALENDAR_DURATION_HOURS` instead of a real duration |
| A week passes with nothing unbilled | The weekly summary still sends, explicitly saying so |

---

## Compliance note

This template performs cross-referencing and record-keeping only — flagging a scheduling gap between calendar activity and logged time entries, never drafting a time-entry narrative or deciding what's billable (ABA Op. 512). It never writes to Clio's time-entry data. The Bar-Compliance Guardrail is used for internal audit logging only, since both emails go to the firm's own attorneys, not clients.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Firms bill only ~2.4 of 8 hours on average — this surfaces the gap in concrete matters and dollars, replacing LeanLaw and manual review |
| Compliance test | Cross-referencing and flagging only; no billing content decided or written automatically |
| Searchability test | Ranks for "find unbilled time law firm" |
| Deployability test | Reuses Clio credentials from other templates if already deployed — under 15 minutes |
| Upsell test | Clear Done-for-You path: productized as a fixed-scope build adding Communications (calls/emails) coverage and direct time-entry drafting for attorney approval |

---

## Want the advanced version?

Protomated builds the done-for-you **Done-for-You** implementation that adds:
- Logged calls and emails (Clio Communications) alongside calendar entries
- Draft time-entry narratives ready for one-click attorney approval, not just a flag
- Firm-wide realization-rate reporting beyond the weekly estimate
- Integration with practice management systems beyond Clio

[Book a call with Protomated](https://protomated.com/book) to get started.
