---
name: thread-mail
description: >-
  Read a user's email through Thread, a scoped mail feed for agents, and, when the
  user allowed it, draft or send replies. Use this whenever you need to see the
  user's mail: triage an inbox, watch for new messages, find what someone is asking,
  pull a receipt or invoice, download an attachment, or answer an email on their
  behalf. One API key, a handful of calls. The feed includes the full message text
  whenever the user allowed it; call GET /me first and it tells you exactly what you
  have, including whether you may draft or send. You only ever see the mail the user
  granted, already filtered, so there is no permission error to handle: out-of-scope
  mail is absent.
---

# Thread mail

You have a **Thread API key**. It gives you a scoped view of one user's mailbox. The
user chose the scope once, when they created the key: which categories of mail, how
much of each email (subject only, text with personal details masked, or full text), and
whether you may write drafts or send replies. Your job is to ask for mail, use what
comes back, and write only what the key allows.

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
  "send": null,                                     // or a policy object; see "Replying and sending"
  "feed": { "endpoint": "/emails", "singleEmail": "/emails/{id}", "attachment": "...", "firstCall": "..." }
}
```

`bodyGrade` is the one field to remember for reading. It tells you what `body` will hold
on every email before you fetch anything. `send` is the one field to remember for writing:
`null` means read only. `/me` discloses no mail and is free: it never counts against the
daily limit, so you can call it whenever you are unsure.

## The whole API

Reading is three calls, plus attachments. Writing is three more, and most keys are not
allowed any of them.

```
GET  {BASE}/me                              what this key can read and do
GET  {BASE}/emails                          everything NEW since your last call (the loop)
GET  {BASE}/emails?limit=20                 a peek at the last 20 in-scope emails
GET  {BASE}/emails/{id}                     one email by id
GET  {BASE}/emails/{id}/attachments/{aid}   one attachment's bytes
POST {BASE}/emails/{id}/reply               reply in a thread you can read (if /me.send allows)
POST {BASE}/emails                          a new email (only if /me.send.replyOnly is false)
POST {BASE}/drafts                          a draft for the user to send (if /me.send is not null)
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

## Replying and sending

Writing is a separate permission, off by default. `GET /me` tells you where you stand in
its `send` field:

```jsonc
"send": {
  "mode": "drafts_only",                 // or "send"
  "replyOnly": true,                     // false: POST /emails may start new emails
  "newRecipients": false,                // true: a reply may cc people who were not on the thread
  "dailySendCap": null,                  // sends per UTC day, or null; drafts never count
  "recipientsFromAllowedCategoriesOnly": true,
  "mailboxEnabled": true,                // false: the user has not turned sending on for the mailbox yet
  "routes": { "reply": "POST /emails/{id}/reply", "newEmail": "POST /emails", "draft": "POST /drafts" },
  "howTo": "..."                         // the same rules in a sentence
}
```

- **`send: null`**: read only. You cannot draft or send. If the user asks you to answer an
  email, write the reply text in chat and say sending is outside this key.
- **`mode: "drafts_only"`**: you write, the user sends. `POST /emails/{id}/reply` with
  `{ "body": "...", "draft": true }` (or `POST /drafts` with `{ "inReplyTo": id, "body": "..." }`)
  puts a threaded reply in the user's Drafts folder. Plain `POST /emails/{id}/reply` is
  refused with `403 SEND_NOT_GRANTED`.
- **`mode: "send"`**: `POST /emails/{id}/reply` with `{ "body": "..." }` sends the reply from
  the user's own address, inside the original thread. Add `"replyAll": true` to answer
  everyone on the thread, `"cc": [...]` for extra people (needs `newRecipients`).
- **`mailboxEnabled: false`**: every draft and send answers `501` until the user turns
  sending on for the mailbox in their Thread portal under Connections. Tell them that; do
  not retry.

Rules that hold regardless of mode:

- **You can only reply to mail you can read.** The original must be in the key's scope;
  an out-of-scope id is `404`, exactly like a read.
- **Plain text only.** What you put in `body` is exactly what is sent. No HTML, no
  attachments on outbound mail in this version.
- **Show the user the text before you send.** Sending is irreversible and goes out under
  their name. The sensible default is to draft (or paste the text in chat), get a yes,
  then send.
- **Skip your own mail.** A reply you sent comes back through the feed a minute later
  flagged `"sentByAgent": true`. Never treat it as new inbound mail or answer it.
- Policy refusals are `403` with a code that says why: `SEND_NOT_GRANTED`,
  `POLICY_REPLY_ONLY`, `POLICY_NEW_RECIPIENTS`, `POLICY_UNKNOWN_RECIPIENT`. A send cap is
  `429 SEND_LIMIT_REACHED` (resets at 00:00 UTC). None of these are transient; do not retry
  them, explain them.

```bash
# Reply (sends when /me.send.mode is "send"; add "draft": true to save instead)
curl -s -X POST -H "$AUTH" -H 'Content-Type: application/json' \
  "$BASE/emails/$ID/reply" -d '{"body":"Thursday at 3 works for me.\n\nParag"}'
# → { "id": "…", "threadId": "…", "status": "sent" }      (or "drafted")
```

## What this key can and can't do

You always get metadata; you get the text at whatever grade the user chose, and attachments
when granted. Drafting and sending exist only when `/me.send` says so, and only as far as it
says. The key can **never** label, archive, delete, or change anything already in the
mailbox, and it cannot receive push: no webhook, no callback. Your scheduled poll is the
mechanism.
