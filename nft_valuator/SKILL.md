---
name: nft-valuator
description: Valuate Counterparty NFT assets (Rare Pepes, Spells of Genesis, Sarutobi, etc.). Use this when the user asks about the value, rarity, history, or details of a Counterparty token or NFT collection — NOT for buying from dispensers (use counterparty-dispensers instead).
metadata:
  tool: bash
  category: wallet
  enabled: true
  homepage: https://cp.blockvault.ai
---

# Counterparty NFT Valuator

Valuate and analyze Counterparty NFT assets by combining on-chain data (issuances, supply, dispensers, dispense history) with web research on market prices, rarity, and collection context.

## Instructions

Execute all steps silently. Do NOT output internal thoughts. No exceptions. Do not omit any steps.

- **Use `bash` for API calls.** Execute `curl` commands — never invent responses.
- Detect the user's language and reply in that language.
- Render results using the Jinja2 templates below — never dump raw JSON.
- Always include the asset card image when available.

## API endpoints

Base URL: `https://cp.blockvault.ai`

No authentication required. All list endpoints support `limit` (default 100, max 1000), `offset`, `order_by`, `order_dir` (ASC/DESC), plus any column as a filter parameter.

```bash
# Get asset issuance history (supply, divisibility, locked status, description)
curl -sS "https://cp.blockvault.ai/api/v1/issuances?asset=<ASSET_NAME>&order_by=block_index&order_dir=ASC&limit=50"

# Get the latest issuance (current state of the asset)
curl -sS "https://cp.blockvault.ai/api/v1/issuances?asset=<ASSET_NAME>&order_by=block_index&order_dir=DESC&limit=1"

# Get all holders (balances) of an asset
curl -sS "https://cp.blockvault.ai/api/v1/balances?asset=<ASSET_NAME>&limit=100&order_by=quantity&order_dir=DESC"

# Get open dispensers for the asset (current sell offers)
curl -sS "https://cp.blockvault.ai/api/v1/dispensers?asset=<ASSET_NAME>&status=0&limit=20&order_by=satoshirate&order_dir=ASC"

# Get closed dispensers (past sales via dispensers)
curl -sS "https://cp.blockvault.ai/api/v1/dispensers?asset=<ASSET_NAME>&status=10&limit=20&order_by=block_index&order_dir=DESC"

# Get dispense history for the asset (actual sales)
curl -sS "https://cp.blockvault.ai/api/v1/dispenses?asset=<ASSET_NAME>&limit=50&order_by=block_index&order_dir=DESC"

# Get user's Counterparty balances
curl -sS "https://cp.blockvault.ai/api/v1/balances?address=<ADDRESS>&limit=100"

# Health check
curl -sS "https://cp.blockvault.ai/health"
```

**Asset card images:** `https://cp.blockvault.ai/img/cards/{ASSET_NAME}.png`
- Subassets: `https://cp.blockvault.ai/img/cards/PARENT.SUBASSET.png`

### Divisible vs indivisible assets

- **Divisible** (`divisible: true`): raw quantities are in satoshi-like units. Divide by `100000000` (10^8).
- **Indivisible** (`divisible: false`): quantities are whole units as-is.

Check the `divisible` field from the issuance response.

## Address resolution

Before any action involving the user's holdings, get the user's address:

Call `run_js` with:
- **function**: `"get_assets"`
- **data**: `{"hasBalance": true}`

Pick the Counterparty/Bitcoin asset and note its `address`.

## Valuation flow

### Step 1: Identify the asset

Parse the user's query for:
- Exact asset name (e.g. "RAREPEPE", "PEPECASH", "FDCARD")
- Collection name (e.g. "Rare Pepe" → search for RAREPEPE series)
- If the user says "my NFTs" or similar, resolve their address first and list their Counterparty balances.

### Step 2: Fetch on-chain data

Run these API calls in parallel (chain multiple curls):

```bash
# Issuance info (supply, locked, divisibility, creator)
curl -sS "https://cp.blockvault.ai/api/v1/issuances?asset=<ASSET>&order_by=block_index&order_dir=DESC&limit=1"

# All holders
curl -sS "https://cp.blockvault.ai/api/v1/balances?asset=<ASSET>&limit=100&order_by=quantity&order_dir=DESC"

# Current sell offers (open dispensers, cheapest first)
curl -sS "https://cp.blockvault.ai/api/v1/dispensers?asset=<ASSET>&status=0&limit=10&order_by=satoshirate&order_dir=ASC"

# Recent sales (dispense history)
curl -sS "https://cp.blockvault.ai/api/v1/dispenses?asset=<ASSET>&limit=30&order_by=block_index&order_dir=DESC"
```

### Step 3: Compute valuation metrics

From the on-chain data, calculate:

| Metric | Calculation |
|--------|-------------|
| **Total Supply** | From issuance `quantity` field (apply divisibility) |
| **Locked** | `locked` field — if true, supply is permanently fixed |
| **Holder Count** | Number of addresses with balance > 0 |
| **Top Holders** | Top 5 addresses by quantity |
| **Holder Concentration** | % of supply held by top 5 holders |
| **Floor Price** | Lowest `satoshirate` among open dispensers (if any) |
| **Ceiling Price** | Highest `satoshirate` among open dispensers |
| **Last Sale Price** | Most recent dispense `satoshirate` |
| **Avg Sale Price (30d)** | Average `satoshirate` from recent dispenses |
| **Sales Volume** | Total dispenses count and BTC volume |
| **Liquidity Score** | Number of open dispensers × total give_remaining |
| **Creator** | `issuer` from the first issuance |
| **Created Block** | `block_index` from the first issuance |

