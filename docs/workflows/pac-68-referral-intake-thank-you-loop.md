# Deploy Guide: Referral Intake & Thank-You Loop

**Template:** PTPAC-68 — C8: Referral Intake & Thank-You Loop
**Pillar:** Convert
**Replaces:** Lawmatics referral management

Get value in under 15 minutes. Every time a referred lead comes through your intake form, this workflow logs it to a referral ledger, thanks the referrer personally, and alerts the firm — automatically. Once a month, it also surfaces anyone who's referred you multiple times recently, so a generous referral source never quietly goes unthanked. Referrals that go untracked and unthanked are a leak that dries up over time; this closes it.

---

## Before you start: heads up on this one

**This has two independent triggers on one canvas** — a webhook for real-time referral intake, and a monthly schedule for the reciprocal-referral reminder. They don't depend on each other at all; the schedule just re-reads the same ledger the webhook writes to.

**A referral is identified by one thing: a referrer's name on the intake form.** Add a "Referred by" field (and ideally "Referred by email") to your intake form. A submission with no referrer name at all simply isn't a referral as far as this template is concerned — it's not this workflow's job to handle normal (non-referred) intake at all; that's a separate process (e.g. PAC-20's Clio Intake Sync).

**The referral ledger is a Google Sheet, not Clio.** Referral tracking doesn't map cleanly onto a confirmed, reliable Clio field (unlike other templates in this catalog that lean on Clio matters/tasks), so this template uses the same proven pattern as PAC-32's Lead-Source ROI Tracker: log everything to a Google Sheet you own and can review directly, with pivot tables if you want them later.

**The referrer thank-you never mentions the lead's legal matter.** It's deliberately a short, generic thank-you — no practice area, no case details, nothing that could reveal anything about why the referred person is seeking legal help. That's the lead's confidential business, not the referrer's.

**The referrer is treated like any other contact who can opt out.** Even though a referrer isn't necessarily a current client, they're still a real person receiving an automated firm email, so the thank-you goes through the Bar-Compliance Guardrail the same way a client communication would — opt-out respected, unsubscribe footer appended, send logged.

**The reciprocal-referral reminder is a nudge, not an action.** It never decides who the firm should refer back to, drafts anything on the firm's behalf, or sends anything to the referrer. It's an internal digest to the firm listing who's earned some reciprocity — what to do about it is entirely a human business-development decision.

**Not yet independently confirmed live:** whether Google Sheets returns the ledger's `date` column back in the exact ISO format it was written in, or reformats it depending on the column's cell format. Check `Read Referral Ledger`'s raw output on your first real monthly run and adjust the date parsing in `Compute Referral Counts By Referrer` if it differs.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — the referrer thank-you goes through it
- A Google account with Sheets access
- An intake form (Fluent Forms or similar) with a "Referred by" field, ideally also "Referred by email"
- An email account with SMTP access

---

## Step 1 — Create the referral ledger (3 min)

Create a new Google Sheet. Add a tab named exactly **Referrals** with these column headers in row 1:

