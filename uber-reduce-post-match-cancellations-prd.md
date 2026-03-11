# PRD: Reduce Post‑Match Cancellations in Uber via a “Pickup Quality Pack”

**Product:** Uber — Ride-hailing (Rider + Driver apps)  
**Author:** Mayank Malviya  
**Status:** v1 — Problem framing + v1 requirements + instrumentation ready for design/eng review

---

## Changelog
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

**Opportunity:** Reduce post‑match cancel rate by meaningfully improving pickup coordination without adding checkout friction.

---

## 2) Goals / non-goals
### Goals
1. **Reduce post‑match cancellations** (rider + driver) in targeted geos/locations.
2. Reduce **arrive → trip start** time (p90) and **pickup confusion** signals.
3. Reduce pickup-related **support contact rate**.

### Non-goals
- Changing core dispatch / pricing algorithms (out of scope for v1).
- Full redesign of the end-to-end booking flow.
- Solving all airport/event pickup complexity (v1 will start with high-frequency patterns).

---

## 3) Success metrics & guardrails
### Primary success
- **Post‑match cancellation rate** ↓ (overall and split by: rider-cancel, driver-cancel)

### Secondary success
- **Arrive → Start time (p90)** ↓
- **Pickup confusion proxy** ↓ (call/chat usage rate during pickup window, reroute count, pin edits)
- **Support contact rate (pickup-related)** ↓

### Guardrails
- **Checkout conversion** (destination_set → request_confirmed) does not meaningfully decrease
- **Driver acceptance rate** does not decrease in treated cohorts/geos
- **False-positive friction** (extra steps shown to “easy pickups”) stays low
- **Safety**: no increase in reported safety incidents attributable to pickup changes

---

## 4) Personas / contexts in scope
### Rider contexts (high leverage)
- Apartments with multiple gates/entrances
- Malls / office parks with large footprints
- Night-time pickups (higher safety + confusion risk)
- Tourists / unfamiliar areas (address entry and landmark confusion)

### Driver contexts
- Drivers facing **no-stopping** constraints or security checkpoints
- Multi-apping drivers (high opportunity cost; more likely to cancel on hard pickups)

---

## 5) Key insight (from teardown)
Pickup failures are often predictable from observable signals:
- Repeated “pin moved” edits
- High call/chat usage right after arrival
- Known POIs with constrained pickup zones
- GPS drift (speed + heading + stop location mismatch)

So v1 should:
1) Detect likely-hard pickups early,  
2) Guide both sides with minimal, contextual prompts,  
3) Provide drivers a “best stopping point” and clear rider rendezvous instructions.

---

## 6) Proposed solution: Pickup Quality Pack (v1)
A feature bundle that upgrades pickup coordination with **three shippable modules** behind flags.

### Module A — Rider Guided Pickup Pin (contextual)
**What:** When the system predicts a “hard pickup”, show a lightweight guided flow to confirm pickup:
- Side of road (if relevant)
- Entrance/gate selection (if relevant)
- Landmark confirmation (from POI + common text)
- Optional rider note templates (e.g., “I’m at Gate 2, blue shirt”) with 1‑tap selection

**Principle:** Only show this flow when prediction confidence is high, otherwise keep default “set pickup” UX.

### Module B — Driver “Best Stop” + Pickup Difficulty Cue
**What:** On driver navigation/arrival:
- Show a “best stop” waypoint (where stopping is feasible) when available
- Label pickup as **Easy / Medium / Hard** with the specific reason (e.g., “Multi-entrance building”)
- Provide 1‑tap suggested message templates (“I’m at the north entrance”) tied to the rider’s selected entrance

### Module C — Pickup State Machine + Rendezvous Cues
**What:** Make the arrival moment explicit and symmetric:
- Rider: a clear “Your driver is at *[entrance/landmark]*” card with car color + plate + live position + walking cue
- Driver: “Waiting at *[spot]*” status with a timer and guidance
- “I’m coming out now” / “I’m at pickup” 1‑tap confirmations to reduce uncertainty

---

## 7) Detailed requirements (v1)
### 7.1 Eligibility / triggering
1. Only trigger modules A/B/C when **pickup difficulty score** exceeds threshold OR pickup is a known constrained POI.
2. Maintain a strict **time budget**: no added step should block booking for more than 2–3 seconds; if services fail, fall back to baseline UX.

### 7.2 Rider guided pickup (Module A)
**Requirements**
- Add a pre-confirm micro-step **only for eligible rides**:
  - Step content must fit in ≤2 screens
  - Default selections pre-filled when possible
- Rider can skip with “Looks good” (track as `guided_pickup_skipped`).

**Acceptance criteria**
- For treated eligible rides, **pin edits after match** decrease.
- No measurable decrease in checkout conversion for non-eligible rides (should be zero exposure).

### 7.3 Driver cues (Module B)
**Requirements**
- “Best stop” should be optional: if confidence low, do not show.
- Difficulty cue must be explainable (not a black box label).
- Provide message templates that auto-include location strings agreed with rider selection.

**Acceptance criteria**
- Driver cancels due to “hard pickup” reason code decreases in treated rides.
- Driver app distraction is minimized (no more than one extra tappable surface on arrival screen).

### 7.4 Rendezvous cues (Module C)
**Requirements**
- Rider screen must surface:
  - car + plate + “where to meet” in plain language
  - safety-friendly details (avoid oversharing exact rider live location beyond what’s necessary)
- Driver screen must surface:
  - where rider is expected to come
  - time-to-wait guidance + no-show policy hint

**Acceptance criteria**
- Arrive → start time (p90) decreases in treated eligible rides.
- Pickup-related support contacts decrease.

---

## 8) Instrumentation (events + properties)
### Core events
- `pickup_difficulty_scored`
  - props: `difficulty_score`, `reason_codes[]`, `poi_type`, `geo_id`, `model_version`
- `guided_pickup_shown`
  - props: `step_count`, `preselected_fields[]`
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

## 9) Experiment plan
### Phase 0 — Offline validation (1–2 weeks)
- Backtest pickup difficulty scoring against historical proxies:
  - pin edits after match
  - call/chat during pickup window
  - arrive→start p90
  - cancel reason codes

### Phase 1 — Limited geo A/B (2–4 weeks)
- Roll out to 1–2 cities, targeted POIs (malls + multi-entrance residential clusters)
- A/B: eligible rides only
- Success = reduction in post‑match cancels + arrive→start p90 without harming acceptance/conversion

### Phase 2 — Expand + harden (4–8 weeks)
- Add more POI templates (airports/events as separate tracks)
- Tune thresholds to minimize unnecessary prompts

---

## 10) Risks & mitigations
1. **Added friction hurts conversion** → strict eligibility + skip + latency budgets.
2. **Prediction errors** annoy users/drivers → explainability + conservative triggers.
3. **Safety concerns** (meeting instructions cause unsafe crossing/standing) → safety copy + avoid suggesting illegal/unsafe stops.
4. **Driver compliance** (ignores “best stop”) → present as suggestion, not mandate; measure adherence.

---

## 11) Open questions (for eng/design/ops)
- What pickup reason codes exist today for cancels, and how reliable are they?
- Can we map POIs to pickup zones consistently (per city) without heavy ops overhead?
- What is the best mechanism for “best stop” suggestions (maps data source, confidence scoring)?
- How should we localize pickup instructions across languages while keeping them short?

---

## Source
Derived from the teardown: `product-teardowns/uber-ridehailing-teardown.md` (pickup loop, cancellation taxonomy, and “Pickup Quality Pack” bet).