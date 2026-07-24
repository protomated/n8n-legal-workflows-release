# Deploy Guide: Missed-Call Instant Text-Back
**NTC-14 | Convert | Free Template**
Target keyword: missed call text back for law firms

Get value in under 15 minutes. When someone calls your firm and you miss it, they instantly receive a text and you get an email — before they dial the next lawyer.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** The text-back goes through it for opt-out checking and TCPA disclaimer verification before sending. Complete the NTC-33 deploy guide before continuing here.
- An n8n instance (self-hosted or n8n Cloud)
- A Twilio account with an SMS-enabled phone number (for sending the text-back)
- An email account with SMTP access (Gmail, Outlook 365, Zoho Mail, or similar — for firm alerts)
- A Twilio **or** CallRail account to receive the incoming call events

> **CallRail users:** You still need a Twilio account to send the outbound text. CallRail handles call tracking; Twilio handles the outbound SMS. Twilio numbers start at $1/month and texts cost about $0.0075 each.

---

## Step 1 — Import the workflow (1 min)

1. Open your n8n canvas.
2. Press `Ctrl+V` (or `Cmd+V` on Mac) and paste the contents of [`ntc-14-missed-call-text-back.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/ntc-14-missed-call-text-back.json).
3. The workflow appears. Do not activate it yet.

---

## Step 2 — Set n8n Variables (2 min)

In n8n, open your project and click the **Variables** tab, then add the following variables. Set them once — every workflow that uses these values picks them up automatically.

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name as it appears in texts (e.g. `Smith Law Firm`) |
| `FIRM_EMAIL` | The email address where missed-call alerts should be sent |
| `FIRM_TWILIO_NUMBER` | Your Twilio number in E.164 format (e.g. `+15550001234`) |
| `FIRM_TIMEZONE` | Your state's timezone (e.g. `America/New_York`, `America/Chicago`, `America/Los_Angeles`) |
| `FIRM_BOOKING_URL` | Your scheduling link (Calendly, Acuity, your website's contact page, etc.) |
| `FIRM_BUSINESS_HOURS_START` | Your office opening hour in 24-hour format (e.g. `9` for 9 AM) |
| `FIRM_BUSINESS_HOURS_END` | Your office closing hour in 24-hour format (e.g. `17` for 5 PM) |
| `FIRM_WORK_DAYS` | Comma-separated weekday numbers (`1,2,3,4,5` = Monday–Friday; 0=Sunday, 6=Saturday) |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33: `.../workflow/WORKFLOW_ID` |

---

## Step 3 — Add credentials (5 min)

### Twilio credential (for sending texts)
1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **Twilio** and select it.
3. Enter your **Account SID** and **Auth Token** (both visible on your Twilio Console dashboard at console.twilio.com).
4. Save it as `Twilio account`.

### Email credential (for firm alert emails)
1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **SMTP** and select it.
3. Enter your email provider's SMTP settings:

   **Gmail:**
   - Host: `smtp.gmail.com`, Port: `465`, Security: SSL
   - Username: your Gmail address
   - Password: an App Password (Google Account → Security → 2-Step Verification → App Passwords — use "Mail" + "Other")

   **Outlook / Microsoft 365:**
   - Host: `smtp.office365.com`, Port: `587`, Security: STARTTLS
   - Username: your email address, Password: your password
   - Note: SMTP AUTH must be enabled in your Microsoft 365 admin centre

   **Zoho Mail:**
   - Host: `smtp.zoho.com`, Port: `465`, Security: SSL
   - Username: your Zoho email, Password: your Zoho password

4. Save it as `Email account (SMTP)`.

---

## Step 4 — Verify n8n Variables are live (1 min)

No code nodes need editing — firm settings are read from the project **Variables** tab (set in Step 2). To verify, open the **Compose Text-Back** node and confirm the code reads `$vars.FIRM_NAME` (not a hardcoded string). If you see a hardcoded string, you are running an older version of the workflow — re-import from the latest JSON.

---

## Step 5 — Wire up your call provider (2 min)

### If you use Twilio

1. Click **When Call Received** and copy the **Production URL** (shown after you activate the workflow).
2. In the Twilio Console, go to your phone number's settings.
3. Under **Voice & Fax → A call comes in**, set the webhook to the copied URL.
4. Under **Call Status Changes**, paste the same URL.
5. Twilio will POST to your workflow on every status change, including answered calls. The **If Valid Missed Call** node automatically filters those out — only `no-answer`, `busy`, `failed`, and `canceled` statuses proceed to the text-back.

### If you use CallRail

1. In CallRail go to **Settings → Notifications → Webhooks**.
2. Add a new webhook and paste the **Production URL** from the **When Call Received** node.
3. Select the **Missed Call** event type.
4. Save.

---

## Step 6 — Activate and test

1. Toggle the workflow **Active** in n8n.
2. **Pinned test (fastest):** The workflow ships with a pinned Twilio `no-answer` payload on the **When Call Received** node. Click the node, then **Test step** — n8n will run the pinned data through the full flow without waiting for a real call.

3. Confirm the golden path:
   - **If Valid Missed Call** routes to the Yes branch (Check After Hours).
   - **Send Text Message** shows a successful execution with the correct message text.
   - **Text delivered?** routes to the Yes branch (confirmation ID present in Twilio response).
   - You receive the firm alert email with "Text-back: Sent ✓".

4. **Negative test — guard node:** Send a POST with a `completed` status to confirm the guard works:

```json
{
  "CallStatus": "completed",
  "Caller": "+15559998888",
  "CallerName": "Jane Prospect",
  "Called": "+15550001234"
}
```

   Execution should route to **Ignore Non-Missed Call** with no text sent.

5. **Quick test (CallRail):** Send a POST with:

```json
{
  "caller_number": "+15559998888",
  "caller_name": "Jane Prospect",
  "business_phone_number": "+15550001234"
}
```

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Call missed during business hours | Compliance check runs → text sent: "Hi Jane, this is Smith Law Firm. We missed your call and want to help. Book a free consult: [link] — Reply STOP to opt out." |
| Call missed after hours or weekend | Compliance check runs → text sent with after-hours copy |
| Caller is on the opt-out list | Text suppressed — **Skip — contact opted out**; suppression logged to audit sheet; firm email still sends |
| Twilio API error (wrong credentials, invalid number, etc.) | **Text not delivered.** The **Text delivered?** node routes No; firm receives: "Text-back: FAILED — please call this person back manually." |
| Call answered (`completed`) via Twilio StatusCallback | **No text sent.** The **If Valid Missed Call** node routes to **Ignore Non-Missed Call**. |
| No caller number in payload | **No text sent.** Same guard — event is treated as invalid when the caller number is missing. |
| Bar-Compliance Guardrail audit log write fails | Send decision still returned — failure does not block or unblock the message |

The firm always gets an email alert with the caller's name, number, timestamp, and the message sent (or a failure notice) — but only for genuine missed calls.

---

## Compliance note

This template is operational only — it does not touch legal work (ABA Op. 512). Every outbound text passes through the Bar-Compliance Guardrail before sending: opt-out status is checked, TCPA disclaimer language is confirmed present, and the attempt is logged to your audit sheet regardless of whether the message is sent or suppressed.

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- CRM integration (Salesmate lead creation on first reply)
- Multi-touch follow-up sequence (SMS + email + voicemail drop)
- Live dashboard showing recovery rate and booked revenue
- Automatic Twilio STOP handler — caller texts STOP and they are added to your opt-out sheet instantly

[Book a call with Protomated](https://protomated.com/book) to get started.
