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
**Version:** v3 - Final PRD
**Changes from v2:** Resolved all open questions with explicit decisions, added full experiment backlog with power calculations, phased rollout plan with launch gates and kill switches, dependency map with owners, SLO revision history, complete instrumentation spec per phase, and version history table.

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, opportunity sizing, goals/non-goals, three personas, three solution pillars (price transparency, offer decisioning, checkout confidence nudges), MVP scope, initial risks, open questions |
| v2 | Detailed requirements and acceptance criteria per pillar, event schemas, metrics with baselines and targets, experimentation strategy with phased rollout, risk register with mitigations, dependency list |
| v3 | Resolved open questions, full experiment backlog with power calculations, phased rollout plan with launch gates and kill switches, dependency map with owners, SLO revision history, production-grade instrumentation spec |

---

## Context

Cart abandonment between "Review Cart" and "Pay" is Zomato's highest-ROI conversion lever. Over 30% of drop-offs cluster in this window, driven by three failure modes: price delta shock (fees visible only at checkout), offer roulette (manual code testing), and commitment anxiety (no reassurance on total finality). This PRD specifies three feature-flagged modules - Early Price Transparency, Offer Decisioning, and Checkout Confidence Nudges - designed to lift cart-to-order conversion by 1.5 to 2 percentage points.

v3 is the production-grade specification and direct input for the system architecture document and launch readiness review. All open questions from v1 and v2 are resolved with explicit decisions. The rollout plan includes gates, kill switches, and dependency owners.

---

## 1. Problem Statement & Opportunity Size

Cart to order conversion on the primary ordering funnel lags the food-delivery benchmark by ~4-6 p.p., with **>30% of drop-offs** clustered between "Review Cart" and "Pay" per teardown heuristics and public metrics. Qual/quant signals:

- CSAT verbatims cite **"fees keep changing"** and **"coupon didn't work"** as top-3 reasons for backing out.
- Users who see a **delta > Rs 40** between menu total and payable amount are **2.1x more likely** to abandon.
- First-time or reactivated users churn 20% more often when an offer fails to apply.
- Offer-related support contacts account for an estimated 12-15% of checkout support volume.

**Opportunity:** Lift cart-to-order conversion by **1.5-2 p.p.** via earlier expectation setting and automated reassurance, yielding ~+0.5% orders per MAU. At Zomato's urban-India scale, this translates to tens of thousands of incremental orders per month - all without changing the fee model or margin structure.

**Competitive context:** Swiggy surfaces an estimated total in the cart drawer; Blinkit uses a "You pay exactly Rs X" lock at payment. Both signal that price finality is table-stakes in quick-commerce. Zomato's delay in surfacing this erodes trust with price-aware users who compare offers across apps before committing.

---

## 2. Goals & Non-Goals

### Goals

1. Increase cart to order conversion (primary success metric).
2. Reduce fee and offer-related confusion (measured through event drop-offs and CSAT tags).
3. Preserve or improve contribution margin by nudging clarity, not blanket discounts.

### Non-Goals

- Repricing or waiving platform or delivery fees.
- Redesigning the complete checkout UI outside specified modules.
- Modifying restaurant or partner-side workflows.
- Extending transparency to menu or listing surfaces in this phase (decision documented below).

---

## 3. Personas & Use Cases

| Persona | Behaviors | JTBD in checkout |
|---------|-----------|------------------|
| **Urban frequent (anchor persona)** | Orders 2-5x/week, price-aware yet convenience-first | "Tell me the real total before I tap Pay so I can commit quickly." |
| **Deal seeker / value hunter** | Cross-checks offers, flips between apps | "Show me the best applicable offer automatically so I don't do extra math." |
| **First-time / trust-sensitive** | Limited history, higher leakage | "Reassure me that nothing else will change after I confirm." |
| **COD user** | Prefers cash on delivery, lower digital trust | "Don't change the amount I see - I'm counting out cash mentally." |

Key journeys covered: Mobile app cart review -> checkout -> payment selection -> order confirmation. Desktop, Dine-in, and Paytm mini-app excluded from v1 rollout.

**Edge case - COD users:** COD users see the "fee lock banner" but without the payment-method-specific copy. Messaging is adjusted to "Your order total is locked at Rs X" (no reference to card/UPI). COD-specific instrumentation split is added to all events (see Section 8).

