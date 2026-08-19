---
name: Manage Knownwell API keys
description: Issue, list and revoke Knownwell API keys. These are the only write operations in the entire API and the only ones that can cause harm, so they carry stricter handling than the read surface.
api: openapi/2x-knownwell-openapi.json
base_url: https://api.knownwell.com/ci/v1
auth: bearer authorization header (separate from the data-plane X-API-Key)
operations:
  - create_api_key_v1_api_keys_post
  - list_api_keys_v1_api_keys_get
  - revoke_api_key_v1_api_keys__key_id__delete
---

# Manage Knownwell API keys

This is the administrative plane. Three of the API's 27 operations live here and they are
the only non-`GET` operations in the contract.

## Before you start

- These operations take an `authorization` **header parameter**, not the `X-API-Key` used by
  the data plane. A data key cannot mint another key.
- There is **no idempotency key** anywhere in this API. A retried `POST` mints a **second
  key**, and nothing in the contract lets you detect or collapse the duplicate. Treat key
  creation as non-retryable: if the response is lost, `list` before you retry.
- Never echo a created key into logs, a transcript, or a summary. It is returned exactly
  once.

## Steps

1. **See what exists first.** Call `list_api_keys_v1_api_keys_get`, optionally scoped with
   `customer_id`. Each `APIKeyResponse` carries `id`, `name`, `description`, `scope`,
   `status`, `created_at`, `expires_at` and `last_used_at`. The `api_key` field is `null`
   here — plaintext is never returned on a list.

2. **Check for an existing key before minting.** Match on `name` and `customer_id`. Most
   "we need a key" requests are actually "we lost the key", and the right answer is to revoke
   and reissue deliberately rather than to stack a third live credential.

3. **Create.** Call `create_api_key_v1_api_keys_post` with an `APIKeyCreate` body:
   `customer_id`, `name`, optional `description`, `scope`, and `expires_days`. Returns `201`.
   - `scope` accepts exactly one value: `read_only`. There is no write scope, so a data-plane
     key can never mutate anything.
   - Always set `expires_days`. The field is optional and an omitted expiry means a
     credential that outlives the reason it was created.
   - The `api_key` field on the `201` response is the only time the plaintext appears.
     Hand it to the requester through a secret channel, then discard it.

4. **Revoke.** Call `revoke_api_key_v1_api_keys__key_id__delete` with `key_id`. Returns
   `204` with no body. Confirm by re-listing and checking `status` — a `204` tells you the
   request was accepted, not what the resulting state is.

5. **Audit.** `last_used_at` is the field that matters for hygiene. A key with a `null` or
   long-stale `last_used_at` is a credential nobody needs and everybody is exposed to.

## Guardrails

- Do not create or revoke a key on your own initiative. Both are consequential and neither is
  reversible in the way a read is — a revoke breaks whatever integration was using it.
- Confirm the target `key_id` against a fresh `list` immediately before revoking. Ids are
  opaque strings with no prefix or checksum, so a transposition is not self-detecting.
- The API declares no `403` in the spec even though the docs document it. If you get a `403`
  on this plane, the authorization header is wrong — do not retry with the data key.

## Failure handling

- `422` — the create body failed validation; read `loc` in the FastAPI validation envelope.
- `429` — applies here too; do not retry a create on a `429` without listing first.
