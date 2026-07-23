---
name: angle-track-savings-yield
description: Read current and historical yield for Angle's staked stablecoins (stEUR for EURA, stUSD for USDA) across the chains Angle supports.
api: Angle API
base_url: https://api.angle.money
auth: none
operations:
  - GET /v1/activeChains
  - GET /v2/savings
  - GET /v2/historical/savings
  - GET /v1/savings
source: openapi/angle-api-openapi.json
generated: '2026-07-19'
method: generated
---

# Track Angle savings yield

Angle stablecoin holders can stake EURA for **stEUR** and USDA for **stUSD** to earn
a native yield. This skill reads the current and historical state of those savings
contracts.

## Before you start

- The API is **public and unauthenticated** — send no credential.
- Every operation is a `GET`. There is nothing to write here.
- The API has **no `operationId`s**, so operations are named by method + path.
- Angle is **winding down** (redemption deadline 2027-03-01) and parts of the API
  return `503 upstream connect error` intermittently. Treat every call as
  failure-prone and retry — `GET` is safe and idempotent, so retrying is always OK.

## Steps

### 1. Find the chains that are live

```
GET https://api.angle.money/v1/activeChains
```

Takes no parameters. Returns supported EVM chain ids, for example
`{"borrowModule":["1","10","137","42161","43114"]}` — Ethereum (1), Optimism (10),
Polygon (137), Arbitrum (42161), Avalanche (43114).

Use this instead of hard-coding a chain id. If it returns 503, retry before falling
back to `1` (Ethereum).

### 2. Read current savings state

```
GET https://api.angle.money/v2/savings?chainId=1
```

`chainId` is **required**. Omitting it returns `400` with a Joi validation envelope:

```json
{"details":{"details":[{"message":"\"chainId\" is required","path":["chainId"],"type":"any.required"}]}}
```

The v1 equivalent `GET /v1/savings?chainId=1` still exists and is documented as
returning the `stEUR` contract state. Prefer **v2** — it is the newer surface and the
only part of the API that documents an error response.

Fields on the savings schema: `chainId`, `totalAssets`, `totalSupply`, `paused`,
`lastUpdate`, `apr`.

- `apr` is the yield figure you want.
- `paused` is a boolean — **check it**. A paused savings contract makes `apr`
  meaningless.
- `totalAssets` and `totalSupply` are **strings** in base units, not numbers. Parse
  them as big integers and apply the token's decimals; do not use floating point.

### 3. Read the history

```
GET https://api.angle.money/v2/historical/savings?chainIds=1&startDate=...&endDate=...&first=100&skip=0
```

All parameters are optional. Note the shape differences from step 2:

- `chainIds` (plural) is a **comma-joined string** here, not the integer `chainId`
  used everywhere else in the API.
- `startDate` / `endDate` bound the range.
- `agToken` filters to one stablecoin — EURA is
  `0x1a7e4e63778B4f12a199C062f3eFdD288afCBce8`.
- `user` scopes to one wallet address.

### 4. Page through results

Pagination is offset-style with `first` (page size) and `skip` (offset), inherited
from the subgraph backing this endpoint.

The response is a **bare array with no envelope** — there is no total count, cursor
or next link. You cannot tell whether more records exist. Page until you get back
fewer rows than you asked for, then stop.

## Error handling

`/v2/historical/savings` is the only operation in the API that documents a 4xx. In
practice you will see three incompatible envelopes, so check for all of them:

| Shape | Meaning |
|---|---|
| `{"details":{...}}` | Missing or invalid parameter (Joi validation) |
| `{"error":"..."}` | Upstream fetch failure or unknown route |
| `{"error":"...","message":"..."}` | The declared `errorResponse` schema |

**Do not apply a normal retry policy based on status code alone.** Angle returns
upstream failures as `400`, not `5xx` — for example
`{"error":"❌ failed to fetch Angle prices"}`. A client that retries only on `5xx`
will wrongly give up on a transient failure. Match on the message string.

See `errors/angle-problem-types.yml` and `conventions/angle-conventions.yml`.
