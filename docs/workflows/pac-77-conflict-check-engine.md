# Deploy Guide: Conflict-Check Engine

**Template:** PTPAC-77 — OPS4: Conflict-Check Engine
**Pillar:** Ops
**Replaces:** Manual conflict checks, practice-management add-ons

Get value in under 15 minutes. Manual conflict checks are slow and miss matches — and a missed conflict is a malpractice risk, not just an inconvenience. This workflow fuzzy-matches a new client or party name against every existing contact and matter in Clio, and hands the attorney a report to make the actual call on.

---

## Before you start: the most important thing to understand

**This workflow never clears or blocks a matter on its own.** It only ever surfaces potential name matches with a similarity score. The attorney reviews every result and decides — there is no code path anywhere in this template that auto-approves a new matter or auto-declines one.

**This is name-similarity matching, not a legal conflicts analysis.** It catches spelling variations, common company-suffix differences (Corp/Corporation/Inc/LLC), and a name appearing inside a matter's description — it cannot catch a conflict that requires actual legal judgment, like a corporate subsidiary relationship or beneficial ownership, unless a name literally matches somewhere. Treat "no matches found" as "nothing matched by name," never as "no conflict exists."

**This has two independent parts on one canvas:**
1. **Run a conflict check** — fires when staff submit a name (or names) for checking, usually before formally opening a matter. Matches against every Clio contact and matter, logs the check to an audit sheet regardless of outcome, and emails the requesting attorney a report with two decision links.
2. **Record the decision** — a webhook fired by clicking Clear or Escalate in that report email. Only ever updates the audit log.

**Every check gets logged, whether or not anything matched.** Being able to prove a conflict check was run — and what it found — matters as much as the result itself.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — the report email's audit logging goes through it
- A Clio account with API access (reuse the OAuth2 credential from other templates in this catalog if already deployed)
- A Google account with Sheets access
- An email account with SMTP access
- Some way for staff to submit a conflict check request — a Fluent Forms form is the simplest option

---

## Step 1 — Create the conflict check log (3 min)

Create a new Google Sheet. Add a tab named exactly **Conflict Checks** with these column headers in row 1:

```
check_id | names_checked | matter_description | requested_by_email | matches_found_count | checked_at | status | decision | decided_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-77-conflict-check-engine.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-77-conflict-check-engine.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog.
3. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
4. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates |
| `FIRM_NAME` | Your law firm name |
| `FIRM_EMAIL` | Fallback recipient if no requester email is submitted |
| `FIRM_FROM_EMAIL` | The sender address for report emails |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `CONFLICT_LOG_SHEET_ID` | The Google Sheet ID from Step 1 |
| `CONFLICT_DECISION_WEBHOOK_URL` | This workflow's own Production webhook URL for the decision branch — copy this in Step 4, below, right after you activate |
| `CONFLICT_MATCH_THRESHOLD` *(optional)* | Similarity sensitivity, 0–1 — defaults to 0.6. Lower catches more names at the cost of more false positives; this tool is meant to over-surface, not under-surface |

---

## Step 4 — Activate and wire the intake form (5 min)

1. Toggle the workflow to **Active**.
2. Open **"When Conflict Check Requested"**, copy its **Production URL**, and wire it to whatever staff use before opening a matter (a Fluent Forms conflict-check form, with fields `new_client_name`, `adverse_party_names`, `matter_description`, `requested_by_email`).
3. Open **"When Conflict Decision Is Made"**, copy its **Production URL**, and set it as `CONFLICT_DECISION_WEBHOOK_URL` in Settings → Variables (Step 3).

---

## Step 5 — Test

The workflow ships with pinned sample data across all Clio- and Sheets-facing nodes.

**Conflict check branch:**
1. Run **"When Conflict Check Requested"** (pinned as checking "Jon Smyth", "Acme Corporation", "Maria Jonas") → **"Parse Conflict Check Request"** and confirm all three names are parsed and `is_valid_request: true`.
2. Continue through **"Fetch All Firm Contacts From Clio"** → **"Fetch All Firm Matters From Clio"** → **"Run Fuzzy Conflict Match"** and confirm: "Acme Corporation" matches the "Acme Corp" contact, the "Acme Corp" matter client, and the "Acme Corporation supply dispute" matter description; "Maria Jonas" matches the "Maria Jones" contact. "Jon Smyth" should NOT match anything in this pinned set (it's below threshold against "John Smith" — a real test of your firm's data may catch it depending on your threshold).
3. Continue through **"Assign Conflict Check ID"** → **"Log Conflict Check For Audit"** and confirm a new row would be logged.
4. Continue through **"Build Conflict Report Email"** and confirm it lists all matches with scores, plus working Clear/Escalate links.
5. Edit the pinned webhook data's `new_client_name` to an empty string and `adverse_party_names` to an empty string, re-run, and confirm it routes to **"Respond Invalid Request"**.

**Decision branch:**
6. Run **"When Conflict Decision Is Made"** (pinned with `action=clear` for the logged check) → **"Parse Conflict Decision Request"** and confirm it parses correctly.
7. Continue through **"Read Conflict Check Log"** → **"Find Matching Conflict Check Row"** → **"If Conflict Check Found"** and confirm it finds the pinned row.
8. Continue through **"Update Conflict Check With Decision"** and confirm `status` would update to `cleared`.
9. Edit the pinned webhook data's `check_id` to something not in the log and confirm it routes to **"Respond Check Not Found"**.

**Testing against your real setup:** unpin every Clio- and Sheets-facing node, connect real credentials, and submit a real test conflict check through your actual intake form. Try a name you know exists in your Clio data under a slightly different spelling to confirm the fuzzy matching actually catches it — if it doesn't, lower `CONFLICT_MATCH_THRESHOLD`.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A name is submitted with no matches found | Still logged to the audit sheet; report email says so explicitly |
| A name matches an existing contact or matter above the threshold | Surfaced in the report with a similarity score and source reference |
| The attorney clicks "Clear" | Audit log status becomes `cleared` — nothing else happens automatically |
| The attorney clicks "Escalate" | Audit log status becomes `escalated` — nothing else happens automatically |
| A decision link is malformed or already resolved | A plain error page is shown; the log is unchanged |

---

## Compliance note

This template performs name-matching and record-keeping only — surfacing potential matches by similarity score, never making or implying a legal conflicts determination (ABA Op. 512). The attorney makes every actual decision via explicit click-through; there is no automatic clearing, blocking, or matter-opening action anywhere in this workflow. The Bar-Compliance Guardrail is used for audit logging on the report email, since it goes to firm staff, not a client.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | A missed conflict is a malpractice risk and potential disqualification from a matter — this catches what manual review misses |
| Compliance test | Surfaces name-similarity matches only; the attorney makes every actual conflict determination |
| Searchability test | Ranks for "law firm conflict check automation" |
| Deployability test | Reuses Clio credentials from other templates if already deployed — under 15 minutes |
| Upsell test | Clear Done-for-You path: productized as a fixed-scope build with firm-specific matching rules and PM-system integration beyond Clio |

---

## Want the advanced version?

Protomated builds the done-for-you **Done-for-You** implementation that adds:
- Matching tuned to your firm's actual naming conventions and practice areas
- Integration with practice management systems beyond Clio
- Structured conflict-waiver tracking alongside the clear/escalate decision
- Firm-wide conflict-check reporting and audit dashboards

[Book a call with Protomated](https://protomated.com/book) to get started.
