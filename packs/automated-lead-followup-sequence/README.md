# Automated Lead Follow-Up Sequence

A free n8n workflow that auto-follows-up with inquiry-form leads so nobody has to remember to do it by hand.

Leads go quiet after the first inquiry because nobody has time to follow up three separate times over a week. This workflow acknowledges every lead instantly, follows up on Day 1, 3, and 7 automatically, stops the moment they reply, and alerts the attorney directly if a lead goes seven days with no response at all.

Works well for a handful of leads a week (roughly 5). At meaningfully higher volume (20+/week), a full CRM pipeline with proper lead scoring and routing is the better fit.

## What's included

1. **`automated-lead-followup-sequence.json`** — the n8n workflow. Import steps and full setup are in `deploy-guides/automated-lead-followup-sequence.md`.
2. **`sample-jotform-template.json`** — the exact field list (names, types, and a sample practice-area dropdown) your intake form needs to send. It's a field specification you recreate in your form tool's builder, not a one-click import file, since most form tools don't support importing a form from a plain JSON file. Works with JotForm, Typeform, Fluent Forms, or anything that can POST to a webhook.

## Getting started

1. Open your n8n canvas and paste (`Ctrl+V` / `Cmd+V`) the contents of `automated-lead-followup-sequence.json`.
2. Follow `deploy-guides/automated-lead-followup-sequence.md` for credentials, required settings, and how to test it safely before going live.
3. Recreate the fields in `sample-jotform-template.json` in your form tool of choice, then point that form's webhook/integration settings at the workflow's intake URL.

The workflow ships turned off, with sample test data already loaded, so you can test it end-to-end before connecting it to a real lead.

## Compliance

This sends templated marketing/operational copy only — never legal advice or a judgment about anyone's case (per ABA Opinion 512). Every email is compliance-checked and logged through the Bar-Compliance Guardrail.

## Want more?

Want deeper CRM-grade lead scoring, multi-channel follow-up (text plus email), or a full pipeline once you're past a handful of leads a week? [Book a call with Protomated](https://protomated.com/book).
