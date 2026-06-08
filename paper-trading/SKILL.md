---
name: paper-trading
description: Paper trading simulator with virtual $10,000 USD. Buy and sell crypto with fake money, track P&L, and visualize portfolio performance. Use when the user wants to practice trading, simulate trades, check paper portfolio, or play a trading game.
metadata:
  tool: get_historical_prices
  category: tools
  enabled: true
---

# Paper Trading Simulator

Simulate crypto trading with a virtual $10,000 USD balance. Track positions, calculate P&L, and generate visual portfolio reports with ECharts.

## Instructions

Execute all steps silently. Do NOT output internal thoughts. No exceptions.
Detect the user's language and reply in that language.

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

Use `text_editor` to read `paper-trading/portfolio.json`.

- If file exists → parse it and continue.
- If file does not exist → create it with `cash: 10000`, empty `positions`, `history`, and `snapshots`. Set `created` to today.

### Step 2: Parse user intent

Determine the action from the user's message:

| Intent | Keywords |
|--------|----------|
| **BUY** | buy, long, comprar, add |
| **SELL** | sell, close, vender, exit |
| **STATUS** | portfolio, status, balance, P&L, estado, how am I doing |
| **RESET** | reset, restart, new game, reiniciar |
| **LEADERBOARD** | history, trades, historial |

Extract: `symbol` (e.g. BTC, ETH, SOL), `amount` (USD amount or token quantity), `action`.

If amount is not specified, default to 10% of available cash for BUY, or 100% of position for SELL.

### Step 3: Fetch current prices

Call `run_js` with:
- **function**: `"get_historical_prices"`
- **data**: `{"symbols": ["BTCUSDT", "ETHUSDT", ...], "interval": "1d", "limit": 7}`

Fetch prices for ALL symbols currently in the portfolio plus the symbol being traded. Use the latest close as the current price.

### Step 4: Execute trade

**BUY:**
1. Calculate total cost = qty × current price
2. Verify cash >= total cost. If not, show error with max affordable qty.
3. Deduct from cash, add/update position (recalculate avg_cost if adding to existing)
4. Log to `history`

**SELL:**
1. Verify position exists and qty is sufficient. If not, show error.
2. Calculate proceeds = qty × current price
3. Add proceeds to cash, reduce/remove position
4. Calculate realized P&L for this trade
5. Log to `history`

**RESET:**
1. Reset to initial state: cash=10000, empty positions/history/snapshots
2. Confirm to user

### Step 5: Update snapshot and save

1. Calculate current equity = cash + sum(position.qty × current_price) for all positions
2. Append today's snapshot `{ date, equity }` (replace if today already exists)
3. Save the full portfolio state using `text_editor` to `paper-trading/portfolio.json`

### Step 6: Render dashboard

Present the dashboard using the template below.
The template is **Jinja2** — substitute `{{ var }}` with concrete values, expand `{% for %}` loops, resolve `{% if %}` blocks. Drop `{# comments #}` from output.

````jinja
# Paper Trading Dashboard

**Balance**: ${{ cash }} | **Equity**: ${{ equity }} | **Total P&L**: {{ total_pnl_pct }}% (${{ total_pnl }}) | **Started**: {{ created }}

{% if action == "BUY" or action == "SELL" %}
## Trade Executed

| | Detail |
|---|---|
| **Action** | {{ action }} |
| **Symbol** | {{ symbol }} |
| **Qty** | {{ qty }} |
| **Price** | ${{ price }} |
| **Total** | ${{ trade_total }} |
{% if action == "SELL" %}| **Realized P&L** | ${{ realized_pnl }} ({{ realized_pnl_pct }}%) |{% endif %}

{% endif %}

## Portfolio Performance

