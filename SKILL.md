---
name: travel-guide-builder
description: Build a structured travel guide in a fixed 5-step workflow. Use whenever the user asks for a travel plan, itinerary, or trip planning, or wants attractions sorted into a route with time and transport estimates. Triggers on phrases like "travel guide", "itinerary for", "trip to", "plan a trip to", "help me plan". Output follows a strict 5-step structure (pre-trip clarification → attractions on map → route ordering → timed itinerary → booking summary) in Lonely Planet voice — objective, restrained, no marketing language. The skill does NOT trigger for single factual questions like "what's the weather in Tokyo" or "how much is a flight to Paris" — those are not guides.
---

# Travel Guide Builder

## What this skill does

Produces a structured travel guide following a strict 5-step workflow. The output is never freeform — it always has the same five sections in the same order, even if the trip is short.

## When to use it

Trigger when the user asks for any of:
- A travel itinerary or guide for a specific destination
- Help planning a multi-stop day trip
- Sorting a list of attractions into a sensible route
- A trip plan that includes timing, transport, and budget

Do NOT trigger for:
- Single-fact questions ("What time does the Louvre open?")
- Pure recommendations without a planning component ("What are some good restaurants in Tokyo?")
- Visa, insurance, or other administrative questions

## Voice and tone

Write in **Lonely Planet voice**:
- Objective and factually-grounded. Describe places by what they are and what's notable, not how amazing they are.
- No marketing words: avoid "must-see", "must-do", "breathtaking", "stunning", "hidden gem", "bucket list", "once in a lifetime", "Instagrammable", "Insta-worthy", "iconic", "world-class", "off the beaten path", "charming", "quaint", "picturesque". If you catch yourself reaching for those, rewrite the sentence.
- Cultural and historical context is welcome where it earns its place — a one-line note on why a place matters, not a Wikipedia paragraph.
- Honest about downsides: crowds, overrated spots, tourist traps. A good guide warns as much as it recommends.
- No exclamation marks. No emoji.

## Required tools

Before writing the guide, you MUST use web search to verify time-sensitive details. Do not rely on training data for any of:
- Opening hours and closing days
- Ticket prices and whether advance booking is required
- Transit lines, routes, or station names
- Whether an attraction is currently open (closures, renovations)

If a search yields nothing reliable, say so explicitly in the guide ("opening hours unconfirmed — verify before going") rather than guessing.

## The 5-step output structure

The guide always has exactly these five sections, in this order, with these headings. Do not rename them, merge them, or add extra top-level sections.

### Step 0 — Pre-trip clarification

Before listing any attractions, confirm a few variables that will shape every later step. Ask ALL of these in a single turn (use ask_user_input_v0 if buttons help), then wait for the user's reply before continuing to Step 1.

**Always ask:**
- **Arrival and departure times** — Arrival and departure datetimes (date + rough time of day). This bounds Step 3's earliest start and latest end. For multi-city trips, ask per city/segment.
- **Time-locked transport** — Any time-locked transport during the trip (flights, KTX/intercity trains, ferries, last bus). Get date + time for each. Treat these as **immovable** when building Step 3. When initially asking, just ask for the times — don't try to compute buffers before you have the data. **After the user answers**, in the Step 0 confirmation line, subtract realistic buffer (airport: 2–3h pre-flight + 45–60min commute; intercity train: 30–40min station buffer + commute) and tell the user the resulting "latest activity end" out loud. Example confirmation line: "Got it: your flight departs Incheon at 19:50 on Day 3, which means the city itinerary needs to wrap up by 16:30."
- **Approximate month / season** — Only ask if not already inferable from arrival date. Affects opening hours, weather warnings, and seasonal closures.
- **Interests (optional)** — Ask what the user most wants to experience (food / shopping / nature / history / design / nightlife, etc.). Multi-select. Crucially, **let the user opt out**: phrase the question to make "no preference / surprise me" a first-class answer, not an afterthought. People often travel to escape their work persona — a designer doesn't necessarily want a design-themed trip, an engineer doesn't necessarily want science museums. **Never derive interests from the user's profession or userPreferences alone.** If the user explicitly opts out (or simply doesn't pick anything), proceed to Step 1 without applying any interest filter — list classic / popular / cluster-coherent attractions instead. Only use this as a hard filter when the user has clearly named preferences in this turn.
- **Travel companions / special needs** — Ask once and only once whether the trip includes anyone with special needs that would meaningfully shape the shortlist: elderly (mobility-limited), young children, pregnancy, disability, allergies, or chronic health conditions. Frame it as "Anyone in your group with X / Y / Z?" with concrete examples so the user knows what to volunteer. If none, the user can say "no" and Step 1 proceeds normally. If yes, log the constraint and use it as a filter in Step 1 (e.g., flag attractions requiring lots of stairs, mark spa/sauna venues as "not pregnancy-safe", warn about late-night dinner spots when children are involved).