---

## 4. Insights from Teardown & Data

1. **Price delta shock:** Fees surface only after tapping checkout; cognitive load peaks right before payment. The transition from "items total" to "payable total" is the primary abandonment trigger.
2. **Offer roulette:** Users must copy and paste codes, leading to failures (ineligible basket, bank-only offers, min order value not met). Each failed attempt compounds frustration and increases the likelihood of switching to a competitor.
3. **Commitment anxiety:** Lack of affirmations (ETA, no extra fee) and "what if" states (restaurant rejects, price surge) nudge re-evaluation. Users who see reassurance copy have higher click-through to payment confirmation.
4. **Instrumentation gaps:** No unified events for "fee breakdown viewed" vs "offer error" in v2; isolating reason codes is manual. v3 closes this with a unified `checkout_funnel_event` schema.
5. **COD-specific behavior:** COD users abandon at a 1.4x higher rate when fees are ambiguous, suggesting stronger price-certainty needs than UPI/card users. This is an untapped segment for confidence nudges.

---

## 5. Solution Pillars

1. **Early Price Transparency** - display an estimated payable amount (items + fees + taxes) in-cart, before checkout CTA.
2. **Offer Decisioning & Explanation** - auto-apply the best offer, surface "why" copy, and allow override without modal hopping.
3. **Checkout Confidence Nudges** - reassure users post-offer decision with ETA validation, fee lock message, and fallback copy for failure states.

Each pillar ships as a feature-flagged module but is designed to work cumulatively. Pillar 3 depends on Pillar 1 data (stable totals signal) being available.

---

## 6. Detailed Requirements & Acceptance Criteria

### 6.1 Early Price Transparency

**User stories**
- As a frequent user, I want to see an accurate total before checkout so I don't abandon at payment.
- As a trust-sensitive user, I want a breakdown of charges so I believe Zomato is fair.

**Requirements**
1. **Estimated Total Card** sits above the "Proceed to Pay" CTA in cart:
   - Shows `items_subtotal`, `platform_fee`, `delivery_fee`, `taxes`, `total_payable`.
   - Uses cached fee service to avoid blocking cart load (>95% responses <150 ms p95).
2. If any fee is dynamic (surge, busy area), show a badge `Likely to stay within +/-Rs 10` with tooltip.
3. Provide "Why this fee?" inline link -> bottom sheet (re-uses existing FAQ content; no new engineering for content layer).
4. Card collapses on cart items change to reflect updated totals within 400 ms.

**Acceptance criteria**
- Estimate accuracy >= 90% within +/-Rs 5 vs final payable for non-surge orders.
- For surge/dynamic fees, accuracy band displayed; if final total drifts beyond band, log `fee_band_breach` event.
- Card renders within 400 ms of cart screen load on median device (Redmi Note 11 equivalent).
- Card is not shown if fee service call fails (graceful degradation; do not show stale or empty state).

**Edge cases**
- *Restaurant goes busy mid-cart:* Estimated Total Card refreshes asynchronously; shows updated delivery fee with a "Delivery time updated" label.
- *Multiple restaurants in cart:* Not applicable (Zomato single-restaurant cart model); guard against any future multi-restaurant experiment surfacing this.

**Instrumentation additions**

| Event | Properties | Purpose |
|-------|------------|---------|
| `cart_fee_breakdown_view` | `city_id`, `user_segment`, `subtotal`, `fees_shown`, `payment_method_default` | Track exposure |
| `cart_fee_breakdown_toggle` | `tapped_section`, `dwell_ms` | Measure engagement |
| `fee_band_breach` | `order_id`, `expected_total`, `final_total`, `fee_type` | Alert pricing team |
| `cart_fee_card_suppressed` | `reason` (fee_service_timeout, surge_active) | Track graceful degradation rate |

### 6.2 Offer Decisioning & Explanation

**User stories**
- As a deal seeker, I want Zomato to auto-apply the best eligible offer.
- As any user, I want to know why an offer is unavailable without testing codes.

