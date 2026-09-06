# Deploy Guide: Negative-Review Rapid-Response

**Template:** PAC-50 — G2: Negative-Review Rapid-Response
**Pillar:** Get
**Replaces:** Reputation-management tools ($99–$2,000/month)

Get value in under 15 minutes. The moment a new Google review lands at 1–3 stars, your firm gets an instant text alert and, a few seconds later, an email with the full review, a direct reply link, and a Claude-drafted public reply that never confirms a client relationship, references case facts, or admits fault. Firms that respond within 24 hours convert roughly a third of negative reviewers to neutral or positive.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** This template calls it to log every negative-review event to your compliance audit sheet. Complete the NTC-33 deploy guide before continuing here.
- An n8n instance (self-hosted or n8n Cloud)
- A free [Zapier](https://zapier.com) account — used to bridge Google Business Profile's "New Review" trigger to this workflow (Google doesn't offer a webhook for new reviews directly, and Zapier's free tier covers this easily)
- A Google Business Profile for your firm, claimed and verified (business.google.com — free, typically takes 3–5 days to verify if you don't have one yet)
- An [Anthropic API key](https://console.anthropic.com) — used to draft the public reply
- A Twilio account with an SMS-enabled phone number — used for the instant alert text
- An email account with SMTP access (Gmail, Outlook 365, Zoho Mail, or similar)

---

## Step 1 — Import the workflow (1 min)

1. Open your n8n canvas.
2. Press `Ctrl+V` (or `Cmd+V` on Mac) and paste the contents of [`pac-50-negative-review-rapid-response.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-50-negative-review-rapid-response.json).
3. Do not activate it yet.

---

## Step 2 — Set n8n Variables (2 min)

In n8n, open your project and click the **Variables** tab, then add:

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | Where the alert + draft email is sent |
| `FIRM_FROM_EMAIL` | The address the alert email is sent from — must be authorised by your SMTP account |
| `FIRM_ALERT_PHONE` | **Your firm's own cell number** (E.164, e.g. `+15550001234`) — this is who gets the instant SMS, never the reviewer |
| `FIRM_TWILIO_NUMBER` | Your Twilio number in E.164 format — the sending number for the SMS alert |
| `FIRM_GBP_DASHBOARD_URL` | A fallback link to your Google Business Profile reviews dashboard (business.google.com → Reviews), used only if Zapier doesn't pass a direct reply link |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33: `.../workflow/WORKFLOW_ID` |

---

## Step 3 — Add credentials (5 min)

### Anthropic credential
1. Get an API key at [console.anthropic.com](https://console.anthropic.com).
2. In n8n: **Credentials → New credential → Header Auth**.
3. Header Name: `x-api-key`. Header Value: your Anthropic API key.
4. Save it as `Anthropic API key`.

### Twilio credential
1. **Credentials → New credential → Twilio**.
2. Enter your **Account SID** and **Auth Token** from console.twilio.com.
3. Save it as `Twilio account`.

### Email credential
1. **Credentials → New credential → SMTP**.
2. Enter your provider's SMTP settings (Gmail: use an App Password; Microsoft 365: enable SMTP AUTH in the admin centre).
3. Save it as `Email account (SMTP)`.

---

## Step 4 — Activate and copy the webhook URL (1 min)

1. Toggle the workflow to **Active**.
2. Open the **When New Review Received** node and copy its **Production URL** (ends in `/webhook/new-review`).

---

## Step 5 — Connect Google Business Profile via Zapier (4 min)

Google doesn't offer a webhook for new reviews, so this template uses a free Zapier zap as the bridge.

1. In Zapier, click **Create Zap**.
2. **Trigger:** app `Google Business Profile` → event `New Review` → connect your Google account → select your business location.
3. **Action:** app `Webhooks by Zapier` → event `POST`.
4. **URL:** paste the Production URL from Step 4.
5. **Payload Type:** `json`.
6. **Data:** map these exact fields (click into each field and insert the matching value from the trigger step):

   | Field name | Map to |
   |---|---|
   | `review_id` | Review ID |
   | `reviewer_name` | Reviewer Display Name |
   | `star_rating` | Star Rating |
   | `review_text` | Comment |
   | `review_url` | Review Link (if Zapier's Google Business Profile trigger doesn't offer a direct link, leave this blank — the workflow falls back to `FIRM_GBP_DASHBOARD_URL`) |
   | `location_name` | Location Name (only needed if you manage more than one location) |
   | `created_at` | Create Time |

7. Turn the Zap on.

**Already use a reputation tool with its own webhooks (Podium, BirdEye, etc.)?** Point it at the same Production URL instead of using Zapier, as long as it sends the same field names above.

---

## Step 6 — Test

The workflow ships with a pinned 2-star review payload on the **When New Review Received** node.

1. Click **When New Review Received** → **Test step**.
2. Confirm **Normalize New Review Payload** outputs `is_valid_review: true`, `rating: 2`, `is_negative_review: true`.
3. Confirm **Check For Duplicate Review** outputs `is_duplicate: false` on the first run.
4. Run it again with the same pinned data — confirm `is_duplicate: true` the second time, and that it routes to **Skip — Duplicate Review**.
5. Run a fresh execution (n8n Test URL, not the pinned data) with a different `review_id` and `star_rating: "FIVE"` — confirm it routes to **Skip — Positive Review**.
6. With a fresh negative-review payload, confirm you receive the SMS alert within a few seconds, then the follow-up email with the drafted reply.

**Test the safety net:** temporarily edit the prompt in **Build Alert And Draft Prompt** to remove one of the guardrail bullet points, run a review through, and confirm **Extract And Safety-Check Draft Response** still catches unsafe language via the Layer 2 keyword scan (`is_response_safe: false`). Undo the edit afterward.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| New review, 1–3 stars, first time seen | Instant SMS to the firm → Claude drafts a reply → safety-checked → logged to the compliance audit sheet → full email delivered |
| New review, 4–5 stars | Skipped — **Skip — Positive Review**. No alert. |
| Same `review_id` delivered again (Zapier redelivery, or Google re-firing after an edit) within 30 days | Skipped — **Skip — Duplicate Review** |
| Malformed payload (missing `review_id` or an unparseable rating) | Skipped — **Skip — Invalid Review Payload** |
| Claude self-reports it cannot draft a safe reply | Draft omitted; email tells the firm to write the reply manually, with the reason given |
| Claude's draft slips past self-check but contains confidentiality- or admission-risk language | Caught by the independent keyword scan; email tells the firm to write the reply manually |
| Twilio isn't configured yet, or the SMS send fails | Workflow continues — the follow-up email still carries the full alert and draft |

---

## Compliance note

This template is operational only — it alerts the firm and drafts text for the firm to review and post itself; nothing is sent to a client or posted publicly without a human in the loop. It does not touch legal work (ABA Op. 512). The Claude prompt hard-codes ABA Formal Opinion 496 guardrails for responding to online reviews — never confirm or deny a client relationship, never reference case facts, never admit fault — and every draft is independently re-checked by a keyword scan before it reaches the firm's inbox. Every event is logged to the Bar-Compliance Guardrail's audit sheet.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces reputation-management tools ($99–$2,000/mo). Firms that respond within 24 hours convert roughly 33% of negative reviewers to neutral or positive. |
| Compliance test | Operational only — no legal work. Draft replies are firm-reviewed before posting and are constrained by ABA Formal Opinion 496 guardrails, checked twice. |
| Searchability test | Ranks for "respond to negative law firm reviews" |
| Deployability test | 7 config values, three credential entries, one Zapier zap — under 15 minutes |
| Upsell test | Quick-Win Build adds direct posting to Google (no manual copy-paste), a weekly reputation digest, and CRM logging |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- One-click posting of the approved reply directly to Google (no copy-paste)
- A weekly digest of every review — positive and negative — with sentiment trends
- CRM integration — logs every review and response in Salesmate
- Multi-location support with per-location routing and alerts
- Escalation rules — 1-star reviews page a partner directly, not just the shared inbox

[Book a call with Protomated](https://protomated.com/book) to get started.
