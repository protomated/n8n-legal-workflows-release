# Deploy Guide: Google Business Profile Auto-Poster

**Template:** PTPAC-60 — G3: Google Business Profile Auto-Poster
**Pillar:** Get
**Replaces:** A marketing retainer's local-SEO upkeep

Get value in under 15 minutes. Once a week, the workflow pulls the next item from a content queue you maintain, turns it into a ready-to-paste Google Business Profile post with Claude, safety-checks it twice, and emails it straight to staff — nobody has to sit down and write a post from scratch every week, just paste and click publish. 74% of prospective clients start with a search; a profile that hasn't posted in months signals a firm that isn't actively practicing.

---

## Before you start: heads up on this one

**This does not auto-publish to Google Business Profile.** Google restricts direct posting access (the Business Profile / My Business API) to developers it specifically approves, and in practice even bridging through Zapier's own partner-level access isn't reliable — this catalog's own Negative-Review Rapid-Response template (PAC-50) has hit that exact wall waiting on Zapier's account approval. Rather than ship something that might silently stop working depending on a firm's Google account status, this template keeps a human as the one who clicks "Publish" — you get a finished, ready-to-paste post in your inbox every week instead of a fully autonomous poster. See "Want the advanced version?" below if full auto-publishing becomes reliably available later.

