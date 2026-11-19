# Thread mail API: reference

The complete contract for the agent-facing mail feed. Mirrors the gateway's OpenAPI
(`GET https://gw.thread.xyz/openapi.json`). If this file and the live spec disagree, the
live spec wins; this is a rendered copy.

Base URL (`{BASE}`): `https://gw.thread.xyz` unless the user gives you another host.

Auth: `Authorization: Bearer {THREAD_API_KEY}` on every call. The key alone resolves its
grant and its account. You never pass an account id unless one key serves multiple
accounts (then add `?accountId=...`).

CLI equivalent for every call: `npx @jointhread/mail <me | new | peek [n] | read <id> |
attachment <id> <aid> [file]>`, with `--json` for the raw response.

---

## GET /me: what this key can read

Describes the grant behind the key. Discloses no mail. Does not count against the daily
limit. Call it first, and again whenever you are unsure what you hold.

Response `200`:

```jsonc
{
  "mailbox": "you@example.com",
  "provider": "gmail",                   // "gmail" | "outlook" | "imap"
  "keyName": "claude-code",              // the name the user gave this key
  "categories": ["financial", "travel"], // catalog slugs, or the string "all"
  "narrowedBy": [],                      // ["advanced"] when a sender/subject/custom rule narrows the grant further
  "bodyGrade": "masked",                 // "subject_only" | "masked" | "full"
  "attachments": true,
  "dailyLimit": null,                    // allowed calls per UTC day, or null for no limit
  "expiresAt": "2027-02-17T00:00:00.000Z", // or null
  "calendar": [],                        // granted calendar scopes, if any
  "feed": {
    "endpoint": "/emails",
    "singleEmail": "/emails/{id}",
    "attachment": "/emails/{id}/attachments/{attachmentId}",
    "firstCall": "GET /emails returns the 50 most recent in-scope emails on the first call ..."
  }
}
```

`403` with `FORBIDDEN`: the grant has expired or is outside its allowed schedule. The feed
would answer the same way.

---

## GET /emails: your mail feed

**Bare call** returns everything new since your last bare call, newest first, and
**advances your position** (consumes). The first bare call after a mailbox is connected
returns the 50 most recent in-scope emails, read or unread. **With any of `limit` /
`since` / `cursor`** it is a **peek** that never moves your position.

Query parameters (all optional):

| Param       | Type    | Meaning |
|-------------|---------|---------|
| `limit`     | integer 1 to 50 | Peek: the last N in-scope emails. |
| `since`     | string (ISO 8601) | Peek: in-scope mail Thread ingested after this time. |
| `cursor`    | string  | Peek: continue an earlier peek downward. |
| `accountId` | string  | Only when one key serves multiple accounts. |

Response `200`:

```jsonc
{
  "emails": [ /* email objects, see below */ ],
  "cursor": "..."   // present on a PEEK when the window may go deeper; null otherwise
}
```

A peek with `?limit=20` returns at most 20 and may return fewer when scope filtering
thins the window. Page with `cursor` until `emails` is empty if you need everything.

---

## GET /emails/{id}: one email by id

`id` is the provider's own message id (from any feed object).

Response `200`: a single email object (same shape as feed items, text included at the
granted grade).
`404`: the email does not exist **or** is outside your scope; indistinguishable by design.

---

## GET /emails/{id}/attachments/{attachmentId}: download bytes

`attachmentId` comes from an email object's `attachments[].id`.

Response `200`: the **raw attachment bytes** (binary), with `Content-Type` and
`Content-Disposition: attachment; filename="..."`.
`404`: the attachment does not exist **or** the grant does not cover attachments or this
email; indistinguishable by design.
`501`: attachment download is not available for this mailbox's provider yet.

---

## The email object

