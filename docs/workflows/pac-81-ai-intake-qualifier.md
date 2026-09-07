# Deploy Guide: AI Intake Qualifier

**Template:** PTPAC-81 — C7: AI Intake Qualifier (routing)
**Pillar:** Convert
**Replaces:** Smith.ai AI, Intaker

Get value in under 15 minutes. Unqualified inquiries eat attorney time while good leads wait behind them. This workflow scores every intake submission against your firm's own practice areas and keyword rules, logs it, replies appropriately, and immediately alerts an attorney for the highest-fit leads.

---

## Before you start: the most important thing to understand

**This is rule-based eligibility routing, not an AI assessment of case merits.** It matches the submitted practice area against a list you configure, and scans for keywords you configure — it never evaluates whether someone has a good case, gives a legal opinion, or makes a binding decision to reject anyone. Every submission gets logged and gets a reply; "decline" tier is just a lower-urgency lane a human can always override, never a hard automatic no (ABA Op. 512).

**This overlaps LegalContext's own Intake Screening feature**, which does true AI-based fit scoring beyond keyword matching. Lower-fit replies (referral and decline tiers) include a soft mention of it — that's this template's upsell path.

**The scoring is deliberately simple and transparent:** a base score, +30 for a practice-area match, +20 for an urgency keyword, -50 for a disqualifying keyword, clamped 0–100. Only a match *plus* an urgency signal reaches the "hot" tier — a plain practice-area match alone lands in "warm," not "hot," so attorney-alert volume stays meaningful.

**This has three independent parts on one canvas:**
1. **Score, log, and reply** — fires on every new intake submission.
2. **Hot lead alert** — a parallel branch off the same submission, only for the highest-fit tier.
3. **Weekly intake quality summary** — a firm-wide digest of the tier mix, average fit score, and top practice areas over the trailing week, so patterns are visible beyond a single submission at a time.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — the lead's auto-reply goes through it
- A Google account with Sheets access
- An email account with SMTP access
- An intake form (Fluent Forms or similar) with fields for name, email, phone, practice area, and case description

---

## Step 1 — Create the intake log (3 min)

Create a new Google Sheet. Add a tab named exactly **Intake Log** with these column headers in row 1:

```
name | email | phone | practice_area | fit_score | routing_tier | submitted_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-81-ai-intake-qualifier.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-81-ai-intake-qualifier.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_PRACTICE_AREAS` | Comma-separated list of practice areas your firm actually handles, e.g. `Personal Injury,Estate Planning,Family Law` |
| `FIRM_FROM_EMAIL` | Sender address for auto-replies |
| `FIRM_EMAIL` | Fallback recipient for hot-lead alerts if `INTAKE_ATTORNEY_EMAIL` isn't set |
| `INTAKE_ATTORNEY_EMAIL` | Who gets the immediate hot-lead alert |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `INTAKE_LOG_SHEET_ID` | The Google Sheet ID from Step 1 |
| `URGENT_KEYWORDS` *(optional)* | Comma-separated urgency signals — defaults to `arrested,court date,eviction notice,restraining order,emergency` |
| `DISQUALIFYING_KEYWORDS` *(optional)* | Comma-separated disqualifying signals — defaults to `already have a lawyer,already retained,just researching,student project` |
| `INTAKE_SUMMARY_LOOKBACK_DAYS` *(optional)* | How many trailing days the weekly summary covers — defaults to 7 |

---

## Step 4 — Activate and wire the intake form (3 min)

1. Toggle the workflow to **Active**.
2. Open **"When Intake Form Submitted"**, copy its **Production URL**, and wire it to your intake form's webhook/notification settings.

---

## Step 5 — Test

The workflow ships with a pinned submission: Alex Rivera, Personal Injury, mentioning a car accident and an upcoming court date — designed to hit both the practice-area match and an urgency keyword.

1. Run **"Parse Intake Submission"** → **"Score Lead Fit And Determine Routing"** and confirm `fit_score: 100, routing_tier: "hot"` (assuming `FIRM_PRACTICE_AREAS` includes Personal Injury).
2. Continue through **"Log Intake For Tracking"** → **"Build Lead Auto-Reply"** and confirm the message reads as a "we'll be in touch shortly" hot-tier reply.
3. Edit the pinned data's `case_description` to remove the court-date mention (e.g. "Looking into a claim, no rush"), re-run from Score — confirm it now lands in `"warm"` tier, not `"hot"`.
4. Edit `practice_area` to something outside your configured list (e.g. "Immigration"), re-run — confirm `routing_tier: "referral"`.
5. Edit `case_description` to include a disqualifying phrase (e.g. "I already have a lawyer"), re-run — confirm `routing_tier: "decline"` even though the practice area still matches.
6. For the hot case, continue through **"If Hot Lead"** → **"Build Hot Lead Alert Email"** and confirm the internal alert includes the full inquiry details.

**Testing against your real setup:** unpin the webhook trigger and Sheets node, connect real credentials, and submit a real test inquiry through your actual intake form.

**Weekly intake quality summary branch:**
7. Run **"Check Weekly Intake Quality Summary"** → **"Read Weekly Intake Log"** and confirm the 3 pinned rows come through.
8. Continue through **"Compute Weekly Intake Quality Summary"** and confirm `total_submissions: 3` (1 hot, 1 warm, 1 referral) with `Personal Injury` as the top practice area (count 2) and an average fit score of `77`.
9. Continue through **"Build Weekly Intake Quality Summary Email"** and confirm the message includes the tier breakdown, average fit score, and top-practice-areas list.
10. Temporarily clear the pinned data on **"Read Weekly Intake Log"** to an empty array, re-run, and confirm the summary still sends, explicitly saying no submissions came in rather than going silent.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Practice area matches and an urgency keyword is present | Hot tier — lead gets a "we'll be in touch shortly" reply, attorney gets an immediate alert |
| Practice area matches, no urgency signal | Warm tier — standard "we'll be in touch" reply, no immediate alert |
| Practice area doesn't match anything the firm handles | Referral tier — polite reply suggesting they seek a specializing firm, soft LegalContext mention |
| A disqualifying phrase is present, regardless of practice area | Decline tier — polite reply, still logged for human review |
| Every submission, any tier | Logged to the Intake Log sheet |
| A week passes | The intake attorney gets a summary of that week's tier mix, average fit score, and top practice areas |

---

## Compliance note

This template performs configuration-based eligibility routing only — matching a submitted practice area against a firm-provided list and scanning for firm-provided keywords, never assessing case merits or giving legal judgment (ABA Op. 512). Every tier still receives a reply and is still logged; nothing is automatically and irreversibly rejected. The Bar-Compliance Guardrail is used on the lead-facing auto-reply (opt-out respected, disclaimer applied, send logged) and for internal audit logging on the hot-lead alert.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Unqualified inquiries eating attorney time is a direct cost; hot-lead alerting recovers response speed on the leads that matter most |
| Compliance test | Rule-based routing only, no case-merit assessment, nothing automatically and permanently rejected |
| Searchability test | Ranks for "ai client intake qualification law firm" |
| Deployability test | One Google Sheet plus standard SMTP credentials — under 15 minutes |
| Upsell test | Direct overlap with LegalContext's own Intake Screening — true AI-based fit scoring beyond these keyword rules |

---

## Want the advanced version?

LegalContext's own **Intake Screening** feature goes further:
- True AI-based fit scoring instead of keyword matching
- Practice-area detection even when a lead doesn't self-select correctly
- Automatic referral-partner matching for out-of-scope inquiries
- Full intake analytics beyond a flat sheet

[Book a call with Protomated](https://protomated.com/book) to see LegalContext in action.
