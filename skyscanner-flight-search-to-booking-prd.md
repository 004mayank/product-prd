# PRD: Skyscanner - Search → Booking Handoff

<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/Skyscanner.png" 
    alt="Skyscanner" 
    width="200"
  />
</p>

**Product:** Skyscanner
**Author:** Mayank Malviya  
**Last updated:** 23 Feb 2026  
**Status:** Final- locked MVP, added concrete SLOs, clarified data contracts + governance thresholds, and specified Trust Pack scoring + UX states

> This PRD defines Skyscanner focused on the Flights funnel from **search → results → itinerary detail → partner selection → booking handoff**, differentiated by a first-class **Trust Pack** (price confidence, rules clarity, partner reliability).
---

## TL;DR
Build a flight comparison product that wins on **trust + clarity** while keeping search fast and click-out high.

Core surfaces:
1) Search form
2) SRP (search results page) with “Best / Cheapest / Fastest” anchors
3) Itinerary detail page with **Partner List** + **Trust Pack**
4) Redirect handoff with robust tracking and mismatch instrumentation

**Primary differentiator:** Make *price/rules/partner risk legible before click-out*-and create enforcement loops when partners misbehave.

---

## 1) Problem statement
Users of flight metasearch hit recurring failures:
- **Price mismatch** between SRP and partner checkout.
- **Rules ambiguity** (baggage, change/refund) that blocks conversion.
- **Low partner trust**, especially for unfamiliar OTAs.

A competitor can differentiate by making trust signals clear *before* click-out without turning the product into a policy document.

---

## 2) Target users & JTBD
### Primary persona
**Value-seeking traveler** booking a flight with constraints (time, stops, airline, baggage, refundability) who wants a booking they can trust.

### JTBD
“Help me find the best flight for my constraints quickly, and send me to a booking path that won’t surprise me on price or rules.”

---

## 3) Goals / Non-goals
### Goals (v3)
1) End-to-end funnel: **Search → SRP → Detail → Partner click-out**.
2) Trust differentiation:
   - price confidence + freshness
   - rules clarity (baggage/change/refund)
   - partner reliability + transparent labels
3) Marketplace foundations:
   - partner governance signals + penalty ladder
   - mismatch/complaint feedback loops
4) Operational excellence:
   - predictable latency + high availability
   - observable funnel with debugging tools

### Non-goals (explicit for v3)
- Owning checkout/payment (metasearch only)
- Hotels/cars
- Bundles, loyalty, subscriptions
- Full itinerary management (“Trips”)
- Multi-city search
- Price freeze / guarantee / “final price” (we provide confidence + recency + accountability signals)

---

## 4) Launch scope (locked for v3)
### Market + routes
**MVP pick (required):** choose exactly one market for v3:
- **India domestic + nearby international (recommended)**
- UK + EU short-haul
- US domestic (higher complexity)

**Route set:** top **50–150 routes** by demand (keep narrow to build trust metrics fast).

### Platform
- Web first: mobile web + desktop
- App out of scope for v3

### Partners (MVP)
- Airline direct links (where available)
- **1–2 OTAs** max at launch
- Requirement: partner must support stable deep links + attribution params; otherwise they ship in later ramp.

---

## 5) Success metrics
### North Star
**Trusted qualified click-outs per search session**
> A click-out that does not quickly bounce back and has low mismatch probability (or low mismatch reporting).

### Core funnel
- Search → SRP rate
- SRP → Itinerary detail open rate
- Detail → Partner click-out rate
- **Bounce-back rate:** return within 30–120s of click-out
- **Mismatch proxy rate:** partner landing price ≠ shown price (user report + automated checks where feasible)
- Repeat searches (7/30-day)

### Guardrails
- SRP **TTFR p95**
- Redirect success rate
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

**Edge cases**
- Invalid/unsupported routes: show “No results for this route yet” + suggestions.
- Cabin/traveler limits enforced at UI + API.

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

**v3 SRP requirements**
- Default sort = **Best** (multi-objective)
- Inline “Direct only” toggle
- Price + duration always visible
- **Clustering:** group offers by *identical itinerary* (same legs/flight numbers/airports/times), with multiple sellers inside
  - SRP shows best offer per cluster + “X sellers” affordance
