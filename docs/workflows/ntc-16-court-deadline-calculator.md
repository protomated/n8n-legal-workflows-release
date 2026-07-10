# Deploy Guide: Court Deadline Calculator
**NTC-16 | Keep | Free Template**
Target keyword: court deadline calculator

Get value in under 15 minutes. When a matter reaches a key date — a complaint
filing, a scheduling order, a service date — this workflow calculates every
downstream court deadline your attorney has defined, drops each one into your
calendar, and creates a high-priority Clio task per deadline, all automatically.

---

## What you need before you start

- An n8n instance (self-hosted or n8n Cloud)
- A Google Calendar **or** Microsoft Outlook / Microsoft 365 account (one, not both)
- A Clio account with API access
- A Google account with access to the **OPS1 Compliance Log** Google Sheet — the same sheet used by the Bar-Compliance Guardrail (NTC-33). If you have not set that sheet up yet, follow Step 1 of the NTC-33 deploy guide to create it, then return here.
- The deadline rule set your attorney has entered for your practice area
  and jurisdiction — **do not proceed without this**

> **Important — the firm owns the rule set:**
> This workflow applies rules mechanically. It never suggests, interprets, or
> confirms legal deadlines. Your attorney must define every offset and deadline
> type before you configure the workflow. Get sign-off from the attorney who will
> own the rule set before going live.

---

## Step 1 — Import the workflow (1 min)

