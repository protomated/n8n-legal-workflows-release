# Deploy Guide: Form-Fill from Intake Data

**Template:** PAC-57 — OPS5: Form-Fill from Intake Data
**Pillar:** Ops
**Replaces:** Lawyaw or Clio Draft ($70–119/month)

Get value in under 15 minutes of setup, once your templates exist. The moment a new intake comes in, this fills your firm's own retainer, intake form, and court cover sheet templates with the client's actual details — no more retyping the same name, email, and case details into three different documents. The retainer is the one document that gets signed, so it fills as a draft DocuSign envelope; everything else fills as a reviewable PDF. Nothing is sent to a client, and nothing is signed, until a human does it manually.

---

## Before you start: what this template does not do

This is field population only. It never drafts legal language, never chooses what a document says beyond substituting the fields you tell it to, and never sends or signs anything automatically — every generated document requires a human to review and act on it (ABA Op. 512).

---

## What you need before you start

- An n8n instance (self-hosted or n8n Cloud)
- A Google Workspace account with your firm's non-retainer templates saved as Google Docs
- A Google Cloud project with OAuth credentials (Drive + Docs API scopes) — see Step 2
- A DocuSign account (a free developer/demo sandbox account works for testing) with a retainer template already built there — see Step 1
- An email account with SMTP access

---

## Step 1 — Build your templates (15 min, one-time)

### Google Docs templates (everything except the retainer)

1. Create a Google Doc for each non-retainer document type (e.g. Client Intake Form, Court Cover Sheet).
2. Anywhere you want intake data to appear, type one of these tags literally into the document text (case-sensitive, including the double curly braces): `{{full_name}}`, `{{email}}`, `{{phone}}`, `{{practice_area}}`, `{{case_details}}`, `{{referral_source}}`, `{{date}}`.
3. Copy each template's file ID from its URL: `https://docs.google.com/document/d/THIS_PART_IS_THE_ID/edit`.

Want a different set of tags? Edit the `MERGE_FIELDS` object in the **Build Merge Field Replacements** node.

### Generated documents folder (one-time)

Create a Drive folder to hold the filled copies this workflow generates (e.g. "Generated Documents"), and copy its folder ID from the URL the same way as above. Every filled Google Doc copy is saved here rather than sitting in the same folder as your templates, and is kept (not deleted) so the review email can link straight to it if the attorney wants to open and edit it directly instead of just reviewing the PDF.

### DocuSign template (the retainer)

