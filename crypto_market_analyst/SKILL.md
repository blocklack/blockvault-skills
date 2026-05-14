---
name: pair-market-analyst
description: Deep-dive analysis of a specific trading pair (e.g., BTC, ETH, SOL). Use this when the user asks to analyze a specific coin, token, or pair — NOT for general market scanning or discovering trending tokens.
metadata:
  tool: get_historical_price
  category: market
  enabled: true
---

# Crypto Market Analyst

Perform a detailed analysis of a **specific trading pair** by combining historical OHLCV price data with the latest news. Generate a markdown report and save it.

This skill is for single-asset deep dives. For market-wide scanning and rankings, use the `alpha-detector` skill instead.

## Instructions

Execute all steps silently. Do NOT output internal thoughts. No exceptions. Do not omit any steps.

### Step 1: Determine parameters

Map the asset to a trading pair by appending USDT: BTC→BTCUSDT, ETH→ETHUSDT, SOL→SOLUSDT, BNB→BNBUSDT, XRP→XRPUSDT.

Choose interval and limit based on the user's timeframe:

- "last week" or default → interval: "1d", limit: 7
- "today" → interval: "1h", limit: 7

### Step 2: Fetch price data

Call `run_js` with:

- **function**: "get_historical_price"
- **data**: JSON string with:
  - **pair**: String, Required. Trading pair (e.g., "BTCUSDT").
  - **interval**: String, Required. Candle interval: "1h", "4h", "1d", "1w".
  - **limit**: Number, Optional. Number of candles (default: 7, max: 7).

### Step 3: Search latest news

Use the `web_search` tool directly:

- **query**: Search query (e.g., "Bitcoin crypto news 2026").
- **backend**: "news" for market and current events.
- **max_results**: Number of results (default: 5).

You can perform up to 3 searches with different queries to get a variety of news sources and angles.
One web search is usually sufficient, but if the results are poor, you can try alternative queries.

### Step 4: Analyze the data

Combine the price data and news to produce a complete analysis:

**From price data:**

- **Trend**: Is price moving up, down, or sideways?
- **Support levels**: Prices where the asset repeatedly bounced or consolidated.
- **Resistance levels**: Prices where the asset repeatedly reversed downward.
- **Range**: The high and low of the period.
- **Volatility**: How much did the price fluctuate?
- **Current price**: The last close price.
- **Key levels**: Any significant price points based on the candles (e.g., gaps, long wicks).

**From news:**

- **Sentiment**: Are the headlines bullish, bearish, or neutral for this asset?
- **Key events**: What recent events (regulatory, adoption, partnerships) could impact price?
- **Catalysts**: Any upcoming events mentioned that could move the market?
- **Overall outlook**: Based on the news sentiment and key events, is the short-term outlook positive, negative, or uncertain?
- **Notable quotes**: Any impactful quotes from industry leaders or analysts in the news?

### Step 5: Save the report

Build a markdown report and save it using the `text_editor` tool directly. The template below is **Jinja2** — substitute `{{ var }}` with concrete values, expand `{% for %}` over the news items, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output.

**file path**: `"reports/crypto-market-analysis-{{ pair }}-{{ current_date }}.md"`

**Report template**:

```jinja
# {{ pair | upper }} Market Report

**Period**: {{ period_from }} to {{ period_to }} ({{ interval }})
**Generated**: {{ current_date }}

## Price Summary

- **Trend**: {{ trend }}  {# up / down / sideways #}
- **Range**: ${{ low }} – ${{ high }}
- **Current**: ${{ last_close }}
- **Change**: {{ change_pct }}%  {# percentage change over the period #}

## Support & Resistance

| Level | Type | Price |
|-------|------|-------|
{% for lvl in levels %}
| {{ lvl.label }} | {{ lvl.type }} | ${{ lvl.price }} |  {# label e.g. S1/R1, type ∈ Support / Resistance #}
{% endfor %}

## Technical Analysis

{{ technical_analysis }}  {# 4-5 sentences analyzing price action, trend strength and key levels #}

## Latest News

{% if news_items %}
{% for n in news_items %}
- **{{ n.title }}**: {{ n.summary }}
{% endfor %}
{% else %}
News unavailable.
{% endif %}

## News Sentiment

- **Overall Sentiment**: {{ sentiment }}  {# Bullish / Bearish / Neutral #}
- **Key catalyst**: {{ key_catalyst }}

## Outlook

{{ outlook }}  {# short-term outlook combining technical analysis with news sentiment #}
```

### Step 6: Respond

Tell the user the report was saved and give a brief verbal summary (2-3 sentences max). Do not output the raw data or the full markdown content.

## Constraints

- Only use the specified trading pairs (BTCUSDT, ETHUSDT, SOLUSDT, BNBUSDT, XRPUSDT).
- For timeframes, use "1d" interval with a limit of 7 for "last week" or default, and "1h" interval with a limit of 7 for "today".
- Do not output raw price data or full report content. Analyze and present insights.
- Always save the report before responding.
- If news search fails or returns no results, still generate the report with price data only and note "News unavailable" in the News section.
