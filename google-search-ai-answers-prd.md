# PRD v1: Google Search — AI Answers (Grounded) Without Breaking the Web

**Product area:** Google Search (SERP + ranking + ads + publisher ecosystem)  
**Author:** Mayank Malviya  
**Date:** 18 Mar 2026  
**Status:** v1 (draft)  

**Related teardown:**
- Google Search with AI teardown (v3): https://github.com/004mayank/product-teardowns/blob/main/google-search-with-ai-teardown.md

---

## 0) Executive summary
Google Search is evolving from a “rank documents” product into a **universal intent router** that can also produce **grounded, cited answers**.

This PRD proposes a v1 of **AI Answers** (an AI block on the SERP) that:
- improves **user success** on informational queries (fewer reformulations, faster task completion)
- preserves **trust** via grounding + citations + safe fallbacks
- minimizes **ecosystem harm** by designing for *qualified* outbound traffic and source diversity
- protects **ads integrity** by keeping a hard trust boundary between ads and AI answers

The core product decision is a policy engine that chooses: **Answer vs Ask vs Route vs Links-only**, based on intent and risk.

---

## 1) Problem statement
### 1.1 User problem
For many informational and multi-step queries, traditional search requires users to:
- open multiple links
- synthesize conflicting information
- translate content into an action plan

This is high effort, especially on mobile.

### 1.2 Ecosystem problem (hard constraint)
If AI answers satisfy intent without sending meaningful value back to publishers, the open-web supply of high-quality content degrades over time, which ultimately harms search quality.

### 1.3 Trust problem
AI answers introduce new failure modes:
- confident wrong answers
- citation mismatches
- freshness failures (breaking news)

A few high-profile errors can erode the Search trust moat.

---

## 2) Goals and non-goals
### 2.1 Goals (v1)
**G1 — User success:** Reduce query effort for low/medium-risk informational intents.
- Target: meaningfully lower reformulation rate and time-to-satisfaction on eligible queries.

**G2 — Trust and safety:** Ship with strong guardrails.
- Target: keep critical error rate below a strict threshold; prefer safe fallbacks.

**G3 — Healthy web exchange (minimum viable):** Ensure citations drive *qualified* outbound traffic.
- Target: citations are visible, claim-anchored where possible, and diverse.

**G4 — Monetization integrity:** Preserve ad trust boundaries.
- Target: ads remain clearly labeled; AI answer content is not sponsored.

### 2.2 Non-goals (v1)
- Replacing the entire SERP with chat (this is not “AI-only search”).
- Solving all query classes (e.g., medical/legal/finance; real-time breaking news).
- Publisher compensation mechanisms (licensing/revenue share) — may be explored later, but not required for v1 launch.

---

## 3) Scope
### 3.1 In scope
- SERP AI Answer block for **eligible** queries
- Eligibility policy engine (intent + risk)
- Grounded answer generation with citations
- Follow-up suggestions that **preserve original intent**
- Safe fallbacks (links-first, vertical modules)
- Instrumentation + evaluation pipeline

### 3.2 Out of scope (initially)
- Personalized answers using private user data
- Long conversational sessions across many intents
- Direct transactions (checkout) inside AI answer

---

## 4) Users and primary use cases
### 4.1 Personas
- **Everyday learner:** wants a clear explanation with sources (low patience)
- **Decision maker:** wants comparison + trade-offs (high consideration)
- **Time-sensitive seeker:** asks about news/events (high freshness sensitivity)

### 4.2 Primary use cases (v1)
- “How do I…?” multi-step guidance (generic)
- Concept explanations (“what is…”, “why does…”) with citations
- Comparisons (“X vs Y”) for non-regulated topics

### 4.3 Explicitly excluded use cases (v1)
- Medical/legal/finance advice (unless purely definitional and heavily constrained)
- Breaking news answers where freshness can’t be guaranteed

---

## 5) Product principles
1) **Answer when it helps, cite when it matters, route when intent is vertical.**
2) **Trust > cleverness.** Prefer “sources-first” over speculative answers.
3) **Design for the web to survive.** Citations are not decorative.
4) **Make uncertainty legible.** Timestamp, qualifiers, and fallbacks.
5) **Ads integrity is sacred.** Never blur boundaries.

---

## 6) Proposed solution (v1)
### 6.1 Eligibility decision: Answer | Ask | Route | Links
For each query, the system predicts:
- intent bucket: Know / Do / Go / Buy
- risk class: low / medium / high
- freshness sensitivity: evergreen / periodic / breaking

**Policy outcomes:**
- **Answer:** show AI block with grounded response + citations
- **Ask:** show clarifying question (when query is underspecified and branching)
- **Route:** show vertical-first (Maps/Shopping/News), AI assistive summary optional
- **Links-only:** no AI answer; show classic SERP and/or modules

