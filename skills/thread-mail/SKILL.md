---
name: thread-mail
description: >-
  Read a user's email through Thread — a scoped, read-only mail feed for agents.
  Use this whenever you need to see the user's mail: triage an inbox, watch for
  new messages, pull a receipt or invoice, or download an attachment (a PDF, an
  image). One API key, one endpoint to loop on. The key only ever returns the
  mail the user granted, already filtered — you never see their whole inbox, and
  you never handle a permission error, because out-of-scope mail is simply absent.
---

# Thread mail

You have a **Thread API key**. It gives you a scoped, read-only view of one user's
mailbox. You did not choose the scope — the user did, once, when they minted the
key. Your job is to ask for mail and use what comes back. Everything about
filtering, classification, and privacy is Thread's job, invisible to you.

## Setup

Two values, both from the user:

- `THREAD_API_KEY` — looks like `thread_sk_…`. Send it as `Authorization: Bearer <key>`.
- `THREAD_BASE_URL` — the gateway base. Defaults to `http://localhost:3002` in dev.
  In production the user gives you the real host.

Never print the key, never write it into a file you commit, never paste it into a
prompt. Read it from the environment.

## The whole API

Four calls. That's the entire surface.

```
GET  {BASE}/emails                         → everything NEW since your last call (the loop)
GET  {BASE}/emails?limit=20                → a peek at the last 20 in-scope emails
GET  {BASE}/emails/{id}                     → one email by id
GET  {BASE}/emails/{id}/attachments/{aid}   → download one attachment's bytes
Authorization: Bearer {THREAD_API_KEY}
```

Full field-by-field contract: **[references/api.md](references/api.md)**. Read it
before you write parsing code — the response shape has a few deliberate subtleties
(below) that will bite you if you guess.

## The one mental model: consume vs peek

This is the only thing you must get right.

- **Bare `GET /emails` CONSUMES.** It returns everything new since *your key's last
  bare call*, then advances a server-side watermark. Call it again and those same
  emails are gone from the response — you've already seen them. This is your **loop**:
  poll it, handle what's new, poll again.
- **Any parameter (`?limit`, `?since`, `?cursor`) is a PEEK.** It never moves the
  watermark. Use peeks for backfill, catch-up, "show me recent mail", or just
  looking around without spending your position.

> If you poll with the bare call and crash before handling the results, those
> emails are already consumed — the next bare call won't return them. When in
> doubt while developing, **peek with `?limit=`**; it's idempotent and safe to
> repeat.

Freshness: Thread ingests new mail within ~60s, and it is **forward-only** — it
does not import mail that arrived before the mailbox was connected. If you're
testing, send a *new* email and wait a minute; old mail will never appear.

## Absence is not an error

Thread already filtered the feed to what the user allowed. So:

- An **empty `emails` array is normal.** It means nothing new is in your scope. It
  does **not** mean the API is broken. Never retry-storm or alarm on it.
- **Out-of-scope mail is simply missing** from the feed — there is no per-email
  "denied". You will never see mail you weren't granted, and you never write code
  to handle a rejection for it.
- **`GET /emails/{id}` returns 404 when the email is out of scope OR doesn't exist
  — the two are indistinguishable by design.** Treat 404 as "not available to me,"
  full stop. Don't infer the email exists.

The only errors you actually handle: `401` (bad/expired/revoked key — stop and tell
the user to re-check the key or re-run Connect), `429` (daily rate limit — back
off), `503` (transient upstream — retry later).

## Attachments — how to actually get a file

Two steps. There is **no "list attachments" call**; the handles come to you in the
feed.

1. **Discover.** Each email object may carry an `attachments` array:
   `[{ id, name, type, size }]`. If it's present, those are downloadable handles.
2. **Download.** `GET /emails/{id}/attachments/{attachmentId}` returns the **raw
   bytes** (not JSON) with a `Content-Type` and a `Content-Disposition` filename.
   Save or process the bytes directly.

Three things that will trip you up if you don't read them:

- **`hasAttachment` can be ABSENT — that means "unknown," not "no."** Some providers
  (Gmail on a metadata-only grant) don't reveal attachments without a content fetch,
  so Thread omits the field rather than lie with `false`. **Never branch on
  `if (email.hasAttachment)`** to decide whether to look — an email with a real PDF
  can have no `hasAttachment` field at all. Check for the `attachments` array
  instead, or fetch the single email by id.
- **`attachments` is present only if the key was granted attachment scope.** If it's
  absent, either the grant doesn't include attachments or this email is out of
  content-scope. Degrade gracefully — don't treat its absence as an error.
- **A 404 on the download** means the same as everywhere else: not granted, or gone.
  Don't retry it as if it were transient.

## Message bodies

If the user granted message content, the email object includes `body` and
`masked: true`. **The body is ALWAYS masked** — emails, phone numbers, cards, SSNs
are redacted, and Thread never stores or serves the raw text. There is no raw
option; don't look for one. If `body` is absent, the key wasn't granted content —
work from `subject`, `from`, and `labels`.

## Two kinds of labels — don't confuse them

- `labels` — **Thread's** semantic labels, e.g. `["financial","invoice"]`. This is
  what the user's grant is written against. Reason about *what a message is* with these.
- `mailboxLabels` — the **mailbox's own** labels, verbatim (Gmail `INBOX`/`UNREAD`/
  `CATEGORY_*`, Outlook categories + folder, IMAP folder + flags). Reason about *inbox
  state* — is it unread, starred, in the inbox — with these (or the `isUnread` boolean).

## A minimal loop

```bash
BASE="${THREAD_BASE_URL:-http://localhost:3002}"
AUTH="Authorization: Bearer $THREAD_API_KEY"

# Peek first while you develop — safe to repeat, never consumes.
curl -s -H "$AUTH" "$BASE/emails?limit=20" | jq '.emails[] | {id, from, subject, labels, attachments}'

# The real loop — consumes: each call returns only what's new since the last.
curl -s -H "$AUTH" "$BASE/emails" | jq

# Download a PDF you saw in the feed (bytes → file):
curl -s -H "$AUTH" "$BASE/emails/$ID/attachments/$ATTACHMENT_ID" -o invoice.pdf
```

## What this key can and can't do

Read-only, metadata by default; body and attachments when granted. It **cannot**
send, reply, label, archive, delete, or change anything in the mailbox. If the user
asks you to act on their mail, you can read to decide *what* to do, but the doing is
outside this key.
