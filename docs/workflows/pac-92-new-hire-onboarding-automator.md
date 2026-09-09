# Deploy Guide: New-Hire Onboarding Automator

**Template:** PTPAC-92 — OPS7: New-Hire Onboarding Automator
**Pillar:** Ops
**Replaces:** Generic HR onboarding checklist tools

Get value in under 15 minutes. Onboarding a new hire by hand is slow, and steps like account access and training get missed. This workflow provisions a Google Workspace account, sends a welcome email with your SOP pack, schedules your firm's training sessions, and flags a few good first matters — all the moment HR submits the new-hire form.

---

## Before you start: the most important thing to understand

**This never grants real matter access or makes a hiring/role decision on anyone's behalf.** Account provisioning creates a login — nothing more. Flagging a "first matter" creates a Clio task for a human to act on, not a permissions change. Nothing here touches legal work (ABA Op. 512) — it's HR/IT provisioning only.

**This inherits the Bar-Compliance Guardrail (OPS1), even though the recipient is an employee, not a client.** The ticket for this template specifically calls for that. In practice, the opt-out/disclaimer checks the guardrail runs are effectively inert for an employee — but the audit log entry is still useful, and it keeps this template consistent with every other send in the catalog.

**This has four independent parts on one canvas, all fanning out from the same account-provisioning step:**
1. A compliance-checked welcome email with your SOP pack links.
2. Calendar invites for your firm's configured training sessions.
3. Clio tasks flagging a few good first matters for the new hire's supervising attorney.
4. An immediate kickoff summary to HR, sent without waiting on the other three to finish.

**A provisioning failure never blocks the rest of onboarding.** If the Google Workspace account can't be created automatically (e.g. it already exists), the workflow still sends the welcome email, schedules training, and flags matters — the failure is just noted in the HR summary for manual follow-up.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed
- A Google Workspace admin account with Super Admin access (for account provisioning)
- A Google Cloud project with OAuth credentials scoped to `admin.directory.user` and domain-wide delegation
- A Google Calendar OAuth2 credential
- A Clio account with API access (reuse the OAuth2 credential from other templates in this catalog if already deployed)
- A Google account with Sheets access
- An email account with SMTP access
- An HR intake form (Fluent Forms or similar) with fields for name, personal email, role, start date, and practice areas

---

## Step 1 — Create the training matters list (3 min)

Create a new Google Sheet. Add a tab named exactly **Training Matters** with these column headers in row 1:

```
matter_id | matter_name | practice_area
```

