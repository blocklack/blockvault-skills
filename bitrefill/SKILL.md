---
name: bitrefill
description: Browse, search and purchase gift cards, mobile top-ups, eSIMs and bill payments on Bitrefill, and consult invoices and orders. Payments go on-chain from the user's BlockVault wallet in USDC on Base. Use this whenever the user wants to explore Bitrefill products, see prices, buy something, or look up a previous purchase.
metadata:
  tool: bash
  category: tools
  enabled: true
  homepage: https://www.bitrefill.com/account/developers
  secrets:
    - key: BITREFILL_API_KEY
      description: "Sign in with your Bitrefill account."
      oauth:
        issuer: https://api.bitrefill.com
        scope: mcp
        resource: https://api.bitrefill.com/mcp
---

# Bitrefill

Interactive Bitrefill assistant. Use the available endpoints to **browse, search, inspect, purchase, and review** products and invoices on behalf of the user. Render results as readable Markdown  so the user can navigate them, never dump raw JSON.

Detect the language of the user's query and reply in that language.

## API

All calls hit the **same MCP JSON-RPC endpoint**: `POST https://api.bitrefill.com/mcp`.

The secret placeholder `{{BITREFILL_API_KEY}}` is resolved automatically at execution time — write it verbatim in the Authorization header, never echo it back to the user.

### The one curl template

Copy this template **as-is**. The ONLY thing you change between calls is the `<TOOL_NAME>` and `<ARGUMENTS>` placeholders in the `-d` body. Headers, URL and everything else are fixed.

```bash
curl -sS -X POST https://api.bitrefill.com/mcp \
  -H "Authorization: Bearer {{BITREFILL_API_KEY}}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"<TOOL_NAME>","arguments":<ARGUMENTS>}}'
```

`<TOOL_NAME>` is one of the 7 tools below; `<ARGUMENTS>` is its JSON arguments object.

Never split `-H` from its value. Never put an `-H` between `-d` and the URL. The URL is the LAST argument.

### Response shape

Bitrefill answers with one Server-Sent Events frame:

```
event: message
data: {"result":{"content":[{"type":"text","text":"<YAML payload>"}]},"jsonrpc":"2.0","id":1}
```

Take the JSON on the `data:` line, then read `result.content[0].text` — that's pre-formatted YAML you can render directly. If you see `error` instead of `result`, surface the error message and stop.

### Tools (7)

For every call, only `<TOOL_NAME>` and `<ARGUMENTS>` change. Pick a tool below and drop its `name` + `arguments` into the template above.

**`search-products`** — list or search gift cards / eSIMs / refills.
Args: `query` (default `*` = all), `country` (ISO-2, default `US`), `category` (enum: `gift-cards`, `esim`, `refill`, `flights`, `accommodation`, `car-rental`, `streaming`, `games`, `groceries`, `travel`, `bill`, …), `product_type` (`giftcard` | `esim`), `in_stock` (default `true`), `page` (default 1), `per_page` (default 25, max 250).

```json
{"name": "search-products", "arguments": {"query": "amazon", "country": "US", "per_page": 10}}
{"name": "search-products", "arguments": {"country": "ES", "category": "accommodation", "per_page": 20}}
```

**`get-product-details`** — full product info, including `packages[]` (each with `package_value` and `price`), `range`, `recipient_type`, `prepayment` (if any).
Args: `product_id` (required slug from search), `language` (default `en`), `currency` (`BTC` | `USD` | `EUR` | `USDC` | `USDT` | `ETH` | `SOL` | `GBP` | `AUD` | `CAD` | `INR` | `BRL`; default `BTC`, use `USDC` for BlockVault).

```json
{"name": "get-product-details", "arguments": {"product_id": "amazon_com-usa", "currency": "USDC"}}
```

**`buy-products`** — create the invoice. See variants below.
Args:

- `cart_items` — array (max 15). Each item: `product_id` + `package_id` (the **`package_value`** string, e.g. `"50"` or `"5GB, 30 Days"` — NOT the full `slug<&>value`). Optional per-item: `refill_input` (phone/account/email when `recipient_type` requires it), `bill_payment_id` (for prepayment products), `gift` (`{recipient_name, recipient_email, sender_name, message?, theme?, send_date?}`).
- `payment_method` — for BlockVault use **`usdc_base`**. Other allowed: `bitcoin`, `lightning`, `ethereum`, `usdc_polygon`, `usdt_polygon`, `usdc_erc20`, `usdt_erc20`, `usdt_trc20`, `solana`, `usdc_arbitrum`, `usdc_solana`, `eth_base`, `litecoin`, `balance`, `cashback`.
- `return_payment_link` — default `true`. Set to **`false`** so the response contains crypto `payment` info instead of a web checkout link.
- `balance_currency` — only with `payment_method:"balance"` (`EUR` | `USD` | `XBT`).