**Requirements**
1. **Auto-apply engine** queries Offer Service with cart context (`restaurant_id`, `payment_method`, `user_tenure_bucket`, `basket_value`) and applies the highest savings offer that meets guardrails (GMV >= threshold, contribution margin floor).
2. **Reason chips:** Beneath the applied offer, show plaintext reason (e.g., "Best offer applied: Zomato Treats") with optional `Change` CTA.
3. If no offer applies, show `No offers apply yet` with top disqualification reason (min order not met, incompatible payment method) and highlight the closest actionable next step (e.g., "Add Rs 30 more to unlock 20% off").
4. Ensure manual override path (user selects a different coupon) with confirmation message `Switching may change savings`.
5. **Gold upsell decision (resolved from v2):** Gold subscription upsell is shown only if (a) no offer applies AND (b) user is on a high-frequency cohort (3+ orders/month) AND (c) upsell banner is below the reason chip - not replacing it. It is not shown to first-time users or users who dismissed it in the last 30 days. This is gated by a separate feature flag and excluded from the primary conversion experiment to avoid confounding.

**Acceptance criteria**
- Auto-application latency <= 200 ms p95.
- Offer success rate +15% relative to control (measured via `offer_apply_success / offer_attempt`).
- Error copy localized (EN + Hindi + Tamil + Telugu + Kannada) and never blank.
- Gold upsell renders in < 1% of sessions for control group (suppressed via flag).

**Edge cases**
- *Offer expires between cart entry and checkout:* Show `Offer expired` chip with next best alternative pre-fetched; do not show blank state.
- *Bank-specific offer with undetected payment method:* Show `Add [Bank] card to unlock Rs X off` with link to payment methods screen.

**Instrumentation**

| Event | Properties |
|-------|------------|
| `offer_auto_apply_attempt` | `offer_id`, `savings_amount`, `success_bool`, `failure_reason`, `payment_method` |
| `offer_reason_view` | `reason_code`, `next_step_shown` |
| `offer_manual_override` | `from_offer_id`, `to_offer_id`, `delta_savings` |
| `offer_expired_shown` | `offer_id`, `alternative_offer_id` |
| `gold_upsell_shown` | `user_cohort`, `order_count_30d` |
| `gold_upsell_dismissed` | `dismiss_action` |

### 6.3 Checkout Confidence Nudges

**User stories**
- I want to know nothing else will be added after this step.
- If something does change (restaurant unavailable), tell me immediately with alternatives.

**Requirements**
1. **Fee lock banner** at top of payment screen: `All fees locked. You'll only pay Rs X.` Displayed once offers resolved. For COD users: `Your order total is locked at Rs X.`
2. **ETA reassurance:** show `Delivered in ~Y mins (90% reliability)` pulled from fulfillment service.
3. **Edge-case handling:**
   - If restaurant toggles to "busy" between cart and payment, show modal with updated ETA/fees + persistent "Review changes" CTA.
   - Provide one-tap support entry if a payment fails after debit but before confirmation (links to existing support flow).
4. **COD-specific behavior (resolved from v2):** COD users receive the same fee lock banner with adjusted copy. Post-Phase 1, COD conversion will be analyzed separately; if COD lift is < 0.5 p.p., the banner will be made COD-optional via flag. The experiment is powered on the aggregate (UPI + COD) with a COD subgroup analysis pre-registered.

**Acceptance criteria**
- Fee lock banner renders only when totals are stable (observed via `checkout_totals_stable` event within 300 ms).
- For reprice events mid-checkout, fallback modal must present new total AND `Continue`/`Choose another restaurant` options; default focus on `Continue`.
- ETA data freshness: pulled within 60 seconds of payment screen load; stale data suppresses ETA line, not banner.

**Edge cases**
- *Simultaneous reprice + offer expiry:* Show single consolidated modal with new total and new offer state; do not stack two modals.
- *Payment deducted but order not confirmed (network failure):* Trigger support entry CTA automatically after 10-second timeout; log `payment_orphan_detected` event.

**Instrumentation**

| Event | Properties |
|-------|------------|
| `checkout_confidence_banner_view` | `total_amount`, `eta_minutes`, `payment_method`, `is_cod` |
| `checkout_totals_stable` | `latency_ms`, `offer_applied` |
| `mid_checkout_reprice` | `delta_amount`, `reason`, `user_action` |
| `support_entry_checkout` | `reason_code`, `payment_state` |
| `payment_orphan_detected` | `order_id`, `payment_amount`, `gateway` |

---

## 7. Metrics & SLO Revision History

### 7.1 Success metrics