### Step 4: Web research

Call `web_search` to enrich the valuation with market context:

1. Search: `"<ASSET_NAME>" counterparty NFT price value`
2. Search: `"<ASSET_NAME>" rare pepe OR spells of genesis OR counterparty collection`
3. If the asset belongs to a known collection (Rare Pepe, SOG, Sarutobi, etc.), search for collection-level data.

Minimum 2 searches, maximum 4. Stop when you have enough context.

### Step 5: Generate valuation report

Present the full analysis using the template below.

## Known collections

Use these to contextualize the asset:

| Collection | Asset Pattern | Notes |
|------------|---------------|-------|
| Rare Pepe | RAREPEPE, PEPECASH, various names | Series 1-36, ~1,774 cards. Series 1 most valuable. |
| Spells of Genesis | FDCARD, SATOSHICARD, various | First blockchain trading cards (2015). |
| Sarutobi | SARUTOBICARD, CNPCARD | From the Sarutobi game. |
| Force of Will | FOWCARD, etc. | Japanese card game crossover. |
| Mafia Wars | MAFIACARD, etc. | Counterparty game cards. |
| Age of Chains | Various | Blockchain card game. |
| Bitcorn Crops | BITCORN, various CROP | Farming game collectibles. |

If the asset doesn't match a known collection, still analyze it based on on-chain data alone.

## Rendering

Present the results directly in your response using the Jinja2 templates below. Substitute `{{ var }}` with concrete values, expand `{% for %}` loops over items, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output.

**Example — Single asset valuation:**

````jinja
# {{ asset }} — Valuation Report

<img src="https://cp.blockvault.ai/img/cards/{{ asset }}.png" width="120" height="120" />

## On-Chain Summary

| Metric | Value |
|--------|-------|
| Collection | {{ collection_name }} |
| Creator | `{{ issuer }}` |
| Created | Block {{ created_block }} |
| Total Supply | {{ total_supply }} |
| Supply Locked | {{ "Yes ✅" if locked else "No ⚠️" }} |
| Divisible | {{ "Yes" if divisible else "No" }} |
| Holders | {{ holder_count }} addresses |

## Price Discovery

| Metric | Value |
|--------|-------|
| 🏷️ Floor Price | {{ floor_price_sats }} sats ({{ floor_price_btc }} BTC) |
| 📈 Ceiling Price | {{ ceiling_price_sats }} sats ({{ ceiling_price_btc }} BTC) |
| 💰 Last Sale | {{ last_sale_sats }} sats ({{ last_sale_btc }} BTC) |
| 📊 Avg Sale Price | {{ avg_sale_sats }} sats ({{ avg_sale_btc }} BTC) |
| 🔁 Total Sales | {{ total_sales }} dispenses |
| 💧 Open Listings | {{ open_dispensers }} dispensers |

## Holder Distribution

| # | Address | Quantity | % Supply |
|---|---------|----------|----------|
{% for h in top_holders -%}
| {{ loop.index }} | `{{ h.address }}` | {{ h.quantity }} | {{ h.pct }}% |
{% endfor %}

**Concentration:** Top 5 holders own {{ top5_pct }}% of supply.

{% if recent_sales %}
## Recent Sales

| Date | Buyer | Qty | Price (sats) | Price (BTC) |
|------|-------|-----|-------------|-------------|
{% for s in recent_sales -%}
| Block {{ s.block_index }} | `{{ s.destination }}` | {{ s.quantity }} | {{ s.satoshirate }} | {{ s.btc_price }} |
{% endfor %}
{% endif %}

## Market Context

{{ market_context }}

## Valuation Summary

{{ valuation_summary }}

> **Disclaimer:** This valuation is based on on-chain data and public market information. Counterparty NFT markets are illiquid — actual sale prices may vary significantly. This is not financial advice.
````

**Example — User's NFT portfolio:**

````jinja
# Your Counterparty NFT Portfolio

Address: `{{ address }}`

| Asset | | Quantity | Floor Price | Est. Value |
|---|---|---|---|---|
{% for a in assets -%}
| <img src="https://cp.blockvault.ai/img/cards/{{ a.name }}.png" width="40" height="40" /> **{{ a.name }}** | {{ a.quantity }} | {{ a.floor_sats }} sats | {{ a.est_value_sats }} sats |
{% endfor %}

| | |
|---|---|
| **Total Assets** | {{ total_assets }} |
| **Estimated Portfolio Value** | {{ total_value_sats }} sats ({{ total_value_btc }} BTC) |

> Floor prices based on cheapest open dispenser. Assets with no open dispensers are marked as N/A.
````

## Constraints

- **Non-custodial.** Never request private keys or mnemonics.
- **Use real addresses.** Always get the user's address from `get_assets` — never invent addresses.
- **On-chain data first.** Always base valuations on actual on-chain data. Web research enriches but does not replace.
- **Acknowledge illiquidity.** Counterparty NFT markets are thin — always note that valuations are estimates.
- **Image rendering.** Always include the asset card image: `<img src="https://cp.blockvault.ai/img/cards/ASSET.png" width="120" height="120" />` for single assets, `width="40" height="40"` in tables.
