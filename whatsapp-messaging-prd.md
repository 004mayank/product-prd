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
**Status:** v1 — problem framing, metrics, and execution-ready proposals
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/whatsapp-messaging-teardown.md

---

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

---

### Solution 3: “Since you were away” micro-summary for high-traffic groups
**What:** When opening a group with high unread count, show a compact “Since you were away” row with:
- number of mentions
- number of replies to you
- (optional) 1–2 pinned “key messages” based on explicit signals (mentions/replies), not content understanding.

**Why:** Reduces re-entry cost and makes it easier to respond.

**Constraints:** Avoid content interpretation claims; stick to observable explicit signals.

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

---

## 11) Open questions
- What’s the best user-facing language for “Needs reply” without inducing guilt?
- How should the system behave for muted chats/groups?
- Should “mark handled” be per-chat only, or a global “clear all” action?

---

## 12) Summary
This PRD targets WhatsApp’s core retention engine by improving **reply speed and conversation continuation** with lightweight prioritization cues, intent-based notification permissioning, and re-entry assistance for noisy groups—without changing WhatsApp into a feed.
