<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/zomato-logo.png" 
    alt="Zomato logo" 
    width="200"
  />
</p>

# PRD: Reduce Checkout Abandonment in Zomato Ordering Flow

**Product:** Zomato - Food Delivery  
**Author:** Mayank Malviya  
**Status:** v2 - Requirements + instrumentation ready for build partner review

---

## 1. Problem Statement & Opportunity Size
Cart → order conversion on the primary ordering funnel lags the food-delivery benchmark by ~4–6 p.p., with **>30% of drop-offs** clustered between "Review Cart" and "Pay" per teardown heuristics + public metrics. Qual/quant signals:
- CSAT verbatims cite **“fees keep changing”** and **“coupon didn’t work”** as top-3 reasons for backing out.
- Users who see a **Δ > ₹40** between menu total and payable amount are **2.1× more likely** to abandon.
- First-time or reactivated users churn 20% more often when an offer fails to apply.

Opportunity: lift cart → order conversion by **1.5–2 p.p.** via earlier expectation setting + automated reassurance, yielding ~+0.5% orders per MAU (~tens of thousands of incremental monthly orders at current scale).

---

## 2. Goals & Non-Goals
### Goals
1. Increase cart → order conversion (primary success metric).
2. Reduce fee/offer-related confusion (measured through event drop-offs and CSAT tags).
3. Preserve or improve contribution margin by nudging clarity, not blanket discounts.

### Non-Goals
- Repricing or waiving platform/delivery fees.
- Redesigning the complete checkout UI outside specified modules.
- Modifying restaurant/partner-side workflows.

---

## 3. Personas & Use Cases
| Persona | Behaviors | JTBD in checkout |
|---------|-----------|------------------|
| **Urban frequent (anchor persona)** | Orders 2–5×/week, price-aware yet convenience-first | "Tell me the real total before I tap Pay so I can commit quickly." |
| **Deal seeker / value hunter** | Cross-checks offers, flips between apps | "Show me the best applicable offer automatically so I don’t do extra math." |
| **First-time / trust-sensitive** | Limited history, higher leakage | "Reassure me that nothing else will change after I confirm." |

Key journeys covered: Mobile app cart review → checkout → payment selection → order confirmation. Desktop, Dine-in, Paytm mini-app excluded.

---

## 4. Insights from Teardown & Data
1. **Price delta shock:** Fees surface only after tapping checkout; cognitive load peaks right before payment.
2. **Offer roulette:** Users must copy/paste codes, leading to failures (ineligible basket, bank-only offers, min order value).
3. **Commitment anxiety:** Lack of affirmations (ETA, no extra fee) and "what if" states (restaurant rejects, price surge) nudge re-evaluation.
4. **Instrumentation gaps:** No unified events for "fee breakdown viewed" vs "offer error"; hard to isolate reason codes today.

---

## 5. Solution Pillars
1. **Early Price Transparency** - display an estimated payable amount (items + fees + taxes) in-cart, before checkout CTA.
2. **Offer Decisioning & Explanation** - auto-apply the best offer, surface "why" copy, and allow override without modal hopping.
3. **Checkout Confidence Nudges** - reassure users post-offer decision with ETA validation, fee lock message, and fallback copy for failure states.

Each pillar ships as a feature flagged module but is designed to work cumulatively.

---

## 6. Detailed Requirements & Acceptance Criteria
### 6.1 Early Price Transparency
**User stories**
- As a frequent user, I want to see an accurate total before checkout so I don’t abandon at payment.
- As a trust-sensitive user, I want a breakdown of charges so I believe Zomato is fair.

**Requirements**
1. **Estimated Total Card** sits above the "Proceed to Pay" CTA in cart:
   - Shows `Items subtotal`, `Platform fee`, `Delivery fee`, `Taxes`, `Total payable`.
   - Uses cached fee service to avoid blocking cart load (>95% responses <150 ms).
2. If any fee is dynamic (surge, busy area), show a badge `Likely to stay within ±₹10` with tooltip.
3. Provide "Why this fee?" inline link → bottom sheet (re-uses existing FAQ content).

**Acceptance criteria**
- Estimate accuracy ≥90% within ±₹5 vs final payable for non-surge orders.
- For surge/dynamic fees, accuracy band displayed; if final total drifts beyond band, log `fee_band_breach` event.
- Card collapses on cart items change to reflect updated totals within 400 ms.

**Instrumentation additions**
| Event | Properties | Purpose |
|-------|------------|---------|
| `cart_fee_breakdown_view` | city_id, user_segment, subtotal, fees_shown | Track exposure |
| `cart_fee_breakdown_toggle` | tapped_section, dwell_ms | Measure engagement |
| `fee_band_breach` | order_id, expected_total, final_total, fee_type | Alert pricing team |

### 6.2 Offer Decisioning & Explanation
**User stories**
- As a deal seeker, I want Zomato to auto-apply the best eligible offer.
- As any user, I want to know why an offer isn’t available without testing codes.

**Requirements**
1. **Auto-apply engine** queries Offer Service with cart context (restaurant, payment method, user tenure) and applies the highest savings offer that meets guardrails (maintain GMV ≥₹X, contribution margin threshold).
2. **Reason chips**: Beneath the applied offer, show plaintext reason (e.g., "Best offer applied: Zomato Treats") with optional `Change` CTA.
3. If no offer applies, show `No offers apply yet` with top disqualification reason (min order not met, incompatible payment method) and highlight the closest actionable next step.
4. Ensure manual override path (user selects a different coupon) with confirmation message `Switching may change savings`.

