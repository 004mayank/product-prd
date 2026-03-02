<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/Cred.png" 
    alt="Cred" 
    width="200"
  />
</p>

# PRD: Offline Trust Pack for CRED Pay (UPI)

**Product:** CRED Pay (UPI)  
**Author:** Mayank Malviya  
**Date:** 19 Feb 2026  
**Status:** Final - API contracts + acceptance tests + launch gates + dependencies + runbooks

**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/cred-pay-upi-teardown.md
---

## 1. Problem statement & context
CRED Pay competes in a **commodity UPI landscape** where reliability and trust are differentiators, not table stakes. Offline **scan & pay** is the highest-frequency surface but also the most anxiety-inducing: flaky networks, PSP downtime, QR misuse, and delayed confirmations create uncertainty.

Observable pain today:
1. **Ambiguous pending states** - When PSP/bank callbacks lag, the UI oscillates between “processing” and “success” without clear guidance, prompting duplicate payments.
2. **Weak merchant identity cues** - Many QRs are indistinguishable; users can’t confirm they paid the right merchant/person, undermining confidence.
3. **Unstructured receipts** - Receipts bury the UTR, timestamp, and funding source, making proof-of-payment sharing/support painful.
4. **Unsafe retry behavior** - Users are nudged to “try again” without knowing whether the first attempt will eventually settle, increasing double-debit risk and support tickets.

Consequence: **support contacts per 1k payments spikes on offline intents**, repeat rate drops, and rewards can’t compensate for lack of trust. To win repeat usage, CRED must make “scan & pay” **trustworthy by default**, even when rails misbehave.

---

## 2. Goals & non-goals
### Goals (measurable)
1. **Reduce panic retries** for offline payments by **30%** (measured via `pay_retry_clicked` within 3 minutes of `pay_auth_started`).
2. **Increase “complete receipt” exposure** to **80%+** of successful offline payments.
3. **Cut support contacts per 1k offline payments** related to “status unclear / duplicate debit” by **25%**.
4. **Improve 7-day repeat scan rate** for offline-heavy users by **10%**.

### Guardrails / non-goals
- Do not slow down happy path (success confirm **p95 ≤ 1.5s** when rails respond normally).
- No promises about reversals without PSP confirmation.
- Don’t change rewards economics in this release (trust layer only).
- Risk controls (AML/fraud rules) remain unchanged; this PRD improves transparency and retries around them.

---

## 3. Target users & JTBD
**Primary segment:** Offline high-frequency payers who already trust CRED for credit card payments but rely on UPI for everyday spends (cafés, fuel, pharmacies). They expect a premium, low-drama experience and are sensitive to ambiguous payment states.

**Secondary segments:**
- **Deal/reward optimizers:** Context-switch only if it feels safe.
- **Merchant-trusting repeaters:** Need proof-of-payment they can show/share instantly.

**JTBD:** “When I scan and pay offline, reassure me instantly that the right person got the money (or guide me clearly if not), so I don’t double pay or waste time chasing support.”

---

## 4. Solution overview - “Offline Trust Pack”
Bundle of UX + system improvements aimed at **making state explicit** and **preventing duplicate actions**.

**Pillars**
1. **State clarity** - Deterministic UI states for *Processing*, *Pending*, *Success*, *Failure*, *Reversed*, with prescribed CTAs.
2. **Receipt-first confirmation** - Merchant identity + amount + funding source + UTR + timestamp as the hero surface, with share/copy.
3. **Safe retry guardrails** - Detect risky retries (same merchant + amount + short window) and guide user.
4. **Merchant identity cues** - Preview card on scan + persistence through outcome states.
5. **Support hooks** - Contextual support entry that auto-attaches structured diagnostic payload.

**In-scope:** Offline scan & pay (static + dynamic QR), including pending/unknown resolution UX.

**Out-of-scope (for now):** Online intent/deeplink flows, collect requests, rewards changes, risk rules.

---

## 5. Product principles
1. **Success must be boring and definitive.** Never trade certainty for delight.
2. **Pending is a first-class state** with guidance; users must not be pushed into blind retries.
3. **Merchant identity must be legible** before PIN (when possible) and after confirmation.
4. **Receipts are the “trust artifact”.** Sharing proof must be instant.
5. **Measure everything.** If it’s not instrumented, it doesn’t exist.

---

## 6. UX flows (primary + edge cases)
Notation: `STATE / screen → action → next screen`.

