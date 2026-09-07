# Deploy Guide: Security & Compliance Content Digest

**Template:** PTPAC-84 — A4: Security & Compliance Content Digest
**Audience:** Internal Protomated tool (Authority pillar) — **not** a law firm template
**Replaces:** N/A — new authority content, not client-deployed

---

## This one is different from most of this catalog

This is an Authority-pillar template, which CLAUDE.md documents as content and SEO infrastructure — **not deployed to clients**. It exists to solve a content gap: solo and small firms rarely see plain-English security/compliance guidance written for their scale, and this is a piece of content Protomated can own to reinforce the "we take your data seriously" brand promise, feeding the newsletter/Field Notes pipeline.

It's still built to this catalog's usual bar (harness-tested code, pinned test data, sticky-note documentation, `active: false` until configured) because that discipline is worth keeping regardless of audience.

---

## Before you start: the most important thing to understand

**This monitors and summarizes public information only — it is explicitly informational, never advice.** It never touches legal work, gives compliance guidance tailored to a specific firm's situation, or drafts anything that could be read as legal or professional advice. The "why this matters" blurbs are templated based on which keyword matched, not real analysis or judgment.

**This never publishes anything on its own.** It only ever produces a draft digest and routes it to a human reviewer. Nothing goes into the newsletter/Field Notes pipeline without that review pass.

**Content quality depends entirely on the feeds you configure.** This template ships with no real feed URLs — you need to find and configure actual RSS feeds (state AG breach notification pages, ABA/state bar security guidance publications, IAPP or similar data-protection-law trackers) that publish RSS. Not every source that would be relevant here actually publishes an RSS feed; some research is needed to build a good feed list.

**This has two independent parts on one canvas:**
1. **Weekly digest draft** — pulls, filters, and drafts the digest, then emails it for review.
2. **Stale draft reminder** — separately checks the log for anything still sitting at `pending_review` past a configurable age and nudges the reviewer, so a draft doesn't quietly go stale between one week's digest and the next.

---

## What you need before you start

- A Google account with Sheets access
- An email account with SMTP access
- A list of real RSS feed URLs relevant to solo/small-firm security and compliance news

---

## Step 1 — Create the digest drafts log (3 min)

Create a new Google Sheet. Add a tab named exactly **Digest Drafts** with these column headers in row 1:

