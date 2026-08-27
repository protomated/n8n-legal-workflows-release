# Deploy Guide: Statute of Limitations Watchdog

**Template:** PTPAC-67 — K2: Statute-of-Limitations Watchdog
**Pillar:** Keep
**Replaces:** Manual docketing suites

Get value in under 15 minutes. Every day, the workflow checks every open Clio task tagged as a statute-of-limitations deadline and escalates automatically as the date approaches — 180 days out, 90 days out, 30 days out, and then every single day once it's overdue, with a direct partner-level escalation the moment it crosses into overdue. A separate weekly rollup lists every tracked deadline firm-wide regardless of what the daily watchdog already alerted on, so silence can be trusted as "nothing changed," not "something broke." A missed statute of limitations is a worst-case malpractice event; this is pure insurance against it costing anyone their license.

---

## Before you start: heads up on this one

**This never calculates or interprets a legal deadline.** The firm enters the actual statute-of-limitations date as the due date on a Clio task — the same way any firm already dockets a deadline. This workflow only does date math against that firm-entered date and escalates. It cannot be wrong about *when* a statute runs, because it never decides that in the first place (ABA Op. 512).

**A deadline is just a Clio task, with one convention** — same pattern as PAC-65's document requests. Create a task named for the filing (e.g. "File complaint before statute of limitations expires"), set its due date to the actual deadline, assign it to the responsible attorney, and set its task type to **Statute of Limitations** (or whatever you set `FIRM_SOL_TASK_TYPE` to). Mark it complete in Clio once it's filed (or otherwise resolved) — that's the entire "stop alerting" mechanism. A matter can have more than one of these tasks if it has multiple claims with different deadlines.

**This workflow has two independent triggers on one canvas**, same structural pattern as PAC-66:

- **The daily watchdog** escalates 180/90/30 days out (each firing once, deduped via workflow static data) plus a continuous daily alert once overdue — and the moment a deadline crosses into overdue, it *also* fires a separate, direct escalation straight to a partner-level address, on top of the assignee's own alert. A missed deadline should never depend on one person seeing one email.
- **The weekly audit rollup** runs on its own schedule and lists *every* open statute-of-limitations task firm-wide, grouped by urgency — including ones still more than 180 days out that the daily watchdog never touches — regardless of the daily dedupe state. This exists because a quiet daily watchdog is ambiguous: it looks identical whether everything's fine or the workflow silently broke. The weekly rollup removes that ambiguity, and is sent even when there's nothing urgent (an explicit "all clear," not silence).

**Two real correctness issues were caught and fixed while building this — worth knowing about even though they're already handled:**
1. **A timezone off-by-one bug.** Clio's task `due_at` comes back as a bare date with no time. Naively parsing that and zeroing it to local midnight shifts the calendar date backward by a full day on any n8n server running west of UTC (most of the Americas) — confirmed live with a harness test (a task due in exactly 30 days computed as 29 days remaining under `America/New_York`). Fixed by anchoring the date at noon UTC first (same technique PAC-65 already uses), which keeps the correct local calendar date in any real-world timezone before zeroing out for day-count math.
2. **A disabled-assignee gap.** If a task's assignee is a departed employee whose Clio user account has since been disabled, the original design would still have silently addressed the alert to their dead inbox. Fixed to treat a disabled assignee (`enabled: false`) the same as an unresolved one, falling back to a live contact.

