---
name: thread-mail
description: >-
  Read a user's email through Thread — a scoped, read-only mail feed for agents.
  Use this whenever you need to see the user's mail: triage an inbox, watch for
  new messages, pull a receipt or invoice, or download an attachment (a PDF, an
  image). One API key, one endpoint you poll on a timer — there is no push, and
  every call (even an empty one) counts against a daily limit, so cadence matters.
  The key only ever returns the mail the user granted, already filtered — you never
  see their whole inbox, and you never handle a permission error, because
  out-of-scope mail is simply absent. Bodies are graded: absent, masked, or raw.
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
> doubt while developing, **peek with `?limit=`**; it never moves your watermark,
> so it's safe to repeat. (Safe for your *position* — it still costs a call against
> the daily limit. See **Cadence and cost**.)

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
off, see below), `503` (transient upstream — retry later).

## Cadence and cost — how often to poll

Mail does not arrive as a push to your key. **Polling the feed is the only way to
see new mail**, so you own the cadence, and the cadence has a price.

Three facts that decide it:

1. **Nothing new can exist sooner than ~60s.** Thread polls the mailbox on its own
   interval (60s by default) and classifies what it finds. Polling every 5s does not
   get you mail 12× sooner — it gets you 12× the cost for the same mail.
2. **Every successful call costs one unit of the daily limit — including a call that
   returns zero emails.** The limit counts *allowed calls*, not emails delivered. An
   empty feed response is a full-price call. So are peeks (`?limit=`), single fetches
   (`GET /emails/{id}`), and attachment downloads. Only *denied* calls are free.
3. **The user may have set that daily limit, and you can't read it.** It's optional —
   many grants have none — but when it exists it's a number the user typed, invisible
   to you until you hit `429`. Assume it's small.

So: **poll no faster than once every 60 seconds.** If you learn the limit (the user
tells you, or you hit `429` and count), keep your interval at or above
`86400 / limit` seconds — a 100/day limit means one call every ~15 minutes, not one
a minute. Budget for the fetches and downloads you make *in addition to* the poll.

Do the work between polls, not more polls. One bare call hands you every new email
at once; there is no reason to poll again until the next interval.

**On `429`:** the window is a **UTC calendar day**, not a rolling 24 hours. Your
budget resets at `00:00Z`, so back off until then — retrying in an hour just burns
another call. Do not retry-storm; `429` is a fact about the grant, not a transient
error, and no amount of retrying will change it before midnight UTC.

If you need mail to arrive faster than a poll interval allows, that's a
conversation with the user about their Thread setup — not something to solve by
polling harder.

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

## Message bodies — three grades, and you must read `masked`

Body access is **graded**. The user picked one grade when they minted the key:

| Grade          | You get                                  | `body`  | `masked` |
|----------------|------------------------------------------|---------|----------|
| **Subject only** | metadata: `from`, `subject`, `labels`  | absent  | absent   |
| **Masked body**  | body with PII redacted (the default)   | present | `true`   |
| **Full body**    | the raw body, verbatim                 | present | `false`  |

**Never assume the body is redacted.** `masked` exists precisely so you don't have to
guess — read it:

- **`masked: true`** — emails, phone numbers, cards and SSNs have been redacted out.
  Identifiers you need may simply not be there. Don't report a redacted string as if
  it were a real account number.
- **`masked: false`** — this is the **raw text of the user's private mail**, nothing
  removed. Treat it accordingly: use it for the task at hand, don't log it, don't echo
  it back wholesale, don't send it anywhere the user didn't ask you to.
- **`body` absent** — the key is subject-only. Work from `subject`, `from`, and
  `labels`, and don't report this as an error.

Masked is the default grade, so a body you get is *most often* redacted — but "most
often" is not "always", and the field is right there.

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

# Peek first while you develop — never consumes, so it's safe to repeat.
curl -s -H "$AUTH" "$BASE/emails?limit=20" | jq '.emails[] | {id, from, subject, labels, attachments}'

# The real loop — consumes: each call returns only what's new since the last.
# 60s is the floor; raise it if the grant has a daily limit (86400 / limit).
while true; do
  code=$(curl -s -o /tmp/feed.json -w '%{http_code}' -H "$AUTH" "$BASE/emails")
  case "$code" in
    200) jq -c '.emails[]' /tmp/feed.json | while read -r email; do
           handle "$email"   # empty output is normal — nothing new in scope
         done ;;
    429) # Daily cap. Resets at 00:00 UTC — sleep to it, don't retry sooner.
         sleep $(( $(date -u -d 'tomorrow 00:00' +%s 2>/dev/null \
                  || date -u -v+1d -v0H -v0M -v0S +%s) - $(date -u +%s) )); continue ;;
    401) echo "key invalid/revoked — stop, ask the user to re-check it"; exit 1 ;;
    503) sleep 300; continue ;;   # transient upstream
  esac
  sleep 60
done

# Download a PDF you saw in the feed (bytes → file). Costs a call, like any other.
curl -s -H "$AUTH" "$BASE/emails/$ID/attachments/$ATTACHMENT_ID" -o invoice.pdf
```

## What this key can and can't do

Read-only. You always get metadata; you get the body at whatever grade the user
chose (masked by default, see above), and attachments when granted. It **cannot**
send, reply, label, archive, delete, or change anything in the mailbox. If the user
asks you to act on their mail, you can read to decide *what* to do, but the doing is
outside this key.

It also **cannot receive push**. There is no webhook, no callback, no long-poll for
an API key — Thread's push delivery exists, but it's for registered third-party apps
with a callback URL, not for the key you hold. Your loop is the mechanism.
