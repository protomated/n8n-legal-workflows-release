# Deploy Guide: Content Repurposer

**Template:** PAC-31 — Content Repurposer  
**Pillar:** Authority  
**Replaces:** Marketing agency retainer ($500–$2,000/month)  
**Time to deploy:** Under 15 minutes

---

## What this workflow does

You submit one field note or case win through a webhook, and this workflow uses AI to produce four ready-to-post marketing assets:

| Asset | Where it goes |
|---|---|
| LinkedIn post | Copy into LinkedIn (350–400 words) |
| Newsletter blurb | Drop into your email newsletter |
| Google Business Profile post | Paste into your GBP dashboard |
| FAQ entry | Add to your website FAQ section |

All four assets arrive in a single email to your firm inbox within seconds. Each asset is flagged **READY TO PUBLISH** or **REVIEW FIRST** — flag means the AI spotted language that warrants a human read before posting.

**Safety guardrails built in:** The workflow never drafts legal advice, case analysis, or client-identifying content. It produces marketing copy only. Case-win assets automatically include the required "Attorney Advertising. Past results do not guarantee future outcomes." disclosure.

Every run is logged by the Bar-Compliance Guardrail for your audit record.

---

## Prerequisites

**Bar-Compliance Guardrail** must be deployed and active before this workflow can run. The guardrail provides the audit log for every content generation run.

---

## Step 1 — Get an Anthropic API key (3 minutes)