**Acceptance criteria**
- Auto-application latency ≤200 ms p95.
- Offer success rate +15% relative to control (measured via `offer_apply_success / offer_attempt`).
- Error copy localized (EN + top 4 languages) and never blank.

**Instrumentation**
| Event | Properties |
| `offer_auto_apply_attempt` | offer_id, savings_amount, success_bool, failure_reason |
| `offer_reason_view` | reason_code |
| `offer_manual_override` | from_offer_id, to_offer_id |

### 6.3 Checkout Confidence Nudges
**User stories**
- I want to know nothing else will be added after this step.
- If something does change (restaurant unavailable), tell me immediately with alternatives.

**Requirements**
1. **Fee lock banner** at top of payment screen: `All fees locked. You’ll only pay ₹X.` Displayed once offers resolved.
2. **ETA reassurance**: show `Delivered in ~Y mins (90% reliability)` pulled from fulfillment service.
3. **Edge-case handling**:
   - If restaurant toggles to "busy" between cart and payment, show modal with updated ETA/fees + persistent "Review changes" CTA.
   - Provide one-tap support entry if a payment fails after debit but before confirmation.

**Acceptance criteria**
- Fee lock banner renders only when totals stable (observed via `checkout_totals_stable` event).
- For reprice events mid-checkout, fallback modal must present new total AND `Continue`/`Choose another restaurant` options; default focus on `Continue`.

**Instrumentation**
| Event | Properties |
| `checkout_confidence_banner_view` | total_amount, eta_minutes |
| `mid_checkout_reprice` | delta_amount, reason |
| `support_entry_checkout` | reason_code |

---

## 7. Metrics & Instrumentation Plan
| Tier | Metric | Target | Notes |
|------|--------|--------|-------|
| Primary | Cart → Order Conversion | +1.5 to +2 p.p. lift vs control | Measured daily, segmented by persona proxy |
| Secondary | Offer success rate | +15% relative | `offer_apply_success / offer_attempt` |
| Secondary | Checkout abandonment reason share (fee/offer) | -20% | Coded via CSAT + support tags |
| Guardrail | Contribution margin per order | ≥ baseline | No erosion from over-subsidizing |
| Guardrail | Payment failure rate | ≤ baseline | Ensure added modules don’t slow checkout |

**Event schema** described in §6; ensure events flow to Amplitude + internal experimentation platform. Add derived metric dashboards (Data Studio) with pre/post comparisons.

---

## 8. Experimentation Strategy
1. **Phase 0 (dogfood + design QA)** - internal ramp, focus on instrumentation accuracy.
2. **Phase 1 (10% traffic A/B)** - Treatment = Pillar 1+2, control = current experience.
   - Run until 95% power to detect 1 p.p. lift (~3–4 days at peak traffic).
   - Guardrails: payment error rate, refund contact rate.
3. **Phase 2 (25% traffic)** - Add Pillar 3, monitor user support tags and cancellation rate.
4. **Phase 3 (100% rollout)** - region-wise (top 8 cities first). Maintain kill-switch per pillar (feature flag) with alerting on metric regressions (PagerDuty on conversion drop >1 p.p. for 2 hrs).

Experiment analysis plan: run stratified cuts (new vs repeat, order value buckets) and log-likelihood ratio test on conversion. Document results in PRD appendix before graduating feature.

---

## 9. Rollout & Monitoring
- **Prerequisites:** Pricing/fee API latency SLA met, localization strings signed-off, instrumentation verified.
- **Launch checklist:** QA checklists for each pillar, experiment config reviewed by Data Science, escalation matrix confirmed with Support.
- **Monitoring:** Real-time dashboard with primary + guardrail metrics; automated alert on `fee_band_breach` spikes or drop in `offer_auto_apply_success`.
- **Rollback path:** Toggle off via config service per pillar without deploy; communicate to CRM/Marketing if offer messaging changes.

---

## 10. Risks & Mitigations
| Risk | Type | Mitigation |
|------|------|------------|
| Estimated totals wrong during surge | Trust | Display band + log breaches to pricing team; suppress banner if confidence <80%. |
| Auto-applied offer increases subsidy cost | Business | Apply margin guardrails, monitor contribution per order daily. |
| UI crowding (especially on small screens) | UX | Use collapsible cards, progressive disclosure for fee FAQ. |
| Localization gaps cause mistrust | UX/Trust | Run copy review with language owners; fallback to English + icon if missing. |
| Added service calls slow checkout | Perf | Cache fee + offer responses for session, enforce SLA monitors. |

---

## 11. Dependencies & Open Questions
**Dependencies**
- Pricing/Fee service for accurate estimates + surge confidence bands.
- Offers platform for auto-apply API + reason codes.
- Experimentation platform (Nova) for ramp + guardrails.
- Design + Content for banners, tooltips, and localized copy.
- Data science for persona proxy definitions (frequency, AOV buckets).

**Open questions**
1. Should we expose Gold subscription upsell when no offer applies, or does that distract? (Need CRM input.)
2. Do COD users respond differently to fee lock messaging? (Requires segmentation post-Phase 1.)
3. Should we extend transparency to menu/listing surfaces once cart module proves impact?

---

## 12. Final Takeaway
We are not changing Zomato’s fee model-we are aligning **expectations + confidence** earlier so users don’t abandon when friction spikes. By coupling price clarity, automatic savings, and checkout reassurance with tight instrumentation, we can lift conversion now while unlocking richer, earlier experimentation (e.g., listing-level price previews) once this foundation is stable.
