# Deploy Guide: Local Legal-Market Signal Scanner

**Template:** PTPAC-89 — A3: Local Legal-Market Signal Scanner
**Audience:** Internal Protomated tool (Authority pillar) — **not** a law firm template
**Replaces:** Manual BD research, paid data vendors

---

## This one is different from most of this catalog

This is an Authority-pillar template, which CLAUDE.md documents as content and SEO infrastructure — **not deployed to clients**. It exists to power Protomated's own BD pipeline: demand signals for a practice area (a new business license, a building permit, an entity filing) are public but never systematically collected, so this watches a configured open-data feed and tags each new record against the practice areas it might create demand for.

It's still built to this catalog's usual bar (harness-tested code, pinned test data, sticky-note documentation, `active: false` until configured) because that discipline is worth keeping regardless of audience.

---

## Before you start: there is no single canonical API for this

Business filing, permit, and license data is scattered across thousands of separate city/county/state open-data portals — there's no unified national API. This template is built against the **Socrata SODA API** pattern, which is genuinely the most common standard (NYC Open Data, Chicago Data Portal, Austin Open Data, and hundreds of other city/county/state portals all run on it) — but you need to:

1. Find a real Socrata-based dataset for your target market's business filings, permits, or licenses.
2. Read that dataset's own API documentation (usually an "API" or "Export" tab on the dataset page) to find its actual column names.
3. Configure this template's field-name variables to match — every dataset names its columns differently.

If your target jurisdiction's open-data portal isn't built on Socrata, the HTTP call in **"Fetch New Signal Records"** will need adjusting to that portal's own API shape — the rest of the workflow (normalization, dedupe, tagging, digesting) is portal-agnostic once you're feeding it records.

---

## Before you start: the most important thing to understand

**This tags and surfaces public records only — it never assesses, scores, or contacts a specific business or individual.** A signal is a keyword match between a public filing's type/name and a practice area's configured keyword list, nothing more. It's a research head start for BD, not a judgment about the business itself.

**This has two independent parts on one canvas:**
1. **Daily signal scan** — queries the configured data source, dedupes, tags, logs, and digests.
2. **Weekly signal volume rollup** — reports the trailing week's signal mix by practice area and top signal types, so patterns are visible beyond one day at a time.

---

## What you need before you start

- A real Socrata-based (or compatible) open-data dataset URL for your target market
- A Google account with Sheets access
- An email account with SMTP access

---

## Step 1 — Find your dataset and its field names (15–30 min, one-time)

This is the one genuinely involved step. Search `[your target city/county/state] open data business license` or `... building permits` — most major metros publish one. Once found:

1. Open the dataset's **API** tab (Socrata portals expose this directly) and copy the API endpoint URL — it looks like `https://data.example.gov/resource/abcd-1234.json`.
2. Note the exact column names for: a unique record ID, the filing/issue date, the business or applicant name, the license/permit type, and (optionally) the address and a direct record link.

---

## Step 2 — Create the signal log (3 min)

Create a new Google Sheet. Add a tab named exactly **Signal Log** with these column headers in row 1:

