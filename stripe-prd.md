# PRD: Stripe AI Billing Primitives - Native Credit Ledger and Agentic Payment Rails

**Product:** Stripe Billing (AI Primitives layer)
**Author:** Mayank Malviya
**Status:** v1 - Initial PRD
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/stripe-teardown.md

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, target personas, three solution pillars, core loops, basic requirements, event schemas, success metrics, competitive context, open questions |

---

## Context

Stripe is already the dominant payment processor for AI companies. OpenAI, Anthropic, Mistral, Perplexity, and the majority of AI-native startups run payments through Stripe. This creates a structural product problem: these companies all have a billing model (token-based credits, metered API usage, or credit pack purchases) that does not map cleanly onto Stripe's native primitives.

Today, every AI company that builds on Stripe must construct one or more of the following from scratch:

1. A credit balance ledger (Stripe has no native credit object)
2. A real-time balance check API (the `Meter` object aggregates asynchronously - it is not queryable per-inference in real time)
3. An automatic top-up flow (threshold-triggered recharge requires custom logic outside Stripe)
4. An agent-payment delegation model (no Stripe product handles "AI agent initiates payment on behalf of user with scoped permission")

This PRD specifies **Stripe AI Billing Primitives**: four new first-party objects and flows that close these gaps and make Stripe the complete billing infrastructure for AI-native products - without forcing merchants to build parallel systems.

---

## 1) Problem statement + why now

### The integration gap

AI products have a billing model that is structurally different from every other product category Stripe was built to serve:

