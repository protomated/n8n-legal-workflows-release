# Deploy Guide — Speed-to-Lead Router

Respond to every new web inquiry in under 60 seconds: an instant SMS to the lead, a confirmation email, and an internal alert to the duty attorney — all triggered the moment someone hits Submit on your contact form.

**Replaces:** Lawmatics, Lead Docket  
**Pillar:** Convert  
**Complexity:** Medium  
**Time to value:** ~15 minutes

---

## Prerequisites

- n8n instance running and accessible
- Bar-Compliance Guardrail workflow deployed and active (PAC-33). Get the numeric ID from the guardrail's URL: `.../workflow/WORKFLOW_ID`
- Twilio account with a phone number capable of sending SMS
- SMTP email credentials (Gmail app password, Outlook 365, Zoho, or any SMTP provider)
- At least one contact form (Fluent Forms, Gravity Forms, Typeform, etc.) ready to receive a webhook URL

---

## Step 1 — Import the workflow

In n8n, open the canvas → press `Ctrl+V` (or `Cmd+V`) and paste the full JSON from `workflows/pac-19-speed-to-lead-router.json`. The workflow imports as inactive.

---

## Step 2 — Set project Variables

Go to **project → Variables** and add the following. Variables already set for other templates in this project (FIRM_NAME, FIRM_FROM_EMAIL, etc.) are shared automatically — skip those if they are already present.

| Variable | Example value | Notes |
|---|---|---|
| `FIRM_NAME` | `Smith Law Firm` | Your firm name as it appears in texts and emails |
| `FIRM_FROM_EMAIL` | `intake@smithlawfirm.com` | Sender address for outbound emails |
| `FIRM_EMAIL` | `intake@smithlawfirm.com` | Destination for internal alerts (can be the same) |
| `FIRM_TWILIO_NUMBER` | `+13125550100` | Your Twilio sending number in E.164 format |
| `FIRM_BOOKING_URL` | `https://calendly.com/smithlaw/consult` | Your consult booking link |
| `GUARDRAIL_WORKFLOW_ID` | `42` | Numeric ID of your deployed Bar-Compliance Guardrail |
| `DUTY_ATTORNEYS` | `Jane Smith\|jane@firmname.com\|+13125550101,Tom Lee\|tom@firmname.com\|+13125550102` | Pipe-separated within each entry, comma-separated between entries |
| `FIRM_SLACK_WEBHOOK_URL` | `https://hooks.slack.com/services/…` | Optional — leave blank to skip Slack alerts |

### DUTY_ATTORNEYS format

Each attorney entry follows this pattern: `Name|email|phone`

- Use `|` (pipe) to separate the three fields within one attorney
- Use `,` (comma) to separate multiple attorneys
- Phone numbers should be in E.164 format: `+13125550101`

**Single attorney:**
```
Jane Smith|jane@smithlawfirm.com|+13125550101
```

**Two attorneys (leads alternate between them):**
```
Jane Smith|jane@smithlawfirm.com|+13125550101,Tom Lee|tom@smithlawfirm.com|+13125550102
```

The workflow uses n8n staticData to remember which attorney received the last lead. The rotation resets only if you re-save the workflow with a code change.

---

## Step 3 — Wire credentials

In **project → Credentials**, add:

1. **Twilio account** — New → search Twilio → enter Account SID and Auth Token → save as `Twilio account`
2. **Email account (SMTP)** — New → search SMTP → enter your SMTP settings → save as `Email account (SMTP)`

The workflow references these credentials by name. If you have already set them up for another template in this project, they are reused automatically.

---

## Step 4 — Activate and copy the webhook URL

1. Toggle the workflow **Active**
2. Open the **When Lead Form Submitted** node
3. Copy the **Production URL** — it looks like `https://your-n8n.example.com/webhook/new-lead`

---

## Step 5 — Connect your form builder

Paste the Production URL into your form builder's webhook settings:

**Fluent Forms:**
Go to Settings → Integrations → Webhook → Add New → paste the URL → set Method to POST → set Request Format to JSON → Save Feed

**Gravity Forms:**
Install the Webhooks add-on → Form Settings → Webhooks → Add New → paste the URL → Request Method POST → Request Format JSON → Save

**Typeform:**
Connect → Webhooks → Add a webhook → paste the URL → Save

**Other builders:**
Look for a Webhook, HTTP Request, or Zapier/Make integration option. Set the method to POST and format to JSON.

---

## Step 6 — Test

1. Submit a test entry through your form with a real phone number you can receive texts on
2. In n8n, open the **Executions** tab and watch the run complete
3. Verify all three outputs:
   - Lead receives an SMS within 60 seconds (check your phone)
   - Lead receives a confirmation email
   - Duty attorney receives an alert email with the lead's details

**To test with the pinned payload (no live form needed):**
Open the workflow → click **When Lead Form Submitted** → click **Execute node** — the pinned Fluent Forms payload runs through the full workflow.

---

## Step 7 — Optional Slack setup

To receive Slack alerts when a new lead arrives:

1. Go to Slack → your workspace → **Apps** → search **Incoming Webhooks** → **Add to Slack**
2. Choose the channel where you want lead alerts to appear
3. Copy the Webhook URL
4. Paste it into `FIRM_SLACK_WEBHOOK_URL` in project → Variables

The attorney alert email fires whether or not Slack is configured — Slack is additive.

> **Microsoft Teams users:** Teams uses a different webhook payload format and is not supported in the free template. The Quick-Win Build adds a Teams alert channel alongside Slack.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No executions appear after form submit | Webhook URL not pasted correctly into form builder | Re-copy the Production URL from the webhook node and re-paste |
| SMS not received | Twilio credential not wired, or FIRM_TWILIO_NUMBER wrong format | Check Twilio credential; ensure number starts with +1 |
| "DUTY_ATTORNEYS is not set" error | Variable missing or blank | Add the variable in project → Variables tab |
| Email sends but attorney name is wrong | DUTY_ATTORNEYS format issue | Check for correct pipe `|` separators and no extra spaces |
| Compliance check errors | GUARDRAIL_WORKFLOW_ID wrong, or guardrail not active | Confirm the numeric ID and that the guardrail is toggled Active |

---

## What the paid Build adds

The free template handles a single firm's contact form. The Quick-Win Build (from $5k) adds:

- CRM write-back to Salesmate on every new lead
- Clio matter creation from lead data
- Microsoft Teams alert channel (in addition to Slack)
- Multi-channel intake: phone calls via CallRail plus form submissions in a single queue
- Priority routing — high-value practice areas jump the round-robin queue
- Lead scoring based on form answers
- Follow-up sequence if the lead does not book within 24 hours