```
signal_id | business_name | signal_type | practice_areas | address | signal_date | logged_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 3 — Import the workflow and add credentials (3 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-89-local-legal-market-signal-scanner.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-89-local-legal-market-signal-scanner.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 4 — Set n8n Variables (10 min)

| Variable | What to enter |
|---|---|
| `SIGNAL_DATA_API_URL` | The dataset API endpoint from Step 1 |
| `SIGNAL_DATE_FIELD` | The dataset's date column name, e.g. `issue_date` |
| `SIGNAL_ID_FIELD` | The dataset's unique record ID column name |
| `SIGNAL_TYPE_FIELD` | The dataset's license/permit type column name |
| `SIGNAL_NAME_FIELD` | The dataset's business/applicant name column name |
| `SIGNAL_KEYWORDS_BY_PRACTICE_AREA` | Semicolon-separated `Area:keyword1,keyword2` pairs — defaults to `Business Formation:llc,corporation,new business,incorporation;Employment Law:staffing,employment agency,payroll;Real Estate:construction,building permit,renovation,commercial lease` if unset |
| `BD_REVIEWER_EMAIL` | Who receives the daily digest and weekly rollup |
| `FIRM_FROM_EMAIL` | Sender address, and fallback recipient if `BD_REVIEWER_EMAIL` isn't set |
| `SIGNAL_LOG_SHEET_ID` | The Google Sheet ID from Step 2 |
| `SIGNAL_ADDRESS_FIELD` *(optional)* | The dataset's address column name |
| `SIGNAL_LINK_FIELD` *(optional)* | The dataset's own record-URL column name, if it has one |
| `SIGNAL_DATA_APP_TOKEN` *(optional)* | An App Token from the data portal's developer settings, to avoid throttling on a daily poll |
| `SIGNAL_LOOKBACK_DAYS` *(optional)* | How many days back the daily query looks — defaults to 1 |
| `SIGNAL_SUMMARY_LOOKBACK_DAYS` *(optional)* | How many trailing days the weekly rollup covers — defaults to 7 |

---

## Step 5 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expressions on **"Check Signal Sources Daily"** (default 7am) and **"Check Weekly Signal Volume Rollup"** (default Monday 8am) if different times suit.

---

## Step 6 — Test

The workflow ships with pinned sample records simulating a mixed daily feed: a general contractor license (matches Business Formation via "llc" in the business name), an employment agency license (matches Employment Law), and a retail food license (matches nothing configured, dropped).

1. Run **"If Data Source Configured"** and confirm it routes to the true branch as long as `SIGNAL_DATA_API_URL` is set to something.
2. Continue through **"Build Signal Query URL"** and confirm the URL includes a `$where` clause using your configured `SIGNAL_DATE_FIELD`.
3. Continue through **"Fetch New Signal Records"** → **"Normalize Signal Records"** and confirm the 3 pinned records map to `signal_id`, `business_name`, `signal_type`, and `address` correctly.
4. Continue through **"Dedupe Against Previously Seen Signals"** and confirm all 3 pass through on a fresh run. Re-run the same input a second time (without clearing static data) and confirm all 3 are now dropped as already-seen.
5. Continue through **"Tag Signals By Practice Area"** and confirm exactly 2 of the 3 records are tagged — the retail food license is dropped since it matches no configured keyword.
6. Continue through **"Log New Signals For Tracking"** → **"Draft Signal Digest"** and confirm `signal_count: 2, area_count: 2` and both signals appear under their respective area headers.
7. Temporarily clear `SIGNAL_DATA_API_URL`, re-run from **"Check Signal Sources Daily"**, and confirm it routes to **"Draft Setup-Needed Notice"** and still emails a setup-needed message.

**Testing against your real setup:** unpin the HTTP and Sheets nodes, connect real credentials, and point `SIGNAL_DATA_API_URL` at your actual chosen dataset to confirm real records come through and normalize correctly.

**Weekly signal volume rollup branch:**
8. Run **"Check Weekly Signal Volume Rollup"** → **"Read Weekly Signal Log"** → **"Compute Weekly Signal Volume Summary"** and confirm the practice-area and signal-type counts match what's in the sheet within the lookback window.
9. Continue through **"Build Weekly Signal Volume Email"** and confirm it reads correctly, including the "no signals this week" case if you clear the pinned data.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| `SIGNAL_DATA_API_URL` isn't configured yet | The run still emails an explicit "setup needed" notice instead of silently producing nothing |
| A new record matches one or more configured practice-area keywords | Logged and included in the daily digest, grouped under each matched area |
| A new record matches no configured keyword | Dropped, not logged |
| A record was already seen on a prior run | Dropped before tagging — dedupe uses n8n workflow static data, not a database |
| No new signals matched anything today | The digest still sends, explicitly saying so |
| `BD_REVIEWER_EMAIL` isn't set | Both emails fall back to `FIRM_FROM_EMAIL` instead of failing with "No recipients defined" |

---

## Compliance note

This is Protomated's own internal BD tool — it never touches legal work or any law firm's systems. It tags and surfaces public records only, never an assessment of a specific business or individual (ABA Op. 512 doesn't directly apply here, but the same discipline is kept). It never contacts anyone on the firm's behalf.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces manual BD research and paid data vendors (Bloomberg-style, permit-tracking services) for this specific use case |
| Compliance test | Tags public records against configured keywords only; no assessment of any business or individual, nothing sent to anyone outside Protomated |
| Searchability test | Feeds authority content that ranks for "new business signal scanner for law firms" |
| Deployability test | Finding and mapping a real dataset is the one real setup step (15–30 min); everything after that is standard Sheets/SMTP credentials |
| Upsell test | N/A — internal BD tool |
