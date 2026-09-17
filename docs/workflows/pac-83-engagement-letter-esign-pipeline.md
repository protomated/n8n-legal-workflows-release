# Deploy Guide: Engagement Letter & E-Sign Pipeline

**Template:** PTPAC-83 — C5: Engagement Letter & E-Sign Pipeline
**Pillar:** Convert
**Replaces:** Gavel plus manual DocuSign chaining

Get value in under 15 minutes. Once a lead is qualified, someone still has to manually fill your engagement letter template, send it through DocuSign, chase the signature, and remember to open the matter afterward — and every manual re-typing of the same name and details is a chance to get something wrong. This workflow fills the letter from structured lead data, gets it in front of an attorney for a one-click approval, and only then sends it to the client — opening the Clio matter automatically the moment it comes back signed.

---

## Before you start: the most important thing to understand

**This workflow never drafts engagement terms.** It fills the text fields your own DocuSign template already defines — name, email, practice area, case details, date — using your firm's pre-approved language. There is no code path anywhere in this template that generates or edits legal text.

**Nothing reaches the client without an attorney's explicit approval.** The envelope is created as a *draft* in DocuSign. It only gets sent to the client after the attorney clicks the approve link in their review email — there is no automatic-send path.

**This is a different deliverable from the Claude Desktop skill that drafts engagement letter language from unstructured intake notes** (shipped separately in the Starter Kit). That skill helps write the letter's content. This workflow assumes your letter's content is already finalized as a DocuSign template — it only fills structured fields, routes for approval, and opens the matter on signature.

**This has three independent parts on one canvas:**
1. **Create the draft** — fires when a lead is qualified for an engagement letter. Resolves the Clio contact, creates a draft DocuSign envelope from your template, logs it, and emails the attorney a review link.
2. **Attorney approval** — a webhook fired by clicking Approve in that review email. Transitions the envelope from draft to sent — DocuSign delivers it to the client directly.
3. **Signature completed** — a webhook DocuSign itself calls when the client finishes signing. Opens the Clio matter and notifies staff.

---

## What you need before you start

- A DocuSign account with API access and your firm's engagement letter built as a **template** with text tabs (see Step 1)
- A Clio account with API access (reuse the OAuth2 credential from other templates in this catalog if already deployed)
- A Google account with Sheets access
- An email account with SMTP access
- Whatever marks a lead "qualified" in your intake process (a hot-lead branch of your intake qualifier, or a manual staff action)

---

## Step 1 — Build the DocuSign template (5 min)

In DocuSign, create (or open) your firm's engagement letter as a **Template**, and add these text tabs to it, using these exact labels (case-sensitive):

```
full_name | email | phone | practice_area | case_details | date
```

Copy the template's ID from the template list — this becomes `ENGAGEMENT_LETTER_TEMPLATE_ID`. Also note the **role name** you assigned to the client's signer role (e.g. "Client") — this becomes `ENGAGEMENT_LETTER_ROLE_NAME`.

---

## Step 2 — Create the tracking sheet (3 min)

Create a new Google Sheet. Add a tab named exactly **Engagement Pipeline** with these column headers in row 1:

