# Deploy Guide: Trust/IOLTA Balance Alerts

**Template:** PTPAC-93 — B4: Trust/IOLTA Balance Alerts
**Pillar:** Keep
**Replaces:** Practice management accounting add-ons

Get value in under 15 minutes. A trust account going negative is a serious compliance event — this workflow watches every open matter's trust balance in Clio and alerts before it goes negative or needs replenishment, before it becomes a bar complaint.

---

## Before you start: the most important thing to understand

**This only ever reports and alerts on balances already recorded in Clio.** It never moves money, adjusts a ledger, transfers funds, or makes any accounting decision — it's operational balance monitoring only (ABA Op. 512).

**Confirm the `trust_balance` field for your Clio setup before deploying.** This template requests `trust_balance` directly on Clio's Matters endpoint. Confirm live that this field name is valid for your Clio API version and plan — if Clio rejects it, the fallback is aggregating `/trust_line_items.json` per matter yourself (summing debits and credits), which isn't built here since the exact ledger structure varies by how each firm's trust accounts are configured.

**This inherits the Bar-Compliance Guardrail (OPS1), even though every recipient is firm staff, not a client.** The ticket for this template specifically calls for that — the opt-out/disclaimer checks are effectively inert for an internal accounting recipient, but the audit log entry is valuable for a compliance-grade template like this one.

**This has two independent parts on one canvas:**
1. **Daily balance check** — tiers every open matter (negative / low / healthy) and alerts on anything that isn't healthy, with a cooldown so an unresolved balance doesn't re-alert every single day. A worsening tier (low → negative) always alerts immediately regardless of cooldown.
2. **Weekly trust health rollup** — reports every open matter's balance, not just the alert-worthy ones, for accounting's own periodic review.

**A negative balance always immediately escalates to the managing partner**, separately from — and regardless of — the accounting alert's cooldown logic. This is the one alert in this template that should never wait.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed
- A Clio account with API access (reuse the OAuth2 credential from other templates in this catalog if already deployed)
- An email account with SMTP access

---

## Step 1 — Import the workflow and add credentials (3 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-93-trust-iolta-balance-alerts.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-93-trust-iolta-balance-alerts.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog.
3. Email credential: **Credentials → New → SMTP**.

---

## Step 2 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates |
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | Sender address for all alert emails |
| `FIRM_TRUST_ACCOUNTING_EMAIL` | Who receives daily balance alerts and the weekly rollup |
| `FIRM_PARTNER_EMAIL` | Who receives immediate negative-balance escalations |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `LOW_BALANCE_THRESHOLD` *(optional)* | Balance below which a matter is flagged "low" — defaults to 500 |
| `ALERT_COOLDOWN_DAYS` *(optional)* | How many days before an unresolved balance re-alerts — defaults to 7 |

---

## Step 3 — Activate (2 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expressions on **"Check Trust Balances Daily"** (default 7am) and **"Check Weekly Trust Health Rollup"** (default Monday 8am) if different times suit.

---

## Step 4 — Test

The workflow ships with three pinned matters: one negative (-$150), one low ($250, below the $500 default threshold), and one healthy ($3,500).

1. Run **"Fetch Open Matters With Trust Balance"** → **"Split Matter Records"** and confirm all 3 matters come through as separate items.
2. Continue through **"Compute Trust Balance Alert Tier"** and confirm the negative matter is tagged `alert_tier: "negative"`, the low matter `alert_tier: "low"`, and the healthy matter is excluded entirely from the output.
3. Re-run the same input a second time (without clearing static data) and confirm both alert-worthy matters now show `should_alert: false` — the cooldown suppresses a same-day repeat.
4. Edit the low matter's pinned `trust_balance` to `-50` (worsening it to negative) and re-run — confirm it shows `should_alert: true` immediately, ignoring the cooldown.
5. Continue through **"Filter To Alert-Worthy Matters"** → **"Build Trust Balance Alert Email"** → **"If Balance Is Negative"** and confirm the negative matter routes to **"Build Partner Escalation Email"** while the low matter routes to **"Skip Partner Escalation"**.
6. Continue through both compliance-check chains and confirm the alert email and partner escalation email both read correctly.

**Testing against your real setup:** unpin the Clio nodes, connect real credentials, and confirm `trust_balance` actually returns real data for your matters — if Clio rejects the field, see the note above about falling back to `/trust_line_items.json`.

**Weekly trust health rollup branch:**
7. Run **"Check Weekly Trust Health Rollup"** → **"Fetch All Matters For Trust Rollup"** → **"Compute Weekly Trust Health Summary"** and confirm the healthy/low/negative counts and total negative exposure match the 3 pinned matters.
8. Continue through **"Build Weekly Trust Health Email"** and confirm it reads correctly.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A matter's trust balance goes negative | Accounting gets an alert AND the managing partner gets an immediate escalation, both through the compliance guardrail |
| A matter's trust balance is low but still positive | Accounting gets an alert; no partner escalation |
| The same matter stays at the same tier day after day | Only alerts once, then waits out the cooldown — not every single day |
| A low balance worsens to negative | Alerts immediately, ignoring any active cooldown |
| A matter recovers to healthy | Its alert history clears, so a future dip alerts fresh rather than staying silenced |
| A matter has no trust balance data | Skipped silently, not treated as an error |

---

## Compliance note

This template performs balance reporting and alerting only — relaying a matter's own trust balance as recorded in Clio, never moving funds or making an accounting decision (ABA Op. 512). The Bar-Compliance Guardrail is used on every alert per this template's ticket, even though recipients are firm staff rather than clients.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces a practice management accounting add-on for trust monitoring, and catches a compliance event (negative trust balance) before it becomes a bar complaint |
| Compliance test | Balance reporting and alerting only; no funds moved, no accounting decisions made |
| Searchability test | Ranks for "iolta trust account balance alerts" |
| Deployability test | Reuses Clio credentials from other templates if already deployed — under 15 minutes once `trust_balance` is confirmed |
| Upsell test | Clear Done-for-You path: productized as a fixed-scope build with firm-specific replenishment rules and PM systems beyond Clio |