Add a few rows for matters your team considers good, low-risk matters to walk a new hire through — this list is entirely yours to curate; the workflow never picks a matter on its own. Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (10 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-92-new-hire-onboarding-automator.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-92-new-hire-onboarding-automator.json). Do not activate it yet.
2. Google API credential (for account provisioning): **Credentials → New → Google API**. This needs a service account or OAuth client with domain-wide delegation and the `https://www.googleapis.com/auth/admin.directory.user` scope, authorized in your Google Workspace admin console. This is the one genuinely involved setup step — Google's own documentation for "domain-wide delegation" walks through it.
3. On **"Provision Google Workspace Account"**, manually select the Google API credential from Step 2 (it isn't pre-wired by import).
4. Google Calendar credential: **Credentials → New → Google Calendar OAuth2 API**, save as "Google Calendar account."
5. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog.
6. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
7. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | Fallback recipient for the HR kickoff summary if `HR_NOTIFICATION_EMAIL` isn't set |
| `FIRM_FROM_EMAIL` | Sender address for both the welcome email and the HR summary |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `FIRM_EMAIL_DOMAIN` | Your firm's email domain, e.g. `smithlaw.com` — used to build the new hire's firm email as `firstname.lastname@yourdomain` |
| `FIRM_SOP_LINKS` | Comma-separated `Label|URL` pairs, e.g. `Employee Handbook|https://...,Time Tracking Guide|https://...` |
| `TRAINING_SESSIONS` | Semicolon-separated `day_offset|title|duration_hours`, e.g. `1|IT & Systems Walkthrough|1;3|Practice Management System Training|2;5|Compliance & Culture Onboarding|1` |
| `TRAINING_MATTERS_SHEET_ID` | The Google Sheet ID from Step 1 |
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates |
| `HR_NOTIFICATION_EMAIL` *(optional)* | Who receives the kickoff summary — falls back to `FIRM_EMAIL` |
| `TRAINING_MATTER_COUNT` *(optional)* | How many matters get flagged per new hire — defaults to 3 |

---

## Step 4 — Activate and wire your HR form (3 min)

1. Toggle the workflow to **Active**.
2. Open **"When New Hire Form Submitted"**, copy its **Production URL**, and wire it to your HR intake form's webhook/notification settings.

---

## Step 5 — Test

The workflow ships with a pinned submission (Morgan Reyes, Associate Attorney, starting 2026-09-21, Personal Injury and Family Law), a pinned successful account-provisioning response, and three pinned training-matters rows (one Personal Injury, one Estate Planning, one Family Law).

1. Run **"Parse New Hire Submission"** → **"Build New Account Credentials"** and confirm `new_account_email: "morgan.reyes@smithlaw.com"` (assuming `FIRM_EMAIL_DOMAIN` is set to `smithlaw.com`) and a 14-character `temp_password`.
2. Continue through **"Provision Google Workspace Account"** → **"Parse Provisioning Response"** and confirm `account_provisioned: true`.
3. Continue through **"Build Welcome Email"** → **"Check Compliance Before Welcome Email"** → **"If Welcome Email Approved"** → **"Send Welcome Email"** and confirm the message includes the new account email, temp password, and SOP links.
4. Separately, continue from **"Parse Provisioning Response"** through **"Build Training Session List"** and confirm 3 sessions on 2026-09-22, 2026-09-24, and 2026-09-26 (1, 3, and 5 days after the start date).
5. Separately, continue through **"Fetch Training-Suitable Matters"** → **"Select Matters For New Hire"** and confirm exactly 2 of the 3 pinned matters are selected (Personal Injury and Family Law — the Estate Planning one is excluded since it doesn't match either of Morgan's practice areas).
6. Separately, continue through **"Build Onboarding Kickoff Summary"** and confirm it reads correctly and doesn't wait on the other three branches.
7. Edit the pinned data on **"Provision Google Workspace Account"** to `{"error": {"message": "Entity already exists."}}`, re-run from **"Parse Provisioning Response"**, and confirm `account_provisioned: false`, the welcome email now says the account is still being set up, and the HR summary surfaces the failure — while training and matter-flagging still proceed normally.

**Testing against your real setup:** unpin the webhook, provisioning, and Sheets nodes, connect real credentials, and submit a real test new-hire form.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A new-hire form is submitted | Account provisioning, welcome email, training scheduling, and matter flagging all fire from the same submission |
| Account provisioning succeeds | Welcome email includes the new firm email and temp password |
| Account provisioning fails (e.g. already exists) | Welcome email says IT will follow up; the failure is called out in the HR summary; everything else still proceeds |
| No training matters match the new hire's practice areas | Falls back to the first `TRAINING_MATTER_COUNT` matters on the list rather than flagging nothing |
| The new hire has opted out somehow (matches an existing opt-out entry) | The welcome email is suppressed — same opt-out check as every other send in this catalog |

---

## Compliance note

This template performs account provisioning, scheduling, and task creation only — it never grants real matter access, evaluates the new hire, or makes a role/hiring decision, and it never touches legal work (ABA Op. 512). The Bar-Compliance Guardrail is used on the welcome email per this template's ticket, even though the recipient is an employee rather than a client.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces a manual HR onboarding checklist and recovers the hours lost to missed steps (account setup, training scheduling, matter introductions) |
| Compliance test | Provisioning, scheduling, and task creation only; no real access granted automatically, no legal work touched |
| Searchability test | Ranks for "law firm employee onboarding automation" |
| Deployability test | One Google API domain-wide-delegation setup plus standard Calendar/Clio/Sheets/SMTP credentials — under 15 minutes once Google Workspace delegation is granted |
| Upsell test | Clear Quick-Win Build path: productized as a fixed-scope build with firm-specific provisioning rules and deeper PM-system integration |