1. In DocuSign, go to **Templates → New → Create a Template**. Upload your retainer agreement.
2. Add a signer role (e.g. name it "Client" — you'll reference this exact name in Step 3).
3. Place fields on the document for each piece of data you want pre-filled, and give each field a **Data Label** matching these exactly: `full_name`, `email`, `phone`, `practice_area`, `case_details`, `date`. (In DocuSign's template editor, click a field → Data Label — this is what the API matches against, not the field's visible name.)
4. Save the template and copy its **Template ID** from the template list.

Want different fields? Edit the `textTabs` array in the **Build DocuSign Envelope Request** node to match your template's actual Data Labels.

---

## Step 2 — Import the workflow and add credentials (10 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-57-form-fill-from-intake-data.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-57-form-fill-from-intake-data.json). Do not activate it yet.

2. **Google credential** (covers the Drive + Docs steps):
   - In [Google Cloud Console](https://console.cloud.google.com) → APIs & Services → Library, enable both the **Google Drive API** and the **Google Docs API**.
   - APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID (Web application type). Copy the Client ID and Client Secret.
   - In n8n: **Credentials → New credential** → search **Google Drive** → select **Google Drive OAuth2 API** → paste in Client ID/Secret → **Connect my account** → approve.
   - Save as `Google Drive account`.

3. **DocuSign credential**:
   - In DocuSign, go to **Settings → Apps and Keys → Add App / Integration Key**. Name it (e.g. "n8n Form-Fill"). Copy the **Integration Key** (this is your Client ID).
   - Under that app's settings, add a **Redirect URI**: `https://YOUR-N8N-URL/rest/oauth2-credential/callback` (same fixed callback n8n uses for every OAuth2 credential — replace YOUR-N8N-URL with your actual n8n domain).
   - Generate a **Secret Key** (this is your Client Secret) and copy it — DocuSign only shows it once.
   - In n8n: **Credentials → New credential → OAuth2 API**:
     - Grant Type: `Authorization Code`
     - Authorization URL: `https://account-d.docusign.com/oauth/auth` (sandbox) or `https://account.docusign.com/oauth/auth` (production)
     - Access Token URL: `https://account-d.docusign.com/oauth/token` (sandbox) or `https://account.docusign.com/oauth/token` (production)
     - Client ID / Client Secret: from above
     - Scope: `signature`
     - Click **Connect my account** and approve.
   - Save as `DocuSign account`.

4. **Email credential**: **Credentials → New credential → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `DOCUSIGN_AUTH_BASE_URL` | `https://account-d.docusign.com` for sandbox, `https://account.docusign.com` for production |
| `DOCUSIGN_RETAINER_TEMPLATE_ID` | The Template ID copied in Step 1 |
| `DOCUSIGN_RETAINER_ROLE_NAME` | The signer role name from your DocuSign template exactly (e.g. `Client`) |
| `TEMPLATE_INTAKE_DOC_ID` | The file ID of your Client Intake Form template |
| `TEMPLATE_COVER_SHEET_DOC_ID` | The file ID of your Court Cover Sheet template |
| `GENERATED_DOCS_FOLDER_ID` | The folder ID of a Drive folder where filled document copies are saved (create one folder, e.g. "Generated Documents," and reuse it — see Step 1b) |
| `FIRM_REVIEWER_EMAIL` | Where filled documents are sent for review. Falls back to `FIRM_EMAIL` if unset |
| `FIRM_FROM_EMAIL` | The address filled-document emails are sent from |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33: `.../workflow/WORKFLOW_ID` |

Only configure the templates you actually have — a document type with no Variable set is skipped automatically, not treated as an error.

---

## Step 4 — Confirm which documents apply to which practice area (2 min)

Open **Build Document Fill List** and check the `TEMPLATE_MAP` object matches how your firm actually works — which practice areas need a retainer (almost all), which need a court cover sheet (usually only litigation), and so on. Add, remove, or edit rows directly in the code. Each row's `method` (`docusign` or `google_docs`) decides which engine fills it — only change a document's method away from `docusign` if it genuinely doesn't need a signature.

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
3. Confirm **Build Document Fill List** finds the documents you configured, with the retainer showing `method: "docusign"` and the others `method: "google_docs"`.
4. Run the rest of the chain and confirm:
   - **The review email arrives** at `FIRM_REVIEWER_EMAIL` with an actual PDF file attached for each Google Docs document (check the bottom of the email, not just the body text) — plus a line for each one with a link to the editable Drive copy, and a separate line referencing a DocuSign envelope ID for the retainer.
   - **Open the PDF attachments** and confirm the `{{merge_tag}}` placeholders were actually replaced with the test intake's real data, not left as literal text.
   - **Click the "Open/edit the copy" link** in the email and confirm it opens the filled Google Doc directly, sitting inside your `GENERATED_DOCS_FOLDER_ID` folder — not the original template.
   - **In DocuSign**, under **Manage → Drafts**, a new draft envelope appears with the retainer template, fields pre-filled with the test intake's data, and status still showing as a draft — **confirm it has NOT been sent**. This is the single most important thing to verify before this goes live.

**If the DocuSign envelope doesn't appear, or fields aren't pre-filled**: check that your template's Data Labels (Step 1) exactly match the `tabLabel` values in **Build DocuSign Envelope Request**, and that `DOCUSIGN_RETAINER_ROLE_NAME` matches your template's signer role name exactly — both are case-sensitive exact-match requirements on DocuSign's side.

**If the email arrives with no PDF attached**: open **Notify Attorney Documents Ready For Review** → Options → Attachments, and confirm the binary property field is set to an expression (not left blank) — a blank/empty attachment entry silently sends no attachment at all on this node, it does not mean "attach everything."

**If the PDF comes through empty or in the wrong format**: open **Export Filled Document As PDF** → Options, and confirm the Google Docs conversion format is set to PDF. This exact option's layout couldn't be verified against a live n8n instance while this template was built — if it looks different from what you'd expect, re-select PDF from whatever conversion dropdown is shown and re-save.

**Negative test:** send a POST with a matter_type not in your `TEMPLATE_MAP` (e.g. `"matter_type": "Immigration"`) and confirm it routes to **Skip — No Templates Configured** with no email sent.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Intake missing full_name, email, or matter_type | Skipped — **Skip — Invalid Intake Payload** |
| Practice area not in `TEMPLATE_MAP`, or its template Variable isn't set | Skipped — **Skip — No Templates Configured** |
| Practice area has some but not all template Variables set | Only the configured documents are generated — no error for the missing ones |
| Retainer configured for the practice area | Created as a **draft** DocuSign envelope (status `created`, never `sent`) with fields pre-filled — attorney reviews and sends manually from DocuSign |
| Non-retainer document configured | Filled via Google Docs and attached to the review email as a PDF |
| One Google Docs export fails partway through | The other documents still generate and email successfully; the failed one is silently excluded — check execution history to see which |
| All configured documents generate successfully | One email with a PDF per Google Docs document and a DocuSign envelope reference for the retainer, sent to `FIRM_REVIEWER_EMAIL` |

---

## Compliance note

This template is operational only — it fills firm-provided template fields with firm-provided intake data. It does not touch legal work (ABA Op. 512): no clause drafting, no legal analysis, no judgment calls about what a document should say. The DocuSign retainer is always created as a draft, never sent for signature automatically — the attorney reviews and sends it manually. Every document requires human review before use, and nothing is ever sent to a client or signed automatically. Every run is logged to the Bar-Compliance Guardrail's audit sheet.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces Lawyaw or Clio Draft ($70–119/mo). Lawyers spend 40–60% of the day on document busywork like retyping the same fields across multiple forms. |
| Compliance test | Field population only — no legal drafting, no automated client delivery, no automated signing, human review required before use. |
| Searchability test | Ranks for "auto fill legal forms from intake" |
| Deployability test | One Google OAuth2 credential, one DocuSign OAuth2 credential, one SMTP credential, plus your existing templates — 15-20 minutes once templates exist |
| Upsell test | Quick-Win Build adds direct upload of filled documents into the matching Clio matter, and support for Word (.docx) templates alongside Google Docs |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Direct upload of filled documents into the matching Clio matter
- Support for Word (.docx) templates alongside Google Docs
- Automatic DocuSign send (with attorney approval built into the workflow) instead of manual send from DocuSign's UI
- A template library with version history, so template edits don't require re-copying file/template IDs

[Book a call with Protomated](https://protomated.com/book) to get started.
