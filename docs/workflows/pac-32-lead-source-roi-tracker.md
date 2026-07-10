# Deploy Guide: Lead-Source ROI Tracker

**Template:** PAC-32 — Lead-Source ROI Tracker  
**Pillar:** Get  
**Replaces:** CallRail Analytics ($45–145/mo) + manual spreadsheet tracking (~3 hrs/week)  
**Time to deploy:** Under 15 minutes

---

## What this workflow does

Every inbound lead — web form, ad click, referral, directory listing, or phone call — hits a single webhook URL. The workflow reads wherever the lead came from, tags it with a canonical channel name, creates a contact in Salesmate with source tags, and appends a row to your Google Sheets ROI dashboard.

| Step | What happens |
|---|---|
| Lead arrives | Fluent Form, CallRail call, or any web form posts to the webhook |
| Source detected | Channel assigned: Google Ads, referral, legal directory, organic, social, etc. |
| Contact created | Salesmate contact tagged `lead-source-google-ads`, `matter-family-law`, etc. |
| Dashboard row logged | Date, channel, source detail, contact info, and Salesmate link written to Google Sheets |
| Acknowledgment sent | Prospect receives a brief "we received your inquiry" email (opt-out compliant) |
| Firm notified | Your intake email receives a full lead summary with a direct Salesmate link |

You then fill in three manual columns — ad spend, signed case (yes/no), and case value — to see cost-per-lead and cost-per-signed-case by channel in a pivot table.

**Prerequisite:** The Bar-Compliance Guardrail must be deployed and active before this workflow can run. The guardrail handles opt-out checking, disclaimer insertion, and compliance logging for the acknowledgment email.

---

## Step 1 — Create the Lead Log Google Sheet (3 minutes)

1. Open Google Sheets and create a new spreadsheet.
2. Name the spreadsheet: `Lead Source ROI Dashboard`
3. Rename the default tab (bottom of the screen) to exactly: `Lead Log`
4. In row 1, enter these column headers — one per cell, left to right:

```
date    channel    source_detail    lead_name    lead_email    lead_phone    matter_type    message_preview    salesmate_contact_id    ad_spend (manual)    signed_case (manual)    case_value (manual)
```

> **Exact match required.** The workflow writes to the first nine columns by name. If the headers differ, the append will fail silently. Copy the headers above and paste them into row 1.

5. Copy the Sheet ID from the URL bar:
   ```
   https://docs.google.com/spreadsheets/d/SHEET_ID/edit
   ```
   You'll need it in Step 5.

---

## Step 2 — Create the Salesmate API credential (2 minutes)

1. In Salesmate, click your profile icon (top right) → **My Account** → **Developer** → **Personal API Token**.
2. Copy the token.
3. In n8n, open your project and click the **Credentials** tab → **New credential**.
4. Search for **Header Auth** and select it.
5. Fill in:
   - **Name:** `Salesmate API`
   - **Header Name:** `accessToken`
   - **Header Value:** paste your Personal API Token
6. Click **Save**.

---

## Step 3 — Create the Google Sheets credential (2 minutes)

1. In n8n, open your project and click the **Credentials** tab → **New credential**.
2. Search for **Google Sheets OAuth2** and select it.
3. Click **Sign in with Google** and complete the OAuth flow using the Google account that owns the Lead Source ROI Dashboard sheet.
4. Click **Save**.

---

## Step 4 — Create the SMTP email credential (2 minutes)

The workflow sends two emails: an acknowledgment to the prospect and a summary to your firm inbox. Both use the same SMTP credential.

1. In n8n, open your project and click the **Credentials** tab → **New credential**.
2. Search for **SMTP** and select it.
3. Enter your firm email settings:

   **Gmail:** Use an App Password — not your regular Gmail password. Generate one at Google Account → Security → App Passwords. Use `smtp.gmail.com`, port `587`.

   **Microsoft 365 / Outlook:** Use your email address and password. SMTP AUTH must be enabled: Microsoft 365 admin centre → Settings → Mail → SMTP AUTH.

   **Other providers:** Check your provider's documentation for SMTP host, port (usually `587` with STARTTLS), username, and password.

4. Click **Save**.

---

## Step 5 — Set n8n Variables (3 minutes)