### 6.1 Offline scan & pay - happy path (fast rails)
1. **Home** → tap **Scan**
2. **Scanner** → QR lock → **Merchant preview**
   - Shows: merchant name (primary), VPA (secondary), “Paid here before” (if known), static/dynamic badge.
3. **Amount**
   - If **dynamic QR**: amount prefilled + locked.
   - If **static QR**: keypad with gentle anti-extra-zero guardrail.
4. **Confirm**
   - Shows: merchant + amount + funding source selector.
   - Primary CTA: **Pay**.
5. **UPI PIN**
6. **Processing (<3s)**
7. **Success** → auto-open **Receipt card**
   - Primary actions: **Share receipt**, **Copy UTR**.
   - Secondary: **Repeat payment**.

**Exit criteria:** user sees a definitive final state + shareable receipt within **≤3 taps after PIN**.

### 6.2 Slow rails: Processing → Pending → Success
Trigger: no final confirmation within threshold.

1. **Processing** shown for up to **3s**.
2. Transition to **Pending**:
   - Copy: “Payment is processing with your bank.”
   - Guidance: “Don’t pay again yet.”
   - Actions:
     - Primary: **Check status** (opens payment details in History)
     - Secondary: **Wait & get notified** (subscribe)
     - Tertiary: **Retry** (guardrailed; see 6.4)
3. When rails resolve:
   - Pending → **Success** OR Pending → **Failed** OR Pending → **Reversed**.

**Pending SLA copy:**
- Show a resolution expectation (e.g., “Usually resolves in ~2 minutes; can take up to 10 minutes.”) - set using PSP + historical data.

### 6.3 Merchant identity uncertainty (new merchant)
If merchant metadata is weak:
- Scan preview shows: **shortened VPA** + “New merchant”
- Confirm screen includes: “Double-check merchant name before paying.”
- Receipt shows: merchant display name (if any) + VPA + UTR; still shareable.

### 6.4 Retry flow (guardrailed)
Condition to guardrail:
- same `merchant_id/vpa_hash` AND same `amount_bucket` AND last state is `pending` AND within **5 minutes**.

When user taps **Retry**:
- Modal title: “Last payment is still pending.”
- Body: “Retrying may debit you again if the bank later confirms the first payment.”
- Options:
  1. **Wait & get notified** (recommended)
  2. **Retry with different account** (opens funding source picker)
  3. **Continue anyway** (requires long-press + explicit disclaimer)

### 6.5 “Paid but merchant says not received” (support-first)
If user opens payment details from Pending/Success and selects “Merchant didn’t receive”:
- Show the receipt card (UTR front-and-center) + “Share receipt with merchant”
- Offer “Contact support” with payload auto-attached.

### 6.6 Offline after payment (network drop)
If network drops after PIN and before final confirmation:
- Show **Pending** with explicit offline copy: “You’re offline. We’ll confirm when you’re back online.”
- Still allow **History** access to the attempt.

---

## 7. State model & UX rules (implementation-directed)
### 7.1 Canonical user-facing states
| State | When used | Allowed CTAs | Never do |
| --- | --- | --- | --- |
| `processing` | post-PIN, awaiting PSP/bank | none | show retry button |
| `pending` | confirmation delayed/ambiguous | check status, notify, guardrailed retry, support | show “Success” |
| `success` | final confirmed | share, copy UTR, repeat | hide UTR |
| `failed` | final failure | guided retry, support | generic “try again” without reason |
| `reversed` | reversal confirmed | view receipt, support | promise instant reversal |

### 7.2 Transition rules
- UI must be **monotonic**: `processing → pending → (success|failed|reversed)`; no regressions.
- State changes must be **idempotent** (replays should not create flicker).
- If state is unknown beyond SLA, keep user in `pending` and encourage **Check status**, not retry.

### 7.3 Failure taxonomy (user-safe)
Map PSP/bank codes into a small set for copy + analytics:
- `bank_unavailable`
- `timeout`
- `insufficient_funds`
- `pin_failed`
- `user_cancelled`
- `limit_exceeded`
- `merchant_invalid`
- `unknown`

Copy rules:
- Never blame user by default.
- For `timeout/unknown`, prefer “We couldn’t confirm yet” + pending guidance.

---

## 8. Receipt spec (fields + behaviors)
### 8.1 Required fields
Receipt is considered **complete** only if it includes:
- Merchant display name (or fallback VPA)
- Amount
- Funding source (bank/account)
- Timestamp
- UTR / reference id

