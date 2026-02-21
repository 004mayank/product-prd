# PRD v1: Flight Metasearch (Competitor) — Search → Booking Handoff

**Product:** Flight metasearch marketplace (Skyscanner competitor)
**Author:** Mayank Malviya
**Date:** 21 Feb 2026
**Status:** v1 — MVP scope + ranking/trust fundamentals + partner handoff requirements

> This PRD defines a v1 competitor to Skyscanner focused on the Flights funnel from search → results → itinerary detail → partner selection → booking handoff.

---

## 0) TL;DR
Build a flight comparison product that wins on **trust + clarity** (price confidence, rules clarity, partner reliability) while maintaining fast search and a high click-out rate. The core surfaces are:
1) Search form
2) SRP (search results page) with “Best / Cheapest / Fastest” anchors
3) Itinerary detail page with a **Partner List** + a first-class **Trust Pack**
4) Redirect handoff with robust tracking + mismatch instrumentation

---

## 1) Problem statement
Flight metasearch users face 3 recurring failures:
- **Price mismatch** between SRP and partner checkout.
- **Rules ambiguity** (baggage, change/refund) that blocks conversion.
- **Low partner trust**, especially with unfamiliar OTAs.

A competitor can differentiate by making the “trust stack” legible *before* click-out without turning the product into a policy document.

---

## 2) Target users & JTBD
### Primary persona
**Value-seeking traveler** booking a flight with constraints (time, stops, airline, baggage, refundability) who wants a booking they can trust.

### JTBD
“Help me find the best flight for my constraints quickly, and send me to a booking path that won’t surprise me on price or rules.”

---

## 3) Goals / Non-goals
### Goals (v1)
1) Enable end-to-end funnel: **Search → SRP → Detail → Partner click-out**.
2) Establish **trust differentiation**:
   - price confidence
   - rules clarity
   - partner reliability
3) Build marketplace foundations:
   - partner governance signals
   - mismatch/complaint feedback loops

### Non-goals (v1)
- Owning checkout/payment (we are metasearch; booking happens at partner)
- Hotels/cars
- Loyalty, subscription, or bundles
- Full “Trips” itinerary management (can be v2)

---

## 4) Success metrics
### North Star
**Trusted qualified click-outs per search session** (a click-out that does not immediately bounce back and has low mismatch probability).

### Core metrics
- **Search → SRP rate**
- **SRP → Itinerary detail open rate**
- **Detail → Partner click-out rate**
- **Bounce-back rate**: return within 30–120s of click-out
- **Mismatch proxy rate**: partner landing price != shown price (via user report + automated checks where feasible)
- **Repeat searches** (7/30-day)

### Guardrails
- SRP time-to-first-result (TTFR) p95
- Complaint rate per 1k click-outs
- Partner diversity / no over-penalization without evidence

---

## 5) Competitive differentiation (the “Comp” strategy)
We compete on:
1) **Trust Pack** as a first-class product surface (not just badges)
2) **Identical flight clustering** (“same flight, different sellers”) to reduce SRP overwhelm
3) **Freshness + transparency** (updated timestamp, price confidence)

We do *not* attempt to beat Skyscanner on sheer partner coverage initially; we win by being meaningfully more reliable in a defined geo/route set.

---

## 6) Scope & user journey (v1)
### 6.1 Entry / Search form
**Requirements**
- From/To autocomplete (airport/city)
- Dates: one-way / return
- Travelers + cabin
- Optional: nearby airports toggle

**Acceptance criteria**
- Users can submit a search in < 30 seconds from landing.
- Autocomplete resolves ambiguity (city vs airport codes).

### 6.2 SRP (Search Results Page)
**Key objects**
- Itinerary cards: price, duration, stops, dep/arr times, airline(s)
- Anchors: **Best / Cheapest / Fastest** (top tabs or chips)
- Filters: stops, times, duration, airlines, airports; sort by the same anchors

