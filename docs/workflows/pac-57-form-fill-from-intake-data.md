# Deploy Guide: Form-Fill from Intake Data

**Template:** PAC-57 — OPS5: Form-Fill from Intake Data
**Pillar:** Ops
**Replaces:** Lawyaw or Clio Draft ($70–119/month)

Get value in under 15 minutes. The moment a new intake comes in, this fills your firm's own retainer agreement, intake form, and court cover sheet templates with the client's actual details — no more retyping the same name, email, and case details into three different documents. Every filled document lands in your reviewing attorney's inbox as a PDF, ready to check before it's used.

---

## Before you start: what this template does not do

This is field population only. It never drafts legal language, never chooses what a document says beyond substituting the fields you tell it to, and never sends anything to a client automatically — every generated document goes to your firm for review first (ABA Op. 512).

---

## What you need before you start

- An n8n instance (self-hosted or n8n Cloud)
- A Google Workspace account with your firm's document templates saved as Google Docs
- A Google Cloud project with OAuth credentials (Drive + Docs API scopes) — see Step 2
- An email account with SMTP access

---

## Step 1 — Build your templates (10 min, one-time)

1. Create a Google Doc for each document type you want auto-filled (e.g. Retainer Agreement, Client Intake Form, Court Cover Sheet).
2. Anywhere you want intake data to appear, type one of these tags literally into the document text (case-sensitive, including the double curly braces):
   - `{{full_name}}`
   - `{{email}}`
   - `{{phone}}`
   - `{{practice_area}}`
   - `{{case_details}}`
   - `{{referral_source}}`
   - `{{date}}` — today's date, formatted like "August 12, 2026"
3. Copy each template's file ID from its URL: `https://docs.google.com/document/d/THIS_PART_IS_THE_ID/edit`.

Want a different set of tags? Edit the `MERGE_FIELDS` object in the **Build Merge Field Replacements** node — add or remove rows to match what your templates actually use.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-57-form-fill-from-intake-data.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-57-form-fill-from-intake-data.json). Do not activate it yet.
2. Google credential:
   - In [Google Cloud Console](https://console.cloud.google.com), create OAuth 2.0 credentials (or reuse an existing project) and enable the **Google Drive API** and **Google Docs API**.
   - In n8n: **Credentials → New credential → Google OAuth2 API**.
   - Client ID / Client Secret: from Google Cloud Console.
   - Scopes: `https://www.googleapis.com/auth/drive https://www.googleapis.com/auth/documents`
   - Save as `Google OAuth2 API`, then **Connect my account** and approve access.
3. Email credential: **Credentials → New credential → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `TEMPLATE_RETAINER_DOC_ID` | The file ID of your Retainer Agreement template |
| `TEMPLATE_INTAKE_DOC_ID` | The file ID of your Client Intake Form template |
| `TEMPLATE_COVER_SHEET_DOC_ID` | The file ID of your Court Cover Sheet template |
| `FIRM_REVIEWER_EMAIL` | Where filled documents are sent for review. Falls back to `FIRM_EMAIL` if unset |
| `FIRM_FROM_EMAIL` | The address filled-document emails are sent from |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33: `.../workflow/WORKFLOW_ID` |

Only configure the templates you actually have — a document type with no Variable set is skipped automatically, not treated as an error.

---

## Step 4 — Confirm which documents apply to which practice area (2 min)

Open **Build Document Fill List** and check the `TEMPLATE_MAP` object matches how your firm actually works — e.g. maybe Estate Planning never needs a court cover sheet, or Business Law needs a document type not listed here at all. Add, remove, or edit rows directly in the code.

---

## Step 5 — Activate and connect your intake form (2 min)

1. Toggle the workflow to **Active**.
2. Open **When Intake Form Submitted** and copy its Production URL.
3. Point your intake form at it:

**Fluent Forms:** Settings → Integrations → Webhook → Add New → paste the URL → Method POST → Format JSON.

**Other builders:** Look for a Webhook, HTTP Request, or Zapier/Make integration option. Expected fields: `full_name`, `email`, `matter_type` (required); `phone`, `case_details`, `referral_source` (optional) — the same field names used by the Clio Intake Sync template, so one form feed can power both.

---

## Step 6 — Test

The workflow ships with a pinned test intake (Jane Doe, Personal Injury) on the trigger node.

1. Click **When Intake Form Submitted** → **Test step**.
2. Confirm **Validate And Normalize Intake Data** outputs `is_valid_intake: true`.
3. Confirm **Build Document Fill List** finds the documents you configured for "Personal Injury" in `TEMPLATE_MAP`.
4. Run the rest of the chain and confirm the review email arrives at `FIRM_REVIEWER_EMAIL` with one PDF attachment per configured document, each one showing the pinned test data (Jane Doe, her email, phone, and case details) merged into your template text.

**If the PDF comes through empty or as JSON instead of a proper file**: open **Export Filled Document As PDF** → Options → Response, and confirm Response Format shows "File" with "Put Output in Field" set to `data`. This one setting couldn't be verified against a live n8n instance while this template was built — if it's off, re-select File from the dropdown and re-save.

**Negative test:** send a POST with a matter_type not in your `TEMPLATE_MAP` (e.g. `"matter_type": "Immigration"`) and confirm it routes to **Skip — No Templates Configured** with no email sent.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Intake missing full_name, email, or matter_type | Skipped — **Skip — Invalid Intake Payload** |
| Practice area not in `TEMPLATE_MAP`, or its template Variable isn't set | Skipped — **Skip — No Templates Configured** |
| Practice area has some but not all template Variables set | Only the configured documents are generated — no error for the missing ones |
| One document's export fails partway through | The other documents still generate and email successfully; the failed one is silently excluded — check execution history to see which |
| All configured documents generate successfully | One email, one PDF attachment per document, sent to `FIRM_REVIEWER_EMAIL` |

---

## Compliance note

This template is operational only — it fills firm-provided template fields with firm-provided intake data. It does not touch legal work (ABA Op. 512): no clause drafting, no legal analysis, no judgment calls about what a document should say. Every document requires human review before use, and nothing is ever sent to a client automatically. Every run is logged to the Bar-Compliance Guardrail's audit sheet.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces Lawyaw or Clio Draft ($70–119/mo). Lawyers spend 40–60% of the day on document busywork like retyping the same fields across multiple forms. |
| Compliance test | Field population only — no legal drafting, no automated client delivery, human review required before use. |
| Searchability test | Ranks for "auto fill legal forms from intake" |
| Deployability test | One Google OAuth2 credential, one SMTP credential, a handful of template IDs — under 15 minutes once templates exist |
| Upsell test | Quick-Win Build adds e-signature routing (DocuSign) straight from the reviewed document, and direct filing into Clio |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- One-click send to DocuSign for e-signature once the attorney approves the retainer
- Direct upload of filled documents into the matching Clio matter
- Support for Word (.docx) templates alongside Google Docs
- A template library with version history, so template edits don't require re-copying file IDs

[Book a call with Protomated](https://protomated.com/book) to get started.