### 8.2 Actions
- **Share receipt**: generates a share card with the 5 required fields.
- **Copy UTR**: one-tap copy with confirmation toast.
- **Repeat payment**: deep link back to confirm screen with merchant prefilled (amount optional).

### 8.3 Caching
- Cache receipt locally after outcome to support “network drop” scenarios.
- On open, validate that the cached receipt matches server-side attempt id.

---

## 9. Instrumentation spec (events + properties)
### 9.1 Identity and joins
All events must include:
- `attempt_id` (UUID) 
- `user_id` (hashed)
- `timestamp_ms`
- `mode` = `offline_scan`
- `qr_type` = `static|dynamic|unknown`
- `merchant_id` (if available) OR `vpa_hash`
- `amount_bucket` (e.g., `<100`, `100-500`, `500-2000`, `>2000`)
- `bank_id` (selected funding source)
- `network_type` (wifi/4g/5g/none)
- `device_tier` (low/med/high; derived)

### 9.2 Funnel events
| Event | When | Key properties |
| --- | --- | --- |
| `scan_opened` | scanner screen visible | `entry_point` (home/shortcut/other) |
| `scan_qr_detected` | QR lock | `qr_type`, `has_amount_prefilled` |
| `merchant_identity_shown` | preview card rendered | `has_verified_badge`, `has_last_paid`, `identity_confidence` (high/med/low) |
| `amount_entered` | amount confirmed | `amount_bucket`, `prefilled` |
| `funding_source_selected` | user selects bank/account | `is_default`, `switch_reason` (optional) |
| `pay_attempt_created` | server attempt created | `attempt_source` |
| `pay_auth_started` | PIN screen shown |  |
| `pay_auth_completed` | PIN result | `status` (ok/fail/cancel) |
| `pay_state_changed` | state transitions | `from_state`, `to_state`, `failure_taxonomy` (if any), `pending_elapsed_ms` |
| `pay_result` | terminal | `status` (success/failed/reversed), `failure_taxonomy`, `time_to_terminal_ms` |

### 9.3 Pending + retry instrumentation
| Event | When | Key properties |
| --- | --- | --- |
| `pay_pending_shown` | UI enters pending | `time_ms_since_auth`, `pending_reason` (timeout/unknown/offline) |
| `pay_status_checked` | user taps check status | `source` (pending_cta/history) |
| `pay_retry_clicked` | user taps retry | `reason_shown` (pending_guardrail/fail_retry), `within_guardrail_window` (bool) |
| `pay_retry_decision` | guardrail modal choice | `decision` (wait_notify/retry_different/continue_anyway/cancel) |
| `pay_duplicate_attempt_blocked` | prevented retry | `block_reason` |

### 9.4 Receipt instrumentation
| Event | When | Key properties |
| --- | --- | --- |
| `receipt_viewed` | receipt card shown | `receipt_complete` (bool) |
| `receipt_shared` | share action completes | `channel` (whatsapp/other), `receipt_complete` |
| `utr_copied` | copy action |  |

### 9.5 Support instrumentation
| Event | When | Key properties |
| --- | --- | --- |
| `support_flow_opened` | contextual help opened | `category` (offline_pending/offline_failed/merchant_dispute) |
| `support_ticket_created` | ticket created | `category`, `auto_payload_attached` (bool) |

### 9.6 Analytics validation checklist (must-have)
- **Joinability:** 99%+ of events must carry `attempt_id`.
- **Receipt completeness:** `receipt_viewed.receipt_complete` computed on-device and also validated server-side for a sample.
- **State monotonicity:** detect any state regressions client-side; emit `state_regression_detected` if found.

---

## 10. Experiment design (per pillar)
### 10.1 Overall experimental setup
- **Unit:** payment attempt (`attempt_id`) and user (for retention).
- **Primary analysis:** offline-heavy cohorts (≥3 offline scans in last 14 days).
- **Holdout required:** permanent 1–5% user holdout to prevent false positives from seasonality.

### 10.2 Experiment A - Pending UX + guidance (state clarity)
**Hypothesis:** Explicit pending guidance reduces panic retries and support contacts without hurting conversion.

- **Treatment:** new Pending screen + Check status + Wait & notify.
- **Control:** current “processing/try again” behavior.
- **Primary metrics:**
  - Panic retry rate (as defined in Goals)
  - Support contacts per 1k offline payments (pending-related)
