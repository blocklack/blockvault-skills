---
name: historical-prices-bg
description: Fetch historical OHLCV price data for any cryptocurrency pair.
metadata:
  category: market
  enabled: true
  background: true
---


## Historical Prices (Background)

Fetch historical OHLCV (Open, High, Low, Close, Volume) candle data for any crypto pair using the BlockVault kline API via `bash`. Deliver analysis via `notify_user`.

### Recognize symbols, timeframe, and period

Extract from the user's message:
- **Symbol(s)**: The cryptocurrency pair(s). Append `USDT` if only base symbol given (BTC -> BTCUSDT).
- **Timeframe**: Map to API interval values (default: `D` for daily).
- **Period**: How far back to look (default: 7 days).

**Timeframe mapping:**

| User says | API interval | Typical limit |
|-----------|-------------|---------------|
| 1 minute, 1m | `1` | 60 |
| 5 minutes, 5m | `5` | 60 |
| 15 minutes, 15m | `15` | 48 |
| 1 hour, hourly | `60` | 24-48 |
| 4 hours, 4h | `240` | 42 |
| daily, 1d (default) | `D` | 7-30 |
| weekly, 1w | `W` | 12-52 |
| monthly, 1M | `M` | 12 |

**Period to limit mapping:**

| Period | Interval | Limit |
|--------|----------|-------|
| last 24h | `60` | 24 |
| last 7 days | `D` | 7 |
| last 30 days | `D` | 30 |
| last 3 months | `W` | 12 |
| last year | `W` | 52 |

### Fetch OHLCV data

For each symbol, call `bash` with:

- **command**: String, Required. A curl to the kline API.

```bash
curl -s "https://api.blockvault.ai/api/v1/market/kline/<PAIR>?interval=<INTERVAL>&category=spot&limit=<LIMIT>"
```

Replace:
- `<PAIR>` with the target pair
- `<INTERVAL>` with the chosen interval
- `<LIMIT>` with the chosen limit (max 50)

Response: `{list: [[timestamp_ms, open, high, low, close, volume], ...]}`. Candles are in reverse chronological order (newest first).

Candle array indices:
- `[0]` = timestamp (ms)
- `[1]` = open
- `[2]` = high
- `[3]` = low
- `[4]` = close
- `[5]` = volume

Make one `bash` call per symbol.

### Analyze the data

Calculate from the fetched candles:

**Basic metrics:**
- Current price: latest candle close
- Period high: max of all highs
- Period low: min of all lows
- Period change: ((latest close - oldest close) / oldest close) * 100
- Average volume: mean of all volume values

**Trend analysis:**
- Direction: compare first half avg close vs second half avg close
- Volatility: (period high - period low) / period low * 100
- Volume trend: compare recent volume vs older volume
- Support/Resistance: identify clusters of lows (support) and highs (resistance)

**Key levels:**
- Identify the 2-3 most significant price levels (round numbers, multiple touches)

### Notify user with analysis

Call `notify_user` with:

- **title**: String, (max 50 chars).
- **message**: String, Required. Plain text price analysis (max 1024 chars). No markdown.

Replace placeholders in the message with calculated values.
**Single asset format:**
```jinja
{{PAIR}} (7D Daily):
Price: {{PRICE}} | Change: {{CHANGE}}
High: {{HIGH}} | Low: {{LOW}}
Avg Vol: {{AVG_VOL}} | Vol Trend: {{VOL_TREND}}
Trend: {{TREND}}
Support: {{SUPPORT}} | Resistance: {{RESISTANCE}}
Volatility: {{VOLATILITY}}

{{ ANALYSIS_NOTES }}
```

**Multiple assets format:**

```jinja
7D Daily Summary:
BTC: ${{BTC_PRICE}} ({{BTC_CHANGE}}) H:${{BTC_HIGH}} L:${{BTC_LOW}} - {{BTC_TREND}}
ETH: ${{ETH_PRICE}} ({{ETH_CHANGE}}) H:${{ETH_HIGH}} L:${{ETH_LOW}} - {{ETH_TREND}}
SOL: ${{SOL_PRICE}} ({{SOL_CHANGE}}) H:${{SOL_HIGH}} L:${{SOL_LOW}} - {{SOL_TREND}}

{{ANALYSIS_NOTES}}
```

## Constraints

- Only use the BlockVault kline API via `bash` — never invent price data.
- Maximum limit per request: 50 candles.
- Maximum 5 symbols per analysis.
- If the API fails for a symbol, note it as unavailable.
