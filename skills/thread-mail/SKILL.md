---
name: thread-mail
description: >-
  Read a user's email through Thread, a scoped, read-only mail feed for agents.
  Use this whenever you need to see the user's mail: triage an inbox, watch for
  new messages, find what someone is asking, pull a receipt or invoice, or
  download an attachment. One API key, three calls. The feed includes the full
  message text whenever the user allowed it; call GET /me first and it tells you
  exactly what you have. You only ever see the mail the user granted, already
  filtered, so there is no permission error to handle: out-of-scope mail is absent.
---

# Thread mail

You have a **Thread API key**. It gives you a scoped, read-only view of one user's
mailbox. The user chose the scope once, when they created the key: which categories
of mail, and how much of each email (subject only, text with personal details masked,
or full text). Your job is to ask for mail and use what comes back.

## Setup

One value from the user, read from the environment, never printed, never committed:

- `THREAD_API_KEY`, looks like `thread_sk_...`. Send it as `Authorization: Bearer <key>`.
- `THREAD_BASE_URL` is optional and defaults to `https://gw.thread.xyz`.

The quickest way to use the key is the official CLI, which needs no install step:

```bash
npx @jointhread/mail me              # what this key can read. Run this first.
npx @jointhread/mail new             # everything new since your last `new`, text included
npx @jointhread/mail peek 20         # the last 20 in-scope emails, without moving your position
npx @jointhread/mail read <id>       # one email with its full text
npx @jointhread/mail attachment <id> <attachmentId> [file]
```

Add `--json` to any command for the raw response. Prefer the CLI over writing your own
client: every hand-written client we have seen dropped the `body` field and concluded
Thread was "metadata only". It is not.

## Step one, always: find out what you have

```bash
curl -s -H "Authorization: Bearer $THREAD_API_KEY" https://gw.thread.xyz/me
```

```jsonc
{
  "mailbox": "you@example.com",
  "provider": "gmail",
  "keyName": "claude-code",
  "categories": ["financial", "travel", "work"],   // or "all"
  "narrowedBy": [],                                 // e.g. ["advanced"] when a sender/subject rule narrows it
  "bodyGrade": "masked",                            // "subject_only" | "masked" | "full"
  "attachments": true,
  "dailyLimit": null,                               // calls per UTC day, or null
  "expiresAt": "2027-02-17T00:00:00.000Z",
  "calendar": [],
  "feed": { "endpoint": "/emails", "singleEmail": "/emails/{id}", "attachment": "...", "firstCall": "..." }
}
```

`bodyGrade` is the one field to remember. It tells you what `body` will hold on every
email before you fetch anything. `/me` discloses no mail and is free: it never counts
against the daily limit, so you can call it whenever you are unsure.

## The whole API

Three calls, plus attachments. That is the entire surface.

```
GET  {BASE}/me                              what this key can read
GET  {BASE}/emails                          everything NEW since your last call (the loop)
GET  {BASE}/emails?limit=20                 a peek at the last 20 in-scope emails
GET  {BASE}/emails/{id}                     one email by id
GET  {BASE}/emails/{id}/attachments/{aid}   one attachment's bytes
Authorization: Bearer {THREAD_API_KEY}
```

Field-by-field contract: **[references/api.md](references/api.md)**. Read it before you
write parsing code.

## The message text is in the feed

Every email object carries `body` when the user granted text. Three states, each
distinguishable without guessing:

| `body` | `bodyGrade` | Meaning |
|---|---|---|
| a string | `masked` or `full` | The text, at the granted grade. `masked: true` means personal details were redacted. |
| `null` | `subject_only` | This key was never granted text. Work from `subject`, `from`, `labels`. Do not report it as an error. |
| absent | absent | The key has text access, but this one email is outside its text scope. |

Do not print only the subject and call it a summary. If the user asks "what is this email
about" or "what are they asking me", read `body` (or `GET /emails/{id}`) and answer from
the text. If `masked: true`, identifiers like `[redacted-phone]` are placeholders; never
report one as a real number.

`masked: false` is the user's private mail with nothing removed. Use it for the task at
hand, do not log it, do not echo it back wholesale, do not send it anywhere the user did
not ask you to.

## The one mental model: consume vs peek

- **Bare `GET /emails` CONSUMES.** It returns everything new since *your key's last bare
  call*, then moves a server-side position forward. Call it again and those same emails
  are gone from the response. This is your loop: poll, handle what is new, poll again.
  Save each batch before you act on it.
- **Any parameter (`?limit`, `?since`, `?cursor`) is a PEEK.** It never moves the
  position. Use peeks for catch-up, "show me recent mail", or looking around while you
  develop. Safe to repeat. Still costs a call against the daily limit.

**First call:** right after a mailbox is connected, the feed returns the user's 50 most
recent in-scope emails, read or unread, so you have something to work with on day one.
After that, only what is new. Thread ingests new mail within about 60 seconds.

## Absence is not an error

Thread already filtered the feed to what the user allowed. So:

- An **empty `emails` array is normal.** Nothing new is in scope. Never retry-storm or
  alarm on it.
- **Out-of-scope mail is simply missing.** There is no per-email "denied".
- **`GET /emails/{id}` returns 404 when the email is out of scope OR does not exist.** The
  two are indistinguishable by design. Treat 404 as "not available to me".

