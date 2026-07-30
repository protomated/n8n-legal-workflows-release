# Deploy Guide: Clio Intake Sync

**Template:** PAC-20 — C4: Intake Form to PM Contact (no retyping)
**Pillar:** Convert
**Replaces:** Manual re-typing of web form submissions into Clio
**Time to deploy:** Under 15 minutes

---

## What this workflow does

The moment a prospective client submits your website intake form, this workflow writes a clean Contact and Matter into Clio automatically — no staff member has to open the form submission and retype it in.

1. Receives the form submission via webhook.
2. Validates the payload and normalizes the practice area to match your Clio setup.
3. Searches Clio for an existing contact by email — if the same person has reached out before, the existing contact is reused instead of creating a duplicate.
4. Creates the Contact (if new) and the Matter, linked together and tagged with the practice area.
5. Emails staff a plain routing summary (and optionally posts to Slack) — who reached out, about what, with a link straight to the new Matter.
6. Optionally sends the client a short confirmation message, gated through the Bar-Compliance Guardrail so opt-outs and required disclaimers are respected.
7. Responds to the form with a success or graceful error message.

This is data entry and routing only — no legal screening, drafting, or judgment happens anywhere in this workflow.

---

## Prerequisites

**Bar-Compliance Guardrail (OPS1)** must be deployed and active before this workflow can run. It handles opt-out checking and disclaimer insertion for the optional client confirmation step.

---

## Step 1 — Get a Clio API token (3 minutes)

1. Log in to Clio at [app.clio.com](https://app.clio.com) (or [eu.app.clio.com](https://eu.app.clio.com) for EU accounts).
2. Go to **Settings → Developer App Center**.
3. Click **Create a new key** under Personal API Keys.
4. Give it a name (e.g. `n8n Intake Sync`) and click **Create**.
5. Copy the token immediately — Clio only shows it once.

> Clio personal API tokens do not expire automatically, but rotate yours every 90 days as a security best practice.

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

Skip this step if you have already set up **Email account (SMTP)** for another template in this project — it will be reused automatically.

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **SMTP** and select it.
3. Enter your firm email account settings (Gmail App Password, Microsoft 365, Zoho, or any SMTP provider).
4. Click **Save**.

---

## Step 4 — Set n8n Variables (3 minutes)

In n8n, open your project and click the **Variables** tab, then add each of the following. Skip any already set for other templates in this project.

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | `https://app.clio.com` — EU firms use `https://eu.app.clio.com` |
| `FIRM_NAME` | Your firm name |
| `FIRM_FROM_EMAIL` | The sender address for outbound emails |
| `FIRM_EMAIL` | The inbox that should receive new-intake alerts (can be the same as FIRM_FROM_EMAIL) |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow. Find it in the n8n URL when viewing that workflow: `.../workflow/12345` → enter `12345` |
| `FIRM_SLACK_WEBHOOK_URL` | Optional — leave blank to skip Slack alerts |

---

## Step 5 — Import and activate the workflow (2 minutes)

1. Open n8n and go to the **Workflows** screen.
2. Click **Add workflow → Import from file** and select [`pac-20-clio-intake-sync.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-20-clio-intake-sync.json).
3. Wire the credentials:
   - On **Search Clio For Existing Contact**, **Create Contact In Clio**, and **Create Matter In Clio**, select `Clio API (Bearer token)`.
   - On **Notify Staff Of New Intake**, **Send Confirmation Email**, **Notify Staff Of Clio Failure**, and **Notify Staff Of Unhandled Error**, select your SMTP credential.
4. Toggle the workflow to **Active**.
5. Open the **New Intake Form Submitted** node and copy the **Production URL**.

---

## Step 6 — Connect your form builder

Paste the Production URL into your form builder's webhook settings:

**Fluent Forms:** Settings → Integrations → Webhook → Add New → paste the URL → set Method to POST → set Request Format to JSON → Save Feed

**Gravity Forms:** Install the Webhooks add-on → Form Settings → Webhooks → Add New → paste the URL → Request Method POST → Request Format JSON → Save

**Other builders:** Look for a Webhook, HTTP Request, or Zapier/Make integration option and set the method to POST with JSON format.

Expected fields: `full_name`, `email`, `matter_type` (required), `phone`, `case_details`, `referral_source` (optional). Rename the fields inside **Validate And Normalize Intake Data** if your form uses different labels.

---

## Step 7 — Confirm your practice-area mapping

Open **Validate And Normalize Intake Data** and check the `MATTER_TYPE_MAP` object matches your Clio practice area names exactly (case-sensitive on some Clio instances). Add or edit entries so every option in your form's matter-type field maps to a real Clio practice area.

---

## Testing

**With the pinned test payload (no live form needed):**
Open the workflow → click **New Intake Form Submitted** → click **Execute node**. The pinned sample payload runs through the full workflow, including a real write to Clio if credentials are wired — use a Clio sandbox or a throwaway test contact for this.

**With a live form submission:**
1. Submit a test entry through your form.
2. In n8n, open the **Executions** tab and confirm the run completed.
3. Check Clio for the new Contact and Matter.
4. Confirm the staff email (and Slack, if configured) arrived.
5. If the client confirmation is enabled, confirm the email arrived with the disclaimer footer intact.

**Edge cases worth testing once:** a submission missing phone, a duplicate email (should reuse the existing contact), and an unrecognized practice area (should still create the Matter using the raw label, or fail gracefully if Clio rejects it).

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No executions after form submit | Webhook URL not pasted into the form builder | Re-copy the Production URL and re-paste |
| "missing required field" error | Form field names don't match `full_name`/`email`/`matter_type` | Update the field names read in **Validate And Normalize Intake Data** |
| Matter creation fails with practice area error | `MATTER_TYPE_MAP` entry missing or doesn't match Clio exactly | Add/fix the mapping, matching Clio's practice area name and casing |
| Duplicate contacts appearing in Clio | Email casing mismatch, or the intake used a different email than a prior submission | Dedupe matches on lowercased email exactly — check for typos or alias addresses |
| Client confirmation never sends | Recipient is opted out, or `GUARDRAIL_WORKFLOW_ID` is wrong | Check the guardrail's opt-out list and confirm the workflow ID |

---

## Done-for-you upgrade

The free template handles a single intake form into Clio. The Protomated done-for-you build adds:

- Support for multiple intake sources (phone via CallRail, chat widget, referral partner forms) into one queue
- MyCase, PracticePanther, and Docketwise as alternate practice-management targets
- Conflict-check flagging before the Matter is created
- CRM sync to Salesmate alongside the Clio write
- Custom field mapping for firm-specific Clio intake fields

Contact [Protomated](https://protomated.com) to get a quote.
