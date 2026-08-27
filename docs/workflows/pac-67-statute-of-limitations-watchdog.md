# Deploy Guide: Statute of Limitations Watchdog

**Template:** PTPAC-67 — K2: Statute-of-Limitations Watchdog
**Pillar:** Keep
**Replaces:** Manual docketing suites

Get value in under 15 minutes. Every day, the workflow checks every open Clio task tagged as a statute-of-limitations deadline and escalates automatically as the date approaches — 180 days out, 90 days out, 30 days out, and then every single day once it's overdue. A missed statute of limitations is a worst-case malpractice event; this is pure insurance against it costing anyone their license.

---

## Before you start: heads up on this one

**This never calculates or interprets a legal deadline.** The firm enters the actual statute-of-limitations date as the due date on a Clio task — the same way any firm already dockets a deadline. This workflow only does date math against that firm-entered date and escalates. It cannot be wrong about *when* a statute runs, because it never decides that in the first place (ABA Op. 512).

**A deadline is just a Clio task, with one convention** — same pattern as PAC-65's document requests. Create a task named for the filing (e.g. "File complaint before statute of limitations expires"), set its due date to the actual deadline, assign it to the responsible attorney, and set its task type to **Statute of Limitations** (or whatever you set `FIRM_SOL_TASK_TYPE` to). Mark it complete in Clio once it's filed (or otherwise resolved) — that's the entire "stop alerting" mechanism. A matter can have more than one of these tasks if it has multiple claims with different deadlines.

**The overdue tier never stops alerting, on purpose.** The 180/90/30-day tiers each fire exactly once (tracked via n8n's own workflow static data, no external database) — but if a deadline passes and the task is still open, this workflow alerts on **every single daily run**, indefinitely, until someone marks the task complete. This is deliberately more aggressive than every other template in this catalog: staying quiet about a missed statute of limitations is a categorically worse outcome than a repeated email.

**No "first sighting, stay silent" grace period, unlike PAC-66.** If you deploy this against a matter that's already 20 days from its deadline, it alerts on the very first run — it does not wait for a second sighting the way PAC-66's status watchdog does. A malpractice deadline is not a case where "let's wait and see if this looks intentional" is an acceptable default.

**This is purely internal — no client is ever contacted, and there is no opt-out gate.** The Bar-Compliance Guardrail (NTC-33) is still called, but only to log the alert to the audit sheet; its approval/suppression result is deliberately ignored, same fire-and-forget pattern already used for internal-only notifications elsewhere in this catalog. A malpractice safety net must never be something a recipient can opt out of.

**A couple of Clio field combinations haven't been independently confirmed live together:** `task_type{id,name}`, `assignee{id,name}`, and `matter{id,display_number,description}` are each individually confirmed elsewhere in this catalog (PAC-59/65), but not all three together in one `tasks.json` call. Check your first real run's output on `Fetch Statute Of Limitations Tasks From Clio` and adjust field names if Clio rejects any of them.

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
| `FIRM_EMAIL` | The escalation address CC'd once a deadline reaches the 30-day or overdue tier, and the fallback recipient if a task's assignee can't be resolved |
| `FIRM_FROM_EMAIL` | The sender address for alert emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_SOL_TASK_TYPE` *(optional)* | The exact Clio task type name to treat as a statute-of-limitations deadline — defaults to `Statute of Limitations` |

---

## Step 3 — Set the daily check time (2 min)

Open **"Check Statute Of Limitations Deadlines Daily"** and adjust the cron expression if 7am (n8n server timezone) isn't the right time for your firm.

---

## Step 4 — Activate

Toggle the workflow to **Active**. It runs on its own from then on — no webhook URL to wire anywhere.

---

## Step 5 — Test

The workflow ships with pinned sample Clio data: a two-user firm directory, and four tasks covering the main scenarios — one 180 days out (`tier_180`), one exactly 30 days out (`tier_30`, resolved assignee), one 7 days overdue with no assignee (tests the fallback-to-firm-email path), and one that isn't a statute-of-limitations task at all (tests the type filter).

1. Run **"Fetch Statute Of Limitations Tasks From Clio"** → **"Split Out Deadline Task Records"** → **"Filter Statute Of Limitations Tasks With Due Date"** and confirm the "Document Request"-typed task is dropped, leaving 3 items.
2. Continue through **"Compute Escalation Tier And Check If Alert Needed"** and confirm the tiers: `tier_180`, `tier_30`, `overdue` respectively, all with `needs_alert: true` (nothing cached yet).
3. Re-run the same node a second time without changing anything, and confirm the `tier_180` and `tier_30` items now show `needs_alert: false` (already alerted for that tier) while the `overdue` item still shows `needs_alert: true` (overdue never dedupes).
4. Continue through **"Filter Deadlines Needing An Alert Today"** and confirm all 3 survive the first run.
5. Continue through **"Build Statute Of Limitations Alert Email"** and confirm each tier's tone, color, and subject line differ, and that the overdue item (no assignee) correctly falls back to `FIRM_EMAIL` as the recipient with no duplicate CC to itself.
6. Continue through **"Log Alert For Compliance Audit"** → **"Send Statute Of Limitations Alert Email"** and confirm delivery, including the CC on the 30-day item.

**Testing against your real Clio account:** unpin both Clio-facing nodes, connect a real credential, and create one real "Statute of Limitations" task due in the next 30 days so it lands on `tier_30` immediately. Check each node's raw output before assuming anything is broken — see the field-name caveat above.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A tracked deadline is 180, 90, or 30 days out for the first time | An escalation email fires once for that tier |
| A tracked deadline stays in the same tier on a later run | No repeat email — already alerted for that tier |
| A tracked deadline is overdue and the task is still open | An alert fires on **every single daily run** until the task is marked complete |
| A task isn't the configured statute-of-limitations type, or has no due date | Excluded before tier computation even runs |
| A task's assignee can't be resolved | The alert still sends — falls back to `FIRM_EMAIL` as the recipient |
| A deadline reaches the 30-day or overdue tier | `FIRM_EMAIL` is CC'd in addition to the assignee (unless they're the same address) |
| The document/deadline is resolved and the task is marked complete in Clio | It no longer appears in the `status=pending` fetch — all future alerts stop automatically |

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
