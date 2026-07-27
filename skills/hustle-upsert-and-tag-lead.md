---
name: Upsert a Hustle lead and tag them
description: Authenticate to the Hustle Public API, upsert a lead by phone number into an organization, then apply a tag to that lead.
api: openapi/hustle-openapi-original.json
operations:
- POST /oauth/token
- POST /leads
- GET /tags
- POST /tags
- PATCH /leads
---

# Upsert a Hustle lead and tag them

Use this to add or update a contact in a Hustle organization and label them.

## Prerequisites
- A Hustle `client_id` (email format) and `client_secret`.
- The target `organizationId`.

## Steps

1. **Mint an access token** — `POST /oauth/token` (base `https://api.hustle.com/v3`) with body `{ "grant_type": "client_credentials", "client_id": "<id>", "client_secret": "<secret>" }`. Read `access_token` from the response and send it as `Authorization: Bearer <access_token>` on every following request.

2. **Upsert the lead** — `POST /leads` with `phoneNumber` (E.164, e.g. `+15551234567`) and `organizationId`. This is idempotent: the pair (organizationId, phoneNumber) identifies the lead, so a repeat call updates rather than duplicates. A `201` means the lead was created; a `200` means an existing lead was updated. Capture the returned lead `id`.

3. **Find or create the tag** — `GET /tags` (filter by `organizationId`) to see if the tag exists; if not, `POST /tags` to create it. Capture the tag `id`.

4. **Apply the tag** — `PATCH /leads` with an `ApplyTagOperation` referencing the lead and the tag id.

## Rules
- Always retry auth (`401`) by minting a fresh token; tokens expire per `expires_in`.
- On `422`, read `message` and `hustleErrorCode` (see errors/hustle-problem-types.yml).
- Never duplicate a lead — rely on the upsert natural key rather than creating variants.
