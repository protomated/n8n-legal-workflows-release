# Deploy Guide: Immigration Case Status Tracker

**Template:** PTPAC-101 — Immigration Case Status Tracker and Client Update Automation
**Replaces:** Manually checking USCIS case status and manually emailing clients about it

Get value in under 15 minutes. Immigration clients check in constantly because USCIS timelines are opaque, and manually checking status for dozens of pending cases eats real staff time. This workflow polls USCIS's public case-status lookup daily, emails the client a plain-English update the moment something changes, and separately tracks Request for Evidence deadlines so one never gets missed.

---

## Before you start: the most important thing to understand

**Verify the USCIS endpoint and status-parsing pattern live before you activate this.** USCIS's public case-status page (`egov.uscis.gov/casestatus/mycasestatus.do`) is not a documented, versioned API — it's a page meant for a human to read, and its markup can change without notice. Submit a real receipt number through it yourself, look at the actual HTML that comes back, and confirm `USCIS_STATUS_REGEX` still extracts the status text correctly. Do this before connecting real clients — an unverified regex either silently reports nothing or, worse, reports the wrong thing.

**This only ever reports status — it never interprets what a status means, predicts an outcome, or gives legal advice** (ABA Op. 512). The RFE deadline is a fixed response window calculated from your own tracker dates, not a legal determination of the actual filing deadline on the RFE letter itself — always confirm the actual deadline on the letter.

**This has two independent parts on one canvas:**
1. **Daily status poll** — checks every open case's USCIS status and emails the client only when something changed, through the compliance guardrail.
2. **RFE deadline alerts** — separately watches for any unresolved Request for Evidence whose response deadline is approaching, and alerts staff (never the client) directly.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed
- A Google account with Sheets access
- An email account with SMTP access
- A way of adding new cases to the tracker sheet as clients come in (manually, or from another intake workflow)

---

## Step 1 — Create the case tracker sheet (3 min)

Create a new Google Sheet. Add a tab named exactly **Case Tracker** with these column headers in row 1:

```
receipt_number | client_name | client_email | case_type | matter_id | last_status | last_status_at | rfe_issued_at | rfe_deadline | rfe_resolved | closed | last_checked_at
```

Add one row per case you're tracking. New cases should start with `last_status` blank or set to whatever the client was told at filing, and `closed`/`rfe_resolved` set to `FALSE`.

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Verify the USCIS lookup live (5 min)

1. Go to `https://egov.uscis.gov/casestatus/mycasestatus.do` in a browser and submit a real receipt number you have permission to check. Confirm it still works and note the page structure around the status heading.
2. If the markup differs from a `<h1>...</h1>` status heading, set `USCIS_STATUS_REGEX` (Step 4) to a regex that matches what you actually see, with the status text captured in group 1.

---

## Step 3 — Import the workflow and add credentials (3 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-101-immigration-case-status-tracker.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-101-immigration-case-status-tracker.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.

No credential is needed for the USCIS lookup itself — it's a public, unauthenticated page.

---

## Step 4 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | Sender address for all case-update and alert emails |
| `FIRM_ATTORNEY_EMAIL` | Who receives RFE deadline alerts |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `IMMIGRATION_CASE_TRACKER_SHEET_ID` | The Google Sheet ID from Step 1 |
| `USCIS_CASE_STATUS_ENDPOINT` *(optional)* | Defaults to the public USCIS case-status URL — override only if you're using a different lookup source |
| `USCIS_STATUS_REGEX` *(optional)* | Only set this if Step 2 showed the page structure differs from the default `<h1>...</h1>` pattern |
| `RFE_RESPONSE_WINDOW_DAYS` *(optional)* | Defaults to 87, the standard RFE response window |
| `RFE_ALERT_LEAD_DAYS` *(optional)* | How many days before the deadline staff get alerted — defaults to 10 |

---

## Step 5 — Activate (1 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expression on **"Poll USCIS Case Status Daily"** (default 6am) if a different time suits.

---

## Step 6 — Test

The workflow ships with three pinned tracker rows (one pending, one with an RFE deadline 9 days out, one already closed) and a pinned sample USCIS response showing "Case Was Approved".

1. Run **"Read Case Tracker Sheet"** → **"Filter To Active Cases"** and confirm the closed case (Priya Nair) is excluded, leaving 2 active cases.
2. Continue through **"Poll USCIS Case Status"** → **"Parse USCIS Status From Response"** and confirm `current_status_text: "Case Was Approved"` with `lookup_succeeded: true` for both.
3. Continue through **"Determine Status Change And RFE"** and confirm the first case (`last_status: "Case Was Received"`) shows `status_changed: true`, and the second (`last_status: "Request For Evidence Was Sent"`, now resolving to "Case Was Approved") shows `status_changed: true` and `rfe_resolved: true`.
4. Continue through **"If Status Changed"** → **"Build Plain-English Client Status Update Email"** and confirm the copy reports the status without interpreting it.
5. Continue through the compliance check and **"Update Case Tracker Row"** and confirm both rows would update correctly.
6. Separately, run **"Read Case Tracker Sheet"** → **"Compute Approaching RFE Deadlines"** and confirm Diego Ramirez's row (9 days from the deadline) is flagged, while Amara Chen's row (no RFE) is not.
7. Continue through **"Build RFE Deadline Alert Email"** and confirm it's addressed to `FIRM_ATTORNEY_EMAIL` with the correct day count.

**Testing against your real setup:** unpin every Sheets- and HTTP-facing node, connect real credentials, and run one real case through with a receipt number you have permission to check.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A case's USCIS status hasn't changed | Silent — the row's `last_checked_at` updates, nothing else does |
| A case's status changes | The client gets a plain-English update through the compliance guardrail; the row updates |
| A case enters Request for Evidence for the first time | `rfe_deadline` is set (87 days out by default); nothing else changes yet |
| An RFE deadline is within the alert window | Staff get a direct alert — this never goes to the client |
| A case moves past its RFE | `rfe_resolved` is set so it stops appearing in deadline alerts |
| The USCIS lookup fails or the page structure has changed | The row's `last_checked_at` still updates so you can see it was attempted, but `last_status` is left untouched rather than being overwritten with a bad parse |
| A client has opted out via the guardrail | The status update email is suppressed; the row still updates |

---

## Compliance note

This template performs status polling, plain-English reporting, and deadline tracking only — it never interprets what a USCIS status means for the case, predicts an outcome, or gives legal advice (ABA Op. 512). RFE deadlines are calculated from a configurable, fixed response window applied to your own tracker dates — always confirm the actual deadline stated on the RFE letter itself.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces manually checking USCIS status and manually emailing clients about it — a major time sink at higher immigration caseloads |
| Compliance test | Status reporting and deadline tracking only; no interpretation, no outcome prediction, no legal advice |
| Searchability test | Ranks for "USCIS case status automation for law firms" |
| Deployability test | No paid API required — under 15 minutes, plus a one-time live verification step |
| Upsell test | Clear Done-for-You path: RFE response tracking, removal-proceedings scheduling, multi-form sequencing, INSZoom/Docketwise integration |

---

## Want the advanced version?

Protomated builds the done-for-you **Done-for-You** implementation that adds:
- Full RFE response tracking and drafting workflow integration
- Removal-proceedings scheduling
- Multi-form filing sequences
- INSZoom / Docketwise integration in place of a standalone tracker sheet

[Book a call with Protomated](https://protomated.com/book) to get started.
