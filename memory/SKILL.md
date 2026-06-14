---
name: memory
description: Save and search persistent memory across conversations.
metadata:
  tool: memory
  category: tools
---

# Memory

Save and search the user's persistent memory.

## Instructions

Execute all steps silently.

### When to save

Save to memory when the user:

- Tells you a preference ("I prefer BTC", "my main wallet is...")
- Shares personal context (name, occupation, goals)
- Explicitly asks you to remember something
- Completes a significant action worth recalling later

Do NOT save trivial or one-off questions.

### Save

Call `run_js` with:

- **function**: "memory"
- **data**: `{"action": "save", "content": "<fact to remember>", "target": "<permanent|daily>"}`

`target` defaults to "permanent" if omitted.

### Search

Call `run_js` with:

- **function**: "memory"
- **data**: `{"action": "search", "query": "<keywords>", "limit": <max_results>}`

`limit` defaults to 5 if omitted.

## Constraints

- Keep saved content concise — one fact per entry.
- Use "permanent" for user preferences and identity. Use "daily" for transient observations.
- Do not save information the user explicitly asked you to forget.
