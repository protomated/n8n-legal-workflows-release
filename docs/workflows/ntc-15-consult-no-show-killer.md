# Deploy Guide: Consult No-Show Killer
**NTC-15 | Convert | Free Template**
Target keyword: reduce law firm consultation no-shows

Get value in under 15 minutes. When a prospect books a consult, they instantly
receive a confirmation and two timed reminders — by both text and email — plus a
reschedule link if they miss the meeting, all without lifting a finger.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first.** Every outbound text in this workflow passes through it for opt-out checking and TCPA disclaimer attachment. Complete the NTC-33 deploy guide before continuing here.
- An n8n instance (self-hosted with **Postgres** backend, or n8n Cloud)
- A Twilio account with an SMS-enabled phone number (for outbound texts)
- An email account with SMTP access (Gmail, Outlook 365, Zoho Mail, or similar)
- A Calendly account **or** any booking tool that sends webhooks
  (Acuity, Cal.com, Fluent Forms, etc.)

> **Self-hosted Postgres requirement:** The Wait nodes suspend execution for up
> to 24+ hours. n8n must be backed by Postgres — not the default SQLite — for
> suspended executions to survive a restart. Run `docker compose up -d` with the
> provided `docker-compose.yml` (which uses Postgres) to satisfy this. n8n Cloud
> handles persistence automatically.

---

## Step 1 — Import the workflow (1 min)

