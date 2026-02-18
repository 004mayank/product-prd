<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/whatsapp.png" 
    alt="WhatsApp logo" 
    width="120"
  />
</p>

# PRD: Improve Conversation Health in WhatsApp (Reduce TTFR, Improve CCR)

**Product:** WhatsApp (Messaging)
**Author:** Mayank Malviya
**Status:** v3 — locked heuristics + instrumentation for Needs reply, intent-based prompts, and group summaries with launch gates
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/whatsapp-messaging-teardown.md

---

## Version history
- **v3 (current):** locked heuristics/state machine + instrumentation for all three solutions, plus launch gates + guardrails.
- **v2:** clarified “Needs reply” spec (eligibility + dismissal), edge cases, and an event schema for safe measurement.
- **v1:** problem framing, metrics, and initial execution-ready proposals.

## 1) Problem statement
WhatsApp’s core retention engine is the **reply loop**: a user sends a message, receives a reply, and the thread continues. When replies are slow or absent, conversations stall, users lose confidence that WhatsApp is the fastest path from intent → outcome, and they shift to other surfaces (calls, other apps) or disengage.

The practical PM problem (observable from user experience): **messages that should get quick replies often don’t**, due to missed notifications, unclear “what needs my attention,” and high noise in busy chat lists.

---

## 2) Goals & non-goals

### Goals
1. **Reduce Time-to-First-Reply (TTFR)** for conversation starts.
2. Increase **Conversation Continuation Rate (CCR)** (reply within 24h).
3. Improve early “meaningful activation” quality:
   - **A2:** first inbound reply within 24h of first outbound message
   - **A3:** stickiness in week 1 (≥3 active days/7 or joins/creates a group)

### Non-goals
- Changing WhatsApp’s identity into a feed/social network.
- Building full email-like inbox features (labels, rules) for everyone.
- Making claims or designs that rely on internal Meta data access.

---

## 3) Users / personas (public-behavior lens)
1. **1:1 coordinators**: care about speed + reliability; missed replies cause anxiety.
2. **Group coordinators**: noise and unread overload are the main enemy.
3. **Low-bandwidth / low-typing users**: depend heavily on notifications and voice notes.

---

## 4) Key insights (from teardown + PM heuristics)
- WhatsApp’s home surface is the **recent chats list**; it acts like a lightweight “reply to-do list.”
- The biggest drivers of fast replies are often **notification delivery + relevance**, not new composition features.
- Busy users don’t need more messages; they need clearer prioritization of **“who is waiting on me.”**

---

## 5) Success metrics (definitions)

### Primary
- **TTFR:** median and p90 time from first outbound message in a conversation start to first inbound reply.
- **CCR (24h):** % of conversation starts that receive a reply within 24 hours.

### Secondary
- **A2 activation rate** (new users): first reply within 24h of first outbound.
- **A3 activation rate** (new users): ≥3 active days/7 OR group join/create within 7 days.
- **Open→Send conversion** for reactivation flows.

### Guardrails
- **Notification opt-out rate** / notification disablement.
- **Spam reports / blocks** (ensure we don’t cause nudges that feel spammy).
- **Mute rate / leave rate** in groups (ensure “attention” features don’t amplify noise).

---

## 6) Scope & user journey (where changes apply)

### Target journey: conversation start → reply
Trigger → user opens WhatsApp → sends message → (recipient sees) → recipient replies → thread continues

Key failure modes to address:
- Recipient never sees the message (notification issues, permission off, notification overload)
- Recipient sees it but doesn’t feel urgency / forgets
- Sender loses confidence and abandons or escalates elsewhere

---

## 7) Proposed solutions (v1)

### Solution 1: “Reply-needed” highlight state (lightweight inbox cue)
**What:** In the chat list, highlight chats where the last message is from the other party and is a question/mention/reply-to-you pattern (heuristic, not ML-heavy), labeled subtly as **“Needs reply”**.

**Why:** Converts a flat list into a minimal prioritization layer without turning into a feed.

**Requirements (MVP):**
- On-device heuristics (question mark, direct reply, mention in group, recent incoming after your last outgoing)
- Dismissal affordance (“mark as handled”) to prevent lingering anxiety

**Expected impact:** Higher CCR and lower TTFR for busy users.

#### V3 spec: Eligibility + state machine
- **Eligibility window:** last inbound message < 12h, thread not muted/archived, user’s last outbound older than latest inbound, and unread count < 50 (avoid stale piles).
- **Signals considered:** question or question mark near end, direct mention in group, message replied-to you, or inbound voice note flagged `reply_to_you`.
- **Exclusions:** business accounts with automation, disappearing chats, muted/archived/blocked contacts, and threads already marked handled.
- **State transitions:** `eligible` → `surfaced` → (`handled` after user replies/marks handled) or `expired` at 24h. `dismissed` state suppresses resurfacing until a new outbound is sent or 48h passes.

#### Instrumentation & controls
- Events: `needs_reply_impression`, `needs_reply_dismiss`, `needs_reply_mark_handled`, `needs_reply_reengaged` (reply sent within 10m).
- Persist dismiss/handled flags in a lightweight per-thread store referenced by both mobile + web; wipe on thread delete.
- Soft cap to 5 concurrent highlights; rotate by recency and drop oldest when new ones arrive.
- Safety levers: auto-disable if user dismisses >5 in 24h or toggles `Don’t show Needs reply`.

---

### Solution 2: Smarter notification timing prompts (permissioning at intent)
**What:** Prompt users to enable notifications **only when** they demonstrate intent that depends on timely replies:
- after sending first message to a new contact
- after joining a group for the first time

**Why:** Users are more likely to consent when the benefit is immediate.

