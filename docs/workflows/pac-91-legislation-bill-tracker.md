# Deploy Guide: Legislation / Bill Tracker

**Template:** PTPAC-91 — A1: Legislation / Bill Tracker
**Audience:** Internal Protomated tool (Authority pillar) — **not** a law firm template
**Replaces:** Bloomberg Government, FiscalNote

---

## This one is different from most of this catalog

This is an Authority-pillar template, which CLAUDE.md documents as content and SEO infrastructure — **not deployed to clients**. Staying current on bills in a practice area is manual and slow; this tracks bills against configured search terms via a real, freely available legislative API, alerts when something new appears or an existing bill's status changes, and feeds the newsletter/SEO content pipeline with a weekly landscape rollup.

It's still built to this catalog's usual bar (harness-tested code, pinned test data, sticky-note documentation, `active: false` until configured) because that discipline is worth keeping regardless of audience.

---

## Before you start: the most important thing to understand

**This monitors and summarizes public legislative information only — it is explicitly informational, never advice.** It never touches legal work, gives compliance guidance tailored to a specific firm's situation, or drafts anything that could be read as legal advice. The "why this matters" blurbs are templated based on keywords in the bill's own last-action text, not real legal analysis (ABA Op. 512 doesn't directly apply since this isn't law-firm-facing, but the same discipline is kept).

**This is built against the LegiScan API**, a real, freely available legislative tracking service covering all 50 states plus federal — you'll need a free API key from legiscan.com/legiscan. LegiScan's free tier has a daily query cap, so keep your configured search terms reasonably focused.

**LEGISCAN_API_KEY lives in a plain n8n Variable, not a Credential.** LegiScan's API takes its key as a query-string token, not OAuth, and there's no dedicated n8n credential type for that shape. Treat the variable with the same care as a secret even though it isn't stored as a credential.

**This has two independent parts on one canvas:**
1. **Daily search, comparison, and alert digest** — searches LegiScan, compares against a maintained tracker sheet, and alerts only on what's new or changed.
2. **Weekly legislation landscape rollup** — summarizes the full tracked bill list by practice area, for newsletter/content planning, not just what changed today.

**Alert-worthiness is determined by comparing against your own tracker sheet, not LegiScan's own change-detection.** A bill is flagged only if it's brand new to your tracker or its `last_action` text differs from what's already stored — everything else is silently updated (to keep `last_checked_at` current) without generating a fresh alert.

---

## What you need before you start

- A free LegiScan API key (legiscan.com/legiscan)
- A Google account with Sheets access
- An email account with SMTP access

---

## Step 1 — Create the tracker and drafts log (5 min)

Create a new Google Sheet with two tabs.

**Bill Tracker** — column headers in row 1:

```
bill_id | bill_number | title | state | practice_areas | last_status_code | last_action | last_action_date | last_checked_at
```

**Legislation Digest Drafts** — column headers in row 1:

```
digest_id | item_count | draft_content | status | created_at | reviewed_at | published_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`. Both tabs live in the same spreadsheet, so it's the same ID for both variables below.

---

## Step 2 — Import the workflow and add credentials (3 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-91-legislation-bill-tracker.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-91-legislation-bill-tracker.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `LEGISCAN_API_KEY` | Your free LegiScan API key |
| `LEGISLATION_SEARCH_TERMS` | Semicolon-separated `Area:term1,term2` pairs, e.g. `Employment Law:overtime,wage theft;Family Law:custody mediation` |
| `CONTENT_REVIEWER_EMAIL` | Who receives the daily alert digest and weekly rollup |
| `FIRM_FROM_EMAIL` | Sender address, and fallback recipient if `CONTENT_REVIEWER_EMAIL` isn't set |
| `BILL_TRACKER_SHEET_ID` | The Google Sheet ID from Step 1 |
| `LEGISLATION_LOG_SHEET_ID` | Same Google Sheet ID as above (both tabs live in one spreadsheet) |

