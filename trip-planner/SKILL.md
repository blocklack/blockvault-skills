---
name: trip-planner
description: Plan complete trips by searching flights, then generate a booking link.
metadata:
  tool: bash
  category: travel
  enabled: true
  secrets:
    - key: DELEGATE_JWT
      description: BlockVault delegate API session token, issued via SIWE wallet sign-in.
      siwe:
        provider: delegate
        blockchain: ethereum
---

Today's date: {{DATE}}

# Instructions

Identify the user's trip-planning intent and follow the steps below.

### Gather trip details

Spawn a sub-agent to collect parameters and resolve IATA codes via web search.

Call `spawn_subagents` with:

- **tasks**: Array, Required. One element:
  - **id**: `"trip_gather"`.
  - **objective**: `"Collect origin, destination, dates, trip type, adults, max stops, budget, and currency from the user interactively. Then resolve IATA airport codes for both airports via web search."`.
  - **output_format**: `"Markdown list — one '- key: value' per field. Keys: trip_type, origin, destination, outbound_date (YYYY-MM-DD), return_date (YYYY-MM-DD or 'none'), adults, stops (0=Any|1=Nonstop only|2=1 stop or fewer|3=2 stops or fewer), budget, currency, origin_iata, destination_iata, hl, gl."`.
  - **instructions**: Today's date is **{{DATE}}** — anchor every relative date the user gives ("tomorrow", "next Friday", "in 2 weeks") against this value, and reject past dates. Ask one field at a time, in order: trip_type → origin → destination → outbound_date → (return_date if round trip) → adults → max stops → budget → currency. **Wait for the user's answer before asking the next**; never queue two questions or interactive components at once. One tool per turn. Default `stops` to `0` (Any) if cancelled — don't re-ask. Validate date format (YYYY-MM-DD, ≥ today) and currency code. After `origin` is confirmed, `web_search` with `query: "IATA code <origin> airport"` and `max_results: 3` to extract the 3-letter primary code; if multiple major airports exist, ask the user to pick. Repeat for `destination`. Never invent IATA codes. Derive `hl` and `gl` (two-letter) from the user's locale.

When the sub-agent completes, use the returned values for all remaining steps. Compute `nights` = `return_date` − `outbound_date` (round trips only).


### Search outbound flight

**Round trip:** (1) outbound search, (2) user picks an option, (3) return search using the chosen option's `departure_token`, (4) user picks return, (5) save the **return** flight's `booking_token` — not the outbound's, which 404s.

**One way:** single search with `"type":"2"`. The chosen flight's `price` is the full one-way total.

> **⚠️ Round-trip pricing.** Google Flights prices are **per complete itinerary**. Every option's `price` in **both** searches is the full round-trip total. **Never sum outbound + return** — that double-counts. The authoritative total is the **chosen return flight's** `price`.

Always present options to the user with the template below; never auto-pick or show a booking link before the user confirms.

Always set `type` (`"1"` round, `"2"` one way), `adults`, `stops`, `hl`, `gl`, `currency`. Pass `budget` as `max_price` (integer, no currency prefix). Omit `stops` only when it is `0`. **Send `adults`, `stops`, `max_price` as JSON strings** (e.g. `"adults":"2"`) — the proxy rejects them as numbers. Omitting `adults` silently prices for 1 passenger and breaks multi-passenger bookings.

```bash
# Round trip
curl -sS -X POST https://402.blockvault.ai/api/v1/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"q":"flights","engine":"google_flights","hl":"<hl>","gl":"<gl>","extra_params":{"departure_id":"<IATA>","arrival_id":"<IATA>","outbound_date":"<YYYY-MM-DD>","return_date":"<YYYY-MM-DD>","currency":"<CUR>","type":"1","adults":"<adults>","stops":"<stops>","max_price":"<budget_number>"}}'

# One way (omit return_date, set type:"2")
curl -sS -X POST https://402.blockvault.ai/api/v1/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"q":"flights","engine":"google_flights","hl":"<hl>","gl":"<gl>","extra_params":{"departure_id":"<IATA>","arrival_id":"<IATA>","outbound_date":"<YYYY-MM-DD>","currency":"<CUR>","type":"2","adults":"<adults>","stops":"<stops>","max_price":"<budget_number>"}}'
```

If both `best_flights` and `other_flights` come back empty, filters are too tight: drop `max_price`, then relax `stops` to `0`, retry once (≤ 2 searches per category). Render `raw_data.best_flights[]` with the template. Surface every useful signal: `extensions`, overnight markers, often-delayed warnings, codeshare (`plane_and_crew_by`, `ticket_also_sold_by`), layovers, carbon vs typical, price-vs-typical hint.