1. Open your n8n canvas.
2. Press `Ctrl+V` (or `Cmd+V` on Mac) and paste the contents of
   [`ntc-15-consult-no-show-killer.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/ntc-15-consult-no-show-killer.json).
3. The workflow appears. Do not activate it yet.

---

## Step 2 — Set n8n Variables (2 min)

In n8n, open your project and click the **Variables** tab, then add the following variables. Set them once — every workflow that uses these values picks them up automatically.

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name as it appears in messages |
| `FIRM_EMAIL` | Your intake email address (firm-side copies of reminders go here) |
| `FIRM_FROM_EMAIL` | The address reminder emails are sent from — must match an address your SMTP account is authorised to send from |
| `FIRM_TWILIO_NUMBER` | Your Twilio number in E.164 format (e.g. `+15550001234`) |
| `FIRM_TIMEZONE` | Your state's timezone (e.g. `America/New_York`, `America/Chicago`, `America/Los_Angeles`) |
| `FIRM_BOOKING_URL` | Your scheduling link, used as the reschedule link when your booking tool doesn't include one per booking |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33: `.../workflow/WORKFLOW_ID` |

---

## Step 3 — Add credentials (5 min)

### Twilio credential (for sending texts)

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **Twilio** and select it.
3. Enter your **Account SID** and **Auth Token** (both in your Twilio Console
   dashboard under Account Info).
4. Save it as `Twilio account`.

### Email credential (for confirmations and reminders)

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **SMTP** and select it.
3. Enter your email provider's settings:

   **Gmail:**
   - Host: `smtp.gmail.com`, Port: `465`, Security: SSL
   - Username: your Gmail address
   - Password: an App Password — not your regular password
     (Google Account → Security → 2-Step Verification → App Passwords,
     select "Mail" and "Other")

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

No code nodes need editing — firm settings are read from the project **Variables** tab (set in Step 2). To verify, open the **Set Up Messages** node and confirm the code reads `$vars.FIRM_NAME` (not a hardcoded string). If you see a hardcoded string, you are running an older version of the workflow — re-import from the latest JSON.

`FIRM_FROM_EMAIL` is the address reminder emails come from. It must be an address your
SMTP account is authorised to send from — for most firms this is the same as
`FIRM_EMAIL` (e.g. `intake@smithlaw.com`).

`FIRM_BOOKING_URL` is used as the reschedule link when your booking tool does not
include a per-booking reschedule URL in its webhook. Calendly always includes
one; other tools may not.

---

## Step 5 — Wire up your booking tool (3 min)

### If you use Calendly

Calendly v2 does not let you paste a webhook URL through its UI — you register
it via API.

**Step 5a — Get your Calendly token and organisation URI**

1. In Calendly go to **Integrations → API & Webhooks → Personal Access Tokens**
   and create a token.
2. Run this to get your organisation URI:

```bash
curl -s https://api.calendly.com/users/me \
  -H "Authorization: Bearer YOUR_TOKEN" | jq '.resource.current_organization'
```

Copy the value — it looks like
`https://api.calendly.com/organizations/XXXXXX`.

**Step 5b — Activate the workflow in n8n first**

Calendly validates the webhook URL before saving the subscription, so the
workflow must be active before you register.

1. In n8n toggle the workflow **Active**.
2. Open the **When Booking Received** node and copy the **Production URL**
   (e.g. `https://your-n8n.com/webhook/consult-booking`).

**Step 5c — Register the webhook subscription**

```bash
curl -s -X POST https://api.calendly.com/webhook_subscriptions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "url": "https://your-n8n.com/webhook/consult-booking",
    "events": ["invitee.created"],
    "organization": "https://api.calendly.com/organizations/XXXXXX",
    "scope": "organization"
  }'
```

A `201 Created` response confirms success.

> **Tip:** Make sure your booking form **requires** a phone number field.
> In Calendly go to **Event Types → your event → Edit → Invitee Questions**,
> add a phone number question, and mark it required. This populates the phone
> number in the webhook payload so the text reminders can reach the prospect.

### If you use Acuity, Cal.com, or another tool

1. Toggle the workflow **Active** in n8n.
2. Open **When Booking Received** and copy the **Production URL**.
3. In your booking tool, find **Webhooks** or **Notifications** and paste the
   URL.
4. Select the **new booking created** event. The exact label varies by tool:
   - Acuity: *New Appointment*
   - Cal.com: *booking.created*
   - Fluent Forms: *Form Submission*
5. Map your tool's fields to what the **Normalize and Validate Booking** node expects
   (see the node notes for the generic field names: `contact_name`,
   `contact_email`, `contact_phone`, `appointment_time`).

---

## Step 6 — Test the workflow (5 min)

### Golden path test (pinned data)

The workflow ships with a pinned Calendly `invitee.created` payload on the
**When Booking Received** node.

1. Click **When Booking Received**, then **Test step** — n8n runs the pinned data
   through the full chain without waiting for a real booking.
2. Step through manually:
   - **Normalize and Validate Booking** → `is_valid_booking: true`
   - **If Valid Booking** → Yes branch
   - **Set Up Messages** → inspect `confirm_sms`, `reminder_24h_at`,
     `reminder_1h_at`, `noshow_check_at`
   - **Compliance Check — Confirmation Text** → calls guardrail, returns `approved: true`
   - **If Approved to Send Confirmation** → Yes branch
   - **Send Confirmation Text** → Twilio returns a `sid`
   - **Send Confirmation Email** → email sends successfully
3. The Wait nodes pause execution — you won't see the reminders fire during a
   manual test. Verify the Wait nodes show a future ISO timestamp in their
   `dateTime` field.

### Cancellation guard test

Send a POST to the webhook URL with a cancellation event:

```json
{
  "event": "invitee.canceled",
  "payload": {
    "invitee": { "email": "jane@example.com", "name": "Jane Smith" },
    "event": {
      "start_time": "2026-06-27T19:00:00Z",
      "name": "30-Minute Consultation"
    }
  }
}
```

Execution should route to **Ignore Invalid Booking** with no messages sent.

### Generic webhook test

Send a POST to the webhook URL with:

```json
{
  "contact_name": "Bob Jones",
  "contact_email": "bob@example.com",
  "contact_phone": "+15559998888",
  "appointment_time": "2026-06-28T15:00:00Z",
  "consult_type": "Initial Consultation"
}
```

Execution should proceed to **Set Up Messages** → Confirmation Text →
Confirmation Email → Wait.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Prospect books via Calendly | Compliance check runs → confirmation text + email sent instantly; 24h + 1h reminders queued with their own compliance checks |
| Prospect is on the opt-out list at booking | Confirmation text suppressed — **Skip — opted out (confirmation)**; entire reminder sequence stops; suppression logged to audit sheet |
| Prospect opts out between reminders | Next reminder's compliance check suppresses that text — remaining sequence stops |
| Prospect books with no phone number | Text steps fail gracefully; confirmation and reminder emails still send |
| Booking received less than 24h before appointment | 24h reminder fires almost immediately; 1h reminder and reschedule nudge fire on schedule |
| Calendly cancellation event arrives | No messages sent — routes to **Ignore Invalid Booking**. Any running reminder sequences for this booking must be stopped manually in n8n → Executions. |
| n8n restarts while waiting | **Postgres:** execution resumes correctly. **SQLite:** execution may be lost — do not use SQLite for this workflow. |
| Prospect replies STOP to any text | Twilio blocks all further texts to that number automatically. Add them to your opt-out sheet too so the guardrail catches them across all templates. |
| Bar-Compliance Guardrail audit log write fails | Send decision still returned — failure does not block or unblock the message |

---

## Known limitations

- **No show/no-show detection.** The reschedule nudge fires 15 minutes after the
  appointment start regardless of whether the prospect attended. The copy is
  written neutrally ("if anything came up") to avoid alienating clients who did
  show. The done-for-you upsell adds Clio/MyCase integration to confirm show
  status before sending.
- **Cancellation does not stop the sequence.** If a prospect cancels, the
  reminder workflow keeps running. Stop it manually in n8n → Executions → find
  the execution → Stop.

---

## Compliance note

This template is operational only — it does not touch legal work (ABA Op. 512).
Every outbound text passes through the Bar-Compliance Guardrail before sending:
opt-out status is checked, TCPA disclaimer language is confirmed present, and the
attempt is logged to your audit sheet. If a client opts out between reminders,
the next guardrail check suppresses that message and stops the remaining sequence.

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:

- Clio / MyCase integration to detect genuine no-shows before sending the
  reschedule nudge
- Automatic sequence cancellation when Calendly fires a cancellation event
- CRM sync (Salesmate) on first reply — no-show becomes a follow-up task
  automatically
- Multi-channel follow-up after no-show (voicemail drop + email + text drip)
- Automatic Twilio STOP handler — prospect texts STOP and they are added to your opt-out sheet instantly

[Book a call with Protomated](https://protomated.com/book) to get started.