---

## Step 4 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expressions on **"Check Tracked Legislation Daily"** (default 7am) and **"Check Weekly Legislation Landscape Rollup"** (default Monday 8am) if different times suit.

---

## Step 5 — Test

The workflow ships with a pinned LegiScan search response (one bill, "Introduced and referred to committee") and a pinned tracker row for a *different* bill already at "Introduced" status.

1. Run **"If Search Terms Configured"** and confirm it routes to the true branch as long as `LEGISLATION_SEARCH_TERMS` is set to something.
2. Continue through **"Build Legislation Search Queries"** and confirm one item per configured (area, term) pair.
3. Continue through **"Search LegiScan For Each Term"** → **"Parse Search Results"** and confirm the pinned bill (AB123) parses correctly, with the `summary` key correctly skipped.
4. Continue through **"Read Bill Tracker"** → **"Compare Against Bill Tracker"** and confirm AB123 is flagged `is_new: true, alert_worthy: true` (it isn't in the pinned tracker at all).
5. Edit the pinned tracker row's `bill_id` to `1234567` (matching AB123) with `last_action: "Introduced and referred to committee"` (matching the pinned search result exactly), re-run — confirm `is_new: false, status_changed: false, alert_worthy: false`.
6. Change that same tracker row's `last_action` to something different (e.g. `"Passed committee"`), re-run — confirm `status_changed: true, alert_worthy: true`.
7. Continue through **"Update Bill Tracker"** → **"Filter To Alert-Worthy Bills"** → **"Draft Legislation Alert Digest"** and confirm the digest includes AB123 under Employment Law with a "committee" blurb.
8. Temporarily clear `LEGISLATION_SEARCH_TERMS`, re-run from **"Check Tracked Legislation Daily"**, and confirm it routes to **"Draft Setup-Needed Notice"** and still emails a setup-needed message.

**Testing against your real setup:** unpin the HTTP and Sheets nodes, connect real credentials, and configure a real, narrow search term to confirm actual LegiScan results come through and compare correctly against your tracker.

**Weekly legislation landscape rollup branch:**
9. Run **"Check Weekly Legislation Landscape Rollup"** → **"Read Full Bill Tracker"** → **"Compute Legislation Landscape Summary"** and confirm the practice-area counts match what's in the sheet.
10. Continue through **"Build Legislation Landscape Email"** and confirm it reads correctly, including the "nothing tracked" case if the tracker is empty.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| `LEGISLATION_SEARCH_TERMS` isn't configured yet | The run still emails an explicit "setup needed" notice instead of silently producing nothing |
| A search returns a bill not yet in the tracker | Flagged `is_new`, included in today's alert digest, added to the tracker |
| A tracked bill's `last_action` text is unchanged since last check | Silently updates `last_checked_at` in the tracker, no alert |
| A tracked bill's `last_action` text has changed | Flagged `status_changed`, included in today's alert digest |
| The same bill matches more than one configured search term | Its practice areas are merged into a single alert entry, not duplicated |
| Nothing new or changed today | The digest still sends, explicitly saying so |
| `CONTENT_REVIEWER_EMAIL` isn't set | Both emails fall back to `FIRM_FROM_EMAIL` instead of failing with "No recipients defined" |

---

## Compliance note

This is Protomated's own internal content-sourcing tool — it never touches legal work or any law firm's systems. It monitors and summarizes public legislative information only, framed as informational, never advice (ABA Op. 512 doesn't directly apply here, but the same discipline is kept). Every digest requires a human review pass before publication.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces Bloomberg Government / FiscalNote-style subscriptions for this specific use case |
| Compliance test | Monitors and summarizes public legislative information only; explicitly informational, never advice |
| Searchability test | Feeds content that itself ranks for "state legislation tracker for law firms" |
| Deployability test | One free LegiScan API key plus standard Sheets/SMTP credentials — under 15 minutes |
| Upsell test | N/A — internal content tool |
