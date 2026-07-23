---
name: angle-build-borrow-transaction
description: Use the Angle API to build unsigned leverage/deleverage transaction payloads for a borrowing vault, and simulate them before anything is signed. CONSEQUENTIAL — the output is meant to be broadcast on chain.
api: Angle API
base_url: https://api.angle.money
auth: none
consequential: true
operations:
  - GET /v1/vaultManagers
  - GET /v1/leverage
  - GET /v1/deleverage
  - GET /v1/simulate
  - GET /v1/simulate/multicall
source: openapi/angle-api-openapi.json
generated: '2026-07-19'
method: generated
---

# Build an Angle borrow transaction

Four Angle endpoints do not return data — they **build unsigned transaction
payloads** for the borrowing module. They are `GET` and unauthenticated, so they are
harmless to call, but their output exists to be signed by a wallet and broadcast.

> **Stop and require explicit human confirmation before any payload produced here is
> signed or submitted.** The API performs no authorization — anyone can build a
> payload for any address. The only control is the wallet signature at the end. An
> agent must never sign or broadcast autonomously.

## Steps

### 1. Resolve the VaultManager and swapper

```
GET https://api.angle.money/v1/vaultManagers?chainId=1&agToken=<stablecoin address>
```

Take `address`, `collateral`, `stablecoin` and `swapper` from the entry you intend to
act on. See `angle-monitor-borrow-vaults` for the full flow, including finding the
`vaultId` you want to modify.

### 2a. Build a leverage payload (add collateral / borrow more)

```
GET https://api.angle.money/v1/leverage
```

Required parameters:

| Parameter | Notes |
|---|---|
| `chainId` | EVM chain id |
| `userAddress` | The wallet that will sign |
| `vaultId` | ERC-721 tokenId of the vault |
| `agTokenAddress` | Stablecoin address |
| `agTokenToBorrow` | Amount to borrow |
| `collateralAddress` | Collateral token address |
| `collateralToAdd` | Amount of collateral to add |
| `slippage` | Tolerance for the embedded swap |

Optional: `agTokenToSwap`, `amountsBrought` (array), `vaultPermit`, `permits`
(array), `oneInchSlippage`.

`vaultPermit` and `permits` let the caller supply EIP-2612 signatures so approvals
ride along in the same transaction instead of needing a separate approve. Use
`collateralHasPermit` from step 1 to decide whether that is available.

### 2b. Build a deleverage payload (repay / withdraw)

```
GET https://api.angle.money/v1/deleverage
```

Required: `chainId`, `userAddress`, `vaultId`, `agTokenAddress`, `agTokenToRepay`,
`repayAll` (boolean), `collateralAddress`, `collateralToWithdrawToWallet`,
`collateralToSwap`, `outputTokenAddress`.

`repayAll` is required and is the highest-consequence flag in this API — it closes
out the debt rather than repaying a partial amount. Confirm it explicitly with the
user; never default it.

Note the asymmetry: `deleverage` has **no** `slippage` parameter, while `leverage`
requires one.

### 3. Simulate before signing — always

```
GET https://api.angle.money/v1/simulate?chainId=1&from=<addr>&to=<addr>&data=<calldata>&value=<wei>
```

All five parameters are required. Returns a string result.

For a batch:

```
GET https://api.angle.money/v1/simulate/multicall?calls=<array>
```

Returns `transfers` (the token movements the batch would cause) and `url` (a link to
the full simulation trace).

**Show the user `transfers` and the trace `url` before asking for confirmation.**
This is the only preview of what the transaction actually does, and it is the
difference between an informed approval and a blind one.

## Failure modes to guard against

- These are `GET` requests carrying large, sensitive inputs **on the query string**.
  Addresses, amounts and calldata will land in browser history, proxy logs and
  server access logs. Prefer a server-side call over a browser call.
- A `400` here may be either your bad input **or** an upstream failure — Angle
  returns both as `400`. Read the body: `{"details":{...}}` is validation,
  `{"error":"..."}` is usually upstream. Do not auto-retry a validation error with
  the same parameters.
- `503 upstream connect error` is common during the protocol wind-down. A failed
  simulate is a **hard stop** — never proceed to signing without a successful
  simulation.
- The API neither knows nor checks the current on-chain state at signing time. Re-read
  the vault (`GET /v1/vaults`) and re-simulate if any meaningful time has passed
  between building the payload and signing it.

## Wind-down warning

Angle has announced the end of protocol operations, with redemption open until
**2027-03-01**. Opening *new* leveraged positions against a sunsetting protocol is
rarely what a user wants. Surface the wind-down before building a leverage payload,
and prefer the deleverage/redemption path. See `lifecycle/angle-lifecycle.yml`.
