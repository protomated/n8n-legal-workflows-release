# Deploy Guide: New-Matter Setup Bot

**Template:** PTPAC-59 — OPS2: New-Matter Setup Bot
**Pillar:** Ops
**Replaces:** Manual matter-opening checklists, Clio's own templates

Get value in under 15 minutes for the base setup (folder tree only), though the full payoff — document templates, task checklist, and intake deadlines — depends on how much time you spend filling in the three configuration maps in Step 5. A matter opened in Clio today, with nothing configured beyond the folder tree, still saves the 5–10 minutes of manual folder-building every new matter costs.

---

## Before you start: heads up on this one

**Staff don't change how they work.** This bot doesn't create matters — it watches for matters your staff already open in Clio the normal way, and does the busywork that follows. Nothing to teach anyone.

**Every stage is independently optional.** Folders always get created. Document templates, the task checklist, and intake deadlines each only run if you've configured them for that matter's practice area — leave a map entry out and that step is skipped, not broken.

**This never sets a legal deadline for you.** The intake deadline stage does date arithmetic only, from rules you enter yourself in `DEADLINE_RULE_MAP`. Same ground rule as the Court Deadline Calculator (NTC-16) — the firm owns every rule.

---

## What you need before you start

- An n8n instance (self-hosted or n8n Cloud)
- A Clio account with API access (reuse the OAuth2 credential from PAC-18/20/55/57/58 if already deployed)
- A Google Drive account (reuse the one from PAC-57/58 if already deployed)
- Internal Google Doc templates for whatever you want auto-copied into new matters (e.g. a Matter Opening Checklist) — optional, skip if you don't have these yet
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-59-new-matter-setup-bot.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-59-new-matter-setup-bot.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if PAC-18, PAC-20, PAC-55, PAC-57, or PAC-58 is already deployed. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide.
3. Google Drive credential: reuse your existing `Google Drive account` credential if PAC-57/58 is already deployed (Drive API + Docs API both need to be enabled — same requirement as PAC-57).
4. Email credential: **Credentials → New credential → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `GENERATED_DOCS_FOLDER_ID` | Reuse the exact same value from PAC-57/58 — matter folders this bot creates live in the same place PAC-58 falls back to creating them, so the two templates never disagree about where a matter's documents are |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_STAFF_EMAIL` | Where the matter-setup summary email goes — a staff/paralegal inbox, not a client address |
| `FIRM_FROM_EMAIL` | The From address for the summary email — falls back to `FIRM_EMAIL` if `FIRM_STAFF_EMAIL` isn't set |

No new variables to create for `GENERATED_DOCS_FOLDER_ID` if PAC-57/58 is already deployed.

---

## Step 3 — Configure document template variables (optional, 5 min per template)

Only needed if you want internal documents auto-copied into new matters. Skip this step entirely if not — the document stage just does nothing.

1. In Google Drive, create the internal templates you want copied in (e.g. "Matter Opening Checklist," "Case Plan"). These are staff-facing working documents, not client-facing paperwork — PAC-57 already handles client-facing intake documents.
2. Type the merge tags your templates need directly into the document text: `{{matter_display_number}}`, `{{matter_description}}`, `{{client_name}}`, `{{practice_area}}`, `{{open_date}}`.
3. Copy each template's file ID from its URL (`.../d/THIS_PART_IS_THE_ID/edit`) and set it as an n8n Variable, e.g. `TEMPLATE_MATTER_CHECKLIST_DOC_ID`, `TEMPLATE_CASE_PLAN_DOC_ID`. The variable names must match what you reference in Step 5's `DOC_TEMPLATE_MAP`.

---

## Step 4 — Set the standard folder structure (2 min)

Open the **Build Standard Subfolder List** node and edit `DEFAULT_FOLDERS` to match your firm's own matter folder structure. Ships with `Correspondence`, `Pleadings & Filings`, `Client Documents`, `Billing & Trust`, `Signed Agreements` — this list applies to every matter regardless of practice area.

---

