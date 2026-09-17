# Deploy Guide: Workflow Backup to GitHub

**Template:** PTPAC-56 — OPS6: Workflow/Asset Backup to GitHub
**Pillar:** Ops
**Replaces:** Manual backups

Get value in under 15 minutes. Every automation and configuration value your firm relies on lives inside one n8n instance, with no version history — if it's ever deleted, corrupted, or misconfigured, there's nothing to roll back to. This workflow backs up every workflow and every configuration variable to a GitHub repository on a schedule, so you always have a point-in-time copy with full version history.

---

## Before you start: the most important thing to understand

**Credentials are never backed up.** n8n's own API strips credential secrets from workflow exports automatically, and this workflow doesn't call the credentials API at all — only workflow definitions and configuration variables. A leaked backup repo can't leak a password.

**This backs up the SAME n8n instance it runs on.** It calls this instance's own REST API to list workflows and variables — there's no separate "source" instance involved.

---

## What you need before you start

- A GitHub account, and a private repository created to hold the backups (e.g. `yourfirm/n8n-backups`)
- Ability to generate an n8n API key from this instance's own settings
- Ability to generate a GitHub Personal Access Token with repo write access
- An email account with SMTP access (for failure alerts only)

---

## Step 1 — Generate an n8n API key (2 min)

In this same n8n instance: **Settings → n8n API → Create an API key**. Copy it — you'll paste it into a credential in Step 3.

---

## Step 2 — Generate a GitHub token and create the backup repo (3 min)

1. Create a new **private** GitHub repository (e.g. `yourfirm/n8n-backups`). It can start empty.
2. Generate a GitHub Personal Access Token (classic or fine-grained) with **Contents: write** access scoped to that repository.

---

## Step 3 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-56-workflow-backup-to-github.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-56-workflow-backup-to-github.json). Do not activate it yet.
2. n8n credential: **Credentials → New → Header Auth** → name it `n8n API (self)` → header name `X-N8N-API-KEY` → value: the API key from Step 1.
3. GitHub credential: **Credentials → New → Header Auth** → name it `GitHub API (backup PAT)` → header name `Authorization` → value: `Bearer <your token>`.
4. Email credential: **Credentials → New → SMTP**.

---

## Step 4 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `N8N_INSTANCE_URL` | This same n8n instance's own base URL, no trailing slash |
| `GITHUB_BACKUP_REPO` | `owner/repo` for the backup repository created in Step 2 |
| `GITHUB_BACKUP_BRANCH` *(optional)* | Defaults to `main` |
| `FIRM_NAME` | Your law firm name |
| `FIRM_FROM_EMAIL` | Sender address for failure alerts |
| `FIRM_STAFF_EMAIL` | Who gets alerted if a backup fails |

---

## Step 5 — Activate (1 min)

1. Toggle the workflow to **Active**.
2. Adjust the cron expression on **"Run Daily Backup"** (default 3am) if a different time suits.

---

## Step 6 — Test

1. Run **"Fetch All Workflows From n8n"** → **"Split Out Workflow List"** and confirm one item per workflow currently in this instance (including this one).
2. Continue through **"Build GitHub File For Workflow"** → **"Fetch Existing Workflow File SHA"** → **"Parse Existing Workflow File SHA"** → **"Commit Workflow To GitHub"** → **"Parse Workflow Commit Result"** and confirm `success: true`, then check the file actually landed in your GitHub repo.
3. Run **"Fetch All n8n Variables"** → **"Build GitHub File For Variables"** → through the same commit chain and confirm `variables-backup.json` lands in the repo.
4. Confirm **"Merge Workflow And Variable Backup Results"** → **"Build Backup Summary"** shows `any_failed: false` when everything succeeds.
5. To test the failure path: temporarily break the GitHub credential (wrong token), re-run, and confirm it routes to **"Build Backup Failure Alert Email"** with the specific error message, rather than failing silently.
6. Re-run the workflow a second time and confirm the same files get **updated** in place (check the repo's commit history) rather than duplicated.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A workflow has never been backed up before | A new file is created in the repo |
| A workflow was already backed up | The existing file is updated in place — Git keeps the full history |
| The GitHub API rejects a commit (bad token, wrong repo) | Surfaced by name in the failure alert email, not silently dropped |
| Everything backs up successfully | No email is sent — the daily run stays silent |
| A firm has renamed a workflow since its last backup | The backup file name is stable (based on workflow ID), so it still updates the same file |

---

## Compliance note

This template performs backup and version-tracking of automation configuration only — it never touches legal work, client data, or case content (ABA Op. 512).

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces manual, easy-to-forget backups with a version-controlled, automatic daily one |
| Compliance test | Configuration backup only; no legal work, no client data, no credentials |
| Searchability test | Ranks for "n8n workflow backup automation" |
| Deployability test | Reuses no external credentials beyond GitHub and this instance's own API key — under 15 minutes |
| Upsell test | Clear Done-for-You path: multi-instance backup, restore tooling, and disaster-recovery runbooks |

---

## Want the advanced version?

Protomated builds the done-for-you **Done-for-You** implementation that adds:
- One-click restore tooling, not just backup
- Multi-instance backup for firms running more than one n8n environment
- Prompt and AI-configuration backup for firms running LLM-based automations
- Full disaster-recovery runbook and testing

[Book a call with Protomated](https://protomated.com/book) to get started.