1. Go to [console.anthropic.com](https://console.anthropic.com) and sign in or create an account.
2. Click **API Keys** in the left sidebar.
3. Click **Create Key**, give it a name (e.g. `n8n Content Repurposer`), and click **Create Key**.
4. Copy the key immediately — it is only shown once.

> **Billing note:** This workflow calls Claude Sonnet on each run. Typical cost is under $0.05 per content package (four assets). Add a credit card and set a monthly spending limit in the Anthropic console.

---

## Step 2 — Create the Anthropic credential in n8n (2 minutes)

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **Header Auth** and select it.
3. Fill in:
   - **Name:** `Anthropic API key`
   - **Header Name:** `x-api-key`
   - **Header Value:** paste your API key from Step 1
4. Click **Save**.

---

## Step 3 — Create the SMTP email credential (2 minutes)

The workflow emails the four-asset package to your firm inbox.

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **SMTP** and select it.
3. Enter your firm email account settings:

   **Gmail:** Use an App Password (Google Account → Security → App Passwords). Your regular Gmail password will not work here.

   **Microsoft 365 / Outlook:** Use your email address and password. Confirm that SMTP AUTH is enabled in the Microsoft 365 admin center (Admin → Settings → Mail → SMTP AUTH).

   **Other providers:** Check your provider's documentation for SMTP host, port (usually 587 with STARTTLS), and credentials.

4. Click **Save**.

---

## Step 4 — Set n8n Variables (3 minutes)

In n8n, open your project and click the **Variables** tab, then add each of the following:

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your firm name as it should appear in marketing copy (e.g. `Smith & Associates`) |
| `FIRM_EMAIL` | The inbox where content packages will be delivered (e.g. `marketing@smithlaw.com`) |
| `FIRM_FROM_EMAIL` | The sender address shown on the delivery email (can be the same as `FIRM_EMAIL`) |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow. Find it in the n8n URL when viewing that workflow: `.../workflow/12345` → enter `12345` |

---

## Step 5 — Import and activate the workflow (3 minutes)

1. Open n8n and go to the **Workflows** screen.
2. Click **Add workflow → Import from file** and select [`pac-31-content-repurposer.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-31-content-repurposer.json).
3. Wire the credentials:
   - On the **Generate Marketing Content** node, select `Anthropic API key`.
   - On the **Email Content Package to Firm** node, select your SMTP credential.
4. Click **Save**, then toggle the workflow to **Active**.

---

## Testing

The workflow ships with a pinned test payload — a commercial eviction case win, anonymized, with no client names. Use it to verify the full flow without needing a real submission:

1. Open the **Content Input** node on the canvas.
2. The pinned payload is already loaded. Click **Execute workflow** from the top toolbar.
3. Watch each node run. The expected path is:
   - **Read & Validate Input** → valid
   - **Generate Marketing Content** → four assets returned from Claude
   - **Extract & Safety-Check Content** → all four assets flagged READY TO PUBLISH (test payload contains no legal language)
   - **Compliance Check — Audit Log** → approved
   - **Email Content Package to Firm** → email delivered to `FIRM_EMAIL`
4. Check your firm inbox for the content package email. It will contain all four assets with word counts and publish-readiness flags.

If the email does not arrive, check the **Email Content Package to Firm** node output for SMTP errors.

---

## Submitting real content

The **Content Input** webhook accepts JSON `POST` requests at the path `/content-repurposer`. Required fields:

| Field | Description | Example |
|---|---|---|
| `content` | Your field note or case win (20+ characters, plain text) | `"Helped a landlord recover a commercial space from a non-paying tenant in 45 days..."` |
| `content_type` | `"field_note"` or `"case_win"` | `"case_win"` |
| `practice_area` | The practice area label for the generated copy | `"business law"` |

Optional:
| Field | Description |
|---|---|
| `firm_name_override` | Override `FIRM_NAME` for this run only |

**Submit via curl:**
```bash
curl -X POST https://YOUR_N8N_URL/webhook/content-repurposer \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Your field note or case win here.",
    "content_type": "case_win",
    "practice_area": "business law"
  }'
```

**Submit via Fluent Forms:** Add a hidden field `content_type` with value `case_win` and map your textarea to `content`. Set the form's webhook URL to the endpoint above.

> **Case wins require client consent.** Before submitting a case win, confirm the client has given written permission to reference the matter in marketing materials, even without identifying them by name.

---

## Understanding the output email

Each content package email contains four sections. Each section shows:

- The asset text, ready to copy and paste
- A word count
- A flag: **READY TO PUBLISH** or **REVIEW FIRST**

**REVIEW FIRST** means the AI-powered keyword scan detected language that could be misread as legal advice. Read that asset before posting and revise any flagged phrase.

Case-win assets always include the line: *"Attorney Advertising. Past results do not guarantee future outcomes."* — required by most state bars for any marketing that references a client outcome.

---

## Customisation

**Change the AI model:** In the **Build Content Prompt** node, update the `model` field. The default is `claude-sonnet-4-6`. Swap to `claude-haiku-4-5-20251001` to reduce cost; swap to `claude-opus-4-7` for maximum copy quality.

**Adjust tone or length:** In the **Build Content Prompt** node, edit the `MARKETING_PROMPT` template. The prompt currently asks for a professional-but-approachable tone. You can add firm-specific style instructions here.

**Add more practice areas:** No configuration needed — `practice_area` is a free-text field in each submission. The generated copy uses whatever label you send.

**Change the output format:** Edit the **Extract & Safety-Check Content** node to restructure the `email_body` variable. The current format groups assets under bold headers. You could output each asset as a separate paragraph or add HTML formatting if your SMTP credential supports HTML email.

---

## Done-for-you upgrade

The free template emails content to your inbox and requires you to post manually. The Protomated done-for-you build adds:

- **Direct scheduling** — posts go directly to LinkedIn and Google Business Profile on a publishing calendar
- **Brand voice calibration** — copy trained on your existing content and reviewed by a marketing strategist
- **Practice-area playbooks** — custom prompt packs per practice area (PI, family law, estate planning, etc.)
- **Monthly content calendar** — 20 posts/month generated from your matter notes automatically
- **State bar compliance review** — copy reviewed against your jurisdiction's advertising rules before publishing

Contact [Protomated](https://protomated.com) to get a quote.
