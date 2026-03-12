# PRD: Reduce Post‑Match Cancellations in Uber via a “Pickup Quality Pack”

**Product:** Uber — Ride-hailing (Rider + Driver apps)  
**Author:** Mayank Malviya  
**Status:** v3 — User journeys + UX copy + service dependencies + rollout checklist (ready for exec + eng kickoff)

---

## Changelog
- **v3 (2026-03-13):** Added end-to-end user journeys, concrete UX copy placeholders, clarified system/service dependencies + failover, expanded experimentation (ramp + stop criteria), added rollout checklist and reason-code dictionary for explainability.
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
- Changing core dispatch / pricing algorithms (out of scope for v3).
- Full redesign of end-to-end booking flow.
- Solving every airport/event pickup complexity in one go (airports/events are separate templates/rollouts).

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
Report all metrics split by:
- Geo (city), time-of-day, rider tenure (new vs returning), POI type
- Eligibility bucket (difficulty decile / Easy-Medium-Hard)
- Exposure flags (A/B/C) and “skipped” vs “completed”

---

## 4) Cancellation taxonomy
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
For attribution, treat as pickup-related if any are true:
- Cancel happens **after driver arrival** OR within X minutes of arrival ETA
- Call/chat happens in pickup window
- Pin moved after match
- Driver navigation reroute loops near pickup

Explicitly exclude “trip economics” cancels from success claims (they may move as a side effect).

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

## 6) Key insight
Pickup failures are often predictable from observable signals:
- Repeated “pin moved” edits
- High call/chat usage right after arrival
- Known POIs with constrained pickup zones
- GPS drift (speed + heading + stop location mismatch)

So v3 should:
1) Detect likely-hard pickups early,  
2) Guide both sides with minimal, contextual prompts,  
3) Provide drivers clear, shared rendezvous instructions (and later, “best stop”).

---

## 7) Proposed solution: Pickup Quality Pack
A feature bundle that upgrades pickup coordination with three modules, behind flags.

### Module A — Rider Guided Pickup Pin (contextual)
**What:** When predicted “hard pickup”, show a lightweight guided flow to confirm pickup:
- Entrance/gate selection (if relevant)
- Landmark confirmation (from POI template + common text)
- Optional rider note templates (e.g., “I’m at Gate 2, blue shirt”) with 1‑tap selection

**Principle:** Only show when prediction confidence is high; otherwise keep default “set pickup” UX.

### Module B — Driver Pickup Difficulty Cue + Templates (+ optional best stop)
**What:** On driver navigation/arrival:
- Label pickup as **Easy / Medium / Hard** with a specific reason
- Provide 1‑tap suggested message templates tied to rider’s selection
- Optional: show a “best stop” waypoint only when confidence is strong (otherwise hide)

### Module C — Pickup State Machine + Rendezvous Cues
**What:** Make arrival explicit and symmetric:
- Rider: “Your driver is at *[entrance/landmark]*” card with car + plate + live position + walking cue
- Driver: “Waiting at *[spot]*” status with timer and guidance
- “I’m coming out now” / “I’m at pickup” 1‑tap confirmations

---

## 8) MVP scope (v3 = shipping plan)
### 8.1 MVP = ship now
1. **Eligibility + difficulty scoring v0** (rules-first; model optional)
2. **Module A — minimal guided pickup flow**
   - entrance/gate selection (only where POI template supports it)
   - 1‑tap note templates
   - skip path
3. **Module B — difficulty cue + message templates** (no best stop by default)
4. **Module C — arrival card + 1‑tap confirmations**
5. **Instrumentation + experiment framework** (eligible-only A/B)

### 8.2 Not in MVP (later)
- Full “best stop” routing optimization at scale (ship only for a few templated POIs later)
- Airports/events pickup flows
- Deep personalization (learn rider preferences)

---

## 9) End-to-end user journeys (v3)
These journeys define the exact surfaces and state transitions we’ll design/build.

### Journey 1 — Hard pickup, rider completes guided flow
1. Rider enters pickup + destination → system computes eligibility
2. If eligible → Module A shown (≤2 screens)
   - Rider selects entrance: “North Gate”
   - Rider selects a note template: “I’m at North Gate, near the security cabin.”
3. Match occurs
4. Driver sees Module B cue: “Hard pickup: multi‑entrance location” + template suggestion
5. On arrival → Module C activates
   - Rider sees “Driver waiting at North Gate” + plate + walking hint
   - Rider taps “I’m coming out now”
