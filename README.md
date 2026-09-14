# n8n Legal Workflows — Free Templates

Free, ready-to-import n8n automation templates for solo and small law firms, built by [Protomated](https://protomated.com).

## How to use a template

1. **Download** the workflow you want. Two ways:
   - Click a template's `.json` link in its deploy guide below (`docs/workflows/`).
   - Or use the direct download link from the [latest release](../../releases/latest) — click a `.json` asset and it downloads immediately, no extra steps.
2. **Import it** into your own n8n instance: open your n8n canvas and paste (`Ctrl+V` / `Cmd+V`) the downloaded file's contents.
3. **Follow the deploy guide** for that template (same filename, under `docs/workflows/`) — each one walks through credentials, required settings, and how to test it before going live.

Every template ships deactivated (`active: false`) with sample test data already in place, so you can safely test it end-to-end before connecting it to real client data.

## What's in this repo

- `workflows/` — the importable `.json` file for each template.
- `docs/workflows/` — a step-by-step deploy guide per template.

This repo is the public distribution point for templates built in Protomated's internal workflow catalog — every file here is automatically kept in sync with what's actually shipping.

## Questions or want something built for your firm?

These free templates handle one specific task each. If you want a fuller setup tailored to your firm — additional practice management systems, deeper integrations, or ongoing support — [book a call with Protomated](https://protomated.com/book).