```
date | lead_name | lead_email | referrer_name | referrer_email | referrer_type | matter_type | status
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-68-referral-intake-thank-you-loop.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-68-referral-intake-thank-you-loop.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2** → sign in with the account that owns the ledger sheet.
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | Where the "new referral" alert and the monthly reciprocity digest are sent |
| `FIRM_FROM_EMAIL` | The sender address for all emails this template sends |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `REFERRAL_LEDGER_SHEET_ID` | The Google Sheet ID you copied in Step 1 |

---

## Step 4 — Wire your intake form and set the monthly schedule (3 min)

1. Toggle the workflow to **Active**.
2. Open **"When Referral Form Is Submitted"**, copy the **Production URL**, and paste it into your intake form's webhook/notification settings (e.g. Fluent Forms → Form Settings → Notifications → Webhooks → add this URL, format JSON).
3. Open **"Send Reciprocal Referral Reminder Monthly"** and adjust the cron expression if the 1st of the month at 8am isn't the right time for your firm.

---

## Step 5 — Test

The workflow ships with pinned sample data on the webhook trigger and both Google-Sheets-facing nodes.

**Intake branch:**
1. Run **"When Referral Form Is Submitted"** → **"Normalize Referral Submission"** → **"If Valid Referral Submission"** and confirm the pinned Dr. Susan Chen referral is recognized (`is_valid_referral: true`).
2. Continue through **"Log Referral To Ledger"** and confirm a new row would be appended (or check your actual Sheet after unpinning).
3. Continue through **"If Referrer Has Email"** → **"Build Referral Thank You Email"** → **"Check Compliance Before Thanking Referrer"** → **"If Thank You Approved"** → **"Send Referral Thank You Email"** and confirm the delivered email is warm, brief, and mentions nothing about the lead's matter.
4. Confirm **"Build Firm Referral Notification"** → **"Send Firm Referral Notification"** fires with a summary including the thank-you status.
5. Edit the pinned data's `referrer_name` to an empty string, re-run, and confirm it now routes to **"Skip — Not A Referral"**.
6. Edit the pinned data's `referrer_email` to an empty string (keep `referrer_name`), re-run, and confirm it reaches **"Skip — No Referrer Email On File"** but still logs to the ledger and still notifies the firm.

**Monthly reminder branch:**
7. Run **"Read Referral Ledger"** → **"Compute Referral Counts By Referrer"** and confirm the pinned data shows Dr. Susan Chen with `referral_count_last_90_days: 2` and Linda Ray with `1`.
8. Continue through **"Filter Referrers Needing A Reciprocity Nudge"** and confirm only Dr. Susan Chen survives (Linda Ray is below the threshold of 2).
9. Continue through **"Aggregate Referrers For Reminder"** → **"Build Reciprocal Referral Reminder Email"** and confirm the digest lists Dr. Susan Chen with her count.
10. Temporarily clear the pinned data on **"Read Referral Ledger"** to an empty array, re-run, and confirm the digest still sends with a "no repeat referrers this period" message rather than going silent.

**Testing against your real setup:** unpin the webhook trigger and both Google-Sheets nodes, submit a real test referral through your actual intake form, and confirm the row lands in your real Sheet and the emails arrive.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A form submission includes a referrer's name | Logged to the ledger; a thank-you and firm alert follow |
| A form submission has no referrer at all | Dropped immediately — not this template's concern |
| The referrer left an email address | They get a personal, compliance-checked thank-you |
| The referrer left only a name, no email | Still logged to the ledger; no thank-you can be sent, firm is still notified |
| The referrer has opted out of firm emails | Thank-you suppressed and logged by the guardrail; ledger entry and firm alert still happen |
| A referrer sends 2+ referrals in a trailing 90-day window | Surfaced in the monthly digest as a reciprocity candidate |
| No referrer meets the 90-day threshold in a given month | The digest still sends, with an explicit "nothing to act on" message |

---

## Compliance note

This template performs scheduling and templated relationship-management messaging only — a thank-you note and a business-development nudge, never legal work, legal advice, or any commentary on a referred lead's matter (ABA Op. 512). Every referrer-facing send goes through the Bar-Compliance Guardrail for opt-out checking, the required disclaimer, and audit logging. The monthly reminder is entirely internal and never contacts a referrer directly.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Referrals that go untracked and unthanked are a business-development leak that quietly dries up a firm's best lead source over time |
| Compliance test | Operational relationship-management only, referrer-facing sends fully wrapped in OPS1, no legal content ever |
| Searchability test | Ranks for "law firm referral tracking" |
| Deployability test | One Google Sheet plus standard SMTP credentials — under 15 minutes |
| Upsell test | Clear Quick-Win Build path: SMS thank-yous, automatic referral-source deduplication, a referral leaderboard dashboard, and reciprocal-referral suggestions matched by practice area |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- SMS thank-yous alongside email
- Automatic deduplication so the same referrer/lead pair never double-logs
- A referral leaderboard dashboard instead of a raw spreadsheet
- Practice-area-matched reciprocal-referral suggestions, so the firm knows exactly who to send business back to

[Book a call with Protomated](https://protomated.com/book) to get started.
