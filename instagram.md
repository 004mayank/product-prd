# PRD: Improve Conversation Health in Instagram DMs (Reduce TTFR, Improve CCR)

**Product:** Instagram (DMs)  
**Author:** Mayank Malviya  
**Status:** v2 — prioritized requirements, UX surfaces, analytics schema, rollout gates  
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/instagram-dms-teardown.md

---

## Changelog
### v2 (this revision)
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

---

## 3) Users / personas
1. **Creators (10k–1M followers):** high volume of story replies + collab requests; needs fast triage.
2. **Small businesses / prosumers:** DMs are a sales/support channel; missed replies = lost revenue.
3. **Everyday users:** mostly social; still experience message requests + missed “important” pings.

**Primary target for V2 experiments:** Creators + SMBs (highest volume, highest cost of missed replies).  
**Secondary:** Everyday users (ensure UX doesn’t add anxiety/clutter).

---

## 4) Key insights
- The DM list is effectively a **work queue** for high-intent users; when it’s noisy, the queue collapses.
- Story replies are both **high-signal** (contextual) and **high-volume** (overload risk).
- Users can tolerate some “assistance UI,” but not pressure/anxiety; any prioritization must be subtle and dismissible.

**Operational definition (for V2):** “High-intent” = inbound messages that are (a) recent, (b) unreplied-to, and (c) higher likelihood of needing action based on *explicit* context (story reply, request inbox, mentions) + lightweight text heuristics (question mark / short request templates) — **not** deep semantic interpretation.

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
- **Notification disablement / opt-out** (if we introduce any nudges).
- **Wrong-priority rate**: user dismisses/deprioritizes surfaced items.
- **DM list “urgency saturation”**: % of threads marked Needs reply (cap to avoid everything looking urgent).

---

## 6) Scope & target user journeys

### Target journey A: inbound message → seen → replied
Trigger (inbound DM / story reply / request) → user opens IG → goes to DMs → finds the message → replies → continues thread

Key failure modes:
- User doesn’t see the message (notification/overload).
- User sees it but can’t tell what’s important.
- User intends to reply but loses the thread later (no “open loops” memory).
- Legitimate messages are buried under spam/requests.

### Target journey B: requests inbox → first action → converted to a reply (or safely removed)
Request arrives → user opens Requests → identifies legit vs spam → Accept/Delete/Report → if accepted, reply

---

## 7) Proposed solutions → requirements (v2)

### Solution 1: “Needs reply” state for DMs (lightweight prioritization)
**User promise:** “When you open DMs, it’s obvious which threads are waiting on you.”

#### P0 requirements
- **FR1 — Needs reply eligibility:** System must mark a thread as *Needs reply* when:
  - last message is inbound (from other user), and
  - no outbound message sent after that inbound, and
  - thread is within recency window (e.g., last inbound ≤ 7 days), and
  - passes conservative triggers (story reply context OR question mark OR common request templates), and
  - thread is not muted/archived/restricted/blocked.
- **FR2 — Visibility:** System must render a subtle *Needs reply* indicator in the DM list row.
- **FR3 — Dismiss/Mark handled:** System must allow user to remove the indicator per-thread (e.g., “Mark as handled”).
- **FR4 — Saturation protection:** System must cap the number (or %) of threads labeled *Needs reply* in the visible list (avoid “everything is urgent”).
- **FR5 — Persistence:** Dismissed state must persist across sessions for a reasonable duration (e.g., until new inbound arrives or 14 days).

#### P1 requirements
- **FR6 — Creator/SMB tuning:** System should allow different thresholds for high-volume users (creators/SMBs) vs everyday users.
- **FR7 — Explainability (light):** System may show a lightweight reason for the label in overflow (“Marked because: Story reply” / “Question”).

#### Acceptance criteria (samples)
- Given a thread with last inbound “?” within 7 days and no reply, when user opens DM list, then the thread shows *Needs reply*.
- Given a muted thread, when last message is inbound, then the thread is not labeled.
- Given user taps “Mark handled”, when user leaves and returns to DMs, then the label stays removed until new inbound arrives.

---

### Solution 2: Message Requests triage assist (reduce time-to-first-action)
**User promise:** “Requests are quicker to clear, and legit requests are easier to spot.”

#### P0 requirements
- **FR8 — Faster first action:** System must present clear one-tap actions in Requests: **Accept**, **Delete**, **Report/Block** (names per policy).
- **FR9 — Signal surfacing:** System must show a compact set of *explicit* trust/context signals where available (e.g., verified badge, mutual followers, shared groups, prior interactions).
- **FR10 — Likely spam bucket (no new ML claims):** System must support a separate “Likely spam” view based on existing classification/signals.
- **FR11 — Logging:** System must log which signals were shown and which action was taken (for learning + audit).

#### P1 requirements
- **FR12 — Bulk handling:** System should support multi-select + bulk delete/report for power users.

#### Acceptance criteria (samples)
- Given a request from a verified account with mutual followers, when user opens Requests, then signals render and Accept is available without extra taps.
- Given a request classified as spam, when user opens Requests, then it appears in Likely spam (not mixed into primary requests).

---

### Solution 3: “Open loops” resurfacing module (since you were away)
**User promise:** “After time away, DMs help you pick up where you left off.”

