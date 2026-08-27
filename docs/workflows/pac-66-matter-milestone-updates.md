# Deploy Guide: Matter Milestone Updates

**Template:** PTPAC-66 — K3: Proactive Matter Milestone Updates
**Pillar:** Keep
**Replaces:** A paid Case Status app

Get value in under 15 minutes. Every time a matter's status changes in Clio, this workflow sends the client a plain-English update automatically — no one has to remember to email them. Poor communication between milestones is one of the top bar-complaint drivers of client dissatisfaction; this closes that gap without adding anything to anyone's to-do list.

---

## Before you start: heads up on this one

**This is event-driven (a webhook), not a scheduled poll** — the only trigger in this catalog so far that isn't a daily cron. It fires the instant Clio sends a matter-updated event, not on a delay.

**The webhook payload is never trusted for data — only for which matter to look up.** Clio's `matter.updated` webhook fires on *any* field edit, not just status changes, and its payload shape can vary. So this workflow extracts only the matter id from it, then makes a fresh, authoritative `GET` call to Clio for that matter's current status, client, and responsible attorney. This also means a generic (non-Clio) practice management system can trigger this template too — it only needs to POST a `matter_id`.

**No database for dedupe — n8n's own workflow static data instead.** Since Clio fires this webhook on every edit, something has to distinguish "the status actually changed" from "someone edited the matter description." `Check If Status Actually Changed` caches each matter's last-known status directly inside the workflow (via `$getWorkflowStaticData`), with no external database or extra lookup call. One consequence worth knowing: **the very first webhook event this workflow ever sees for a given matter never sends an email** — there's no way to know what the status was immediately before that event, so it's cached silently rather than risking a misleading "your matter is now in X" message for an edit that had nothing to do with status. Every genuine status change after that first sighting sends normally.

**The entire content surface is one node you must edit before going live.** `Look Up Status Update Message` has a `STATUS_MESSAGES` map with placeholder example statuses. Replace the keys with your firm's actual Clio status names (case-sensitive) and the values with your firm's own plain-English, non-legal update copy. A status not in the map is silently treated as internal-only — no client email, no error. This is the compliance-required design: the workflow never generates, paraphrases, or interprets update text — it only fills a client's name and matter reference into text your firm already approved (ABA Op. 512).