In n8n, open your project and click the **Variables** tab. Add each of the following:

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your firm name as it should appear in emails (e.g. `Smith & Associates`) |
| `FIRM_EMAIL` | The inbox where firm alerts and prospect acknowledgments are sent from (e.g. `intake@smithlaw.com`) |
| `SALESMATE_BASE_URL` | Your Salesmate instance URL with no trailing slash (e.g. `https://smithlaw.salesmate.io`) |
| `SALESMATE_LINK_NAME` | The hostname portion only (e.g. `smithlaw.salesmate.io`) |
| `ROI_SHEET_ID` | The Google Sheet ID from Step 1 |
| `FIRM_BOOKING_URL` | Your online booking or contact page URL (e.g. `https://smithlaw.com/book`) — included in the acknowledgment email |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow. Open that workflow in n8n and read the ID from the URL: `.../workflow/12345` → enter `12345` |
| `OPS1_SHEET_ID` | The Google Sheet ID used by the Bar-Compliance Guardrail for its audit log. If you already deployed the guardrail, this variable is already set — do not change it. Find the value in the guardrail's deploy guide if setting it for the first time. |

---

## Step 6 — Import and activate the workflow (3 minutes)

1. In n8n, open your project and go to **Workflows** → **Add workflow** → **Import from file**.
2. Select [`pac-32-lead-source-roi-tracker.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-32-lead-source-roi-tracker.json).
3. Wire the credentials on each node that requires them:
   - **Add Lead to Salesmate** → select `Salesmate API`
   - **Log Lead to Dashboard** → select `Google Sheets account`
   - **Send Acknowledgment to Lead** → select `Email account (SMTP)`
   - **Notify Firm of New Lead** → select `Email account (SMTP)`
4. Click **Save**, then toggle the workflow to **Active**.
5. Copy the production webhook URL shown on the **Inbound Lead Webhook** node. You'll paste it into your forms and/or CallRail in the next step.

---

## Step 7 — Connect your lead sources

### Fluent Forms

For each intake form where you want to capture lead source:

1. Open the form in Fluent Forms → **Settings** → **Notifications** → **Webhooks** tab.
2. Add a new webhook, paste the production URL, and set the format to **JSON**.
3. Save the form.

**UTM parameter capture (recommended for ad campaigns):** Add four hidden fields to the form with the exact field names `utm_source`, `utm_medium`, `utm_campaign`, and `gclid`. On your WordPress page, add a small JavaScript snippet to read those values from the URL and pre-fill the hidden fields. This lets the workflow detect Google Ads clicks automatically. Guides: [UTM hidden fields in Fluent Forms](https://fluentforms.com/) — search "hidden field URL parameter".

In Google Ads: enable **Auto-tagging** in your account settings. Google then appends `gclid` to every ad click URL, and the workflow recognises this as a Google Ads lead regardless of UTM tags.

### CallRail

1. In CallRail, go to **Settings** → **Integrations** → **Webhooks**.
2. Add a new webhook, select **All Call Events**, and paste the production URL.
3. Save.

The workflow detects CallRail payloads automatically and assigns the channel `inbound-call`.

### Any other web form

Send a standard `HTTP POST` to the webhook URL with `Content-Type: application/json`. The workflow recognises any payload that contains standard field names (`first_name`, `last_name`, `email`, `phone`, `message`, etc.).

---

## Testing

The workflow ships with a pinned test payload — a Google Ads lead for a family law inquiry (Maria Gonzalez, Chicago), with UTM parameters and a `gclid`. Use it to verify the full flow without a real form submission:

1. Open the **Inbound Lead Webhook** node on the canvas.
2. The pinned payload is already loaded. Click **Execute workflow** from the top toolbar.
3. Watch each node run. The expected path is:
   - **Read Lead Details** → `is_valid_lead: true`
   - **Valid lead?** → Yes branch
   - **Tag Lead Source** → `channel: google-ads`
   - **Add Lead to Salesmate** → contact created (requires live credentials)
   - **Read Salesmate Response** → `salesmate_contact_id` populated
   - **Log Lead to Dashboard** → row appended to your Google Sheet
   - **Has email address?** → Yes (test email provided)
   - **Compliance Check — Acknowledgment** → approved
   - **Approved to send acknowledgment?** → Yes
   - **Send Acknowledgment to Lead** → email sent to `mgonzalez@example.com`
   - **Build Firm Alert** → `ack_status: Sent ✓`
   - **Notify Firm of New Lead** → email sent to `FIRM_EMAIL`

4. Open your Google Sheet and confirm the row appeared in the `Lead Log` tab.
5. Check your firm inbox for the lead summary email.

**What to check if something goes wrong:**

| Symptom | Likely cause |
|---|---|
| Row not added to Google Sheets | Wrong `ROI_SHEET_ID`, or the sheet tab is not named exactly `Lead Log`, or Google Sheets credential not connected |
| Salesmate contact not created | Wrong `SALESMATE_BASE_URL` or `SALESMATE_LINK_NAME`, or API token expired |
| Acknowledgment email not sent | SMTP credential not wired, or `GUARDRAIL_WORKFLOW_ID` points to a non-existent or inactive workflow |
| Workflow stopped at `Valid lead?` (No branch) | Payload missing name, email, and phone — or both name and email contain the word "test" |

---

## Understanding the ROI dashboard

### Columns the workflow fills automatically

| Column | Description |
|---|---|
| `date` | ISO timestamp when the lead arrived |
| `channel` | Canonical source: `google-ads`, `referral`, `legal-directory`, `website-organic`, `social-media`, `yelp`, `email-campaign`, `inbound-call`, `unknown` |
| `source_detail` | Campaign name, keyword, or referring URL |
| `lead_name` | Full name from the form |
| `lead_email` | Email address |
| `lead_phone` | Phone number |
| `matter_type` | Practice area or case type the lead specified |
| `message_preview` | First 200 characters of their message |
| `salesmate_contact_id` | Numeric ID — click into Salesmate at `https://yourfirm.salesmate.io/contacts/{id}` |

### Columns you fill manually

| Column | What to enter |
|---|---|
| `ad_spend (manual)` | What you spent on this channel this period. Enter once at the top of each channel group, not per row. |
| `signed_case (manual)` | `yes` when the lead becomes a paying client. Leave blank otherwise. |
| `case_value (manual)` | Retainer or total fee when the case signs. |

### Calculating ROI with a pivot table

Once you have a few weeks of data with some `signed_case = yes` rows:

1. Add a helper column to the right of your data. In the first data row (row 2) enter:
   ```
   =IF(K2="yes",1,0)
   ```
   Drag this formula down to cover all rows. Name the column header `signed_case_count`.

2. Click any cell in the data → **Insert** → **Pivot table** → **New sheet**.
3. Add **Rows:** `channel`
4. Add **Values:**
   - `lead_name` (COUNT) → total leads per channel
   - `signed_case_count` (SUM) → signed cases per channel
   - `case_value` (SUM) → total revenue per channel
   - `ad_spend` (SUM) → total spend per channel
5. In the pivot sheet, add a column to the right of the pivot output:
   ```
   =SUM(ad_spend column) / SUM(signed_case_count column)
   ```
   Label it `cost per signed case`.

This gives you a table of every channel sorted by cost-per-signed-case — the core insight the template is designed to surface.

---

## How source detection works

The workflow maps every lead to one channel. The first match wins:

| Channel | Detected when |
|---|---|
| `google-ads` | `gclid` is present (Google Ads auto-tag), OR signals contain `google ads`, `cpc`, or `ppc` |
| `social-media` | Signals contain `facebook`, `instagram`, `meta`, or `social` |
| `referral` | Signals contain `referral`, `attorney`, `colleague`, `word of mouth`, `friend`, or `family` |
| `legal-directory` | Signals contain `avvo`, `justia`, `findlaw`, `martindale`, `lawyers.com`, `nolo`, or `directory` |
| `yelp` | Signals contain `yelp` |
| `email-campaign` | Signals contain `email`, `newsletter`, `campaign`, or `drip` |
| `inbound-call` | Payload came from a CallRail webhook |
| `website-organic` | Signals contain `organic`, `seo`, `blog`, or `search`; OR any referrer URL is present |
| `unknown` | No recognisable signal found |

"Signals" = combined `utm_source` + `utm_medium` + `referrer` + `how_did_you_hear` from the form payload.

**To improve accuracy:** Add UTM hidden fields to every Fluent Form. Enable auto-tagging in Google Ads. For referrals, add a "How did you hear about us?" dropdown to your forms with options that include the word "referral".

---

## Done-for-you upgrade

The free template gives you a Google Sheets dashboard you populate manually for ad spend, signed cases, and case values. The Protomated done-for-you build adds:

- **Automatic ad spend pull** — connects to Google Ads and Meta Ads APIs to write spend data directly into your dashboard, no manual entry
- **Looker Studio dashboard** — live charts of cost-per-lead, cost-per-signed-case, and revenue by channel, updated in real time
- **Clio case status sync** — when a lead's matter opens in Clio, the `signed_case` column updates automatically
- **Lead deduplication** — checks Salesmate before creating a contact, merges duplicates rather than creating two records
- **Multi-firm reporting** — single dashboard across multiple intake sources, practice areas, and office locations

Contact [Protomated](https://protomated.com) to get a quote.
