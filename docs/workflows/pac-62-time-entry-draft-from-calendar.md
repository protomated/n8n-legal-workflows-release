# Deploy Guide: Time-Entry Draft from Calendar

**Template:** PTPAC-62 — B3: Time-Entry Draft from Calendar
**Pillar:** Keep
**Replaces:** Clio/MyCase native partial capture, manual time entry

Get value in under 15 minutes. Every morning, the workflow reads yesterday's calendar events tagged with a Clio matter reference, drafts a factual billing narrative for each one, and emails one digest listing every draft with an Approve button. Firms typically bill only about 2.4 of 8 worked hours — most of the gap is simply unlogged work that never became revenue because nobody sat down and wrote the entry.

---

## Before you start: heads up on this one

**Matter matching works off a tag you type, not fuzzy matching.** Include the Clio matter's display number in square brackets anywhere in the calendar event title — e.g. `[2026-CH-042] Settlement call with opposing counsel`. Untagged events (lunch, personal appointments, internal standups) are never touched; the tag *is* the entire filter, so there's no keyword blocklist to maintain.

**The narrative is drafted from calendar metadata only.** Claude never sees a linked document, an email body, or anything beyond the event's own title, description, attendees, and duration. This keeps it squarely on the "operational" side of ABA Op. 512 — it's billing documentation, not legal work product.

**Approving is two steps, not one click, on purpose.** A single GET link that creates a real time entry is a known trap: email security scanners (Microsoft Defender Safe Links, Proofpoint, etc.) automatically pre-fetch every link in an inbound email, which would silently "approve" every draft before the attorney ever saw it. Clicking Approve opens a review page; only an actual button click (a POST) creates the entry. Nothing reaches Clio until that second step.

**The Clio time-entry creation call (`activities.json`) hasn't been independently confirmed live** — the endpoint, the `type: "TimeEntry"` discriminator, and `quantity` being in seconds all follow Clio's general v4 REST conventions but weren't tested against a live account during authoring. Verify your first real approval creates an entry with the correct duration before relying on this for real billing.

