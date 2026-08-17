---
name: web-research
description: Research any topic on the internet using chained web searches. Use when the user asks to investigate, research, look up, or find information about anything — news, people, companies, technology, events, crypto projects, or any general knowledge query.
metadata:
  category: tools
  enabled: true
  background: true
---

# Web Research

Perform multi-step web research by chaining `web_search` calls. Start broad, then narrow down based on findings. Deliver a structured summary via `notify_user`.

## Instructions

Execute all steps silently. Detect the user's language and reply in that language.

### Step 1: Plan search strategy

Break the user's query into 2-4 focused sub-queries. Move from general to specific.

Rules for queries:
- Use keyword-rich phrasing (3-8 keywords), not full sentences.
- Include the current year for time-sensitive topics.
- Use `backend: "news"` for recent events; `backend: "text"` for general knowledge.

### Step 2: Execute broad search

Call `web_search` with:

- **query**: String, Required. The first (broadest) search query.
- **backend**: String, Optional. `"text"` (default) or `"news"` for recent events.
- **max_results**: Integer, Optional. Set to `5`.

Read the results carefully. Identify:
- Key facts and figures
- Names, dates, entities mentioned
- Gaps in knowledge that need follow-up searches

### Step 3: Execute follow-up searches

Based on Step 2 results, call `web_search` 1-3 more times with progressively narrower queries.

- **query**: String, Required. A more specific query informed by previous results.
- **backend**: String, Optional. `"text"` or `"news"`.
- **max_results**: Integer, Optional. Set to `5`.

Adapt queries based on what you learned:
- If Step 2 revealed a key person/project, search specifically for them.
- If Step 2 was too generic, add more specific keywords.
- If looking for recent events, switch to `backend: "news"`.

Do NOT repeat the same or very similar queries. Each search must add new information.

### Step 4: Synthesize findings

Cross-reference all search results to build a coherent understanding:
- Identify consensus facts (mentioned in multiple sources)
- Note conflicting information
- Separate facts from opinions
- Establish timeline of events if relevant

### Step 5: Notify user with research summary

Call `notify_user` with:

- **title**: String, Required. Concise topic summary (max 50 chars). E.g. `"Research: Solana DeFi"`.
- **message**: String, Required. Plain text research summary (max 1024 chars). No markdown.

**Summary structure:**
1. One-sentence overview answering the user's question
2. Key findings (3-5 bullet points as plain text, use - prefix)
3. Notable data points or statistics
4. Current status / recent developments
5. One-sentence conclusion or outlook

**Example notification:**
```
title: "Research: Solana DeFi"
message: "Solana DeFi TVL has grown 40% in Q2 2026, reaching $12B. Key findings: - Jupiter leads with $3.2B TVL and launched v4 with limit orders. - Marinade Finance staking surpassed 15M SOL. - New protocol Drift expanded to cross-margin perps. - Transaction costs remain under $0.01 avg. - Ecosystem saw 3 new token launches this week. Outlook: Strong growth trajectory with institutional interest increasing."
```

## Constraints

- Use only `web_search` to gather information — never invent facts.
- Maximum 4 `web_search` calls per research session.
- Always attribute key claims to sources when possible.
- If search results are insufficient, note gaps honestly in the summary.
- Always call `notify_user` as the final action with a structured summary.
- Keep the notification message concise but informative — prioritize actionable insights.