- Show **offer freshness**: “Updated X min ago”
- Show **price confidence** badge (High/Med/Low) at least on top offers
- “Sold out” / “Price changed” states per seller inside a cluster (do not remove silently)

**Error/empty states**
- If aggregation fails partially: show partial results with banner “Some partners unavailable-showing available results.”
- If no results: show filters reset + alternative dates (if available) + nearby airports suggestion.

**Performance & perceived latency**
- Progressive results: show partial results while completing aggregation
- Cancellable filter updates (avoid blocking UI)

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

**UX behavior**
- If rules are Unknown: show compact explainer + “Details at partner” disclaimer (do not pretend certainty).
- If price confidence is Low: show “Prices may change at checkout” + “Why?” modal with factors.

**Acceptance criteria**
- At least 2 partner options for top itineraries in launch markets (where supply exists).
- Partner row shows: partner name, price, trust label, caveats.

---

### 7.4 Booking handoff (redirect)
**Requirements**
- Construct deep link/payload to partner
- Include attribution parameters
- Preserve locale, currency, pax count
- Maintain consistent **offer_id** (or derived signature) through click-out

**Redirect UX requirements**
- Interstitial behavior for v3:
  - **ON by default** for “New” or “Caution” partners
  - Optional for “Trusted” partners (experiment)
- Interstitial shows:
  - “Redirecting to {partner}”
  - price shown + “Updated X min ago”
  - lightweight disclaimer + “Report a price issue” link

**Failure modes**
- If partner deep link fails: show error page with retry + alternative sellers.
- If currency/locale mismatch risk detected: show warning (“Partner may display a different currency”).

**Acceptance criteria**
- Click-out logs include: search_id, itinerary_id, offer_id, partner_id, price_shown, fetched_at, timestamp, device, locale/currency.

---

## 8) The Trust Pack (v3 spec)
### Components (user-facing)
1) **Price confidence** (High / Medium / Low)
2) **Rules clarity** (Clear / Partial / Unknown)
3) **Partner reliability** (Trusted / New / Caution)

### Underlying scoring (v3)
All three chips are derived from a **0–100 score** each, then bucketed.

#### 8.1 Price Confidence score (0–100)
**Inputs** (minimum viable)
- Partner mismatch rate (route-level preferred, fallback global)
- Offer age (minutes since fetched_at)
- Route volatility bucket (learned from historical reprice frequency)
- “Requires reprice” flag frequency (partner capability)

**Bucketing**
- High: ≥ 80
- Medium: 50–79
- Low: < 50

#### 8.2 Rules Clarity score (0–100)
**Inputs**
- Completeness of: baggage, changes, refunds fields
- Normalization success rate by partner

**Bucketing**
- Clear: ≥ 80
- Partial: 40–79
- Unknown: < 40

#### 8.3 Partner Reliability score (0–100)
**Inputs**
- Complaint/report rate per 1k click-outs
- Bounce-back rate (30–120s)
- Mismatch proxy rate
- Deep-link failure rate

**Bucketing**
- Trusted: ≥ 80
- New: insufficient volume (see governance thresholds)
- Caution: < 80 once volume threshold met

### Calibration plan (must-have for v3)
- Start with heuristic weights; re-fit monthly using observed mismatch + bounce-back outcomes.
- Keep a “reason codes” log for each score (top 2 factors) to power “Why?” UI and internal debugging.

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

### v3 approach
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

### Guardrails
- Never hide the cheapest offer for a cluster; trust affects ranking and badges, not availability.

---

## 10) Offer freshness & caching
**Why:** stale offers increase mismatch and bounce-backs.

**Requirements**
- Store **fetched_at** per offer.
- Display “Updated X min ago” when age exceeds threshold.
- Thresholds (v3 defaults, configurable):
  - High freshness: ≤ 10 min
  - Medium: 11–30 min
  - Stale: > 30 min
- If stale: show “Prices may have changed” warning (and consider “Refresh prices” as v4 candidate).

**Acceptance criteria**
- Offers older than threshold are labeled; click-outs still tracked.

---

## 11) Partner marketplace & governance (v3)
### Launch strategy
- Start with small set of high-quality partners.

### Governance metrics (tracked per partner + per route)
- mismatch proxy rate
- bounce-back rate
- report/complaint rate
- deep-link failure rate