**This is purely internal — no client is ever contacted, and there is no opt-out gate.** The Bar-Compliance Guardrail (NTC-33) is called on the assignee alert and the weekly rollup, but only to log them to the audit sheet; the approval/suppression result is deliberately ignored, same fire-and-forget pattern already used for internal-only notifications elsewhere in this catalog. The partner escalation for an overdue deadline bypasses the guardrail entirely (same pattern as PAC-65's staff escalation) — it's a second internal notification about a deadline already logged by the assignee-alert branch, not a new send needing its own audit entry. A malpractice safety net must never be something a recipient can opt out of.

**No "first sighting, stay silent" grace period, unlike PAC-66.** If you deploy this against a matter that's already 20 days from its deadline, it alerts on the very first run — it does not wait for a second sighting the way PAC-66's status watchdog does. A malpractice deadline is not a case where "let's wait and see if this looks intentional" is an acceptable default.

**Not yet independently confirmed live: `task_type{id,name}`, `assignee{id,name}`, and `matter{id,display_number,description}` haven't been confirmed together in one `tasks.json` call**, though each is individually confirmed elsewhere in this catalog (PAC-59/65). Check your first real run's output on either task-fetch node and adjust field names if Clio rejects any of them.

**A firm with more than 200 total open tasks (of any type) could theoretically push genuine statute-of-limitations tasks past a single fetched page.** `limit=200` covers every 1-10 attorney firm this catalog targets comfortably, but check `meta.paging.next` on the task-fetch nodes if you suspect a deadline is being missed.

---

## What you need before you start

- A Clio account with API access, task and user-directory read scope (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61/63/64/65/66 if already deployed)
- A Clio task type set up for statute-of-limitations deadlines (default expected name: **Statute of Limitations**)
- Every tracked deadline as a task of that type, with a due date, assigned to a real Clio user
- An email account with SMTP access
- The Bar-Compliance Guardrail (NTC-33) already deployed (used here for audit logging only — this template does not depend on it for opt-out gating)

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-67-statute-of-limitations-watchdog.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-67-statute-of-limitations-watchdog.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide.
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | The fallback contact if a task's assignee can't be resolved, the recipient of the weekly audit rollup, and the fallback overdue-escalation target |
| `FIRM_FROM_EMAIL` | The sender address for alert emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_SOL_TASK_TYPE` *(optional)* | The exact Clio task type name to treat as a statute-of-limitations deadline — defaults to `Statute of Limitations` |
| `FIRM_PARALEGAL_EMAIL` *(optional)* | Fallback contact for the assignee alert, tried before `FIRM_EMAIL` |
| `FIRM_PARTNER_EMAIL` *(optional)* | Where overdue escalations go — falls back to `FIRM_EMAIL` if unset |

---

## Step 3 — Set the daily and weekly schedule times (2 min)

1. Open **"Check Statute Of Limitations Deadlines Daily"** and adjust the cron expression if 7am (n8n server timezone) isn't the right time for your firm.
2. Open **"Send Weekly Deadline Audit Rollup"** and adjust the cron expression if Monday 8am isn't the right day/time.

---

## Step 4 — Activate

Toggle the workflow to **Active**. Both schedules run on their own from then on — no webhook URL to wire anywhere.

---

## Step 5 — Test

The workflow ships with pinned sample Clio data on all four Clio-facing nodes.

**Daily watchdog branch** — a three-user firm directory (including one disabled account) and four tasks: one 180 days out (`tier_180`), one exactly 30 days out (`tier_30`), one 7 days overdue and assigned to the disabled user (tests both the overdue-escalation branch and the disabled-assignee fallback), and one that isn't a statute-of-limitations task at all (tests the type filter).

1. Run **"Fetch Statute Of Limitations Tasks From Clio"** → **"Split Out Deadline Task Records"** → **"Filter Statute Of Limitations Tasks With Due Date"** and confirm the "Document Request"-typed task is dropped, leaving 3 items.
2. Continue through **"Compute Escalation Tier And Check If Alert Needed"** and confirm the tiers: `tier_180`, `tier_30`, `overdue`, all with `needs_alert: true`.
3. Re-run the same node a second time without changing anything, and confirm the `tier_180` and `tier_30` items now show `needs_alert: false` while the `overdue` item still shows `needs_alert: true` (overdue never dedupes).
4. Continue through **"Filter Deadlines Needing An Alert Today"** and confirm it fans out to both **"Build Statute Of Limitations Alert Email"** and **"If Deadline Is Overdue"**.
5. Confirm the overdue task's alert resolves its recipient to `FIRM_EMAIL` (or `FIRM_PARALEGAL_EMAIL`), not the disabled user's address.
6. Confirm only the `overdue` task reaches **"Build Partner Escalation Email"** → **"Send Partner Escalation Email"** — the other two should hit **"Skip — Partner Escalation Not Needed"**.
7. Continue through **"Log Alert For Compliance Audit"** → **"Send Statute Of Limitations Alert Email"** and confirm delivery and styling differ by tier (amber → orange → red → dark red).

**Weekly audit rollup branch** — a two-user directory and two tasks, both far in the future, to demonstrate the `on_track` bucket the daily watchdog never touches.

8. Run **"Fetch All Statute Of Limitations Tasks For Audit"** → ... → **"Aggregate All Deadlines For Audit"** and confirm one item comes out with a `deadlines` array of 2.
9. Continue through **"Build Weekly Deadline Audit Email"** and confirm both entries appear under "More than 180 days out."
10. Temporarily clear the pinned data on **"Fetch All Statute Of Limitations Tasks For Audit"** down to an empty `data` array, re-run the branch, and confirm the digest still sends with an "all clear" message rather than going silent.

**Testing against your real Clio account:** unpin all four Clio-facing nodes, connect a real credential, and create one real "Statute of Limitations" task due in the next 30 days so it lands on `tier_30` immediately. Check each node's raw output before assuming anything is broken — see the field-name caveat above.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A tracked deadline is 180, 90, or 30 days out for the first time | An escalation email fires once for that tier |
| A tracked deadline stays in the same tier on a later run | No repeat email — already alerted for that tier |
| A tracked deadline is overdue and the task is still open | The assignee alert fires on **every daily run**, and a separate partner-level escalation also fires on every run |
| A task isn't the configured statute-of-limitations type, or has no due date | Excluded before tier computation even runs |
| A task's assignee can't be resolved, or is a disabled/departed user | The alert still sends — falls back to `FIRM_PARALEGAL_EMAIL` or `FIRM_EMAIL` |
| The deadline is resolved and the task is marked complete in Clio | It no longer appears in the `status=pending` fetch — all future alerts stop automatically, including the partner escalation |
| Every Monday, regardless of daily alert activity | A full firm-wide rollup lists every open deadline by urgency, sent even when there's nothing to report |

---

## Compliance note

This template performs scheduling and templated operational alerting only — pure date-math against a deadline the firm already entered, never any comment on the matter's substance or legal analysis of when a statute actually runs (ABA Op. 512). It never contacts a client. The Bar-Compliance Guardrail is used for audit logging only; alerts are never suppressible, since this is a malpractice safety net, not a marketing message.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | A missed statute of limitations is a worst-case malpractice event — this is insurance against the single most expensive mistake a firm can make |
| Compliance test | Pure countdown/reminder logic against a firm-entered date, no legal judgment, fully internal |
| Searchability test | Ranks for "statute of limitations tracker" |
| Deployability test | Reuses Clio and SMTP credentials from earlier templates if already deployed — under 15 minutes, no webhook to wire |
| Upsell test | Clear Quick-Win Build path: SMS alerts alongside email, a firm-wide deadline dashboard, and automatic task creation from an intake checklist by practice area |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- SMS alerts alongside email so an urgent deadline can't be missed in a crowded inbox
- A firm-wide dashboard showing every tracked deadline and its status at a glance
- Automatic statute-of-limitations task creation from a practice-area intake checklist, so nothing depends on someone remembering to set it up by hand
- Configurable escalation tiers and recipients per practice area

[Book a call with Protomated](https://protomated.com/book) to get started.
