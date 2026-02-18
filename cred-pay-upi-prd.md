# PRD: Offline Trust Pack for CRED Pay (UPI)

**Product:** CRED Pay (UPI)  
**Author:** Mayank Malviya  
**Date:** 18 Feb 2026  
**Status:** v1 — Problem & scope definition  
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/cred-pay-upi-teardown.md

---

## Version history
- **v1 (this doc):** Establishes the core problem framing for offline scan & pay trust gaps, articulates target users, goals, proposed solution pillars, high-level requirements, metrics, and rollout guardrails.
- **v2 (planned):** Flesh out UX flows, instrumentation specs, and experiment design for each pillar; add detailed copy variants and failure-state mocks.
- **v3 (planned):** Finalize implementation-ready specs (API contracts, acceptance tests, launch plan with precise thresholds, dependency tracker).

---

## 1. Problem statement & context
CRED Pay competes in a **commodity UPI landscape** where reliability and trust are differentiators, not table stakes. Offline **scan & pay** is the highest-frequency surface but also the most anxiety-inducing: flaky networks, PSP downtime, QR misuse, and delayed confirmations create uncertainty.

Observable pain today:
1. **Ambiguous pending states** — When PSP/bank callbacks lag, the UI oscillates between “processing” and “success” without clear guidance, prompting duplicate payments.
2. **Weak merchant verification cues** — Many QRs are indistinguishable; users cannot confirm if they paid the right merchant/person, undermining confidence.
3. **Unstructured receipts** — Receipts bury the UTR, timestamp, and funding source, making proof-of-payment sharing/support painful.
4. **Unsafe retry behavior** — Users are nudged to “try again” without knowing whether the first attempt will eventually settle, increasing double-debit risk and support tickets.

Consequence: **support contacts per 1k payments spikes on offline intents**, repeat rate drops, and rewards cannot compensate for lack of trust. To win repeat usage, CRED must make “scan & pay” **trustworthy by default**, even when rails misbehave.

---

## 2. Goals & non-goals
### Goals (for V1 delivery window)
1. **Reduce panic retries** for offline payments by 30% (measured via `pay_retry_clicked` within 3 minutes of `pay_auth_started`).
2. **Increase receipt views** that contain complete merchant + UTR info to 80%+ of successful offline payments.
3. **Cut support tickets per 1k offline payments** related to “status unclear / duplicate debit” by 25%.
4. **Improve 7-day repeat scan rate** for offline-heavy users by 10% (baseline from teardown instrumentation plan).

### Guardrails / non-goals
- Do not slow down the happy path (success state must still confirm under 1.5s p95 when rails respond normally).
- No automatic “money reversal” promises without PSP confirmation.
- Do not alter rewards economics in this release (messaging only references existing reward logic).
- Risk controls (AML, fraud rules) remain unchanged; this PRD only improves transparency and retries around them.

---

## 3. Target users & JTBD
**Primary segment:** Offline high-frequency payers who already trust CRED for credit card payments but rely on UPI for everyday spends (cafés, fuel, pharmacies). They expect a premium, low-drama experience and are sensitive to ambiguous payment states.

**Secondary segments:**
- **Deal/reward optimizers:** Use CRED Pay for “safe” transactions when rewards justify context switch; churn quickly if a payment feels risky.
- **Merchant-trusting repeaters:** Small merchants who rely on showing proof-of-payment on the customer’s phone; they need receipts they can screenshot/share instantly.

**JTBD:** “When I scan and pay offline, reassure me instantly that the right person got the money (or guide me clearly if not), so I don’t double pay or waste time chasing support.”

---

## 4. Key insights & assumptions
- **Trust > novelty:** The teardown highlights that differentiation must come from habit and trust loops, not raw PSP features. Offline reliability is the most frequent loop.
- **Rail volatility is inevitable:** Instead of hiding it, guiding users through pending → success/failure reduces support load.
- **Merchant identity is observable:** CRED already captures merchant/vpa info post-scan; surfacing it prominently helps.
- **Receipts are leverage:** A polished receipt with share/export + “repeat payment” entry point feeds retention loops.
- **Instrumentation gap:** Existing events need offline-specific context (already defined in teardown §11); this PRD leans on those events for measurement.

---

## 5. Solution overview — “Offline Trust Pack”
Bundle of UX + system improvements aimed at **making state explicit** and **preventing duplicate actions**:
1. **State clarity cards** — Deterministic UI states for *Processing*, *Pending (bank delay)*, *Success*, *Failure*, with prescribed CTAs per state.
2. **Receipt-first confirmation** — Immediately show merchant identity, amount, funding source, UTR, and share options; receipt becomes the hero surface.
3. **Safe retry guardrails** — Intelligent detection of risky retries (same amount, same merchant, within decay window) with warnings and guided options.
4. **Merchant identity cues** — Pull last-paid snippet, merchant badges, and QR origin hints into both scan and confirmation stages.
5. **Support + instrumentation hooks** — Lightweight “Need help?” entry anchored to state + auto-attached context, plus emitted events for each state transition.

Scope for V1 is **offline scan & pay** (static and dynamic QR). Online intent/deeplink flows are excluded until V2.

---

## 6. Detailed requirements
### 6.1 State clarity cards
- Define a finite state machine: `Processing` ( <3s ), `Pending` (callback delayed, funds debited unknown), `Success`, `Failed`, `Reversed`.
- Each state maps to copy, iconography, CTAs:
  - *Processing:* spinner + “Hold on, confirming with your bank.” No action buttons.
  - *Pending:* countdown chip showing elapsed time, CTA: “I’ll wait” (dismiss) + “Check status” deep link to payment history; show reminder that bank may take up to X minutes.
  - *Success:* green confirmation, prominent merchant info, share receipt.
  - *Failed:* explain failure reason (from PSP code mapping), CTAs: “Try again with guidance” (pre-populate same merchant) + “Contact support”.
  - *Reversed:* show reversal timestamp and funding source.
