---
name: Assess Knownwell portfolio health
description: Read the overall health of a client portfolio managed in Knownwell — the aggregate score, its distribution across risk bands, which accounts are trending in each direction, and which are currently at risk.
api: openapi/2x-knownwell-openapi.json
base_url: https://api.knownwell.com/ci/v1
auth: X-API-Key header
operations:
  - get_portfolio_health_v1_clients_portfolio_health_get
  - get_statistics_v1_clients_statistics_overview_get
  - get_trending_clients_v1_clients_trending_get
  - get_clients_by_risk_v1_clients_by_risk__risk_level__get
  - list_portfolios_v1_portfolios_get
  - get_portfolio_v1_portfolios__portfolio_id__get
---

# Assess Knownwell portfolio health

Use this to answer "how is the book doing?" before drilling into any single account.

## Before you start

- Send `X-API-Key: <key>` on every request. Without it you get `401` with
  `{"detail":"API key is required. Provide it in the X-API-Key header."}`.
- The key selects the tenant. `customerId` comes back on every response; you never send it.
- Everything here is a `GET`. Nothing in this skill changes state.
- Budget your calls: 100 requests/minute, 5,000/hour. Watch `X-RateLimit-Remaining`.

## Steps

1. **Get the headline number.** Call `get_portfolio_health_v1_clients_portfolio_health_get`.
   It returns `score`, a `breakdown`, and a `distribution` across risk bands. Pass
   `include_archived=false` (the default) unless you specifically want dead accounts counted.

2. **Get the counts.** Call `get_statistics_v1_clients_statistics_overview_get` for the
   overview totals. Use this to state how many accounts the health score is computed over —
   a score means little without its denominator.

3. **Find the movers.** Call `get_trending_clients_v1_clients_trending_get` twice, once with
   `direction=declining` and once with `direction=improving`. Cap with `limit`. Declining
   accounts are the ones worth a human's attention this week.

4. **Find the at-risk accounts.** Call
   `get_clients_by_risk_v1_clients_by_risk__risk_level__get` with `risk_level=high_risk`.
   Valid values are `high_risk`, `medium_risk`, `low_risk`, `on_track` — anything else
   returns `422`. Paginate with `limit` and `offset`.

5. **Segment if asked.** If the question is about a named book rather than the whole tenant,
   call `list_portfolios_v1_portfolios_get` to find the portfolio id, then
   `get_portfolio_v1_portfolios__portfolio_id__get` with `include_clients=true` — the detail
   response embeds that portfolio's own `health` block, its `clients` and its `users`.

## Reading the results honestly

- `hasInsufficientData: true` on a client means the score is not trustworthy for that
  account. Say so rather than reporting the number as fact.
- `scoreSource` tells you where the score came from. Do not present a score as observed
  behaviour if its source says otherwise.
- `scoreChanges` carries `7day` and `30day` deltas, each with `change`, `percentage` and
  `previousScore`. A one-week move on a noisy account is not a trend.
- `chr2CutoverDate` on the health response marks a scoring-methodology cutover. Comparisons
  across that date are not like-for-like.

## Failure handling

- `401` — missing or invalid key.
- `403` — key valid but not permitted for the resource.
- `422` — a parameter failed validation; the body is FastAPI's
  `{"detail":[{"loc":[...],"msg":"...","type":"..."}]}`. Read `loc` to find the bad param.
- `429` — you exceeded a window. Back off until `X-RateLimit-Reset`; there is no
  `Retry-After` header.