```
digest_id | item_count | draft_content | status | created_at | reviewed_at | published_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (3 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-84-security-compliance-content-digest.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-84-security-compliance-content-digest.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `SECURITY_DIGEST_FEED_URLS` | Comma-separated list of real RSS feed URLs to monitor |
| `CONTENT_REVIEWER_EMAIL` | Who receives the weekly digest draft for review |
| `FIRM_FROM_EMAIL` | Sender address for the review request email |
| `DIGEST_LOG_SHEET_ID` | The Google Sheet ID from Step 1 |
| `RELEVANCE_KEYWORDS` *(optional)* | Comma-separated relevance filter — defaults to `small firm,solo firm,law firm,breach,ransomware,data protection,compliance,privacy law` |
| `DIGEST_LOOKBACK_DAYS` *(optional)* | How far back to consider an item "recent" — defaults to 7 |
| `STALE_DRAFT_REMINDER_DAYS` *(optional)* | How many days a draft can sit at `pending_review` before the reminder nudges the reviewer — defaults to 5 |

---

## Step 4 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expression on **"Check Security Content Sources Weekly"** (default Monday 8am) and **"Check For Stale Digest Drafts Weekly"** (default Thursday 9am) if different times suit.

---

## Step 5 — Test

The workflow ships with pinned sample feed data: two genuinely relevant items (a breach notification guidance update, a ransomware trend report) and one irrelevant item (a bar association golf tournament writeup), simulating a mixed real-world feed.

1. Run **"If Feed URLs Configured"** and confirm it routes to the true branch (**"Build Feed URL List"**) as long as `SECURITY_DIGEST_FEED_URLS` is set to something.
2. Continue through **"Build Feed URL List"** → **"Fetch Each Security Feed"** → **"Filter Relevant Recent Items"** and confirm exactly 2 of the 3 pinned items survive — the golf tournament item should be filtered out.
3. Continue through **"Draft Plain-English Digest"** and confirm `item_count: 2` and both surviving items appear in `draft_content` with a "why this matters" blurb each.
4. Continue through **"Assign Digest ID"** → **"Log Digest Draft For Review"** and confirm a new row would be logged.
5. Continue through **"Build Digest Review Request Email"** and confirm the message reads correctly.
6. Temporarily clear the pinned data on **"Fetch Each Security Feed"** to an empty array, re-run, and confirm the digest still logs and sends with `(no relevant items found this week)` rather than breaking or going silent.
7. Temporarily clear `SECURITY_DIGEST_FEED_URLS` (or leave it unset before you configure Step 3), re-run from **"Check Security Content Sources Weekly"**, and confirm it routes to **"Draft Setup-Needed Notice"** and still logs and emails a `(no feed URLs configured yet...)` message rather than the run silently producing nothing.

**Testing against your real setup:** unpin the RSS and Sheets nodes, connect real credentials, and configure at least one real feed URL to confirm actual items come through and get filtered as expected.

**Stale draft reminder branch:**
8. Run **"Check For Stale Digest Drafts Weekly"** → **"Read Digest Drafts Log"** and confirm the 2 pinned rows come through (one `pending_review` logged 13 days ago, one already `approved`).
9. Continue through **"Find Stale Pending Drafts"** and confirm `stale_count: 1` — only the still-pending, 13-day-old draft counts; the approved one doesn't, regardless of age.
10. Continue through **"If Any Stale Drafts Found"** → **"Build Stale Draft Reminder Email"** and confirm the message lists that one draft.
11. Edit the pinned `pending_review` row's `created_at` to today, re-run, and confirm `stale_count: 0` and the run routes to **"Skip — No Stale Drafts"** instead of sending a reminder.

**If a send ever fails with "No recipients defined":** it means neither `CONTENT_REVIEWER_EMAIL` nor `FIRM_FROM_EMAIL` is set — both review emails fall back to `FIRM_FROM_EMAIL` if `CONTENT_REVIEWER_EMAIL` isn't configured, but at least one of the two must be set in n8n Variables (Settings → Variables) for any send to succeed.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| `SECURITY_DIGEST_FEED_URLS` isn't configured yet | The run still logs and emails an explicit "setup needed" notice instead of silently producing nothing |
| A feed item mentions a relevance keyword and was published recently | Included in the digest with a templated "why this matters" blurb |
| A feed item is old (outside the lookback window) or irrelevant | Excluded |
| No relevant items found across any feed this week | The digest still sends, explicitly saying so |
| The digest draft is logged | Status starts `pending_review` — publishing to the newsletter pipeline is always a manual step afterward |
| `CONTENT_REVIEWER_EMAIL` isn't set | Both review emails fall back to `FIRM_FROM_EMAIL` instead of failing with "No recipients defined" |
| A draft sits at `pending_review` past `STALE_DRAFT_REMINDER_DAYS` | The reviewer gets a separate reminder email listing every stale draft |
| No drafts are currently stale | No reminder is sent — the run ends quietly at "Skip — No Stale Drafts" |

---

## Compliance note

This is Protomated's own internal content-sourcing tool — it never touches legal work or any law firm's systems. It monitors and summarizes public information only, framed as informational, never advice (ABA Op. 512 doesn't directly apply here, but the same discipline is kept). Every digest requires a human review pass before publication.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | N/A — new authority content, not a revenue-recovery template |
| Compliance test | Monitors and summarizes public information only; explicitly informational, never advice |
| Searchability test | Feeds content that itself ranks for "law firm data security basics" |
| Deployability test | Two Sheets/SMTP credentials plus real feed URLs — under 15 minutes for whoever maintains Protomated's own content pipeline |
| Upsell test | N/A — internal content tool |
