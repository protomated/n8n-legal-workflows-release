# New Client Kickoff Kit

A free n8n workflow that handles the busywork the moment a client is marked signed.

The first week after a client signs sets the tone for the whole relationship, but the setup work — a Drive matter folder, a kickoff call on the calendar, a document checklist, follow-up tasks logged — is easy to let slip when everyone's focused on the actual matter. This workflow does all of it automatically: creates the Drive folder structure, sends a branded welcome email with a practice-area-specific document checklist and a "schedule your kickoff call" link, logs follow-up tasks, and reports a weekly onboarding summary.

Runs entirely on its own. If you've also deployed the Client Onboarding Sequence template, see the deploy guide for how to avoid sending a client two different document checklists.

## What's included

1. **`new-client-kickoff-kit.json`** — the n8n workflow. Import steps and full setup are in `deploy-guides/new-client-kickoff-kit.md`.
2. **`sample-drive-folder-structure.md`** — exactly what folder structure gets created automatically, and how to customize the subfolder list without touching any code.

## Getting started

1. Open your n8n canvas and paste (`Ctrl+V` / `Cmd+V`) the contents of `new-client-kickoff-kit.json`.
2. Follow `deploy-guides/new-client-kickoff-kit.md` for credentials, required settings, and how to test it safely before going live.

The workflow ships turned off, with sample test data already loaded, so you can test it end-to-end before connecting it to a real signed client.

## Compliance

This is pure administrative setup — folder creation, scheduling links, checklist copy, and task creation. It never drafts legal text or exercises legal judgment (per ABA Opinion 512).

## Want more?

Want Clio matter creation, DocuSign, payment processing, and a client portal built into the same kickoff flow? [Book a call with Protomated](https://protomated.com/book).