```echarts
{
  "backgroundColor": "transparent",
  "tooltip": { "trigger": "axis" },
  "xAxis": {
    "type": "category",
    "data": [{% for s in snapshots %}"{{ s.date }}"{{ "," if not loop.last else "" }}{% endfor %}],
    "axisLabel": { "color": "#999", "fontSize": 10 },
    "axisLine": { "lineStyle": { "color": "#333" } }
  },
  "yAxis": {
    "type": "value",
    "axisLabel": { "color": "#999", "fontSize": 10, "formatter": "${value}" },
    "axisLine": { "show": false },
    "splitLine": { "lineStyle": { "color": "#222" } }
  },
  "series": [{
    "type": "line",
    "data": [{% for s in snapshots %}{{ s.equity }}{{ "," if not loop.last else "" }}{% endfor %}],
    "smooth": true,
    "lineStyle": { "color": "{{ '#00ff94' if total_pnl >= 0 else '#ff4d4d' }}", "width": 2 },
    "areaStyle": {
      "color": {
        "type": "linear",
        "x": 0, "y": 0, "x2": 0, "y2": 1,
        "colorStops": [
          { "offset": 0, "color": "{{ 'rgba(0,255,148,0.3)' if total_pnl >= 0 else 'rgba(255,77,77,0.3)' }}" },
          { "offset": 1, "color": "transparent" }
        ]
      }
    },
    "itemStyle": { "color": "{{ '#00ff94' if total_pnl >= 0 else '#ff4d4d' }}" },
    "markLine": {
      "silent": true,
      "data": [{ "yAxis": 10000, "label": { "formatter": "Start $10k", "color": "#666", "fontSize": 10 }, "lineStyle": { "color": "#444", "type": "dashed" } }]
    }
  }]
}
```

## Positions

```echarts
{
  "backgroundColor": "transparent",
  "tooltip": { "trigger": "item", "formatter": "{b}: ${c} ({d}%)" },
  "series": [{
    "type": "pie",
    "radius": ["40%", "70%"],
    "center": ["50%", "50%"],
    "label": { "color": "#ccc", "fontSize": 11 },
    "data": [
      { "name": "Cash", "value": {{ cash }}, "itemStyle": { "color": "#666" } }{{ "," if positions else "" }}
{% for p in positions -%}
      { "name": "{{ p.symbol }}", "value": {{ p.value }}, "itemStyle": { "color": "{{ p.color }}" } }{{ "," if not loop.last else "" }}
{% endfor %}
    ]
  }]
}
```

| Asset | Qty | Avg Cost | Current | Value | P&L | % |
|-------|-----|----------|---------|-------|-----|---|
{% for p in positions -%}
| **{{ p.symbol }}** | {{ p.qty }} | ${{ p.avg_cost }} | ${{ p.current_price }} | ${{ p.value }} | ${{ p.pnl }} | {{ p.pnl_pct }}% |
{% endfor %}

{% if history %}
## Recent Trades

| Date | Action | Symbol | Qty | Price | Total |
|------|--------|--------|-----|-------|-------|
{% for h in history[-5:] -%}
| {{ h.date }} | {{ h.action }} | {{ h.symbol }} | {{ h.qty }} | ${{ h.price }} | ${{ h.total }} |
{% endfor %}
{% endif %}
````

## ECharts Data Formatting Rules

1. **Equity line chart**: X axis = snapshot dates (strings). Y axis = equity values (numbers). Line color: green (`#00ff94`) if P&L >= 0, red (`#ff4d4d`) if negative. The $10,000 starting line is always shown as a dashed markLine.
2. **Pie chart**: Shows portfolio allocation. Cash is always gray (`#666`). Position colors cycle through: `#00ff94`, `#00bcd4`, `#ff9800`, `#e040fb`, `#ffd740`, `#ff5252`. Value = qty × current_price.
3. All numbers must be raw (no quotes, no $ signs) in JSON.
4. Remove all Jinja syntax from final output — expand into concrete values.
5. If there are fewer than 2 snapshots, show only the pie chart (skip the line chart).

## Constraints

- Starting capital is always $10,000.
- Supported symbols: any token available on Binance with a USDT pair.
- Minimum trade: $10.
- Cannot short (sell more than owned).
- Cannot spend more than available cash.
- All prices come from `get_historical_prices` — never invent prices.
- The portfolio file is the single source of truth. Always read before modifying, always save after.
- When showing P&L percentages, calculate from $10,000 initial equity.
- Respond with the dashboard directly in chat. Do NOT require the user to ask for status after a trade — always show it.