```json
{"name": "buy-products", "arguments": {"cart_items": [{"product_id": "amazon_com-usa", "package_id": "50"}], "payment_method": "usdc_base", "return_payment_link": false}}
```

**`submit-prepayment-step`** — only for products whose `get-product-details` response contains a `prepayment` block (e.g. prepaid Visa, utility bills). Loop until the response has `step:"final"` and a `bill_payment_id`, then pass that id into `buy-products.cart_items[].bill_payment_id`.
Args: `product_id`, `step_number` (start at 1), `form_data` (object keyed by field id), `bill_payment_id` (echoed back from step 1 onwards).

```json
{"name": "submit-prepayment-step", "arguments": {"product_id": "<slug>", "step_number": 1, "form_data": {"first_name": "Alice"}}}
```

**`get-invoice-by-id`** — invoice status, payment info, embedded `orders[]` with `redemption_info` once delivered.
Args: `invoice_id` (UUID with dashes), `invoice_access_token` (returned by `buy-products`).

```json
{"name": "get-invoice-by-id", "arguments": {"invoice_id": "<uuid>", "invoice_access_token": "<token>"}}
```

**`list-invoices`** — paginated invoice history for the authenticated user.
Args: `limit` (1–50, default 25), `start`, `after` / `before` (ISO 8601), `include_orders` (default `true`).

```json
{"name": "list-invoices", "arguments": {"limit": 20}}
```

**`update-order`** — order housekeeping. `order_id` is the **24-char hex** id from `invoice.orders[].id` (not the invoice UUID).
Args: `order_id`, `remaining_amount?`, `is_archived?`.

```json
{"name": "update-order", "arguments": {"order_id": "<24-char hex>", "is_archived": true}}
```

### Invoice creation variants

`buy-products.cart_items[i]` shape per recipient_type:

**Fixed-denomination gift card**: `package_id` is the `package_value` (just the amount), not the full `slug<&>value`.

```json
{"product_id": "amazon_com-usa", "package_id": "50"}
```

**Range / custom value**: products with a `range` accept any numeric value in range as `package_id`.

```json
{"product_id": "amazon_com-usa", "package_id": "37.5"}
```

**Phone top-up** (`recipient_type:"phone_number"`): add `refill_input` in international format.

```json
{"product_id": "att-usa-topup", "package_id": "25", "refill_input": "14155551234"}
```

**eSIM**: `package_id` is the plan string from `packages[].package_value`.

```json
{"product_id": "esim-europe", "package_id": "5GB, 30 Days"}
```

**Bill payment**: complete the `submit-prepayment-step` loop first, then include the resulting `bill_payment_id`.

```json
{"product_id": "<bill slug>", "package_id": "<amount>", "bill_payment_id": "<from prepayment loop>"}
```

Always: `"payment_method": "usdc_base"` and `"return_payment_link": false`. The invoice response then carries `payment.address`, `payment.amount`, `payment.currency` and `expiration_minutes` for the on-chain transfer.

### Redemption info (from `get-invoice-by-id.orders[i].redemption_info` once invoice is `complete`)

**gift_card**: `code`, optional `pin`, `link`, `instructions`
**phone_refill**: usually empty (credit is applied directly to the number)
**esim**: `esim_install_link` (LPA activation), `pin`
**bill_payment**: provider receipt fields, `instructions`

Never mask the redemption code — the user paid for it.

## Rendering

The templates below are **Jinja2** — substitute `{{ var }}` placeholders with concrete values from the API response, expand `{% for %}` loops, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output.

Ignore the `price` field entirely — it is an internal Bitrefill unit. The user-facing denomination is `value` + `currency`. The actual USDC cost only appears in the invoice as `payment.amount`. Never expose product slugs / ids to the user; keep them internal for chaining tool calls and refer to products by row number or display name.

**Product list** (search / browse results):

```jinja
{# one-line intro paraphrasing the user intent #}

| # | Product | Country | Currency |
|---|---------|---------|----------|
{% for p in data[:10] %}
| **{{ loop.index }}** | **{{ p.name }}** | {{ p.country_name or p.country_code }} | {{ p.currency }} |
{% endfor %}

Pick a number to see details or buy.
```

**Product detail**:

```jinja
## {{ product.name }}

{{ product.description | truncate(200) if product.description }}

| | |
|---|---|
| Country | {{ product.country_name }} |
| Currency | {{ product.currency }} |
{% if product.range %}| Range | {{ product.currency }} {{ product.range.min }} – {{ product.range.max }} |{% endif %}
{% if product.packages %}| Packages | {{ product.packages | map(attribute='value') | join(' · ') }} {{ product.currency }} |{% endif %}
{% if product.recipient_type == 'phone_number' %}| Requires | Phone number |{% endif %}
```


