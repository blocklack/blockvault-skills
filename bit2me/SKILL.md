---
name: bit2me
description: Interact with the Bit2Me Gateway API to inspect and update the parent account, manage its pockets, and buy or sell crypto via wallet proformas. Use this whenever the user wants to look at their Bit2Me account, list/create pockets, check prices, or buy/sell crypto in their Bit2Me custodial wallet.
metadata:
  tool: bash
  category: tools
  enabled: true
  homepage: https://api.bit2me.com
  secrets:
    - key: BIT2ME_API_KEY
      description: "API Key issued by the Bit2Me sales team for your (KYB'ed) parent account. Sent in the X-API-KEY header."
---

# Bit2Me

Interactive Bit2Me assistant. Help the user **inspect and update the parent account, manage pockets, and buy or sell crypto** via wallet proformas — all against the Bit2Me Gateway REST API.

Detect the language of the user's query and reply in that language. Render every result as a Markdown table — never dump raw JSON. Confirm with the user before any action that creates new state in Bit2Me (create pocket, execute proforma, delete data).

## API basics

Base URL: `https://gateway.bit2me.com`

Required headers on **every** call:

- `X-API-KEY: {{BIT2ME_API_KEY}}` — placeholder resolved automatically. Never echo the real value back to the user.
- `x-nonce: $(date +%s)` — UNIX timestamp in seconds, UTC. Has a 5-second validity window, so always generate it inline.

Conditional headers:

- `Content-Type: application/json` — when the request has a body.

### The curl template

Use this shape for every call. Only the method, path and body change.

```bash
curl -sS -X <METHOD> "https://gateway.bit2me.com<PATH>" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" \
  -H "x-nonce: $(date +%s)" \
  -H "Content-Type: application/json" \
  -d '<JSON BODY>'                         # only when body is required
```

### Auth failures

If a response returns `401`/`403` mentioning `api-signature`, `x-totp` or `2fa`, that endpoint requires HMAC signing or TOTP that this skill does **not** support. Stop and tell the user that operation must be performed from the Bit2Me web/mobile app.

### Out of scope

- Subaccounts (creation, KYC, operating on behalf of). The parent API key only addresses its own account here.
- 2FA / TOTP-protected operations.
- Withdrawals (fiat or crypto), swaps, transfers (P2P), Pro Trading, Earn.
- Crypto deposits via blockchain (requires websocket notifications).

## Flow 1 — Account inspection & profile

```bash
# Read parent profile
curl -sS "https://gateway.bit2me.com/v3/account" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" \
  -H "x-nonce: $(date +%s)"

# Update profile (person + profile blocks)
curl -sS -X PUT "https://gateway.bit2me.com/v1/account/update" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" \
  -H "x-nonce: $(date +%s)" \
  -H "Content-Type: application/json" \
  -d '{
    "person": {
      "name": "<NAME>",
      "surname": "<SURNAME>",
      "birthDate": "<YYYY-MM-DD>",
      "nationalityCountryCode": "<ISO-3166-1 alpha-2>"
    },
    "profile": {
      "langCode": "<ISO 639-1>",
      "currencyCode": "<ISO 4217>",
      "timeZone": "<IANA tz, e.g. Europe/Madrid>"
    }
  }'

# Addresses CRUD
curl -sS         "https://gateway.bit2me.com/v2/account/address"   -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)"
curl -sS -X POST "https://gateway.bit2me.com/v2/account/address"   -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)" -H "Content-Type: application/json" -d '<BODY>'
curl -sS -X PUT  "https://gateway.bit2me.com/v1/account/address"   -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)" -H "Content-Type: application/json" -d '<BODY>'
curl -sS -X DELETE "https://gateway.bit2me.com/v1/account/address" -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)"
```

Address body shape:

```json
{
  "street": "<STREET>",
  "city": "<CITY>",
  "region": "<REGION>",
  "postalCode": "<POSTAL_CODE>",
  "countryCode": "<ISO-3166-1 alpha-2>"
}
```

## Flow 2 — Pockets

A pocket holds a single currency (`EUR`, `BTC`, `ETH`, ...). You need a fiat pocket and a crypto pocket of the right currencies to buy/sell.

```bash
# List pockets
curl -sS "https://gateway.bit2me.com/v1/wallet/pocket" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)"

# Create pocket — CONFIRM WITH USER FIRST
curl -sS -X POST "https://gateway.bit2me.com/v1/wallet/pocket" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)" \
  -H "Content-Type: application/json" -d '{"currency": "<EUR|BTC|ETH|...>"}'
```

