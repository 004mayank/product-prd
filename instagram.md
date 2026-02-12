# PRD: Improve Conversation Health in Instagram DMs (Reduce TTFR, Improve CCR)

**Product:** Instagram (DMs)  
**Author:** Mayank Malviya  
**Status:** v3 — shipped-ready spec (final thresholds, copy, dependencies, go/no-go)  
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/instagram-dms-teardown.md

---

## Changelog
### v3 (this revision)
- Finalized **eligibility logic** (scoring + caps) for *Needs reply* and *Open loops*.
- Added **user-facing copy options**, recommended default strings, and “tone” constraints.
- Added **go/no-go launch criteria**, monitoring, and stop conditions.
- Added **dependencies** + implementation notes (client vs server, storage, flags).
- Resolved key open questions with recommendations.

### v2
- Converted “proposed solutions” into **P0/P1 requirements** with **acceptance criteria**.
- Added a concrete **information architecture** for: *Needs reply* state, *Requests triage*, *Open loops* module.
- Added **analytics/event schema**, experiment design details, and guardrail definitions.
- Added **privacy/integrity considerations** and explicit mitigations (spam amplification, anxiety/clutter).
- Tightened rollout plan with **feature gates** and sequencing.

### v1
- Problem framing, metrics, and execution-ready proposals.

---

## 1) Problem statement
Instagram DMs increasingly function as a **high-intent communication surface** for creators, consumers, and small businesses: responding to story replies, handling “where can I buy this?” questions, coordinating collaborations, and closing transactions.

The retention engine is still the **reply loop**:
1) someone messages you (often via story reply),
2) you see it,
3) you reply,
4) the thread continues → relationship/transaction progresses.

Today, many conversations that *should* get quick replies don’t, because:
- **Inbox overload** (story replies + mentions + group chats + message requests) makes “what needs attention” unclear.
- **Message Requests** and spam create a triage tax; legitimate requests get buried.
- The product doesn’t consistently help users return to “open loops” (people waiting on me) across sessions.

Observable PM problem: **high-intent inbound messages are missed or replied to late**, reducing conversation continuation and downstream outcomes (sales, collabs, satisfaction).

---

## 2) Goals & non-goals

### Goals
1. **Reduce Time-to-First-Reply (TTFR)** for inbound DMs that are likely high-intent.
2. Increase **Conversation Continuation Rate (CCR)** (reply within 24h) for DM starts.
3. Improve “creator/SMB activation quality” in DMs:
   - **A2:** first meaningful reply within 24h of first high-intent inbound
   - **A3:** repeat DM handling behavior in week 1 (e.g., replies on ≥3 days/7)

### Non-goals
- Turning DMs into a full email-like inbox for all users (labels/rules/workflows at parity with email).
- Building heavy content understanding that requires claiming internal ML capabilities.
- Solving spam end-to-end; this PRD focuses on **triage + reply completion**, not adversarial abuse systems.
- Reordering inbox globally (ranking/priority inbox) in V3; we stick to **light indicators + modules**.

---

## 3) Users / personas
1. **Creators (10k–1M followers):** high volume of story replies + collab requests; needs fast triage.
2. **Small businesses / prosumers:** DMs are a sales/support channel; missed replies = lost revenue.
3. **Everyday users:** mostly social; still experience message requests + missed “important” pings.

**Primary target for launch:** Creators + SMBs (highest volume, highest cost of missed replies).  
**Secondary:** Everyday users (ensure UX doesn’t add anxiety/clutter).

---

## 4) Key insights
- The DM list is effectively a **work queue** for high-intent users; when it’s noisy, the queue collapses.
- Story replies are both **high-signal** (contextual) and **high-volume** (overload risk).
- Users can tolerate assistance, but not pressure; prioritization must be subtle, optional, and reversible.

**Operational definition:** “High-intent” = inbound messages that are (a) recent, (b) unreplied-to, and (c) higher likelihood of needing action based on *explicit* context + lightweight heuristics — **not** deep semantic interpretation.

---

## 5) Success metrics (definitions)

### Primary
- **TTFR (DM starts):** median and p90 time from inbound DM start → first outbound reply.
- **CCR (24h):** % of inbound DM starts that receive a reply within 24 hours.

### Secondary
- **DM return rate:** % of users who return to DMs within 24h after receiving a high-intent message.
- **Open→Reply conversion:** session-level conversion from opening DMs to sending ≥1 reply.
- **Request acceptance efficiency:** time-to-first-action on Message Requests (accept/decline/report).

