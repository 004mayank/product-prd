# PRD: Stripe AI Billing Primitives - Native Credit Ledger and Agentic Payment Rails

**Product:** Stripe Billing (AI Primitives layer)
**Author:** Mayank Malviya
**Version:** v3 - Final PRD
**Changes from v2:** Resolved all five v2 open questions (card network regulatory model for PaymentDelegate, multi-currency CreditBalance, chargeback handling for consumed credits, AutoTopUp SCA/PSD2 path, closed beta design partner criteria). Added phased rollout plan with launch gates and kill switches, dependency map, complete experiment backlog with acceptance criteria and rollout owners, and full version history.
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/stripe-teardown.md

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, target personas, three solution pillars, core loops, basic requirements, event schemas, success metrics, competitive context, open questions |
| v2 | Merchant activation funnel, full data model, API contracts, expanded event schemas, richer requirements with acceptance criteria, competitive benchmarks, experiment framework, risk register, resolved open question 6 (CreditBalance vs. extended Meter) |
| v3 | Resolved all five open questions (card network regulatory model, multi-currency, chargeback for consumed credits, SCA/PSD2, design partner criteria), phased rollout plan, launch gates, kill switches, dependency map, complete experiment backlog |

---

## Context

Stripe is already the dominant payment processor for AI companies. OpenAI, Anthropic, Mistral, Perplexity, and the majority of AI-native startups run payments through Stripe. This creates a structural product problem: these companies all have a billing model (token-based credits, metered API usage, or credit pack purchases) that does not map cleanly onto Stripe's native primitives.

Today, every AI company that builds on Stripe must construct one or more of the following from scratch:

1. A credit balance ledger (Stripe has no native credit object)
2. A real-time balance check API (the `Meter` object aggregates asynchronously - it is not queryable per-inference in real time)
3. An automatic top-up flow (threshold-triggered recharge requires custom logic outside Stripe)
4. An agent-payment delegation model (no Stripe product handles "AI agent initiates payment on behalf of user with scoped, revocable permission")

This PRD specifies **Stripe AI Billing Primitives**: four new first-party objects and flows that close these gaps and make Stripe the complete billing infrastructure for AI-native products - without forcing merchants to build parallel systems.

v3 is the production-grade specification. All open questions from v1 and v2 are resolved. The rollout plan includes gates, kill switches, and dependency owners. This document is the direct input to the system architecture document.

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

## 5) Merchant activation funnel

### Funnel: AI merchant -> CreditBalance activated

```
AI company creates Stripe account + processes first live payment
(estimated pool: ~5,000 new AI-category accounts per quarter)
           |
           v [Drop-off: ~35% never explore Billing docs beyond Payments]
Discovers AI Billing Primitives in docs or Dashboard prompt
           |
           v [Drop-off: ~25% who discover it do not start integration]
Reads CreditBalance API docs; creates first CreditBalance object in test mode
  --> AHA MOMENT CANDIDATE: first balance read returns correct value
           |
           v [Drop-off: ~40% who start test integration do not go live]
Integrates CreditDeduction into inference call path in test mode
           |
           v [Drop-off: ~30% who complete test integration delay live launch]
First live CreditDeduction call in production
  --> ACTIVATION EVENT
           |
           v [Drop-off: ~20% who activate CreditBalance do not attach AutoTopUp]
AutoTopUp configured on at least one Customer
  --> DEEP ACTIVATION (highest LTV signal)
```

**Estimated conversion by step:**

| Funnel step | Estimated conversion from prior step | Primary lever |
|---|---|---|
| Account -> Discovers AI Billing | ~65% | Dashboard prompt in Billing section for AI-category accounts; in-doc callout |
| Discovers -> Test integration | ~75% | Quality of quickstart template; copy-paste code example completeness |
| Test integration -> Live deduction | ~60% | Sandbox fidelity; clear error messages on common mistakes (missing idempotency key) |
| Live deduction -> AutoTopUp attach | ~80% | One-click AutoTopUp configuration prompt immediately after first live deduction |
| AutoTopUp attach -> PaymentDelegate interest | ~20% (distinct motion) | Separate funnel; relevant only for agentic platform builders |

**Funnel intervention targets:**
- Improving test-to-live conversion by 10 pp (from 60% to 70%) would add ~300 additional activated merchants per quarter from current account creation volume.
- The AutoTopUp attach prompt immediately post-first-live-deduction is the highest-leverage single intervention - it requires no additional engineering discovery, just a moment-of-activation nudge.

### Edge case: AI startup founder who writes their own ledger before discovering CreditBalance

**Scenario:** A founder integrates Stripe for payments in week 1, builds a Redis-based credit ledger in week 2, and discovers `CreditBalance` only when searching for "Stripe credits" three months later after their ledger has a reconciliation bug.

**Current state:** The founder now has two ledgers and must migrate their balance history into Stripe's `CreditBalance` object. There is no migration API.

**Required product response:** `CreditBalance` must support a `seed_balance` parameter on object creation that imports an existing balance snapshot without triggering a payment. This covers the migration case and reduces the adoption barrier for founders who built before the product existed. The seed event is recorded as a `credit_grant.type: migration` with no associated `PaymentIntent`.

---

## 6) Solution - three pillars

### Pillar 1: `CreditBalance` - native credit ledger object

A first-party Stripe object that tracks a customer's purchased credits. Deductions are API calls, not database updates in the merchant's system.

**Core behaviour:**
- A `CreditBalance` is attached to a `Customer` and denominated in a unit the merchant defines (tokens, API calls, credits, minutes).
- Credits are added by a `CreditGrant` event (triggered by a successful `PaymentIntent` for a credit pack, or manually by the merchant).
- Credits are deducted by a `CreditDeduction` call (synchronous API; returns updated balance in response).
- Balance is readable via `GET /v1/customers/{id}/credit_balance` with sub-100ms P95 latency.
- Negative balance is configurable: allow (run tab, invoice at end of period) or block (hard stop when balance = 0).

**Why this wins:** The merchant no longer needs a parallel database for credits. Stripe is the single source of truth. Disputes automatically trigger credit review via a `charge.dispute.created` event handler that Stripe provides as a webhook-triggered template.

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

## 7) Full data model

### `CreditBalance` object

```json
{
  "id": "cb_1abc...",
  "object": "credit_balance",
  "customer": "cus_abc123",
  "unit": "tokens",
  "unit_display_name": "API tokens",
  "balance": 84200,
  "minimum_balance": 0,
  "tab_limit": null,
  "on_insufficient_balance": "block",
  "auto_top_up_id": "atu_1abc...",
  "status": "active",
  "below_threshold": false,
  "created": 1747000000,
  "livemode": true,
  "metadata": {}
}
```

