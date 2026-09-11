# Deploy Guide: New Client Kickoff Kit

**Template:** PTPAC-99 — New Client Welcome and Onboarding Sequence (scoped)
**Replaces:** Manual Drive folder setup, manual kickoff scheduling, manual onboarding tracking

Get value in under 15 minutes. The moment a client is marked signed, this workflow builds their Drive matter folder, sends a kickoff email with a scheduling link, creates the firm's standard follow-up tasks in Clio, and logs the whole thing — no one has to remember any of these setup steps by hand.

---

## Before you start: this is scoped to complement, not duplicate, the Client Onboarding Sequence (K7)

**This assumes your welcome-email day sequence is already handled elsewhere.** If you haven't deployed the Client Onboarding Sequence template (`pac-64-client-onboarding-sequence.json`) yet, deploy that first — it already covers the Day 0/1/3/7 welcome, what-to-expect, and document-checklist emails. This kit is deliberately scoped to what that template doesn't do: the Drive matter folder, the Calendly kickoff link, and a master onboarding log.

**This never touches legal work** (ABA Op. 512) — folder creation, scheduling links, and task creation are pure administrative setup.

**This has two independent parts on one canvas:**
1. **Kickoff kit** — fires when a client is marked signed: creates the Drive folder structure, sends the kickoff email, creates follow-up tasks, and logs it all.
2. **Weekly onboarding rollup** — reports how many clients were onboarded this week and flags any Drive folder creation failures for manual follow-up.

**A Drive folder creation failure never blocks the rest of kickoff.** If the parent folder can't be created (e.g. a permissions issue), the kickoff email, follow-up tasks, and master log still proceed — the failure is just flagged in the weekly rollup instead. Subfolders are only attempted if the parent folder actually succeeded.

---

## What you need before you start

- The Client Onboarding Sequence template (K7) already deployed, for the welcome-email sequence
- The Bar-Compliance Guardrail (NTC-33) already deployed
- A Google Cloud project with the Drive API enabled and OAuth credentials with Drive scope
- A Clio account with API access (reuse the OAuth2 credential from other templates in this catalog if already deployed)
- A Calendly account with a "kickoff meeting" event type set up
- A Google account with Sheets access
- An email account with SMTP access
- A way to trigger this when a client signs (a button, a CRM/PM tool's own webhook, or Zapier)

---

## Step 1 — Create a Drive parent folder for matter folders (2 min)

Create (or choose) a Google Drive folder to hold every new matter folder this workflow creates. Copy its folder ID from the URL — the part after `/folders/`.

---

## Step 2 — Create the master onboarding log (3 min)

Create a new Google Sheet. Add a tab named exactly **Master Onboarding Log** with these column headers in row 1:

```
client_name | matter_id | practice_area | drive_folder_url | folder_created | onboarded_at
```

Copy the Sheet's ID from its URL.

---

## Step 3 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-99-new-client-kickoff-kit.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-99-new-client-kickoff-kit.json). Do not activate it yet.
2. Google API credential (for Drive): **Credentials → New → Google API**, with Drive scope. Manually select it on **"Create Matter Drive Folder"** and **"Create Drive Subfolder"** after import.
3. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog.
4. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
5. Email credential: **Credentials → New → SMTP**.

---

## Step 4 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | Sender address for the kickoff email |
| `FIRM_EMAIL` | Recipient for the weekly onboarding rollup |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `ONBOARDING_DRIVE_PARENT_FOLDER_ID` | The Drive folder ID from Step 1 |
| `CALENDLY_KICKOFF_EVENT_URL` | Your firm's Calendly scheduling link for the kickoff meeting event type |
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates |
| `ONBOARDING_LOG_SHEET_ID` | The Google Sheet ID from Step 2 |
| `ONBOARDING_DRIVE_SUBFOLDERS` *(optional)* | Comma-separated subfolder names — defaults to `Correspondence,Documents,Discovery,Billing` |
| `ONBOARDING_FOLLOWUP_TASKS` *(optional)* | Semicolon-separated `day_offset|task_name` pairs — defaults to `3|Send engagement letter;7|Confirm signed retainer received` |

---

## Step 5 — Activate and wire your "signed" trigger (3 min)

1. Toggle the workflow to **Active**.
2. Open **"When Client Marked Signed"**, copy its **Production URL**, and wire it to whatever marks a client signed in your process (a button, your CRM/PM tool's own webhook, or a Zapier/Make step).

---

## Step 6 — Test

The workflow ships with a pinned signed-client payload (Jordan Blake, Family Law) and a pinned successful Drive folder creation response.

1. Run **"Parse Signed Client Data"** → **"Create Matter Drive Folder"** → **"Parse Folder Creation Response"** and confirm `folder_created: true` with a `drive_folder_url`.
2. Continue through **"Build Subfolder List"** and confirm 4 subfolders are queued (the default list).
3. Edit the pinned Drive response to `{"error": {"message": "insufficientPermissions"}}`, re-run from **"Parse Folder Creation Response"**, and confirm `folder_created: false` and **"Build Subfolder List"** now produces zero subfolders — while **"Build Kickoff Email"** and **"Build Onboarding Follow-Up Task List"** still proceed normally.
4. Continue through **"Build Kickoff Email"** and confirm the message includes your configured Calendly link (or the graceful "not configured yet" fallback if you haven't set one).
5. Continue through **"Build Onboarding Follow-Up Task List"** and confirm 2 tasks are queued with due dates 3 and 7 days out.
6. Continue through **"Log Onboarding To Master Sheet"** and confirm a row would be logged regardless of the folder/email outcomes.

**Testing against your real setup:** unpin the webhook and Drive/Clio nodes, connect real credentials, and submit a real test "signed" payload to confirm a real Drive folder gets created.

**Weekly onboarding rollup branch:**
7. Run **"Check Weekly Onboarding Rollup"** → **"Read Full Onboarding Log"** → **"Compute Weekly Onboarding Summary"** and confirm the pinned data (one successful, one failed folder creation, one outside the 7-day window) produces `total_onboarded: 2, failed_folder_count: 1`.
8. Continue through **"Build Weekly Onboarding Email"** and confirm the failed folder is listed, including the "no new clients" case if you clear the pinned data.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A client is marked signed | Drive folder + subfolders created, kickoff email sent, follow-up tasks created in Clio, everything logged |
| The Drive folder creation fails | Kickoff email, follow-up tasks, and master log still proceed; the failure is flagged in the weekly rollup for manual follow-up |
| `CALENDLY_KICKOFF_EVENT_URL` isn't set | The kickoff email still sends, with a placeholder note instead of a broken link |
| A client opts out via the guardrail | The kickoff email is suppressed; folder creation, tasks, and logging still proceed |
| No new clients onboarded in a week | The weekly rollup still sends, explicitly saying so |

---

## Compliance note

This template performs administrative setup only — folder creation, scheduling link delivery, and task creation, never legal work or judgment (ABA Op. 512). The Bar-Compliance Guardrail is used on the kickoff email: opt-out respected, required disclaimer applied, send logged.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Recovers the staff time spent manually creating matter folders, scheduling kickoff calls, and tracking onboarding steps |
| Compliance test | Administrative setup only; no legal work or judgment |
| Searchability test | Ranks for "law firm client onboarding automation" |
| Deployability test | One Drive parent folder plus standard Clio/Sheets/SMTP credentials — under 15 minutes once Calendly is set up |
| Upsell test | Clear Done-for-You path: Clio matter creation, DocuSign, Stripe/LawPay, and a client portal beyond Google-tools onboarding |
