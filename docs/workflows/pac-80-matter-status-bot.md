# Deploy Guide: Matter Status Bot

**Template:** PTPAC-80 — K4: Matter Status Bot
**Pillar:** Keep
**Replaces:** Standalone case-status apps

Get value in under 15 minutes. Clients interrupt attorneys constantly to ask what's happening with their case. This workflow answers directly from Clio's own matter status — by text or email — with no attorney interruption needed.

---

## Before you start: the most important thing to understand

**This bot only ever reports status — it never gives legal advice, predicts outcomes, or answers "what happens next."** The reply is limited strictly to the matter's status, practice area, and reference number as recorded in Clio. Nothing is interpreted, summarized, or added (ABA Op. 512). Every reply ends by directing the client to their attorney for anything beyond status.

**Client identification is phone/email matching against Clio contacts — not authentication.** Whoever texts or emails from a number/address on file as a client's contact gets that client's status back. There's no PIN, code, or second factor. If your firm handles matters where this level of identification isn't sufficient (shared family phone lines, sensitive matters), think carefully before enabling the SMS branch specifically.

**This has two independent parts on one canvas:**
1. **SMS status requests** — a client texts your firm's Twilio number; the bot replies by text.
2. **Email status requests** — the same idea watching a dedicated inbox via IMAP (e.g. `case-status@yourfirm.com`).

**Multiple open matters are listed, not guessed at.** If a client has more than one open matter, the reply lists all of them rather than assuming which one they mean.

---

## What you need before you start

- The Bar-Compliance Guardrail (NTC-33) already deployed — every reply goes through it
- A Clio account with API access (reuse the OAuth2 credential from other templates in this catalog if already deployed)
- A Twilio account and phone number, for the SMS branch
- A dedicated email inbox with IMAP access, for the email branch (a fresh mailbox like `case-status@yourfirm.com` is cleaner than reusing a general inbox)
- An SMTP-capable email account for sending replies

---

## Step 1 — Import the workflow and add credentials (5 min)

1. Open your n8n canvas, press `Ctrl+V` (or `Cmd+V`), and paste the contents of [`pac-80-matter-status-bot.json`](https://raw.githubusercontent.com/protomated/n8n-legal-workflows-release/main/workflows/pac-80-matter-status-bot.json). Do not activate it yet.
2. Clio credential: reuse your existing `Clio API (OAuth2)` credential if already deployed elsewhere in this catalog.
3. Twilio credential: **Credentials → New → Twilio** → Account SID and Auth Token.
4. IMAP credential: **Credentials → New → IMAP** → your case-status mailbox's server/login details.
5. SMTP credential: **Credentials → New → SMTP**.

---

## Step 2 — Set n8n Variables (3 min)

| Variable | What to enter |
|---|---|
| `CLIO_BASE_URL` | Same value as your other Clio-integrated templates |
| `FIRM_NAME` | Your law firm name |
| `FIRM_TWILIO_NUMBER` | Your firm's Twilio number, in E.164 format (e.g. `+15559998888`) |
| `FIRM_FROM_EMAIL` | The sender address for email replies |
| `GUARDRAIL_WORKFLOW_ID` | The numeric ID of the Bar-Compliance Guardrail workflow |

---

## Step 3 — Activate and wire Twilio (3 min)

1. Toggle the workflow to **Active**.
2. Open **"When Client Texts For Status"**, copy its **Production URL**, and set it as your Twilio number's "A message comes in" webhook (Twilio Console → Phone Numbers → your number → Messaging).

The email branch needs no separate webhook wiring — it polls the IMAP mailbox directly on its own.

---

## Step 4 — Test

The workflow ships with pinned sample data: a text from a known client's phone number (matching a Clio contact with one open matter), and an email from a known client's address (matching a contact with two open matters, to exercise the multi-matter case).

**SMS branch:**
1. Run **"When Client Texts For Status"** → **"Parse Incoming Status Text"** → **"Fetch Clio Contact By Phone"** → **"Match Client By Phone Number"** and confirm `is_found: true` despite the pinned phone number being formatted differently (`+1 (555) 123-4567` vs `+15551234567`).
2. Continue through **"Fetch Client Open Matters From Clio"** → **"Build Matter Status Reply"** and confirm the reply names the matter, practice area, and status — nothing else.
3. Edit the pinned webhook data's `From` to a number not in the pinned contact list, re-run, and confirm it routes to **"Build Client Not Found Reply"** with a generic message.

**Email branch:**
4. Run **"When Client Emails For Status"** → through **"Fetch Client Open Matters For Email Reply"** → **"Build Email Matter Status Reply"** and confirm the reply lists **both** pinned matters (Personal Injury and Estate Planning), since this contact has two open matters.

**Testing against your real setup:** unpin all Clio-facing nodes, connect real credentials, and text your Twilio number from a phone whose number is on file for a real (or test) Clio contact with an open matter. Confirm the reply arrives and reads correctly. Repeat by emailing your case-status inbox from that contact's email address.

---

## How the workflow behaves

| Scenario | What happens |
|---|---|
| A known client texts or emails asking about their case | They get a status-only reply for their open matter(s) |
| The phone/email doesn't match any Clio contact | A generic "please call our office" reply, no client-specific content |
| A client has zero open matters | The reply says so explicitly rather than going silent |
| A client has multiple open matters | All of them are listed, not just one |
| The client has opted out of texts/emails | The status reply is suppressed — the opt-out is honored even for a reply they themselves requested |

---

## Compliance note

This template performs status reporting only — relaying a matter's existing status field, practice area, and reference number exactly as recorded in Clio, never any legal judgment, prediction, or advice (ABA Op. 512). The Bar-Compliance Guardrail is used on every reply: opt-out respected, required disclaimer applied, send logged.

---

## Quality bar

| Check | Status |
|---|---|
| Money / leak test | Replaces a standalone case-status app subscription and recovers attorney hours lost to routine status interruptions |
| Compliance test | Status-only replies from Clio's own data; no legal advice or judgment |
| Searchability test | Ranks for "client case status chatbot" |
| Deployability test | Reuses Clio credentials from other templates if already deployed — under 15 minutes |
| Upsell test | Clear Done-for-You path: productized as a fixed-scope build with authentication options and PM systems beyond Clio |

---

## Want the advanced version?

Protomated builds the done-for-you **Done-for-You** implementation that adds:
- Stronger client identification (PIN/code verification) for sensitive matters
- Support for practice management systems beyond Clio
- Rich status detail (next scheduled event, document requests outstanding) within the same compliance boundaries
- Web chat and portal channels alongside SMS and email

[Book a call with Protomated](https://protomated.com/book) to get started.