### Guardrails
- **Spam reports / blocks** (ensure we don’t amplify spam).
- **Mute/leave** rate for DM groups.
- **Notification disablement / opt-out** (we do not introduce new push in V3; still monitor).
- **Wrong-priority rate:** user marks handled/dismisses surfaced items without replying.
- **Urgency saturation:** share of visible threads labeled *Needs reply*.

---

## 6) Scope & target user journeys

### Journey A: inbound message → seen → replied
Inbound DM/story reply/request → user opens IG → goes to DMs → finds the message → replies → continues

Failure modes:
- User doesn’t see it (notification/overload).
- User sees it but can’t tell what matters.
- User intends to reply but loses it across sessions.
- Legit messages buried under spam/requests.

### Journey B: requests inbox → first action → converted to reply (or safely removed)
Request arrives → user opens Requests → identifies legit vs spam → Accept/Delete/Report → if accepted, reply

---

## 7) Solutions → requirements (v3)

### Solution 1: “Needs reply” state for DMs (lightweight prioritization)
**User promise:** “When you open DMs, it’s obvious which threads are waiting on you.”

#### P0 requirements
- **FR1 — Eligibility (rules + scoring):** System must mark a thread as *Needs reply* if it is **eligible** and passes a conservative **score threshold**.
  - **Hard eligibility gates (must all be true):**
    - last message inbound (other user)
    - no outbound message after that inbound
    - last inbound age ≤ **7 days** (default; tunable)
    - thread not muted/archived
    - counterparty not blocked
    - thread not in restricted/sensitive states (per policy)
  - **Score model (additive, no semantics):**
    - +3: inbound is a **Story reply** (explicit context)
    - +2: message is in **Primary** inbox (not Requests)
    - +2: contains `?` (question mark)
    - +2: sender is **verified**
    - +1: **mutual follow** / strong graph signal available
    - +1: inbound age ≤ 24h
  - **Threshold:** mark as Needs reply when score ≥ **4**.
- **FR2 — Visibility:** Must render a low-salience row indicator in DM list.
- **FR3 — Mark handled:** Must allow user to remove indicator per thread.
- **FR4 — Saturation cap:** Must cap Needs reply to avoid “everything urgent”.
  - Default: show at most **min(5, 10% of first screen threads)**.
- **FR5 — Persistence:** Mark-handled state persists until (a) new inbound arrives or (b) 14 days.

#### P1 requirements
- **FR6 — Segment tuning:** Support different caps/thresholds for high-volume users.
  - Creators/SMBs default: same threshold (≥4) but cap can be higher (e.g., max 7) if saturation remains low.
- **FR7 — Explainability (light):** Optional “Why this?” with *explicit* reason (Story reply / Question / Verified).

#### Acceptance criteria (samples)
- Given a story reply received 2h ago and unreplied, when opening DM list, then it is labeled Needs reply.
- Given a thread with inbound older than 7 days, it is not labeled.
- Given user marks handled, the label disappears and stays off until new inbound.

---

### Solution 2: Message Requests triage assist
**User promise:** “Requests are quicker to clear, and legit requests are easier to spot.”

#### P0 requirements
- **FR8 — One-tap actions:** Requests must expose Accept, Delete, Report/Block with minimal friction.
- **FR9 — Trust/context signals:** Must show explicit signals (verified, mutuals, shared groups, prior interaction) when available.
- **FR10 — Likely spam bucket:** Must support separate “Likely spam” view using existing classification.
- **FR11 — Logging:** Must log action taken + which signals were displayed.

#### P1 requirements
- **FR12 — Bulk handling:** Multi-select + bulk delete/report for power users.

---

### Solution 3: “Open loops” resurfacing module
**User promise:** “After time away, DMs help you pick up where you left off.”

#### P0 requirements
- **FR13 — Show only when justified:** Open loops module appears only if:
  - last DM session > **6h**, and
  - open loops count ≥ **2** (or ≥1 for creator/SMB), and
  - not shown already in last **24h** (frequency cap).
- **FR14 — Contents (explicit only):** Needs reply threads + pending Requests + explicit mentions in group contexts.
- **FR15 — Dismiss:** Dismiss hides for session; do not re-show until next day unless open loops spike (e.g., +5).

---

## 8) UX spec (v3)

