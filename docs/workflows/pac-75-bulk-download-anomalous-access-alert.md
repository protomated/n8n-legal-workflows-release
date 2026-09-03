# Deploy Guide: Bulk-Download & Anomalous-Access Alert

**Template:** PTPAC-75 — OPS8: Bulk-Download & Anomalous-Access Alert
**Pillar:** Ops
**Replaces:** Enterprise SIEM/DLP tools, manual log review

Get value in under 15 minutes. A phished or careless account can quietly download a firm's entire matter archive before anyone notices — solo and small firms have no SIEM and no one watching access logs. This workflow watches Google Workspace's own Drive access log for unusual bulk downloads or access bursts, and emails the firm admin the moment anyone crosses a threshold — with the user, the matters touched, and the volume.

---

## Before you start: the most important thing to understand

**This workflow only ever reads metadata — who accessed which file name and when.** It never opens, reads, or analyzes what's actually inside a document. It can't tell you whether a downloaded file contained privileged material, only that it was downloaded. That boundary is deliberate: this is security monitoring, not legal review or judgment of any kind (ABA Op. 512).

**This has two independent parts on one canvas:**
1. **Hourly anomaly alert** — pulls the last hour of Drive activity, counts downloads/views per user, and immediately emails the firm admin if anyone crosses a bulk-download or access-burst threshold. A per-user cooldown (default 6 hours) stops the same person from re-triggering an email every single hour while they're still elevated.
2. **Weekly visibility digest** — re-reads the last 7 days of the same log and always sends a plain summary of who accessed the most files firm-wide, even when nothing crossed a threshold — so the firm has ongoing visibility, not just alarms.

**Coverage today is Google Workspace Drive only.** If your firm's documents live primarily in Clio, MyCase, or another practice management system rather than Google Drive, this template won't see that activity — Microsoft 365 / SharePoint audit log support and PM-system document logs are natural extensions (coming soon).

**The matter reference in each alert comes from the file name, not a system lookup.** The template looks for a pattern like `2026-CH-050` inside the file's title. If your firm's Drive files and folders don't already include the matter number in their names, alerts will still fire correctly but will show "Unmatched / General Files" instead of a specific matter — still useful, just less precise. Adjust `MATTER_REF_PATTERN` in "Normalize Drive Access Event" if your firm uses a different numbering convention.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — alert audit logging goes through it
- A Google Workspace **Super Admin** account (needed once, for the setup step below)
- An email account with SMTP access

---

## Step 1 — Grant read access to your Drive audit log (5 min, the one real setup step)

This is the only non-trivial part of this template, and it only has to be done once:

1. In the [Google Cloud Console](https://console.cloud.google.com/), create (or reuse) a project and enable the **Admin SDK API**.
2. Create a Service Account, generate a JSON key for it, and note its Client ID.
3. In the **Google Workspace Admin Console** (admin.google.com) → Security → API Controls → Domain-wide Delegation, add that Client ID with this OAuth scope:
   ```
   https://www.googleapis.com/auth/admin.reports.audit.readonly
   ```
4. In n8n, create a **Google API** credential using that service account and scope.

If your firm already has a Google API credential set up for another integration with this scope included, you can reuse it here instead of creating a new one.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-75-bulk-download-anomalous-access-alert.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-75-bulk-download-anomalous-access-alert.json). Do not activate it yet.
2. Google API credential: the one you created in Step 1.
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | The sender address for alert and digest emails |
| `SECURITY_ALERT_EMAIL` | Where these alerts should go — usually your office manager or IT contact, not necessarily the same inbox as other firm notifications. Falls back to `FIRM_EMAIL` if unset. |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `BULK_DOWNLOAD_THRESHOLD` *(optional)* | Downloads/prints by one user in one hour that trigger an alert — defaults to 15 |
| `ACCESS_BURST_THRESHOLD` *(optional)* | Total file accesses by one user in one hour that trigger an alert — defaults to 40 |
| `ALERT_COOLDOWN_HOURS` *(optional)* | How long to wait before re-alerting the same user — defaults to 6 |
| `LOOKBACK_HOURS` *(optional)* | How far back the hourly check looks — defaults to 1, should match your schedule cadence |

---

## Step 4 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expressions on **"Check Google Drive Access Logs Hourly"** (default every hour) and **"Send Weekly Access Pattern Summary"** (default Monday 8am) if different timing suits your firm.

---

## Step 5 — Test

