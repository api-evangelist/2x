---
name: Track weekly client alignment in Knownwell
description: Pull the weekly alignment reading across the whole book or for one client, with its audit trail, and compare it against the Knownwell score to find accounts where the team's stated view and the measured signal disagree.
api: openapi/2x-knownwell-openapi.json
base_url: https://api.knownwell.com/ci/v1
auth: X-API-Key header
operations:
  - list_client_alignments_v1_clients_alignment_get
  - get_client_alignment_v1_clients__client_id__alignment_get
  - list_clients_v1_clients_get
---

# Track weekly client alignment in Knownwell

Alignment is the human-entered weekly read on an account. The Knownwell score is the
measured one. The interesting accounts are where they disagree.

## Before you start

- `X-API-Key: <key>` on every call; both operations are `GET`.
- Alignment is addressed by **week**, not by date range and not by id. Omit `week` to get the
  current week.

## Steps

1. **Pull the book for a week.** Call `list_client_alignments_v1_clients_alignment_get`,
   optionally with `week=<week>`. Paginate with `limit` and `offset`; `include_archived`
   defaults to excluding archived accounts. Each row is a `ClientAlignment` carrying
   `clientId`, `clientName`, `externalClientId`, `knownwellScore`, `scoreSource` and the
   `alignment` object itself.

2. **Read the audit trail.** The nested `alignment` object has `value`, `week`, `updatedAt`
   and `updatedBy`. An alignment with a stale `updatedAt` was not refreshed this week — treat
   it as missing data rather than as a current opinion.

3. **Find the disagreements.** In the same rows you already have both `knownwellScore` and
   the human `alignment.value`. Flag accounts where a healthy alignment sits on top of a
   falling score, or the reverse. That gap is the reason this endpoint exists.

4. **Drill into one account.** Call
   `get_client_alignment_v1_clients__client_id__alignment_get` with `client_id` and an
   optional `week` to inspect a single client's reading for a specific week.

5. **Join to a customer's own systems.** `externalClientId` is the join key back to the
   customer's CRM. Use it rather than matching on `clientName`, which is not guaranteed
   unique or stable.

## Reading the results honestly

- `alignment` may be `null`. That means nobody submitted a reading — it does not mean the
  account is unaligned. Report coverage (how many accounts have a reading this week) before
  reporting the distribution of values.
- `knownwellScore` may be `null` on an alignment row. Do not impute it from another endpoint
  without saying you did.
- `updatedBy` is a person. Do not surface it in a summary that will be read as performance
  data about that person without being asked to.

## Failure handling

- `422` — most often a malformed `week` value. Echo the `loc` from the validation body.
- There is no bulk history endpoint for alignment: one call per week. Sweeping a quarter is
  13 calls, which is fine against the 100/minute ceiling but should still be sequenced.
