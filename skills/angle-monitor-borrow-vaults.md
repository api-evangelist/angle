---
name: angle-monitor-borrow-vaults
description: Inspect Angle borrowing-module vaults for a wallet — which VaultManagers exist on a chain, what collateral and debt each vault holds, and the wallet's token balances and allowances.
api: Angle API
base_url: https://api.angle.money
auth: none
operations:
  - GET /v1/activeChains
  - GET /v1/vaultManagers
  - GET /v1/vaults
  - GET /v1/balances
  - GET /v1/allowances
source: openapi/angle-api-openapi.json
generated: '2026-07-19'
method: generated
---

# Monitor Angle borrowing vaults

Angle's borrowing module lets a user deposit collateral into a **vault** and mint an
Angle stablecoin against it. Each vault is an ERC-721 token owned by a
**VaultManager** contract, which pairs exactly one collateral token with one
stablecoin. This skill reads those positions.

## Before you start

- Public and unauthenticated — send no credential.
- All operations are `GET`, so all are safe and retryable.
- **There is no user account.** A "user" is just an EVM wallet address you pass as a
  query parameter. Angle stores no user record and issues no user id.
- Identifiers are on-chain addresses, not Angle-issued ids.

## Steps

### 1. Pick a live chain

```
GET https://api.angle.money/v1/activeChains
```

See `angle-track-savings-yield` for the response shape.

### 2. List the VaultManagers on that chain

```
GET https://api.angle.money/v1/vaultManagers?chainId=1&agToken=0x1a7e4e63778B4f12a199C062f3eFdD288afCBce8
```

Required: `chainId`, `agToken`. Optional: `user`, `blockNumber` (for a historical
snapshot).

`agToken` is the stablecoin address — EURA is
`0x1a7e4e63778B4f12a199C062f3eFdD288afCBce8`.

The response is a **map keyed by address**, not an array. Iterate its values. Each
carries `address`, `symbol`, `collateral`, `stablecoin`, `swapper`,
`collateralHasPermit` and `collateralPermitVersion`.

Keep `collateralHasPermit` — it tells you whether a later approval can be done with
an EIP-2612 `permit` signature instead of a separate on-chain approve transaction.

### 3. Read the wallet's vaults

```
GET https://api.angle.money/v1/vaults?chainId=1&user=0xa116f421ff82A9704428259fd8CC63347127B777&agToken=0x1a7e4e63778B4f12a199C062f3eFdD288afCBce8
```

All three of `chainId`, `user` and `agToken` are **required**.

Each vault carries `address` (the owning VaultManager), `id` (the ERC-721 tokenId),
`collateral`, `stablecoin`, `rate` (the oracle price), `collateralAmount` and `debt`.

To judge health, compare `collateralAmount * rate` against `debt`. Note that these
are **numbers** here, while the savings schema returns strings — the API is not
consistent about amount encoding, so normalise per schema rather than globally.

Join back to step 2 on `address` to recover the VaultManager's `symbol` and swapper.

### 4. Read balances and allowances

```
GET https://api.angle.money/v1/balances?chainId=1&user=<address>
GET https://api.angle.money/v1/allowances?chainId=1&user=<address>&spenders=<address>,<address>
```

Both require `chainId` and `user`. `allowances` additionally requires `spenders`, an
**array** parameter. Both accept `additionalTokenAddresses` to include tokens outside
Angle's default list.

`balances` returns `symbol`, `balance` (a **string** in base units) and `decimals`
per token. Divide by `10 ** decimals` using big-integer arithmetic.

Check allowances before assuming a wallet can transact — an insufficient allowance is
the most common reason a subsequent on-chain call fails.

## Notes and caveats

- No pagination on these endpoints. They return everything for the wallet.
- No rate limit is published and no `RateLimit` / `Retry-After` headers are sent, so
  there is no quota signal. Be conservative.
- Parts of the API return `503` intermittently during the protocol wind-down. Retry.
- `blockNumber` on `/v1/vaultManagers` is the only historical-snapshot control in this
  flow; the vault endpoints themselves are current-state only.

See `conventions/angle-conventions.yml` and `data-model/angle-data-model.yml`.