6. Driver sees rider confirmation; trip starts

### Journey 2 — Hard pickup, rider skips guided flow
1. Eligible → Module A shown → Rider taps “Looks good”
2. Match occurs
3. Driver sees difficulty cue + generic template: “I’m at the pickup point shown in app.”
4. Module C still shows rendezvous card with baseline pickup string

### Journey 3 — Easy pickup, no intervention
1. Not eligible → baseline UX only
2. No exposure flags set

### Journey 4 — Rider moves pin after match
1. Rider edits pickup after match
2. Recompute difficulty + refresh rendezvous string
3. Notify driver with non-spammy banner (“Pickup updated: North Gate”) + quick acknowledge

---

## 10) UX copy placeholders (non-final)
Copy matters here (trust + safety + conversion). Treat as draft.

### Module A (rider)
- Title: “Confirm your pickup spot”
- Subtitle (reason-based):
  - “This location has multiple entrances. Picking one helps your driver find you faster.”
- Primary CTAs:
  - “Looks good” (skip)
  - “Choose entrance”
- Entrance selector label: “Where should your driver meet you?”
- Note template header: “Quick message (optional)”

### Module B (driver)
- Difficulty chip:
  - “Hard pickup” (tap → reason)
- Reason examples:
  - “Multi‑entrance building”
  - “No‑stopping zone nearby”
- Templates:
  - “I’m waiting at {entrance_name}.”
  - “I’m at {landmark}. Please meet me there.”

### Module C (rider + driver)
- Rider card: “Driver waiting at {spot}. {car_color} {car_model}, plate {plate}.”
- Rider CTA: “I’m coming out now”
- Driver card: “Waiting at {spot}. Rider expected at {spot}.”
- Driver CTA: “I’m at pickup spot”

---

## 11) Pickup Difficulty Scoring (v0)
We need an interpretable score that drives eligibility and explainability.

### 11.1 Inputs (candidate signals)
- POI template match (mall, multi-entrance residential, office park)
- Historical pickup friction at pickup cell/POI: pin edits, call/chat, arrive→start p90
- GPS drift risk proxies (urban canyon density, known low-accuracy zones)
- Rider context: new rider, language mismatch (if available), late night

### 11.2 Outputs
- `difficulty_score` in [0, 1]
- `difficulty_bucket` ∈ {easy, medium, hard}
- `reason_codes[]` (top 1–3) used to explain prompts

### 11.3 Eligibility rule (v0)
Eligible if any:
- `difficulty_bucket == hard`
- POI is in curated constrained list
- historical friction above threshold (per geo)

Hard requirement: **no dependency on fragile services** at booking time; on timeout → baseline UX.

---

## 12) System dependencies & failure modes (v3)
### 12.1 Services / components
1. **Difficulty Scorer**
   - Input: pickup lat/lng, geo_id, POI metadata, (optional) rider context
   - Output: score/bucket/reasons
2. **POI Template Service**
   - Maps pickup to template: entrances/gates, canonical meeting strings
3. **Message Template Generator**
   - Localized strings with variable substitution
4. **Rendezvous State Store**
   - Persists rider selection (entrance/landmark/note) for both apps

### 12.2 Failover requirements
- If scorer fails/slow → treat as not eligible (no extra steps)
- If POI template missing → Module A falls back to note templates only
- If state store fails after match → do not block; show baseline pickup pin on both sides

---

## 13) Detailed requirements
### 13.1 Global requirements
1. **Latency budgets**
   - Booking step add-on must not add more than ~2–3s p95.
   - If scoring/template fails, fall back instantly.
2. **Minimal friction**
   - Guided step shown only to eligible rides.
   - Always offer a quick “Looks good” path.
3. **Explainability**
   - “Hard pickup” must have a human-readable reason (no black box label).
4. **Privacy & safety**
   - No prompts that encourage oversharing (“exact outfit”, “phone number”).
   - Avoid instructions that suggest unsafe crossing/standing.

### 13.2 Module A — Rider guided pickup
**Requirements**
- Pre-confirm micro-step only for eligible rides:
  - content fits in ≤2 screens
  - default selections pre-filled when possible
  - supports “skip” at every step
- Templates:
  - 3–5 templates max
  - safe-by-default