### Volume thresholds
- **New partner** state until ≥ **500 click-outs** (or 30 days, whichever comes first).
- Only apply strong penalties after meeting threshold.

### Penalty ladder
1) Add warning label on partner row (“Price issues reported”)
2) Ranking penalty (reduce exposure)
3) Route-level removal
4) Market-level removal

### Partner transparency (internal ops requirement)
- Weekly partner report (email/dashboard) showing:
  - mismatch rate trend
  - top routes impacted
  - deep-link errors
  - recommended fixes

---

## 12) Mismatch Program (v3)
### Detection
- **User report**: “Price changed” report flow post-click-out and/or on return
- **Bounce-back proxy**: returned_from_partner within 30–120s
- **Automated sampling** (optional): headless checks on subset of offers where allowed

### User experience
- Report UI (v3 must-have):
  - “Price was higher / lower / flight not available / rules different / other”
  - optional free text
  - optional screenshot upload (out of scope for v3)

### Enforcement
- Dashboard + alerts for spikes.
- Auto-create incident when mismatch spikes > threshold on high-volume routes.

---

## 13) Data & instrumentation (v3)
### Entities
- Search
- Itinerary (normalized)
- Offer (itinerary + partner + price)
- Click-out
- Return event (bounce-back)
- Feedback / mismatch report

### Identifiers (contract)
- **search_id**: UUID per submitted search
- **itinerary_signature**: stable hash of normalized legs (flight numbers + airports + times)
- **offer_id**: hash(search_id, itinerary_signature, partner_id, price, fetched_at) or partner offer token where available
- **click_id**: UUID generated at click-out time (primary key for redirect + return attribution)

### Event schema (minimum viable)
- search_submitted {search_id, query, locale, currency}
- srp_viewed {search_id}
- filter_changed {search_id, filters}
- itinerary_opened {search_id, itinerary_signature}
- partner_clicked_out {search_id, itinerary_signature, offer_id, click_id, partner_id, price_shown, fetched_at, ts}
- returned_from_partner {search_id, click_id, ts, time_away_sec}
- mismatch_reported {search_id, click_id, type, delta?, free_text?}

### Attribution requirements
- Carry click_id to partner as params where allowed.
- If partner strips params: use fallback (referrer + time-window) but mark attribution_quality=low.

---

## 14) Privacy, compliance, and accessibility
### Privacy
- Minimize PII: do not log passenger names, emails, or payment data.
- Document retention for IP/device identifiers.
- Provide user-facing disclosure for redirects and tracking.

### Accessibility
- Keyboard navigable search and SRP filters.
- SRP cards and partner list meet contrast standards.
- Screen-reader labels for Trust Pack chips and badges.

---

## 15) Performance & reliability SLOs
### User-facing performance
- SRP TTFR p95:
  - **Mobile web:** ≤ 2.5s
  - **Desktop:** ≤ 1.8s
- Filter interaction p95 (time to visibly updated results): **≤ 1.5s**
- Detail page load p95: **≤ 2.0s**

### Redirect reliability
- Partner click-out redirect success rate: **≥ 99.5%** (no hard errors)
- Deep-link validation coverage: **100%** for launch partners/routes

### Observability (must-have)
- Dashboards: funnel, latency, partner errors, mismatch spikes
- Alerts: deep-link failures, mismatch spikes, TTFR regression

---

## 16) Rollout plan + experiments + go/no-go
### Rollout
1) Private alpha (internal + friends) on limited routes
2) Beta in a single market (v3 launch market)
3) Expand routes within market
4) Expand partners once trust metrics stabilize

### Experiments
- Interstitial ON vs OFF for Trusted partners (keep ON for New/Caution)
- Best ranking weights: trust-heavy vs price-heavy
- Trust Pack display density: chips-only vs chips + “Why?” affordance

### Go/no-go gates
- Mismatch proxy rate below threshold for 2 consecutive weeks
- Bounce-back rate stable/improving
- TTFR p95 within SLO
- No critical partner deep-link failures

---

## 17) Open questions
1) Which launch market (geo/routes) do we target ?
2) Which partners are available and what deep-link/attribution constraints exist?
3) Minimum viable rules normalization: which baggage/refund fields must be present for “Clear”?
4) Do we optimize primarily for cheapest or for “trusted best” as the default?
