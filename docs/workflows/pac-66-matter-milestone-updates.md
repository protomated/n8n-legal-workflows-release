# Deploy Guide: Matter Milestone Updates

**Template:** PTPAC-66 — K3: Proactive Matter Milestone Updates
**Pillar:** Keep
**Replaces:** A paid Case Status app

Get value in under 15 minutes. Once a day, this workflow checks every Clio matter that's changed recently and sends the client a plain-English update automatically the moment its status genuinely moves — no one has to remember to email them. Poor communication between milestones is one of the top bar-complaint drivers of client dissatisfaction; this closes that gap without adding anything to anyone's to-do list.

---

## Before you start: heads up on this one

**This is a daily poll, not a webhook.** An earlier version of this template tried to fire instantly off a Clio webhook. Live testing surfaced enough friction that it isn't worth it for most firms building this themselves: Clio has no Settings UI for webhooks at all (API-only registration), it requires its own OAuth scope, a newly created webhook silently sits in status `pending` and delivers nothing until a one-time verification handshake is answered correctly, subscriptions expire every 31 days, and — the one that actually killed it — a firm's day-to-day "status" is very often a custom practice-area workflow stage (e.g. "In Review," "In Progress"), not the same field Clio's built-in matter webhooks watch. A daily poll sidesteps all of that: same dedupe, build, compliance, and send logic, just triggered by a schedule instead of a webhook. The cost is up to a day of delay instead of instant delivery — a trade almost every firm will take for something that reliably works.

**The webhook payload is never trusted for data — only the field selection here is.** `Fetch Recently Updated Matters From Clio` pulls each matter's authoritative current status, client, and responsible attorney straight from Clio on every run, using Clio's `updated_since` filter (confirmed API v4 parameter) to only pull matters that changed in roughly the last 2 days — a small overlap buffer so a cron running a bit late never misses one.

**No database for dedupe — n8n's own workflow static data instead.** Since this fetch pulls every recently-touched matter (not just ones with a status change), something has to distinguish "the status actually changed" from "someone edited the matter description." `Check If Status Actually Changed` caches each matter's last-known status directly inside the workflow (via `$getWorkflowStaticData`), with no external database or extra lookup call. One consequence worth knowing: **the first time this workflow ever sees a given matter, it never sends an email for it** — there's no way to know what the status was immediately before that first sighting, so it's cached silently rather than risking a misleading "your matter is now in X" message. Every genuine status change after that first sighting sends normally.

**The entire content surface is one node you must edit before going live.** `Look Up Status Update Message` has a `STATUS_MESSAGES` map with placeholder example statuses. Replace the keys with your firm's actual Clio status names (case-sensitive) and the values with your firm's own plain-English, non-legal update copy. A status not in the map is silently treated as internal-only — no client email, no error. This is the compliance-required design: the workflow never generates, paraphrases, or interprets update text — it only fills a client's name and matter reference into text your firm already approved (ABA Op. 512).

**Know which field your firm actually calls "status."** Before customizing the STATUS_MESSAGES map, open a real matter in Clio and check: is the value you want to track Clio's built-in status field (Open/Pending/Closed), or a custom practice-area workflow stage your firm configured? `Fetch Recently Updated Matters From Clio` requests the top-level `status` field on the matter resource — if your firm actually tracks progress through custom stages instead, this template may need that field's actual name adjusted (check this node's raw output against what you see in Clio's UI before assuming the wrong thing is broken).

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first** — this template cannot send a client update without it.
- A Clio account with API access — matter and user-directory read scope (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61/63/64/65 if already deployed)
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-66-matter-milestone-updates.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-66-matter-milestone-updates.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide.
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Customize the status message map (5 min)

Open **"Look Up Status Update Message"** and replace the `STATUS_MESSAGES` object with your firm's own Clio statuses and firm-approved update copy. Example:

```js
const STATUS_MESSAGES = {
  'Discovery': "We've moved into the discovery phase...",
  'Awaiting Court Date': "We're currently waiting for the court to schedule...",
  // add every status you want clients notified about
};
```

Any Clio status you don't list here is treated as internal-only and never reaches a client.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | The generic contact address used when the responsible attorney can't be resolved |
| `FIRM_FROM_EMAIL` | The sender address for update emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |

---

## Step 4 — Set the daily check time and activate (2 min)

1. Open **"Check Matter Statuses Daily"** and adjust the cron expression if 8am (n8n server timezone) isn't the right time for your firm.
2. Toggle the workflow to **Active**. It runs on its own from then on — no webhook URL to wire anywhere, nothing to register with Clio.

---

## Step 5 — Test

The workflow ships with pinned sample data on both Clio-facing nodes: a two-user firm directory, and three matters covering the main scenarios — one with a status that has a configured message (`Discovery`), one with no client email on file (to confirm the Filter step), and one with a status that has no configured message (`Intake`).

1. Run **"Fetch Recently Updated Matters From Clio"** → **"Split Out Matter Records"** → **"Filter Matters With Client Email"** and confirm the no-client-email matter is dropped, leaving 2 items.
2. Continue through **"Check If Status Actually Changed"** and confirm both remaining matters show `has_changed: false` — expected on a first run, since there's nothing cached yet.
3. Re-run the same node (or the whole workflow) a second time without changing anything — still `false` (same status, correctly suppressed).
4. Manually edit the pinned data's `status` field for the `Discovery` matter to a different configured value (e.g. `"Closed"`), re-run, and confirm `has_changed: true` for that matter this time.
5. Continue through **"Look Up Status Update Message"** and confirm `has_message: true` for the `Discovery`/`Closed` matter, and `false` for the `Intake` one.
6. Continue through **"Build Status Update Email"** and confirm the copy is personalized with the resolved attorney's contact details.
7. Continue through **"Check Compliance Before Sending"** → **"If Update Approved"** → **"Send Status Update Email"** and confirm the delivered email carries the guardrail's unsubscribe footer and renders the styled HTML.

**Testing against your real Clio account:** unpin both Clio-facing nodes, then actually change a real matter's status in Clio. Manually run the workflow once — you don't need to wait for the daily schedule while testing. Check each node's raw output in turn; in particular confirm `Fetch Recently Updated Matters From Clio` actually picked up your change (if it didn't, double-check the status field you changed is the same one named in this node's `fields` query parameter — see the "Know which field your firm actually calls 'status'" note above).

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A matter's status genuinely changes to one with a configured message | The client gets a personalized update email (unless opted out) — on the next daily run |
| A matter is edited but its status doesn't change | Skipped — no email, no error |
| The first time this workflow ever sees a given matter | Cached silently, no email — there's no prior status to compare against |
| A matter's status changes to one with no configured message | Skipped — treated as internal-only |
| A matter's client has no email on file | Filtered out before status is even checked |
| The client has opted out of firm emails | Guardrail suppresses the send; already logged to the audit sheet |
| The Clio matter or user-directory fetch fails | Degrades gracefully (`neverError`) rather than crashing the run |

---

## Compliance note

This template performs scheduling and templated operational notification only — it never generates, paraphrases, or interprets case status commentary. Every message is a firm-approved, pre-written template selected by a status lookup; the workflow only fills in the client's name and matter reference (ABA Op. 512). Every send goes through the Bar-Compliance Guardrail for opt-out checking, the required disclaimer, and audit logging.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces a paid Case Status app; poor communication between milestones is a top driver of bar complaints and client attrition |
| Compliance test | Firm-approved, non-legal template messages only, fully wrapped in OPS1 |
| Searchability test | Ranks for "automated client case updates" |
| Deployability test | Reuses Clio and SMTP credentials from earlier templates if already deployed — under 15 minutes, no webhook or external API call to register |
| Upsell test | Clear Quick-Win Build path: instant webhook-based delivery done right (with the account/scope/field setup handled for the firm), SMS updates alongside email, a client portal timeline view, and richer status-specific content (next-steps checklists per milestone) |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Instant delivery via a properly configured Clio webhook, set up and maintained for the firm
- SMS updates alongside email for clients who respond faster to text
- A branded client portal showing the full matter timeline, not just the latest update
- Status-specific next-steps checklists instead of a single message per status
- Multi-language message templates for firms with non-English-speaking clients

[Book a call with Protomated](https://protomated.com/book) to get started.
