# Deploy Guide: Invoice Reminder & Collections Ladder

**Template:** PAC-18 — B1: Invoice Reminder & Collections Ladder  
**Pillar:** Keep  
**Replaces:** LeanLaw AR follow-up, manual collections outreach  
**Time to deploy:** Under 15 minutes

---

## What this workflow does

Every morning at 8am, this workflow checks your Clio account for overdue invoices and sends an escalating series of payment reminders to each client. The ladder runs automatically:

| Day overdue | Action |
|---|---|
| 1–2 | Friendly first reminder with pay link |
| 7–8 | Second reminder |
| 14–15 | Third notice |
| 30–31 | Final notice — collection warning |
| 60+ | Partner escalation alert (internal email) |

Reminders stop automatically when a client pays — Clio removes paid invoices from the overdue set, so no payment-detection logic is needed. Clients who have opted out are suppressed by the compliance guardrail and their suppression is logged.

---

## Prerequisites

**NTC-33 Bar-Compliance Guardrail** must be deployed and active before this workflow can run. The guardrail handles opt-out checking, unsubscribe footer insertion, and audit logging for all client-facing messages.

---

## Step 1 — Get a Clio API token (3 minutes)

1. Log in to Clio at [app.clio.com](https://app.clio.com) (or [eu.app.clio.com](https://eu.app.clio.com) for EU accounts).
2. Go to **Settings → Developer App Center**.
3. Click **Create a new key** under Personal API Keys.
4. Give it a name (e.g. `n8n Invoice Reminder`) and click **Create**.
5. Copy the token immediately — Clio only shows it once.

> **Note:** Clio personal API tokens do not expire automatically, but rotate yours every 90 days as a security best practice. Set a recurring calendar reminder.

---

## Step 2 — Create the Clio credential in n8n (2 minutes)

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **Header Auth** and select it.
3. Fill in:
   - **Name:** `Clio API (Bearer token)`
   - **Header Name:** `Authorization`
   - **Header Value:** `Bearer YOUR_TOKEN_HERE` (paste your token from Step 1)
4. Click **Save**.

---

## Step 3 — Create the SMTP email credential (2 minutes)

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **SMTP** and select it.
3. Enter your firm email account settings. Common setups:

   **Gmail:** Use an App Password (Google Account → Security → App Passwords). Your regular Gmail password will not work.

   **Microsoft 365 / Outlook:** Use your email address and password. Make sure SMTP AUTH is enabled in the Microsoft 365 admin center.

   **Other providers:** Check your provider's documentation for SMTP host, port, and authentication settings.

4. Click **Save**.

---

## Step 4 — Set n8n Variables (3 minutes)

In n8n, open your project and click the **Variables** tab, then add each of the following:

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your firm name as it should appear in emails (e.g. `Smith & Associates`) |
| `FIRM_EMAIL` | Your firm's general or billing inbox (e.g. `billing@smithlaw.com`) |
| `FIRM_FROM_EMAIL` | The sender address clients will see (can be the same as FIRM_EMAIL) |
| `FIRM_PARTNER_EMAIL` | The managing partner's email for 60-day escalation alerts |
| `FIRM_PAY_LINK` | Your payment portal URL (LawPay, CPACharge, etc.) — used as a fallback if Clio does not return a per-invoice pay link |
| `CLIO_BASE_URL` | `https://app.clio.com` — EU firms use `https://eu.app.clio.com` |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the NTC-33 workflow. Find it in the n8n URL when viewing that workflow: `.../workflow/12345` → enter `12345` |

---

## Step 5 — Import and activate the workflow (2 minutes)

1. Open n8n and go to the **Workflows** screen.
2. Click **Add workflow → Import from file** and select [`pac-18-invoice-reminder-ladder.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-18-invoice-reminder-ladder.json).
3. Wire the credentials:
   - On the **Fetch Overdue Invoices from Clio** node, select `Clio API (Bearer token)`.
   - On the **Alert Partner** and **Send Invoice Reminder** nodes, select your SMTP credential.
4. Click **Save**, then toggle the workflow to **Active**.

The workflow will run automatically at 8am each day.

---

## Testing

Run the workflow manually on its first day to confirm everything is wired up:

1. Click **Execute workflow** in the top-right of the canvas.
2. The **Fetch Overdue Invoices from Clio** node will return live overdue invoices from your Clio account.
3. Check the output of **Normalize & Triage Invoices** to see which invoices hit a ladder step today.
4. Check your inbox and your partner's inbox for the expected emails.

If you want to test without sending real emails, set **FIRM_EMAIL** and **FIRM_PARTNER_EMAIL** temporarily to your own address before running.

---

## Customisation

**Change the send time:** Edit the cron expression on the **Daily Invoice Check** node. The default `0 8 * * *` means 8am every day in n8n server time.

**Add more ladder steps:** Extend the `steps` object in the **Build Client Reminder Email** node and add a new day range in **Normalize & Triage Invoices**.

**Change the partner escalation threshold:** Update the `>= 60` comparison in **Normalize & Triage Invoices** and the `day60` condition in the **Partner escalation?** IF node.

---

## Done-for-you upgrade

The free template sends email reminders only. The Protomated done-for-you build adds:

- **SMS reminders** via Twilio alongside every email step
- **LawPay / CPACharge** integration for one-click payment from the SMS
- **Payment plan routing** — clients who respond requesting a plan are flagged in Clio
- **QuickBooks sync** — payments reconciled automatically
- **Customised reminder copy** reviewed by your state bar compliance team

Contact [Protomated](https://protomated.com) to get a quote.