**Ask only when applicable:**
- **Multi-city time split** — If the destination spans 2+ cities, ask how the user wants to split time across them (e.g. "2 nights Busan + 1 night Seoul"). Skip if the user already specified, and skip entirely for single-city trips.
- **Lodging location** — Ask if they've booked accommodation and where. If they have, use it as Step 3's start/end anchor each day. If they haven't booked yet, do NOT push — Step 4's lodging-area suggestions will cover it. Never block on this question.

**Do not ask:**
- Anything already given in the user's initial message — re-asking is annoying and signals not reading.
- Budget, group size, dietary restrictions — only ask if the user volunteers something that makes them relevant. (Note: physical/medical constraints are covered by the "travel companions" question above, not here.)

After collecting answers, briefly confirm what you heard in 1–2 lines ("Got it: 2 nights Busan + 1 night Seoul, arriving Busan morning of 5/20, departing Seoul evening of 5/23, interested in X / Y") and proceed to Step 1.

**Before moving to Step 1, run two quick sanity checks at the end of the confirmation:**

1. **Companion-friendliness check.** If Step 0 surfaced a special-need constraint (elderly, mobility-limited, pregnancy, young children) AND the itinerary density is high (>3 cities in 7 days, or >2 cities in 5 days, or one-night hotel stays back-to-back), gently flag the mismatch. Don't override the user's choice — just surface the tradeoff once. Example: "Heads-up: 3 cities in 7 days means changing hotels twice, which can be tiring with mobility-limited parents. We can still proceed as-is, or you can consider dropping one city. Your call."

2. **Implicit-theme check.** If the user's initial message already contains a strong implicit theme (cherry blossom season, autumn foliage, hot springs trip, aurora chasing, pilgrimage, food tour, etc.), don't ask "interests?" as if it's a blank slate. Adopt the implicit theme as the default and phrase the question as a refinement: "Cherry blossom is already the main thread — anything else you'd want to weave in (food / shopping / temples)?" The "no preference" option then means "just stick with cherry blossom".

Both checks are one-line nudges, not interrogations. If neither applies, skip them and proceed straight to Step 1.

### Step 1 — Attraction shortlist

**Display order for this step: map first, then text fields.** The map answers "where are these things and how do they cluster"; the text fields answer "should I pick this one" with hard info (hours, price, time needed).

**Map first.**

Use `places_map_display_v0` to plot every candidate. Each marker should carry a one-line `notes` field that says what district/cluster the place belongs to (e.g. "Nampo / Yeongdo — southwest corner") so the geographic grouping is obvious from the map alone.

Rules:
- Use `places_search` first to resolve `place_id` — never invent place IDs.
- For trips spanning multiple cities, output ONE map PER CITY, not a single zoomed-out map. A combined map of Busan + Seoul makes both unreadable.
- `show_route` should be false at this stage — the user hasn't picked yet, there's no route.
- After the map, add a 2–3 line plain-language summary of how the candidates cluster geographically. This is what the map alone can't say — e.g. "Busan attractions fall into 4 directions: Nampo/Yeongdo, Seomyeon, Haeundae, Gijang. Each forms its own cluster; cross-cluster travel runs 30–60 min."

