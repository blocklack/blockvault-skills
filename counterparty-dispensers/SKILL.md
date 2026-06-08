---
name: counterparty-dispensers
description: Browse Counterparty dispensers, view asset card images, check dispense history, and initiate transfers of Counterparty assets. Use this when the user asks about XCP dispensers, wants to buy from a dispenser, explore available Counterparty tokens, or send/transfer a Counterparty asset.
metadata:
  tool: bash
  category: wallet
  enabled: true
  homepage: https://cp.blockvault.ai
---

# Counterparty Dispensers

Interactive Counterparty assistant. Help the user **browse dispensers, view asset cards, check dispense history, and transfer Counterparty assets** via the BlockVault Counterparty node.

## Instructions

- **Get the address first.** Call `run_js` → `get_assets` with `{"hasBalance": true}` to obtain the user's real Counterparty address before any action.
- **Use `bash` for API calls.** Execute `curl` commands — never invent responses.
- Detect the user's language and reply in that language.
- Render results as Markdown with images and tables — never dump raw JSON.
- Lead with **price, asset image and availability**, not technical fields.
- Confirm with the user before initiating any transfer or purchase.

## API endpoints

Base URL: `https://cp.blockvault.ai`

No authentication required — all endpoints are public. All list endpoints support `limit` (default 100, max 1000), `offset`, `order_by`, `order_dir` (ASC/DESC), plus any column as a filter parameter.

```bash
# List open dispensers (status=0 means open)
curl -sS "https://cp.blockvault.ai/api/v1/dispensers?status=0&limit=20&order_by=block_index&order_dir=DESC"

# Filter dispensers by asset
curl -sS "https://cp.blockvault.ai/api/v1/dispensers?asset=<ASSET_NAME>&status=0&limit=20"

# Filter dispensers by source address (owner)
curl -sS "https://cp.blockvault.ai/api/v1/dispensers?source=<SOURCE_ADDRESS>&status=0&limit=20"

# Get dispense history for a specific dispenser
curl -sS "https://cp.blockvault.ai/api/v1/dispenses?dispenser_tx_hash=<DISPENSER_TX_HASH>&limit=20&order_by=block_index&order_dir=DESC"

# Get dispense history for a buyer address
curl -sS "https://cp.blockvault.ai/api/v1/dispenses?destination=<BUYER_ADDRESS>&limit=20&order_by=block_index&order_dir=DESC"

# Get asset issuance info (check divisibility)
curl -sS "https://cp.blockvault.ai/api/v1/issuances?asset=<ASSET_NAME>&order_by=block_index&order_dir=DESC&limit=1"

# Get balances for an address
curl -sS "https://cp.blockvault.ai/api/v1/balances?address=<USER_ADDRESS>&limit=50"

# Health check
curl -sS "https://cp.blockvault.ai/health"
```

**Notes:**
- Dispenser `status`: `0` = Open (accepting BTC), `10` = Closed, `11` = Closing.
- `satoshirate` = BTC cost in satoshis per dispense.
- `give_quantity` = amount dispensed per payment.
- `give_remaining` = total remaining in escrow.
- Asset card images are available at: `https://cp.blockvault.ai/img/cards/{ASSET_NAME}.png`
  - Example: `https://cp.blockvault.ai/img/cards/PEPECASH.png`
  - Subassets use dot notation: `https://cp.blockvault.ai/img/cards/PARENT.SUBASSET.png`

### Divisible vs indivisible assets

Counterparty assets can be **divisible** (8 decimal places) or **indivisible** (whole units only). The API always returns raw integer quantities regardless of divisibility.

- **Divisible** (`divisible: true`): quantities are in satoshi-like units. Divide by `100000000` (10^8) to get the real amount.
  - Example: `give_quantity: 10000000` → actual amount is `0.10000000`
- **Indivisible** (`divisible: false`): quantities are whole units as-is.
  - Example: `give_quantity: 10` → actual amount is `10`

**Inference shortcut**: If `give_quantity` or `escrow_quantity` are very large round numbers (e.g. 10000000, 100000000, 500000000), the asset is almost certainly divisible — treat it as divisible without an extra API call. Only fetch issuance info to confirm when values are ambiguous (e.g. could be 100 indivisible units or 0.000001 divisible).

**To confirm divisibility via API**, collect unique asset names and make **one** issuances call per unique asset (NOT per dispenser):

```bash
curl -sS "https://cp.blockvault.ai/api/v1/issuances?asset=<ASSET_NAME>&order_by=block_index&order_dir=DESC&limit=1"
```

The response includes a `divisible` field (boolean). If `true`, divide all quantities (`give_quantity`, `give_remaining`, `escrow_quantity`) by `100000000` for display and transfers.

**Shortcut**: If all dispensers in the result share the same asset (common case when filtering by asset), only ONE extra call is needed.

## Address resolution

Before any action, get the user's address:

Call `run_js` with:
- **function**: `"get_assets"`
- **data**: `{"hasBalance": true}`

Pick the Counterparty/Bitcoin asset and note its `address`. Use this exact address everywhere.

## Browse dispensers flow

Follow all steps silently, DO NOT OMIT ANY STEP.

1. Get the user's address (address resolution above).

2. Determine search criteria from user intent:

   | User wants | Parameters |
   |------------|------------|
   | Cheapest dispensers | `order_by=satoshirate&order_dir=ASC` |
   | Most expensive | `order_by=satoshirate&order_dir=DESC` |
   | Newest (recently created) | `order_by=block_index&order_dir=DESC` |
   | Oldest | `order_by=block_index&order_dir=ASC` |
   | Most stock available | `order_by=give_remaining&order_dir=DESC` |
   | Specific asset | `asset=ASSET_NAME` |
   | Specific owner | `source=ADDRESS` |
   | Default (no preference) | `order_by=block_index&order_dir=DESC` |

   Combine filters as needed (e.g. cheapest PEPECASH: `asset=PEPECASH&order_by=satoshirate&order_dir=ASC`).

