# PRD: Reduce Post‑Match Cancellations in Uber via a “Pickup Quality Pack”

**Product:** Uber — Ride-hailing (Rider + Driver apps)  
**Author:** Mayank Malviya  
**Status:** v2 — MVP scope locked + clearer taxonomy + experiment/ops plan (ready for design/eng review)

---

## Changelog
- **v2 (2026-03-12):** Tightened MVP scope (what ships vs later), added cancellation taxonomy + targeting strategy, clarified pickup difficulty scoring inputs/outputs, added rollout/ops plan, and expanded requirements + edge cases.
- **v1 (2026-03-11):** First cut PRD derived from `product-teardowns/uber-ridehailing-teardown.md` (pickup loop + cancellation taxonomy). Focused on a single outcome: reduce post‑match cancellations by improving pickup rendezvous success.

---

## 1) Problem statement
Post‑match cancellations (after a rider is matched to a driver) are among the highest-cost failures in ride-hailing:
- They **waste time** for both sides (driver deadhead + rider waiting)
- They **erode trust** (“Uber is unreliable”, “drivers keep canceling”)
- They increase **support load** (refunds, fare disputes, safety concerns)
- They reduce marketplace efficiency (more rematching, higher ETAs, lower supply utilization)

The pickup moment is uniquely failure-prone because it combines:
- GPS ambiguity (pin drift, multi-entrance buildings, malls/airports)
- Human coordination (finding the right car/person)
- Local constraints (no-stopping zones, gates, security)

**Opportunity:** Reduce post‑match cancel rate by improving pickup coordination **only when needed** (high-confidence “hard pickups”), without adding baseline booking friction.

---

## 2) Goals / non-goals
### Goals
1. **Reduce post‑match cancellations** (rider + driver) in targeted geos/locations.
2. Reduce **arrive → trip start** time (p90) and pickup confusion signals.
3. Reduce pickup-related **support contact rate**.

### Non-goals
- Changing core dispatch / pricing algorithms (out of scope for v2).
- Full redesign of end-to-end booking flow.
- Solving every airport/event pickup complexity in one go (treat airports/events as separate templates/rollouts).

---

## 3) Success metrics & guardrails
### Primary success
- **Post‑match cancellation rate** ↓ (overall and split by: rider-cancel, driver-cancel)

### Secondary success
- **Arrive → Start time (p90)** ↓
- **Pickup confusion proxy** ↓ (call/chat usage during pickup window, reroute count, pin edits)
- **Support contact rate (pickup-related)** ↓

### Guardrails
- **Checkout conversion** (destination_set → request_confirmed) does not meaningfully decrease
- **Driver acceptance rate** does not decrease in treated cohorts/geos
- **Match latency** does not regress (p95)
- **False-positive friction** (extra steps shown to “easy pickups”) stays low
- **Safety**: no increase in reported safety incidents attributable to pickup changes

### Metric segmentation (required)
Report all primary/secondary/guardrails split by:
- Geo (city), time-of-day, rider tenure (new vs returning), POI type
- Eligibility bucket (difficulty decile / Easy-Medium-Hard)
- Exposure flags (A/B/C) and “skipped” vs “completed”

---

## 4) Cancellation taxonomy (v2)
We need a taxonomy that is actionable (design/ops) and measurable (instrumentation).

### 4.1 Post‑match cancellation buckets
**Rider-cancel (after match)**
- Confusion: can’t find driver / wrong side / wrong entrance
- Safety discomfort at pickup
- ETA too long / driver not moving
- Changed mind / alternative transport

**Driver-cancel (after match)**
- Hard pickup / rider not found
- Illegal stop / no-stopping / enforcement risk
- Rider not ready / long wait / poor communication
- Trip not worth it (distance, direction) — *not targeted by this PRD*

### 4.2 “Pickup-related” vs “non-pickup-related”
For v2 attribution, treat as pickup-related if any are true:
- Cancel happens **after driver arrival** OR within X minutes of arrival ETA
- Call/chat happens in pickup window
- Pin moved after match
- Driver navigation reroute loops near pickup

We should explicitly exclude “trip economics” cancels from success claims (still may move as a side effect).

---

## 5) Personas / contexts in scope
### Rider contexts (high leverage)
- Apartments with multiple gates/entrances
- Malls / office parks with large footprints
- Night-time pickups (higher safety + confusion risk)
- Tourists / unfamiliar areas

### Driver contexts
- Drivers facing **no-stopping** constraints or security checkpoints
- Multi-apping drivers (high opportunity cost; more likely to cancel on hard pickups)

---

