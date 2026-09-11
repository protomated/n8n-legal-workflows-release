# Deploy Guide: Automated Lead Follow-Up Sequence

**Template:** PTPAC-97 — Automated Lead Follow-Up Sequence
**Replaces:** Manual lead follow-up tracking, LeadDocket-style follow-up sequencing

Get value in under 15 minutes. Leads go quiet after the first inquiry because nobody has time to follow up three separate times over a week. This workflow acknowledges every lead instantly, follows up on Day 1, 3, and 7 automatically, stops the moment they reply, and alerts the attorney directly if a lead goes seven days with no response at all.

---

## Before you start: the most important thing to understand

**This sends templated marketing/operational copy only — never legal advice or a judgment about anyone's case** (ABA Op. 512). Every email is compliance-checked and logged through the Bar-Compliance Guardrail.

**This works well for a handful of leads a week.** At meaningfully higher volume (20+/week), a full CRM pipeline with proper lead scoring and routing is the better fit — that's the natural conversion point to a Done-for-You engagement, not something this free template tries to solve.

**This has three independent parts on one canvas:**
1. **Instant acknowledgment** — fires the moment a new lead comes in.
2. **Daily follow-up sequence** — advances each still-unresponsive lead through Day 1 → Day 3 → Day 7, one stage at a time, catching up automatically if a run is missed rather than skipping ahead. If Day 7 passes with no response, the attorney gets a direct alert instead of another automated email.
3. **Response detection** — watches your inbox for a reply from a tracked lead and marks them responded immediately, stopping all future follow-ups.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed
- A Google account with Sheets access
- An email account with SMTP access (for sending)
- The same inbox's IMAP access (for detecting replies)
- An inquiry form (JotForm, Typeform, Fluent Forms, or similar) that can POST to a webhook

---

## Step 1 — Create the lead follow-up tracker (3 min)

Create a new Google Sheet. Add a tab named exactly **Lead Follow-Up Tracker** with these column headers in row 1:

```
lead_id | name | email | phone | practice_area | submitted_at | responded | last_stage_sent | last_sent_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-97-automated-lead-followup-sequence.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-97-automated-lead-followup-sequence.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.
4. IMAP credential: **Credentials → New → IMAP** → the same inbox's server/login details (so replies can be detected).

---

## Step 3 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | Sender address for all lead-facing emails |
| `FIRM_ATTORNEY_EMAIL` | Who receives the no-response alert after the final stage |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `LEAD_FOLLOWUP_SHEET_ID` | The Google Sheet ID from Step 1 |
| `FOLLOW_UP_STAGES` *(optional)* | Semicolon-separated `day_offset|stage_label` pairs — defaults to `1|day1;3|day3;7|day7` |

---

## Step 4 — Activate and wire your form (3 min)

1. Toggle the workflow to **Active**.
2. Open **"When Lead Form Submitted"**, copy its **Production URL**, and wire it to your inquiry form's webhook/notification settings.

---

## Step 5 — Test

The workflow ships with a pinned lead submission (Jamie Torres, Family Law), two pinned tracker rows (one fresh, one already at Day 3), and a pinned reply email matching the fresh lead's address.

1. Run **"Parse Lead Submission"** → **"Build Instant Acknowledgment Email"** and confirm the message uses the family-law-specific copy.
2. Continue through the compliance check and confirm it sends, then continue to **"Log Lead For Follow-Up Tracking"** and confirm the row would be logged with `responded: false`.
3. Separately, run **"Fetch Lead Follow-Up Tracker"** → **"Compute Due Follow-Up Stage Per Lead"** and confirm the pinned Day-3 lead is flagged due for `day3` (not skipping ahead), and that whether the fresh lead is due depends on its `submitted_at` relative to today.
4. Edit the Day-3 lead's pinned `last_stage_sent` to `day3` and re-run — confirm it's now due for `day7` with `is_final_stage: true`.
5. Continue through **"If Final Stage Reached"** → **"Build No-Response Alert Email"** and confirm it reads correctly and is addressed to `FIRM_ATTORNEY_EMAIL`.
6. Separately, run **"When Lead Replies Via Email"** → **"Parse Reply Sender"** → **"Read Tracker For Reply Match"** → **"Match Reply To Tracked Lead"** and confirm the pinned reply (from Jamie Torres's address) matches the tracker row and continues to **"Mark Lead As Responded"**.
7. Edit the pinned reply's `from` address to one not in the tracker, re-run, and confirm **"Match Reply To Tracked Lead"** produces no output (ignored, not an error).

**Testing against your real setup:** unpin the webhook, Sheets, and IMAP nodes, connect real credentials, and submit a real test lead through your actual inquiry form.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A new lead submits the form | Gets an instant, practice-area-aware acknowledgment; logged for follow-up |
| A lead hasn't responded and enough days have passed for the next stage | Sends exactly the next stage, never skipping ahead |
| A lead replies at any point | Marked responded immediately; no further follow-ups send |
| A lead reaches Day 7 with no response | The attorney gets a direct alert instead of another automated email |
| A lead opts out via the guardrail | The send is suppressed; mark them `responded: true` directly in the tracker to stop future attempts cleanly |
| A reply comes from an address not in the tracker | Ignored — not every inbound email is from a tracked lead |

---

## Compliance note

This template sends templated marketing/operational copy only — never legal advice or a judgment about anyone's case (ABA Op. 512). The Bar-Compliance Guardrail is used on every lead-facing send: opt-out respected, required disclaimer applied, send logged.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Recovers leads that would otherwise go cold from lack of timely follow-up, without adding staff workload |
| Compliance test | Templated marketing copy only, no legal advice, no automated judgment about a lead's case |
| Searchability test | Ranks for "law firm lead follow up automation" |
| Deployability test | One Google Sheet plus standard SMTP/IMAP credentials — under 15 minutes |
| Upsell test | Clear Done-for-You path: a full CRM pipeline for firms with meaningfully higher lead volume |
