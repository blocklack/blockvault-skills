---
name: crypto-news-dashboard
description: Generate a visual crypto news dashboard with ECharts charts — sentiment radar, fear/greed gauge, market heatmap, and trending narratives. Use this when the user asks for a news summary, daily briefing, market sentiment overview, or "what's happening in crypto today".
metadata:
  tool: web_search
  category: market
  enabled: true
---

# Crypto News Dashboard

Generate a rich visual news dashboard combining web research with ECharts charts. Produce a sentiment radar, fear/greed gauge, sector heatmap, and categorized news digest.

## Instructions

Execute all steps silently. Do NOT output internal thoughts. No exceptions. Do not omit any steps.
Detect the user's language and reply in that language.

### Step 1: Chained web research

Call `web_search` repeatedly. Start broad, then narrow based on gaps.

```
Search 1: "crypto market news today" (backend: "news")
Search 2: "bitcoin ethereum price analysis today" (backend: "news")
Search 3: "crypto regulation DeFi news" (backend: "news")
Search 4: "<topic from gaps>" (backend: "news")
Search 5: "crypto fear greed index" (backend: "text")
```

Minimum 4 searches, maximum 7. Cover these dimensions:
- **Bitcoin & macro** — BTC dominance, ETF flows, institutional moves
- **Altcoins & DeFi** — major alt movers, DeFi TVL, protocol updates
- **Regulation** — government actions, SEC/CFTC, global policy
- **Narratives** — trending sectors (AI, RWA, memes, L2s, etc.)
- **Risk events** — hacks, exploits, depegs, exchange issues

### Step 2: Fetch price data for context

Call `run_js` with:
- **function**: `"get_historical_prices"`
- **data**: `{"symbols": ["BTCUSDT", "ETHUSDT", "SOLUSDT", "BNBUSDT", "XRPUSDT", "DOGEUSDT", "ADAUSDT", "AVAXUSDT", "LINKUSDT", "DOTUSDT", "MATICUSDT", "UNIUSDT", "AAVEUSDT", "NEARUSDT", "SUIUSDT"], "interval": "1d", "limit": 7}`

Use the price changes to contextualize the news and build the treemap.

### Step 3: Analyze and categorize

From all research, extract:

**News items** — Categorize each into:
- `bitcoin` — BTC-specific: price, ETFs, mining, halvings, dominance
- `ethereum` — ETH, L2s, staking, EIPs, gas
- `alt_l1` — SOL, AVAX, SUI, NEAR, DOT, ADA and other L1s
- `defi` — Protocols, yields, TVL, hacks, stablecoins
- `memes_nft` — Meme coins, NFTs, gaming, culture
- `regulation` — Government, legal, compliance, CBDCs

**Sentiment scores** — Rate each category 0-100:
- 0-20: Very Bearish
- 21-40: Bearish
- 41-60: Neutral
- 61-80: Bullish
- 81-100: Very Bullish

**Overall fear/greed** — Single score 0-100 based on all signals.

**Token sector mapping** — Assign each token to a sector for the treemap:

| Sector | Tokens |
|--------|--------|
| Store of Value | BTC |
| Smart Contracts | ETH, SOL, ADA, AVAX, NEAR, SUI, DOT |
| DeFi | UNI, AAVE, LINK |
| Exchange / Infra | BNB |
| Payments | XRP, MATIC |
| Memes | DOGE |

### Step 4: Generate dashboard

Present the full dashboard directly in your response using the template below.
The template is **Jinja2** — substitute `{{ var }}` with concrete values, expand `{% for %}` loops, resolve `{% if %}` blocks. Drop `{# comments #}` from output.

Output the complete dashboard as your message:

````jinja
# Crypto News Dashboard

**Date**: {{ current_date }} | **Coverage**: Last 24h

## Market Sentiment Radar

```echarts
{
  "backgroundColor": "transparent",
  "radar": {
    "indicator": [
      { "name": "Bitcoin", "max": 100 },
      { "name": "Ethereum", "max": 100 },
      { "name": "Alt L1s", "max": 100 },
      { "name": "DeFi", "max": 100 },
      { "name": "Memes/NFT", "max": 100 },
      { "name": "Regulation", "max": 100 }
    ],
    "shape": "polygon",
    "splitNumber": 5,
    "axisName": { "color": "#ccc", "fontSize": 11 },
    "splitLine": { "lineStyle": { "color": "#333" } },
    "splitArea": { "areaStyle": { "color": ["transparent"] } },
    "axisLine": { "lineStyle": { "color": "#444" } }
  },
  "series": [{
    "type": "radar",
    "data": [{
      "value": [{{ bitcoin_score }}, {{ ethereum_score }}, {{ alt_l1_score }}, {{ defi_score }}, {{ memes_nft_score }}, {{ regulation_score }}],
      "name": "Sentiment",
      "areaStyle": { "color": "rgba(0, 255, 148, 0.15)" },
      "lineStyle": { "color": "#00ff94", "width": 2 },
      "itemStyle": { "color": "#00ff94" }
    }]
  }]
}
```

## Fear & Greed Index