| Tier | Metric | Baseline (est.) | Target | Measurement window |
|------|--------|----------------|--------|-------------------|
| Primary | Cart to order conversion | ~72% | +1.5 to +2 p.p. | Daily, segmented by persona proxy |
| Secondary | Offer success rate | Baseline TBD in Phase 0 | +15% relative | `offer_apply_success / offer_attempt` |
| Secondary | Checkout abandonment (fee/offer tag) | ~30% of drop-offs | -20% share | CSAT + support tag |
| Secondary | Checkout p95 latency | Baseline from Phase 0 | No regression | Continuous |
| Guardrail | Contribution margin per order | Baseline | >= baseline | Daily |
| Guardrail | Payment failure rate | Baseline | <= baseline | Continuous |
| Guardrail | Support contact rate (checkout) | Baseline | <= baseline | Daily |

### 7.2 SLO revision history

| Version | SLO change | Rationale |
|---------|------------|-----------|
| v1 | None defined | Early-stage problem framing |
| v2 | Fee estimate accuracy >= 90% within +/-Rs 5; auto-apply latency <= 200 ms p95 | Set based on backend capability assessment |
| v3 | Added: Cart fee card renders within 400 ms; ETA data freshness within 60 s; fee lock banner renders within 300 ms of totals stable; payment orphan timeout 10 s | Production-grade thresholds aligned to fulfillment service SLAs and p95 device benchmarks |

---

## 8. Full Experiment Backlog with Power Calculations

### Experiment 1: Early Price Transparency (Pillar 1)

| Parameter | Value |
|-----------|-------|
| Primary metric | Cart to order conversion |
| Baseline conversion | 72% |
| Minimum detectable effect | 1 p.p. lift |
| Statistical power | 80% |
| Significance level | 95% (two-sided) |
| Required sample (per arm) | ~38,000 cart sessions |
| Estimated runtime at 10% traffic | 3-4 days (peak urban traffic) |
| Allocation | 50/50 treatment/control within 10% slice |

**Pre-registered subgroup analyses:** new vs repeat users (>3 orders/month), order value buckets (< Rs 200, Rs 200-500, > Rs 500), surge-active vs non-surge, COD vs UPI/card.

**Guardrail stops:** payment failure rate > baseline + 0.5 p.p. for 2 consecutive hours; checkout latency p95 regression > 200 ms.

### Experiment 2: Offer Decisioning (Pillar 2) - layered onto Experiment 1 treatment arm

| Parameter | Value |
|-----------|-------|
| Primary metric | Offer success rate, cart to order conversion incremental |
| Baseline offer success rate | TBD from Phase 0 instrumentation |
| MDE on offer success | 10% relative lift |
| Statistical power | 80% |
| Significance level | 95% |
| Required sample | ~20,000 sessions with offer exposure |
| Estimated runtime | 2-3 days |

**Gold upsell is excluded from this experiment** and tested independently in Phase 3 to avoid confounding.

### Experiment 3: Checkout Confidence Nudges (Pillar 3) - final layer

| Parameter | Value |
|-----------|-------|
| Primary metric | Cart to order conversion (cumulative three-pillar) |
| MDE | 0.5 p.p. incremental over Pillars 1+2 |
| Statistical power | 80% |
| Significance level | 95% |
| Required sample | ~80,000 cart sessions (lower base rate for incremental) |
| Estimated runtime | 5-7 days at 25% traffic |

**COD subgroup analysis pre-registered:** if COD lift < 0.5 p.p., COD-specific banner becomes optional via feature flag in post-Phase 1 config.

### Experiment 4: Gold Upsell (isolated, Phase 3)

| Parameter | Value |
|-----------|-------|
| Eligibility | No-offer + high-frequency cohort only |
| Primary metric | Gold subscription starts (not conversion - tested independently) |
| Guardrail | Cart to order conversion does not decrease in exposed cohort |
| Runtime | 14 days minimum (subscription decision cycle) |

### Experiment 5: Menu-level Price Preview (Phase 4 - future, not in this PRD)

**Decision (resolved from v2):** Extending price transparency to menu and listing surfaces is explicitly deferred to a follow-on PRD, contingent on Phase 1-2 proving statistically significant lift and instrumentation proving stable. Reason: menu-level previews require restaurant catalog integration and dynamic fee prefetch at scale - a separate infrastructure investment. This PRD closes that scope.

---

## 9. Phased Rollout Plan

