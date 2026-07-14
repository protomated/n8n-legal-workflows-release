# Deploy Guide: Bar-Compliance Guardrail
**NTC-33 | Ops | Free Template**
Target keyword: law firm marketing compliance automation

A sub-workflow that every client-facing template calls before sending a message. One setup, then it runs silently on every outbound SMS and email: checks your opt-out list, appends the required STOP instruction, logs the attempt. A single opted-out client who receives an unsolicited text can cost $500–$1,500 under TCPA. This catches them automatically.

This is not a standalone workflow — it is wired into other templates (NTC-17, and every subsequent client-facing template). Set it up once, then the rest is automatic.

---

## What you need before you start

- An n8n instance (self-hosted or n8n Cloud)
- A Google account with Google Sheets access
- At least one client-facing template already imported (NTC-17 is the first to wire it in)

---

## Step 1 — Create the compliance Google Sheet (3 min)

This is the single storage layer for opt-outs and the audit log. All templates share one sheet.

1. Go to [sheets.google.com](https://sheets.google.com) and click **New spreadsheet**.
2. Name it `OPS1 Compliance Log`.
3. You need two tabs. The first tab is already created — rename it to `Opt-Outs` (right-click the tab → Rename).
4. Add a second tab named `Audit Log` (click the **+** icon at the bottom).

### Set up the Opt-Outs tab headers

Click the `Opt-Outs` tab. In row 1, add these four headers exactly (one per cell, A through D):

```
phone | email | opted_out_at | source
```

Leave rows 2 onward empty — entries are added manually or via a STOP handler when clients opt out.

### Set up the Audit Log tab headers

Click the `Audit Log` tab. In row 1, add these seven headers (one per cell, A through G):

```
logged_at | template_id | matter_id | recipient | channel | message_preview | status
```

### Copy your Sheet ID

Look at the URL of your new sheet:

```
https://docs.google.com/spreadsheets/d/SHEET_ID/edit
```

Copy the long alphanumeric string between `/d/` and `/edit`. You will paste it in Step 4.

---

## Step 2 — Add a Google Sheets credential in n8n (3 min)

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **Google Sheets** and select **Google Sheets OAuth2 API**.
3. Follow the OAuth2 flow to sign in with the same Google account that owns the `OPS1 Compliance Log` sheet.
4. Save it as `Google Sheets account`.

> **First-time OAuth setup:** n8n will ask you to configure a Google Cloud OAuth2 client. Go to [console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID. Enable the Google Sheets API first (APIs & Services → Library → search "Google Sheets API" → Enable). Copy the Client ID and Client Secret into n8n. See [n8n's Google Sheets credential docs](https://docs.n8n.io/integrations/builtin/credentials/google/) for the full walkthrough.

---

## Step 3 — Import the sub-workflow (1 min)

1. Open your n8n canvas.
2. Press `Ctrl+V` (or `Cmd+V` on Mac) and paste the contents of [`ntc-33-bar-compliance-guardrail.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/ntc-33-bar-compliance-guardrail.json).
3. The workflow appears. **Do not activate it yet** — you will activate it in Step 5 after wiring your Google Sheets credential.

> **Note on the workflow name:** The n8n UI will show this as **Bar-Compliance Guardrail Layer**. Leave the name unchanged — the CI deploy script and caller templates identify it by this name.

---

## Step 4 — Set n8n Variables (1 min)

In n8n, open your project and click the **Variables** tab, then add the following two variables. Set them once — every template that calls this guardrail inherits these values automatically.

| Variable | What to enter |
|---|---|
| `OPS1_SHEET_ID` | The Google Sheet ID you copied in Step 1 (the long alphanumeric string from the URL) |
| `FIRM_NAME` | Your law firm name — appears in email opt-out footers |

No code nodes need editing. If you ever move the sheet or rename the firm, update the variable value once and all templates pick it up automatically.

---

## Step 5 — Activate the workflow and point Twilio at the webhook (2 min)

This workflow has two paths: the sub-workflow path (Path A) that other templates call, and the Twilio STOP handler (Path B) that receives inbound SMS replies. Path B requires the workflow to be active and a Twilio webhook URL configured.

1. In the guardrail workflow, click the **Activate** toggle (top-right) to turn the workflow on.
2. Click the **Receive STOP Reply** node → **Copy Production URL**. It will look like:
   ```
   https://your-n8n-instance.com/webhook/twilio-stop
   ```
3. In the [Twilio Console](https://console.twilio.com), go to **Phone Numbers → Manage → Active Numbers** and click your firm's SMS number.
4. Under **Messaging Configuration → A message comes in**, set:
   - **Webhook** → paste the Production URL from step 2
   - **HTTP POST**
5. Save.

> **One URL per Twilio number:** Twilio can only send inbound messages to one URL per number. If you already have another n8n workflow (e.g., a review reply handler) pointed at this number, you will need to route inbound messages through a single dispatcher or use a different Twilio number for STOP replies.

> **Path A works regardless of active state** — the sub-workflow path is invoked directly by calling templates and does not depend on the workflow being active. Activating it only enables the STOP webhook.

---

## Step 6 — Wire the guardrail into a calling template (5 min)

Each client-facing template needs an **Execute Sub-workflow** node placed **before** its send step (Twilio SMS or emailSend). Here is how to wire it in.

### Add the node to the calling template

1. Open the calling template (e.g., NTC-17: Review Request Engine).
2. Add an **Execute Sub-workflow** node between the config/setup node and the Twilio/email node.
3. Name it `Call Compliance Guardrail`.
4. Under **Workflow**, select **By ID** and paste the workflow ID of the Bar-Compliance Guardrail (find it in the URL when you open the guardrail workflow: `.../workflow/WORKFLOW_ID`).
5. Set **Mode** to **Run once for each item**.
6. Under **Workflow Inputs**, map the five fields the guardrail expects:

| Input field | Value expression | Notes |
|---|---|---|
| `channel` | `sms` or `email` | Hard-code the string for the template's channel |
| `recipient` | `={{ $json.client_phone }}` | Phone (E.164) for SMS; email address for email — adjust field name to match the template |
| `message` | `={{ $json.rating_sms }}` | The planned outbound message — adjust field name to match the template |
| `template_id` | `NTC-17` | Hard-code the template's ticket ID — appears in the audit log |
| `matter_id` | `={{ $json.matter_id }}` | Optional — pass if the template has a matter reference |

### Add the approved/suppressed gate after the guardrail

The guardrail returns all the caller fields you passed in (so `$json.recipient`, `$json.matter_id`, etc. are still available) plus three new fields: `approved`, `message_out`, and `suppression_reason`. The calling template must check `approved` before sending.

1. After **Call Compliance Guardrail**, add an **IF** node named `Approved to Send?`.
2. Condition: `{{ $json.approved }}` → is `true`.
3. **Yes branch** → connect to the Twilio or emailSend node. Replace the original message expression with `{{ $json.message_out }}` — this version has the disclaimer appended. Use `{{ $json.recipient }}` (or the original field name you passed) for the To/recipient field.
4. **No branch** → connect to a **NoOp** node named `Skip — contact opted out`.

### Example connection order (NTC-17, Path A)

```
Set Up Review Request → Call Compliance Guardrail → Approved to Send? → [Yes] Send Rating Request SMS
                                                                      → [No]  Skip — contact opted out
```

---

## Step 7 — Test the guardrail

### Test 1 — Clean approval (no opt-outs)

1. Open the guardrail workflow. Click **Receive Compliance Request** → **Test step**.
2. The pinned payload fires (channel: sms, recipient: +14155551234, message: NTC-17 rating request).
3. The Opt-Outs tab is empty — the guardrail should route to the **No** branch of **If Opted Out**.
4. Confirm `Result — Approved` outputs `{ approved: true, message_out: "...Reply STOP to opt out.", suppression_reason: null }`.
5. Check the Audit Log tab in your Google Sheet — one new row should appear with status `approved`.

### Test 2 — Opt-out suppression

1. In the `Opt-Outs` tab, add a row: phone = `+14155551234`, email = (leave blank), opted_out_at = today's date, source = `manual`.
2. Re-run the pinned test payload.
3. The guardrail should route to the **Yes** branch of **If Opted Out**.
4. Confirm `Result — Suppressed` outputs `{ approved: false, message_out: "", suppression_reason: "opted_out" }`.
5. Check the Audit Log — a new row should appear with status `suppressed`.
6. Remove the test row from Opt-Outs before going live.

### Test 3 — Missing field error

Temporarily remove `channel` from the pinned payload and re-run. Confirm the `Validate & Prepare` node throws a descriptive error naming the missing field. Restore the payload after.

### Test 4 — End-to-end with a calling template

1. In the calling template (e.g., NTC-17), run its test payload through to the `Call Compliance Guardrail` node.
2. Confirm the `Approved to Send?` IF node receives `approved: true` and routes to the send step.
3. Confirm the Twilio or emailSend node uses `$json.message_out` — check that the outbound message text in the execution output ends with `Reply STOP to opt out.`

### Test 5 — STOP reply captures opt-out

1. Open the guardrail workflow. Click **Receive STOP Reply** → **Test step**.
2. The pinned payload fires (`Body: "STOP"`, `From: "+14155551234"`).
3. Confirm `Append Opt-Out Entry` writes a row to the `Opt-Outs` tab with `source: "twilio-stop"`.

### Test 6 — Duplicate STOP is ignored

1. Run the same STOP pinned payload a second time.
2. Confirm `Skip — already opted out` fires and no second row appears in the `Opt-Outs` tab.

### Test 7 — UNSTOP removes the opt-out

1. In the pinned payload editor for `Receive STOP Reply`, change `Body` to `UNSTOP` and run.
2. Confirm `Delete Opt-Out Row` fires and the row added in Test 5 is removed from `Opt-Outs`.

### Test 8 — Non-STOP inbound is silently ignored

1. Change the pinned `Body` to `"Hi, when is my court date?"` and run.
2. Confirm `Skip — not a STOP or UNSTOP reply` fires and the `Opt-Outs` tab is unchanged.

---

## How the guardrail behaves

### Path A — outbound compliance check

| Scenario | What happens |
|---|---|
| Contact not on opt-out list | STOP instruction appended → message approved → audit log row added (status: approved) |
| Contact is on opt-out list | Message suppressed → no send → audit log row added (status: suppressed) |
| Opt-Outs tab is empty | Treats as no opt-outs — all contacts approved |
| Message already has STOP instruction | STOP instruction not duplicated |
| Message is an email | Unsubscribe footer appended instead of STOP instruction |
| Google Sheets write fails (logging) | Approval/suppression result still returned to caller — logging failure does not block or unblock a message |
| Caller passes a missing required field | Validation throws a descriptive error — execution halts before opt-out check |
| Caller passes unknown channel (not sms/email) | Validation throws a descriptive error |

### Path B — inbound STOP handler

| Scenario | What happens |
|---|---|
| Client texts STOP (or UNSUBSCRIBE / CANCEL / END / QUIT) | Phone appended to Opt-Outs tab with `source: "twilio-stop"` |
| Client texts STOP and is already on list | Deduplication check fires → Skip — already opted out. No duplicate row written. |
| Client texts UNSTOP (or START) and is on list | Row removed from Opt-Outs tab — contact is reachable again |
| Client texts UNSTOP and is not on list | Skip — not on opt-out list. No error. |
| Inbound message is not a STOP/UNSTOP keyword | Skip — not a STOP or UNSTOP reply. Opt-Outs tab unchanged. |
| Google Sheets API error | `onError: continueRegularOutput` on all Sheets nodes — Twilio still receives HTTP 200; failed executions visible in n8n execution log |

---

## Managing opt-outs

When a client texts STOP to your Twilio number, the STOP handler (Path B) automatically appends them to the `Opt-Outs` tab. You can also add opt-outs manually for opt-outs received via other channels (email, phone call, web form):

| Column | Format | Example |
|---|---|---|
| phone | E.164 (+1XXXXXXXXXX) | +14155551234 |
| email | email address | jane.smith@example.com |
| opted_out_at | ISO date or YYYY-MM-DD | 2026-07-01 |
| source | how they opted out | twilio-stop, manual, web-form |

> **Note:** Twilio's own STOP filter blocks messages at the carrier level regardless of this sheet. The guardrail adds a second layer that enforces opt-out status across all channels (email, future templates) and creates a local audit record in your Google Sheet.

---

## Compliance note

This sub-workflow is operational compliance only — it checks opt-out status and appends legally required opt-out language. It makes no legal judgments and does not touch substantive legal work (ABA Op. 512). TCPA requires opt-out language on every commercial text; this guardrail enforces that requirement automatically.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | TCPA violations run $500–$1,500 per unsolicited message. One suppressed opt-out pays for months of the system. Eliminates manual opt-out tracking (1–2 hr/week for busy firms). |
| Compliance test | Operational only — opt-out checking and disclaimer insertion. No legal work. |
| Searchability test | Ranks for "law firm marketing compliance automation" |
| Deployability test | Google Sheet setup + credential + two config constants + Twilio webhook URL — under 15 minutes |
| Upsell test | Quick-Win Build adds multi-channel opt-out unification, audit dashboard, Clio sync, and retention archiving |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Multi-channel unification — an opt-out via SMS suppresses email too, and vice versa
- Audit dashboard — Google Looker Studio report showing message volume, opt-out rate, and suppression history by template
- Retention policy — automatic archiving of audit log rows older than 7 years per your state's record-keeping requirements
- Clio integration — syncs opt-out status directly into the client's Clio contact record

[Book a call with Protomated](https://protomated.com/book) to get started.
