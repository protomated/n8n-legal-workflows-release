# Deploy Guide: Duplicate-Contact & Data-Hygiene Watchdog

**Template:** PAC-55 — OPS10: Duplicate-Contact & Data-Hygiene Watchdog
**Pillar:** Ops
**Replaces:** Manual CRM cleanup, Insycle/Cloudingo-style dedupe tools

Get value in under 15 minutes. On a schedule, this scans every contact in Clio, scores every pair for duplicate likelihood, and emails your paralegal a digest of the ones worth a look — each with a direct link to both records and a one-click Dismiss if it's a false positive. No client ever gets contacted twice, and no invoice ever goes to the wrong version of a contact, because nobody noticed the same person existed as two records.

---

## Before you start: what this template does not do

**Clio has no contact-merge feature — not in the app UI, not via API.** This template only finds and flags likely duplicates; combining two contacts always happens as a manual step in Clio, following [Clio's own documented process](https://support.clio.com/hc/en-us/articles/115002325034-Is-There-a-Way-to-Merge-Contacts-): open both contacts, copy anything missing from the older record into the one you want to keep, then delete the duplicate. If you want that step automated too, that's a fair ask for the Build upgrade — see the bridge at the bottom of this guide.

---

## What you need before you start

- An n8n instance (self-hosted or n8n Cloud)
- A Clio account with API access (a free trial works fine for testing)
- An email account with SMTP access (Gmail, Outlook 365, Zoho Mail, or similar)

---

## Step 1 — Import the workflow (1 min)

1. Open your n8n canvas.
2. Press `Ctrl+V` (or `Cmd+V` on Mac) and paste the contents of [`pac-55-duplicate-contact-data-hygiene-watchdog.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-55-duplicate-contact-data-hygiene-watchdog.json).
3. Do not activate it yet.

---

## Step 2 — Add credentials (5 min)

### Clio credential
1. In Clio: **Settings → Developer Applications** → create an application (or reuse one you already have for another template in this catalog) → copy the Bearer token.
2. In n8n: **Credentials → New credential → Header Auth**.
3. Header Name: `Authorization`. Header Value: `Bearer YOUR_TOKEN`.
4. Save it as `Clio API (Bearer token)` — reuse the same credential if you've already deployed PAC-18 or PAC-20.

### Email credential
1. **Credentials → New credential → SMTP**.
2. Enter your provider's SMTP settings (Gmail: use an App Password; Microsoft 365: enable SMTP AUTH in the admin centre).
3. Save it as `Email account (SMTP)`.

---

## Step 3 — Set n8n Variables (3 min)

In n8n, open your project and click the **Variables** tab, then add:

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Your Clio region's base URL, e.g. `https://app.clio.com` (US) or `https://eu.app.clio.com` (EU) — reuse the same value if PAC-18 or PAC-20 is already deployed |
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | The address the digest email is sent from |
| `FIRM_PARALEGAL_EMAIL` | Where the duplicate-pair digest is sent. Optional — falls back to `FIRM_EMAIL` if unset |
| `DUPLICATE_CONFIDENCE_THRESHOLD` | Optional. Defaults to `70` if unset. Lower it to catch more possible duplicates (more false positives); raise it to only see near-certain matches |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow — find it in the n8n URL when you open NTC-33: `.../workflow/WORKFLOW_ID` |

`DUPLICATE_DISMISS_WEBHOOK_URL` is set in Step 4, after activation — you need the Production URL first.

---

## Step 4 — Activate and wire the Dismiss link (2 min)

1. Toggle the workflow to **Active**.
2. Open the **When Dismiss Link Clicked** node and copy its **Production URL** (ends in `/webhook/dismiss-duplicate`).
3. Back in **Variables**, add `DUPLICATE_DISMISS_WEBHOOK_URL` with that URL as the value.

This is the one variable that can't be set until after activation — the digest email builds its Dismiss links from it, so pairs flagged before this is set will have broken links until you fix it.

---

## Step 5 — Test

The workflow ships with pinned test data on all three trigger-adjacent nodes, including two deliberately-planted duplicate pairs so you can verify the matching logic without live Clio data.

1. Click **Fetch All Clio Contacts** → **Test step** using the pinned data (five contacts: two near-duplicate people, two near-duplicate companies, one contact with no match).
2. Click **Find And Score Likely Duplicate Pairs** → confirm it finds exactly 2 pairs, both scoring 70+ (the "Maria Alvarez" contact should not appear in any pair).
3. Run the rest of the chain through to **Notify Paralegal Of Possible Duplicates** → confirm you receive the digest email with both pairs, correct Clio contact links, and working Dismiss links.
4. Click one of the Dismiss links from the email → confirm you see the "Marked as not a duplicate" confirmation page.
5. Re-run the scan from **Fetch All Clio Contacts** with the same pinned data → confirm the dismissed pair no longer appears in the digest (the other pair does, since only one was dismissed).

Once your real Clio trial is set up, remove the pin from **Fetch All Clio Contacts** (or just run the full workflow manually) to test against your actual contacts.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Scheduled scan finds no pairs above the noise floor | Skipped — **Skip — No Duplicate Pairs Found**. No email. |
| Scan finds pairs, but all are below the confidence threshold or already flagged/dismissed | Skipped — **Skip — No New Pairs To Report**. No email. |
| Scan finds new pairs above threshold | Digest email sent to the paralegal; scan logged to the OPS1 audit sheet |
| Paralegal clicks Dismiss on a pair | Pair is suppressed permanently (never re-flagged); confirmation page shown |
| Same pair appears in a later scan before being dismissed or merged | Not re-reported — each pair is flagged once until dismissed, merged (deleted from Clio, which removes it from future scans automatically), or 90 days pass with no action |
| A "Company" contact and a "Person" contact happen to share a name | Never compared — pairs are only scored within the same contact type |

---

## Compliance note

This template is operational only — data hygiene, no judgment on matter substance, no legal work (ABA Op. 512). It never merges or deletes anything automatically; every pair requires a human to review and act, either by dismissing a false positive or merging manually in Clio. Every scan is logged to the Bar-Compliance Guardrail's audit sheet.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces manual CRM cleanup and dedupe tools like Insycle/Cloudingo. Duplicate contacts silently break billing, dunning, and reporting until a client complains — this catches them before that happens. |
| Compliance test | Data hygiene only — no legal work, no automated merging of records. |
| Searchability test | Ranks for "law firm CRM duplicate contact cleanup" |
| Deployability test | One credential, six config values, under 15 minutes |
| Upsell test | Quick-Win Build adds automated field-level merge suggestions and a Slack alert channel |

---

## Want the advanced version?

Protomated builds the done-for-you **Quick-Win Build** that adds:
- Guided merge assistant — shows exactly which fields differ and pre-fills the merge so the paralegal clicks once instead of copying data by hand in Clio
- Slack/Teams alerts alongside email
- Cross-system dedupe — checks Clio contacts against your CRM (Salesmate) too, catching duplicates that span both systems
- Weekly data-hygiene score — trending count of flagged, dismissed, and resolved pairs over time

[Book a call with Protomated](https://protomated.com/book) to get started.
