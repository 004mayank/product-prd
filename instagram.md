# PRD: Improve Conversation Health in Instagram DMs (Reduce TTFR, Improve CCR)

**Product:** Instagram (DMs)
**Author:** Mayank Malviya
**Status:** v1 — problem framing, metrics, and execution-ready proposals
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/instagram-dms-teardown.md

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

---

## 4) Key insights
- The DM list is effectively a **work queue** for high-intent users; when it’s noisy, the queue collapses.
- Story replies are both **high-signal** (contextual) and **high-volume** (overload risk).
- Users can tolerate some “assistance UI,” but not pressure/anxiety; any prioritization must be subtle and dismissible.

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
- **Notification disablement / opt-out**.
- **Wrong-priority rate** (user dismisses/deprioritizes surfaced items).

---

## 6) Scope & user journey

### Target journey: inbound message → seen → replied
Trigger (inbound DM / story reply / request) → user opens IG → goes to DMs → finds the message → replies → continues thread

Key failure modes:
- User doesn’t see the message (notification/overload).
- User sees it but can’t tell what’s important.
- User intends to reply but loses the thread later (no “open loops” memory).
- Legitimate messages are buried under spam/requests.

---

## 7) Proposed solutions (v1)

### Solution 1: “Needs reply” state for DMs (lightweight prioritization)
**What:** Add a subtle, dismissible **Needs reply** marker in the DM list when:
- the last message is inbound,
- the thread is recent,
- and the pattern suggests a question/request (simple heuristics: question mark, short request templates, reply-to-story context, etc.).

**Why:** Converts a flat inbox into a minimal “reply to-do list” without introducing heavy workflow.

**MVP requirements:**
- On-device / rule-based heuristics (start with conservative triggers).
- **Dismiss/Mark handled** per thread.
- Frequency caps to prevent the whole inbox becoming “urgent.”

**Expected impact:** Lower TTFR and higher CCR for high-volume users.

---

### Solution 2: Message Requests triage assist (reduce time-to-first-action)
**What:** In Message Requests, provide a compact triage row that helps users take the first action faster:
- Show **top reasons** a request is surfaced (mutuals, shared followers, verified, prior interactions) when available.
- Add **one-tap actions**: Accept, Delete, Report.
- Add a **“Likely spam”** *bucket* based on existing signals (do not claim new ML; can be rules + current classification).

**Why:** Legitimate requests getting stuck is a major “open loop” failure mode.

**MVP requirements:**
- Clear separation of Request inbox vs Primary.
- Action logging to learn which signals correlate with acceptance.

**Expected impact:** Faster clearing of requests; more legitimate requests convert into replies.

---

### Solution 3: “Open loops” resurfacing (since you were away)
**What:** When opening DMs after time away, show a compact **Open loops** module:
- Threads marked Needs reply.
- Message Requests not acted on.
- Mentions/replies-to-you in group contexts.

**Why:** Helps users complete intended replies across sessions.

**MVP requirements:**
- Only use explicit signals (unreplied inbound, request pending, mentions/replies) to avoid content interpretation.
- Easy dismissal.

**Expected impact:** Increased DM return rate and higher open→reply conversion.

---

## 8) Experiment plan

### Experiment A: Needs reply marker
- **Control:** current DM list
- **Variant:** needs-reply marker + dismiss
- **Measure:** TTFR, CCR; guardrail wrong-priority dismiss rate

### Experiment B: Requests triage assist
- **Control:** current Requests UX
- **Variant:** triage row + one-tap actions + spam bucket
- **Measure:** time-to-first-action, acceptance rate; guardrail spam reports/blocks

### Experiment C: Open loops module
- **Control:** no module
- **Variant:** open loops on DM entry
- **Measure:** open→reply conversion, DM return rate; guardrail module dismiss rate

---

## 9) Risks & trade-offs
- **Risk:** “Needs reply” increases anxiety/guilt.
  - *Mitigation:* subtle labeling, dismiss, and user control.
- **Risk:** Prioritization surfaces spam.
  - *Mitigation:* conservative heuristics; integrate existing spam classification; guardrails.
- **Risk:** Extra UI modules add clutter.
  - *Mitigation:* only show when there are true open loops; strict frequency caps.

---

## 10) Rollout plan
1. Ship **Solution 1 (Needs reply)** MVP behind an experiment.
2. Iterate thresholds/heuristics based on dismiss + reply outcomes.
3. Add **Requests triage assist** as a parallel track focused on creators/SMBs first.
4. Add **Open loops module** once we validate it improves completion without increasing anxiety/clutter.

---

## 11) Open questions
- What’s the best user-facing copy for “Needs reply” that feels helpful, not judgmental?
- Should Needs reply behave differently for creators (high volume) vs everyday users?
- Which explicit signals best identify “high-intent” in a policy-safe way (story reply context, verified, mutuals, etc.)?
- How should the system behave for muted threads / restricted accounts?

---

## 12) Summary
This PRD targets Instagram DM retention and utility by improving **reply speed and conversation continuation** through lightweight prioritization (Needs reply), faster Message Requests triage, and resurfacing of open loops across sessions—without turning DMs into a heavy workflow tool.
