# Deploy Guide: Regulatory & Agency Alert Monitor

**Template:** PTPAC-90 — A2: Regulatory & Agency Alert Monitor
**Audience:** Internal Protomated tool (Authority pillar) — **not** a law firm template
**Replaces:** Manual monitoring, paid regulatory feed subscriptions

---

## This one is different from most of this catalog

This is an Authority-pillar template, which CLAUDE.md documents as content and SEO infrastructure — **not deployed to clients**. Relevant agency and bar updates are scattered across dozens of sources and easy to miss; this turns that into a weekly digest that also informs BD conversations (knowing which practice areas a new regulation touches is useful beyond content).

It's still built to this catalog's usual bar (harness-tested code, pinned test data, sticky-note documentation, `active: false` until configured) because that discipline is worth keeping regardless of audience.

---

## Before you start: the most important thing to understand

**This monitors and summarizes public information only — it is explicitly informational, never advice.** It never touches legal work, gives compliance guidance tailored to a specific firm's situation, or drafts anything that could be read as legal or professional advice. The "why this matters" framing is templated based on which keyword matched, not real analysis.

**The differentiator versus a generic content digest is practice-area tagging.** Every item is checked against a configurable map of practice areas to keywords — an item can affect more than one practice area (a single new rule might touch both Employment Law and general compliance), or none, in which case it's dropped. The digest groups items by which areas they affect, not just a flat list.

**This never publishes anything on its own.** It only ever produces a draft digest and routes it to a human reviewer. Nothing goes into the newsletter/Field Notes pipeline without that review pass.

**Content quality depends entirely on the feeds and keyword map you configure.** This template ships with no real feed URLs — you need to find and configure actual RSS feeds (state bar publications, agency press releases, regulator feeds) that publish RSS, and tune the practice-area keyword map to match what you actually want tagged.

**This has two independent parts on one canvas:**
1. **Weekly digest draft** — pulls, tags, and drafts the digest, then emails it for review.
2. **Stale draft reminder** — separately checks the log for anything still sitting at `pending_review` past a configurable age and nudges the reviewer.

---

## What you need before you start

- A Google account with Sheets access
- An email account with SMTP access
- A list of real RSS feed URLs relevant to agency/bar updates
- A practice-area keyword map for tagging (you can start from the built-in default and tune it)

---

## Step 1 — Create the digest drafts log (3 min)

Create a new Google Sheet. Add a tab named exactly **Regulatory Digest Drafts** with these column headers in row 1:

```
digest_id | item_count | draft_content | status | created_at | reviewed_at | published_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (3 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-90-regulatory-agency-alert-monitor.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-90-regulatory-agency-alert-monitor.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `AGENCY_BAR_FEED_URLS` | Comma-separated list of real RSS feed URLs to monitor |
| `PRACTICE_AREA_KEYWORDS` | Semicolon-separated `Area:keyword1,keyword2` pairs, e.g. `Employment Law:overtime,wage,discrimination,EEOC;Family Law:custody,child support,divorce;Immigration:USCIS,visa,asylum` — defaults to that exact example if unset |
| `CONTENT_REVIEWER_EMAIL` | Who receives the weekly digest draft for review |
| `FIRM_FROM_EMAIL` | Sender address, and fallback recipient if `CONTENT_REVIEWER_EMAIL` isn't set |
| `REGULATORY_LOG_SHEET_ID` | The Google Sheet ID from Step 1 |
| `DIGEST_LOOKBACK_DAYS` *(optional)* | How far back to consider an item "recent" — defaults to 7 |
| `STALE_DRAFT_REMINDER_DAYS` *(optional)* | How many days a draft can sit at `pending_review` before the reminder nudges the reviewer — defaults to 5 |

---

## Step 4 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expressions on **"Check Regulatory Feeds Weekly"** (default Monday 8am) and **"Check For Stale Regulatory Drafts Weekly"** (default Thursday 9am) if different times suit.

---

## Step 5 — Test

The workflow ships with pinned sample feed data: an overtime-rule item (matches Employment Law), a custody-mediation item (matches Family Law), and an irrelevant golf-tournament item, simulating a mixed real-world feed.

1. Run **"If Feed URLs Configured"** and confirm it routes to the true branch as long as `AGENCY_BAR_FEED_URLS` is set to something.
2. Continue through **"Build Feed URL List"** → **"Fetch Each Agency Bar Feed"** → **"Tag Items By Practice Area Impact"** and confirm exactly 2 of the 3 pinned items survive, tagged `Employment Law` and `Family Law` respectively — the golf item is dropped since it matches no configured keyword.
3. Continue through **"Draft Regulatory Digest"** and confirm `item_count: 2, area_count: 2` and both items appear under their respective area headers in `draft_content`.
4. Continue through **"Assign Digest ID"** → **"Log Digest Draft For Review"** and confirm a new row would be logged.
5. Continue through **"Build Digest Review Request Email"** and confirm the message groups items by practice area.
6. Temporarily clear the pinned data on **"Fetch Each Agency Bar Feed"** to an empty array, re-run, and confirm the digest still logs and sends with `(no items matched any configured practice area this week)` rather than breaking or going silent.
7. Temporarily clear `AGENCY_BAR_FEED_URLS`, re-run from **"Check Regulatory Feeds Weekly"**, and confirm it routes to **"Draft Setup-Needed Notice"** and still logs and emails a setup-needed message.

**Testing against your real setup:** unpin the RSS and Sheets nodes, connect real credentials, and configure at least one real feed URL to confirm actual items come through and get tagged as expected.

**Stale draft reminder branch:**
8. Run **"Check For Stale Regulatory Drafts Weekly"** → **"Read Regulatory Digest Drafts Log"** → **"Find Stale Pending Regulatory Drafts"** and confirm only the still-pending, old draft counts as stale — an approved draft never counts, regardless of age.
9. Continue through **"If Any Stale Regulatory Drafts Found"** → **"Build Stale Regulatory Draft Reminder Email"** and confirm the message lists the stale draft(s).

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| `AGENCY_BAR_FEED_URLS` isn't configured yet | The run still logs and emails an explicit "setup needed" notice instead of silently producing nothing |
| A feed item matches one or more configured practice areas and was published recently | Included in the digest, grouped under each matched area |
| A feed item matches no configured practice area | Dropped |
| No items matched anything this week | The digest still sends, explicitly saying so |
| `CONTENT_REVIEWER_EMAIL` isn't set | The review email falls back to `FIRM_FROM_EMAIL` instead of failing with "No recipients defined" |
| A draft sits at `pending_review` past `STALE_DRAFT_REMINDER_DAYS` | The reviewer gets a separate reminder email listing every stale draft |

---

## Compliance note

This is Protomated's own internal content-sourcing tool — it never touches legal work or any law firm's systems. It monitors and summarizes public information only, framed as informational, never advice (ABA Op. 512 doesn't directly apply here, but the same discipline is kept). Every digest requires a human review pass before publication.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | N/A — new authority content, not a revenue-recovery template |
| Compliance test | Monitors and summarizes public information only; explicitly informational, never advice |
| Searchability test | Feeds content that itself ranks for "regulatory alert monitoring law firm" |
| Deployability test | Two Sheets/SMTP credentials plus real feed URLs and a keyword map — under 15 minutes for whoever maintains Protomated's own content pipeline |
| Upsell test | N/A — internal content tool |
