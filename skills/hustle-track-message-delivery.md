---
name: Track Hustle message delivery status
description: Authenticate to the Hustle Public API, subscribe to messageStatus-v1 webhooks, and poll GET /messages/{id} to resolve a message to a terminal delivery status without tripping the 25 req/s rate limit.
api: openapi/hustle-messages-api-openapi.yml
operations:
- POST /oauth/token
- POST /webhook_registrations
- GET /messages/{id}
- GET /webhook_registrations
---

# Track Hustle message delivery status

Use this when you need to know whether an outbound Hustle SMS/MMS actually reached the
recipient. Messages are **created in the Hustle product, not through the Public API** — this
skill only reads their status.

## Steps

1. **Mint an access token** — `POST /oauth/token` with
   `{"grant_type":"client_credentials","client_id":…,"client_secret":…}`. Send the payload as
   JSON (`json=`, not `data=`) — form encoding flattens Hustle's nested objects and produces
   `[property]: Expected object, received array`. The returned `access_token` is a JWT valid for
   `expires_in` = 7200 seconds; use it as `Authorization: Bearer <token>`.

2. **Prefer push over poll** — `POST /webhook_registrations` with an https `targetUrl` and
   `config.type: messageStatus-v1`, scoped with `config.scope.type` of `organizations` or
   `account`. Hustle then pushes status transitions to you, HMAC-signed with the
   per-registration secret. Confirm the registration with `GET /webhook_registrations`.

3. **Read a single message** — `GET /messages/{id}` returns a `MessageStatus`:
   - `id`, `threadId` — the message and its conversation. `threadId` is emitted but has **no
     resolvable endpoint** in the Public API; treat it as an opaque grouping key.
   - `type` — `SMS` or `MMS`.
   - `status` — an **open string**, not an enum. Non-terminal: `pending_lookup` (documented
     under a 1-hour SLA) and `queued`. Terminal: `sent`, `failed`, `undelivered`, plus carrier
     statuses. Never hard-code the terminal set; treat "not `pending_lookup` and not `queued`"
     as terminal and log anything unrecognized.
   - `errorCode` — present only on failure (e.g. `not_textable`, `lookup_timeout`, or a carrier
     failure). Absent on success; do not branch on its presence alone.
   - `createdAt` — ISO-8601.

4. **Back off while polling** — the API allows **25 requests per second per account across all
   resource endpoints**, and publishes **no** `RateLimit-*` / `Retry-After` headers and no `429`
   in the spec. You cannot read remaining quota, so throttle yourself: poll a `pending_lookup`
   message no more than once per minute, cap the wait at the documented 1-hour lookup SLA, and
   stop polling the moment the status is terminal.

## Rules
- On `401`, mint a **new** token and retry once. Do not retry the token request in a loop:
  10 failed token attempts blocks your IP for that account, and 100 in 24 hours blocks the IP
  outright.
- On `422`, read `message` and the `hustleErrorCode` field from the shared `ErrorResponse`
  envelope (`{code, message, hustleErrorCode?}`) — Hustle support triages on that code. Errors
  are plain `application/json`, **not** RFC 9457 `problem+json`.
- A `404` on `GET /messages/{id}` means the id is unknown to the authenticated account, not
  that the message failed.
- This operation is read-only and safe to retry.
