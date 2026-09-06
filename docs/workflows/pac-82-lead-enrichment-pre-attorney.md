# Deploy Guide: Lead Enrichment Pre-Attorney

**Template:** PTPAC-82 — C6: Lead Enrichment Pre-Attorney
**Pillar:** Convert
**Replaces:** Clay ($149/mo and up), ZoomInfo

Get value in under 15 minutes. Attorneys spend the first minutes of every lead manually researching who it is. This workflow does that research automatically — company signal, industry, and a rough matter-value tier — before the attorney ever sees the raw lead.

---

## Before you start: the most important thing to understand

**The matter-value tier is a rough sorting heuristic, not a valuation.** It's a keyword scan against terms you configure (e.g. "wrongful death," "class action") plus a check for dollar amounts mentioned in the inquiry — never a legal or financial assessment of what a matter is actually worth. Treat "High" as "worth a closer look first," not "this is a big case" (ABA Op. 512 — operational triage only).

**This needs an enrichment provider you bring yourself.** There's no free, reliable company-lookup API built into this template — it's built against a generic response shape (company name, domain, industry, employee count) that you point at whatever provider you sign up for (Clearbit, Apollo, Hunter, or similar). The exact field names in a real provider's response will differ from this template's assumptions — check them on first live test.

**A non-consumer email domain is a signal, not proof.** Someone emailing from `@theircompany.com` is *probably* inquiring in a business context, but this can't confirm it — it's a free, useful heuristic for deciding whether an enrichment lookup is worth attempting, nothing more.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — the briefing's audit logging goes through it
- A Google account with Sheets access
- An email account with SMTP access
- An enrichment provider account and API key (Clearbit, Apollo, Hunter, or similar)
- A lead source (intake form or similar) that can POST to a webhook

---

## Step 1 — Create the enrichment log (3 min)

Create a new Google Sheet. Add a tab named exactly **Enrichment Log** with these column headers in row 1:

```
name | email | company_name | company_industry | matter_value_tier | received_at
```

Copy the Sheet's ID from its URL — the long string between `/d/` and `/edit`.

---

## Step 2 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-82-lead-enrichment-pre-attorney.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-82-lead-enrichment-pre-attorney.json). Do not activate it yet.
2. Enrichment provider credential: **Credentials → New → Header Auth** → set the header name/value your provider's API expects (check their docs — commonly `Authorization: Bearer <key>`).
3. Google Sheets credential: **Credentials → New → Google Sheets OAuth2**.
4. Email credential: **Credentials → New → SMTP**.

---

## Step 3 — Set n8n Variables (5 min)

| Variable | What to enter |
|---|---|
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | Sender address for the briefing email |
| `FIRM_EMAIL` | Fallback recipient if `INTAKE_ATTORNEY_EMAIL` isn't set |
| `INTAKE_ATTORNEY_EMAIL` | Who receives the enrichment briefing |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |
| `ENRICHMENT_LOG_SHEET_ID` | The Google Sheet ID from Step 1 |
| `ENRICHMENT_API_URL` | Your enrichment provider's actual API endpoint |
| `CONSUMER_EMAIL_DOMAINS` *(optional)* | Comma-separated list of personal email domains to exclude from lookups — defaults to `gmail.com,yahoo.com,hotmail.com,outlook.com,aol.com,icloud.com,live.com` |
| `HIGH_VALUE_KEYWORDS` *(optional)* | Comma-separated value signals — defaults to `wrongful death,class action,commercial litigation,catastrophic injury,multi-million` |

---

## Step 4 — Update the enrichment response parsing for your real provider (5 min)

Open **"Parse Company Enrichment Response"** and check the field names against your actual provider's documentation — this template assumes a generic `{name, domain, industry, employeeCount}` shape, which real providers structure differently. Adjust the `d.name || d.company || ...` fallback chains to match.

---

## Step 5 — Activate and wire your lead source (3 min)

1. Toggle the workflow to **Active**.
2. Open **"When New Lead Received"**, copy its **Production URL**, and wire it to your intake form or lead source's webhook/notification settings.

---

## Step 6 — Test

The workflow ships with a pinned lead from a business email domain (`acmecorp.com`) with a case description mentioning both a value keyword ("commercial litigation") and a dollar amount, plus a pinned enrichment API response.

1. Run **"Parse New Lead"** → **"Extract Company Domain From Email"** and confirm `is_business_domain: true` for the pinned `@acmecorp.com` address.
2. Continue through **"Fetch Company Enrichment Data"** → **"Parse Company Enrichment Response"** and confirm `company_name: "Acme Corp", company_data_found: true`.
3. Continue through **"Compute Matter Value Signal"** and confirm `matter_value_tier: "High"`.
4. Continue through to **"Build Lead Enrichment Briefing"** and confirm the message includes the company details and value tier.
5. Edit the pinned webhook data's `email` to a Gmail address and re-run — confirm it routes to **"Note No Company Signal Available"** instead of attempting enrichment.
6. Edit the pinned data's `case_description` to remove both the keyword and dollar amount, re-run — confirm the tier drops to `"Medium"` (company still found) or `"Low"` (no company signal).

**Testing against your real setup:** unpin the webhook and enrichment nodes, connect your real provider credential, and submit a real test lead with a real company email domain to confirm the actual API response shape matches what Step 4 expects.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| Lead's email is on a company domain, enrichment finds a match | Briefing includes company name, industry, and employee count |
| Lead's email is on a company domain, enrichment finds nothing | Briefing notes the domain but says no match was found |
| Lead's email is a personal/consumer domain | Briefing notes there's no company signal to enrich |
| Case description mentions a value keyword or dollar amount | Value tier is "High" regardless of company signal |
| No value keyword, but a company was found | Value tier is "Medium" |
| No value keyword and no company signal | Value tier is "Low" |

---

## Compliance note

This template performs data lookup and keyword-based sorting only — surfacing publicly enrichable company information and scanning firm-configured keywords, never a legal or financial valuation of a matter (ABA Op. 512). The Bar-Compliance Guardrail is used for internal audit logging only, since the briefing goes to the firm's own attorney, not a client or lead.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces a Clay or ZoomInfo subscription for this specific use case, and recovers the minutes an attorney spends manually researching every new lead |
| Compliance test | Data lookup and keyword sorting only, no valuation or legal judgment |
| Searchability test | Ranks for "law firm lead enrichment" |
| Deployability test | Needs one enrichment provider key plus standard Sheets/SMTP credentials — under 15 minutes once you have a provider account |
| Upsell test | Clear Done-for-You path: productized as a fixed-scope build with a pre-vetted enrichment provider and firm-specific value scoring |

---

## Want the advanced version?

Protomated builds the done-for-you **Done-for-You** implementation that adds:
- A pre-vetted, pre-configured enrichment provider (no account setup on your end)
- Firm-specific value scoring tuned to your actual practice areas and past matter outcomes
- CRM integration so the briefing lands directly on the lead record, not just an email
- LinkedIn and public-records signals beyond company domain lookup

[Book a call with Protomated](https://protomated.com/book) to get started.