```jsonc
{
  "id": "1a1c6f2da569fa4d",          // provider message id, stable; use it for dedup and memory
  "from": "sender@example.com",      // nullable
  "to": ["you@example.com"],
  "subject": "Invoice INV-2041 due Dec 1",
  "receivedAt": "2026-11-19T09:30:00.000Z",  // nullable

  "labels": ["financial", "invoice"],    // THREAD's labels: what the grant matches on
  "mailboxLabels": ["INBOX", "UNREAD"],  // the MAILBOX's own labels, verbatim
  "isUnread": true,
  "threadId": "1a1c6f2da569fa4d",        // provider conversation id ("" on IMAP)

  // The text. See the three states below.
  "body": "Hi, your invoice for card [redacted-card] is attached. Pay by Dec 1.",
  "masked": true,                        // true: personal details redacted; false: raw
  "bodyGrade": "masked",                 // "subject_only" | "masked" | "full"

  // Present only when the user granted ATTACHMENTS:
  "attachments": [
    { "id": "0.1", "name": "INV-2041.pdf", "type": "application/pdf", "size": 84213 }
  ],

  // Present only when knowable. ABSENT means UNKNOWN, not false.
  "hasAttachment": true
}
```

### The three states of `body`

| `body` | `bodyGrade` | `masked` | Meaning |
|---|---|---|---|
| a string | `masked` | `true` | The text with emails, phone numbers, card numbers and ids replaced by `[redacted-...]` placeholders. |
| a string | `full` | `false` | The raw text, nothing removed. The user's private mail; handle it as such. |
| `null` | `subject_only` | absent | This key was never granted text. Work from `subject`, `from` and `labels`. Not an error. |
| absent | absent | absent | The key has text access, but this one email is outside its text scope (for example a grant that reads bodies only for invoices). |

Field notes:

- **`hasAttachment` absent means unknown.** Gmail only reveals attachments on a content
  fetch, which a subject-only grant does not pay for. Do not use it as a gate; check
  `attachments` or fetch the email by id.
- **`attachments` absent means not granted** (or this email is out of content scope), not
  "no files".
- **`labels` vs `mailboxLabels`:** the first is Thread's classification (what the grant is
  written against); the second is the provider's own labels, folders and flags (inbox
  state). Do not cross the streams.

---

## Errors

| Status | Code | Meaning | What to do |
|--------|------|---------|------------|
| 401 | `UNAUTHORIZED` / `NO_ACTIVE_GRANT` | Missing or invalid key, or grant revoked or expired | Stop; tell the user to check the key in their dashboard. |
| 403 | `FORBIDDEN` | Expired grant, or outside the allowed schedule | Do not retry as-is; tell the user. |
| 404 | `NOT_FOUND` | Out of scope OR nonexistent (indistinguishable) | Treat as "not available to me". |
| 429 | `RATE_LIMITED` | Daily limit reached | Back off until `00:00Z`. Retrying sooner just burns calls. |
| 503 | `PROVIDER_ERROR` / `INTERNAL` | Transient engine or provider error | Retry later. |

Out-of-scope mail on the **feed** is never an error; it is simply absent. The 404 appears
on **direct id fetches**.

---

## Rate limit: what a call costs

A grant *may* carry a daily limit. `GET /me` reports it as `dailyLimit` (null when there
is none).

What counts against it:

- **Any call that succeeds counts as one unit.** The counter is over *allowed calls*, not
  emails returned. A feed call that returns zero emails still counts.
- **Peeks count.** `?limit` / `?since` / `?cursor` spare your *position*, not your budget.
- `GET /emails/{id}` and attachment downloads each count as their own call.
- **`GET /me` is free.** So are denied calls (401, 403, 404).

The window is a **UTC calendar day**; the counter resets at `00:00Z`. On `429`, sleep to
the next UTC midnight.

Cadence: poll no faster than every **60s** (Thread's own mailbox poll interval; nothing
newer exists before then), and with a daily limit keep your interval at or above
`86400 / dailyLimit` seconds, leaving headroom for the id fetches and downloads you make
between polls.

There is **no push path for an API key.** Polling the feed is the only way to learn of new
mail.

---

## Doctrine you can rely on

- **Identity is the provider message id.** Stable forever; use it for dedup and memory.
- **Classify once, look up forever.** No model runs on repeat reads.
- **The feed never takes mail back.** Once an email was served to your key, it stays
  retrievable by id even if later re-labeling would narrow it.
- **50 on day one, then forward.** The first call returns the 50 most recent in-scope
  emails, read or unread; after that, only new mail.
- **Content is graded, and the response states the grade.** Read `bodyGrade` (and
  `masked`); never guess.
- **Nothing is stored.** Thread keeps three facts per email (id, sender, labels). Subject,
  text and attachments are fetched live from the mailbox when your key asks.
- **Pull, not push.** An API key has no callback; the feed is the only way in.
