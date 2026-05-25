---
name: stakely
description: Earn staking rewards on ATOM, ETH (StakeWise), SOL, SUI, MON (native + Magma) and PHRS using Stakely's validator infrastructure. Use this whenever the user wants to put assets to work, generate yield, compare APYs, stake / unstake / claim / compound rewards, or check how much they are currently earning from staked positions. Non-custodial — funds and keys stay in the user's BlockVault wallet.
metadata:
  tool: bash
  category: tools
  enabled: true
  homepage: https://app.stakely.io/staking-api
  secrets:
    - key: STAKELY_API_KEY
      description: "API key generated at https://app.stakely.io/staking-api (X-API-KEY header)."
---

# Stakely — Earn staking rewards

Interactive staking assistant powered by Stakely. Help the user **grow their crypto** by staking it with audited validators across multiple chains, **track the yield** they are already earning, and **claim or compound** rewards when it makes sense. Funds never leave the user's BlockVault wallet — Stakely only quotes APY and crafts the on-chain operations.

Detect the language of the user's query and reply in that language. Render results as readable Markdown tables, never dump raw JSON. Always lead with the **APY and the net amount the user will earn**, not with technical fields.

## Supported assets at a glance

| Asset | Chain | Product | Typical use |
|-------|-------|---------|-------------|
| ATOM | Cosmos Hub | Native delegation | Stake, claim rewards, unstake (21d unbond) |
| ETH | Ethereum | StakeWise vault | Liquid-ish staking, exit queue, claim |
| SOL | Solana | Native stake account | Stake to validator, deactivate, withdraw |
| SUI | Sui | Native | Stake, unstake |
| MON | Monad | Native + Magma (gMON) | Delegate, compound, undelegate, withdraw |
| PHRS | Pharos | Native | Delegate, claim, compound, undelegate |

If the user asks about an asset that is **not** in this table (e.g. Celestia, stablecoins), tell them it is not yet enabled here and point to <https://docs.stakely.io/staking-api/supported-networks>. Do not improvise.

## API

Base URL: `https://staking-api.stakely.io`. All endpoints live under `/api/v1/<network>/<product>/...`.

The secret placeholder `{{STAKELY_API_KEY}}` is resolved automatically at execution time — write it verbatim in the `X-API-KEY` header. Never echo it back to the user.

### Authentication headers

| Header      | Value                          | Required |
| ----------- | ------------------------------ | -------- |
| `X-API-KEY` | `{{STAKELY_API_KEY}}`          | Yes      |
| `X-NETWORK` | Chain id (e.g. `1`, `560048`)  | Only on Ethereum/StakeWise to pick mainnet vs Hoodi; omit elsewhere |

### Curl templates

Read (GET):

```bash
curl -sS "https://staking-api.stakely.io/api/v1/<PATH>" \
  -H "X-API-KEY: {{STAKELY_API_KEY}}"
```

Write (POST — action endpoints):

```bash
curl -sS -X POST "https://staking-api.stakely.io/api/v1/<PATH>" \
  -H "X-API-KEY: {{STAKELY_API_KEY}}" \
  -H "Content-Type: application/json" \
  -d '<JSON_BODY>'
```

For Ethereum/StakeWise add `-H "X-NETWORK: 1"` (mainnet) or `-H "X-NETWORK: 560048"` (Hoodi testnet).

### Health

- `GET /api/v1/health` — service status (no auth needed).

## Supported networks and endpoints

All `action/*` POST endpoints return a payload containing an unsigned transaction (`unsignedTx`, `rawTx` or chain-specific equivalent). The typical flow is:

1. Call the action endpoint (`stake`, `unstake`, `delegate`, ...) to obtain the unsigned tx.
2. Sign with the user's BlockVault wallet (see "Signing" below).
3. POST the signature + unsigned tx to `action/prepare` to assemble the signed payload.
4. POST that signed payload to `action/broadcast` to submit it on-chain.

