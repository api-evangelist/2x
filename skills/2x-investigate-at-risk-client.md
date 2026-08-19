---
name: Investigate an at-risk client in Knownwell
description: Build the full picture of a single account whose Knownwell score is falling — its score history, open priorities, recent notes, key people, and revenue streams — so a human can decide what to do about it.
api: openapi/2x-knownwell-openapi.json
base_url: https://api.knownwell.com/ci/v1
auth: X-API-Key header
operations:
  - search_clients_v1_clients_search_get
  - list_clients_v1_clients_get
  - get_client_v1_clients__client_id__get
  - get_client_history_v1_clients__client_id__history_get
  - get_client_priorities_v1_clients__client_id__priorities_get
  - get_client_notes_v1_clients__client_id__notes_get
  - get_client_key_people_v1_clients__client_id__key_people_get
  - list_client_streams_v1_clients__client_id__streams_get
---

# Investigate an at-risk client in Knownwell

Use this after portfolio health has surfaced an account, or when someone names a client.

## Before you start

- `X-API-Key: <key>` on every call. All operations here are `GET`.
- You need a `client_id` before step 2. Ids are opaque strings with no documented prefix.

## Steps

1. **Resolve the account.** If you have a name and not an id, call
   `search_clients_v1_clients_search_get` with `query=<name>`; narrow with `fields` if the
   match is noisy. If you need to browse instead, use `list_clients_v1_clients_get` with
   `limit` (max 500) and `offset`.

2. **Get the account.** Call `get_client_v1_clients__client_id__get`. This detail response
   carries what the list form does not: `spotlightSummary` (the narrative read on the
   account) and `topics`. It also carries `metadata` — account owner, industry, domain,
   account status, `revenueTTM` and `revenueF12M` — which is how you size the account.

3. **Get the trajectory.** Call `get_client_history_v1_clients__client_id__history_get` with
   `months=<n>`. Pair it with `scoreChanges` from step 2 so you can distinguish a sharp
   recent drop from a long slow decline. These are different problems.

4. **Get the open work.** Call `get_client_priorities_v1_clients__client_id__priorities_get`.
   Filter with `status_filter` when you only want open items. This tells you whether anyone
   is already acting on the decline.

5. **Get the recent context.** Call `get_client_notes_v1_clients__client_id__notes_get`,
   paginating with `limit`/`offset`. Read the most recent notes first — they usually explain
   a score move faster than the score does.

6. **Get the relationships.** Call
   `get_client_key_people_v1_clients__client_id__key_people_get`. A score drop that coincides
   with a champion leaving is a relationship problem, not a delivery problem.

7. **Check the streams.** Call `list_client_streams_v1_clients__client_id__streams_get`. If
   one stream is dragging the account, `get_stream_v1_clients__client_id__streams__stream_id__get`
   gives that stream's own `insights`, `contextualClues` and `growthRecommendations`.

## Reading the results honestly

- Check `hasInsufficientData` before you report any score. If it is true, report the gap.
- `lastContactDate` far in the past is itself the finding — the score may be stale rather
  than the relationship bad.
- `spotlightSummary`, `insights` and `growthRecommendations` are model-generated narrative,
  not observed fact. Attribute them as Knownwell's read, not as ground truth.
- `archived: true` accounts are excluded from lists by default. If a caller expects an
  account and it is missing, retry with `include_archived=true` before concluding it is gone.

## Failure handling

- `422` on a path id usually means the id is malformed, not that the client is absent.
- `429` — this skill makes six or more calls per account. Sequence them and respect
  `X-RateLimit-Remaining` when sweeping several accounts in one run.