```echarts
{
  "backgroundColor": "transparent",
  "series": [{
    "type": "gauge",
    "min": 0,
    "max": 100,
    "startAngle": 180,
    "endAngle": 0,
    "data": [{ "value": {{ fear_greed_score }}, "name": "{{ fear_greed_label }}" }],
    "axisLine": {
      "lineStyle": {
        "width": 18,
        "color": [
          [0.2, "#ff4d4d"],
          [0.4, "#ff7043"],
          [0.6, "#ffd740"],
          [0.8, "#66bb6a"],
          [1, "#00ff94"]
        ]
      }
    },
    "pointer": { "width": 4, "length": "60%", "itemStyle": { "color": "#fff" } },
    "axisTick": { "show": false },
    "splitLine": { "show": false },
    "axisLabel": { "show": false },
    "detail": { "formatter": "{value}", "fontSize": 20, "color": "#fff", "offsetCenter": [0, "20%"] },
    "title": { "fontSize": 12, "color": "#999", "offsetCenter": [0, "45%"] }
  }]
}
```

## Market Treemap

```echarts
{
  "backgroundColor": "transparent",
  "tooltip": { "formatter": "{b}: {c}%" },
  "series": [{
    "type": "treemap",
    "left": 0,
    "top": 0,
    "right": 0,
    "bottom": 0,
    "roam": false,
    "nodeClick": false,
    "breadcrumb": { "show": false },
    "label": { "show": true, "fontSize": 11, "color": "#fff" },
    "levels": [
      {
        "itemStyle": { "borderColor": "#111", "borderWidth": 2, "gapWidth": 2 }
      },
      {
        "itemStyle": { "borderColor": "#222", "borderWidth": 1, "gapWidth": 1 },
        "upperLabel": { "show": true, "height": 20, "color": "#ccc", "fontSize": 10, "backgroundColor": "#1a1a1a", "padding": [2, 4] }
      }
    ],
    "data": [
{% for sector in treemap_sectors -%}
      {
        "name": "{{ sector.name }}",
        "children": [
{% for token in sector.tokens -%}
          { "name": "{{ token.symbol }}\n{{ token.change }}%", "value": {{ token.market_cap_rank }}, "itemStyle": { "color": "{{ '#00c853' if token.change >= 5 else '#00ff94' if token.change >= 0 else '#ff7043' if token.change >= -5 else '#ff4d4d' }}" } }{{ "," if not loop.last else "" }}
{% endfor %}
        ]
      }{{ "," if not loop.last else "" }}
{% endfor %}
    ]
  }]
}
```

## Top Stories

### � Bitcoin & Macro

{% for n in bitcoin_news -%}
- **{{ n.title }}** — {{ n.summary }} *({{ n.sentiment }})*
{% endfor %}

### 🔵 Ethereum & L2s

{% for n in ethereum_news -%}
- **{{ n.title }}** — {{ n.summary }} *({{ n.sentiment }})*
{% endfor %}

### 🟣 Alt L1s

{% for n in alt_l1_news -%}
- **{{ n.title }}** — {{ n.summary }} *({{ n.sentiment }})*
{% endfor %}

### 🟢 DeFi

{% for n in defi_news -%}
- **{{ n.title }}** — {{ n.summary }} *({{ n.sentiment }})*
{% endfor %}

### ⚖️ Regulation

{% for n in regulation_news -%}
- **{{ n.title }}** — {{ n.summary }} *({{ n.sentiment }})*
{% endfor %}

{% if memes_nft_news %}
### 🐸 Memes & NFT

{% for n in memes_nft_news -%}
- **{{ n.title }}** — {{ n.summary }} *({{ n.sentiment }})*
{% endfor %}
{% endif %}

## Price Snapshot

| Asset | Price | 24h | 7d | Trend |
|-------|-------|-----|-----|-------|
{% for p in prices -%}
| **{{ p.symbol }}** | ${{ p.price }} | {{ p.change_24h }}% | {{ p.change_7d }}% | {{ p.trend }} |
{% endfor %}

## Key Takeaways

{{ takeaways }}
````

## ECharts Data Formatting Rules

1. **Radar chart**: All scores are integers 0-100. Array order must match the indicator order: `[bitcoin, ethereum, alt_l1, defi, memes_nft, regulation]`.
2. **Gauge chart**: `fear_greed_score` is integer 0-100. Label: "Extreme Fear" (0-20), "Fear" (21-40), "Neutral" (41-60), "Greed" (61-80), "Extreme Greed" (81-100).
3. **Treemap**: Each sector has a `name` and `tokens` array. Each token has `symbol` (clean ticker WITHOUT "USDT" suffix — e.g. "BTC" not "BTCUSDT"), `change` (24h % as float), and `market_cap_rank` (integer, larger = bigger box; use 10 for BTC, 8 for ETH, 5 for mid-caps, 3 for small). Color rules: ≥5% → `#00c853`, ≥0% → `#00ff94`, ≥-5% → `#ff7043`, <-5% → `#ff4d4d`. Sectors with no tokens should be omitted. The default label shows `{b}` (name) — embed the change % in the token name (e.g. `"BTC\n+2.5%"`).
4. All numbers must be raw (no quotes, no $ signs) in JSON.
5. Remove all Jinja syntax from final output — expand into concrete values.

## Constraints

- Minimum 4 web searches, chain them — each result informs the next query.
- If news search returns no results, still present the dashboard with available data and note gaps.
- The ECharts JSON must be valid — no trailing commas, matching brackets.
- Do not invent news. All items must come from search results.
- Respond with the full dashboard directly in chat. Do NOT save a file unless explicitly asked.
- Trend column uses emoji: 📈 (up), 📉 (down), ➡️ (flat).
- Sentiment labels in parentheses: *(Bullish)*, *(Bearish)*, *(Neutral)*.