**Then the text fields.**

For each candidate, provide exactly these fields, in this order:

- **Name** — Name (local language + English where useful)
- **Description** — ~120-character description. Should answer "what is this place and why is it notable", not "how great it is". Aim for compactness; if you go over 150, you're padding.
- **Hours** — Opening hours, including closing day if any
- **Ticket** — Ticket price in local currency + approximate USD. Note if advance booking is required.
- **Suggested visit** — Suggested visit duration (e.g. "1.5–2 hours")
- **Accessibility note** — **Only include this field if Step 0 surfaced a relevant constraint** (elderly, mobility-limited, pregnancy, young children, etc.). One line, factual: stairs count, walking distance, slope, surface (gravel/cobblestone), elevator availability, stroller-friendliness. Examples: "Mostly flat, wheelchair accessible." / "~50 steps from gate to main hall, plus 400m uphill path." / "Gravel paths throughout — tough on knees." If Step 0 had no relevant constraint, omit this field entirely — don't add it to every shortlist by default.

How many attractions to list:
- If the user gave a specific number, use that
- If they gave duration (e.g. "3 days in Kyoto"), list 1.5× what a tight schedule would hold — give them choice
- **For single-city trips:** if unspecified, list 8–12 total
- **For multi-city trips:** list 4–6 per city, total grows naturally with city count. Don't force the total down to fit an arbitrary cap — a 3-city trip will land around 12–18 candidates, which is fine.

After the list, ask the user: "Which ones would you like to visit? Reply with the numbers, or say 'all of them' / 'pick for me' and I'll move on to routing."

**Stop here and wait for the user's selection.** Do not auto-proceed to Step 2.

**If the user names a new attraction not on the shortlist** (whether at this step or later), do NOT just append a one-line acknowledgement. Re-run Step 1 in full for the affected city: search for the new attraction's place_id, web-search opening hours and pricing, regenerate the city's map with all current candidates (existing selections + the new one), redo the cluster summary (the new attraction may shift cluster balance), and re-list every candidate's full text fields. The point: the user should always see the complete current state of the shortlist, not a patchwork of additions. This is verbose by design — drift caused by incomplete updates is worse than the cost of redrawing.

### Step 2 — Route ordering

Once the user has picked their attractions, sort them into a geographically sensible order. For each consecutive pair (A → B), show three transport options in this exact format:

```
A → B
  Walk: x.x km, ~xx min
  Public transit: walk to [station] → [line] → walk to destination, ~xx min
  Taxi: x.x km, ~xx min, ~$xx
```

Rules:
- Verify transit lines and station names via web search — these are easy to hallucinate
- All times and distances are **estimates**. Add this disclaimer once at the top of Step 2: "Times below are estimates — verify on a maps app before you go."
- If a leg has an obvious best option (e.g. walking is clearly fastest), say so in one line after the three options.
- If the user picked attractions in different cities, group by city first, then sort within each.
- **For multi-day stays in the same city: first group by geographic cluster, then assign clusters to days.** Each day should ideally hold one cluster — cross-cluster travel within a single day burns 30–60 min of commute that's better spent on activities. If clusters ≠ days, say so explicitly: either "This city has 4 clusters but only 3 days, so Day X will cross clusters with ~40 min of extra commute" or "This city has 2 clusters but 3 days, so we can split one cluster into two half-days". Don't silently merge or split — surface the imbalance so the user sees the tradeoff.

After the ordering, ask the user to confirm or adjust before moving on.

### Step 3 — Timed itinerary

Build a time-blocked itinerary using the order confirmed in Step 2. Format:

```
Day 1
  09:00  Arrive at A
  09:00–11:00  Visit A (~2h)
  11:00–11:30  A → B (subway, 30 min)
  11:30–13:30  Visit B (~2h)
  ...
```