**v1 policy defaults (conservative):**
- Answer: low/medium-risk Know queries with multi-source retrieval
- Ask: ambiguous “best X” informational queries
- Route: Buy and Go intents (AI assistive, not terminal)
- Links-only: high-risk domains + breaking topics

### 6.2 AI Answer block UX
**Core components:**
- 3–6 bullet summary (scannable)
- expandable sections (steps, definitions, pros/cons)
- visible citations (minimum set; ideally claim-anchored)
- “As of <time>” for periodic content
- follow-up suggestions that maintain intent (“Compare options”, “Show sources”, “See more results”)

**Key UX constraint:** AI block must not become a dead-end.
- add an explicit “Explore sources” affordance
- surface 2–5 primary citations above the fold

### 6.3 Retrieval and grounding (product requirements)
- Retrieval must return a **diverse** set of candidate sources
- Generation must be **grounded** (constrained to retrieved text)
- Citations must be **correct** (support the claim)
- If grounding/citations fail thresholds → fallback to links-only

### 6.4 Freshness handling
- Evergreen: allow caching (short TTL) for cost/latency
- Periodic: require timestamped sources + show “as of”
- Breaking: default to sources-first (News module), no synthesized answer unless high-confidence constraints met

### 6.5 Ads and monetization integration
- Ads are rendered as usual with clear labels
- AI answer must not incorporate sponsored content as “facts”
- For Buy queries: route to Shopping/merchant modules; AI can summarize options but must encourage comparison actions

---

## 7) Success metrics and guardrails
### 7.1 User success (utility)
- **Reformulation rate** on eligible queries
- **Time-to-satisfying-action** (click, save, call, navigate, long dwell)
- **Query success proxy** (session ends without back-to-SERP pogo-sticking)

### 7.2 Trust + safety
- **Critical error rate** (high severity factual/safety failures per 10k)
- **Citation correctness rate** (human + automated checks)
- **Freshness incidents** (breaking topics answered incorrectly/stale)

### 7.3 Publisher ecosystem health
- **Qualified outbound traffic from citations** (engaged visits, not just clicks)
- **Citation diversity index** (domain concentration)
- **Publisher complaint rate** / escalation volume

### 7.4 Revenue integrity
- **Commercial intent capture** (Buy queries reaching shopping actions)
- **Ad trust metrics** (misclicks/complaints; ad-label recognition surveys)

---

## 8) Requirements (functional)
### 8.1 Eligibility + routing
- Must classify intent + risk + freshness
- Must support category-level kill switches
- Must support query-cluster exclusions

### 8.2 Answer generation + citations
- Must produce answers only when minimum retrieval constraints met
- Must attach citations to major claims (or at least sections)
- Must support “Show sources” UI

### 8.3 Logging and feedback
- Must log: query class, eligibility decision, sources retrieved, citations shown, user actions
- Must provide user feedback: “wrong”, “outdated”, “unsafe”, “missing sources”

---

## 9) Requirements (non-functional)
- **Latency:** AI block should not materially degrade perceived SERP load; use streaming and/or progressive render
- **Reliability:** graceful degradation to classic SERP
- **Cost controls:** caching + eligibility constraints; measure $/satisfied-session
- **Abuse resistance:** detect AI-targeted SEO spam; maintain source quality thresholds

---

## 10) Rollout plan
### Phase 0: internal + dogfood
- restricted query sets
- heavy human review and rapid rollback

### Phase 1: public, low-risk
- low/medium-risk Know queries only
- geo/language limited (start with English)
- strict safety fallback

### Phase 2: expand carefully
- broaden query coverage
- introduce clarifying questions for ambiguous queries
- iterate citations UX to improve qualified clicks

---

## 11) Risks and mitigations
- **Hallucinations / incorrect synthesis** → strict grounding thresholds + links-only fallback
- **Citation mismatch** → claim-anchoring + automated checks + human eval
- **Publisher backlash** → design citations for qualified clicks; monitor ecosystem scorecard
- **Regulatory scrutiny** → transparent labeling, uncertainty communication, audit logs
- **Ads trust erosion** → strict separation; never blend sponsored content into AI response

---

## 12) Open questions
- What is the minimal acceptable **publisher value exchange** for v1 (qualified clicks threshold)?
- How should we display **uncertainty** without confusing users?
- What’s the best UX for **multiple perspectives** on contested topics?
- Where should we draw the line on **breaking news** — ever answer, or always sources-first?

---

## 13) Appendix: v1 launch checklist
- [ ] Eligibility policy doc + approved risk taxonomy
- [ ] Citation correctness evaluation harness
- [ ] Breaking-news detection + hard fallback
- [ ] Kill switches per category + incident response runbook
- [ ] Scorecard dashboards (user success, trust/safety, publisher health, revenue)