```
review_id | full_name | email | practice_area | contact_id | envelope_id | status | created_at | sent_at | signed_at | matter_id
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 3 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-83-engagement-letter-esign-pipeline.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-83-engagement-letter-esign-pipeline.json). Do not activate it yet.
2. DocuSign credential: reuse your existing `DocuSign account` OAuth2 credential if already deployed elsewhere in this catalog (e.g. Form-Fill from Intake Data).
3. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog.
4. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
5. Email credential: **Credentials → New → SMTP**.

---

## Step 4 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | The sender address for internal notification emails |
| `FIRM_ATTORNEY_EMAIL` | Fallback recipient for the review request if a lead has no specific attorney assigned |
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates |
| `DOCUSIGN_AUTH_BASE_URL` | Same value as your other DocuSign-integrated templates |
| `ENGAGEMENT_LETTER_TEMPLATE_ID` | The DocuSign template ID from Step 1 |
| `ENGAGEMENT_LETTER_ROLE_NAME` | The signer role name from Step 1 |
| `ENGAGEMENT_PIPELINE_SHEET_ID` | The Google Sheet ID from Step 2 |
| `ENGAGEMENT_DECISION_WEBHOOK_URL` | This workflow's own Production webhook URL for the approval branch — copy this in Step 5, right after you activate |

---

## Step 5 — Activate and wire the triggers (5 min)

1. Toggle the workflow to **Active**.
2. Open **"When Lead Qualified For Engagement"**, copy its **Production URL**, and wire it to whatever marks a lead qualified in your intake process.
3. Open **"Attorney Decision Request Trigger"**, copy its **Production URL**, and set it as `ENGAGEMENT_DECISION_WEBHOOK_URL` in Settings → Variables (Step 4).
4. Open **"DocuSign Signature Webhook"**, copy its **Production URL**, and register it in DocuSign under **Admin → Connect** as a webhook subscribed to the **Envelope Completed** event.

---

## Step 6 — Test

The workflow ships with pinned sample data across all three trigger nodes.

**Create the draft:**
1. Run **"When Lead Qualified For Engagement"** (pinned as Jordan Casey, Estate Planning) → step through **"Fetch Existing Clio Contact"** → **"Determine If Contact Exists"** → **"If Contact Exists In Clio"** and confirm it routes to create a new contact (no match in your pinned/test Clio data).
2. Continue through **"Fetch DocuSign User Info"** → **"Build Engagement Letter Envelope Request"** and confirm the six text tabs are populated correctly from the lead data.
3. Continue through **"Post Draft Envelope To DocuSign"** → **"Parse Envelope Creation Response"** and confirm `envelope_created: true` with a real `envelope_id`.
4. Continue through **"Assign Review Id"** → **"Log Engagement For Review"** and confirm a new row would be logged with `status: pending_review`.
5. Continue through **"Build Attorney Review Request Email"** and confirm the approve link contains the correct `review_id`.

**Attorney approval:**
6. Copy the `review_id` your test just logged into the pinned data on **"Attorney Decision Request Trigger"**, then run it through **"Parse Attorney Decision Request"** → **"Read Engagement Pipeline Log"** → **"Find Matching Engagement Row"** → **"If Engagement Row Found"** and confirm it finds the row.
7. Continue through **"Fetch DocuSign Account Info For Send"** → **"Transition Envelope To Sent"** and confirm the envelope's status becomes `sent` in your DocuSign account — check that the client actually receives it.
8. Continue through **"Update Engagement Row As Sent"** and confirm `status` becomes `sent`.
9. Edit the pinned `review_id` to something not in the sheet and confirm it routes to **"Respond Engagement Not Found"**.

**Signature completed:**
10. Once you've actually signed the test envelope yourself (or use DocuSign's own template test-send), confirm **"DocuSign Signature Webhook"** fires for real, or manually run it with the pinned `envelope-completed` payload (update the `envelopeId` to match your test envelope) through **"Parse DocuSign Signature Event"** → **"If Envelope Fully Signed"**.
11. Continue through **"Read Pipeline Log For Signature"** → **"Find Matching Signed Row"** → **"Create New Clio Matter"** → **"Parse Matter Creation Response"** and confirm a matter is created under the correct Clio contact.
12. Continue through **"Update Engagement Row As Signed"** → **"Notify Staff Of Signed Engagement"** and confirm staff would be notified with the new matter ID.
13. Edit the pinned payload's `event` to `envelope-sent` and confirm it routes to **"Skip — Not A Completion Event"** instead of processing.

**Testing against your real setup:** unpin every Clio-, DocuSign-, and Sheets-facing node, connect real credentials, and run one lead all the way through — from qualified lead, to attorney approval, to a real signature, to a real Clio matter.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A lead is qualified | A draft envelope is created and logged; the attorney is emailed a review link — nothing goes to the client yet |
| The attorney approves | The envelope is transitioned to sent; DocuSign delivers it to the client directly |
| The attorney never approves | The envelope stays a draft in DocuSign indefinitely — follow up manually from the tracking sheet |
| The approval link is malformed or already used | A plain error page is shown; nothing changes |
| The client signs | The Clio matter opens automatically and staff are notified |
| DocuSign fires a non-completion event (viewed, delivered, declined) | Ignored — only a full completion event triggers matter creation |

---

## Compliance note

This template performs field population, routing, and record-keeping only — it fills a firm-approved DocuSign template with structured lead data, and opens a matter on signature. It never drafts or edits engagement terms, and every envelope requires an attorney's explicit approval before it reaches the client (ABA Op. 512). Inherits OPS1 conventions for this catalog; since every message here goes to firm staff, not a client, no client-facing compliance wrapper is needed.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces Gavel plus manual DocuSign chaining; removes the re-typing and manual follow-up between qualifying a lead and opening the matter |
| Compliance test | Field population only; the attorney approves every envelope before it reaches the client, and no legal terms are drafted |
| Searchability test | Ranks for "engagement letter automation for law firms" |
| Deployability test | Reuses DocuSign and Clio credentials from other templates if already deployed — under 15 minutes |
| Upsell test | Clear Done-for-You path: productized as a fixed-scope build with firm-specific template logic and PM-system integration beyond Clio |

---

## Want the advanced version?

Protomated builds the done-for-you **Done-for-You** implementation that adds:
- Multiple engagement letter templates by practice area, auto-selected from lead data
- Reminder sequences for a draft the attorney hasn't approved yet
- Integration with practice management systems beyond Clio
- Firm-wide pipeline reporting on time-to-signature

[Book a call with Protomated](https://protomated.com/book) to get started.