- **Guardrails:**
  - Attempt→terminal success rate (no worse than -0.2pp)
  - p95 time-to-terminal (no worse than +250ms when rails are fast)
- **Go/no-go:** ship if panic retries drop ≥15% in pilot with guardrails green.

### 10.3 Experiment B - Retry guardrail modal
**Hypothesis:** Guardrailed retry reduces duplicate debits and improves perceived trust.

- **Treatment:** guardrail modal with decision capture.
- **Primary metrics:**
  - Duplicate attempts per 1k attempts (same merchant+amount within window)
  - Support contacts tagged “duplicate debit”
- **Guardrails:**
  - % users who exit app from Pending (no worse than +1pp)
- **Go/no-go:** adopt if duplicate attempts drop ≥20%.

### 10.4 Experiment C - Receipt-first success surface
**Hypothesis:** Making receipt the hero reduces disputes and increases repeat usage.

- **Treatment:** auto-open receipt card with Share + Copy UTR; merchant identity persistent.
- **Primary metrics:**
  - Receipt share/copy rate
  - Support contacts tagged “merchant dispute / didn’t receive”
- **Secondary:** 7-day repeat scan rate.
- **Guardrail:** median time-to-dismiss success screen (avoid slowing flow): +≤300ms.

### 10.5 Experiment D - Merchant identity cues on scan preview
**Hypothesis:** Identity cues reduce wrong-merchant anxiety and improve completion.

- **Treatment:** merchant preview card with “Paid here before” + confidence label.
- **Primary metrics:**
  - Scan→attempt created conversion
  - “Back/cancel” rate on confirm screen
- **Guardrail:** time-to-first-frame scanner (no worse than +100ms).

---

## 11. Rollout plan
1. **Dogfood (week 1):** feature flag for staff; validate state monotonicity + receipt completeness.
2. **City pilot (week 2–3):** 10% Bangalore offline payments; run Experiments A/B concurrently if interference minimal.
3. **Step-up (week 4):** expand to top 5 offline cities if:
   - panic retries ↓ and support contacts ↓
   - no success rate regression
   - no fraud/risk alerts spike
4. **Nationwide (week 5):** default on + keep kill switch tied to PSP outage detection.

Dependencies: state machine service changes, receipt caching, instrumentation updates, support tooling ingestion, push notification templates.

---

## 12. API contracts:
> Purpose: make the implementation unambiguous across client, payments backend, and support tooling.

### 12.1 Entities
**PaymentAttempt** (server canonical)
- `attempt_id` (uuid)
- `user_id` (hash)
- `mode` = `offline_scan`
- `qr_type` = `static|dynamic|unknown`
- `merchant`:
  - `merchant_id` (optional)
  - `display_name` (optional)
  - `vpa` (string)
  - `identity_confidence` = `high|med|low`
- `amount_paise`
- `currency` = `INR`
- `funding_source`:
  - `bank_id`
  - `masked_account`
- `state` = `created|auth_started|processing|pending|success|failed|reversed`
- `failure_taxonomy` (optional; only in terminal or when known)
- `utr` (optional; required for `success`, may be present in some `pending`)
- `created_at_ms`
- `updated_at_ms`

### 12.2 Create attempt
**POST** `/v1/payments/upi/attempts`

Request
```json
{
  "mode": "offline_scan",
  "qr_payload": "<raw-qr-string>",
  "qr_type": "static",
  "amount_paise": 12500,
  "merchant_hint": {
    "merchant_id": null,
    "vpa": "merchant@upi",
    "display_name": null
  },
  "funding_source": { "bank_id": "hdfc_001" },
  "client_context": {
    "network_type": "4g",
    "device_tier": "med",
    "app_version": "x.y.z"
  }
}
```

Response (201)
```json
{
  "attempt_id": "uuid",
  "state": "created",
  "merchant": {
    "merchant_id": null,
    "display_name": "ABC Stores",
    "vpa": "merchant@upi",
    "identity_confidence": "med"
  },
  "amount_paise": 12500,
  "currency": "INR"
}
```

### 12.3 Fetch attempt (status / details)
**GET** `/v1/payments/upi/attempts/{attempt_id}`

Response (200)
```json
{
  "attempt_id": "uuid",
  "state": "pending",
  "failure_taxonomy": null,
  "utr": null,
  "merchant": { "display_name": "ABC Stores", "vpa": "merchant@upi", "identity_confidence": "med" },
  "amount_paise": 12500,
  "funding_source": { "bank_id": "hdfc_001", "masked_account": "xx1234" },
  "timestamps": { "created_at_ms": 0, "updated_at_ms": 0 }
}
```