#### P0 requirements
- **FR13 — Entry condition:** System must show an *Open loops* module only when (a) there are true open loops and (b) user has been away long enough (e.g., last DM session > 6h) to justify the surface.
- **FR14 — Contents (explicit signals only):** Module must include:
  - threads marked Needs reply,
  - pending Message Requests not acted on,
  - mentions/replies-to-you in group contexts (where explicit).
- **FR15 — Dismiss:** Module must be dismissible, and dismissal must persist for the session.

#### P1 requirements
- **FR16 — Frequency caps:** Module should not appear more than once per day unless open loops exceed a threshold.

#### Acceptance criteria (samples)
- Given user has 3 Needs reply threads and last DM open was > 6h, when entering DMs, then Open loops module shows those threads.
- Given user dismisses module, when they go back to DM list within session, then module stays hidden.

---

## 8) UX surfaces (v2)

### A) DM list row
- Add a small, low-salience pill/label: “Needs reply” (copy TBD).
- Keep existing ordering; do **not** reorder the whole inbox in V2 (reduces risk).

### B) Thread overflow actions
- “Mark handled” (removes Needs reply)
- Optional: “Why am I seeing this?” (only if policy/UX approves)

### C) Requests inbox
- A compact “trust context” row (verified/mutuals/etc.)
- One-tap actions surfaced without scrolling
- Separate “Likely spam” tab/bucket

### D) DM entry module: Open loops
- Compact module at top, collapsible/dismissible
- 3–5 items max; link to “See all” if needed

---

## 9) Analytics & experimentation (v2)

### Event schema (minimal)
**All events should include:** user_id hash, surface (dm_list / requests / dm_entry), experiment_variant, timestamp.

- `dm_needs_reply_impression` (thread_id, eligibility_reason)
- `dm_needs_reply_mark_handled` (thread_id)
- `dm_thread_open` (thread_id, from_surface)
- `dm_reply_send` (thread_id, message_type)

- `dm_requests_view` (tab: requests|likely_spam)
- `dm_request_action` (request_id, action: accept|delete|report|block, signals_shown[])

- `dm_open_loops_impression` (count_threads, count_requests)
- `dm_open_loops_dismiss` (reason_optional)
- `dm_open_loops_click_item` (item_type: thread|request)

### Experiments
- **Experiment A — Needs reply marker**
  - Control: current DM list
  - Variant: needs-reply label + mark handled + caps
  - Success: ↓ median TTFR, ↑ CCR (24h)
  - Guardrails: wrong-priority rate, saturation, reports/blocks

- **Experiment B — Requests triage assist**
  - Success: ↓ time-to-first-action, ↑ acceptance-to-reply conversion (where measurable)
  - Guardrails: spam surfaced to Primary, increased spam engagement

- **Experiment C — Open loops module**
  - Success: ↑ open→reply conversion, ↑ DM return rate
  - Guardrails: dismiss rate, “anxiety signals” proxy (rapid dismiss + reduced session length)

---

## 10) Privacy, integrity, and safety (v2)
- **Spam amplification risk:** Prioritization could surface adversarial messages.
  - Mitigation: conservative eligibility; exclude restricted/low-trust; integrate existing spam classifications; monitor reports/blocks.
- **User anxiety / guilt risk:** “Needs reply” can create obligation.
  - Mitigation: subtle UI; user control (mark handled); caps; avoid pushy notifications in V2.
- **Data minimization:** Prefer explicit signals and lightweight heuristics; avoid storing sensitive derived intent.
- **Teen safety / sensitive accounts:** Ensure restricted accounts and safety states exclude prioritization surfaces by default.

---

## 11) Rollout plan (v2)
1. **Needs reply (P0) behind feature gate**
   - Start: employees/dogfood → 1% creators/SMBs → 5% → 25%.
   - Ship only after guardrails stable (spam/blocks not up, saturation controlled).
2. **Requests triage assist (P0) parallel gate**
   - Launch to high-volume DM users first.
   - Validate time-to-first-action improvements.
3. **Open loops module**
   - Add only after Needs reply proves it doesn’t increase anxiety/clutter.

---

## 12) Risks & trade-offs
- **Risk:** Needs reply increases anxiety/guilt.
  - Mitigation: subtle labeling, dismiss/mark handled, caps, no push notifications in V2.
- **Risk:** Prioritization surfaces spam.
  - Mitigation: conservative heuristics; exclude low-trust; use existing classification; guardrails.
- **Risk:** Extra UI modules add clutter.
  - Mitigation: strict entry conditions; dismiss; frequency caps.

---

## 13) Open questions
- What’s the best user-facing copy for “Needs reply” that feels helpful, not judgmental?
- Should Needs reply behave differently for creators (high volume) vs everyday users?
- Which explicit signals best identify “high-intent” in a policy-safe way (story reply context, verified, mutuals, etc.)?
- What are the correct exclusion lists: muted, restricted, archived, “sensitive” states?

---

## 14) Summary
This PRD targets Instagram DM retention and utility by improving **reply speed and conversation continuation** through lightweight prioritization (*Needs reply*), faster Message Requests triage, and resurfacing of open loops across sessions—without turning DMs into a heavy workflow tool.
