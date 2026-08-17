---
name: paper-trading-bg
description: Background paper trading simulator with virtual $10,000 USD. Buy and sell crypto with fake money and track P&L. Runs silently in background — results delivered via notification.
metadata:
  category: tools
  enabled: true
  background: true
---

# Paper Trading Simulator (Background)

Simulate crypto trading with a virtual $10,000 USD balance. Track positions and calculate P&L. Deliver results via `notify_user`.

## Instructions

Execute all steps silently. Detect the user's language and reply in that language via notifications.

### Portfolio State File

The portfolio is persisted at `paper-trading/portfolio.json`. Structure:

```json
{
  "cash": 10000,
  "positions": [
    { "symbol": "BTC", "qty": 0.05, "avg_cost": 68000, "entries": [{ "qty": 0.05, "price": 68000, "date": "2026-06-01" }] }
  ],
  "history": [
    { "date": "2026-06-01", "action": "BUY", "symbol": "BTC", "qty": 0.05, "price": 68000, "total": 3400 }
  ],
  "snapshots": [
    { "date": "2026-06-01", "equity": 10000 }
  ],
  "created": "2026-06-01"
}
```

### Step 1: Load or initialize portfolio

Call `text_editor` with:

- **command**: String, Required. Set to `"view"`.
- **path**: String, Required. Set to `"paper-trading/portfolio.json"`.

If file does not exist, call `text_editor` with:

- **command**: String, Required. Set to `"create"`.
- **path**: String, Required. Set to `"paper-trading/portfolio.json"`.
- **file_text**: String, Required. Initial portfolio JSON with `cash: 10000`, empty `positions`, `history`, `snapshots`, and `created` set to today.

### Step 2: Parse user intent

Determine the action from the user's message:

| Intent | Keywords |
|--------|----------|
| **BUY** | buy, long, comprar, add |
| **SELL** | sell, close, vender, exit |
| **STATUS** | portfolio, status, balance, P&L, estado, how am I doing |
| **RESET** | reset, restart, new game, reiniciar |

Extract: `symbol` (e.g. BTC, ETH, SOL), `amount` (USD amount or token quantity), `action`.

If amount is not specified, default to 10% of available cash for BUY, or 100% of position for SELL.

### Step 3: Fetch current prices

For each symbol needed (all portfolio positions + the symbol being traded), fetch the latest price.

If the user provides just the base symbol (BTC, ETH, SOL), append `USDT` to form the pair (e.g. `BTCUSDT`).

Call `bash` with:

- **command**: String, Required. A curl command per symbol.

```bash
curl -s "https://api.blockvault.ai/api/v1/market/kline/<SYMBOL>?interval=<TIMEFRAME>&category=spot&limit=<LIMIT>"
```

Replace `<SYMBOL>` with the trading pair (e.g. `BTCUSDT`), `<TIMEFRAME>` with the candle interval, and `<LIMIT>` with the number of candles to fetch (default `1` for current price, max `50`).

**Accepted `<TIMEFRAME>` values:**

| Value | Meaning |
|-------|---------|
| `1` | 1 minute |
| `3` | 3 minutes |
| `5` | 5 minutes |
| `15` | 15 minutes |
| `30` | 30 minutes |
| `60` | 1 hour |
| `120` | 2 hours |
| `240` | 4 hours |
| `360` | 6 hours |
| `720` | 12 hours |
| `D` | 1 day (default) |
| `W` | 1 week |
| `M` | 1 month |

The response JSON contains `list` — an array of candle arrays `[timestamp_ms, open, high, low, close, volume]`. Use `list[0][4]` (the close price) as the current price.

Make one `bash` call per symbol. Example for multiple symbols:

```bash
curl -s "https://api.blockvault.ai/api/v1/market/kline/BTCUSDT?interval=D&category=spot&limit=1"
```

```bash
curl -s "https://api.blockvault.ai/api/v1/market/kline/ETHUSDT?interval=D&category=spot&limit=1"
```

### Step 4: Execute trade

**BUY:**
1. Calculate total cost = qty x current price
2. Verify cash >= total cost. If not, notify error with max affordable qty.
3. Deduct from cash, add/update position (recalculate avg_cost if adding to existing)
4. Log to `history`

**SELL:**
1. Verify position exists and qty is sufficient. If not, notify error.
2. Calculate proceeds = qty x current price
3. Add proceeds to cash, reduce/remove position
4. Calculate realized P&L for this trade
5. Log to `history`

**RESET:**
1. Reset to initial state: cash=10000, empty positions/history/snapshots
2. Notify confirmation

### Step 5: Update snapshot and save

1. Calculate current equity = cash + sum(position.qty x current_price) for all positions
2. Append today's snapshot `{ date, equity }` (replace if today already exists)
3. Use `str_replace` to update each changed section of the portfolio file individually.

For each field that changed (cash, positions, history, snapshots), call `text_editor` with:

- **command**: String, Required. Set to `"str_replace"`.
- **path**: String, Required. Set to `"paper-trading/portfolio.json"`.
- **old_str**: String, Required. The exact current value of the field (e.g. `"cash": 6600`).
- **new_str**: String, Required. The updated value (e.g. `"cash": 3200`).

Apply one `str_replace` per changed section. For arrays like `positions` or `history`, replace the entire array value. Example:

```
old_str: "history": []
new_str: "history": [{"date":"2026-06-18","action":"BUY","symbol":"BTC","qty":0.05,"price":68000,"total":3400}]
```

### Step 6: Notify user with results

Call `notify_user` with:

- **title**: String, Required. Short summary (max 50 chars). E.g. `"Paper Trade: BUY BTC"`.
- **message**: String, Required. Plain text results summary (max 1024 chars). No markdown.
- **level**: String, Optional. Set to `"warning"` for trades, `"info"` for status.

**Trade notification format:**
- title: `"Paper Trade: <ACTION> <SYMBOL>"`
- message: `"Bought 0.05 BTC at $68,000 ($3,400). Portfolio: $6,600 cash + $3,400 BTC = $10,000 equity. P&L: $0 (0%)."`

**Status notification format:**
- title: `"Paper Portfolio"`
- message: `"Equity: $10,500 (+5%). Cash: $6,600. BTC: 0.05 ($3,500, +2.9%). ETH: 1.2 ($3,900, +8.3%)."`

## Constraints

- Starting capital is always $10,000.
- Supported symbols: any token available on Bybit with a USDT pair.
- Minimum trade: $10.
- Cannot short (sell more than owned).
- Cannot spend more than available cash.
- All prices come from the API via `bash` — never invent prices.
- The portfolio file is the single source of truth. Always read before modifying, always save after.
- When showing P&L percentages, calculate from $10,000 initial equity.
- Always call `notify_user` as the final action with a results summary.