**Requirements:**
- Detect first-send / first-join milestones
- Copy that is benefit-first (“Don’t miss replies”) not platform-first

**Expected impact:** Improved A2 (new users) and reduced missed replies.

#### V3 spec: Triggering & guardrails
- **Triggers:** (a) user sends first message to a contact after installing; (b) user joins first group; (c) user re-enables notifications after >30d gap.
- **Frequency caps:** max 2 prompts per user per 7d, never twice in the same session, and suppress when user is already opted in.
- **Copy variants:** `“Don’t miss replies — turn on notifications?”` (primary) or softer `“Stay updated when <contact> replies”`; localized + A/B tested.
- **Dismissal logic:** `Not now` snoozes for 30d; `Never ask again` toggles a pref in account settings.

#### Instrumentation & controls
- Events: `intent_prompt_surface`, `intent_prompt_accept`, `intent_prompt_decline`, `intent_prompt_autosuppress` (cap reached).
- Log trigger type, thread/group id hash, and prior notification state for each event.
- Tie acceptance to notification enablement event to ensure we don’t double-count OS dialogs.
- Guardrail: auto-disable per user if decline rate >80% over 5 prompts or if OS reports “too many prompts” (iOS 17).

---

### Solution 3: “Since you were away” micro-summary for high-traffic groups
**What:** When opening a group with high unread count, show a compact “Since you were away” row with:
- number of mentions
- number of replies to you
- (optional) 1–2 pinned “key messages” based on explicit signals (mentions/replies), not content understanding.

**Why:** Reduces re-entry cost and makes it easier to respond.

**Constraints:** Avoid content interpretation claims; stick to observable explicit signals.

#### V3 spec: Eligibility & payload
- **Eligibility:** groups with >30 members or >40 unread messages, and at least one mention/reply-to-you in the last 12h. Exclude muted, archived, and community announcement-only groups.
- **Payload:** top metrics row (mentions, replies to you, total new messages) + up to two pinned snippets chosen via explicit signals (mentions to you first, then replies to your messages).
- **Freshness:** summarize last 12h window; if user re-opens within 30m show a lighter “You’re caught up” state.
- **Dismissal:** swipe away hides summary for that group for 24h; `Never show for this group` stored in per-group settings.

#### Instrumentation & controls
- Events: `group_summary_surface`, `group_summary_expand`, `group_summary_dismiss`, `group_summary_action` (user replies or taps mention).
- Capture payload size (counts) + generation latency to ensure <150 ms added to open.
- Guardrail auto-disable if summary latency >300 ms p95 or if mute/leave rate increases >5% vs control.

---

## 8) Experiment plan

### Experiment A: Chat list “Needs reply”
- **Control:** current list
- **Variant:** needs-reply label + dismissal
- **Measure:** TTFR, CCR; guardrail notification disablement

### Experiment B: Intent-based notification prompt
- **Control:** existing notification permission flow
- **Variant:** prompt after first-send / first-join
- **Measure:** notification opt-in, A2 lift, TTFR; guardrail opt-out/disablement

### Experiment C: Group re-entry micro-summary
- **Control:** open group normally
- **Variant:** micro-summary row
- **Measure:** reply rate after re-entry, time-to-first-action in group; guardrail mute/leave

---

## 9) Risks & trade-offs
- **Risk:** “Needs reply” increases pressure/anxiety.
  - *Mitigation:* provide dismiss/mark-handled; keep it subtle; allow easy control.
- **Risk:** Notification prompts feel spammy.
  - *Mitigation:* strict frequency caps; trigger only at high-intent moments.
- **Risk:** Group summaries feel like a feed.
  - *Mitigation:* only explicit signals (mentions/replies), no content ranking.

---

## 10) Rollout plan
1. Build MVP for Solution 1 + Experiment A (lowest dependency, highest leverage).
2. 1% → 10% → 50% gradual rollout, monitor TTFR/CCR + guardrails.
3. Add Solution 2 as a parallel track (Experiment B) focused on new users.
4. Ship Solution 3 after validating that it reduces re-entry friction without increasing mute/leave.

## 11) Instrumentation & launch gates (V3)

### Telemetry coverage
- `needs_reply_*`, `intent_prompt_*`, and `group_summary_*` events land in the same analytics table with `user_id`, `thread_id_hash`, `surface`, and `experiment arm` for easy joins.
- Daily completeness alert if ingestion <99% of experiment exposure to avoid blind spots.
- TTFR/CCR computed via existing conversation-start dataset but now annotated with whether a Needs reply cue fired.

### Go / no-go criteria
- **Phase 0 (dogfood):** require <200 ms added latency per surface and <5% dismiss/complaint rate before expanding.
- **Phase 1 (1% holdout):** ship if TTFR improves ≥3% and guardrails (mute/leave, notification opt-outs) move <1% vs control.
- **Phase 2 (10–50% ramp):** continue only if CCR improves ≥2% absolute for exposed conversations and spam/block reports do not increase.

### Operational guardrails
- Auto-kill switch wired to experimentation framework; flip off if latency, crash, or guardrail metrics breach thresholds.
- Weekly review: ratio of prompts accepted + subsequent OS notification success vs declines to ensure prompts still valuable.
- Documented on-call playbook linking states → remediation (e.g., clear handled KV if corruption detected).

---

## 12) Open questions
- What’s the best user-facing language for “Needs reply” without inducing guilt?
- How should the system behave for muted chats/groups?
- Should “mark handled” be per-chat only, or a global “clear all” action?

---

## 13) Summary
This PRD targets WhatsApp’s core retention engine by improving **reply speed and conversation continuation** with lightweight prioritization cues, intent-based notification permissioning, and re-entry assistance for noisy groups—without changing WhatsApp into a feed.
