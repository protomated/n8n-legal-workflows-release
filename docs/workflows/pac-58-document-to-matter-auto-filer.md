# Deploy Guide: Document to Matter Auto-Filer

**Template:** PAC-58 — OPS3: Document to Matter Auto-Filer
**Pillar:** Ops
**Replaces:** Manual filing, DMS add-ons

Get value in under 15 minutes of setup, though this one has more moving parts to test than most templates in this catalog — set aside time for the testing pass in Step 6. Inbound documents to your intake mailbox get matched to the right Clio matter by sender email and matter number — never by reading the document — then filed into that matter's Google Drive folder with a clean, date-prefixed name. Anything that can't be confidently matched goes to a staff alert with the original attachment, instead of being filed to a guess.

---

## Before you start: heads up on this one

This template touches more genuinely new integration surface than most others in this catalog (IMAP email polling, Google Drive folder search/create) and a few specific node settings couldn't be verified against a live n8n instance while it was built — they're clearly flagged in the affected nodes' notes and called out again in Step 6's testing checklist. Budget extra time for the first test pass; this is not a "paste and go" template the way some others are.

**This never guesses which matter a document belongs to.** If the sender isn't a known Clio contact, or that contact has more than one open matter and none of them are referenced in the subject line, the document routes to a staff alert instead of being auto-filed — filing a confidential document to the wrong matter is a worse outcome than asking a human to handle it.

---

## What you need before you start

- An n8n instance (self-hosted or n8n Cloud)
- A dedicated intake mailbox with IMAP access (e.g. documents@yourfirm.com)
- A Clio account with API access (reuse the OAuth2 credential from PAC-18/20/55/57 if already deployed)
- A Google Drive account with a root "Matters" folder created
- An email account with SMTP access

---

## Step 1 — Set up the intake mailbox (5 min)

1. Create or designate a mailbox for inbound documents (e.g. documents@yourfirm.com). This can be a shared mailbox, an alias, or a dedicated account.
2. Enable IMAP access: Gmail → Settings → Forwarding and POP/IMAP → Enable IMAP (use an App Password, not your regular password). Microsoft 365 → enable IMAP for the mailbox in the admin center.
3. Point whatever produces your inbound documents at this address — a scanner's "scan to email" feature, a fax-to-email service, or just tell clients/opposing counsel to CC it.

---

## Step 2 — Create the Matters root folder (2 min)

1. In Google Drive, create a folder named something like "Client Matters."
2. Copy its folder ID from the URL: `https://drive.google.com/drive/folders/THIS_PART_IS_THE_ID`.

The workflow creates one subfolder per matter under this root automatically the first time a document arrives for it — you don't need to pre-create matter folders.

---

## Step 3 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-58-document-to-matter-auto-filer.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-58-document-to-matter-auto-filer.json). Do not activate it yet.
2. IMAP credential: **Credentials → New credential → IMAP** → Host/Port/User/Password for your intake mailbox → save as `Intake mailbox (IMAP)`.
3. Clio credential: reuse your existing `Clio API (OAuth2)` credential if PAC-18, PAC-20, PAC-55, or PAC-57 is already deployed. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide.
4. Google Drive credential: reuse your existing `Google Drive account` credential if PAC-57 is already deployed. Otherwise follow the Google OAuth2 setup in the PAC-57 deploy guide (Drive API + Docs API not needed here — just enable the Drive API).
5. Email credential: **Credentials → New credential → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 4 — Set n8n Variables (2 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `MATTERS_ROOT_FOLDER_ID` | The Drive folder ID from Step 2 |
| `FIRM_STAFF_EMAIL` | Where filing exceptions (unmatched documents) are sent |
| `FIRM_FROM_EMAIL` | The address alert emails are sent from |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |

---

## Step 5 — Activate

Toggle the workflow to **Active**. The IMAP trigger polls the mailbox on its own from here — no webhook URL to wire anywhere.

---

## Step 6 — Test (budget real time for this one)

The workflow ships with a pinned test email (from a fictional "Jane Doe" with one PDF attachment) on the trigger node, but because IMAP triggers poll a live mailbox rather than receiving a webhook payload, the most reliable test is sending a real email to your intake mailbox once everything's wired up.

**Test 1 — the happy path.** In Clio, find (or create) a test contact whose email you control, with exactly one open matter. Email that address from the account matching the Clio contact's email, with any PDF attached. Within a few minutes (IMAP polling interval), confirm:
- A new folder appears under your Matters root folder, named after the matter's display number.
- The attachment appears inside it, renamed with today's date prefixed.
- No alert email is sent.

**Test 2 — the exception path.** Email the intake mailbox from an address that is *not* a Clio contact. Confirm `FIRM_STAFF_EMAIL` receives an alert with the original attachment and a clear reason ("No Clio contact matches this sender's email address").

**Test 3 — the ambiguous-matter case (optional but recommended).** If your test contact has more than one open matter, send an email with no matter reference in the subject and confirm it routes to the exception alert rather than picking one at random.

**If something doesn't work, check these specific spots first** — each is flagged in its node's notes as unverified against live n8n during authoring:
- **When Document Email Received**: if attachments aren't coming through as binary data at all, confirm "Download Attachments" is toggled on wherever it appears in your n8n version's IMAP trigger node.
- **Search For Matter Folder**: if this errors or the search query looks wrong, confirm the query string landed in the correct field — it should read as a standard Google Drive search query (`'FOLDER_ID' in parents and name = 'MATTER_NUMBER' and ...`).
- **Upload Attachment To Matter Folder**: if files upload to the wrong location (e.g. Drive root instead of the matter folder), the parent-folder option may be in a different spot than expected — open the node and locate wherever "Parent Folder" actually appears, and point it at `{{ $json.folder_id }}`.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Email has no attachments | Skipped silently — nothing to file |
| Sender's email matches no Clio contact | Routed to staff alert with original attachment(s); logged as an exception |
| Contact has exactly one open matter | Confident match — auto-filed |
| Contact has multiple open matters, one referenced in the subject line | Confident match — auto-filed to that specific matter |
| Contact has multiple open matters, none referenced in the subject | Routed to staff alert rather than guessing |
| Matter's Drive folder doesn't exist yet | Created automatically under `MATTERS_ROOT_FOLDER_ID`, then filed |
| Multiple attachments on one email | Every attachment filed to the same matched matter, each logged individually in the audit summary |

---

## Compliance note

This template classifies and files by metadata and rules only — sender email address and matter reference numbers — never by reading document content, and never makes a legal judgment about what a document is or means (ABA Op. 512). When a confident match isn't possible, it defers to a human rather than guessing. Every filing and every exception is logged to the Bar-Compliance Guardrail's audit sheet.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces manual filing and DMS add-ons. Inbound documents that pile up unfiled cost real time to locate later. |
| Compliance test | Filing and routing only — metadata/rules-based matching, no content analysis, no legal judgment. |
| Searchability test | Ranks for "auto file documents to matter" |
| Deployability test | Reuses Clio and Google Drive credentials if already deployed elsewhere in this catalog; new setup is one IMAP mailbox and one root Drive folder |
| Upsell test | Quick-Win Build adds OCR-based classification for scanned documents with no useful subject line, and direct filing into Clio's own document store |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- OCR/content-aware classification for scanned documents where sender and subject alone aren't enough
- Direct filing into Clio's native document store (not just Drive) — Clio's document upload API is genuinely complex to integrate reliably, and this is exactly the kind of hardening a paid build is for
- Multi-mailbox support for firms with more than one intake address
- A weekly digest of everything auto-filed and everything that needed manual attention

[Book a call with Protomated](https://protomated.com/book) to get started.