### Phase 0: Dogfood + Instrumentation Validation (Week 1-2)

**Scope:** Internal Zomato employees on Android (primary device cohort).

**Gate criteria to advance:**
- All events in Section 8 firing correctly (verified in Amplitude + internal event pipeline).
- Fee estimate accuracy >= 90% on sample of 500 dogfood orders.
- No new crash rate in checkout flow (Crashlytics baseline).
- Latency SLOs met (p95 cart load unchanged).

**Kill switch:** Disable all three pillars via config service. No deploy required.

### Phase 1: 10% Traffic A/B (Week 3-4)

**Scope:** 10% of urban mobile app users (Android + iOS), top 8 cities (Mumbai, Delhi NCR, Bengaluru, Hyderabad, Pune, Chennai, Kolkata, Ahmedabad). Treatment = Pillars 1 + 2. Control = current experience.

**Launch gates:**
1. Phase 0 gate criteria met.
2. Data Science sign-off on experiment config and guardrail definitions.
3. Localization strings reviewed and merged for EN, Hindi, Tamil, Telugu, Kannada.
4. Offers platform team confirms auto-apply API at <= 200 ms p95 on staging load test (10k req/min).
5. Rollback runbook reviewed by on-call engineer.

**Success criteria to advance to Phase 2:**
- Primary metric: >= 1 p.p. lift in cart to order conversion (p = 0.05, one-tailed) OR >= 0.7 p.p. lift with all secondary metrics trending positive.
- No guardrail violation over 3 consecutive days.

**Kill switch:** Per-pillar feature flags (three independent toggles). PagerDuty alert triggers auto-disable if conversion drops > 1 p.p. below control for 2 consecutive hours.

### Phase 2: 25% Traffic + Pillar 3 (Week 5-6)

**Scope:** 25% of urban mobile users. Add Pillar 3 (Checkout Confidence Nudges) as an additional layer on the treatment arm.

**Launch gates:**
1. Phase 1 success criteria met.
2. Fulfillment service team confirms ETA freshness SLO (within 60 seconds, p95).
3. Mid-checkout reprice modal QA'd on all target device profiles.
4. Support team briefed on new support entry point from checkout.

**Kill switch:** Pillar 3 has its own flag, independent of Pillars 1+2.

### Phase 3: 100% Rollout (Week 7-8)

**Scope:** All mobile users, then Web (with UI adaptation). Region sequence: top 8 cities -> Tier 2 cities -> Rest of India.

**Launch gates:**
1. Phase 2 cumulative lift >= 1.5 p.p. (primary metric).
2. Guardrail metrics stable across Phase 1 and 2.
3. COD subgroup analysis reviewed; COD flag decision documented.
4. Exec sign-off (PM + Eng + Finance lead) on contribution margin guardrail.
5. Post-Phase 2 experiment report filed and linked in this PRD.

**Kill switch:** Global disable toggle remains active post-launch for 30 days. After 30 days, per-pillar flags are maintained as permanent configuration levers.

### Phase 4: Gold Upsell Experiment (Week 9-10, separate)

Runs in isolation on the now-stable checkout experience. See Experiment 4 parameters above.

---

## 10. Dependency Map with Owners

| Dependency | Owner team | What is needed | Needed by phase |
|------------|------------|----------------|-----------------|
| Fee / Pricing Service API | Platform Engineering | Cached fee endpoint, <= 150 ms p95, surge confidence band field | Phase 0 |
| Offers Platform API | Growth Engineering | Auto-apply endpoint with basket context + reason codes + 200 ms p95 SLA | Phase 0 |
| Experimentation Platform (Nova) | Data Platform | Experiment config, guardrail alerts, subgroup splits | Phase 1 |
| Localization / Content | Design + L10n | Copy strings in 5 languages (EN, Hindi, Tamil, Telugu, Kannada) | Phase 1 |
| Fulfillment / ETA Service | Logistics Engineering | ETA endpoint with freshness timestamp field | Phase 2 |
| Config Service | Core Engineering | Feature flag infrastructure for per-pillar toggles | Phase 0 |
| Analytics / Amplitude | Data Engineering | Event schema validation, derived dashboard setup | Phase 0 |
| Support Team | Customer Experience | Briefing on new checkout support entry, tagging taxonomy update | Phase 2 |
| CRM / Marketing | Growth | Gold upsell eligibility rules, 30-day suppression logic | Phase 4 |
| Data Science | Analytics | Persona proxy definitions, experiment power review, COD subgroup pre-registration | Phase 1 |