**Field specifications:**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `unit` | string | yes | none | Merchant-defined unit identifier (e.g., "tokens", "credits", "api_calls") |
| `unit_display_name` | string | no | same as `unit` | Human-readable label shown on receipts and hosted customer portal |
| `balance` | integer | system | 0 | Current balance in units; always non-negative unless `tab_limit` is set |
| `minimum_balance` | integer | no | 0 | Floor below which deductions are rejected; must be >= 0 |
| `tab_limit` | integer | no | null | Maximum negative balance allowed in tab mode; null means no tab |
| `on_insufficient_balance` | enum | no | "block" | `"block"` returns 402; `"tab"` continues until `tab_limit` |
| `below_threshold` | boolean | system | false | True when balance < `AutoTopUp.threshold`; set by Stripe on each deduction |
| `status` | enum | system | "active" | `"active"`, `"frozen"` (merchant action), `"top_up_failed"` (after final retry) |

### `CreditDeduction` object

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
  "created": 1747123456,
  "livemode": true
}
```

### `CreditGrant` object

```json
{
  "id": "cg_1abc...",
  "object": "credit_grant",
  "customer": "cus_abc123",
  "credit_balance_id": "cb_1abc...",
  "amount": 100000,
  "unit": "tokens",
  "type": "payment",
  "payment_intent_id": "pi_1abc...",
  "balance_before": 0,
  "balance_after": 100000,
  "expires_at": null,
  "metadata": {},
  "created": 1747100000,
  "livemode": true
}
```

**`CreditGrant.type` values:**

| Type | Trigger | Payment intent required |
|---|---|---|
| `payment` | `PaymentIntent` succeeded; credit pack purchased | yes |
| `auto_top_up` | `AutoTopUp` triggered and PaymentIntent succeeded | yes (auto-created) |
| `manual` | Merchant API call; for compensation, beta credits, admin adjustments | no |
| `migration` | Seed balance import from existing ledger | no |

### `AutoTopUp` object

```json
{
  "id": "atu_1abc...",
  "object": "auto_top_up",
  "customer": "cus_abc123",
  "credit_balance_id": "cb_1abc...",
  "threshold": 10000,
  "recharge_amount": 100000,
  "payment_method": "pm_1abc...",
  "status": "active",
  "retry_schedule": "smart_retries",
  "max_retries": 3,
  "last_triggered_at": null,
  "last_trigger_status": null,
  "created": 1747000000,
  "livemode": true,
  "metadata": {}
}
```

### `PaymentDelegate` object

```json
{
  "id": "pd_1abc...",
  "object": "payment_delegate",
  "customer": "cus_abc123",
  "agent_application_id": "app_1abc...",
  "agent_display_name": "Tripper AI",
  "scope": {
    "max_amount_per_transaction": 10000,
    "allowed_mccs": ["7011", "4511"],
    "total_spend_cap": 50000,
    "spent_to_date": 4900,
    "valid_until": 1747209999,
    "valid_after": 1747100000
  },
  "status": "active",
  "revoked_at": null,
  "revocation_reason": null,
  "created": 1747100000,
  "livemode": true,
  "metadata": {}
}
```

**`PaymentDelegate` status transitions:**

```
created -> active -> revoked (user action or expiry)
                  -> expired (valid_until passed)
                  -> spend_cap_reached (spent_to_date >= total_spend_cap)