| Billing model | What AI companies do today | Stripe native support |
|---|---|---|
| Credit pack purchase (100K tokens upfront) | `PaymentIntent` + custom credit DB table | Partial - payment works; ledger is DIY |
| Metered post-pay (per token consumed) | `Meter` events + monthly invoice | Partial - works for monthly billing; not real-time |
| Hybrid (subscription + overage) | Subscription + metered Price + custom overage math | Partial - proration on overages is complex |
| Real-time per-inference deduction | Custom Redis/Postgres ledger + Stripe for periodic charge | Unsupported - entirely DIY |
| Agent-initiated payment (AI buys on user's behalf) | Not currently possible in Stripe's auth model | Unsupported |

The result: every AI company that grows past $1M ARR has built a bespoke billing service that sits between their product and Stripe. This parallel system introduces reconciliation risk, engineering overhead, and is a direct reason for AI companies to evaluate alternative billing platforms (Lago, Orb, Metronome) that were purpose-built for usage-based billing.

### Observable failure modes

- **Revenue leakage:** A user's credit balance hits zero during an inference session. Without a real-time balance check API, the product either blocks the request (bad UX) or continues serving and loses money until the next reconciliation window.
- **Top-up friction:** A user's balance drops below a threshold. The AI product has to implement its own polling job, detect the threshold, create a Stripe `PaymentIntent`, and update the credit ledger - four systems coordinating for one UX moment. Top-up conversion is materially lower because of the latency.
- **Disputed credits:** A user files a chargeback on a credit pack purchase. The merchant's credit ledger has already debited the credits. Reversing the chargeback reverses the payment but the credits are already consumed - requiring manual reconciliation per dispute.
- **Agentic commerce blind spot:** An AI agent authorised by a user to "book travel" needs to create a `PaymentIntent`. Currently, there is no Stripe product that models "agent acts on behalf of user with scoped, revocable permission." The agent must use the user's stored payment method directly - no scope limits, no per-session revocation, no agent-specific audit trail.

### Why now

- The `Meter` object GA'd in 2024 - Stripe has made the first move toward usage-based billing infrastructure. The logical next step is completing the stack.
- Lago, Orb, and Metronome are growing specifically because Stripe does not cover this gap. These are not adjacent competitors; they are winning on the exact problem this PRD solves.
- Stripe's agentic payment MCP tools shipped in early 2025, signalling internal alignment that agentic commerce is a priority. This PRD is the product-layer complement.
- The AI billing market is growing at >100% annually. Every month of delay means more AI companies build their own ledger service and reduce their Stripe dependency.

---

## 2) Goals / non-goals

### Goals

1. Increase **Stripe Billing attach rate among AI-category merchants** from an estimated 45% to **65%** within 6 months of GA.
2. Achieve **Meter event activation rate >60%** among AI-category merchants within 60 days of first live charge.
3. Drive **AI merchant ARPU** (Payments + Billing combined) to 2x the ARPU of Payments-only AI merchants.
4. Reduce **AI company integration support tickets** related to credit ledger and top-up by **40%** (measured against current volume for AI-category accounts).
5. Capture **$10M+ in delegated GPV** from agentic platforms within 6 months of closed beta launch.

### Non-goals

- Replacing Lago, Orb, or Metronome for enterprise usage-based billing with multi-dimensional pricing matrices. Those products serve finance teams with custom SLAs. This PRD targets product and engineering teams building AI products.
- Building a full real-time fraud model for agent-initiated transactions (this is Radar's roadmap).
- Changing core `Subscription` or `Invoice` behaviour. These primitives remain unchanged; the new objects compose with them.
- Supporting crypto or stablecoin denominated credits in v1. Bridge integration is a separate workstream.

---

## 3) Target users / personas + JTBD

| Persona | Company stage | Primary billing model | Pain today | JTBD |
|---|---|---|---|---|
| **AI startup founder / solo engineer** | Seed to Series A; 1-5 engineers | Credit packs; no subscriptions yet | Building credit ledger as first non-product code written; top-up is manual | "Let me monetise tokens without building a billing service" |
| **Platform engineering lead** | Series A to C; dedicated billing squad | Metered post-pay; hybrid subscription + overage | Custom reconciliation job between Stripe and internal DB; constant drift | "Give me one authoritative source of truth for usage and balance" |
| **AI product PM** | Growth stage; >$5M ARR | Per-seat or hybrid with metered add-on | Overage billing breaks subscription mental model; per-seat doesn't capture value at AI scale | "Price on value - tokens consumed - without confusing my customers" |
| **Agentic platform builder** | Seed to Series B | Agent-transacted purchases (travel, ecommerce, SaaS) | No way to let an AI agent initiate payment with bounded scope; must use raw stored credentials | "Let my agent pay on behalf of users safely, with user-visible audit trail" |

**Activation insight:** For AI startups, the aha moment is not the first `PaymentIntent` - it is the moment the credit balance system works end-to-end without a parallel database. The product must collapse the "write my own ledger" step from a week of engineering to a zero-config first-party object.

---

## 4) Insights from teardown

1. **Stripe's AI billing gap is structural, not tactical.** The teardown identifies that the `Meter` object is the foundation but explicitly notes the remaining gap: "a native real-time balance API that merchants can call per-inference to check and deduct credits without building a parallel ledger service."

2. **Stripe already processes most AI-company volume.** This is a conversion problem, not an acquisition problem. The merchants are already on Stripe. The product task is to capture the billing layer they're currently building themselves.

3. **Lago/Orb are winning on this exact gap.** The teardown notes that usage-based billing platforms "were purpose-built for usage-based billing." Stripe's response must be a native product, not documentation encouraging workarounds.

4. **Agentic payment delegation is the next frontier.** The teardown explicitly identifies "OAuth-style delegation for agentic payments where the AI agent has scoped, revocable permission to initiate specific payment types up to a defined limit" as unresolved and strategically important.

5. **Capital and lock-in compound here.** An AI company that embeds Stripe's credit ledger as its balance source of truth is more deeply locked into Stripe than one using only Payments. Every dispute, every top-up event, every balance query runs through Stripe - ARPU and switching cost both increase.

---

## 5) Solution - three pillars

### Pillar 1: `CreditBalance` - native credit ledger object

A first-party Stripe object that tracks a customer's purchased credits. Deductions are API calls, not database updates in the merchant's system.

**Core behaviour:**
- A `CreditBalance` is attached to a `Customer` and denominated in a unit the merchant defines (tokens, API calls, credits, minutes).
- Credits are added by a `CreditGrant` event (triggered by a successful `PaymentIntent` for a credit pack, or manually by the merchant).
- Credits are deducted by a `CreditDeduction` call (synchronous API; returns updated balance in response).
- Balance is readable via `GET /v1/customers/{id}/credit_balance` with sub-100ms P95 latency.
- Negative balance is configurable: allow (run tab, invoice at end of period) or block (hard stop when balance = 0).

**Why this wins:** The merchant no longer needs a parallel database for credits. Stripe is the single source of truth. Disputes automatically trigger credit reversal via a `charge.dispute.created` event handler that Stripe provides as a webhook-triggered template.

### Pillar 2: `AutoTopUp` - threshold-triggered recharge

A first-party Stripe flow that recharges a customer's `CreditBalance` when it drops below a configured threshold.

**Core behaviour:**
- Merchant configures `AutoTopUp` on a `Customer`: `threshold` (trigger balance), `recharge_amount` (credits to add), `payment_method` (stored `PaymentMethod` or default source).
- When `CreditBalance` drops below `threshold`, Stripe triggers a `PaymentIntent` using the stored payment method, creates a `CreditGrant` on success, and emits an `auto_top_up.succeeded` or `auto_top_up.failed` event.
- Merchant receives webhooks; no polling job required.
- Customer sees a clear charge description: "Token refill - [Product Name] - [amount] credits."

**Edge case - top-up payment fails:** `AutoTopUp` follows the Smart Retries schedule (ML-timed retry). After N failed attempts, the balance enters a `below_threshold_unresolved` state. Stripe emits `auto_top_up.failed_final` and the merchant's dunning flow handles the rest. Merchant can configure "block new requests on failed top-up" or "allow usage on tab."

**Edge case - concurrent inference requests drain balance simultaneously:** Multiple parallel API calls from the same customer can simultaneously attempt `CreditDeduction`. Stripe handles this with optimistic locking at the balance object level - deductions are serialised, not best-effort. If balance is insufficient for a deduction, the API returns `402 Insufficient Credits` with the current balance in the response body. Merchant decides whether to allow or block at the application layer.

### Pillar 3: `PaymentDelegate` - scoped agent-initiated payment

An OAuth-style delegation model that allows an AI agent to initiate payment on behalf of a user with explicitly scoped, revocable permission.

**Core behaviour:**
- A user grants a `PaymentDelegate` to an agent application. Scope parameters: max amount per transaction, allowed merchant categories (MCCs), validity window (e.g., 24h, one trip booking, etc.), and total spend cap.
- The `PaymentDelegate` is a first-party Stripe object with its own ID, audit log, and revocation endpoint.
- When the agent creates a `PaymentIntent`, it passes the `payment_delegate_id`. Stripe validates scope (amount, MCC, window) before authorising.
- The user sees every agent-initiated transaction in their Stripe-hosted receipts as "Authorised by [Agent Name]" with the delegate scope visible.
- Users can revoke a `PaymentDelegate` at any time; in-flight `PaymentIntent`s that have not yet been confirmed are cancelled.

**Why this is the right abstraction:** Card networks already have a stored credential model for recurring charges. `PaymentDelegate` is a layer on top that adds agent identity, scope limits, and user-visible auditability - things the stored credential model does not provide. It is not a new card network primitive; it is a Stripe-side enforcement layer with the same underlying card rails.

**Edge case - agent attempts out-of-scope transaction:** The `PaymentIntent` creation call returns `403 Delegate Scope Exceeded` with the violated scope parameter in the response. The agent surfaces this to the user as a permission error, not a payment failure - different UX handling.

**Edge case - user revokes delegate mid-transaction:** If the user revokes while the `PaymentIntent` is in `requires_confirmation` state, Stripe cancels the intent and returns a `payment_intent.cancelled` event to both the merchant and the agent application. Agent must handle this gracefully (abort the task, notify user).

---

## 6) Core loops

### Loop A - Purchase credits -> use product -> auto-refill (credit lifecycle loop)

```
User purchases credit pack
  -> PaymentIntent confirmed -> CreditGrant created -> CreditBalance updated
     |
     v
User consumes product (API calls, inferences, generations)
  -> CreditDeduction called per request -> balance decrements in real time
     |
     v
Balance drops below threshold
  -> AutoTopUp triggered -> PaymentIntent created -> CreditGrant on success
     |
     v [if top-up fails]
  -> dunning flow -> balance freeze or tab mode
     |
     v [if top-up succeeds]
  -> balance restored -> user continues without interruption
```

**Loop metric:** Auto-refill success rate (target: >85% of triggered top-up events succeed on first or second attempt). Time from `below_threshold` to balance restored (target: <30s P95 for top-up flows with a stored payment method).

**Reinforcement:** Every successful auto-refill is a zero-friction upsell. The user bought more credits without a checkout step. This is the AI billing equivalent of Stripe's Smart Retries recovering failed payments - compounding ARPU without merchant effort.

### Loop B - Agent authorised -> agent transacts -> user reviews (agentic commerce loop)

```
User authorises PaymentDelegate for agent application
  -> delegate scope set (amount cap, MCC, window)
     |
     v
AI agent executes task (book flight, purchase API credits, subscribe to tool)
  -> agent creates PaymentIntent with payment_delegate_id
  -> Stripe validates scope -> confirms payment
  -> agent_payment.succeeded event emitted
     |
     v
User reviews agent spend in Stripe receipt / platform UI
  -> sees "Authorised by [Agent Name]" with scope disclosure
     |
     v
User adjusts delegate scope or revokes for future tasks
```

**Loop metric:** Delegated GPV as share of total AI-category merchant GPV (target: 5% within 12 months of closed beta). `PaymentDelegate` revocation rate within 7 days of first agent transaction (guardrail: <10% - high revocation signals trust failure).

---

## 7) Key event schemas

### `credit_balance.updated`

```json
{
  "id": "evt_1abc...",
  "object": "event",
  "type": "credit_balance.updated",
  "created": 1747123456,
  "data": {
    "object": {
      "id": "cb_1abc...",
      "object": "credit_balance",
      "customer": "cus_abc123",
      "unit": "tokens",
      "balance": 84200,
      "previous_balance": 100000,
      "last_deduction_id": "cd_1abc...",
      "below_threshold": false,
      "auto_top_up_id": "atu_1abc...",
      "livemode": true
    }
  }
}
```

### `auto_top_up.triggered`

```json
{
  "id": "evt_2abc...",
  "object": "event",
  "type": "auto_top_up.triggered",
  "created": 1747123500,
  "data": {
    "object": {
      "id": "atu_1abc...",
      "object": "auto_top_up",
      "customer": "cus_abc123",
      "trigger_balance": 10000,
      "current_balance": 9800,
      "recharge_amount": 100000,
      "payment_method": "pm_1abc...",
      "status": "pending",
      "payment_intent_id": "pi_1abc...",
      "livemode": true
    }
  }
}
```

### `payment_delegate.used`

```json
{
  "id": "evt_3abc...",
  "object": "event",
  "type": "payment_delegate.used",
  "created": 1747123600,
  "data": {
    "object": {
      "id": "pd_1abc...",
      "object": "payment_delegate",
      "customer": "cus_abc123",
      "agent_application_id": "app_1abc...",
      "payment_intent_id": "pi_1abc...",
      "amount": 4900,
      "currency": "usd",
      "scope": {
        "max_amount_per_transaction": 10000,
        "allowed_mccs": ["7011", "4511"],
        "total_spend_cap": 50000,
        "spent_to_date": 4900,
        "valid_until": 1747209999
      },
      "status": "confirmed",
      "livemode": true
    }
  }
}
```

### `credit_deduction` API response

```json
{
  "id": "cd_1abc...",
  "object": "credit_deduction",
  "customer": "cus_abc123",
  "credit_balance_id": "cb_1abc...",
  "amount": 850,
  "unit": "tokens",
  "balance_before": 85050,
  "balance_after": 84200,
  "idempotency_key": "inference_session_xyz_call_12",
  "metadata": {
    "model": "gpt-4o",
    "session_id": "sess_xyz"
  },
  "status": "succeeded",
  "created": 1747123456
}
```

---

## 8) Basic requirements

### Req 1 - `CreditBalance` object

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 1.1 | `CreditDeduction` API responds with updated balance in <100ms P95 | Latency SLO met under 1,000 concurrent deduction calls/sec per merchant | Timeout >150ms: return cached balance with `balance_stale: true` flag; log `credit_deduction_latency_breach` |
| 1.2 | Concurrent deductions serialised at balance level; no overdraft beyond configured tolerance | Balance never goes below configured `minimum_balance` floor under race conditions (verified with concurrent load test) | If `minimum_balance` is not set, allow to zero then return `402 Insufficient Credits` |
| 1.3 | `CreditGrant` on successful `PaymentIntent` auto-creates within 5s of `payment_intent.succeeded` event | Verified by webhook lag measurement in test suite | If auto-grant fails, emit `credit_grant.failed` event; merchant must handle manually; do not silently lose credits |
| 1.4 | Balance readable via GET endpoint at <50ms P95 | Perf test across 10K merchant accounts | No read lock during concurrent deductions; reads are eventually consistent within 200ms |

### Req 2 - `AutoTopUp` flow

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 2.1 | Top-up triggered within 10s of balance crossing threshold | P95 trigger latency <10s measured from `credit_balance.updated` event with `below_threshold: true` | Network partition: trigger fires from balance update event; idempotency key prevents duplicate top-up if event delivered twice |
| 2.2 | Smart Retries applied to failed top-up `PaymentIntent` | Retry behaviour matches Smart Retries on standard subscription invoices | After final retry failure, emit `auto_top_up.failed_final`; balance status set to `top_up_failed`; no further auto-retries without merchant action |
| 2.3 | Merchant can configure `on_insufficient_balance` as `block` or `tab` | Both modes work; `tab` allows negative balance up to configured limit; `block` returns `402` at zero | If `tab` limit is reached, switch to `block` behaviour automatically; emit `credit_balance.tab_limit_reached` |

### Req 3 - `PaymentDelegate` object

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 3.1 | Scope validation runs before `PaymentIntent` confirmation; out-of-scope returns `403` with violated scope field named | <50ms scope check; correct error body in 100% of test cases | If scope record is unavailable (service error), fail safe: reject the `PaymentIntent` with `503 Delegate Validation Unavailable` |
| 3.2 | Revocation takes effect within 60s across all endpoints | Revoked delegate ID rejected by P99 within 60s globally | In-flight `PaymentIntent` in `requires_confirmation` at revocation time: cancel the intent; emit `payment_intent.cancelled` |
| 3.3 | User-visible audit trail: every `payment_delegate.used` event readable by the customer in hosted receipt UI | Audit trail shows agent name, amount, scope used, and timestamp | If agent application is deleted by merchant, replace agent name with "Deleted application" in audit trail; do not remove the record |

---

## 9) Competitive analysis

| Competitor | How they address AI billing | Stripe's gap vs. this competitor | This PRD's response |
|---|---|---|---|
| **Lago** (open-source metering) | Native credit ledger, real-time balance queries, usage-based invoicing; purpose-built for AI/API billing | Lago is free to self-host; Stripe currently requires DIY to match its feature set | `CreditBalance` + `AutoTopUp` as first-party objects; hosted, no infra overhead |
| **Orb** | Subscription + usage billing platform with sub-second metering, plan version control, and credit grants | Orb's metering latency is lower than Stripe's current `Meter` object; purpose-built billing UI for finance teams | Real-time `CreditDeduction` at <100ms closes the latency gap; finance-team UI is not this PRD's scope (v2) |
| **Metronome** | Enterprise usage-based billing with multi-dimensional pricing, ERP integrations, and revenue recognition | Metronome serves large enterprise with complex pricing; higher touch sales motion | This PRD targets seed to Series B AI companies; different motion. Metronome is not the head-to-head competitor here |
| **Recurly / Chargebee** | SaaS subscription billing; limited native usage-based billing | Neither has a real-time balance deduction primitive or agent-payment delegation | Not the relevant competitor for AI billing; legacy SaaS billing posture |

**Win condition:** Stripe wins the AI billing market if it ships `CreditBalance` before Lago achieves enterprise distribution. Lago's open-source model creates a free alternative, but Stripe's network effect (fraud data, global acquiring, Connect) means merchants who want to use Stripe as their payments layer still benefit from a native billing complement. The risk is a two-vendor world (Lago for billing, Stripe for payments) that reduces Stripe's ARPU and switching cost.

---

## 10) Success metrics

### North Star
**AI Billing Primitives Merchant Activation Rate** - percentage of AI-category Stripe merchants (identified by `Meter` usage or explicit AI category tag) who activate at least one of `CreditBalance`, `AutoTopUp`, or `PaymentDelegate` within 90 days of GA.

### Input metrics (leading indicators)

| Metric | Baseline (estimated) | 90-day target | Instrumentation event |
|---|---|---|---|
| `CreditBalance` objects created per week (AI-category merchants) | 0 (new product) | 500 | `credit_balance.created` count |
| `AutoTopUp` attach rate (of merchants with `CreditBalance`) | 0 | >70% | `auto_top_up.created` / `credit_balance.created` |
| Auto-refill success rate (first or second attempt) | 0 | >85% | `auto_top_up.succeeded` / `auto_top_up.triggered` |
| Delegated GPV ($) | 0 | $10M (closed beta) | `payment_delegate.used` sum of `amount` |
| AI merchant Billing attach rate | ~45% (est.) | 65% | merchants with active `Subscription` or `Meter` / total AI-category merchants |
| AI billing integration support ticket volume | Baseline T-90 days | -40% | Zendesk AI-billing label count |

### Guardrail metrics

| Metric | Threshold | Risk if breached |
|---|---|---|
| `CreditDeduction` P95 latency | <100ms | Inference products cannot tolerate >100ms billing overhead; merchants revert to DIY |
| `AutoTopUp` trigger P95 latency | <10s from threshold crossing | Users experience service interruption between balance hitting zero and top-up restoring |
| `PaymentDelegate` revocation P99 propagation | <60s | Revoked agents can initiate payments in the gap; user trust failure |
| Credit overdraft incidents (balance below `minimum_balance`) | Zero in prod test suite | Revenue leakage for merchant; Stripe liability if concurrent deduction bug causes overdraft |
| `PaymentDelegate` revocation rate within 7 days of first use | <10% | High revocation = trust failure in agentic payment model; signals UX or security concern |

---

## 11) Trade-offs

### Trade-off 1: Real-time balance at the cost of consistency window

Sub-100ms deduction latency requires an in-memory balance store with asynchronous persistence. This introduces a consistency window - if Stripe's in-memory layer fails between deduction and persistence, the deduction may be lost (credits consumed but not recorded) or double-counted (credits recorded twice). Idempotency keys on `CreditDeduction` calls mitigate double-counting. Lost deductions in failure scenarios are a revenue leakage risk.

**Decision:** Accept consistency window with idempotency key requirement. Require merchants to pass unique idempotency keys per inference call. Stripe's recovery path: if a deduction's idempotency key is replayed within 24h, return the original response without re-deducting.

### Trade-off 2: `PaymentDelegate` scope enforcement at Stripe vs. at the merchant

Stripe can enforce `PaymentDelegate` scope at the API layer (reject out-of-scope `PaymentIntent`s) or leave enforcement to the merchant and provide the delegate record as an audit primitive only. Enforcement at Stripe is more secure but reduces merchant flexibility. Enforcement at the merchant is weaker but allows merchants to define custom scope semantics.

**Decision for v1:** Enforce at Stripe for the defined scope fields (`max_amount_per_transaction`, `allowed_mccs`, `total_spend_cap`, `valid_until`). Merchants can add custom scope metadata that Stripe does not enforce - this is advisory metadata for the merchant's own application logic.

### Trade-off 3: `CreditBalance` currency (money vs. units)

Should `CreditBalance` be denominated in currency (cents) or in merchant-defined units (tokens, API calls, credits)? Currency denomination is simpler (maps directly to Stripe's existing money model). Unit denomination is more flexible (AI companies want to bill per token, not per cent, to decouple pricing changes from billing infrastructure changes).

**Decision:** Unit denomination with merchant-defined `unit` string. Stripe stores the balance in units; the merchant controls the conversion rate between units and currency. This decouples pricing changes (update the conversion rate in the merchant's application) from billing infrastructure (credit object unchanged).

---

## 12) Open questions

1. **Regulatory model for `PaymentDelegate`:** Card networks (Visa, Mastercard) have stored credential frameworks for recurring charges. Does agent-initiated payment fit stored credential rules, or does it require a new network-level primitive? What are the network rules on liability shift for agent-initiated transactions where the cardholder did not initiate the specific purchase?

2. **`CreditBalance` multi-currency:** If an AI company sells credit packs in USD and EUR, should `CreditBalance` support multiple balance buckets by currency, or should it be a single unit balance that the merchant converts? Multi-currency credit balances add significant complexity; single-unit is simpler but requires merchant-side FX handling.

3. **Chargeback handling for consumed credits:** If a user files a chargeback on a credit pack purchase after consuming 80% of the credits, what is the correct Stripe behaviour? Reversing the full payment while 80% of credits are consumed harms the merchant. Reversing only the unused credit value is complex and requires Stripe to track credit consumption at the `CreditGrant` level. This is the most complex dispute scenario and must be resolved before GA.

4. **`AutoTopUp` and SCA (PSD2):** Strong Customer Authentication in Europe requires explicit cardholder authentication for most payment initiation. Auto top-up is a merchant-initiated transaction without the cardholder present. Does this require SCA exemption requests (Stripe's Transaction Risk Analysis exemption) or stored credential rules, and what is the expected auth rate vs. a customer-present top-up?

5. **Design partner selection for `PaymentDelegate` closed beta:** Which 10 agentic platforms should be in the first closed beta cohort? Criteria: live agentic product, existing Stripe merchant, clear use case for scoped agent payment, and willingness to provide structured feedback every 2 weeks.

6. **`CreditBalance` vs. enhanced `Meter` object:** Internally, Stripe could implement `CreditBalance` as a specialised `Meter` with deduction semantics, or as a net-new object. The net-new object is cleaner from an API perspective but adds to Stripe's already-wide surface. The extended `Meter` reuses existing infrastructure but stretches the `Meter` object's semantics beyond its current design intent.

---

*All metrics are directional estimates based on public information, teardown analysis, and industry benchmarks - not internal Stripe data.*