---

## 11. Resolved Open Questions

| Question (from v1/v2) | Decision | Rationale |
|-----------------------|----------|-----------|
| Should we expose Gold subscription upsell when no offer applies? | **Yes, but gated.** Show only to high-frequency users (3+ orders/month) who have no active offer, and only if they have not dismissed in 30 days. Tested in Phase 4 as an isolated experiment. | Avoids confounding conversion experiment; high-frequency users are the right target for subscription upsell. |
| Do COD users respond differently to fee lock messaging? | **Hypothesis: yes. Design accounts for this.** COD users get adjusted copy ("Your total is locked at Rs X" - no payment method reference). COD subgroup analysis pre-registered in Phase 1. If COD lift < 0.5 p.p., the banner becomes COD-optional via flag post-Phase 1. | COD users have higher price-certainty needs per teardown data; separate copy reduces confusion while keeping them in the experiment. |
| Should we extend transparency to menu/listing surfaces? | **Deferred to a follow-on PRD.** Not in scope for this initiative. Decision logged and linked. | Requires separate infrastructure (restaurant catalog + dynamic fee prefetch at listing scale). Proves the cart-layer first; menu extension is a larger, distinct project. |

---

## 12. Risks & Mitigations

| Risk | Type | Mitigation |
|------|------|------------|
| Estimated totals wrong during surge | Trust | Display band +/- Rs 10; suppress banner if confidence < 80%; log `fee_band_breach` for pricing team |
| Auto-applied offer increases subsidy cost | Business | Apply contribution margin guardrails; monitor margin per order daily; Gold upsell tested separately |
| UI crowding on small screens | UX | Use collapsible cards; progressive disclosure for fee FAQ; QA on Redmi Note 11 (median device) |
| Localization gaps cause mistrust | UX / Trust | Copy review with language owners; fallback to EN + icon if string missing |
| Added service calls slow checkout | Perf | Cache fee + offer responses for session; enforce SLA monitors; suppress module on timeout |
| COD users see confusing payment reference copy | UX | Separate COD copy variant; suppression flag if lift is insufficient |
| Mid-checkout reprice stacks with offer expiry | UX | Single consolidated modal spec'd in 6.3; QA test case for simultaneous events |
| Payment orphan (debit without confirmation) | Trust / Finance | 10-second timeout triggers support CTA; `payment_orphan_detected` event feeds reconciliation pipeline |

---

## 13. Launch Readiness Checklist

- [ ] Phase 0 gate criteria met and documented.
- [ ] All event schemas validated in staging Amplitude.
- [ ] Fee estimate accuracy >= 90% confirmed on 500+ dogfood orders.
- [ ] Localization strings merged for all 5 languages.
- [ ] Offers platform load-tested at 10k req/min with p95 <= 200 ms.
- [ ] Experiment config reviewed and approved by Data Science.
- [ ] PagerDuty alerts configured for conversion regression and payment failure guardrails.
- [ ] Per-pillar kill switches verified working in staging.
- [ ] Support team briefed and tagging taxonomy updated.
- [ ] COD subgroup analysis pre-registered with Data Science.
- [ ] Exec sign-off on margin guardrail definition.
- [ ] Rollback runbook reviewed by on-call engineer.
- [ ] Post-launch monitoring dashboard live (Data Studio) with primary + guardrail metrics.

---

## 14. Final Takeaway

This PRD does not change Zomato's fee model. It aligns **expectations and confidence earlier** so users do not abandon when friction peaks. The three pillars - price clarity in cart, automatic offer decisioning with explanations, and fee lock reassurance at payment - are individually reversible but cumulatively designed to move conversion by 1.5-2 p.p.

v3 is the production-grade specification: all open questions are resolved, the rollout is gated and kill-switch-enabled, the experiment backlog is powered, and the dependency map has owners. The direct input to the architecture document is this PRD's instrumentation schema, the three feature-flagged module boundaries, and the service dependencies in Section 10.

**The menu-level transparency extension is explicitly deferred.** Once this checkout layer proves impact, that follow-on PRD inherits both the instrumentation foundation and the credibility to request the larger infrastructure investment.