## 6) Key insight (from teardown)
Pickup failures are often predictable from observable signals:
- Repeated “pin moved” edits
- High call/chat usage right after arrival
- Known POIs with constrained pickup zones
- GPS drift (speed + heading + stop location mismatch)

So v2 should:
1) Detect likely-hard pickups early,  
2) Guide both sides with minimal, contextual prompts,  
3) Provide drivers a “best stopping point” and clear rider rendezvous instructions.

---

## 7) Proposed solution: Pickup Quality Pack
A feature bundle that upgrades pickup coordination with three modules, all behind flags.

### Module A — Rider Guided Pickup Pin (contextual)
**What:** When predicted “hard pickup”, show a lightweight guided flow to confirm pickup:
- Entrance/gate selection (if relevant)
- Landmark confirmation (from POI + common text)
- Optional rider note templates (e.g., “I’m at Gate 2, blue shirt”) with 1‑tap selection

**Principle:** Only show when prediction confidence is high; otherwise keep default “set pickup” UX.

### Module B — Driver “Best Stop” + Pickup Difficulty Cue
**What:** On driver navigation/arrival:
- Show a “best stop” waypoint (where stopping is feasible) when available
- Label pickup as **Easy / Medium / Hard** with the specific reason (e.g., “Multi-entrance building”)
- Provide 1‑tap suggested message templates (“I’m at the north entrance”) tied to rider’s selection

### Module C — Pickup State Machine + Rendezvous Cues
**What:** Make arrival explicit and symmetric:
- Rider: “Your driver is at *[entrance/landmark]*” card with car + plate + live position + walking cue
- Driver: “Waiting at *[spot]*” status with timer and guidance
- “I’m coming out now” / “I’m at pickup” 1‑tap confirmations to reduce uncertainty

---

## 8) MVP scope (what ships in v2)
### 8.1 MVP = ship now
1. **Eligibility + difficulty scoring v0** (rules-first + simple model optional)
2. **Module A (Rider Guided Pickup) — minimal flow**
   - entrance/gate selection (when POI template supports it)
   - 1‑tap note templates
   - skip path
3. **Module B — difficulty cue + templates**
   - difficulty label + reason codes
   - 1‑tap message templates (no “best stop” if confidence isn’t strong)
4. **Module C — explicit arrival card + 1‑tap confirmations**
5. **Instrumentation + experiment framework** for eligibility-only A/B

### 8.2 Not in MVP (later)
- Full “best stop” routing optimization across all maps sources (ship only for a few templated POIs later)
- Airport/event pickup (separate rollouts)
- Deep personalization (rider preference learning)

---

## 9) Pickup Difficulty Scoring (v0)
We need an interpretable score that can drive eligibility and explainability.

### 9.1 Inputs (candidate signals)
- POI template match (mall, multi-entrance residential, office park)
- Historical pickup friction at pickup cell/POI: pin edits, call/chat, arrive→start p90
- GPS drift risk proxies (urban canyon density, known low-accuracy zones)
- Rider context: new rider, language mismatch (if available), late night

### 9.2 Outputs
- `difficulty_score` in [0, 1]
- `difficulty_bucket` ∈ {easy, medium, hard}
- `reason_codes[]` (top 1–3) used to explain prompts

### 9.3 Eligibility rule (v0)
Eligible if any:
- `difficulty_bucket == hard`
- POI is in curated constrained list
- historical friction above threshold (per geo)

Hard requirement: **no eligibility dependency on fragile services** at booking time; if scoring times out, fall back to baseline UX.

---

## 10) Detailed requirements
### 10.1 Global requirements
1. **Latency budgets**
   - Booking step add-on must not add more than ~2–3s p95.
   - If scoring/template service fails, use baseline pickup UX.
2. **Minimal added friction**
   - Guided step shown only to eligible rides.
   - Always offer a quick “Looks good” path.
3. **Explainability**
   - If we label “Hard pickup”, show a human-readable reason.

### 10.2 Rider guided pickup (Module A)
**Requirements**
- Add a pre-confirm micro-step only for eligible rides:
  - content fits in ≤2 screens
  - default selections pre-filled when possible
  - supports “skip” at every step
- Rider-generated pickup note templates:
  - provide 3–5 templates max
  - safe-by-default (no sensitive info prompts)

**Acceptance criteria**
- For treated eligible rides, pin edits after match decrease.
- No measurable decrease in checkout conversion for non-eligible rides (should be zero exposure).

**Edge cases**
- Rider changes pickup after match: re-run eligibility and update shared rendezvous string.
- Accessibility: voice-over friendly labels; avoid map-only instructions.

