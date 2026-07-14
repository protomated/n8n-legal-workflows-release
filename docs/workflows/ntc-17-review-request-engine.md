# Deploy Guide: Review Request Engine
**NTC-17 | Get | Free Template**
Target keyword: automated google review requests for law firms

Get value in under 15 minutes. When you close a matter, your client automatically receives a review request at the 48-hour sweet spot. Clients who loved the experience are routed to your Google Business Profile. Clients with concerns are routed to a private feedback form — protecting your public rating while giving you a chance to make it right.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** Every outbound message in this workflow passes through it for opt-out checking and TCPA disclaimer attachment. Complete the NTC-33 deploy guide before continuing here.
- An n8n instance (self-hosted or n8n Cloud)
- A Twilio account with an SMS-enabled phone number
- An email account with SMTP access (Gmail, Outlook 365, Zoho Mail, or similar)
- **Your Google Reviews short link** — find it in [Google Business Profile](https://business.google.com): Home → "Get more reviews" → Share review form. It looks like `https://g.page/r/YOUR_PLACE_ID/review`. No Google Business Profile yet? Create one at business.google.com — it is free and typically takes 3–5 days to verify.
- **A private feedback form URL** — a free [Google Form](https://forms.google.com) works well. Add questions like "What could we have done better?" and copy the share link. Alternatively use Typeform (free tier) or your website contact page.

> **Self-hosted n8n note:** The 48-hour wait requires Postgres as your n8n database. If you are running n8n with the default SQLite and the server restarts during the wait, the execution will be lost. n8n Cloud handles this automatically. See [n8n's database docs](https://docs.n8n.io/hosting/databases/) for Postgres setup.

---

## Step 1 — Import the workflow (1 min)

1. Open your n8n canvas.
2. Press `Ctrl+V` (or `Cmd+V` on Mac) and paste the contents of [`ntc-17-review-request-engine.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/ntc-17-review-request-engine.json).
3. The workflow appears with two parallel paths (Path A and Path B). Do not activate it yet.

---

## Step 2 — Set n8n Variables (2 min)

In n8n, open your project and click the **Variables** tab, then add the following variables. Set them once — Path A and Path B both read these same values automatically.

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name as it appears in texts |
| `FIRM_EMAIL` | Your intake email address (firm notifications go here) |
| `FIRM_FROM_EMAIL` | The address notification emails are sent from — must be authorised by your SMTP account |
| `FIRM_TWILIO_NUMBER` | Your Twilio number in E.164 format (e.g. `+15550001234`) — clients text replies to this number |
| `FIRM_GOOGLE_REVIEW_URL` | Google Business Profile → Home → "Get more reviews" → copy the short URL (e.g. `https://g.page/r/YOUR_PLACE_ID/review`) |
| `FIRM_FEEDBACK_URL` | Your private feedback form or contact page URL |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33: `.../workflow/WORKFLOW_ID` |

---

## Step 3 — Add credentials (5 min)

### Twilio credential
1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **Twilio** and select it.
3. Enter your **Account SID** and **Auth Token** (both on your Twilio Console dashboard at console.twilio.com).
4. Save it as `Twilio account`.

### Email credential
1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **SMTP** and select it.
3. Enter your email provider's SMTP settings:

   **Gmail:**
   - Host: `smtp.gmail.com`, Port: `465`, Security: SSL
   - Username: your Gmail address
   - Password: an App Password (Google Account → Security → 2-Step Verification → App Passwords)

   **Outlook / Microsoft 365:**
   - Host: `smtp.office365.com`, Port: `587`, Security: STARTTLS
   - Username: your email address, Password: your password
   - Note: SMTP AUTH must be enabled in your Microsoft 365 admin centre

   **Zoho Mail:**
   - Host: `smtp.zoho.com`, Port: `465`, Security: SSL

4. Save it as `Email account (SMTP)`.

---

## Step 4 — Verify n8n Variables are live (1 min)

No code nodes need editing — firm settings are read from the project **Variables** tab (set in Step 2). Both Path A (Set Up Review Request) and Path B (Parse Rating Reply) read the same variables automatically.

To verify, open the **Set Up Review Request** node and confirm the code reads `$vars.FIRM_NAME` (not a hardcoded string). If you see a hardcoded string, you are running an older version of the workflow — re-import from the latest JSON.

---

## Step 5 — Wire your practice management system (2 min)

Configure your practice management system to POST to the **Path A webhook URL** whenever a matter is closed.

**Activate the workflow first** (Step 6) so the URL is live before you register it.

### Clio

1. In Clio: **Settings → Webhooks → New Webhook**.
2. Topic: **Matter**.
3. Paste the **When Matter Closed** Production URL.
4. Subscribe to **matter.updated** events.
5. Save.

Clio fires `matter.updated` on any matter status change. The **Normalize and Validate Closure** node checks `status === 'Closed'` and skips all other updates automatically.

### Generic / other systems

Send a POST request to the Path A URL with this JSON body when a matter closes:

```json
{
  "matter_status": "closed",
  "matter_id": "M-2026-042",
  "matter_title": "Smith v. Jones — Personal Injury",
  "client_name": "Jane Smith",
  "client_phone": "+14155551234",
  "client_email": "jane.smith@example.com"
}
```

---

## Step 6 — Wire Twilio inbound messaging (2 min)

This tells Twilio where to send a client's reply when they text your firm's number.

1. **Activate the workflow** in n8n (toggle the Active switch) — the URL must be live before Twilio can reach it.
2. Copy the **When Rating Reply Received** Production URL (`/webhook/review-reply`).
3. In the Twilio Console: **Phone Numbers → Manage → Active Numbers** → click your firm's number.
4. Under **Messaging Configuration**, find **"When a message comes in"**.
5. Select **Webhook** and paste the Path B URL.
6. Set method to **HTTP POST**.
7. Click **Save**.

> **Important:** This replaces any existing incoming-message webhook on that Twilio number. If you already use inbound SMS routing on that number, coordinate with your existing setup before saving.

> **Twilio console warnings:** After wiring, you may see "Empty/Invalid TwiML" warnings in the Twilio debugger. These are safe to ignore — n8n returns JSON rather than TwiML, which Twilio accepts as a 200 OK. Your workflow is working correctly.

---

## Step 7 — Activate and test

### Test Path A (matter close → 48h wait → SMS)

The workflow ships with a pinned Clio `matter.updated` payload on the **When Matter Closed** node.

1. Click **When Matter Closed** → **Test step**.
2. n8n runs the pinned payload through Normalize and Validate Closure and If Valid Matter Closure.
3. Confirm `is_valid_closure = true` and the data parses correctly.
4. Confirm the Set Up Review Request node outputs `send_at` (48 hours from now) and a well-formed `rating_sms`.

> To test the full SMS delivery, change `send_at` in Set Up Review Request temporarily to `new Date(now.getTime() + 30_000).toISOString()` (30 seconds from now), run a live execution, and check that the text arrives on your phone. Reset to `MS_48H` before going live.

**Negative test — guard node:**

Send a POST with a non-closure status to confirm the guard works:

```json
{
  "matter_status": "open",
  "client_phone": "+15559998888"
}
```

The execution should route to **Ignore Non-Closure Event** with no text sent.

---

### Test Path B (client reply → routing)

The workflow ships with a pinned Twilio inbound SMS payload (`Body: "5"`) on the **When Rating Reply Received** node.

1. Click **When Rating Reply Received** → **Test step**.
2. Confirm `rating = 5`, `is_happy = true`, `is_valid_rating = true`.
3. Confirm the **Determine Rating Path** node routes to the Yes branch → **Send Google Reviews Link**.

**Test the unhappy path:**

Change the pinned `Body` value to `"2"` and re-run. Confirm:
- `is_happy = false`
- Routes to **Send Feedback Form Link** → **Alert Firm — Unhappy Client**
- You receive the feedback alert email

**Test the guard:**

Change `Body` to `"hello"` and confirm it routes to **Ignore Non-Rating Reply**.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Matter closes in Clio with client phone | 48 hours later: compliance check runs → rating request SMS sent (TCPA disclaimer confirmed) |
| Matter updated (not to Closed) | Event skipped — **Ignore Non-Closure Event** |
| Matter closed but no client phone | Event skipped — **Ignore Non-Closure Event** |
| Client is on the opt-out list at the 48h mark | Rating request suppressed — **Skip — contact opted out**; suppression logged to audit sheet |
| Client replies 4 or 5 | Compliance check runs → Google Reviews link sent via SMS |
| Client replies 1, 2, or 3 | Compliance check runs → private feedback form link sent + firm gets alert email |
| Client replies with a rating but is on the opt-out list | Message suppressed — **Skip — opted out (4–5 stars)** or **Skip — opted out (1–3 stars)**; suppression logged; firm alert is not triggered |
| Client replies STOP | Twilio handles opt-out; workflow routes to **Ignore Non-Rating Reply** — add them to your opt-out sheet too |
| Client replies with text, not a number | Routes to **Ignore Non-Rating Reply** — no Google link sent |
| Twilio send error (wrong credentials, invalid number) | Workflow continues to firm notification email (Path A only) |
| Bar-Compliance Guardrail audit log write fails | Approval/suppression result still returned — send decision is unaffected; logging failure does not block the message |

---

## Compliance note

This template is operational only — it sends a follow-up message to a former client after matter close. It does not touch legal work (ABA Op. 512). Every outbound message passes through the Bar-Compliance Guardrail before sending: opt-out status is checked, TCPA disclaimer language is confirmed present, and the attempt is logged to your audit sheet regardless of whether the message is sent or suppressed.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces Clio Reputation and Birdeye ($49–$250+/mo). Eliminates 2–3 hours/week of manual review follow-up. Rating-gating prevents unhappy clients from reaching your public Google profile. |
| Compliance test | Operational only — sends a follow-up SMS to a former client. No legal work. Opt-out checking and TCPA disclaimer handled by Bar-Compliance Guardrail. |
| Searchability test | Ranks for "automated google review requests for law firms" |
| Deployability test | 6 config values, two credential entries, two webhook registrations — under 15 minutes |
| Upsell test | Quick-Win Build adds reply keyword detection, 7-day no-response follow-up, and CRM logging |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Automatic 7-day follow-up if the client never replies to the initial request
- Keyword detection for free-text replies ("great", "disappointed", etc.) alongside numeric ratings
- CRM integration — logs every review request, rating, and response in Salesmate
- Response rate dashboard — see which matters generate the most reviews and why
- Automatic Twilio STOP handler — client texts STOP and they are added to your opt-out sheet instantly

[Book a call with Protomated](https://protomated.com/book) to get started.