### A) DM list row
- Indicator: small pill.
- No reordering in V3.

**Recommended default copy:**
- Pill: **“Reply”** (preferred) or “Needs reply” (fallback).
  - Rationale: “Reply” reads as an action, not judgment.

### B) Thread overflow actions
- Primary: **Mark handled**
- Optional: “Why this?” → sheet with 1 reason string.

### C) Requests inbox
- Signals row: Verified / Mutuals / Shared followers / Prior interaction.
- Buttons: Accept, Delete, Report.
- Tabs: Requests | Likely spam.

### D) Open loops module (DM entry)
- Title copy options (choose 1):
  - **“To reply”** (recommended)
  - “Waiting on you” (risk: guilt)
  - “Pick up where you left off” (longer)
- Body: list 3–5 items max + “See all”.

**Tone constraints (must):**
- No guilt language (“You haven’t replied…”, “Don’t forget…”) in V3.
- No red badges/counts for this feature in V3.

---

## 9) Analytics & experimentation (v3)

### Event schema (minimal)
All events include: user_id hash, surface, experiment_variant, timestamp.

- `dm_needs_reply_impression` (thread_id, reason: story_reply|question|verified|graph)
- `dm_needs_reply_mark_handled` (thread_id)
- `dm_thread_open` (thread_id, from_surface)
- `dm_reply_send` (thread_id)

- `dm_requests_view` (tab)
- `dm_request_action` (request_id, action, signals_shown[])

- `dm_open_loops_impression` (count_threads, count_requests)
- `dm_open_loops_dismiss`
- `dm_open_loops_click_item` (item_type)

### Experiment design
- Randomization unit: user.
- Ramp: 0% → employee/dogfood → 1% → 5% → 25% → 50% → 100%.
- Target cohort first: creators/SMBs.

### Go/no-go criteria (measured vs control)
Ship from 25% → 50% only if all true for 7 days:
- **TTFR median:** improves by **≥2%** OR **CCR (24h)** improves by **≥1%**.
- **Guardrails:** reports/blocks do not increase by > **0.5% relative**.
- **Urgency saturation:** ≤ **10%** of first-screen threads labeled.
- **Wrong-priority:** mark-handled without reply ≤ **40%** of needs-reply interactions (baseline to be established at 1–5%).

Stop / roll back if:
- Reports/blocks increase by **>2% relative** sustained 48h, or
- Saturation > **20%** on first screen, or
- Crash-free sessions regress.

---

## 10) Privacy, integrity, and safety
- **Spam amplification:** conservative eligibility; exclude restricted/low-trust; rely on existing classification.
- **Anxiety risk:** subtle UI, dismiss/mark handled, strict caps, no new push.
- **Data minimization:** avoid storing derived “intent”; store only state needed for dismiss persistence.

---

## 11) Dependencies & implementation notes (v3)
- **Client (iOS/Android):** UI pill rendering, module surface, overflow actions.
- **Server or local state:**
  - Eligibility can start client-side (heuristics) if needed; preferred: server-provided flags per thread for consistency.
  - Storage for `mark_handled` per thread (server) or local with sync best-effort.
- **Requests:** depends on existing request classification + ability to present “Likely spam” bucket.
- **Feature flags:**
  - `dm_needs_reply_enabled`
  - `dm_requests_triage_enabled`
  - `dm_open_loops_enabled`

---

## 12) Rollout plan
1. **Needs reply** (P0) → prove TTFR/CCR lift with stable guardrails.
2. **Requests triage assist** (P0) → optimize time-to-first-action.
3. **Open loops** → add only after Needs reply shows low anxiety + low clutter.

---

## 13) Open questions (resolved in v3)
- **Copy for Needs reply:** Recommend **“Reply”** pill + “To reply” module title; avoid guilt.
- **Creator vs everyday behavior:** Keep same threshold; adjust cap (creators can see a few more) after saturation monitoring.
- **Policy-safe signals:** Prefer explicit context (story reply, verified, mutuals) + `?`; avoid semantic intent classification.
- **Exclusions:** Default exclude muted, archived, restricted, blocked; also exclude threads in safety-sensitive states per policy.

---

## 14) Summary
This PRD improves Instagram DM conversation health by making **unreplied, high-likelihood threads** easier to spot and finish (Needs reply), reducing Requests triage friction, and resurfacing open loops after time away—while protecting against spam amplification and anxiety.
