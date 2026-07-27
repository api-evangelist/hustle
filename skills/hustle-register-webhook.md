---
name: Register and verify a Hustle webhook
description: Authenticate to the Hustle Public API, register a webhook for consent or message-status events, and send a mock event to verify the receiver.
api: openapi/hustle-openapi-original.json
operations:
- POST /oauth/token
- POST /webhook_registrations
- POST /webhook_registrations/mock_event
- GET /webhook_registrations
---

# Register and verify a Hustle webhook

Use this to subscribe an endpoint to Hustle events and confirm it works.

## Steps

1. **Mint an access token** — `POST /oauth/token` with the client-credentials body; use the returned `access_token` as `Authorization: Bearer <token>`.

2. **Register the webhook** — `POST /webhook_registrations` with a `targetUrl` (https) and a `config` object. Choose the event via `config.type`:
   - `consentUpdate-v1` — lead opt-in / opt-out changes (toggle `includeOptIn` / `includeOptOut`).
   - `messageStatus-v1` — message delivery status changes (delivered, failed).
   Set `config.scope.type` to `organizations` (with specific org ids) or `account`. Store the returned registration `id`.

3. **Verify signing** — the response `secretForHMAC` indicates whether an HMAC signing secret is active. The secret is never returned; if you did not set/keep it, reset it. Verify the HMAC signature on inbound events.

4. **Send a mock event** — `POST /webhook_registrations/mock_event` to make Hustle POST a test event to your `targetUrl`, and confirm your receiver validates and processes it.

5. **List/manage** — `GET /webhook_registrations` to review; `PUT /webhook_registrations/{id}` to update; `DELETE /webhook_registrations/{id}` to remove.

## Rules
- Only accept events whose HMAC signature validates against your stored secret.
- Re-authenticate on `401`; inspect `hustleErrorCode` on `422`.