3. Fetch dispensers with pagination:

   ```bash
   curl -sS "https://cp.blockvault.ai/api/v1/dispensers?status=0&limit=10&offset=0&order_by=block_index&order_dir=DESC"
   ```

   - Use `limit=10` per page for readability.
   - Use `offset` for pagination: page 1 = `offset=0`, page 2 = `offset=10`, page 3 = `offset=20`, etc.
   - If the user asks for "more" or "next page", increment offset by the limit.

4. For each dispenser result, calculate:
   - BTC price: `satoshirate / 100000000`
   - Check if asset is divisible (fetch issuance). If divisible, divide `give_quantity` and `give_remaining` by `100000000`.
   - Price per unit: `satoshirate / display_give_quantity` sats
   - Availability: `give_remaining / give_quantity` dispenses left

5. Render results using the template below. At the end, indicate if more results are available:
   - If the response returned exactly `limit` items, tell the user: "There are more results. Say **next** to see more."
   - Track the current offset so the next request increments correctly.

## Buy from dispenser flow

1. Resolve address and fetch the target dispenser info.

2. Show the user what they will get:
   - Asset name and image
   - BTC cost (satoshirate in sats and BTC)
   - Quantity received (give_quantity)

3. On user confirmation, send BTC to the dispenser source address. Call `run_js`:

   - **function**: `"transfer"`
   - **data**: `{"symbol": "BTC", "blockchain": "bitcoin", "toAddress": "<DISPENSER_SOURCE>", "amount": "<AMOUNT_BTC>"}`

   The `amount` is `satoshirate / 100000000` as a string.

4. Confirm the transaction was initiated. Explain the dispenser will auto-send the asset once the BTC transaction is confirmed on-chain (typically 1-3 blocks).

## Transfer asset flow

1. Resolve address.

2. Confirm with the user:
   - Asset being sent
   - Amount
   - Destination address

3. On confirmation, call `run_js`:

   - **function**: `"transfer"`
   - **data**: `{"symbol": "<ASSET_NAME>", "blockchain": "counterparty", "toAddress": "<DESTINATION_ADDRESS>", "amount": "<AMOUNT>"}`

4. The wallet will prompt for transaction signing.

## Rendering

Present the results directly in your response using the Jinja2 templates below. Substitute `{{ var }}` with concrete values, expand `{% for %}` loops over items, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output. Always include the asset card image when available.

**Guidelines:**
- Lead with the most relevant info (price for buyers, availability for browsers, quantity for history).
- Use tables for lists, key-value pairs for single items.
- Always render asset images inline at 40×40px: `<img src="https://cp.blockvault.ai/img/cards/ASSET.png" width="40" height="40" />`
- Show full addresses — never truncate.
- Show BTC prices in both sats and BTC.
- For divisible assets, always show the human-readable amount (after dividing by 10^8). Show up to 8 decimal places, trimming trailing zeros (e.g. `0.1` not `0.10000000`). If indivisible, show the raw integer as-is.
- When paginated, end with "Say **next** to see more results." if there are more.

**Example — Dispenser list:**

````jinja
## Counterparty Dispensers — {{ filter }} ({{ count }} results)

| Asset | 💰 Price | 📦 Per Dispense | 📊 Remaining | 🏠 Source |
|---|---|---|---|---|
{% for d in dispensers -%}
| <img src="https://cp.blockvault.ai/img/cards/{{ d.asset }}.png" width="40" height="40" /> **{{ d.asset }}** | {{ d.satoshirate }} sats ({{ d.btc_price }} BTC) | {{ d.display_give_quantity }} | {{ d.display_give_remaining }} / {{ d.display_escrow_quantity }} | `{{ d.source }}` |
{% endfor %}
````

**Example — Dispense history:**

````jinja
## Dispense History

| Block | Buyer | Asset | Quantity | Tx |
|-------|-------|-------|----------|-----|
{% for p in dispenses -%}
| {{ p.block_index }} | `{{ p.destination }}` | {{ p.asset }} | {{ p.display_quantity }} | `{{ p.tx_hash }}` |
{% endfor %}
````

**Example — Purchase confirmation:**

````jinja
## Buy from Dispenser

| | |
|---|---|
| 🎨 Asset | <img src="https://cp.blockvault.ai/img/cards/{{ asset }}.png" width="40" height="40" /> **{{ asset }}** |
| 💸 Cost | {{ satoshirate }} sats ({{ btc_price }} BTC) |
| 📦 You Receive | {{ display_give_quantity }} {{ asset }} |
| 📍 Send To | `{{ source }}` |

> Send exactly **{{ satoshirate }} sats** to the source address. The dispenser auto-sends {{ display_give_quantity }} {{ asset }} after 1 confirmation.
````

## Constraints

- **Non-custodial.** Never request private keys or mnemonics — transfers go through `run_js` → `transfer` which prompts wallet signing.
- **Use real addresses.** Always get the user's address from `get_assets` — never invent, guess or fabricate an address. Every address used in API calls and transfers MUST come from the user's wallet or from a prior API response (e.g. a dispenser's `source` field).
- **Confirm before spending.** Always show cost and what the user receives before executing a buy or transfer.
- **Image rendering.** Always include the asset card image at 40×40px: `<img src="https://cp.blockvault.ai/img/cards/ASSET.png" width="40" height="40" />`