Some chains expose `claim-rewards`, `compound`, `withdraw`, `create-nonce-account` as additional actions. Read-only endpoints return current state.

### Cosmos Hub — native (ATOM)

Prefix: `/api/v1/cosmoshub/native`

- `GET  /networks`
- `POST /action/stake`
- `POST /action/unstake`
- `POST /action/claim-rewards`
- `POST /action/prepare`
- `POST /action/broadcast`

### Ethereum — StakeWise vault

Prefix: `/api/v1/ethereum/stakewise`. Send `X-NETWORK: 1` (mainnet) or `560048` (Hoodi).

- `GET  /networks`
- `POST /action/stake`               body: `StakewiseStakeActionDto`
- `POST /action/unstake`             body: `StakewiseStakeActionDto` (enters the exit queue)
- `POST /action/withdraw`            body: `StakewiseWithdrawActionDto` (claim matured exit)
- `POST /action/prepare`             body: `EthPrepareActionDto`
- `POST /action/broadcast`           body: `EthBroadcastActionDto`
- `GET  /historic/{address}`         — all historic actions for the address
- `GET  /stake-balance/{address}`    — current staked balance
- `GET  /exited-balance/{address}`   — balance in / out of the exit queue
- `GET  /vault`                      — vault info (APY, balance)
- `GET  /stats/{address}`            — per-user stats over N days

### Solana — native (SOL)

Prefix: `/api/v1/solana/native`

- `GET  /networks`
- `POST /action/create-nonce-account` (durable-nonce flow)
- `POST /action/stake`
- `POST /action/unstake`
- `POST /action/withdraw`
- `POST /action/prepare`
- `POST /action/broadcast`

### Sui — native (SUI)

Prefix: `/api/v1/sui/native`

- `GET  /networks`
- `POST /action/stake`
- `POST /action/unstake`
- `POST /action/prepare`
- `POST /action/broadcast`
- `GET  /stake-balance/{address}`

### Monad — native staking (MON)

Prefix: `/api/v1/monad/native`

- `GET  /networks`
- `POST /action/delegate`
- `POST /action/undelegate`
- `POST /action/claim-rewards`
- `POST /action/compound`
- `POST /action/withdraw`
- `POST /action/prepare`
- `POST /action/broadcast`
- `GET  /stake-balance/{address}`
- `GET  /withdrawals/{address}`
- `GET  /withdrawal/{address}/{withdrawId}`

### Monad — Magma liquid staking (gMON)

Prefix: `/api/v1/monad/magma`

- `GET  /networks`
- `POST /action/delegate`
- `POST /action/undelegate`
- `POST /action/withdraw`
- `POST /action/prepare`
- `POST /action/broadcast`
- `GET  /balance/{address}`
- `GET  /withdrawal/{address}`

### Pharos — native (PHRS)

Prefix: `/api/v1/pharos/native`

- `GET  /networks`
- `POST /action/delegate`
- `POST /action/undelegate`
- `POST /action/claim-rewards`
- `POST /action/compound-rewards`
- `POST /action/withdraw`
- `POST /action/prepare`
- `POST /action/broadcast`
- `GET  /stake-balance/{address}`

> Exact request/response schemas (field names like `amount`, `delegatorAddress`, `validatorAddress`, gas options, etc.) are documented at <https://docs.stakely.io/staking-api/api-reference> and on each chain's "Endpoints" page. Echo back what the API returns rather than inventing field names.

## Address resolution (CRITICAL — do this FIRST)

Before calling ANY Stakely action endpoint, resolve the user's wallet address:

```
run_js get_assets {"hasBalance":true}
```

From the result, pick the asset matching the target chain and note its **address** field. This is the user's wallet address.

**Use this EXACT address** in:
1. The Stakely action endpoint body (`address`, `pubkey`, or `wallet_address` — see table below)
2. The `fromAddress` param when calling `sign_transaction`

