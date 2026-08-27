# Deploy Guide: Matter Milestone Updates

**Template:** PTPAC-66 — K3: Proactive Matter Milestone Updates
**Pillar:** Keep
**Replaces:** A paid Case Status app

Get value in under 15 minutes. Every time a matter's status changes in Clio, this workflow sends the client a plain-English update automatically — no one has to remember to email them. Poor communication between milestones is one of the top bar-complaint drivers of client dissatisfaction; this closes that gap without adding anything to anyone's to-do list.

---

## Before you start: heads up on this one

**Confirmed live: Clio's matter "status" is a built-in field with exactly three values — Open, Pending, Closed.** It lives on the matter's own Dashboard tab, next to the client name — not on a Task (Tasks have their own separate Pending/Complete status, which is a different field entirely and this template does not watch it), and not a custom field, unless your firm has separately configured one. If your firm tracks day-to-day progress through custom practice-area stages (e.g. "In Review," "In Progress"), that's a different concept — check the actual Matter Status control before assuming this template is broken.

**This is event-driven (a webhook), not a scheduled poll** — the only trigger in this catalog so far that isn't a daily cron. It fires the instant Clio reports a matter status transition, not on a delay.

**Subscribed specifically to `matter_opened`, `matter_pended`, and `matter_closed` — Clio's dedicated events for status transitions, not the generic `updated` event.** Clio's docs describe these three event names as firing "whenever a matter's status changes to" each value, which is more precise than watching `updated` with a fields filter and means this webhook fires only on genuine status changes, nothing else.

**The webhook payload is never trusted for data — only for which matter to look up.** This workflow extracts only the matter id from the webhook payload, then makes a fresh, authoritative `GET` call to Clio for that matter's current status, client, and responsible attorney. This also means a generic (non-Clio) practice management system can trigger this template too — it only needs to POST a `matter_id`.

**No database for dedupe — n8n's own workflow static data instead.** `Check If Status Actually Changed` caches each matter's last-known status directly inside the workflow (via `$getWorkflowStaticData`), with no external database or extra lookup call — mainly a safety net against a redelivered or duplicate event, since the webhook is already scoped to real status transitions. One consequence worth knowing: **the very first webhook event this workflow ever sees for a given matter never sends an email** — there's no way to know what the status was immediately before that event, so it's cached silently rather than risking a misleading message. Every genuine status change after that first sighting sends normally.

**The entire content surface is one node you must edit before going live.** `Look Up Status Update Message` has a `STATUS_MESSAGES` map defaulting to Pending/Closed copy — replace it with your firm's own plain-English, non-legal wording. A status not in the map is silently treated as internal-only — no client email, no error. `Open` is deliberately left unconfigured by default since it's the status every new matter starts in, and PAC-64 (Client Onboarding Sequence) already sends a day-0 welcome email when a matter opens — add an `Open` entry only if you're not also running PAC-64. This is the compliance-required design: the workflow never generates, paraphrases, or interprets update text — it only fills a client's name and matter reference into text your firm already approved (ABA Op. 512).

**A few Clio details haven't been independently confirmed live:** the exact payload shape of the matter_opened/matter_pended/matter_closed webhook events (this template deliberately doesn't rely on it beyond extracting an id, precisely because of this uncertainty), a single-resource `GET /matters/{id}.json` call, `status` as a returned field, and whether that single-resource response is wrapped in a `data` object the same way list endpoints are. Check your first real webhook delivery's output on `Fetch Matter Details From Clio` and adjust if anything differs.

**Confirmed live: Clio has no Settings UI for webhook management at all — it's an API-only action, and requires its own OAuth scope.** The "Webhooks" scope (Read + Write) must be granted on your Clio OAuth2 app separately from Matters/Tasks/Users — if you add it after the credential was already authorized, you must reconnect the credential, same as the `ForbiddenError` lesson from PAC-64.

**Confirmed live: Clio webhooks expire (3 days by default, 31 days maximum) — but this template handles that for you.** Rather than asking you to remember a recurring manual API call, the workflow ships with a second, independent trigger branch — `Check Webhook Subscription Daily` and the nodes after it — that runs once a day, checks whether the webhook is still valid, and (re)creates it automatically if it's missing or close to expiring. This branch also doubles as the way you register the webhook the very first time: no curl, no Postman, no temporary nodes, no extracting a bearer token from n8n — just click through the same nodes already on the canvas with "Execute step," using the Clio credential you've already connected. See Step 4 below.