The workflow ships with pinned sample Drive activity: 4 downloads/views by `paralegal@smithlaw.com` across two matters, 1 view by `attorney@smithlaw.com`, and 1 edit (correctly excluded, since edits aren't an access-relevant event).

**Hourly alert branch:**
1. Run **"Fetch Recent Drive Activity From Google Workspace"** → **"Split Out Drive Activity Records"** → **"Normalize Drive Access Event"** and confirm each item shows the right `matter_ref` (two items resolve to `2026-CH-050`, two to `2026-CH-051`, one to `2026-CH-052`, and the edit resolves to `Unmatched / General Files` but with `is_access_event: false`).
2. Continue through **"Filter Access-Relevant Events"** and confirm 5 of the 6 items survive (the edit is dropped).
3. Continue through **"Aggregate Access Counts By User"** and confirm `paralegal@smithlaw.com` shows `access_count: 4, download_count: 3` and `attorney@smithlaw.com` shows `access_count: 1, download_count: 0`.
4. Continue through **"Check Against Anomaly Thresholds"** — with the default thresholds (15/40), confirm `needs_alert: false` for both users, since the pinned sample data is intentionally modest.
5. **To see the alert path fire:** temporarily set `BULK_DOWNLOAD_THRESHOLD` to `2` in Settings → Variables, then re-run from "Check Against Anomaly Thresholds" — confirm `paralegal@smithlaw.com` now shows `needs_alert: true`. Continue through **"Filter Users Needing An Alert"** → **"Build Anomalous Access Alert Email"** and confirm the email correctly lists both matters and the 3 sample file names. Set the threshold back to its real value afterward.
6. Re-run the same chain a second time immediately after and confirm `needs_alert` is now `false` again for that user — the cooldown is working.

**Weekly digest branch:**
7. Run **"Fetch Weekly Drive Activity From Google Workspace"** through **"Aggregate Weekly Access By User"** and confirm `total_events: 5`, `total_users: 2`, with `paralegal@smithlaw.com` listed first (highest volume).
8. Continue through **"Build Weekly Access Pattern Summary Email"** and confirm the table renders both users correctly.
9. Temporarily clear the pinned data on **"Fetch Weekly Drive Activity From Google Workspace"** to an empty `items: []`, re-run, and confirm the digest still sends with a "No Google Drive activity logged this week" message rather than an empty or broken table.

**Testing against your real setup:** unpin both Google Workspace nodes, connect the real Google API credential, and manually run each branch (Execute step, don't wait for the schedule). Real alert volume will depend entirely on your firm's actual Drive usage — if nothing crosses the default thresholds during initial testing, temporarily lower `BULK_DOWNLOAD_THRESHOLD` the same way as step 5 above to confirm the email actually arrives, then restore it.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A user downloads/prints more files than `BULK_DOWNLOAD_THRESHOLD` in an hour | Firm admin is emailed with the user, matters touched, and sample file names |
| A user accesses (view/download/print) more files than `ACCESS_BURST_THRESHOLD` in an hour | Same alert, triggered by total volume rather than downloads specifically |
| A user stays above threshold for several hours in a row | Only the first hour alerts; the cooldown suppresses repeats until it expires |
| A file name doesn't match the firm's matter numbering pattern | Still counted and alerted on, shown as "Unmatched / General Files" |
| A week passes with zero Drive activity | The weekly digest still sends, explicitly saying so |
| Every week, regardless of alerts | A firm-wide top-10 access summary is sent for visibility |

---

## Compliance note

This template performs security monitoring only — counting and comparing access-log metadata (who, what file name, when, how many times) against firm-configured thresholds, never reading, analyzing, or making any legal judgment about document content (ABA Op. 512). The Bar-Compliance Guardrail is used for internal audit logging only; these are internal security notifications to the firm admin, not client-facing messages, and are never suppressible by an opt-out.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces enterprise SIEM/DLP tooling and manual log review; catches account compromise/insider exfiltration before a firm's entire matter archive walks out the door |
| Compliance test | Metadata-only monitoring, no document content ever read, no legal judgment made |
| Searchability test | Ranks for "law firm data breach bulk download alert" |
| Deployability test | One Google service account credential plus SMTP — under 15 minutes once the domain-wide delegation scope is granted |
| Upsell test | Clear Done-for-You path: multi-platform coverage (Microsoft 365, PM systems), managed threshold tuning, and incident response coordination |

---

## Want the advanced version?

Protomated builds the done-for-you **Done-for-You** implementation that adds:
- Microsoft 365 / SharePoint and practice-management-system audit log coverage alongside Google Workspace
- Managed threshold tuning based on your firm's actual usage patterns, reducing false positives
- Direct incident response coordination — account lockout, forced re-authentication, and IT notification
- A firm-wide access dashboard instead of email-only alerts

[Book a call with Protomated](https://protomated.com/book) to get started.