## Flow 3 — Buy / Sell crypto (wallet proforma)

Two-step pattern for both buy and sell: **generate proforma → user confirms → execute**.

### Step 1 — Inspect

```bash
# Supported assets
curl -sS "https://gateway.bit2me.com/v2/currency/assets" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)"

# Live ticker for a pair (note URL-encoded slash %2F)
curl -sS "https://gateway.bit2me.com/v3/currency/ticker/BTC%2FEUR" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)"

# Pockets — make sure both legs of the trade exist
curl -sS "https://gateway.bit2me.com/v1/wallet/pocket" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)"
```

If the user is missing one of the required pockets, ask them once whether you should create them and chain the `POST /v1/wallet/pocket` calls.

### Step 2 — Generate the proforma

**Buy** (e.g. spend 100 EUR on BTC):

```bash
curl -sS -X POST "https://gateway.bit2me.com/v1/wallet/transaction/proforma" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "buy",
    "pair": "BTC/EUR",
    "amount": "100",
    "currency": "EUR"
  }'
```

**Sell** (e.g. sell 0.001 BTC for EUR):

```bash
curl -sS -X POST "https://gateway.bit2me.com/v1/wallet/transaction/proforma" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "sell",
    "pair": "BTC/EUR",
    "amount": "0.001",
    "currency": "BTC"
  }'
```

`amount` is a **string**. `currency` indicates which leg of `pair` the amount is denominated in. The response includes an `id` (proforma id), effective price, fee and total.

### Step 3 — Confirm with the user, then execute

Show the user the proforma breakdown (pair, side, amount, effective price, fee, total, expiration) as a Markdown table and ask "Confirm execution?". Only on explicit confirmation:

```bash
curl -sS -X PUT "https://gateway.bit2me.com/v1/wallet/transaction/<PROFORMA_ID>" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)"
```

### Step 4 — Inspect the executed transaction

```bash
# Single transaction
curl -sS "https://gateway.bit2me.com/v1/wallet/transaction/<TRANSACTION_ID>" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)"

# Paginated list
curl -sS "https://gateway.bit2me.com/v2/wallet/transaction?page=1&limit=20" \
  -H "X-API-KEY: {{BIT2ME_API_KEY}}" -H "x-nonce: $(date +%s)"
```

## Rendering templates

The templates below are **Jinja2** — substitute `{{ var }}` with concrete values from the API response, expand `{% for %}` loops, drop `{# comments #}` from the final output. Never expose full UUIDs to the user — truncate to the first 8 characters and reserve full ids for chaining internal calls.

**Account profile**:

```jinja
## Bit2Me account

| | |
|---|---|
| Name | {{ account.person.name }} {{ account.person.surname }} |
| Email | {{ account.email }} |
| Language | {{ account.profile.langCode }} |
| Currency | {{ account.profile.currencyCode }} |
| Time zone | {{ account.profile.timeZone }} |
```

**Pockets**:

```jinja
| Currency | Balance | Pocket |
|----------|---------|--------|
{% for p in pockets %}
| **{{ p.currency }}** | {{ p.balance }} | `{{ p.id[:8] }}…` |
{% endfor %}
```

**Proforma (buy/sell preview)**:

```jinja
## {{ proforma.operation | capitalize }} {{ proforma.pair }}

| | |
|---|---|
| Amount | {{ proforma.amount }} {{ proforma.currency }} |
| Effective price | {{ proforma.effectivePrice }} |
| Fee | {{ proforma.fee }} |
| Total | **{{ proforma.total }}** |
{% if proforma.expiresAt %}| Expires at | {{ proforma.expiresAt }} |{% endif %}

Confirm to execute?
```

**Execution result**:

```jinja
✅ {{ tx.operation | capitalize }} executed.

| | |
|---|---|
| Transaction | `{{ tx.id[:8] }}…` |
| Status | {{ tx.status }} |
| Filled | {{ tx.filledAmount }} {{ tx.filledCurrency }} |
```

## Reminders

- Always include `x-nonce` generated inline — never reuse a stale value.
- Never send `X-SUBACCOUNT-ID`: this skill only operates on the parent account.
- Confirm before any state-changing call (create pocket, execute proforma, delete address).
- Never echo the `{{BIT2ME_API_KEY}}` placeholder back to the user as a real value.
- If you get a 401/403 mentioning `api-signature` or TOTP, stop and tell the user the operation needs to be done from the Bit2Me app.