The only errors you handle: `401` (bad, expired or revoked key: stop and tell the user to
check it in their dashboard), `429` (daily limit: back off, see below), `503` (transient:
retry later).

## Cadence and cost

Mail does not arrive as a push. Polling the feed is the only way to see new mail, so you
own the cadence, and the cadence has a price.

1. **Nothing new can exist sooner than about 60 seconds.** Thread polls the mailbox on its
   own interval. Polling every 5 seconds gets you the same mail 12 times the cost.
2. **Every successful call counts one unit against the daily limit, including a call that
   returns zero emails.** Peeks, single fetches and attachment downloads count too. Denied
   calls and `GET /me` do not.
3. **The daily limit, if any, is in `/me` as `dailyLimit`.** You no longer have to discover
   it by hitting 429.

So: poll no faster than once every 60 seconds. With a daily limit, keep your interval at
or above `86400 / dailyLimit` seconds (100 a day means one call every 15 minutes), and
leave room for the single-email reads you make between polls.

**On `429`:** the window is a UTC calendar day. Your budget resets at `00:00Z`, so back off
until then. Retrying in an hour just burns another call.

## Running the loop for real

The loop only watches while it is running, and a chat turn is not a place it can live.
Run it inline and you either block the conversation or stop watching when the session
ends, and it looks like "the skill doesn't show me emails". It does; nothing was running.

**What one scheduled run does.** Exactly one cycle, then exit: one bare `GET /emails`
(or `npx @jointhread/mail new`), handle what came back, tell the user if anything
arrived, done. The server-side position is your state: the next run resumes where this
one stopped. No cursor file, no dedup store.

**What to schedule it with**, in order. Take the first one your runtime has:

1. **A host scheduler or cron tool** (Claude Code `CronCreate`, `crontab`, a systemd timer,
   a platform cron trigger). The only option that keeps running after the conversation
   ends, which is what "watch my mail" nearly always means.
2. **A host loop primitive** that re-invokes you on an interval (Claude Code `/loop 15m`).
   Good for this session only, with the user watching.
3. **Nothing available? Say so.** Do one poll, report, and tell the user plainly that you
   cannot keep watching without a scheduler. Do not substitute an inline `while true` or a
   long `sleep`.

**Pick the interval** from the cadence rules above. When you do not know the limit, 15
minutes is the safe default.

## Attachments

There is no "list attachments" call; the handles come in the feed.

1. **Discover.** An email object may carry `attachments: [{ id, name, type, size }]`.
2. **Download.** `GET /emails/{id}/attachments/{attachmentId}` returns the raw bytes with
   a `Content-Type` and a `Content-Disposition` filename.

Two things that will trip you up:

- **`hasAttachment` can be ABSENT, which means unknown, not no.** Gmail does not reveal
  attachments on a metadata-only grant. Never branch on `if (email.hasAttachment)`;
  check for the `attachments` array, or fetch the email by id.
- **`attachments` is present only if the key was granted attachment access** (`/me` says).
  Absent means not granted or out of scope for this email, not "no files".

## Two kinds of labels

- `labels`: **Thread's** labels, e.g. `["financial", "invoice"]`. What the user's grant is
  written against. Reason about *what a message is* with these.
- `mailboxLabels`: the **mailbox's own** labels, verbatim (Gmail `INBOX`, `UNREAD`,
  `CATEGORY_*`; Outlook categories and folder; IMAP folder and flags). Reason about
  *inbox state* with these, or with the `isUnread` boolean.

## A minimal loop without the CLI

```bash
BASE="${THREAD_BASE_URL:-https://gw.thread.xyz}"
AUTH="Authorization: Bearer $THREAD_API_KEY"

# First: what do I have?
curl -s -H "$AUTH" "$BASE/me" | jq '{mailbox, categories, bodyGrade, attachments, dailyLimit}'

# Peek while you develop. Never consumes, so it is safe to repeat. Print the text too.
curl -s -H "$AUTH" "$BASE/emails?limit=20" \
  | jq -r '.emails[] | "\(.id)  \(.from)  \(.subject)\n\(.body // "(no text)")\n---"'

# The real loop. Consumes: each call returns only what is new since the last.
# 60s is the floor; raise it if /me reports a dailyLimit (86400 / dailyLimit).
code=$(curl -s -o /tmp/feed.json -w '%{http_code}' -H "$AUTH" "$BASE/emails")
case "$code" in
  200) jq -c '.emails[]' /tmp/feed.json | while read -r email; do
         handle "$email"   # the whole object, body included. Empty output is normal.
       done ;;
  429) echo "daily limit reached; resets at 00:00 UTC" ;;
  401) echo "key invalid or revoked; ask the user to check it"; exit 1 ;;
  503) echo "transient; retry later" ;;
esac

# One email's full text
curl -s -H "$AUTH" "$BASE/emails/$ID" | jq -r '.body'

# One attachment to a file
curl -s -H "$AUTH" "$BASE/emails/$ID/attachments/$ATTACHMENT_ID" -o invoice.pdf
```

## What this key can and can't do

Read-only. You always get metadata; you get the text at whatever grade the user chose,
and attachments when granted. It **cannot** send, reply, label, archive, delete, or change
anything in the mailbox. If the user asks you to act on their mail, read to decide what to
do, draft the reply for them, and say that sending is outside this key.

It also **cannot receive push.** No webhook, no callback. Your scheduled poll is the
mechanism.