**Invoice list**:

```jinja
| # | Invoice | Created | Status | Total |
|---|---------|---------|--------|-------|
{% for inv in data %}
| {{ loop.index }} | `{{ inv.id }}` | {{ inv.created_time | date }} | {{ inv.status }} | {{ inv.payment.amount }} {{ inv.payment.currency }} |
{% endfor %}
```

**Invoice detail**:


```jinja
## Invoice `{{ inv.id }}`

| | |
|---|---|
| Status | {{ inv.status }} |
| Total | {{ inv.payment.amount }} {{ inv.payment.currency }} |
| Method | {{ inv.payment.method }} |
{% if inv.payment.address %}| Address | `{{ inv.payment.address[:8] }}…{{ inv.payment.address[-6:] }}` |{% endif %}
{% if inv.expiration_minutes %}| Expires in | {{ inv.expiration_minutes }} min |{% endif %}

## Orders

| Order | Product | Amount |
|-------|---------|--------|
{% for o in inv.orders %}
| `{{ o.id }}` | {{ o.product_name }} | {{ o.value }} {{ o.currency }} |
{% endfor %}
```

**Order redemption** (read from `inv.orders[i].redemption_info` after `get-invoice-by-id` returns `status:"complete"`):

```jinja
## {{ order.product_name }} — {{ order.value }} {{ order.currency }}

| | |
|---|---|
{% if order.redemption_info.code %}| Code | `{{ order.redemption_info.code }}` |{% endif %}
{% if order.redemption_info.pin %}| PIN | `{{ order.redemption_info.pin }}` |{% endif %}
{% if order.redemption_info.link %}| Link | {{ order.redemption_info.link }} |{% endif %}
{% if order.redemption_info.esim_install_link %}| eSIM activation | {{ order.redemption_info.esim_install_link }} |{% endif %}

{% if order.redemption_info.instructions %}{{ order.redemption_info.instructions }}{% endif %}
```

After rendering a list, ask the user to pick a number to drill down or proceed to purchase.

## Purchase flow

How to pay an invoice | buy a product on Bitrefill, step by step:

### Check balance

Call `run_js` with:

- **function**: "get_assets"
- **data**: `{"hasBalance": true}`

Confirm USDC on `base` has enough for the purchase. If not, tell the user and stop.

### Resolve product

Search or browse using `search-products` if needed, then call `get-product-details` with `currency:"USDC"` to read `packages[]` and pick the exact `package_value` to use as `package_id`. Do NOT invent ids — they must come from a previous API response. If the product has a `prepayment` block, run the `submit-prepayment-step` loop until you get a `bill_payment_id`.

### Create invoice

Call `buy-products` with `payment_method:"usdc_base"` and `return_payment_link:false`. Extract from the response: `invoice_id`, `invoice_access_token`, `payment.address`, `payment.amount`, `payment.currency`, and `expiration_minutes`.

### Confirm with the user

Show product, denomination, USDC amount, chain (Base) and truncated payment address. Wait for an explicit "yes" / "confirm". Do NOT proceed without it.

### Pay on-chain

Call `run_js` with:

- **function**: "transfer"
- **data**: `{"symbol": "USDC", "amount": "<payment.amount>", "toAddress": "<payment.address>", "blockchain": "base"}`

Capture the returned `txHash`.

### Poll invoice status

Call `get-invoice-by-id` with `invoice_id` + `invoice_access_token` up to 5 times, ~15 s apart, until `status` is `complete`. If it transitions to `failed` / `expired`, report it and stop. Intermediate statuses: `unpaid`, `payment_detected`, `payment_confirmed`.

### Redeem

The successful `get-invoice-by-id` response contains `orders[].redemption_info`. Show the redemption code, link and / or PIN to the user. Never mask the code.

### Save receipt

Use the `text_editor` tool to create `receipts/bitrefill-<INVOICE_ID>.md` containing product name, totals, `txHash` and redemption codes.

## Constraints

- Always `payment_method: "usdc_base"` + `return_payment_link: false`. Never `"balance"` / `"cashback"` (the user has no Bitrefill account balance).
- In `cart_items[].package_id` use the `package_value` string, NOT the full `slug<&>value`.
- Never invent product / invoice / order ids — derive them from API responses.
- Never retry a failed payment without explicit user confirmation.
- Never echo `{{BITREFILL_API_KEY}}` to the user.
- The OAuth access token is audience-bound to `https://api.bitrefill.com/mcp`. The legacy REST `/v2/...` endpoints will reject it with `invalid_token` — always use the MCP JSON-RPC endpoint above.
