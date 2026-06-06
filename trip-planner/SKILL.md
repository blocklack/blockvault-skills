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

## Instructions

Identify the user's trip planning intent and gather the necessary details to search for flights. and use the instructions in this skill to meet the user's needs.

  - **Gather trip details**: how to gather and compute the necessary parameters for the flight search.
  - **Resolve IATA codes for the airports**: how to reliably get the correct 3-letter codes for the origin and destination based on user input.
  - **Search outbound flight**: how to call the search API for the initial flight options, and how to render them in a user-friendly way that surfaces all relevant details.
  - **Search return flight**: if it's a round trip, how to search for return flights based on the user's outbound choice and how to render those options as well.
  - **Save final booking**: how to generate the final booking link and save a summary file with all the trip details.

Follow these guidelines throughout the process:
  - **One category at a time.** Wait for user input before moving on.
  - **Base params on every curl**, even when adding a token. Never drop them.
  - **Never echo tokens in chat.** Tokens are used internally to build the next request and the final booking URL.
  - **Never fabricate tokens, IATA codes, prices, dates, or URLs.** All of those come from the user or from API responses; if a value is missing, run the missing search again or ask the user.
  - **Booking link only after the user confirms.**
  - For the currency use the 3-letter code (e.g. `USD`, `EUR`) — never the symbol — so it works reliably across all APIs and locales.
  - For the language, identify the user language and use the 2-letter code (e.g. `en`, `es`) that SerpAPI expects in the `hl` param.


### Gather trip details

Ask the user for: 
 - origin
 - destination
 - outbound date
 - return date (if round trip)
 - number of nights (if one-way)
 - number of adults
 - budget
 - currency

Compute `nights` yourself when you already know the dates:

- **Round trip:** `nights` = `return_date` − `outbound_date` (in days).
- **One way:** use the value the user gave; if absent, ask only when needed for the itinerary.

### Resolve IATA codes

Never invent IATA codes. Call `web_search` with:

- **query**: String, Required. `IATA code <city or airport name> airport`.
- **backend**: String, Optional. `"text"` (default).
- **limit**: Number, Optional. `3`.

Pick the 3-letter primary airport code. If a city has multiple, ask the user. Use these codes verbatim in every flight curl.

### Search outbound flight

For round trips:
  - First search for outbound flights with the gathered params.
  - After the user picks an outbound option, search for return flights using the saved `departure_token` from the chosen outbound flight, and present those options to the user as well.
  - After the user picks the return flight, save the `booking_token` from that search — **not** the outbound one, which doesn't include the return and will cause a 404 at booking time.
  
For one-way trips:
  - Search once with the gathered params, but add `"type":"2"` to `extra_params` to specify it's a one-way search and get the correct booking token.

Always present the user with a list of options to choose from, rendered with the provided template that surfaces all relevant details.
Never choose a flight for the user, or show them a booking link, before they pick an option and confirm they're ready to book.

```bash
# Round trip
curl -sS -X POST https://402.blockvault.ai/api/v1/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"q":"flights","engine":"google_flights","hl":"<hl>","extra_params":{"departure_id":"<IATA>","arrival_id":"<IATA>","outbound_date":"<YYYY-MM-DD>","return_date":"<YYYY-MM-DD>","currency":"<CUR>"}}'

# One way (omit return_date, add type:"2")
curl -sS -X POST https://402.blockvault.ai/api/v1/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"q":"flights","engine":"google_flights","hl":"<hl>","extra_params":{"departure_id":"<IATA>","arrival_id":"<IATA>","outbound_date":"<YYYY-MM-DD>","currency":"<CUR>","type":"2"}}'
```

Render `raw_data.best_flights[]` with the next template. Surface every useful signal SerpAPI returns: per-segment amenities (`extensions`), overnight markers, often-delayed warnings, codeshare (`plane_and_crew_by`, `ticket_also_sold_by`), layover details between segments, carbon emissions vs. the typical route, and the price-vs-typical hint when present.

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

- **Round trip:** `best_flights[N].departure_token` (needed for the return search) and a human summary of the outbound flight.
- **One way:** `best_flights[N].booking_token` (used directly in the final booking link) and the outbound summary.

### Search return flight (round trip only)

Skip this section entirely if `return_date` is empty (one-way trip). Otherwise, reuse the same flight base params (including top-level `hl`) **plus** the saved `departure_token`:

```bash
curl -sS -X POST https://402.blockvault.ai/api/v1/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{DELEGATE_JWT}}" \
  -d '{"q":"flights","engine":"google_flights","hl":"<hl>","extra_params":{"departure_id":"<IATA>","arrival_id":"<IATA>","outbound_date":"<YYYY-MM-DD>","return_date":"<YYYY-MM-DD>","currency":"<CUR>","departure_token":"<departure_token>"}}'
```

Render the new `best_flights[]` with the same template. User picks N. Keep in memory the return summary, the final `total_flight_price`, and `best_flights[N].booking_token` (used in the final booking link).


Ask: **"Ready to book?"** The user can also say *"change flights"* to revisit the flight phase.

### Save final booking

Only after the user confirms. Verify the `booking_token` is present in memory; if it is not, stop and re-run the relevant flight search to obtain it. Never fabricate a token.
Always add the booking link at the end of the markdown file.

`text_editor create trips/<destination-slug>-<outbound_date>.md` with the next template (verbatim):

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

- total flight price = sum of outbound + return (if round trip) — use the `price` field from the chosen flights, not the `total_flight_price` that SerpAPI returns, since that one may include fees and taxes that aren't in the booking token price and can cause confusion at checkout.
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

The booking URL must be emitted on a **single line** (no whitespace between query parameters) — otherwise the markdown link breaks.
After saving, **render the full content of the file inline in the chat** so the user sees the booking summary and link without having to open the file. Then mention the file path on a separate line (e.g. `Saved to trips/<destination-slug>-<outbound_date>.md`) so they know where to find it later.

### Insufficient credits (402)

When any search curl returns HTTP **402**, the user has no credits left. Tell them they can purchase more with:

```bash
curl -i -X POST https://402.blockvault.ai/api/v1/inference/credits \
  -H "Authorization: Bearer {{DELEGATE_JWT}}"
```

After the purchase succeeds (HTTP 200), retry the failed search automatically. If the purchase also fails, stop and report the error to the user.

## Constraints

- **No state file during the search phase.** Keep flight options, tokens, summaries, and totals in conversation memory. The only file ever written is the final booking summary in the *Save final booking* step.
- **Never invent dates.** `outbound_date` and `return_date` come only from the user or the chosen API result. If a date is missing, ask the user — do not guess based on "today" or the season.
- **≤ 2 searches per category.** Reuse data already in memory instead of re-querying.
- Use `hl` matching the user's language so airline names come back localized.
- 401 → ask the user to re-authenticate. 402 → insufficient credits, run the credits purchase flow.