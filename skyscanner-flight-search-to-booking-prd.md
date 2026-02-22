# PRD v2: Flight Metasearch (Competitor) — Search → Booking Handoff

**Product:** Flight metasearch marketplace (Skyscanner competitor)
**Author:** Mayank Malviya
**Last updated:** 22 Feb 2026
**Status:** v2 — clarified scope, requirements, instrumentation, and rollout; added explicit trust/mismatch program

> This PRD defines a competitor to Skyscanner focused on the Flights funnel from **search → results → itinerary detail → partner selection → booking handoff**, differentiated by a first-class **Trust Pack** (price confidence, rules clarity, partner reliability).

---

## v2 changelog (vs v1)
- Added **launch scope** (markets/routes), explicit **MVP constraints**, and **non-goals** refinements.
- Expanded functional requirements for **SRP clustering**, **Offer freshness**, and **redirect handoff**.
- Added a concrete **Mismatch Program** (detection, user reporting, partner penalties, and UI).
- Strengthened **data/instrumentation** (event schema, identifiers, attribution, bounce-back logic).
- Added **privacy/compliance** and **accessibility** requirements.
- Added **experimentation** plan and **go/no-go** gates for rollout.

---

## 0) TL;DR
Build a flight comparison product that wins on **trust + clarity** (price confidence, rules clarity, partner reliability) while maintaining fast search and a high click-out rate.

Core surfaces:
1) Search form
2) SRP (search results page) with “Best / Cheapest / Fastest” anchors
3) Itinerary detail page with **Partner List** + **Trust Pack**
4) Redirect handoff with robust tracking and mismatch instrumentation

**Primary differentiator:** We make *price/rules/partner risk legible before click-out*.

---

## 1) Problem statement
Users of flight metasearch hit recurring failures:
- **Price mismatch** between SRP and partner checkout.
- **Rules ambiguity** (baggage, change/refund) that blocks conversion.
- **Low partner trust**, especially for unfamiliar OTAs.

A competitor can differentiate by making the trust stack clear *before* click-out without turning the product into a policy document.

---

## 2) Target users & JTBD
### Primary persona
**Value-seeking traveler** booking a flight with constraints (time, stops, airline, baggage, refundability) who wants a booking they can trust.

### JTBD
“Help me find the best flight for my constraints quickly, and send me to a booking path that won’t surprise me on price or rules.”

---

## 3) Goals / Non-goals
### Goals (v2)
1) End-to-end funnel: **Search → SRP → Detail → Partner click-out**.
2) Trust differentiation:
   - price confidence + freshness
   - rules clarity (baggage/change/refund)
   - partner reliability and transparent labels
3) Marketplace foundations:
   - partner governance signals
   - mismatch/complaint feedback loops
4) Operational excellence:
   - predictable latency (TTFR)
   - observable funnel with debugging tools

### Non-goals (for this PRD version)
- Owning checkout/payment (metasearch only)
- Hotels/cars
- Bundles, loyalty, subscriptions
- Full itinerary management (“Trips”)
- Guaranteed “final price” (we provide confidence + recency + accountability signals)

---

## 4) Launch scope (explicit)
### Initial geography / routes (MVP)
Pick **one primary market** and a **narrow route set** (e.g., top 50 routes by demand) to maximize data density for mismatch + trust scoring.

**Decision required:** choose one of:
- India domestic + nearby international
- UK + EU short-haul
- US domestic (harder; more partners/complexity)

### Device/platform
- Web first (mobile web + desktop)
- App later (out of scope for MVP unless explicitly required)

---

## 5) Success metrics
### North Star
**Trusted qualified click-outs per search session**
> A click-out that does not quickly bounce back and has low mismatch probability (or results in low mismatch reporting).

### Core funnel
- Search → SRP rate
- SRP → Itinerary detail open rate
- Detail → Partner click-out rate
- **Bounce-back rate**: return within 30–120s of click-out
- **Mismatch proxy rate**: partner landing price ≠ shown price (via report + automated checks where feasible)
- Repeat searches (7/30-day)