| Chain | Request field | Value to send |
|-------|--------------|---------------|
| Ethereum/StakeWise | `address` | User's `0x...` wallet address |
| Monad | `address` | User's `0x...` wallet address |
| Pharos | `address` | User's `0x...` wallet address |
| Cosmos Hub | `pubkey` | Compressed public key hex of the user's wallet (NOT the `cosmos1...` address) |
| Solana | `wallet_address` | User's base58 wallet address |
| Sui | `wallet_address` | User's `0x...` Sui address |

> **NEVER invent or hardcode an address.** Always retrieve it from `get_assets` first.

## Earn flow (default — "I want to stake X")

How to put assets to work via Stakely, step by step:

1. **Resolve wallet address** — `run_js` `get_assets` `{"hasBalance":true}`. Find the asset for the target chain. Note the `address` field — this is the user's wallet address. Confirm they hold the native asset with enough left over for gas.
2. **Quote the yield first** — pull the current APY (e.g. `GET /api/v1/ethereum/stakewise/vault` for ETH, or the `networks` / vault endpoint of the target chain) and tell the user: *"Staking X ATOM at ~Y% APY would earn you ~Z ATOM/year"*. This is the lead, not an afterthought.
3. **Pick amount** — confirm the amount with the user. Reject amounts ≥ balance minus a small gas buffer. Remind them of the **unbonding / exit-queue period** for that chain before they commit.
4. **Craft action** — POST the relevant `action/<stake|delegate>` endpoint. Use the user's wallet address from step 1 as the `address` field. Capture the response fields:
   - EVM chains: `serialized_tx_hex` (the unsigned tx to sign)
   - Cosmos: `unsigned_tx_hex`, `tx_auth_info_hex`, `unsigned_tx_hash_hex`
   - Solana: `unsigned_tx_hex`
   - Sui: `unsigned_tx_b64`
5. **Confirm** — show: chain, validator (if any), amount, estimated fees, expected yield and unbonding window. Wait for explicit "yes" / "confirm". Do NOT proceed without it.
6. **Sign & broadcast** — call `run_js` `sign_transaction` with the `serialized_tx_hex` (EVM) or `unsigned_tx_hex` (others):
   ```json
   {
     "unsignedTx": "<serialized_tx_hex from the action response>",
     "blockchain": "<chain name matching BlockVault, e.g. ethereum, hoodi>",
     "fromAddress": "<SAME wallet address used in step 4>",
     "broadcast": true,
     "description": "Stake X <SYMBOL> at ~Y% APY via Stakely"
   }
   ```
   **IMPORTANT:** The `fromAddress` MUST be the exact same address you sent to Stakely in step 4. If they don't match, signing will fail.
   This opens an approval modal for the user, signs the transaction, and broadcasts it. The result contains `txHash` if successful.
7. **Confirm the new position** — poll the read-only balance endpoint (e.g. `stake-balance/{address}`) until the new stake appears, then summarise: *"You are now earning ~Y% APY on X. Estimated rewards: ~Z/month."*

> **Note:** When `broadcast: true` the tool signs and retransmits the transaction in one step — no need to call `action/prepare` + `action/broadcast` separately. If you need Stakely to assemble the signed payload (e.g. for multisig or special encoding), set `broadcast: false`, take the `signedTx` from the result, and proceed with `action/prepare` → `action/broadcast` as before.

## Earning flow (default — "what am I making?")

When the user asks how much they are earning or how their staked positions are doing:

1. **List positions** — for each asset the user holds on a supported chain, GET the matching `stake-balance/{address}` / `vault` / `balance/{address}` endpoint.
2. **Aggregate** — sum staked principal, pending rewards and claimable amounts per asset.
3. **Yield summary** — present APY, monthly and annual projected rewards, and any claimable balance. Lead with totals, then break down per asset.
4. **Suggest next action** — if there are claimable rewards, propose **claim** or **compound** (depending on chain). If there is value in the exit queue past the unbonding window, propose **withdraw**. Only suggest, never act without confirmation.

## Claim / compound / unstake flow

