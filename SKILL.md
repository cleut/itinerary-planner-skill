---
name: itinerary-planner
description: Build high-quality travel itineraries using live web research in a dedicated browser session. Focuses on day-by-day plans, pacing, activities, and context — with Google Maps-based walking/transit sanity checks (no full transport planning, no accommodations).
metadata:
  openclaw:
    emoji: 🗺️
---

# Itinerary Planner

**Great itineraries come from current, real research — not generic templates.**

This skill designs **day-by-day travel itineraries** grounded in live information, local context, and user preferences.  
It intentionally excludes **accommodation selection** and **detailed transport planning**, but it **must** sanity-check feasibility using Google Maps (walk/transit) to avoid unrealistic day plans.

---

## Scope

### This skill DOES:
- Design daily itineraries (what to do, when, and why)
- Balance pacing (busy vs relaxed)
- Research attractions, neighborhoods, food areas, restaurants/cafés, viewpoints, nature, culture
- Account for weather, seasonality, and local events
- Suggest alternatives (bad weather / low-energy days)
- Produce clean, structured outputs (First in chat, in Notion when asked)
- **Validate intra-day movement feasibility using Google Maps** (walking/transit time checks + obvious route sanity)

### This skill DOES NOT:
- Select or compare accommodations
- Plan intercity transport (flights/trains/buses) end-to-end
- Provide step-by-step navigation instructions
- Book anything

---

## Reference Files

### `/references/wishlist.md`
Persistent travel preferences and constraints.

The agent must:
- Read it at the start of every session
- Apply it to activity choice, pacing, and structure
- Append durable preferences when they emerge (e.g. “prefer mornings free”, “avoid overly touristy attractions”)

---

## Inputs

Collect only what’s missing:
- Destination(s)
- Date range
- Departure datetime from origin (home/other city) and origin city/airport
- Departure time from destination (or departure flight/train time)
- Accommodation location(s) (address or neighborhood) for context only
- Already prebooked activities with date and duration
- Travelers (count, adults/children)
- Travel style (slow, packed, relaxed, work-friendly)
- Interests (food, culture, nature, nightlife, photography, etc.)
- Constraints (work blocks, weather sensitivity)

Hard gate before final itinerary output:
- If exact arrival time is unknown, ask for origin departure datetime + origin location and estimate arrival time before producing the final day-by-day plan.
- If departure time from destination is unknown, ask explicitly before finalizing the last day.
- A draft skeleton is allowed, but it must be labeled provisional until timings are confirmed/estimated.

---

## Tools

### Google Maps Grounding Lite (required)
Use `grounding-lite.search_places` for:
- Place grounding (stable place IDs + Maps links)
- Rating + review signal (when present)

Use `grounding-lite.compute_routes` for:
- **Walking or driving time sanity checks** between daily anchors

**Note:** `WALK` routes are beta; include the standard warning whenever you present walking routes.

### Web Search (Brave API)
Use `web_search` for broad discovery and quick source triage:
- Research attractions, districts, and experiences
- Check opening days/hours and seasonal closures
- Verify current events, festivals, exhibitions
- Validate freshness and relevance

Setup note:
- `web_search` requires `BRAVE_API_KEY` in the Gateway environment (or `openclaw configure --section web`).
- Shell startup files like `.zshrc` may not be loaded by the Gateway service.

### Dedicated Browser (Live Search)
Use the agent’s dedicated Chrome window to:
- Validate top candidates from `web_search`
- Cross-check official venue/hotel pages and booking details
- Resolve ambiguous or conflicting info from search snippets

---

## Process

### 1. Load Context
- Read `/references/wishlist.md`
- Check calendar (read-only) for events tied to the destination and date constraints
- Confirm inferred details with the user before planning if needed

### 2. Structure the Trip
- Decide daily rhythm (start/end intensity)
- Group activities geographically per day
- Balance highlights with downtime