**Confirmed live: a newly created webhook does nothing at all until you answer a one-time verification handshake.** Right after creation (and again whenever its URL changes), Clio immediately POSTs to your webhook URL with a unique secret in an `X-Hook-Secret` header — and per Clio's own docs, "a webhook will not be enabled until this handshake is successful." Until it is, the webhook sits in status `pending` and delivers nothing, with no error anywhere to tell you why. This template handles it automatically: `Is Verification Handshake` detects that header and routes to `Confirm Webhook Handshake`, which echoes the exact same secret straight back. You don't need to do anything for this — just know it's why the webhook trigger's response mode is "responseNode" instead of an immediate default response, and why a genuine matter-update event gets its own separate instant acknowledgment (`Acknowledge Event Received`) so Clio's delivery call never has to wait on the compliance check and email send.

---

## What you need before you start

- **The Bar-Compliance Guardrail (NTC-33) must be set up first** — this template cannot send a client update without it.
- An n8n instance reachable from the internet (self-hosted with a public URL, or n8n Cloud) — Clio needs to be able to reach your webhook URL
- A Clio account with API access — matter and user-directory read scope, **plus Webhooks (Read + Write)** (reuse the OAuth2 credential from PAC-18/20/55/57/58/59/61/63/64/65 if already deployed, but add the Webhooks scope and reconnect if it wasn't already granted)
- An email account with SMTP access

No curl, Postman, or external API tools needed — the webhook is registered and kept alive entirely from inside n8n. See Step 4.

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-66-matter-milestone-updates.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-66-matter-milestone-updates.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog. Otherwise follow the OAuth2 setup in the PAC-55 or PAC-57 deploy guide.
3. Email credential: **Credentials → New → SMTP** → your provider's settings → save as `Email account (SMTP)`.

---

## Step 2 — Customize the status message map (5 min)

Open **"Look Up Status Update Message"** and edit the `STATUS_MESSAGES` object with your firm's own approved wording. Clio's matter status only has three possible values, so this map can only ever have up to three entries:

```js
const STATUS_MESSAGES = {
  // 'Open' deliberately omitted by default -- see the note above about PAC-64 overlap.
  'Pending': "Your matter is currently pending — we'll follow up as soon as there's an update.",
  'Closed': 'Your matter has been closed. Thank you for trusting us with your case.',
};
```

A status you don't list here (including `Open`, by default) is treated as internal-only and never reaches a client.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates, e.g. `https://app.clio.com` or `https://eu.app.clio.com` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | The generic contact address used when the responsible attorney can't be resolved |
| `FIRM_FROM_EMAIL` | The sender address for update emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33 |
| `FIRM_MATTER_WEBHOOK_URL` | This workflow's own Production webhook URL — you'll copy this in step 4, below, right after you activate |

---

## Step 4 — Activate and register the Clio webhook (5 min, entirely inside n8n)

1. Toggle the workflow to **Active**.
2. Open **"When Matter Status Changes"**, click the **Production URL** tab (not Test URL — the Test URL only works while you're actively in "Listen for test event" mode in the editor), and copy it.
3. Go to **Settings → Variables**, set `FIRM_MATTER_WEBHOOK_URL` to the URL you just copied, and save.
4. Confirm your Clio OAuth2 app has the **Webhooks** scope (Read + Write) granted, and that you've reconnected the credential in n8n if that scope was added after the credential was first authorized.
5. Scroll down to the second, independent branch on the canvas that starts at **"Check Webhook Subscription Daily"**. Click **"List Existing Clio Webhooks"** and hit **Execute step**, then do the same for **"Check If Webhook Needs Renewal"**, **"If Webhook Needs Renewal"**, and finally **"Register Or Renew Clio Webhook"** — confirm its output shows a new webhook `id` and `expires_at` roughly 31 days out, not an error.

   That's it — no curl, no Postman, no bearer tokens, no temporary nodes to add and delete. The webhook is now live, and this same branch runs automatically every day from then on (`Check Webhook Subscription Daily`'s schedule trigger), checking whether the subscription is still valid and quietly re-creating it a few days before it would otherwise expire. You never have to repeat this step manually again as long as the workflow stays Active.

---

## Step 5 — Test

The workflow ships with pinned sample data on the webhook trigger and both Clio-facing nodes, so you can test the logic without a real webhook delivery yet.

1. Run **"Fetch Matter Details From Clio"** → **"Check If Status Actually Changed"** and confirm `has_changed: false` the first time (nothing cached yet), then manually re-run it and confirm it's still `false` (same status, correctly suppressed).
2. Edit the pinned data's `status` field to `"Closed"`, re-run, and confirm `has_changed: true`.
3. Continue through **"Look Up Status Update Message"** and confirm `has_message: true` for `Pending`/`Closed`, and `false` if you leave `status` as `Open` (or any value not in your map).
4. Continue through **"Build Status Update Email"** and confirm the copy is personalized with the resolved attorney's contact details.
5. Continue through **"Check Compliance Before Sending"** → **"If Update Approved"** → **"Send Status Update Email"** and confirm the delivered email carries the guardrail's unsubscribe footer and renders the styled HTML.

**Testing against your real Clio account:** unpin all three Clio-facing nodes (trigger, matter fetch, users fetch), then actually change a real matter's status — **on the matter's own Dashboard tab, in the "Status" dropdown** (Open/Pending/Closed), not a Task's status. The webhook should fire within seconds — check n8n's **Executions** tab for a run whose trigger source is the webhook, not a manually-started one. If nothing shows up there at all, the problem is upstream of n8n entirely: re-confirm the webhook was actually created (step 4) with the correct Production URL, that it's subscribed to `matter_opened`/`matter_pended`/`matter_closed`, and that the workflow is Active. To check the subscription's status directly without any external tool, unpin **"List Existing Clio Webhooks"** and manually execute it — its output lists every webhook on the account, including each one's `status` and `events`.

**If the status field on your webhook shows `pending`, that's the entire problem — real events aren't being delivered because the verification handshake described above never happened (or failed).** Clio only attempts that handshake once, at creation time (or when the URL changes) — it does not keep retrying a `pending` subscription on its own. If you created your webhook *before* this workflow had `Is Verification Handshake` / `Confirm Webhook Handshake` in place, that specific subscription already failed its one shot and will stay `pending` forever. The fix is simply to create a fresh one now that the handshake is handled correctly: run **"Register Or Renew Clio Webhook"** manually to create a brand-new webhook, which triggers a brand-new handshake attempt that this workflow will now answer correctly. Then re-check **"List Existing Clio Webhooks"** and confirm the new entry shows `status: active` (or whatever non-`pending` value Clio uses once confirmed) rather than `pending`.

**Testing the self-maintenance branch specifically:** run **"Check Webhook Subscription Daily"** manually at any time (it doesn't touch matter events at all). With a freshly-created webhook, **"Check If Webhook Needs Renewal"** should report `needs_renewal: false`; temporarily lower `RENEWAL_BUFFER_DAYS` in that node's code (or just wait until you're within 5 days of the real expiry) to see it flip to `true` and confirm **"Register Or Renew Clio Webhook"** creates a fresh one.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A matter's status genuinely changes to one with a configured message (Pending or Closed by default) | The client gets a personalized update email (unless opted out) |
| A matter's status changes to Open (or any other status with no configured message) | Skipped — treated as internal-only by default |
| The very first webhook event ever received for a matter | Cached silently, no email — there's no prior status to compare against |
| The webhook payload has no usable matter id | Skipped — invalid or unrelated event |
| A Task's status is changed (not the Matter's own Status field) | Nothing happens — this template only watches the Matter's built-in status |
| The client has opted out of firm emails | Guardrail suppresses the send; already logged to the audit sheet |
| The Clio matter or user-directory fetch fails | Degrades gracefully (`neverError`) rather than crashing the run |
| The Clio webhook is missing, or within 5 days of its 31-day expiry | The daily maintenance branch (re)creates it automatically — no manual renewal needed |
| The Clio webhook already exists with plenty of life left | The daily check does nothing — no duplicate subscriptions pile up |
| Clio sends the one-time webhook verification handshake (`X-Hook-Secret` header) | Echoed straight back immediately — this is what flips the webhook from `pending` to active |
| Clio sends a real matter-updated event | Acknowledged instantly (so the delivery call doesn't hang), and processed in parallel |

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
| Deployability test | Reuses Clio and SMTP credentials from earlier templates if already deployed — under 15 minutes; webhook setup and ongoing renewal are both handled entirely inside n8n, no external tools |
| Upsell test | Clear Quick-Win Build path: SMS updates alongside email, a client portal timeline view, and richer status-specific content (next-steps checklists per milestone) |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- SMS updates alongside email for clients who respond faster to text
- A branded client portal showing the full matter timeline, not just the latest update
- Status-specific next-steps checklists instead of a single message per status
- Multi-language message templates for firms with non-English-speaking clients

[Book a call with Protomated](https://protomated.com/book) to get started.
