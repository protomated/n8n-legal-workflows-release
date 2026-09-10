# Deploy Guide: Win-Back / Reactivation Drip

**Template:** PTPAC-94 — G7: Win-Back / Reactivation Drip
**Pillar:** Get
**Replaces:** Lawmatics

Get value in under 15 minutes. Lost and cold inquiries are never re-engaged, wasting prior marketing spend. This workflow re-engages them automatically on a sequenced, timed drip — no one has to remember to follow up with a lead who went quiet three weeks ago.

---

## Before you start: the most important thing to understand

**This is templated marketing copy, not legal work or a legal opinion on anyone's situation** (ABA Op. 512). The message content is fixed per stage and personalized only with the lead's name and practice area — never AI-generated, never tailored to case specifics.

**This never decides on its own that a lead is unreachable.** Opting out (respected automatically via the guardrail) or marking a lead "reactivated" is always a manual, firm-driven decision made directly in the tracker sheet — the workflow only reads that sheet, it never changes a lead's status itself except recording which drip stage was last sent.

**A lead catches up automatically if a run is missed — it never permanently skips a stage.** If your server is down for a few days, or a lead's countdown crosses two stage thresholds before the workflow next runs, only the NEXT stage in sequence goes out (not the furthest one that would technically be "due") — one stage at a time, always in order.

**This has two independent parts on one canvas:**
1. **Daily drip send** — works out which stage each active lead is due for and sends it.
2. **Weekly win-back performance rollup** — reports how many leads are active, reactivated, or unsubscribed, and flags anyone who's received every stage in the sequence.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed
- A Google account with Sheets access
- An email account with SMTP access
- A way to mark a lead "cold" or "lost" (manually, or wired from your CRM/intake form's own lost-lead action)

---

## Step 1 — Create the win-back tracker (3 min)

Create a new Google Sheet. Add a tab named exactly **Win-Back Tracker** with these column headers in row 1:

```
lead_id | name | email | practice_area | status | marked_cold_at | last_stage_sent | last_sent_at
```

Add a row for each lost/cold lead: `status` starts as `active`, `marked_cold_at` is the date they went cold (ISO format, e.g. `2026-09-10`), and `last_stage_sent` starts blank. Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (3 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-94-win-back-reactivation-drip.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-94-win-back-reactivation-drip.json). Do not activate it yet.
2. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | Sender address for all drip emails |
| `FIRM_EMAIL` | Recipient for the weekly rollup |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `WIN_BACK_TRACKER_SHEET_ID` | The Google Sheet ID from Step 1 |
| `WIN_BACK_DRIP_STAGES` *(optional)* | Semicolon-separated `day_offset|stage_label` pairs — defaults to `0|check-in;14|resources;30|last-call` |

---

## Step 4 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expressions on **"Check Win-Back Drip Daily"** (default 9am) and **"Check Weekly Win-Back Rollup"** (default Monday 8am) if different times suit.

---

## Step 5 — Test

The workflow ships with three pinned leads: one fresh (marked cold 5 days ago, never contacted), one mid-sequence (marked cold ~14 days ago, already received the `check-in` stage), and one already reactivated.

1. Run **"Fetch Win-Back Tracker"** → **"Compute Due Drip Stage Per Lead"** and confirm the fresh lead is due for `check-in`, the mid-sequence lead is due for `resources`, and the reactivated lead produces no output at all.
2. Edit the mid-sequence lead's pinned `marked_cold_at` to 40+ days ago (simulating a missed run) and re-run — confirm it's still only due for `resources`, not skipping ahead to `last-call`.
3. Edit that same lead's `last_stage_sent` to `last-call` and re-run — confirm no output (sequence complete).
4. Continue through **"Build Win-Back Drip Email"** and confirm the message is personalized with the lead's name and practice area.
5. Continue through the compliance-check chain and confirm the email sends (or routes to **"Skip — Lead Opted Out"** if you test with an opted-out address).
6. Continue through **"Update Tracker With Stage Sent"** and confirm it only updates `last_stage_sent` and `last_sent_at` for that lead — the row's other columns are untouched.

**Testing against your real setup:** unpin the Sheets nodes, connect real credentials, and add a real test lead with a `marked_cold_at` a few days in the past to confirm a real send.

**Weekly win-back rollup branch:**
7. Run **"Check Weekly Win-Back Rollup"** → **"Read Full Win-Back Tracker"** → **"Compute Weekly Win-Back Summary"** and confirm the active/reactivated/unsubscribed counts match the pinned data, and that a lead already on the final stage is correctly flagged in `completed_sequence_count`.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A lead has never been contacted and enough days have passed for the first stage | Sends the first stage's email |
| A lead is mid-sequence and enough days have passed for the next stage | Sends exactly the next stage, never skipping ahead even if more time has passed than needed |
| A lead has received every configured stage | No further emails send; flagged in the weekly rollup for a manual decision |
| A lead is marked `reactivated` or `unsubscribed` in the tracker | Skipped entirely, regardless of timing |
| A lead opts out via the guardrail | The send is suppressed; the tracker isn't advanced, so mark them `unsubscribed` directly in the sheet to stop future attempts cleanly |

---

## Compliance note

This template sends templated, non-personalized-beyond-name-and-practice-area marketing copy only — never legal advice or a judgment about anyone's situation (ABA Op. 512). The Bar-Compliance Guardrail is used on every send: opt-out respected, required disclaimer applied, send logged.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces a Lawmatics-style win-back campaign, recovering value from prior marketing spend on leads that never converted |
| Compliance test | Templated marketing copy only; no legal work, no automated judgment that a lead is unreachable |
| Searchability test | Ranks for "law firm lead win back campaign" |
| Deployability test | One Google Sheet plus standard SMTP credentials — under 15 minutes |
| Upsell test | Clear Done-for-You path: productized as a fixed-scope build with CRM-native lead tracking and reactivation detection beyond a manual sheet |