**A few Clio details haven't been independently confirmed live:** the exact payload shape of Clio's `matter.updated` webhook event (this template deliberately doesn't rely on it beyond extracting an id, precisely because of this uncertainty), a single-resource `GET /matters/{id}.json` call, `status` as a returned field, and whether that single-resource response is wrapped in a `data` object the same way list endpoints are. Check your first real webhook delivery's output on `Fetch Matter Details From Clio` and adjust if anything differs.

**Confirmed live: Clio has no Settings UI for webhook management at all — it's an API-only action, and requires its own OAuth scope.** See Step 4 below for the exact API call. This also means the "Webhooks" scope (Read + Write) must be granted on your Clio OAuth2 app separately from Matters/Tasks/Users — if you add it after the credential was already authorized, you must reconnect the credential, same as the `ForbiddenError` lesson from PAC-64.

**Confirmed live: Clio webhooks expire.** 3 days by default, 31 days maximum. This is not a one-time setup — whoever deploys this needs a recurring reminder (or a separate small process) to re-create the webhook subscription before it expires, or it will silently stop delivering with no error visible anywhere in n8n.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first** — this template cannot send a client update without it.
- An n8n instance reachable from the internet (self-hosted with a public URL, or n8n Cloud) — Clio needs to be able to reach your webhook URL
- A Clio account with API access — matter, user-directory, **and Webhooks (Read + Write)** scope (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61/63/64/65 if already deployed, but add the Webhooks scope and reconnect if it wasn't already granted)
- A way to make one authenticated `POST` API call to Clio (curl, Postman, or a temporary n8n HTTP Request node) — there is no Settings UI for creating a webhook
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

## Step 4 — Activate and register the Clio webhook (10 min)

1. Toggle the workflow to **Active**.
2. Open **"When Matter Status Changes"**, click the **Production URL** tab (not Test URL — the Test URL only works while you're actively in "Listen for test event" mode in the editor), and copy it.
3. Confirm your Clio OAuth2 app has the **Webhooks** scope (Read + Write) granted, and that you've reconnected the credential in n8n if that scope was added after the credential was first authorized.
4. Register the webhook by making one authenticated `POST` request to Clio's API — there's no Settings page for this:

   ```
   POST https://eu.app.clio.com/api/v4/webhooks.json   (use your own CLIO_BASE_URL region)
   Authorization: Bearer <your Clio access token>
   Content-Type: application/json

   {
     "data": {
       "url": "<the Production URL you copied in step 2>",
       "model": "matter",
       "events": ["updated"],
       "fields": "id,status",
       "expires_at": "2026-09-25T00:00:00Z"
     }
   }
   ```

   Set `expires_at` to the maximum allowed (31 days from today) so you're not re-doing this every few days. The easiest way to fire this one-time call if you don't have curl/Postman handy: add a temporary **HTTP Request** node anywhere on your n8n canvas (method POST, the URL and body above, authenticated with your Clio OAuth2 credential), click **Execute step**, confirm success, then delete the node — it's not part of the deployed workflow.

5. **Set a reminder to redo step 4 before the webhook expires.** Clio webhooks are not permanent — 31 days is the maximum lifespan regardless of what you set. When it expires, deliveries stop silently; nothing in n8n will show an error, because n8n simply never receives another call. Recreating the webhook (repeat step 4 with a fresh `expires_at`) is the only fix.

---

## Step 5 — Test

The workflow ships with pinned sample data on the webhook trigger and both Clio-facing nodes, so you can test the logic without a real webhook delivery yet.

1. Run **"Fetch Matter Details From Clio"** → **"Check If Status Actually Changed"** and confirm `has_changed: false` the first time (nothing cached yet), then manually re-run it and confirm it's still `false` (same status, correctly suppressed).
2. Edit the pinned data's `status` field to a different value (e.g. `"Settlement Negotiation"`), re-run, and confirm `has_changed: true`.
3. Continue through **"Look Up Status Update Message"** and confirm `has_message: true` for a status you configured, and `false` for one you didn't.
4. Continue through **"Build Status Update Email"** and confirm the copy is personalized with the resolved attorney's contact details.
5. Continue through **"Check Compliance Before Sending"** → **"If Update Approved"** → **"Send Status Update Email"** and confirm the delivered email carries the guardrail's unsubscribe footer and renders the styled HTML.

**Testing against your real Clio account:** unpin all three Clio-facing nodes (trigger, matter fetch, users fetch), then actually change a real matter's status in Clio. The webhook should fire within seconds — check n8n's **Executions** tab for a run whose trigger source is the webhook, not a manually-started one. If nothing shows up there at all, the problem is upstream of n8n entirely: re-confirm the webhook was actually created (step 4) with the correct Production URL, that it hasn't expired, and that the workflow is Active. `GET https://eu.app.clio.com/api/v4/webhooks/{webhook_id}.json` (the id returned when you created it) should return the subscription's current status — a quick way to confirm it still exists and hasn't expired without needing any UI.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A matter's status genuinely changes to one with a configured message | The client gets a personalized update email (unless opted out) |
| A matter is edited but its status doesn't change | Skipped — no email, no error |
| The very first webhook event ever received for a matter | Cached silently, no email — there's no prior status to compare against |
| A matter's status changes to one with no configured message | Skipped — treated as internal-only |
| The webhook payload has no usable matter id | Skipped — invalid or unrelated event |
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
| Deployability test | Reuses Clio and SMTP credentials from earlier templates if already deployed — under 15 minutes, plus one webhook to wire in Clio |
| Upsell test | Clear Quick-Win Build path: SMS updates alongside email, a client portal timeline view, and richer status-specific content (next-steps checklists per milestone) |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- SMS updates alongside email for clients who respond faster to text
- A branded client portal showing the full matter timeline, not just the latest update
- Status-specific next-steps checklists instead of a single message per status
- Multi-language message templates for firms with non-English-speaking clients

[Book a call with Protomated](https://protomated.com/book) to get started.