**Contract rules**
- `state` is monotonic by server contract (no regressions).
- `utr` must be non-null for `success`.
- `failure_taxonomy` must be non-null for `failed` (use `unknown` if no better mapping).

### 12.4 Subscribe to pending updates (push)
**POST** `/v1/payments/upi/attempts/{attempt_id}/subscriptions`

Request
```json
{ "channel": "push", "quiet_hours_respect": true }
```

Response (200)
```json
{ "subscribed": true }
```

Notes
- Server triggers push when attempt enters `success|failed|reversed`.
- Rate limit: max 1 subscription per `attempt_id` per device.

### 12.5 Retry guardrail evaluation
**POST** `/v1/payments/upi/retry/eligibility`

Request
```json
{
  "user_id": "hash",
  "merchant_key": { "merchant_id": null, "vpa_hash": "..." },
  "amount_paise": 12500,
  "last_attempt_id": "uuid",
  "last_state": "pending",
  "last_attempt_age_ms": 120000
}
```

Response (200)
```json
{
  "should_guardrail": true,
  "window_ms": 300000,
  "reasons": ["pending_within_window"],
  "allowed_actions": ["wait_notify", "retry_different_account", "continue_anyway"],
  "block": false
}
```

### 12.6 Support payload handoff
**POST** `/v1/support/tickets`

Request (excerpt)
```json
{
  "category": "offline_pending",
  "attempt_id": "uuid",
  "auto_payload": {
    "state": "pending",
    "merchant": { "display_name": "ABC Stores", "vpa": "merchant@upi" },
    "amount_paise": 12500,
    "funding_source": { "bank_id": "hdfc_001" },
    "network_type": "4g",
    "client_events_sample": ["pay_pending_shown", "pay_status_checked"]
  }
}
```

---

## 13. Acceptance tests
> “Done” means these pass in staging + dogfood builds.

### 13.1 State monotonicity
1. **Processing→Pending→Success monotonicity**
   - Given an attempt with delayed confirmation
   - When the client polls / receives update
   - Then UI must never show Success before server state becomes `success`
   - And must never return from `pending` to `processing`

2. **Idempotent updates (no flicker)**
   - Given repeated identical `GET attempt` responses
   - Then UI should not re-render state banner repeatedly (no toast spam)

### 13.2 Pending UX rules
3. **No retry button in `processing`**
4. **Pending CTAs appear**: Check status, Wait & notify, Retry (guardrailed)
5. **Pending offline copy** when `network_type=none` post-auth

### 13.3 Retry guardrail
6. **Guardrail triggers in-window** (same merchant + amount_bucket + pending + ≤5m)
7. **Guardrail does not trigger** when:
   - amount differs bucket OR
   - merchant differs OR
   - last state is terminal OR
   - age > 5m
8. **Continue anyway requires long-press** (accessibility: also allow “press & hold” with haptics; ensure screen reader hint)

### 13.4 Receipt completeness and actions
9. **Receipt_complete true** for success attempts with all 5 fields
10. **Share card includes**: merchant, amount, funding source, timestamp, UTR
11. **Copy UTR** copies only the UTR string (no extra whitespace)
12. **Receipt caching**
   - Given success then network drop
   - When user opens History
   - Then receipt still renders from cache and later reconciles with server

### 13.5 Instrumentation
13. **attempt_id present** on 99%+ of client events (verified by automated test that asserts event payloads)
14. **pay_state_changed emits** on every state transition with correct `from_state/to_state`
15. **state_regression_detected** is emitted if server returns a regressive state (this should be 0 in practice; used for detection)

### 13.6 Support flow
16. **Support ticket attaches payload** and includes `attempt_id`
17. **Merchant dispute path** shows receipt first, then support

---

## 14. Launch gates (v3)
### 14.1 Gating metrics & thresholds (must all be green)
Pilot scope: city pilot (10%), offline-heavy cohort primary.

| Category | Metric | Threshold | Window |
| --- | --- | --- | --- |
| Reliability | `pay_result.success_rate` | ≥ baseline - 0.2pp | rolling 24h |
| Latency | p95 time-to-terminal (fast rails cohort*) | ≤ baseline + 250ms | rolling 24h |
| Trust | Panic retry rate | ≥ 15% reduction | 7 days |
| Support | Support contacts per 1k (pending/duplicate categories) | ≥ 10% reduction | 7 days |
| Integrity | Duplicate attempts per 1k (same merchant+amount within 5m) | ≥ 20% reduction | 7 days |
| Data | Joinability (`attempt_id` coverage) | ≥ 99% | daily |
| UX | Receipt completeness exposure | ≥ 80% of success | daily |