**Acceptance criteria**
- For treated eligible rides, pin edits after match decrease.
- No measurable decrease in checkout conversion for non-eligible rides (zero exposure).

**Edge cases**
- Rider changes pickup after match: re-run eligibility and refresh rendezvous string
- Accessibility: voice-over friendly; avoid map-only instructions

### 13.3 Module B — Driver cues
**Requirements**
- Cue is informative, not alarming.
- Suggested templates:
  - localized
  - include rider-selected entrance string when available
  - one-tap to send; optional edit
- “Best stop”:
  - only when confidence high; otherwise hide entirely

**Acceptance criteria**
- Driver cancels due to “can’t find rider / hard pickup” reason codes decrease.
- No more than one additional tap target on arrival screen.

### 13.4 Module C — Rendezvous cues
**Requirements**
- Rider card must show:
  - car + plate + “where to meet” string
  - live position cue (bounded; no unnecessary precision)
- Driver card must show:
  - where rider is expected
  - timer + no-show guidance
- Confirmations:
  - rider: “I’m coming out now”
  - driver: “I’m at pickup spot”

**Acceptance criteria**
- Arrive → start time (p90) decreases.
- Pickup-related support contacts decrease.

---

## 14) Instrumentation (events + properties)
### Core events
- `pickup_difficulty_scored`
  - props: `difficulty_score`, `difficulty_bucket`, `reason_codes[]`, `poi_type`, `geo_id`, `model_version`, `latency_ms`
- `guided_pickup_shown`
  - props: `step_count`, `preselected_fields[]`, `eligibility_reason`
- `guided_pickup_completed`
  - props: `selected_entrance`, `landmark_used`, `note_template_id`
- `guided_pickup_skipped`
  - props: `skip_point`
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

## 15) Experiment plan (v3)
### Phase 0 — Offline validation (1–2 weeks)
- Backtest scoring against proxies:
  - pin edits after match
  - call/chat during pickup window
  - arrive→start p90
  - cancel reason codes

### Phase 1 — Limited geo A/B (2–4 weeks)
- Roll out to 1–2 cities + targeted POIs
- A/B: eligible rides only
- Success = reduction in post‑match cancels + arrive→start p90 without harming acceptance/conversion

### Phase 1b — Ramp plan + stop criteria
**Ramp:** 1% → 5% → 25% → 50% of eligible rides (per geo)

**Stop criteria (any trigger pauses ramp):**
- checkout conversion drop beyond threshold
- driver acceptance drop beyond threshold
- support contact rate spike (pickup-related)
- safety incident uptick in treated cohort

### Phase 2 — Expand + harden (4–8 weeks)
- Add POI templates (airports/events as separate tracks)
- Tune thresholds to minimize unnecessary prompts

---

## 16) Ops / rollout checklist (v3)
Before launching beyond a single geo:
- [ ] Eligibility rate sanity check (target: low single-digit % of rides initially)
- [ ] Latency p95 within budget
- [ ] Localized strings for top languages
- [ ] Support macros updated + agent training snippet
- [ ] Driver comms copy approved (avoid punitive framing)
- [ ] Dashboards live (primary + guardrails + segmentation)

---

## 17) Reason-code dictionary (for explainability)
These reason codes must map to user-facing explanations.

- `MULTI_ENTRANCE_POI` → “This location has multiple entrances.”
- `NO_STOPPING_RISK` → “Stopping is limited near this pickup.”
- `HISTORICAL_HIGH_FRICTION` → “Pickups here often take longer.”
- `GPS_DRIFT_ZONE` → “GPS can be inaccurate here.”

---

## 18) Risks & mitigations
1. **Added friction hurts conversion** → strict eligibility + skip + latency budgets.
2. **Prediction errors annoy users/drivers** → explainability + conservative triggers.
3. **Safety concerns** → safety copy + avoid unsafe/illegal suggestions.
4. **Driver compliance** → suggestions not mandates; measure adherence.

---

## 19) Open questions
- What pickup reason codes exist today for cancels, and how reliable are they?
- What’s the cleanest shared storage for rider’s entrance selection across apps?
- Where should guided pickup live in booking UX to minimize disruption?
- What thresholds define “acceptable” conversion/acceptance movement to continue ramp?

---

## Source
Derived from the teardown: `product-teardowns/uber-ridehailing-teardown.md` (pickup loop, cancellation taxonomy, and “Pickup Quality Pack” bet).