**v1 SRP requirements**
- Default sort = **Best** (multi-objective)
- Inline “Direct only” toggle
- Price + duration always visible
- “Same flight, different sellers” clustering (at least on detail page; ideally hinted on SRP)

**Acceptance criteria**
- SRP renders initial results with skeleton states and progressive refinement.
- Filter changes update results within reasonable latency (target: <2s perceived).

### 6.3 Itinerary detail + Partner list
**Detail requirements**
- Full legs + layover details
- Summary of baggage + change/refund where available
- Partner list with prices
- Clear redirect disclosure (“You’ll be redirected to …”)

**Partner list requirements**
- Highlight: “Best deal” and “Most trusted” (may be different)
- Show partner labels (official site / trusted / best price)

**Acceptance criteria**
- At least 2 partner options for top itineraries in initial launch markets.
- Partner row shows: partner name, price, trust label, and any known caveats.

### 6.4 Booking handoff (redirect)
**Requirements**
- Construct deep link / payload to partner
- Include attribution params
- Preserve locale, currency, pax count

**Acceptance criteria**
- Click-out logs include: search_id, itinerary_id, partner_id, price_shown, timestamp, device.

---

## 7) The Trust Pack (v1 spec)
### Components
1) **Price confidence** (High / Medium / Low)
   - Inputs (initial): partner historical mismatch, fare freshness age, route volatility bucket
2) **Rules clarity** (Clear / Partial / Unknown)
   - Inputs: baggage/change/refund completeness + normalization success
3) **Partner reliability** (Trusted / New / Caution)
   - Inputs: complaint rate proxy, bounce-back rate, mismatch rate

### UI placement
- Always visible on itinerary detail; condensed variant on partner rows.

### Measurement
- Primary: mismatch proxy ↓, bounce-back ↓
- Guardrail: click-out rate does not regress

---

## 8) Ranking & “Best” definition (v1)
### Ranking objective
Maximize user satisfaction and trusted click-outs, not just cheapest price.

### v1 model approach (pragmatic)
- Start with a **scoring function** (not ML-heavy):
  - price percentile (lower better)
  - duration penalty
  - stops penalty
  - red-eye / bad arrival penalty (configurable)
  - trust adjustments (penalize low confidence/reliability)

### Outputs
- Badges:
  - Cheapest = min price
  - Fastest = min duration
  - Best = max composite score

---

## 9) Partner marketplace (v1)
### Launch strategy
- Start with a small set of high-quality partners (airline direct + 1–2 OTAs).
- Prefer partners with stable deep links and predictable pricing.

### Governance hooks
- Maintain per-partner:
  - mismatch rate
  - bounce-back rate
  - complaint/review signals
- Introduce automatic penalties in ranking only after enough volume.

---

## 10) Data & instrumentation (minimum viable)
### Entities
- Search
- Itinerary
- Offer (itinerary + partner + price)
- Click-out
- Return event (bounce-back)
- Feedback / report mismatch

### Events
- search_submitted
- srp_viewed
- filter_changed
- itinerary_opened
- partner_clicked_out
- returned_from_partner
- mismatch_reported

---

## 11) Risks & mitigations
1) **Mismatch rates damage trust**
   - Mitigation: show price confidence, freshness, and partner reliability; reduce partner set early.
2) **Insufficient inventory / coverage**
   - Mitigation: launch narrow (geo/route focus) and be explicit about scope.
3) **Over-trusting partner labels**
   - Mitigation: require thresholds + disclose “New partner” states.

---

## 12) Rollout plan (v1)
1) Private alpha (internal + friends) on limited routes
2) Beta in a single geo/market
3) Expand partner set once trust metrics stabilize

---

## 13) Open questions
1) Which launch market (geo/routes) do we target first?
2) Which partners are available and what deep-linking/attribution constraints exist?
3) What level of rules normalization is feasible in v1 (baggage/refund)?
4) Do we optimize primarily for cheapest or for “trusted best” as the default?