### 3. Deep Research (Live)
- Identify must-sees vs optional experiences
- Build a vetted food list by area/day (2–3 restaurants + 1–2 cafés)
- Prefer Google Maps-grounded picks with strong recent review signals
- Note seasonal/weather dependencies
- Spot local-only or time-bound experiences

### 4. Feasibility Check (Google Maps)
For each day:
- Choose 2–4 **anchors** (major stops + meal area) and validate:
  - Hotel area → first anchor (if provided)
  - Anchor → anchor (sequence)
  - Last anchor → hotel area (if relevant)
- If any leg is unrealistic for the day’s pacing (e.g. too many long walks), revise the grouping.

**Rules of thumb:**
- Keep “commute overhead” low by default: avoid stacking far-apart neighborhoods in one day.
- Prefer walkable clusters; use transit/short rides implicitly but don’t detail them.
- Only mention approximate travel times when it impacts feasibility (e.g. “~35 min walk”).
- If walking is used, display: “Walking routes are in beta and may miss pedestrian paths.”

### 5. Build the Itinerary
For each day:
- Morning / afternoon / evening outline
- Clear intent (“slow start”, “exploration-heavy”, “recovery day”)
- Optional swaps or fallbacks
- If needed: a brief feasibility note (e.g. “kept everything within ~15–20 min walk”)

### 6. Persist Preferences
- Add durable itinerary-related preferences to `/references/wishlist.md`

### 7. Notion Output (when asked)
- Use Notion API with `NOTION_API_KEY` from environment.
- Search accessible pages; if none are visible, ask the user to share the target page with the integration.
- Ask which page to publish into, or use the page they explicitly provided.
- **Default behavior: try updating an existing itinerary page first** (same destination/date range, or latest itinerary page in the selected parent).
- If no suitable page exists, create a new page in the selected workspace/page context.
- Title format: `Itinerary - <Destination> (<YYYY-MM-DD to YYYY-MM-DD>)`.

#### Preferred Notion format (default)
When the user asks for template-only output (or says they’ll edit content manually):
- Load and append JSON blocks from:
  - `templates/notion-template.json`
- Treat this JSON file as the single source of truth for template structure.
- Replace placeholders (e.g. `<Destination>`, `<date>`) before appending.
- Keep formatting clean and scannable with light, category-based emoji (not decorative emoji spam).
- Use collapsible day toggles and short callouts for intent/feasibility when supported by the template.
- If the template needs edits, update the JSON file instead of duplicating format definitions in SKILL.md.

#### Full content mode (when user asks for complete itinerary)
- Write structured blocks in this order:
  1. Trip summary (destination, dates, traveler count, style)
  2. Day-by-day plan (Day 1, Day 2, ...)
  3. Booked/fixed items
  4. Weather/backup alternatives
  5. Notes and assumptions
- Include Google Maps links for every grounded place (activity anchors, restaurants/cafés, and named backup options).
- Prefer direct place links when available.
- Fallback to query links if direct links are not available.
- For each grounded place, append rating in: `⭐4.8 (99)` (or `⭐4.8` if count missing).

- Prefer append/update over destructive rewrites.
- Only publish after explicit user confirmation.

### 8. Google Maps List (when asked)
- If asked, create a Google Maps saved list for the trip (e.g., `Porto Apr 2026`).
- Add itinerary anchors + selected restaurants/cafés.
- Prefer place-level pins over broad queries.
- Keep list curation tight.
- Confirm completion and share the list name used.

---

## Output Format

### Default: Day-by-Day Itinerary
- Clear daily headings
- Bullet-based activity flow
- Daily food picks: 1 lunch option, 1 dinner option, 1 café option
- Notes for pacing, alternatives, and feasibility (when relevant)

### Supported Formats
| Format | Use case |
|------|---------|
| **Chat Markdown** | Quick review and iteration |
| **Notion page** | Final saved itinerary |

---

## Guardrails
- Do not plan or recommend accommodations.
- Avoid generic “top 10”, “best of”, or “hidden gems” content; rely on live research.
- Re-evaluate days if dates, constraints, or fixed bookings change.
---