Rules:
- **Use Step 0's arrival/departure times as hard boundaries.** Day 1 cannot start before arrival; the final day cannot end after departure. If the user arrives at 14:00, do not schedule a 09:00 attraction that day — start Day 1 from arrival time + buffer (typically 1–1.5h for hotel drop-off / transit from station). Same logic for the last day's end.
- Default start time: 09:00, default end time: 18:00, used ONLY when arrival/departure don't constrain that day.
- Insert lunch (12:00–13:30 area) and dinner (18:30–20:00 area) as blocks even if no specific restaurant is chosen — leave them as "Lunch" / "Dinner" placeholders so the user sees realistic pacing.
- Account for transit between attractions using the times from Step 2.
- If the schedule overflows the day, do not silently cut attractions — flag it: "Day is tight — consider moving X to the next day, or shortening time at Y."
- For multi-day trips, group by Day 1 / Day 2 / Day 3.
- If the user gave a lodging location in Step 0, use it as each day's start/end anchor and route accordingly.

**After the timed itinerary, you MUST give an itinerary risk audit. Do not wait for the user to ask.**

The itinerary as written always looks neat and feasible. The risk audit's job is to surface what's actually fragile — the segments where 30 minutes of delay will cascade, where buffer is zero, where two hard constraints are in tension. Without this, users often discover problems only mid-trip.

Structure:
- Three severity tiers: **🔴 High** (could derail the trip), **🟡 Medium** (degrades experience, not fatal), **🟢 Low** (worth knowing, not worth acting on). Tiers don't need balanced counts — sometimes the audit is mostly 🔴, sometimes mostly 🟡.
- Each risk item must contain four things: **which day + which time block**, **the specific conflict mechanism**, **the likely consequence**, **a concrete mitigation**.
- After the per-risk list, end with **2–3 lines of overall rhythm assessment** — buffer per day, biggest single risk point, and a one-liner verdict ("Doable, but Day 2 requires taxis throughout" / "Overall tight, Day 3 morning has zero buffer").

What counts as a real risk:
- Time math that doesn't quite work (transit + queue + visit > available window)
- Two hard constraints in tension (last train at 20:30 vs Spa Land closing at 19:30)
- Zero-buffer cascades (any delay propagates because nothing has slack)
- Day-of-week / seasonal effects with specific impact (not "lots of crowds in May" — say "Saturday morning at Gyeongbokgung, peak crowd around the 09:30–10:30 changing-of-the-guard")