**Nothing reaches your inbox without passing two independent safety checks.** Claude self-reports whether the content it generated is safe, and a separate keyword scan re-checks the output afterward — same two-layer approach as the Content Repurposer (PAC-31) and Negative-Review Rapid-Response (PAC-50). If either layer catches anything legal-sounding, nothing is drafted; the item is held for a human to fix instead.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** This template calls it to log every draft (and every flagged exception) to your compliance audit sheet.
- An n8n instance (self-hosted or n8n Cloud)
- A Google Sheet you'll use as the content queue
- A Google Business Profile for your firm, claimed and verified (business.google.com)
- An [Anthropic API key](https://console.anthropic.com) — reuse the one from PAC-31/PAC-50 if already deployed
- An email account with SMTP access

---

## Step 1 — Create the Content Queue sheet (3 min)

1. Create a new Google Sheet named **Content Queue**.
2. Rename its default tab (bottom-left, usually labeled `Sheet1`) to exactly **Content Queue** — this is the tab name the workflow looks for, not the spreadsheet file's name. Mismatched tab names are the single most common setup mistake here.
3. Add a header row with these exact column names: `content`, `status`, `practice_area`, `posted_at`, `notes`.
4. Add one row per post idea — a field note, a practical tip, a firm announcement, anything you'd want a prospective client searching locally to see. Type the raw text into `content`. Leave `status` blank (blank means "queued and ready"). `practice_area` is optional but helps Claude tailor the copy.
5. Copy the sheet's ID from its URL: `.../spreadsheets/d/THIS_PART_IS_THE_ID/edit`.

Already running the Content Repurposer (PAC-31)? You can paste the `gbp_post` asset it emails you straight into a queue row — Claude will polish it further, which is mostly a no-op since it's already close to ready.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-60-google-business-profile-auto-poster.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-60-google-business-profile-auto-poster.json). Do not activate it yet.
2. Anthropic credential: reuse your existing `Anthropic API key` credential if PAC-31 or PAC-50 is already deployed. Otherwise: **Credentials → New → Header Auth** → Header Name `x-api-key`, Header Value your key from console.anthropic.com → save as `Anthropic API key`.
3. Google Sheets credential: **Credentials → New → Google Sheets OAuth2** → connect the Google account that owns your Content Queue sheet → save as `Google Sheets account` (reuse if NTC-33 is already deployed).
4. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `GBP_CONTENT_QUEUE_SHEET_ID` | The Content Queue sheet ID from Step 1 |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | Where compliance-log entries reference your firm |
| `FIRM_FROM_EMAIL` | The sender address for notification emails |
| `FIRM_STAFF_EMAIL` | Where weekly ready-to-paste posts and flagged-content alerts go — falls back to `FIRM_EMAIL` if unset |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_GBP_CTA_URL` *(optional)* | A URL to suggest for the post's call-to-action button when you paste it in (e.g. your booking page) |
| `FIRM_GBP_CTA_TYPE` *(optional)* | One of `LEARN_MORE`, `BOOK`, `CALL`, `SIGN_UP` — defaults to `LEARN_MORE` if unset |
| `FIRM_GBP_DASHBOARD_URL` *(optional)* | A direct link to your Business Profile's post page, included in the email for convenience — reuse PAC-50's value if already deployed |

---

## Step 4 — Activate

Toggle the workflow to **Active**. It runs weekly on its own (Mondays at 9am server time by default) — no webhook URL to wire anywhere. Adjust the cron expression on **Weekly GBP Post Trigger** for a different day or cadence.

---

## Step 5 — Test

The workflow ships with two pinned test rows on **Fetch Content Queue Sheet** (one already "Posted," one queued and ready) so you can test without waiting for the real schedule.

1. Run **Fetch Content Queue Sheet** → **Find Next Queued Content Row** and confirm it picks the queued row (`row_number: 3`), not the already-posted one.
2. Run the workflow forward through **Extract & Safety-Check GBP Post** and confirm `is_marketing_safe: true` and a `gbp_post` under 1,500 characters.
3. Let it continue through **Mark Queue Row As Drafted** and confirm the Content Queue sheet's row 3 now shows `status: Drafted` with a `posted_at` timestamp.
4. Confirm `FIRM_STAFF_EMAIL` received the ready-to-paste post, including the suggested call-to-action line if you set `FIRM_GBP_CTA_URL`.
5. Copy the post text from that email into Google Business Profile yourself (business.google.com → your profile → Add update) to confirm it fits and reads well — this is the one manual step in the whole workflow.

**Test the flagged path:** temporarily add a queue row with content like "pursuant to the applicable statute, we advise clients to file within 30 days," run the workflow, and confirm it routes to **Mark Queue Row Needs Review** instead of drafting — the keyword scan should catch it even if Claude doesn't self-flag it. Confirm the row's `status` becomes `Needs Review` with a reason in `notes`, and that staff get the flagged-content email instead of a ready-to-paste post. Delete the test row afterward.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Queue has a row with blank/"Queued" status | Repurposed via Claude, safety-checked, emailed to staff as a ready-to-paste post, row marked "Drafted" |
| Every row is already "Drafted" or "Needs Review" | Staff get a reminder email; nothing drafted this week |
| Claude self-reports it can't produce safe copy | Row marked "Needs Review" with the reason; nothing drafted |
| Generated post passes Claude's self-check but the keyword scan catches legal-sounding language | Same — marked "Needs Review," nothing drafted |
| Generated post exceeds 1,500 characters | Truncated at a word boundary before it's emailed; flagged in the email as worth a review |

---

## Compliance note

This template performs marketing copy generation only — it never drafts legal text, never references case facts, and never identifies a specific client or matter (ABA Op. 512). The same compliance rules used in the Content Repurposer (PAC-31) are enforced here, checked twice: once by Claude's own self-report, once by an independent keyword scan, before anything reaches a staff inbox. Every draft and every flagged exception is logged to the Bar-Compliance Guardrail's audit sheet.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces the weekly copywriting piece of local-SEO upkeep — a stale Business Profile is a real leak given 74% of clients start with a search |
| Compliance test | Marketing copy only, checked twice before it reaches anyone, no legal work |
| Searchability test | Ranks for "google business profile posts for law firms" |
| Deployability test | One sheet, one Anthropic key, one SMTP credential — under 15 minutes, no third-party bridge tool required |
| Upsell test | Quick-Win Build adds true one-click auto-publishing once Business Profile API access is secured on the firm's behalf, plus photo posts and multi-location support |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- True auto-publishing — Protomated handles securing Business Profile posting access so the post goes live with zero manual paste step
- Photo and offer-post support (not just text updates)
- Multi-location posting for firms with more than one office
- An AI-suggested content calendar so the queue never runs dry
- A monthly local-SEO performance digest (views, searches, actions taken on the profile)

[Book a call with Protomated](https://protomated.com/book) to get started.