## Step 5 — Configure practice-area setup (10–20 min, the main customization step)

Open the **Determine Practice Area Setup Config** node and edit the three maps, each keyed by practice area name exactly as it appears in Clio (Settings → Practice Areas):

- `DOC_TEMPLATE_MAP` — which document templates from Step 3 to copy in, per practice area
- `TASK_LIST_MAP` — the standard task checklist to create in Clio, with each task's due date as an offset (in calendar days) from the matter's open date
- `DEADLINE_RULE_MAP` — intake deadlines to calendar in Clio, with `offset_type: 'business'` (skips weekends and US federal holidays) or `'calendar'` (counts every day) — **replace the placeholder rules with your firm's actual rules before activating**

A practice area left out of any map, or a document template Variable left unset, simply skips that one step for matters in it — not an error.

---

## Step 6 — Activate

Toggle the workflow to **Active**. It polls Clio hourly on its own — no webhook URL to wire anywhere. Adjust the cron expression on **Poll For New Matters** to `*/15 * * * *` for faster pickup, or loosen it for firms that open matters in batches.

**First activation is a bootstrap run:** every currently-open matter in Clio gets marked as "already seen" without triggering setup — otherwise activating this template would fire folder/document/task/deadline creation for every matter your firm has ever opened, all at once. Only matters opened *after* activation get the full setup treatment.

---

## Step 7 — Test

1. In Clio, open a new test matter in a practice area you configured in Step 5.
2. Wait for the next poll (or run the workflow manually from n8n for an immediate test — the pinned test payload simulates one matter).
3. Confirm:
   - A new folder appears under `GENERATED_DOCS_FOLDER_ID`, named after the matter's display number, with the standard subfolders inside it.
   - Configured document templates appear in the folder with merge tags filled in.
   - Configured tasks appear on the matter in Clio with the right due dates.
   - Configured deadlines appear as calendar entries in Clio.
   - `FIRM_STAFF_EMAIL` receives a summary with links.
4. Open a matter in a practice area you *didn't* configure and confirm the folder tree still gets created, with the summary email noting no templates/tasks/deadlines were configured — not an error.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| First activation | Every existing open matter is marked seen but not processed — no setup fires for your existing matter list |
| New matter opened, practice area fully configured | Folder tree, document templates, task checklist, and deadlines all created |
| New matter opened, practice area not in any map | Folder tree still created; document/task/deadline stages skipped |
| PAC-58 already filed a document to this matter before setup ran | Reuses the existing Drive folder instead of creating a duplicate |
| Matter's `open_date` or `practice_area` missing from Clio's response | Falls back to today's date / empty string — check the pinned test payload and your Clio API response shape if this happens often |
| A single task or deadline API call fails | Logged in execution history; the rest of the checklist/deadlines still get created |

---

## Compliance note

This template performs operational setup only — folder creation, document copying with metadata merge fields, task assignment, and deadline date arithmetic from firm-entered rules. It never drafts legal text, never interprets a deadline, and never makes a legal judgment (ABA Op. 512). Every matter setup is logged to the Bar-Compliance Guardrail's audit sheet.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces the 5–10 minutes of manual folder/checklist work per new matter, plus the Clio template feature it stands in for |
| Compliance test | Folder/document/task/calendar setup only — no legal judgment, no interpreted deadlines |
| Searchability test | Ranks for "automated matter setup" |
| Deployability test | Folder tree alone is live in under 15 minutes; reuses Clio and Drive credentials from PAC-57/58 if already deployed |
| Upsell test | Quick-Win Build adds Clio custom-field population, e-signature engagement letters via DocuSign, and multi-office folder templates |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Auto-populating Clio custom fields from the intake data, not just tasks and folders
- Sending the engagement letter for e-signature via DocuSign as part of matter opening (not just copying a checklist)
- Per-office or per-attorney folder template variants for multi-location firms
- A weekly digest of every matter opened and what was auto-configured for it

[Book a call with Protomated](https://protomated.com/book) to get started.