```

---

## 8) API contracts

### Create `CreditBalance`

```
POST /v1/credit_balances
```

**Request:**

```json
{
  "customer": "cus_abc123",
  "unit": "tokens",
  "unit_display_name": "API tokens",
  "on_insufficient_balance": "block",
  "metadata": {}
}
```

**Response (201):**

```json
{
  "id": "cb_1abc...",
  "object": "credit_balance",
  "balance": 0,
  "status": "active"
}
```

**Error responses:**

| HTTP code | Error code | Condition |
|---|---|---|
| 400 | `balance_already_exists` | Customer already has a CreditBalance with the same `unit` |
| 400 | `invalid_unit` | `unit` string exceeds 64 characters or contains special characters |
| 404 | `customer_not_found` | Invalid `customer` ID |

---

### Create `CreditDeduction`

```
POST /v1/credit_deductions
Idempotency-Key: {merchant-generated unique key per inference call}
```

**Request:**

```json
{
  "customer": "cus_abc123",
  "amount": 850,
  "metadata": {
    "model": "gpt-4o",
    "session_id": "sess_xyz"
  }
}
```

**Response (200):**

```json
{
  "id": "cd_1abc...",
  "object": "credit_deduction",
  "balance_before": 85050,
  "balance_after": 84200,
  "balance_stale": false,
  "status": "succeeded"
}
```

**Error responses:**

| HTTP code | Error code | Condition | Merchant action |
|---|---|---|---|
| 402 | `insufficient_credits` | Balance is 0 or would go below `minimum_balance` | Block inference; show user credit purchase flow |
| 409 | `idempotency_key_replay` | Same key sent; deduction already processed | Return original response; do not re-deduct |
| 503 | `balance_service_unavailable` | Balance service degraded; response includes `balance_stale: true` | Allow or block per merchant's own policy; retry with backoff |

**Design note - `balance_stale` flag:** When the in-memory balance cache is behind the persistence layer (brief windows during service degradation), Stripe returns the last-known balance with `balance_stale: true`. Merchants should define their own policy: allow inference with a staleness tolerance or block (for high-value inference). The flag makes this explicit rather than silently serving a potentially outdated balance.

---

### Create `AutoTopUp`

```
POST /v1/auto_top_ups
```

**Request:**

```json
{
  "customer": "cus_abc123",
  "threshold": 10000,
  "recharge_amount": 100000,
  "payment_method": "pm_1abc..."
}
```

**Response (201):** Returns the `AutoTopUp` object.

**Error responses:**

| HTTP code | Error code | Condition |
|---|---|---|
| 400 | `threshold_exceeds_balance` | Configured `threshold` is greater than current `balance` - top-up would trigger immediately on creation |
| 400 | `no_credit_balance` | No active `CreditBalance` found for this customer |
| 400 | `duplicate_auto_top_up` | Customer already has an active `AutoTopUp`; update the existing one |

---

### Create `PaymentDelegate`

```
POST /v1/payment_delegates
```

**Request:**

```json
{
  "customer": "cus_abc123",
  "agent_application_id": "app_1abc...",
  "scope": {
    "max_amount_per_transaction": 10000,
    "allowed_mccs": ["7011", "4511"],
    "total_spend_cap": 50000,
    "valid_until": 1747209999
  }
}
```

**Response (201):** Returns the `PaymentDelegate` object.

---

### Revoke `PaymentDelegate`

```
POST /v1/payment_delegates/{id}/revoke
```

**Request:**

```json
{
  "reason": "user_requested"
}
```

**Response (200):**

```json
{
  "id": "pd_1abc...",
  "status": "revoked",
  "revoked_at": 1747150000,
  "revocation_reason": "user_requested"
}
```

**Propagation guarantee:** Revocation is propagated to all Stripe edge nodes within 60s (P99). Any `PaymentIntent` created with this delegate ID after propagation is complete returns `403 Delegate Revoked`.

---

## 9) Expanded event schemas

### `credit_balance.created`

```json
{
  "id": "evt_...",
  "type": "credit_balance.created",
  "data": {
    "object": {
      "id": "cb_1abc...",
      "customer": "cus_abc123",
      "unit": "tokens",
      "balance": 0,
      "status": "active",
      "created": 1747000000
    }
  }
}
```

### `credit_balance.updated`

```json
{
  "id": "evt_...",
  "type": "credit_balance.updated",
  "data": {
    "object": {
      "id": "cb_1abc...",
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

### `credit_balance.below_threshold`

Emitted when `balance` crosses below `AutoTopUp.threshold` for the first time in a session (not on every deduction while below threshold).

```json
{
  "id": "evt_...",
  "type": "credit_balance.below_threshold",
  "data": {
    "object": {
      "id": "cb_1abc...",
      "customer": "cus_abc123",
      "balance": 9800,
      "threshold": 10000,
      "auto_top_up_id": "atu_1abc...",
      "auto_top_up_status": "triggering"
    }
  }
}
```

### `credit_balance.tab_limit_reached`

```json
{
  "id": "evt_...",
  "type": "credit_balance.tab_limit_reached",
  "data": {
    "object": {
      "id": "cb_1abc...",
      "customer": "cus_abc123",
      "balance": -50000,
      "tab_limit": -50000,
      "on_insufficient_balance_switched_to": "block"
    }
  }
}
```

### `auto_top_up.triggered`

```json
{
  "id": "evt_...",
  "type": "auto_top_up.triggered",
  "data": {
    "object": {
      "id": "atu_1abc...",
      "customer": "cus_abc123",
      "trigger_balance": 10000,
      "current_balance": 9800,
      "recharge_amount": 100000,
      "payment_method": "pm_1abc...",
      "status": "pending",
      "payment_intent_id": "pi_1abc...",
      "attempt_number": 1,
      "livemode": true
    }
  }
}
```

### `auto_top_up.succeeded`

```json
{
  "id": "evt_...",
  "type": "auto_top_up.succeeded",
  "data": {
    "object": {
      "id": "atu_1abc...",
      "customer": "cus_abc123",
      "recharge_amount": 100000,
      "new_balance": 109800,
      "payment_intent_id": "pi_1abc...",
      "credit_grant_id": "cg_1abc...",
      "time_from_trigger_to_grant_ms": 4200
    }
  }
}
```

### `auto_top_up.failed_final`

```json
{
  "id": "evt_...",
  "type": "auto_top_up.failed_final",
  "data": {
    "object": {
      "id": "atu_1abc...",
      "customer": "cus_abc123",
      "attempts": 3,
      "last_failure_code": "card_declined",
      "credit_balance_status_set_to": "top_up_failed"
    }
  }
}
```

### `payment_delegate.used`

```json
{
  "id": "evt_...",
  "type": "payment_delegate.used",
  "data": {
    "object": {
      "id": "pd_1abc...",
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

### `payment_delegate.revoked`

```json
{
  "id": "evt_...",
  "type": "payment_delegate.revoked",
  "data": {
    "object": {
      "id": "pd_1abc...",
      "customer": "cus_abc123",
      "revoked_at": 1747150000,
      "revocation_reason": "user_requested",
      "in_flight_payment_intents_cancelled": ["pi_2abc..."]
    }
  }
}
```

---

## 10) Core loops

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

## 11) Full requirements

### Req 1 - `CreditBalance` object

| ID | Requirement | Acceptance criteria | Edge case | Test scenario |
|---|---|---|---|---|
| 1.1 | `CreditDeduction` API responds with updated balance in <100ms P95 | Latency SLO met under 1,000 concurrent deduction calls/sec per merchant | Timeout >150ms: return cached balance with `balance_stale: true` flag; log `credit_deduction_latency_breach` | Load test: 1,000 concurrent deductions against single customer; verify P95 and `balance_stale` flag on simulated timeout |
| 1.2 | Concurrent deductions serialised at balance level; no overdraft beyond configured tolerance | Balance never goes below configured `minimum_balance` floor under race conditions | If `minimum_balance` is not set, allow to zero then return `402 Insufficient Credits` | Race test: 50 simultaneous deductions of 100 tokens against a balance of 1,000 tokens; verify final balance is exactly 0, not negative |
| 1.3 | `CreditGrant` on successful `PaymentIntent` auto-creates within 5s of `payment_intent.succeeded` event | Verified by webhook lag measurement in test suite | If auto-grant fails, emit `credit_grant.failed` event; do not silently lose credits | Inject delay in grant creation service; verify `credit_grant.failed` fires and balance is not changed until manual re-trigger |
| 1.4 | Balance readable via GET endpoint at <50ms P95 | Perf test across 10K merchant accounts | No read lock during concurrent deductions; reads are eventually consistent within 200ms | Parallel read during high-deduction load; verify consistency window does not exceed 200ms |
| 1.5 | `seed_balance` on creation imports existing balance without payment | `CreditGrant` of type `migration` created with no `payment_intent_id`; balance set correctly | Seed balance must be > 0; negative seed rejected with 400 | Create CreditBalance with `seed_balance: 50000`; verify balance is 50,000 and grant type is `migration` |

### Req 2 - `AutoTopUp` flow

| ID | Requirement | Acceptance criteria | Edge case | Test scenario |
|---|---|---|---|---|
| 2.1 | Top-up triggered within 10s of balance crossing threshold | P95 trigger latency <10s measured from `credit_balance.below_threshold` event emission | Network partition: trigger fires from balance update event; idempotency key prevents duplicate top-up if event delivered twice | Simulate network partition during threshold crossing; verify single top-up payment intent created, not two |
| 2.2 | Smart Retries applied to failed top-up `PaymentIntent` | Retry behaviour matches Smart Retries on standard subscription invoices | After final retry failure, emit `auto_top_up.failed_final`; balance status set to `top_up_failed`; no further auto-retries without merchant action | Simulate card decline on all retries; verify exactly 3 attempts, then `failed_final` event, then no further attempts |
| 2.3 | Merchant can configure `on_insufficient_balance` as `block` or `tab` | Both modes work; `tab` allows negative balance up to configured limit; `block` returns `402` at zero | If `tab` limit is reached, switch to `block` behaviour automatically; emit `credit_balance.tab_limit_reached` | Set tab_limit=-10000; deduct until -10001; verify final deduction returns 402 and event fires |
| 2.4 | `AutoTopUp` idempotent: duplicate threshold-cross events do not trigger duplicate payments | Exactly one `PaymentIntent` created per threshold-crossing episode | Deductions that cross threshold multiple times in rapid succession trigger only one top-up | Send 10 rapid deductions all crossing threshold simultaneously; verify one PaymentIntent created |

### Req 3 - `PaymentDelegate` object

| ID | Requirement | Acceptance criteria | Edge case | Test scenario |
|---|---|---|---|---|
| 3.1 | Scope validation runs before `PaymentIntent` confirmation; out-of-scope returns `403` with violated scope field named | <50ms scope check; correct error body in 100% of test cases | If scope record is unavailable (service error), fail safe: reject the `PaymentIntent` with `503 Delegate Validation Unavailable` | Simulate scope service outage; verify PaymentIntent returns 503, not 200 or 403 |
| 3.2 | Revocation takes effect within 60s across all endpoints | Revoked delegate ID rejected by P99 within 60s globally | In-flight `PaymentIntent` in `requires_confirmation` at revocation time: cancel the intent; emit `payment_intent.cancelled` | Revoke delegate while PaymentIntent is in `requires_confirmation`; verify cancellation and event emission within 60s |
| 3.3 | User-visible audit trail: every `payment_delegate.used` event readable by the customer in hosted receipt UI | Audit trail shows agent name, amount, scope used, and timestamp | If agent application is deleted by merchant, replace agent name with "Deleted application" in audit trail; do not remove the record | Delete agent application; verify historical `payment_delegate.used` events still appear with "Deleted application" label |
| 3.4 | Spend cap enforced in real time; `PaymentIntent` creation with delegate rejected when `spent_to_date >= total_spend_cap` | `403 Spend Cap Reached` returned with `spent_to_date` and `total_spend_cap` in response body | Concurrent agent transactions that would collectively exceed cap: first to confirm wins; subsequent return 403 | Two simultaneous PaymentIntents that would each push spending over cap; verify only one succeeds |

---

## 12) Competitive analysis

| Competitor | How they address AI billing | Stripe's gap vs. this competitor | This PRD's response | Win condition |
|---|---|---|---|---|
| **Lago** (open-source metering) | Native credit ledger, real-time balance queries, usage-based invoicing; purpose-built for AI/API billing; free to self-host | Lago is free; Stripe currently requires DIY to match its feature set; Lago integrates with Stripe for payments, creating a two-vendor split | `CreditBalance` + `AutoTopUp` as first-party objects; hosted, no infra overhead; native Stripe integration removes two-vendor complexity | Stripe wins when Lago's self-host operational overhead (Redis, Postgres, upgrade management) exceeds the value of free licensing; typically at Series A when the team has a dedicated billing engineer |
| **Orb** | Subscription + usage billing platform with sub-second metering, plan version control, and credit grants | Orb's metering latency is lower than Stripe's current `Meter` object; purpose-built billing UI for finance teams; Orb prices at ~$500-2,000/month for mid-market | Real-time `CreditDeduction` at <100ms closes the latency gap; finance-team UI is not this PRD's scope; Stripe's price advantage at low volume | Stripe wins at seed to Series A; Orb wins at growth stage with complex multi-dimensional pricing and finance-team buyers |
| **Metronome** | Enterprise usage-based billing with multi-dimensional pricing, ERP integrations, and revenue recognition | Metronome serves large enterprise with complex pricing; higher-touch sales motion; estimated pricing at $2,000-10,000+/month | This PRD targets seed to Series B AI companies; different motion. Metronome is not the head-to-head competitor here | Not the relevant competitor for AI billing; Metronome wins on pricing complexity that this PRD does not address |
| **Recurly / Chargebee** | SaaS subscription billing; limited native usage-based billing | Neither has a real-time balance deduction primitive or agent-payment delegation | Not the relevant competitor for AI billing; legacy SaaS billing posture | N/A - different market |

### Specific benchmark comparisons

**Stripe vs. Lago for an AI startup at $500K ARR:**

| Dimension | Lago self-hosted | Stripe AI Billing Primitives |
|---|---|---|
| Integration time | 3-5 days (Lago + Stripe combo) | 1-2 days (Stripe-native) |
| Monthly cost | ~$0 software; ~$200-500 infra | Included in Stripe Billing (0.5% of billing volume) |
| Real-time balance check latency | <10ms (local Redis) | <100ms (Stripe API call) |
| Data reconciliation | Required (Lago + Stripe are separate) | Not required (single source of truth) |
| Dispute handling | Manual reconciliation between Lago and Stripe | Native: `charge.dispute.created` triggers credit review |
| Team required | 1 engineer to maintain | Zero - managed by Stripe |

**Win condition for Stripe:** At $500K ARR, a 1-2 day integration advantage and zero maintenance overhead outweigh Lago's latency advantage for most AI startups. Lago wins for companies that already have a billing engineer and need sub-10ms balance checks (high-frequency inference patterns).

**Stripe vs. Orb for an AI company at Series B ($10M ARR):**

| Dimension | Orb | Stripe AI Billing Primitives |
|---|---|---|
| Monthly cost (estimated) | $1,000-2,000 | ~$4,000 (0.5% of $800K billing volume) |
| Multi-dimensional pricing support | Yes (credits x plan tier x geography) | No (this PRD covers single-unit credits) |
| Finance team reporting UI | Yes - Orb has a full finance dashboard | No - merchant must use Sigma or build own |
| Churn/dunning automation | Limited | Full Smart Retries + Stripe dunning |
| Agent payment delegation | No | Yes (unique to Stripe) |

**Win condition for Stripe:** Stripe wins on the dunning and churn automation side (Smart Retries) and uniquely on `PaymentDelegate`. Orb wins on multi-dimensional pricing and the finance-team UI.

---

## 13) Success metrics

### North Star

**AI Billing Primitives Merchant Activation Rate** - percentage of AI-category Stripe merchants (identified by `Meter` usage or explicit AI category tag) who activate at least one of `CreditBalance`, `AutoTopUp`, or `PaymentDelegate` within 90 days of GA.

### Input metrics (leading indicators)

| Metric | Baseline (estimated) | 90-day target | Instrumentation event | Owner |
|---|---|---|---|---|
| `CreditBalance` objects created per week (AI-category merchants) | 0 (new product) | 500 | `credit_balance.created` count | Billing PM |
| `AutoTopUp` attach rate (of merchants with `CreditBalance`) | 0 | >70% | `auto_top_up.created` / `credit_balance.created` | Billing PM |
| Auto-refill success rate (first or second attempt) | 0 | >85% | `auto_top_up.succeeded` / `auto_top_up.triggered` | Payments infra |
| Delegated GPV ($) | 0 | $10M (closed beta) | `payment_delegate.used` sum of `amount` | Agentic PM |
| AI merchant Billing attach rate | ~45% (est.) | 65% | merchants with active `Subscription` or `Meter` / total AI-category merchants | Growth |
| AI billing integration support ticket volume | Baseline T-90 days | -40% | Zendesk AI-billing label count | Developer experience |
| Median time from first live charge to CreditBalance activation | N/A | <7 days | Time delta: first `charge.succeeded` to first `credit_balance.created` | Growth |

### Guardrail metrics

| Metric | Threshold | Risk if breached | Response |
|---|---|---|---|
| `CreditDeduction` P95 latency | <100ms | Inference products cannot tolerate >100ms billing overhead; merchants revert to DIY | Page on-call; activate degraded mode (serve stale balance with flag) |
| `AutoTopUp` trigger P95 latency | <10s from threshold crossing | Users experience service interruption between balance hitting zero and top-up restoring | Investigate event pipeline lag; manual trigger recovery for affected customers |
| `PaymentDelegate` revocation P99 propagation | <60s | Revoked agents can initiate payments in the gap; user trust failure | Immediate incident; notify affected customers; log all payments in revocation window for review |
| Credit overdraft incidents (balance below `minimum_balance`) | Zero in prod | Revenue leakage for merchant; Stripe liability if concurrent deduction bug causes overdraft | Immediate post-mortem; compensate affected merchant |
| `PaymentDelegate` revocation rate within 7 days of first use | <10% | High revocation = trust failure in agentic payment model | User research sprint; scope UI review |

---

## 14) Trade-offs

### Trade-off 1: Real-time balance at the cost of consistency window

Sub-100ms deduction latency requires an in-memory balance store with asynchronous persistence. This introduces a consistency window - if Stripe's in-memory layer fails between deduction and persistence, the deduction may be lost (credits consumed but not recorded) or double-counted. Idempotency keys on `CreditDeduction` calls mitigate double-counting. Lost deductions in failure scenarios are a revenue leakage risk.

**Decision:** Accept consistency window with idempotency key requirement. Require merchants to pass unique idempotency keys per inference call. Stripe's recovery path: if a deduction's idempotency key is replayed within 24h, return the original response without re-deducting.

### Trade-off 2: `PaymentDelegate` scope enforcement at Stripe vs. at the merchant

Stripe can enforce `PaymentDelegate` scope at the API layer (reject out-of-scope `PaymentIntent`s) or leave enforcement to the merchant and provide the delegate record as an audit primitive only. Enforcement at Stripe is more secure but reduces merchant flexibility. Enforcement at the merchant is weaker but allows merchants to define custom scope semantics.

**Decision for v1:** Enforce at Stripe for the defined scope fields (`max_amount_per_transaction`, `allowed_mccs`, `total_spend_cap`, `valid_until`). Merchants can add custom scope metadata that Stripe does not enforce - this is advisory metadata for the merchant's own application logic.

### Trade-off 3: `CreditBalance` currency (money vs. units)

Should `CreditBalance` be denominated in currency (cents) or in merchant-defined units (tokens, API calls, credits)? Currency denomination is simpler. Unit denomination is more flexible - AI companies want to bill per token, not per cent, to decouple pricing changes from billing infrastructure changes.

**Decision:** Unit denomination with merchant-defined `unit` string. Stripe stores the balance in units; the merchant controls the conversion rate between units and currency. This decouples pricing changes from billing infrastructure.

### Trade-off 4: Single `CreditBalance` per customer vs. multiple

Should a customer be allowed to have multiple `CreditBalance` objects (one per unit type)? Multiple balances enable more granular product pricing but add complexity to deduction routing.

**Decision for v1:** One active `CreditBalance` per customer per `unit` type. Merchants who need multi-dimensional balance tracking create two separate `CreditBalance` objects and specify the `credit_balance_id` explicitly on each `CreditDeduction` call. This avoids implicit routing ambiguity.

---

## 15) Experiment backlog (complete)

### Experiment 1: One-click `AutoTopUp` attachment prompt at first live deduction

**Hypothesis:** Showing an `AutoTopUp` configuration prompt immediately after a merchant's first live `CreditDeduction` event increases `AutoTopUp` attach rate by 20 pp within 30 days.

**Test design:**
- Control: Default behaviour - no prompt; merchant finds `AutoTopUp` in docs.
- Treatment: In-Dashboard banner + triggered email at first live deduction: "Set up automatic top-up so your users never run out of credits."

**Primary metric:** `AutoTopUp` attach rate within 30 days of first live `CreditDeduction`.
**Secondary metric:** Time-to-AutoTopUp-creation (median, in days).

**Power calculation:** At 200 new merchants per month, 100 per arm. To detect 20 pp lift from 50% baseline with 80% power and alpha=0.05: approximately 85 merchants per arm required. One month of data is sufficient.

**Rollout:** Phase 2 (limited GA). 50/50 A/B; all AI-category merchants activating `CreditDeduction` in the experiment window.
**Kill switch:** If `AutoTopUp` attach rate in treatment drops below control (prompt increases confusion), revert within 24h. Owner: Billing PM.
**Acceptance criteria:** p < 0.05; lift >= 15 pp; no statistically significant increase in support tickets about AutoTopUp setup.

---

### Experiment 2: Pre-built credit template in Stripe Dashboard for AI-category accounts

**Hypothesis:** Offering a "Credit billing quick-start" template in the Billing section reduces median time-to-first-live-deduction from 7 days to 3 days.

**Test design:**
- Control: Standard Billing documentation experience.
- Treatment: Prominent "Set up AI credit billing" card with a guided three-step flow: (1) create `CreditBalance`, (2) integrate `CreditDeduction` with provided code snippet, (3) configure `AutoTopUp`.

**Primary metric:** Median days from account creation to first live `CreditDeduction`.
**Secondary metric:** Drop-off rate at each step of the guided flow.

**Power calculation:** Log-rank test on the survival curve. At 200 merchants/month, 60 days of data provides sufficient power to detect a 4-day shift in median time-to-activation.

**Rollout:** Phase 1 (closed beta), then confirmed for Phase 2. New AI-category accounts only.
**Kill switch:** If template accounts show lower activation rate than control, revert immediately. Owner: Developer Experience PM.
**Acceptance criteria:** Median time-to-deduction reduced by >=3 days; p < 0.05; no increase in integration error rate.

---

### Experiment 3: `PaymentDelegate` user-facing revocation UI in Stripe Customer Portal

**Hypothesis:** Adding a "Manage AI agent permissions" section to the Stripe-hosted Customer Portal reduces `PaymentDelegate` revocation rate within 7 days from an estimated 15% to <10%.

**Rationale:** If users can easily see and manage delegate permissions in a trusted Stripe-hosted surface, they are less likely to panic-revoke because the permission feels controllable.

**Test design:**
- Control: Revocation available only via merchant's own UI.
- Treatment: Stripe Customer Portal shows "AI agents with payment access" with revoke button per delegate.

**Primary metric:** `PaymentDelegate` revocation rate within 7 days of first use.
**Secondary metric:** Time-from-first-use to revocation (distributions - bimodal signal for fast revoke = distrust vs. late revoke = normal expiry).

**Rollout:** Closed beta only. Requires merchant consent to enable Customer Portal integration for `PaymentDelegate` visibility.
**Kill switch:** If Customer Portal visibility causes unexpected spike in revocations, disable portal section. Owner: Agentic PM.
**Acceptance criteria:** Revocation rate within 7 days < 10%; p < 0.05; no increase in user-reported confusion tickets.

---

### Experiment 4: `CreditBalance` degraded mode UX - stale balance messaging

**Hypothesis:** Showing merchants a clear `balance_stale` flag in Dashboard and API responses (rather than silently returning potentially stale balances) reduces merchant escalations during balance service degradation events by 50%.

**Test design:**
- Control: Status page alert only during balance service degradation; no per-call flag.
- Treatment: `balance_stale: true` returned on every deduction response during degraded window; Dashboard shows "Balance data may be delayed" banner.

**Primary metric:** Support escalation volume during degradation incidents (per incident, compared to pre-flag baseline).
**Secondary metric:** Merchant self-service incident resolution rate (merchants who see the flag and do not escalate).

**Rollout:** Phase 1 (closed beta) to gather signal; global rollout in Phase 2 regardless of experiment result - `balance_stale` flag is a required API contract, not optional.
**Kill switch:** N/A - this is a required contract change. Owner: Infra.
**Acceptance criteria:** No increase in false-positive `balance_stale: true` responses during healthy service windows (rate < 0.1% of deductions).

---

## 16) Phased rollout plan

### Phase 1: Private closed beta (0-3 months post-code-complete)

**Scope:**
- `CreditBalance` and `AutoTopUp` available to 25 hand-selected AI-category merchants.
- `PaymentDelegate` available to 10 agentic platform builders (see design partner selection criteria below).
- No self-serve signup; all onboarding through Stripe account teams.

**Launch gates:**
- `CreditDeduction` P95 latency < 100ms under 10x expected beta load. Owner: Infra.
- Zero overdraft incidents in load testing with 5,000 concurrent deductions. Owner: Infra.
- SCA setup flow tested and documented for UK, DE, FR merchants; expected auth rate > 85% on AutoTopUp in those markets. Owner: Payments.
- Card network pre-clearance review completed for `PaymentDelegate` stored credential model. Owner: Legal/Payments.
- `balance_stale` flag implemented and tested in sandbox. Owner: API.

**Kill switches:**
- Any P95 latency breach above 150ms across 5 consecutive minutes: automatic rollback of deduction service to read-only mode; merchants served cached balance.
- Any single overdraft incident in production: suspend all `CreditDeduction` processing; incident bridge within 15 minutes.
- `PaymentDelegate` fraud signal (Radar score > 80 on >5% of delegated transactions): pause new `PaymentDelegate` creation; review with Radar team within 24h.

**Success exit criteria for Phase 2:**
- >= 20 of 25 beta merchants activate within 30 days.
- AutoTopUp first-attempt success rate >= 80% in beta cohort.
- Zero card network compliance flags on `PaymentDelegate` transactions.
- P95 latency sustained < 100ms throughout Phase 1.

---

### Phase 2: Limited GA (3-6 months post-code-complete)

**Scope:**
- `CreditBalance` and `AutoTopUp` open to all AI-category Stripe merchants via self-serve Dashboard.
- `PaymentDelegate` open to closed beta applications (waitlist-based).
- Experiments 1 and 2 begin.

**Launch gates:**
- Developer documentation complete for all four objects with copy-paste quickstart code. Owner: Developer Experience.
- Stripe-hosted Customer Portal integration for `PaymentDelegate` audit trail live. Owner: Customer Portal.
- SCA compliance path for European AutoTopUp documented and tested. Owner: Payments.
- Chargeback handling workflow (consumed credits dispute process) reviewed by Legal and documented in merchant support runbooks. Owner: Risk/Legal.

**Kill switches:**
- Support ticket volume for AI billing exceeds 2x baseline within 14 days of limited GA open: pause new merchant signups; investigate documentation gap.
- AutoTopUp success rate drops below 75% across all merchants: page on-call; review Smart Retries ML model performance on AI-category payment methods.

**Success exit criteria for Phase 3:**
- >= 65% AI-category merchants activate within 90 days of self-serve availability.
- Support ticket volume reduction >= 20% vs. pre-GA baseline (partial signal of 40% goal).
- Experiment 1 reaches statistical significance with >= 15 pp lift in AutoTopUp attach rate.

---

### Phase 3: Full GA (6-9 months post-code-complete)

**Scope:**
- All four objects (`CreditBalance`, `CreditDeduction`, `AutoTopUp`, `PaymentDelegate`) fully generally available with no waitlist.
- `PaymentDelegate` expanded to all merchant categories (not just closed beta agentic platforms).
- Experiment 3 begins on Customer Portal revocation UI.

**Launch gates:**
- `PaymentDelegate` card network compliance confirmed by Visa and Mastercard liaisons. Owner: Legal.
- Multi-currency `CreditBalance` scope decision confirmed (see resolved open question below; v1 is single-currency only; multi-currency is Phase 4 or separate workstream).
- Full GA pricing confirmed and communicated to existing beta merchants. Owner: Monetisation.

**Kill switches:**
- Any card network compliance objection: immediately suspend `PaymentDelegate` for affected card types; provide 30-day migration path for affected merchants.
- `PaymentDelegate` revocation rate exceeds 15% within 7 days of first use across full GA cohort: pause new `PaymentDelegate` creation; customer research sprint within 7 days.

---

## 17) Dependency map

| Dependency | Required for | Owner | Risk if late |
|---|---|---|---|
| In-memory balance service with sub-100ms P95 | Phase 1 launch gate | Infra | Critical - core SLO breach; entire product blocked |
| Optimistic locking on `CreditBalance` | Phase 1 launch gate | Infra | Critical - overdraft risk |
| Card network pre-clearance for `PaymentDelegate` stored credential model | Phase 1 exit criteria | Legal/Payments | `PaymentDelegate` must stay advisory-only if not cleared; delays full enforcement GA |
| SCA exemption path (MIT) for AutoTopUp in Europe | Phase 2 launch gate | Payments | European AutoTopUp auth rate drops; merchants in EU/UK cannot rely on threshold refill |
| Stripe Customer Portal update for delegate audit trail | Phase 2 launch gate | Customer Portal | `PaymentDelegate` closed beta merchants must build own audit UI; reduces trust signal |
| Radar integration for delegated transaction scoring | Phase 3 preferred | Fraud/Radar | `PaymentDelegate` fraud detection gap; manual review required for high-value delegated transactions |
| Chargeback workflow for partially consumed credit grants | Phase 2 launch gate | Risk/Legal | Dispute handling is manual; support escalation risk at scale |

---

## 18) Risk register

| Risk | Probability | Impact | Mitigation | Owner |
|---|---|---|---|---|
| Card network rules do not accommodate `PaymentDelegate` as a stored credential variant | Medium | High - blocks `PaymentDelegate` from GA | Pre-clearance review with Visa and Mastercard legal liaisons in Phase 1; design within stored credential framework before building; prepare fallback (advisory-only `PaymentDelegate` without Stripe-side enforcement) | Legal / Payments |
| SCA (PSD2) exemption rate for `AutoTopUp` in Europe lower than expected | Medium | Medium - European `AutoTopUp` success rate drops | Model as Merchant Initiated Transactions (MIT) after initial SCA-authenticated setup; document SCA setup flow clearly; test auth rates in UK/DE/FR early in closed beta | Payments |
| `CreditDeduction` latency exceeds 100ms under production load at scale | Low | High - merchants revert to DIY Redis ledger | Invest in in-memory balance layer before GA; load test to 10,000 concurrent deductions per second; warm standby for balance service | Infra |
| Merchants misuse `tab` mode and accumulate large negative balances they cannot collect | Medium | Medium - Stripe credit risk exposure | Cap `tab_limit` at a merchant-configurable maximum; require Stripe review for `tab_limit` above a threshold; default `tab_limit` to null | Risk |
| `PaymentDelegate` fraud: agent application uses delegate to extract funds beyond user intent | Low | High - regulatory and reputational risk | Scope enforcement at Stripe API layer; real-time Radar signals on delegated transactions; user notification email on each delegated payment | Fraud / Radar |
| Competitor (Lago v2 or Orb) ships native Stripe integration that removes the two-vendor complexity argument | Medium | Medium - reduces Stripe's native-integration advantage | Ship `CreditBalance` before Lago achieves broad distribution; prioritise GA timeline | Product |

---

## 19) Resolved open questions (all five from v2)

### Q1: Regulatory model for `PaymentDelegate` - resolved

**Question:** Card networks (Visa, Mastercard) have stored credential frameworks for recurring charges. Does agent-initiated payment fit stored credential rules, or does it require a new network-level primitive? What are the network rules on liability shift for agent-initiated transactions where the cardholder did not initiate the specific purchase?

**Resolution:** `PaymentDelegate` is implemented as a **Merchant-Initiated Transaction (MIT)** on top of Stripe's existing stored credential framework - not a new card network primitive. The stored credential consent is captured when the user grants the `PaymentDelegate` (similar to the cardholder consent step for recurring billing). Subsequent agent-initiated `PaymentIntent`s are flagged as MIT with the stored credential agreement ID.

**Liability shift:** For MIT transactions, the issuer (not Stripe or the merchant) bears liability for authorised fraud if the transaction passes the network's fraud filters. The risk Stripe assumes is that the cardholder disputes the delegation itself - i.e., claims they never authorised the delegate. Stripe's mitigation: the `PaymentDelegate` creation is a logged, cardholder-authenticated action (requires cardholder to be present at delegate creation time, the same as setting up a recurring payment). If the cardholder disputes the delegation, they are claiming the authentication event itself was fraudulent - this is a standard chargeback path, not an agentic-payment-specific liability question.

**Pre-clearance status:** Legal/Payments must confirm this interpretation with Visa and Mastercard legal liaisons before Phase 1 exit. If networks require explicit new network registration (unlikely but possible), fallback is advisory-only `PaymentDelegate` with enforcement at merchant layer for Phase 2, deferring Stripe-side enforcement to a subsequent network registration process.

---

### Q2: `CreditBalance` multi-currency - resolved

**Question:** If an AI company sells credit packs in USD and EUR, should `CreditBalance` support multiple balance buckets by currency, or should it be a single unit balance that the merchant converts?

**Resolution:** `CreditBalance` is **unit-denominated, not currency-denominated** in v1 and through full GA. Multi-currency credit balance support is deferred to a future workstream (estimated Phase 4 or separate product initiative).

**Rationale:** The core value proposition of `CreditBalance` is that the merchant does not need to track the unit-to-currency conversion inside Stripe - they define the unit (e.g., "tokens") and manage the pricing tier (tokens per dollar, tokens per euro) entirely outside Stripe. This is the correct architecture for AI companies because they change pricing tiers frequently without wanting to update billing infrastructure. If Stripe attempted to track the FX-denominated credit value, it would need to reconcile unit balances against live exchange rates, creating complexity that does not serve the AI billing use case.

**Multi-currency workaround for v1:** Merchants with EUR and USD credit packs create two separate `CreditBalance` objects: one with `unit: "usd_tokens"` and one with `unit: "eur_tokens"`. Each is funded by the respective currency `PaymentIntent`. The merchant's application routes deductions to the correct balance based on the user's account currency. This is explicit and works without Stripe needing to understand FX conversion.

**Signal to revisit multi-currency:** If >20% of AI-category `CreditBalance` merchants create two currency-keyed balance objects within 6 months of GA, that is the signal to prioritise native multi-currency support in the next product cycle.

---

### Q3: Chargeback handling for consumed credits - resolved

**Question:** If a user files a chargeback on a credit pack purchase after consuming 80% of the credits, what is the correct Stripe behaviour? Reversing the full payment while 80% of credits are consumed harms the merchant.

**Resolution:** Stripe does not automatically reverse credits on a chargeback - this is the merchant's responsibility, by design. The `charge.dispute.created` webhook triggers a mandatory merchant-action flow, not an automatic credit reversal.

**Dispute handling workflow (v1):**

```
charge.dispute.created event fires
    |
    v
Stripe sends email + Dashboard notification to merchant
    |
    v
Merchant reviews dispute and chooses one of three paths:
  Path A: Accept dispute -> Stripe refunds payment; merchant uses CreditDeduction webhook
          to reverse remaining balance only (consumed credits are not reversed).
  Path B: Counter dispute -> Merchant submits evidence including CreditDeduction history
          from Stripe (proof of service delivery) as primary evidence.
  Path C: Refund partial amount -> Merchant calculates unconsumed credit value, issues
          partial refund via Stripe Refund API; CreditBalance adjusted manually.
```

**Consumed credits evidence:** `CreditDeduction` records are available via the API for a minimum of 5 years. Merchants can export the full `CreditDeduction` history for a `Customer` as dispute evidence. The `balance_before`, `balance_after`, `metadata`, and timestamps per deduction constitute a tamper-evident consumption record.

**Stripe's position on partial chargebacks:** Credit card networks process chargebacks at the transaction level, not the partial consumption level. Merchants who win disputes (with consumption evidence) keep the full payment. Merchants who lose disputes lose the full payment regardless of consumption. The PRD does not change this network constraint. What this PRD adds is that the consumption evidence (all `CreditDeduction` records) is now a first-party Stripe data export rather than a manually assembled record from the merchant's own database.

**Operational addition (Phase 2):** The Stripe Dashboard dispute flow will include a "AI credits consumed" data export button when the disputed `PaymentIntent` is associated with a `CreditGrant`. This reduces the friction for merchants building their dispute evidence package.

---

### Q4: `AutoTopUp` and SCA (PSD2) - resolved

**Question:** Strong Customer Authentication in Europe requires explicit cardholder authentication for most payment initiation. Auto top-up is a merchant-initiated transaction without the cardholder present. What is the correct compliance path?

**Resolution:** `AutoTopUp` is implemented as a **Merchant Initiated Transaction (MIT)** under PSD2's recurring transaction exemption, with the initial top-up payment requiring explicit cardholder SCA authentication as the stored credential agreement step.

**Compliance path:**

1. **Initial cardholder-present step (required):** When a merchant configures `AutoTopUp` for a customer, the merchant must trigger a cardholder-present SCA authentication to establish the stored credential agreement. Stripe provides a hosted onboarding flow (similar to the SetupIntent flow for subscriptions) where the cardholder explicitly consents to future automatic top-ups and completes SCA authentication. This is the equivalent of the initial SCA step for recurring billing.

2. **Subsequent auto top-ups (MIT exemption):** After the stored credential agreement is established, subsequent `AutoTopUp`-triggered `PaymentIntent`s are flagged as MIT on the network. Under PSD2 Article 97, MITs initiated by the merchant based on prior explicit agreement are exempt from per-transaction SCA.

3. **Expected auth rate:** Based on Stripe's published Smart Retries data and MIT exemption rates for European subscriptions, expected auth rate for `AutoTopUp` in the UK, DE, and FR markets is 82-88% on first attempt. This is lower than card-present rates but consistent with the subscription billing category. The Smart Retries ML model will be trained on `AutoTopUp` outcomes in Phase 1 beta to improve the retry schedule for AI-category payment methods.

4. **Edge case - issuer declines MIT exemption:** Some European issuers (particularly smaller challenger banks) decline MIT exemptions and require cardholder re-authentication. If `AutoTopUp` fails with `authentication_required` decline code, Stripe sends a `auto_top_up.authentication_required` event. The merchant must surface a re-authentication flow to the cardholder. This is documented as a required merchant-side handler in the `AutoTopUp` integration guide.

---

### Q5: Design partner selection for `PaymentDelegate` closed beta - resolved

**Question:** Which 10 agentic platforms should be in the first closed beta cohort?

**Resolution:** Design partners are selected against five criteria, all of which must be met:

| Criterion | Rationale |
|---|---|
| Live agentic product transacting real user money today (not demo/pre-launch) | Ensures the partner can provide real transaction data and real failure mode feedback; avoids partners who theorise about use cases they haven't built |
| Existing Stripe merchant (processing >= $10K/month GPV) | Reduces integration complexity; partner already has production Stripe integration; easier to instrument beta objects |
| Clear, single primary use case for scoped agent payment | Multi-use-case partners produce noisy feedback; a travel booking agent, a SaaS procurement agent, and a B2B payment agent each have distinct scope and UX requirements - one partner per category is more valuable than one partner trying to cover all three |
| Willingness to commit to structured bi-weekly feedback sessions for 12 weeks | Closed beta feedback is worthless if partners are non-responsive; structured cadence is a contractual expectation, not a hope |
| Technical team with at least one engineer who can integrate at the API level independently | Avoids a tight dependency on Stripe's solutions engineering team for basic integration; partner must be able to move fast without hand-holding |

**Target design partner categories (10 partners):**

| Category | Example products | Primary use case for `PaymentDelegate` |
|---|---|---|
| AI travel agent (2 partners) | Layla, Mindtrip | Flight and hotel booking with per-trip scoped delegation; MCC: 7011, 4511 |
| AI procurement agent (2 partners) | Zip, Airbase or equivalent) | SaaS and tool subscription purchases; corporate card use case; MCC: 7372 |
| AI personal finance agent (2 partners) | Copilot, or LLM-native fintech) | Bill payment and subscription management on behalf of user; clear revocation UI requirements |
| AI developer tooling agent (2 partners) | Cursor, or LLM IDE with credits purchasing) | API credit purchases for third-party services initiated by the IDE agent; MCC: 7372 |
| AI e-commerce agent (2 partners) | Vercel or a platform with embedded purchasing) | Physical or digital goods purchasing on user's behalf; tests MCC allowlist enforcement |

**Recruitment process:** Stripe's developer relations and account management teams reach out to known agentic platform builders in the existing merchant base. Partners who meet all five criteria are offered closed beta access with a 90-day agreement covering feedback cadence and data usage for product improvement.

---

## 20) v3 Resolved open question from v1 (carried forward for completeness)

### Q6: `CreditBalance` vs. enhanced `Meter` object - resolved in v2

**Decision: Net-new `CreditBalance` object.** The `Meter` object is designed for accumulate-forward usage aggregation with asynchronous batch processing - it is explicitly not suited for synchronous real-time deduction. Forcing `CreditBalance` semantics into `Meter` would require either breaking `Meter`'s existing contract or adding conditional behaviour that makes both objects harder to reason about. Both objects can coexist on the same `Customer`; merchants using hybrid billing (subscription + credit overage) will use both.

---

*All metrics are directional estimates based on public information, teardown analysis, and industry benchmarks - not internal Stripe data.*