- Requirements: state transitions must be idempotent, and the UI should not regress (e.g., no flicker back to processing once pending).

### 6.2 Receipt-first confirmation
- On success/pending, slide up a receipt module with:
  - Merchant name + verification badge + last 4 digits of VPA/UPI ID.
  - Amount + instrument used (bank, last 4 digits).
  - Timestamp + UTR + transaction reference.
  - Buttons: **Share receipt**, **Copy UTR**, **Repeat payment**.
- Receipts must be cached for offline access (if user loses network immediately after paying).

### 6.3 Safe retry guardrails
- Detect duplicate attempts: same merchant/vpa + amount within 5 minutes of pending state.
- If user taps “Try again,” show modal: “Your last payment is still pending. Retry will debit again if the bank confirms the first payment. Prefer waiting X mins or switch account.”
- Provide options:
  1. “Wait & get notified” (subscribe to status push, clear CTA).
  2. “Retry with different account” (preselect alternate bank if configured).
  3. “Continue anyway” (requires long-press confirmation + risk disclaimer).
- Log decision via `pay_retry_clicked(reason_shown, resolution)` event.

### 6.4 Merchant identity cues
- On scan, show merchant preview card: logo/initials, verified ribbon, last-paid label (“Paid here 3d ago”).
- In pending/success states, anchor same card; include QR source metadata (static vs dynamic) for education.
- Requirements: fall back gracefully when metadata missing (show shortened VPA + “New merchant”).

### 6.5 Support & instrumentation hooks
- Persistent “Need help?” link opens contextual sheet auto-filling attempt ID, timestamps, PSP error, network status screenshot prompt.
- Extend instrumentation from teardown §11:
  - Emit `pay_state_changed` with new `state_detail` field (processing/pending/success/failure/reversed).
  - Emit `receipt_shared(channel)` whenever share/copy occurs.
  - Emit `support_flow_opened(category="offline_pending")` from contextual link.
- Support system receives structured payload for faster resolution.

---

## 7. User stories & acceptance criteria
| User story | Acceptance criteria |
| --- | --- |
| As a payer, I want to know if my scan payment is still processing so I don’t double pay. | Processing state shows automatically for ≤3s; if no confirmation, UI transitions to Pending with timer + guidance; duplicate taps blocked during processing. |
| As a payer, I need a trustworthy receipt I can show the merchant instantly. | Success state auto-opens receipt card with merchant info, amount, timestamp, UTR; share/copy actions complete <500ms even on low bandwidth; receipt cached in history. |
| As a cautious user, I want the app to warn me before retrying a pending payment. | Tapping retry when last attempt pending shows guardrail modal with 3 options; continuing requires explicit confirmation. |
| As a support agent, I need structured context when a user reports an unclear status. | “Need help” sends attempt ID, state, PSP code, timestamps, device/network info to support tooling; ticket creation SLA <5s. |

---

## 8. Metrics & instrumentation plan
| Metric | Definition | Source |
| --- | --- | --- |
| Panic retry rate | % of offline payments where `pay_retry_clicked` occurs within 3 minutes of `pay_auth_started` while last state = Pending | Event stream |
| Receipt completeness rate | % of success states where receipt card displayed all five required fields (merchant, amount, funding source, timestamp, UTR) | Client logs + validation ping |
| Support tickets per 1k offline payments | Count of support tickets tagged `offline_pending` / (offline payments * 1000) | Support tooling |
| 7-day repeat scan rate | % of users who perform ≥2 offline payments within 7 days of first attempt | Payments DB |
| Pending dwell time | Median time spent in Pending before resolution (Success or Failure) | Event stream |

Instrumentation tasks:
- Update analytics schema to include `state_detail`, `pending_elapsed_ms`, `retry_decision`.
- Add push notification event `pending_status_update(sent)` for the “Wait & get notified” option.

---

## 9. Experiment & rollout plan
1. **Internal dogfood (week 1):** enable for CRED staff accounts with feature flag; monitor state transitions + guardrails.
2. **City pilot (week 2-3):** roll out to 10% of Bangalore offline payments; compare panic retry + support metrics vs control.
3. **Step-up (week 4):** expand to top 5 offline cities if guardrails green (<5% increase in time-to-success, no spike in fraud flags).
4. **Nationwide (week 5):** default on; retain kill switch tied to PSP outage detection.

Dependencies: state machine service changes, receipt caching, instrumentation updates, support tooling ingestion, push notification templates.

---

## 10. Open questions
1. What is the acceptable SLA for “Wait & get notified” pushes before users lose trust (5 min vs 10 min)?
2. Can we enrich merchant identity with verified badges across PSPs without manual ops?
3. Should we surface reward state within receipt to reinforce habit, or keep trust UI minimal?
4. How do we message when PSP returns ambiguous “status unknown” codes (fallback copy variants needed)?
5. Are there regulatory constraints on storing UTR locally for offline access that we must account for?

---

## 11. Next steps toward V2
- Partner with design to produce state/receipt mocks and copy variants.
- Define API contract for state machine service (`/payments/{id}/state`).
- Align with support tooling team on payload schema and SLA for contextual tickets.
- Draft experiment analysis plan (segmentation by merchant type, device quality, default bank).
