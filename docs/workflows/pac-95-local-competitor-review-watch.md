# Deploy Guide: Local Competitor & Review Watch

**Template:** PTPAC-95 — G6: Local Competitor & Review Watch
**Pillar:** Get
**Replaces:** SEO and ad-intel tools (for review/rating tracking specifically)

Get value in under 15 minutes. Firms have no view of how competitors are moving in their metro. This workflow tracks each configured competitor's Google review count and rating every week, and tells you what changed.

---

## Before you start: what this does and doesn't track

**This tracks review count and rating only — it does not track search ranking or ad presence.** Both require paid SEO/ad-intelligence APIs (SEMrush, Ahrefs, SpyFu, or similar) that aren't free or generically available the way Google's Places API is. If ranking or ad tracking matters to a firm, that's a Done-for-You conversation, not something this free template can honestly promise.

**This monitors public review data only.** It never contacts a competitor, scrapes anything beyond Google's own public API, or makes any legal or ethical judgment about a competitor's marketing (ABA Op. 512).

**This inherits the Bar-Compliance Guardrail (OPS1), even though every recipient is firm staff, not a client.** The ticket for this template specifically calls for that — the opt-out/disclaimer checks are effectively inert for an internal marketing recipient, but the audit log entry keeps this consistent with the rest of the catalog.

**This has two independent parts on one canvas:**
1. **Weekly competitor check** — fetches each competitor's current review count/rating, compares against last week, and digests what moved.
2. **Monthly competitor trend rollup** — reports the longer-horizon trend per competitor over the trailing month, not just week-over-week movement.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed
- A Google Cloud project with the **Places API** enabled and billing configured (Google provides a monthly free credit that covers light use of this template)
- A Google account with Sheets access
- An email account with SMTP access

---

## Step 1 — Find each competitor's Google Place ID (10–15 min, one-time)

For each competitor firm you want to track, find their Google Place ID. Search "Google Place ID Finder" for Google's own lookup tool, or use the Places API's Find Place request with the competitor's business name. Note each competitor's name and Place ID.

---

## Step 2 — Create the competitor watch log (3 min)

Create a new Google Sheet. Add a tab named exactly **Competitor Watch Log** with these column headers in row 1:

```
place_id | firm_name | rating | review_count | checked_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 3 — Import the workflow and add credentials (3 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-95-local-competitor-review-watch.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-95-local-competitor-review-watch.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 4 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `GOOGLE_PLACES_API_KEY` | Your Google Cloud API key with the Places API enabled |
| `COMPETITOR_PLACE_IDS` | Semicolon-separated `Firm Name|Place ID` pairs from Step 1, e.g. `Rival Law Group|ChIJ...;Second Rival|ChIJ...` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | Sender address, and fallback recipient if `FIRM_MARKETING_EMAIL` isn't set |
| `FIRM_MARKETING_EMAIL` | Who receives the weekly digest and monthly trend rollup |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `COMPETITOR_LOG_SHEET_ID` | The Google Sheet ID from Step 2 |
| `COMPETITOR_TREND_LOOKBACK_DAYS` *(optional)* | How many trailing days the monthly rollup covers — defaults to 30 |

---

## Step 5 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expressions on **"Check Competitor Watch Weekly"** (default Monday 8am) and **"Check Monthly Competitor Trend"** (default the 1st of the month, 8am) if different times suit.

---

## Step 6 — Test

The workflow ships with one pinned competitor (Rival Law Group, currently 4.6★/132 reviews) and a pinned prior snapshot (4.5★/125 reviews from a week earlier).

1. Run **"If Competitors Configured"** and confirm it routes to the true branch as long as `COMPETITOR_PLACE_IDS` is set to something.
2. Continue through **"Build Competitor List"** → **"Fetch Competitor Place Details"** → **"Parse Place Details Response"** and confirm the pinned rating/review count come through correctly.
3. Continue through **"Read Last Competitor Snapshot"** → **"Compute Competitor Movement"** and confirm `review_count_change: 7` and `rating_change: 0.1`.
4. Continue through **"Draft Competitor Watch Digest"** and confirm the digest line reads "+7 new review(s) — rating up 0.1".
5. Edit the pinned Places API response to `{"status": "NOT_FOUND"}`, re-run from **"Parse Place Details Response"**, and confirm the competitor is marked `fetch_failed: true` and shows up in the digest as "could not fetch" rather than crashing the run.
6. Temporarily clear `COMPETITOR_PLACE_IDS`, re-run from **"Check Competitor Watch Weekly"**, and confirm it routes to **"Draft Setup-Needed Notice"** and still emails a setup-needed message.

**Testing against your real setup:** unpin the HTTP and Sheets nodes, connect real credentials, and confirm your Places API key and Place IDs actually resolve to real competitor data.

**Monthly competitor trend branch:**
7. Run **"Check Monthly Competitor Trend"** → **"Read Full Competitor Log"** → **"Compute Monthly Competitor Trend"** and confirm the trend uses the oldest-vs-newest snapshot within the trailing 30 days (the pinned data includes an older snapshot outside that window, which should be excluded from the trend calculation).
8. Continue through **"Build Monthly Trend Email"** and confirm it reads correctly, including the "no snapshots yet" case if the log is empty.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| `COMPETITOR_PLACE_IDS` isn't configured yet | The run still emails an explicit "setup needed" notice instead of silently producing nothing |
| A competitor gains reviews since last check | Digest notes the increase and, if applicable, the rating change |
| A competitor's Place ID is invalid or the API errors | That competitor is flagged "could not fetch" in the digest rather than crashing the whole run |
| A competitor is tracked for the first time | Reports zero movement rather than a misleading comparison against nothing |
| `FIRM_MARKETING_EMAIL` isn't set | Both emails fall back to `FIRM_FROM_EMAIL` instead of failing with "No recipients defined" |

---

## Compliance note

This template monitors public Google review data only — it never contacts a competitor or makes any legal/ethical judgment about their marketing (ABA Op. 512). The Bar-Compliance Guardrail is used on the weekly digest per this template's ticket, even though the recipient is firm staff rather than a client.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Gives a firm visibility into competitor review/rating movement they'd otherwise pay an SEO/ad-intel tool for |
| Compliance test | Public review data monitoring only; no contact with competitors, no legal/ethical judgment made |
| Searchability test | Ranks for "law firm competitor monitoring" |
| Deployability test | One Google Cloud API key plus standard Sheets/SMTP credentials — under 15 minutes once Place IDs are found |
| Upsell test | Clear Done-for-You path: search ranking and ad-presence tracking via paid SEO/ad-intelligence tools this free template can't include |