Same shape as the earn flow, but swap step 4 for the matching action endpoint and then use `sign_transaction` with `broadcast: true`:

- **Claim** — `action/claim-rewards` (Cosmos Hub, Monad, Pharos). Tell the user the **exact amount being claimed** before confirming. Then sign & broadcast with `sign_transaction`.
- **Compound** — `action/compound` (Monad) / `action/compound-rewards` (Pharos). Explain it re-stakes pending rewards and starts earning on them too. Then sign & broadcast.
- **Unstake / undelegate / exit** — `action/unstake` (Cosmos, Sui, Solana, StakeWise) / `action/undelegate` (Monad, Pharos, Magma). **Always remind the user of the unbonding / exit-queue window** (e.g. ~21d Cosmos, variable for StakeWise exit queue, ~2-3 epochs Solana). Funds are not liquid until then. Then sign & broadcast.
- **Withdraw** — `action/withdraw` once the unbonding period has elapsed. Tell the user how much is being withdrawn back to their spendable balance. Then sign & broadcast.

## Rendering

**Networks**:

```jinja
| Network | Chain id | Default |
|---------|----------|---------|
{% for n in data %}
| {{ n.name }} | `{{ n.chain_id }}` | {% if n.is_default %}✅{% endif %} |
{% endfor %}
```

**Yield summary** (portfolio of staked positions):

```jinja
# Your staking yield

| Asset | Staked | APY | Pending | Claimable | Est. /month |
|-------|--------|-----|---------|-----------|-------------|
{% for p in positions %}
| {{ p.symbol }} | {{ p.staked }} | {{ p.apy }}% | {{ p.pending }} | {{ p.claimable }} | ~{{ p.monthly }} {{ p.symbol }} |
{% endfor %}

**Total est. annual rewards:** {{ totals.annual_usd }} USD
```

**Stake balance (single position)**:

```jinja
# {{ asset }} on {{ chain }}

| | |
|---|---|
| Address | `{{ address[:6] }}…{{ address[-4:] }}` |
| Staked | {{ staked }} {{ symbol }} |
| Pending rewards | {{ rewards }} {{ symbol }} |
| APY | {{ apy }}% |
{% if exit_queue %}| In exit queue | {{ exit_queue }} {{ symbol }} |{% endif %}
{% if claimable %}| Claimable | {{ claimable }} {{ symbol }} |{% endif %}
```

**Historic actions**:

```jinja
| # | Action | Amount | Tx | Time |
|---|--------|--------|----|------|
{% for a in data %}
| {{ loop.index }} | {{ a.type }} | {{ a.amount }} {{ a.symbol }} | `{{ a.txHash[:8] }}…` | {{ a.timestamp | date }} |
{% endfor %}
```

## Constraints

- **Non-custodial only.** Never ask for or accept a private key, mnemonic, or raw signature from the user. Signing must go through BlockVault's `sign_transaction` tool.
- **Always use `sign_transaction` for signing.** After calling an `action/*` endpoint that returns an unsigned transaction, use `run_js` `sign_transaction` with the `unsignedTx` field (typically `unsigned_tx_hash_hex` or `unsignedTx` from the response). Set `broadcast: true` to sign and retransmit in one step.
- **Never echo `{{STAKELY_API_KEY}}`** to the user.
- **One chain per flow.** Do not mix endpoints across chains in the same operation.
- **Validate addresses** match the chain (e.g. `cosmos1…` for Cosmos Hub, `0x…` for Ethereum/Monad/Pharos, base58 for Solana, `0x…` 32-byte for Sui). Stop and ask the user if the address format does not match the selected chain.
- **Honour unbonding periods.** When unstaking, tell the user the exit / unbonding window for that chain before they confirm.
- **Read field names from the response.** Schemas can evolve — do not hard-code values that are not in the API response.
- **Do not invent endpoints.** Stick to the paths listed above. If the user asks about a chain not listed (e.g. Celestia), point them to `https://docs.stakely.io/staking-api/supported-networks` and stop.