````jinja
{#
  Brief summary of the flights below, if is outbound or return flights if is a round trip
#}

## ✈️ {{ origin }} → {{ destination }}{% if price_insights %} · 💡 typical price {{ currency }} {{ price_insights.typical_price_range[0] }}–{{ price_insights.typical_price_range[1] }}{% endif %}

{% for f in best_flights[:5] %}
  {%- set hours = f.total_duration // 60 -%}
  {%- set minutes = f.total_duration % 60 -%}
  {%- set stops = f.layovers | length -%}
### {{ loop.index }}. {{ currency }} {{ f.price }} — {{ hours }}h {{ minutes }}m{% if stops == 0 %} · 🟢 Direct{% else %} · 🔁 {{ stops }} stop{{ "s" if stops > 1 }}{% endif %}{% if f.type %} · {{ f.type }}{% endif %}

{%- if f.carbon_emissions %}
  {%- set kg = (f.carbon_emissions.this_flight / 1000) | round(0, "floor") | int -%}
  {%- set typical = (f.carbon_emissions.typical_for_this_route / 1000) | round(0, "floor") | int -%}
  {%- set diff = f.carbon_emissions.difference_percent -%}
> 🌱 {{ kg }} kg CO₂ ({{ "+" if diff > 0 }}{{ diff }}% vs typical {{ typical }} kg)
{% endif %}

{% for seg in f.flights %}
- <img src="{{ seg.airline_logo }}" width="32" height="32" /> **{{ seg.airline }}** · {{ seg.flight_number }}{% if seg.travel_class %} · 🎟️ {{ seg.travel_class }}{% endif %}{% if seg.overnight %} · 🌙 overnight{% endif %}{% if seg.often_delayed_by_over_30_min %} · ⚠️ often delayed 30m+{% endif %}
  - 🛫 **{{ seg.departure_airport.time }}** — {{ seg.departure_airport.name }} ({{ seg.departure_airport.id }})
  - 🛬 **{{ seg.arrival_airport.time }}** — {{ seg.arrival_airport.name }} ({{ seg.arrival_airport.id }})
  - ⏱ {{ seg.duration // 60 }}h {{ seg.duration % 60 }}m · ✈️ {{ seg.airplane }}{% if seg.legroom %} · 📏 {{ seg.legroom }}{% endif %}
  {%- if seg.plane_and_crew_by %}
  - 🤝 Operated by {{ seg.plane_and_crew_by }}
  {%- endif %}
  {%- if seg.ticket_also_sold_by %}
  - 🏷️ Also sold by: {{ seg.ticket_also_sold_by | join(", ") }}
  {%- endif %}
  {%- if seg.extensions %}
  - 🛎️ {{ seg.extensions | join(" · ") }}
  {%- endif %}
{%- if not loop.last and f.layovers and f.layovers[loop.index0] %}
  {%- set lay = f.layovers[loop.index0] -%}
  - ⏸️ **Layover** {{ lay.duration // 60 }}h {{ lay.duration % 60 }}m at {{ lay.name }} ({{ lay.id }}){% if lay.overnight %} · 🌙 overnight{% endif %}
{%- endif %}
{% endfor %}

---
{% endfor %}
````

User picks N. Keep in memory:

- **Round trip:** `best_flights[N].departure_token` and a summary of the outbound flight. Do **not** record this `price` as the total — the real total comes from the return search.
- **One way:** `best_flights[N].booking_token`, the flight summary, and `best_flights[N].price` as the final total.

### Search return flight (round trip only)

Skip if `return_date` is empty. Otherwise reuse the same base params plus the saved `departure_token`:

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"q":"flights","engine":"google_flights","hl":"<hl>","gl":"<gl>","extra_params":{"departure_id":"<IATA>","arrival_id":"<IATA>","outbound_date":"<YYYY-MM-DD>","return_date":"<YYYY-MM-DD>","currency":"<CUR>","type":"1","adults":"<adults>","departure_token":"<departure_token>"}}'
```

Keep `adults`, `type`, `hl`, `gl`, `currency` identical to the outbound call. Render `best_flights[]` with the same template. **Each option's `price` is the complete round-trip total.** User picks N. Keep:
- the return flight summary,
- `best_flights[N].booking_token`,
- **`best_flights[N].price` as the final round-trip total** — never add it to the outbound price.

Ask: **"Ready to book?"** The user can also say *"change flights"* to revisit the search.

### Save final booking

Only after the user confirms. Verify `booking_token` is present; if not, re-run the search. Never fabricate a token. Always emit the booking link at the end of the file.

`text_editor create trips/<destination-slug>-<outbound_date>.md` with the template (verbatim):

````jinja
# 🎉 {{ origin }} → {{ destination }}

> 📅 {{ outbound_date }}
{% if return_date %} 
  → {{ return_date }} · {{ nights }} nights
{% endif %} · {{ adults }} adult(s) · 💰 {{ budget }}

## ✈️ Flight — {{ total_flight_price }}

- 🛫 **Outbound:** {{ flight_out }}
{% if flight_return %}
  - 🛬 **Return:** {{ flight_return }}
{% endif %}


{#
For the budget summary, compute the numbers yourself (do **not** leave placeholders in the output):

- `total_flight_price` comes from a single chosen flight's `price` field — **never sum legs**:
  - **Round trip:** use the **chosen return flight's** `price`. It already encodes outbound + return as one round-trip total. Adding the outbound price on top is the classic double-count bug.
  - **One way:** use the chosen flight's `price`.
  Use the per-flight `price` rather than SerpAPI's `total_flight_price` field, since that one may include extra fees/taxes not in the booking-token price and can confuse checkout.
- `remaining` = `budget` − `total_flight_price` (strip any currency prefix, subtract, then re-add the currency). If negative, prefix with `-` and flag it (`⚠️ over budget`).

#}

## 💰 Budget Summary

| Category | Cost |
|----------|------|
| ✈️ Flights | {{ total_flight_price }} |
| **Total** | **{{ currency }} {{ total_flight_price }}** |
| **Remaining** | **{{ currency }} {{ remaining }}** |

{# 
  Before emitting the booking link, verify:

  1. **`booking_token` exists and is non-empty.** It must come from:
    - **Round trip:** step  (the call that included `departure_token`).
    - **One way:** step  with `"type":"2"`
    - ❌ Never use `departure_token` here. That token only encodes the outbound
      and SerpAPI will return 404 `No booking options found`.
  2. **Base params match the outbound direction.** `departure_id` / `arrival_id`
    / `outbound_date`.
#}

[**Book flight →**](https://402.blockvault.ai/api/v1/search/pay?engine=google_flights&departure_id={{ departure_id }}&arrival_id={{ arrival_id }}&outbound_date={{ outbound_date }}{% if return_date %}&return_date={{ return_date }}{% endif %}&currency={{ currency }}&hl={{ hl }}&booking_token={{ booking_token}})
---

> Prices are estimates. Verify the final amount on the booking site.
````

The booking URL must be on a **single line** (no whitespace between query parameters). After saving, **render the full file content inline in the chat** so the user sees the booking summary and link, then mention the file path on a separate line (e.g. `Saved to trips/<destination-slug>-<outbound_date>.md`).

### Insufficient credits (402)

On HTTP **402** from any search, the user has no credits. Tell them they can purchase more:

```bash
curl -i -X POST https://402.blockvault.ai/api/v1/inference/credits \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

On HTTP 200, retry the failed search automatically. If the purchase fails, stop and report the error.

### Corrupt or rejected `departure_token` (return search fails)

Most return-search failures come from a bad `departure_token`. **Stop and reason step by step before retrying**:

1. Valid source is **only** `raw_data.best_flights[N].departure_token` of the **outbound** search (`type:"1"`, no `departure_token` in the request). Never read it from `other_flights`, a return search, or a `flights[*]` segment.
2. Pass it **verbatim** — no URL-encode, no trim, no decode, no line breaks.
3. The return call must reuse the **exact same** base params (`departure_id`, `arrival_id`, `outbound_date`, `return_date`, `currency`, `hl`, `gl`, `type:"1"`, `adults`). Do **not** include `stops` or `max_price` — they are encoded in the token; re-sending them invalidates it.
4. On "invalid token" / empty result: re-run the outbound search to get a **fresh** token (they expire), then immediately re-run the return search.
5. Never substitute `booking_token` for `departure_token` or vice versa.

### Corrupt or rejected `booking_token` (404 / "No booking options found")

**Stop and reason step by step before retrying**:

1. Valid `booking_token` comes only from: **round trip** → the **return** search (the one with `departure_token`); **one way** → the search where `"type":"2"`.
2. Confirm you did **not** copy `departure_token` into `booking_token` — the #1 cause of 404.
3. Confirm `departure_id`, `arrival_id`, `outbound_date` (and `return_date` for round trips) in the booking URL match the chosen flight option.
4. If the right token still fails, treat it as expired: re-run the corresponding search and rebuild the link.
5. Never fabricate, truncate, or re-encode tokens.

## Constraints

- **No state file during search.** Keep options, tokens, summaries, totals in memory; only the final booking is written.
- **Never invent dates.** Ask if missing.
- **≤ 2 searches per category.** Reuse memory.
- Use `hl` for the user's language and `gl` for the search country.
- **Always send `adults`** as a string. Omitting it prices for 1 passenger.
- **Set `type` explicitly** — `"1"` round, `"2"` one way. `booking_token` only from the leg with the full itinerary (the `departure_token` call for round trips, or the `type:"2"` call for one way).
- 401 → ask the user to re-authenticate. 402 → run the credits purchase flow.