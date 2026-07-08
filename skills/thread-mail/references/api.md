# Thread mail API — reference

The complete contract for the agent-facing mail feed. Mirrors the gateway's
OpenAPI (`services/gateway/src/openapi.ts` in the Thread repo). If this file and
the live `GET {BASE}/openapi.json` disagree, the live spec wins — this is a
rendered copy.

Base URL (`{BASE}`): the Thread gateway. `http://localhost:3002` in dev; the host
the user gives you in production.

Auth: `Authorization: Bearer {THREAD_API_KEY}` on every call. The key alone
resolves its grant and its account — you never pass an account id unless one key
serves multiple accounts (then add `?accountId=…`).

---

## GET /emails — your mail feed

**Bare call** returns everything new since your last bare call, newest first, and
**advances your watermark** (consumes). **With any of `limit` / `since` / `cursor`**
it's a **peek** that never moves your watermark.

Query parameters (all optional):

| Param       | Type    | Meaning |
|-------------|---------|---------|
| `limit`     | integer 1–50 | Peek: the last N in-scope emails. |
| `since`     | string (ISO 8601) | Peek: in-scope mail Thread ingested after this time. |
| `cursor`    | string  | Peek: continue an earlier peek downward. |
| `accountId` | string  | Only when one key serves multiple accounts. |

Response `200`:

```jsonc
{
  "emails": [ /* email objects, see below */ ],
  "cursor": "…"   // present on a PEEK when the window may go deeper; null/absent otherwise
}
```

Note on peeks: `?limit=20` returns *at most* 20 and may return fewer when scope
filtering thins the window. If `cursor` is absent you may still not have reached
the end on a heavily-filtered window — page with `cursor` until `emails` is empty.

---

## GET /emails/{id} — one email by id

`id` is the provider's own message id (from any feed object).

Response `200`: a single email object (same shape as feed items).
`404`: the email does not exist **or** is outside your scope — indistinguishable
by design. If a read beats Thread's sync, the email is classified inline on this call.

---

## GET /emails/{id}/attachments/{attachmentId} — download bytes

`attachmentId` comes from an email object's `attachments[].id`.

Response `200`: the **raw attachment bytes** (binary), with `Content-Type` and
`Content-Disposition: attachment; filename="…"`.
`404`: the attachment doesn't exist **or** the grant doesn't cover attachments /
this email — indistinguishable by design.
`501`: attachment download isn't available for this mailbox's provider yet.

---

## The email object

```jsonc
{
  "id": "19f36c29307f49c4",       // provider message id — stable; use for dedup/memory
  "from": "sender@example.com",   // nullable
  "to": ["you@example.com"],
  "subject": "Invoice INV-2041 — $420.00 due July 15",
  "receivedAt": "2026-07-08T10:14:00Z",  // nullable

  "labels": ["financial", "invoice"],    // THREAD's labels — what the grant matches on
  "mailboxLabels": ["INBOX", "UNREAD"],  // the MAILBOX's own labels, verbatim
  "isUnread": true,
  "threadId": "…",                       // provider conversation id ("" on IMAP)

  // Present only when the user granted ATTACHMENTS:
  "attachments": [
    { "id": "att_1", "name": "invoice.pdf", "type": "application/pdf", "size": 84213 }
  ],

  // Present only when the user granted CONTENT. Body is ALWAYS masked.
  "body": "Hi — your invoice for $420.00 is attached. Reply to …@…",
  "masked": true,

  // Present only when knowable at metadata cost. ABSENT means UNKNOWN, not false.
  "hasAttachment": true
}
```

Field notes — the load-bearing subtleties:

- **`hasAttachment` absent ⇒ unknown.** Gmail only reveals attachments on a content
  fetch, which a metadata-only grant doesn't pay for. Thread omits the field rather
  than report a false `false`. Do not use it as a gate; check `attachments` or fetch
  the email by id.
- **`attachments` absent ⇒ not granted** (or this email is out of content-scope), not
  "no files."
- **`body` is always masked** (`masked: true`); there is no raw option, and raw text
  is never stored by Thread.
- **`labels` vs `mailboxLabels`:** the first is Thread's semantic classification (what
  the grant is written against); the second is the provider's own labels/folders/flags
  (inbox state). Don't cross the streams.

---

## Errors

| Status | Code | Meaning | What to do |
|--------|------|---------|------------|
| 401 | `UNAUTHORIZED` / `NO_ACTIVE_GRANT` | Missing/invalid key, or grant revoked/expired | Stop; tell the user to check the key or re-run Connect. |
| 403 | `FORBIDDEN` / `NO_ACTIVE_GRANT` | The grant doesn't cover this read | Don't retry as-is. |
| 404 | `NOT_FOUND` | Out of scope OR nonexistent (indistinguishable) | Treat as "not available to me." |
| 429 | rate limit | Daily cap reached | Back off; retry tomorrow / next window. |
| 503 | `PROVIDER_ERROR` | Transient engine/provider error | Retry later. |

Out-of-scope mail on the **feed** is never an error — it's simply absent. The 403/404
codes appear on **direct id fetches** and revoked/expired grants.

---

## Doctrine you can rely on

- **Identity = the provider message id.** Stable forever; use it for dedup and memory.
- **Classify once, look up forever.** No LLM runs on repeat reads.
- **The feed never takes mail back.** Once an email was served to your key, it stays
  retrievable by id even if later re-labeling would narrow it.
- **Forward-only.** Thread watches mail from connection time on; it never backfills
  old mail.
- **Raw content never leaves.** Bodies are masked or absent; there is no raw option.