### Guardrails
- SRP TTFR p95
- Complaint rate per 1k click-outs
- Partner diversity (avoid over-penalization without evidence)

---

## 6) Competitive differentiation (“Comp” strategy)
We compete on:
1) **Trust Pack** as a first-class surface
2) **Identical flight clustering** (“same flight, different sellers”) to reduce SRP overwhelm
3) **Freshness + transparency** (updated timestamp, confidence, and clear disclaimers)

We do not attempt to beat Skyscanner on partner coverage initially.

---

## 7) Scope & user journey (requirements)

### 7.1 Search form
**Requirements**
- From/To autocomplete (city + airport)
- Dates: one-way / return
- Travelers + cabin
- Optional: nearby airports toggle
- Currency + locale derived from geo/selection (override allowed)

**Acceptance criteria**
- Users can submit a search in < 30s from landing.
- Autocomplete disambiguates city vs airport codes.
- Search URL is shareable (encodes query).

---

### 7.2 SRP (Search Results Page)
**Key objects**
- Itinerary cards: price, duration, stops, dep/arr times, airline(s)
- Anchors: **Best / Cheapest / Fastest**
- Filters: stops, times, duration, airlines, airports

**v2 SRP requirements**
- Default sort = **Best** (multi-objective)
- Inline “Direct only” toggle
- Price + duration always visible
- **Clustering:** group offers by *identical itinerary* (same legs/flight numbers/airports/times), with multiple sellers inside
  - SRP may show best offer per cluster + “X sellers” affordance
- Show **offer freshness**: “Updated X min ago” on cluster or selected offer
- Show **price confidence** badge (High/Med/Low) at least on top offers

**Performance requirements (targets)**
- TTFR p95: set target based on stack; track continuously
- Progressive results: show partial results while completing aggregation

**Acceptance criteria**
- SRP renders with skeleton states.
- Filter changes update results quickly (target <2s perceived) and are cancellable.

---

### 7.3 Itinerary detail + partner list
**Detail requirements**
- Full legs + layover details
- Baggage summary + change/refund summary (where available)
- Partner list with prices and trust labels
- Clear redirect disclosure (“You’ll be redirected to …”)

**Partner list requirements**
- Highlight: “Best deal” and “Most trusted” (may differ)
- Partner labels: airline direct / trusted OTA / new / caution
- Each partner row includes:
  - partner name
  - total price (with currency)
  - freshness timestamp / age
  - Trust Pack condensed chips (price confidence, rules clarity, partner reliability)
  - known caveats (e.g., “Baggage details incomplete”)

**Acceptance criteria**
- At least 2 partner options for top itineraries in launch markets.
- Partner row shows: partner name, price, trust label, caveats.

---

### 7.4 Booking handoff (redirect)
**Requirements**
- Construct deep link/payload to partner
- Include attribution parameters
- Preserve locale, currency, pax count
- Maintain consistent **offer_id** (or derived signature) through click-out

**Redirect UX requirements**
- Interstitial (optional) for transparency:
  - “Redirecting to {partner}”
  - price shown + timestamp
  - lightweight disclaimer + “Report a price issue” link

**Acceptance criteria**
- Click-out logs include: search_id, itinerary_id, offer_id, partner_id, price_shown, timestamp, device, locale/currency.

---

## 8) The Trust Pack (v2 spec)
### Components
1) **Price confidence** (High / Medium / Low)
   - Inputs (v2):
     - partner historical mismatch rate (route-level + global)
     - offer freshness age
     - route volatility bucket
     - % of offers that require re-price at partner
2) **Rules clarity** (Clear / Partial / Unknown)
   - Inputs:
     - baggage/change/refund completeness
     - normalization success and vendor reliability
3) **Partner reliability** (Trusted / New / Caution)
   - Inputs:
     - complaint/report rate
     - bounce-back rate
     - mismatch rate
     - customer support signals if available

### UI placement
- Always visible on itinerary detail.
- Condensed chips on partner rows.

