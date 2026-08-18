# Deploy Guide: Google Business Profile Auto-Poster

**Template:** PTPAC-60 — G3: Google Business Profile Auto-Poster
**Pillar:** Get
**Replaces:** A marketing retainer's local-SEO upkeep

Get value in under 15 minutes of setup, plus a few minutes in Zapier. Once a week, the workflow pulls the next item from a content queue you maintain, turns it into a ready-to-publish Google Business Profile post with Claude, safety-checks it twice, and publishes it automatically — no one has to remember to log into Google Business Profile and post something every week. 74% of prospective clients start with a search; a profile that hasn't posted in months signals a firm that isn't actively practicing.

---

## Before you start: heads up on this one

**This can't call the Google Business Profile API directly.** Google restricts posting access (the Business Profile / My Business API) to developers it has specifically approved — most firms and most agencies never get that approval. So this template hands the approved post text to a Zapier zap instead, which already has that access through Zapier's own partner integration. You'll spend about 4 minutes setting up a free Zapier zap as part of deployment.

**Nothing publishes without passing two independent safety checks.** Claude self-reports whether the content it generated is safe, and a separate keyword scan re-checks the output afterward — same two-layer approach as the Content Repurposer (PAC-31) and Negative-Review Rapid-Response (PAC-50). If either layer catches anything legal-sounding, nothing is published; the item is held for a human to fix instead.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** This template calls it to log every publish (and every flagged exception) to your compliance audit sheet.
- An n8n instance (self-hosted or n8n Cloud)
- A Google Sheet you'll use as the content queue
- A Google Business Profile for your firm, claimed and verified (business.google.com)
- A free [Zapier](https://zapier.com) account
- An [Anthropic API key](https://console.anthropic.com) — reuse the one from PAC-31/PAC-50 if already deployed
- An email account with SMTP access

---

## Step 1 — Create the Content Queue sheet (3 min)

1. Create a new Google Sheet named **Content Queue**.
2. Add a header row with these exact column names: `content`, `status`, `practice_area`, `posted_at`, `notes`.
3. Add one row per post idea — a field note, a practical tip, a firm announcement, anything you'd want a prospective client searching locally to see. Type the raw text into `content`. Leave `status` blank (blank means "queued and ready"). `practice_area` is optional but helps Claude tailor the copy.
4. Copy the sheet's ID from its URL: `.../spreadsheets/d/THIS_PART_IS_THE_ID/edit`.

Already running the Content Repurposer (PAC-31)? You can paste the `gbp_post` asset it emails you straight into a queue row — Claude will polish it further, which is mostly a no-op since it's already close to publish-ready.

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
| `FIRM_STAFF_EMAIL` | Where weekly publish confirmations and flagged-content alerts go — falls back to `FIRM_EMAIL` if unset |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `ZAPIER_GBP_POST_WEBHOOK_URL` | The catch hook URL from your Zapier zap — set this in Step 4 below |
| `FIRM_GBP_CTA_URL` *(optional)* | A URL for the post's call-to-action button (e.g. your booking page) |
| `FIRM_GBP_CTA_TYPE` *(optional)* | One of `LEARN_MORE`, `BOOK`, `CALL`, `SIGN_UP` — defaults to `LEARN_MORE` if unset |

---

## Step 4 — Connect Zapier for publishing (4 min)

1. In Zapier, click **Create Zap**.
2. **Trigger:** app `Webhooks by Zapier` → event `Catch Hook`. Copy the generated catch hook URL — this is your `ZAPIER_GBP_POST_WEBHOOK_URL` from Step 3.
3. **Action:** app `Google Business Profile` → action `Create Post` → connect your Google account → select your business location.
4. Map the action's post/summary field to the hook's `content` field. If your Zapier plan's action exposes call-to-action button fields, map them to `cta_type` and `cta_url`.
5. Turn the Zap on.

**A 200 response from this step only confirms Zapier received the request — not that Google accepted the post.** Spot-check your Business Profile and Zapier's Zap history periodically, especially right after first deploying.

---

## Step 5 — Activate

Toggle the workflow to **Active**. It runs weekly on its own (Mondays at 9am server time by default) — no webhook URL to wire anywhere. Adjust the cron expression on **Weekly GBP Post Trigger** for a different day or cadence.

---

## Step 6 — Test

The workflow ships with two pinned test rows on **Fetch Content Queue Sheet** (one already "Posted," one queued and ready) so you can test without waiting for the real schedule.

1. Run **Fetch Content Queue Sheet** → **Find Next Queued Content Row** and confirm it picks the queued row (`row_number: 3`), not the already-posted one.
2. Run the workflow forward through **Extract & Safety-Check GBP Post** and confirm `is_marketing_safe: true` and a `gbp_post` under 1,500 characters.
3. Let it continue to **Post To Google Business Profile Via Zapier**, then check Zapier's Zap history for a successful run, and your Business Profile a few minutes later for the live post.
4. Confirm the Content Queue sheet's row 3 now shows `status: Posted` with a `posted_at` timestamp.
5. Confirm `FIRM_STAFF_EMAIL` received the "post published" summary.

**Test the flagged path:** temporarily add a queue row with content like "pursuant to the applicable statute, we advise clients to file within 30 days," run the workflow, and confirm it routes to **Mark Queue Row Needs Review** instead of publishing — the keyword scan should catch it even if Claude doesn't self-flag it. Confirm the row's `status` becomes `Needs Review` with a reason in `notes`, and that staff get the flagged-content email instead of a publish confirmation. Delete the test row afterward.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Queue has a row with blank/"Queued" status | Repurposed via Claude, safety-checked, published, row marked "Posted" |
| Every row is already "Posted" or "Needs Review" | Staff get a reminder email; nothing published this week |
| Claude self-reports it can't produce safe copy | Row marked "Needs Review" with the reason; nothing published |
| Generated post passes Claude's self-check but the keyword scan catches legal-sounding language | Same — marked "Needs Review," nothing published |
| Generated post exceeds 1,500 characters | Truncated at a word boundary before publishing; flagged in the confirmation email as worth a review |
| Zapier call fails or times out | Workflow continues (doesn't error out) — check Zapier's Zap history if a post seems to be missing |

---

## Compliance note

This template performs marketing copy generation and scheduled publishing only — it never drafts legal text, never references case facts, and never identifies a specific client or matter (ABA Op. 512). The same compliance rules used in the Content Repurposer (PAC-31) are enforced here, checked twice: once by Claude's own self-report, once by an independent keyword scan, before anything reaches the public profile. Every publish and every flagged exception is logged to the Bar-Compliance Guardrail's audit sheet.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces the local-SEO upkeep piece of a marketing retainer — a stale Business Profile is a real leak given 74% of clients start with a search |
| Compliance test | Marketing copy only, checked twice before publishing, no legal work |
| Searchability test | Ranks for "google business profile posts for law firms" |
| Deployability test | One sheet, one Anthropic key, one Zapier zap — under 15 minutes plus Zapier setup |
| Upsell test | Quick-Win Build adds photo posts, multi-location support, and a content calendar with AI-suggested topics |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Photo and offer-post support (not just text updates)
- Multi-location posting for firms with more than one office
- An AI-suggested content calendar so the queue never runs dry
- A monthly local-SEO performance digest (views, searches, actions taken on the profile)

[Book a call with Protomated](https://protomated.com/book) to get started.
