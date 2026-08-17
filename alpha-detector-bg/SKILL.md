---
name: alpha-detector-bg
description: Scan the entire crypto market for alpha opportunities using rankings (trending, alpha, top search, social hype, smart money, meme). Background version — generates a written findings report. Use when the user asks about market trends, what's hot, trending tokens, or wants to earn or yield using crypto assets.
metadata:
  category: market
  enabled: true
  background: true
---

# Alpha Detector (Background)

Scan the market for alpha opportunities using the BlockVault ranking API + price data + web context. Generate a written findings report as your final response — do NOT send notifications and do NOT use tables.

## Instructions

Execute all steps silently. Detect the user's language and reply in that language.

### Step 1: Fetch ranked token list

Call `bash` with:

- **command**: String, Required. A curl to the BlockVault ranking API.

Pick the endpoint based on user intent (default: trending):

```bash
#Trending tokens (default)
curl -s "https://api.blockvault.ai/api/v1/ranking/trending?limit=<limit>"
#alpha tokens
curl -s "https://api.blockvault.ai/api/v1/ranking/alpha?limit=<limit>"
#most searched tokens
curl -s "https://api.blockvault.ai/api/v1/ranking/top-search?limit=<limit>"
#social hype tokens
curl -s "https://api.blockvault.ai/api/v1/ranking/social-hype?limit=<limit>"
#smart money / whales
curl -s "https://api.blockvault.ai/api/v1/ranking/smart-money?limit=<limit>"
#meme tokens
curl -s "https://api.blockvault.ai/api/v1/ranking/meme?limit=<limit>"
```

Optional query params: `limit` (default 10), `chain_id` (`1`=ETH, `56`=BSC, `8453`=Base, `CT_501`=SOL), `period` (smart-money only: `5m`,`1h`,`4h`,`24h`).

Response: `{tokens: [{rank, symbol, price, price_change, market_cap, volume, holders, chain_id, contract_address, ...}], total}`. Extra fields vary by endpoint: `liquidity`, `inflow`+`traders`, `social_hype`+`sentiment`, `score`.

Take top 5-10 tokens. Map each symbol to a trading pair by appending USDT (e.g. BTC -> BTCUSDT).

If the API fails, fall back to: BTC, ETH, SOL, BNB, XRP.

### Step 2: Fetch OHLCV price data

For each of the top 5 symbols from Step 1, call `bash` with:

- **command**: String, Required. A curl to the kline API.

```bash
curl -s "https://api.blockvault.ai/api/v1/market/kline/<symbol>?interval=D&category=spot&limit=7"
```

Replace `<symbol>` with each symbol's pair. Use `interval=D` and `limit=7` for 7-day price history.

Candles are in reverse chronological order.

Make one `bash` call per symbol. Extract:
- Current price: `list[0][4]`
- 24h change: compare `list[0][4]` vs `list[1][4]`
- 7d change: compare `list[0][4]` vs `list[6][4]`
- Volume trend: compare recent vs older candle volumes
- Price action: breakout patterns, wicks, range expansion

### Step 3: Web search for context

Call `web_search` 2-3 times to find crypto news related to the top movers.

- **query**: String, Required. E.g. `"crypto market trends 2026 trending tokens"`.
- **backend**: String, Optional. Use `"news"` for recent events.
- **max_results**: Integer, Optional. Set to `5`.

Start broad ("crypto market trends 2026"), then narrow to specific assets showing unusual action from Steps 1-2.

### Step 4: Analyze and rank

For each asset, cross-reference rank data + OHLCV + news:

- **Price action**: trend direction, % change, breakout above/below prior range, wick patterns
- **Volume**: increasing/decreasing trend, spikes vs average
- **Catalyst**: news sentiment (bullish/bearish/neutral), catalyst strength, narrative alignment

Rate each asset:
- **Strong Alpha**: 3+ confirming signals (breakout + volume spike + bullish news)
- **Moderate Alpha**: 2 confirming signals
- **Watchlist**: 1 interesting signal, insufficient confirmation
- **No Signal**: nothing notable

Rank by conviction, not just % change.

### Step 5: Output the findings report

Do NOT use markdown tables, use plain text with short lines and simple lists.

**Report format:**
```
TOP OPPORTUNITIES:
1. TOKEN (+24h%) - Strong Alpha. Signal: breakout + volume spike. Catalyst: [news]. MCap: $X.
2. TOKEN (+24h%) - Moderate Alpha. Signal: [description].

WATCHLIST: TOKEN1, TOKEN2

SENTIMENT: Bullish/Bearish/Neutral. Dominant narrative: [theme]. Risk: Low/Medium/High.
```

Prioritize actionable information and keep the report concise.

## Constraints

- Only use the BlockVault ranking and kline APIs via `bash` — never invent data.
- Maximum 5 symbols for detailed analysis (top 5 from ranking).
- Maximum 3 `web_search` calls for context.
- Do NOT call `notify_user` — output the findings as your final response.
- Do NOT use markdown tables anywhere in the output.
- Do NOT save reports to files — this is the background version.