### Measurement
- Primary: mismatch proxy ↓, bounce-back ↓
- Guardrail: click-out rate does not regress materially

---

## 9) Ranking & “Best” definition
### Objective
Maximize user satisfaction and trusted click-outs, not just cheapest price.

### v2 approach
- Start with a **scoring function** (configurable weights):
  - price percentile
  - duration penalty
  - stops penalty
  - undesirable schedule penalties (red-eye, long layover)
  - **trust adjustments** (penalize low confidence/reliability; boost clear rules)

### Outputs
- Cheapest = min price offer (per cluster)
- Fastest = min duration itinerary
- Best = max composite score (per cluster)

---

## 10) Offer freshness & caching
**Why:** stale offers increase mismatch and bounce-backs.

**Requirements**
- Store **fetched_at** per offer.
- Display “Updated X min ago” when age exceeds threshold (configurable).
- If offer is older than threshold, show “Refresh prices” affordance (optional in MVP).

**Acceptance criteria**
- Offers older than threshold are labeled; click-outs still tracked.

---

## 11) Partner marketplace
### Launch strategy
- Start with small set of high-quality partners (airline direct + 1–2 OTAs).
- Prefer partners with stable deep links and predictable pricing.

### Governance hooks
Maintain per-partner and per-route:
- mismatch rate
- bounce-back rate
- report/complaint rate

Penalty policy (v2)
- Apply ranking penalties only after minimum volume thresholds.
- Show “New partner” state until thresholds met.

---

## 12) Mismatch Program (v2)
### Detection
- **User report**: “Price changed” report flow post-click-out and/or on return
- **Bounce-back proxy**: returned_from_partner within 30–120s
- **Automated sampling** (where feasible): periodic headless checks on a subset of offers (optional; depends on partner constraints)

### User experience
- Provide a simple report UI:
  - “Price was higher / lower / flight not available / rules different”
  - optional screenshot upload (out of scope unless needed)

### Enforcement
- Dashboard + alerts for partner mismatch spikes.
- Graduated penalties:
  1) label warnings
  2) ranking penalty
  3) remove partner from route/market

---

## 13) Data & instrumentation
### Entities
- Search
- Itinerary (normalized)
- Offer (itinerary + partner + price)
- Click-out
- Return event (bounce-back)
- Feedback / mismatch report

### Event schema (minimum viable)
- search_submitted {search_id, query, locale, currency}
- srp_viewed {search_id}
- filter_changed {search_id, filters}
- itinerary_opened {search_id, itinerary_id}
- partner_clicked_out {search_id, itinerary_id, offer_id, partner_id, price_shown, fetched_at, ts}
- returned_from_partner {search_id, offer_id, ts, time_away_sec}
- mismatch_reported {search_id, offer_id, type, delta?, free_text?}

### Identifiers & attribution
- Use globally unique **search_id** and **offer_id**.
- Carry offer_id + click_id to partner as params where allowed.

---

## 14) Privacy, compliance, and accessibility
### Privacy
- Minimize PII: do not log passenger names, emails, or payment data.
- If IP/device identifiers are stored, document retention and purpose.
- Provide user-facing disclosure for redirects and tracking.

### Accessibility
- Keyboard navigable search and SRP filters.
- SRP cards and partner list meet contrast standards.
- Screen-reader labels for Trust Pack chips and badges.

---

## 15) Rollout plan + go/no-go gates
### Rollout
1) Private alpha (internal + friends) on limited routes
2) Beta in a single geo/market
3) Expand routes
4) Expand partners once trust metrics stabilize

### Go/no-go gates (examples)
- Mismatch proxy rate below threshold for 2 consecutive weeks
- Bounce-back rate stable/improving
- TTFR p95 within target
- No critical partner deep-link failures

---

## 16) Open questions
1) Which launch market (geo/routes) do we target first?
2) Which partners are available and what deep-link/attribution constraints exist?
3) Minimum viable rules normalization: which baggage/refund fields must be present for “Clear”?
4) Do we optimize primarily for cheapest or for “trusted best” as the default?