1. Open your n8n canvas.
2. Press `Ctrl+V` (or `Cmd+V` on Mac) and paste the contents of
   [`ntc-16-court-deadline-calculator.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/ntc-16-court-deadline-calculator.json).
3. The workflow appears. Do not activate it yet.

---

## Step 2 — Add credentials (6 min)

Wire credentials only for the services your firm actually uses.

### Google Sheets credential (for audit log)

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **Google Sheets OAuth2 API** and select it.
3. Click **Connect with Google** and authorise with the Google account that owns
   the OPS1 Compliance Log sheet.
4. Save it as `Google Sheets account`.

> If you already created this credential for NTC-33, skip this step — it is
> shared automatically.

### Google Calendar credential (if using Google Workspace)

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **Google Calendar OAuth2 API** and select it.
3. Click **Connect with Google** and authorise with the attorney's Google account.
4. Save it as `Google Calendar account`.

### Outlook credential (if using Microsoft 365)

1. In n8n, open your project and click the **Credentials** tab, then **New credential**.
2. Search for **Microsoft Outlook OAuth2 API** and select it.
3. Click **Connect with Microsoft** and authorise with the attorney's Microsoft
   365 account.
4. Save it as `Outlook account`.

### Clio API credential

Clio uses token-based authentication. You need the API token from your Clio
account settings.

1. In Clio go to **Settings → API Keys → New API Key**.
2. Give the key a name (e.g. `n8n deadline automation`) and copy the token.
3. In n8n, open your project and click the **Credentials** tab, then **New credential**.
4. Search for **Header Auth** and select it.
5. Set **Name** to `Authorization` and **Value** to `Bearer YOUR_TOKEN`
   (replace `YOUR_TOKEN` with the token from Clio — keep the word `Bearer`
   followed by a space before the token).
6. Save it as `Clio API token`.

> **Note:** Clio's newer developer portal (`developer.clio.com`) issues OAuth2
> tokens. For the free template, a long-lived API token from Clio's Settings
> page is the simplest option. For automated production use, ask Protomated
> about the done-for-you OAuth2 setup.

---

## Step 3 — Set n8n Variables and configure your rule set (5 min)

### Firm settings — n8n Variables

In n8n, open your project and click the **Variables** tab, then add the following variables. Set them once — every workflow that uses these values picks them up automatically.

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | Your intake email address |
| `FIRM_TIMEZONE` | Your state's timezone in IANA format (e.g. `America/New_York`, `America/Chicago`, `America/Los_Angeles`) |
| `OPS1_SHEET_ID` | The ID from your OPS1 Compliance Log URL: `https://docs.google.com/spreadsheets/d/SHEET_ID/edit` — the same sheet used by the Bar-Compliance Guardrail |
| `FIRM_OUTLOOK_TIMEZONE` | **Outlook only** — your state's timezone in Windows format: `Eastern Standard Time`, `Central Standard Time`, `Mountain Standard Time`, or `Pacific Standard Time`. Leave blank if using Google Calendar. |

### Rule set — edit directly in Calculate Deadlines

Replace the three placeholder entries in `DEFAULT_RULES` with the actual rules
your attorney has entered:

```js
const DEFAULT_RULES = [
  {
    name:        'Example Deadline A — replace this',
    offset_days: 30,
    offset_type: 'business',
    description: 'Placeholder — replace with your firm rule',
  },
  // … replace all entries …
];
```

Each rule needs four fields:

| Field | What to enter |
|---|---|
| `name` | The deadline's display name — appears in calendar events and Clio tasks |
| `offset_days` | Number of days from trigger date (positive = after, negative = before) |
| `offset_type` | `'business'` to skip weekends + US federal holidays; `'calendar'` to count every day |
| `description` | A longer note for the calendar event body and Clio task description |

**Example** — a firm that tracks a 90-calendar-day service window and a
60-business-day discovery close might write:

```js
{
  name:        'Serve Defendant',
  offset_days: 90,
  offset_type: 'calendar',
  description: 'Firm rule: 90 calendar days from complaint filing — confirmed by attorney J. Smith on 2026-06-01',
},
{
  name:        'Close of Fact Discovery',
  offset_days: 60,
  offset_type: 'business',
  description: 'Firm rule: 60 business days from trigger date — confirmed by attorney J. Smith on 2026-06-01',
},
```

> **Record keeping:** Note which attorney confirmed each rule and on what date.
> Keep that record outside n8n (a document, a Clio note, your firm's procedure
> manual). The workflow description field is a good place to echo this for
> visibility, as in the example above.

---

## Step 4 — Verify n8n Variables are live (1 min)

No code nodes need editing — firm settings are read from the project **Variables** tab (set in Step 3). To verify, open the **Calculate Deadlines** node and confirm the code reads `$vars.FIRM_NAME` (not a hardcoded string). If you see a hardcoded string, you are running an older version of the workflow — re-import from the latest JSON.

If you are using **Microsoft Outlook**, confirm `FIRM_OUTLOOK_TIMEZONE` is set in the project **Variables** tab. The **Create Outlook Calendar Event** node reads it automatically — no node edits required. Valid values: `Eastern Standard Time`, `Central Standard Time`, `Mountain Standard Time`, `Pacific Standard Time`.

Google Calendar does not require a timezone variable — all-day events use date-only format and are timezone-independent.

---

## Step 5 — Test the workflow (5 min)

### Golden path test (pinned data)

The workflow ships with a pinned three-rule payload on the **Deadline Webhook**
node.

1. Click **Deadline Webhook**, then **Test step** — n8n runs the pinned data
   through the full chain.
2. Step through manually:
   - **Read Deadline Request** → `is_valid_request: true`
   - **Valid request?** → Yes branch
   - **Calculate Deadlines** → three output items, each with a `deadline_date`
     in `YYYY-MM-DD` format. Verify the dates look correct for a 2026-07-01
     trigger date:
     - 30 business days forward from 2026-07-01 skips July 4 (observed
       Friday July 3) → lands on 2026-08-13
     - 14 calendar days forward → 2026-07-15
   - **Calendar platform?** → Google branch (true)
   - **Create Google Calendar Event** → runs three times, one per deadline;
     each response should include an event `id`
   - **Update Matter in Clio** → runs three times; each response should include
     a task `id`
   - **Log to Audit** → runs three times; check the Audit Log tab in your Google
     Sheet for three new rows with `status: calculated`

### Invalid request guard test

Send a POST to the webhook URL with a missing trigger date:

```json
{ "matter_id": "M-2026-0001" }
```

Execution should route to **Skip — invalid request** with
`validation_error: "Invalid or missing trigger_date — use YYYY-MM-DD"`.

### Outlook path test

Send a payload with `"calendar_platform": "outlook"`:

```json
{
  "matter_id": "M-2026-0043",
  "clio_matter_id": 11223,
  "matter_title": "Williams v. City",
  "trigger_date": "2026-07-15",
  "attorney_email": "attorney@smithlaw.com",
  "calendar_platform": "outlook",
  "rules": [
    { "name": "Test Deadline", "offset_days": 10, "offset_type": "business", "description": "10 business days" }
  ]
}
```

Execution should route to **Create Outlook Calendar Event** (false branch) and
skip the Google Calendar node.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Rules omitted from payload | DEFAULT_RULES from Calculate Deadlines are used |
| trigger_date falls on a weekend | Business-day count starts from that date; the first counted business day is the following Monday |
| A deadline lands on a US federal holiday (business-day rule) | The count continues past the holiday; the result is the next business day |
| Calendar step fails (wrong credential) | Workflow continues — Clio tasks are still created |
| Clio step fails (wrong token or numeric ID mismatch) | Execution continues to Log to Audit; calendar events were already created |
| Log to Audit fails (credentials not wired, sheet not found) | Workflow completes — calendar events and Clio tasks are not affected; check execution logs for the error |
| clio_matter_id not provided | Clio task is created without a matter link; link it manually inside Clio |
| calendar_platform not 'google' | Routes to Outlook branch — any value other than 'google' uses Outlook |

---

## Known limitations

- **No calendar deduplication.** Triggering the workflow twice for the same
  matter creates duplicate calendar events and Clio tasks. The done-for-you
  build adds idempotency checking.
- **Clio numeric ID required.** Clio's API needs the internal numeric matter ID,
  not the display number. Pass `clio_matter_id` as a number in your payload.
- **US federal holidays only.** State and local court holidays are not included
  — your attorney should review any deadline that falls close to a known state
  holiday and adjust manually.

---

## Compliance note

This template is operational only — it applies firm-entered rules mechanically
and never touches legal work (ABA Op. 512). The firm owns and confirms every
rule definition. Every deadline calculated is logged to the Audit Log tab of
your OPS1 Compliance Log Google Sheet — the same sheet used by the
Bar-Compliance Guardrail.

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:

- Direct Clio integration — trigger calculations automatically when a matter
  reaches a defined stage; no manual webhook needed
- Idempotency checks — prevents duplicate deadlines on re-trigger
- State and local court holiday calendars for your jurisdiction
- Multi-attorney routing — correct calendar per matter based on assigned attorney
- Deadline conflict detection — flags when two deadlines fall on the same day
- Full compliance layer pre-wired with audit logging and rule-change history

[Book a call with Protomated](https://protomated.com/book) to get started.