What does NOT count as a risk (do not list these — they're empty filler):
- "Might get stuck in traffic" / "Might be a queue" without naming the specific segment and consequence
- "Crowded attraction" / "Weather might be bad" as generic warnings
- Risks the user already knows from common sense ("Flight might be delayed")

Bad example (vague, filler):
> 🟡 May is peak season, expect queues

Good example (specific, actionable):
> 🔴 **Day 1 X the Sky → Blueline Park last train** I scheduled 18:00–19:30 at X the Sky, but it actually needs 1.5–2h (elevator queue + 99th/100th floor + high-altitude Starbucks). Leaving at 19:30 + 7 min walk to Mipo Station = 19:37. Blueline last train is around 20:30 and requires advance reservation — only 53 min margin, very easy to miss. **Mitigation:** book a 19:40 or 20:00 timed slot in advance on Klook.

Bad example (no mitigation):
> 🔴 Day 2 Busan Station lockers might be full

Good example (with mitigation):
> 🔴 **Day 2 Busan Station luggage storage risk** Large lockers can fill up at morning peak. If you can't store bags at 09:20, you'll lose 30–40 min finding paid storage, which could push you past the 16:50 KTX departure. **Mitigation:** pre-book a paid storage spot near Busan Station via Bounce / Stasher (~$5/bag/day, guaranteed spot).

After the audit, do NOT bundle pre-booking action items here — those belong to Step 4 and will be reorganized by risk tier there. Step 3's audit ends with the rhythm assessment.

### Step 4 — Pre-trip booking checklist

Organize this checklist **by risk tier from Step 3, not by category** (tickets/transit/lodging). The point is to tell the user what they MUST pre-book to avoid the 🔴 risks surfaced in Step 3, vs. what's nice to pre-book to avoid 🟡 hassle, vs. what's fine to handle on arrival. This structure makes Step 3 and Step 4 reinforce each other.

**🔴 Must book in advance (skipping risks the trip)**
- [item] — [corresponds to which 🔴 risk in Step 3] — Where to buy: [official site / platform]
- e.g. KTX Busan→Seoul 5/22 16:50 — corresponds to "Day 2 KTX hard constraint" — Korail Talk app, book at least 2 weeks ahead

**🟡 Strongly recommended to pre-book (otherwise long queues / missed slots)**
- [item] — [reason + corresponding 🟡 risk in Step 3] — Where to buy: [platform]
- e.g. Haeundae Sky Capsule — sells out at peak season, corresponds to Day 1 dusk capsule reservation risk — Klook, 3–4 weeks ahead

**🟢 Buy on arrival**
- [item] — brief reason why pre-booking isn't needed
- e.g. Gyeongbokgung ticket — counter or kiosk on-site, 3,000 KRW

**Lodging (reference)**
- Give 2–3 area suggestions (not specific hotels), explaining what kind of traveler each area suits (e.g. Kyoto's "Shijo Kawaramachi — transit hub, plenty of dining, mid-range prices; suited to first-time visitors with a full schedule").
- If the user gave a lodging location in Step 0, **skip this section** — they don't need area suggestions.

**Practical reminders**
- Cash / transit card / mobile data / local etiquette, etc. One line each, max 3 items — don't let it become a list pile-up.

Rules for Step 4:
- Every item in 🔴 or 🟡 should reference back to a specific Step 3 risk by name. If you can't tie an item to a Step 3 risk, ask yourself whether it really belongs in 🔴/🟡 or if it's actually 🟢.
- Do not invent risks here that weren't surfaced in Step 3. Step 4 reorganizes; it does not create new threats.
- If Step 3's audit had no 🔴 items, that section here can simply say "No must-book items for this trip — all pre-booking falls into 🟡 or 🟢" — don't fabricate to fill the section.

## Hidden gems & avoidance tips (woven throughout)

Across all 5 steps, when relevant, weave in:
- **Local-perspective hidden recommendations**: must explain "why this place is worth it" (one sentence). Don't just say "off-the-beaten-path / photogenic / no crowds".
- **Tourist-trap warnings**: explicitly call out tourist traps, overcharging spots, overrated attractions. Say "X gets mixed reviews mainly because of [specific reason]", not just "X is not recommended".

These should be ≤2 per step, integrated into the relevant section's prose — not a separate bullet list at the end.

## Examples of voice

Bad (overly excited, marketing-speak):
> Kyoto's Fushimi Inari Shrine is an absolute must-see! The thousand torii gates are super Instagrammable — a total bucket list spot. Don't miss the early-morning photo op!

Good (Lonely Planet voice):
> Fushimi Inari Shrine is known for the vermilion torii gates that line the mountain path; the full uphill loop takes about two hours. Crowds thin before 7am and grow heavy after 9am, when the lower section becomes nearly impossible to photograph without people. The summit view is unremarkable — most visitors turn back at Yotsutsuji.

Bad (vague disclaimer):
> Times are approximate.

Good (specific, actionable):
> Times below are estimates — verify on Google Maps or your preferred maps app before you go. Subway frequency and real-time traffic affect actual transit time.

Marketing words to avoid in English: "must-see", "must-do", "Instagrammable", "bucket list", "hidden gem" (unless you can specifically explain why), "once in a lifetime", "unmissable", "world-class", "breathtaking".

## What success looks like

A user can read the output and:
1. Feel that Step 0's clarifying questions were minimal, non-redundant, and respected what they already said
2. Know what places they're choosing between, with enough info to choose — and see at a glance where those places sit on a map
3. See a route that makes geographical sense
4. Get a realistic time-blocked plan that respects their arrival and departure times
5. Know exactly what to book before leaving home

If any of these five is missing or vague, the skill has failed.