### 10.3 Driver cues (Module B)
**Requirements**
- Difficulty cue must be explainable and not overly alarming.
- Suggested message templates:
  - localized
  - include rider-selected entrance string
  - one-tap to send; optional edit
- Optional “best stop”:
  - only show when confidence high; otherwise hide entirely.

**Acceptance criteria**
- Driver cancels due to “hard pickup / can’t find rider” reason codes decrease in treated rides.
- Driver distraction minimized (no more than one extra tappable surface on arrival screen).

### 10.4 Rendezvous cues (Module C)
**Requirements**
- Rider screen surfaces:
  - car + plate + “where to meet” in plain language
  - safety-friendly details (avoid oversharing beyond what’s necessary)
- Driver screen surfaces:
  - where rider is expected to come
  - time-to-wait guidance + no-show policy hint
- Confirmations:
  - “I’m coming out now” (rider)
  - “I’m at pickup spot” (driver)

**Acceptance criteria**
- Arrive → start time (p90) decreases in treated eligible rides.
- Pickup-related support contacts decrease.

---

## 11) Instrumentation (events + properties)
### Core events
- `pickup_difficulty_scored`
  - props: `difficulty_score`, `difficulty_bucket`, `reason_codes[]`, `poi_type`, `geo_id`, `model_version`
- `guided_pickup_shown`
  - props: `step_count`, `preselected_fields[]`, `eligibility_reason`
- `guided_pickup_completed`
  - props: `selected_entrance`, `landmark_used`, `note_template_id`
- `guided_pickup_skipped`
  - props: `skip_point`
- `best_stop_shown`
  - props: `confidence`, `delta_distance_m`, `reason_code`
- `pickup_message_template_sent`
  - props: `sender` (rider/driver), `template_id`
- `pickup_arrival_state_changed`
  - props: `state` (enroute/arrived/waiting/rider_confirmed/driver_confirmed)

### Outcome attribution helpers
- `post_match_cancel`
  - props: `side` (rider/driver), `reason_code`, `time_since_match_s`, `pickup_difficulty_bucket`, `module_exposure_flags`
- `arrive_to_start_time`
  - props: `seconds`, `pickup_difficulty_bucket`, `module_exposure_flags`
- `pickup_support_contact`
  - props: `issue_tag`, `time_since_trip_s`, `module_exposure_flags`

---

## 12) Experiment plan
### Phase 0 — Offline validation (1–2 weeks)
- Backtest scoring against historical proxies:
  - pin edits after match
  - call/chat during pickup window
  - arrive→start p90
  - cancel reason codes

### Phase 1 — Limited geo A/B (2–4 weeks)
- Roll out to 1–2 cities + targeted POIs (malls + multi-entrance residential clusters)
- A/B: eligible rides only
- Success = reduction in post‑match cancels + arrive→start p90 without harming acceptance/conversion

### Phase 2 — Expand + harden (4–8 weeks)
- Add more POI templates (airports/events as separate tracks)
- Tune thresholds to minimize unnecessary prompts

### Analysis plan (must-have)
- Report lift overall + by difficulty decile.
- Estimate “treatment on treated” (eligible-only) vs overall impact.
- Check adverse shifts: driver acceptance, match latency, support contacts.

---

## 13) Ops / rollout considerations
- **POI template management:** start with a curated list maintained per geo; build tooling later.
- **Localization:** keep templates short; validate with top languages per geo.
- **Support readiness:** add macros for pickup-quality-pack flows + new reason code mapping.
- **Driver comms:** ensure “Hard pickup” label doesn’t feel punitive; framing = “more guidance available”.

---

## 14) Risks & mitigations
1. **Added friction hurts conversion** → strict eligibility + skip + latency budgets.
2. **Prediction errors annoy users/drivers** → explainability + conservative triggers.
3. **Safety concerns** (meeting instructions cause unsafe crossing/standing) → safety copy + avoid suggesting illegal/unsafe stops.
4. **Driver compliance** (ignores suggestions) → present as suggestion, not mandate; measure adherence.

---

## 15) Open questions (for eng/design/ops)
- What pickup reason codes exist today for cancels, and how reliable are they?
- Can we map POIs to pickup zones consistently (per city) without heavy ops overhead?
- What is the best mechanism for “best stop” suggestions (maps data source, confidence scoring)?
- How should we localize pickup instructions across languages while keeping them short?
- Where should the “guided pickup” live in the booking UX so it’s least disruptive?

---

## Source
Derived from the teardown: `product-teardowns/uber-ridehailing-teardown.md` (pickup loop, cancellation taxonomy, and “Pickup Quality Pack” bet).
