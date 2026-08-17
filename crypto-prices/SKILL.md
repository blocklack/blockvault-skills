---
name: crypto-prices
description: Fetch current prices for Bitcoin and other cryptocurrencies. Use when the user asks about crypto prices, token value, how much a coin costs, or wants a quick price check.
metadata:
  category: market
  enabled: true
  background: true
---

# Crypto Prices

Fetch real-time cryptocurrency prices via the BlockVault kline API and deliver a concise summary.

## Instructions

Execute all steps silently. Detect the user's language and reply in that language.

### Step 1: Determine symbols

Parse the user's message to extract which cryptocurrencies they want prices for.

Always append `USDT` to the base symbol to form the trading pair. If the user already provides a pair (e.g. BTCUSDT), use it as-is.

Default set when no specific coin is mentioned: BTC, ETH, SOL, BNB, XRP.

### Step 2: Fetch current prices

For each symbol, call `bash` with:

- **command**: String, Required. A curl command to the kline API.

```bash
curl -s "https://api.blockvault.ai/api/v1/market/kline/BTCUSDT?interval=D&category=spot&limit=2"
```

Replace `BTCUSDT` with the target symbol. Use `limit=2` to get today's candle and yesterday's close for calculating 24h change.

The response JSON contains `list` — an array of candle arrays `[timestamp_ms, open, high, low, close, volume]`:
- `list[0]` = current/latest candle
- `list[0][4]` = current price (close)
- `list[1][4]` = previous close (for 24h change calculation)

Calculate 24h change: `((current_close - previous_close) / previous_close) * 100`

Make one `bash` call per symbol. Do NOT combine multiple curls into one call.

### Step 3: Notify user with results

Call `notify_user` with:

- **title**: String, Required. E.g. `"Crypto Prices"` or `"BTC Price"` (max 50 chars).
- **message**: String, Required. Plain text price summary (max 1024 chars). No markdown.

**Format for single asset:**
```
BTC: $104,250.50 (+2.3% 24h)
```

**Format for multiple assets:**
```
BTC: $104,250.50 (+2.3%)
ETH: $3,820.00 (-0.5%)
SOL: $178.30 (+5.1%)
BNB: $620.00 (+1.2%)
XRP: $0.62 (-1.8%)
```

## Constraints

- Only use the BlockVault kline API — never invent prices.
- Always show 24h percentage change alongside the price.
- Format prices with commas for thousands and appropriate decimal places (2 for most, 4+ for sub-dollar tokens).
- If the API fails for a symbol, skip it and note "unavailable" in the notification.
- Always call `notify_user` as the final action.