\*Fast rails cohort definition (for gating): `time_to_terminal_ms <= 1500` in control; compare only that slice to avoid punishing rails downtime.

### 14.2 Kill switches
- Flag 1: **Pending UX** (revert to existing)
- Flag 2: **Retry guardrail** (disable modal; fallback to guided retry)
- Flag 3: **Receipt-first success** (disable auto-open receipt)
- Flag 4: **Push subscription** (disable notify)

### 14.3 Escalation triggers (page-oncall)
- Success rate drops >0.5pp vs baseline for 30 minutes
- `state_regression_detected` > 0.05% of attempts
- Duplicate attempts per 1k increases >10% vs baseline

---

## 15. Dependency tracker
| Workstream | Owner | Depends on | Deliverable | Target | Notes |
| --- | --- | --- | --- | --- | --- |
| Client state machine + UI | Mobile | Payments API `GET attempt` monotonicity | Processing/Pending/Receipt screens | W1 | Ensure idempotent rendering |
| Payments attempt service | Backend | PSP mapping + status source of truth | `/attempts` create + get | W1 | Must guarantee monotonic state |
| Push notify templates | Backend/Platform | subscription endpoint | pending resolution pushes | W2 | Quiet hours compliance |
| Retry eligibility | Backend | merchant key normalization | `/retry/eligibility` | W1 | Prefer server decision even if client can pre-check |
| Receipt caching | Mobile | storage encryption policy | local receipt cache | W2 | Clarify UTR storage constraints |
| Instrumentation + dashboards | Data | event schema final | dashboards + alerts | W2 | Gatekeeping metrics |
| Support ingestion | Support Eng | ticket API | auto-payload tickets | W2 | Verify PII policy |

---

## 16. Operational runbooks
### 16.1 Monitoring
Dashboards (must exist before pilot):
- Attempt funnel: scan→auth→processing→pending→terminal
- State distribution over time (spikes in `pending`)
- Time-to-terminal percentiles
- Panic retry + duplicate attempts
- Receipt completeness + share/copy rates
- Support tickets by category (offline_pending, duplicate_debit, merchant_dispute)

### 16.2 Incident: PSP/bank outage (pending spike)
**Symptoms:** pending rate spikes, time-to-terminal inflates, support tickets rise.

**Actions:**
1. Enable kill switch: disable Retry (or enforce stronger guardrail: block continue-anyway) if duplicates rising.
2. Update Pending copy to outage variant (server-driven copy if available) that strongly discourages retry.
3. Increase status poll interval to reduce backend load (e.g., exponential backoff capped at 30s).
4. Notify support team with incident tag; auto-attach PSP status page if available.

### 16.3 Incident: Duplicate debits rise
**Symptoms:** duplicate attempts per 1k increases; support “double debit” tickets spike.

**Actions:**
1. Turn on strict duplicate block for in-window retries (server `block=true`).
2. Ensure “Wait & get notified” CTA is primary and sticky.
3. Audit guardrail eligibility logs for merchant_key / amount_bucket mismatches.

### 16.4 Incident: Receipt missing UTR
**Symptoms:** receipt_complete drops; UTR null on success.

**Actions:**
1. Treat as **SEV** if >0.5% success attempts missing UTR.
2. Roll back receipt-first success if it exposes incomplete receipts; show “Receipt updating…” but keep share disabled.
3. Backend: verify PSP response mapping; ensure UTR is captured before marking `success`.

### 16.5 Rollback plan
- Roll back in this order (least to most disruptive):
  1) Disable receipt auto-open
  2) Disable push subscription
  3) Disable pending UX (revert to old)
  4) Disable guardrail
- Keep instrumentation on (do not roll back analytics unless it breaks core flows).

---

## 17. Open questions
1. Pending SLA copy: 5 vs 10 minutes - choose based on historical settlement.
2. Merchant verification: what’s the “verified” source of truth across PSPs?
3. Should reward feedback appear inside receipt or remain separate?
4. Exact push notification behavior for pending updates (rate limits, quiet hours).
5. Any regulatory constraints on storing UTR locally for offline access.