**"Yesterday" is computed from the n8n server's own clock and timezone, not the attorney's.** If your n8n server isn't running in the firm's own timezone, "yesterday" can be off by a full day for several hours around each day's boundary — e.g. a server in America/New_York and a firm in Africa/Lagos disagree on what day it is for 5 hours out of every 24. Confirm your n8n server's timezone (check any trigger node's timestamp output) matches, or is close enough to, the firm's before relying on the daily digest timing.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** Called purely for audit logging — every message here is internal, so opt-out checking doesn't apply, but every digest and every approval is still recorded.
- An n8n instance (self-hosted or n8n Cloud) with a **publicly reachable URL** — the Approve links in the digest email need to work from the attorney's inbox, not just from inside your network.
- A Clio account with API access (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61 if already deployed)
- A Google Calendar account with OAuth2 access
- An [Anthropic API key](https://console.anthropic.com) — reuse the one from PAC-31/50/60 if already deployed
- An email account with SMTP access
- **One attorney's calendar and Clio user ID per deployment** — see "Multi-attorney firms" below

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-62-time-entry-draft-from-calendar.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-62-time-entry-draft-from-calendar.json). Do not activate it yet.
2. Google Calendar credential: **Credentials → New → Google Calendar OAuth2 API** → connect the attorney's Google account → save as `Google Calendar account` (reuse NTC-16's if already deployed).
3. Clio credential: reuse your existing `Clio API (OAuth2)` credential if PAC-18, PAC-20, PAC-55, PAC-57, PAC-58, PAC-59, or PAC-61 is already deployed.
4. Anthropic credential: reuse your existing `Anthropic API key` credential if PAC-31, PAC-50, or PAC-60 is already deployed.
5. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `N8N_PUBLIC_URL` | Your n8n instance's public base URL, no trailing slash (e.g. `https://your-n8n.example.com`) — the Approve links in the digest email are built from this |
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_ATTORNEY_CALENDAR_ID` | Use `primary` for the connected Google account's own calendar, or a specific calendar ID/email for a shared/delegated calendar |
| `FIRM_ATTORNEY_CLIO_USER_ID` | The attorney's Clio user ID — find it in Clio under Settings → Users. This attributes every created time entry to the right person. |
| `FIRM_ATTORNEY_EMAIL` | Where the daily digest is sent |
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | The sender address for the digest email |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |

---

## Step 3 — Activate

Toggle the workflow to **Active**. The compile trigger runs every morning at 7am (n8n server timezone) on its own — no manual step needed there. The two Approve webhooks activate automatically alongside it; you don't need to copy their URLs anywhere manually since the digest email builds them itself from `N8N_PUBLIC_URL`.

---

## Step 4 — Test

**One-click full run:** the digest compile path is pinned end to end — the calendar fetch, the Clio matter search, and the Claude narrative call all ship with realistic canned sample data. That means anyone on the team can open the workflow and click **Execute workflow** and it runs all the way through to a real digest email in your inbox, with zero live Google Calendar, Clio, or Anthropic credentials required for this test path (SMTP and your Bar-Compliance Guardrail connection still need to be live, since those two are what actually deliver and log the result). This is the fastest way to confirm the whole pipeline and the email styling both work before doing a real end-to-end test against live accounts.

1. Open the workflow and click **Execute workflow**.
2. Confirm you receive a digest email with one **Ready to approve** card (`[2026-CH-042] Settlement call with opposing counsel`, a Claude-drafted narrative, and a working Review & Approve button) — the untagged "Team lunch" event should not appear anywhere, not even as an error.
3. Click **Review & Approve** and confirm you land on a review page showing the matter, date, duration, and narrative — **not** a Clio entry created yet.
4. Click **Confirm & Log This Time Entry** and confirm you land on a success page, then check Clio directly to confirm the entry was created with the correct matter, date, duration, and narrative.
5. **Test the scanner-safety behavior directly:** open the Approve link's URL in a plain `curl` request or a second browser without clicking the confirm button, and verify no Clio entry gets created just from that GET request alone.

**Test with your own live calendar and Clio account:** once the pinned run above looks right, do one real pass to confirm the actual integrations work — unpin **"Fetch Yesterdays Calendar Events"**, **"Search Clio Matter By Reference"**, and **"Generate Time Entry Narrative"** (or just delete their pinned data if you don't need it going forward), then create a real Google Calendar event dated for whatever day your n8n server currently considers "yesterday." Check that day against the server's own clock — open **"Daily Time Entry Digest Trigger"**'s last output, not your own device's calendar — since a firm's local timezone and the n8n server's system timezone can disagree on what day it currently is. See "Before you start" above for why this matters for your actual deployment, not just testing.

**Test the flagged path:** temporarily edit the pinned calendar event's matter tag to something that doesn't exist in your Clio account (or unpin and use a real mismatched tag), run the digest compile, and confirm it shows up under "Needs attention" in the digest instead of silently disappearing.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Calendar event tagged `[matter-ref]`, matter found, narrative passes both safety checks | Appears in the digest under "Ready To Approve" with a working button |
| Calendar event with no bracket tag | Silently excluded — not billable by definition, not an error |
| All-day event, even if tagged | Excluded — no meaningful duration to compute |
| Tagged event, but the reference doesn't match any real Clio matter | Appears under "Needs Attention" with the reason — no narrative generated, no entry created |
| Narrative flagged by either safety layer | Appears under "Needs Attention" — needs a human rewrite before it can be approved |
| Approve link clicked (GET) | Shows a review page only — nothing written to Clio yet |
| Confirm button clicked (POST) | Creates the real Clio time entry, logs the approval, shows a success page |
| Every tagged event yesterday | No digest sent that day |

---

## Multi-attorney firms

This template is scoped to one calendar and one Clio user per deployment. For a firm with several attorneys, deploy one copy of this workflow per attorney — change `FIRM_ATTORNEY_CALENDAR_ID`, `FIRM_ATTORNEY_CLIO_USER_ID`, and `FIRM_ATTORNEY_EMAIL` each time, and give each copy's webhook paths a unique suffix (e.g. `approve-time-entry-jsmith`) so the Approve links from different attorneys' digests don't collide.

---

## Compliance note

This template performs scheduling, metadata-based drafting, and internal notification only — the narrative generator never has access to a linked document, an email body, or anything beyond what's already visible on the calendar event itself, and it never contacts a client (ABA Op. 512). Nothing is written to Clio until the attorney explicitly reviews and confirms it. Every digest compilation and every approval is logged to the Bar-Compliance Guardrail's audit sheet.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Firms bill roughly 2.4 of 8 worked hours — this recovers unlogged time that would otherwise never become revenue |
| Compliance test | Metadata-only narrative drafting, two-layer safety check, no legal work, attorney approval required before anything is logged |
| Searchability test | Ranks for "automatic time entry from calendar lawyers" |
| Deployability test | Reuses Clio, Anthropic, and SMTP credentials from earlier templates if already deployed — under 15 minutes plus a public n8n URL |
| Upsell test | Quick-Win Build adds email-thread-based entries, multi-attorney support in one workflow, and one-click bulk approval |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Email-thread-based draft entries alongside calendar events, for work that never made it onto the calendar
- Multi-attorney support in a single workflow instead of one deployment per attorney
- One-click bulk approval for an entire day's digest instead of approving each entry individually
- Smart duration suggestions that account for typical rounding/minimum-increment billing policies per matter type

[Book a call with Protomated](https://protomated.com/book) to get started.
