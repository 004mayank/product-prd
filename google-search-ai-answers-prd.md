# PRD: Google Search - AI Answers (Grounded) Without Breaking the Web

**Product area:** Google Search (SERP + ranking + ads + publisher ecosystem)
**Author:** Mayank Malviya
**Date:** 22 Mar 2026
**Status:** Final

**Related teardown:**
- Google Search with AI teardown : https://github.com/004mayank/product-teardowns/blob/main/google-search-with-ai-teardown.md

---

## 0) Executive summary
Google Search is shifting from “rank documents” to a **universal intent router** that can also produce **grounded, cited answers**.

This PRD defines **AI Answers** (an AI block on the SERP) designed to:
- improve **user success** on eligible informational intents (fewer reformulations, faster completion)
- preserve **trust** via grounding, citation correctness, freshness handling, and conservative fallbacks
- preserve **web health** by making citations drive **qualified** publisher value (not decorative links)
- preserve **ads integrity** by keeping a hard trust boundary between ads and AI output

**v3 core product decision:** AI Answers ships with an explicit **Web Value Exchange** contract:
1) **Don’t answer** when the SERP’s best outcome is “go read/compare sources” (policy)
2) When we do answer, **design the UI to create real publisher value** (UX + incentives)
3) Enforce **ecosystem guardrails** at query-cluster level (metrics + automatic tightening)

---

## 1) Problem statement
### 1.1 User problem
For many informational and multi-step queries, classic search requires:
- opening multiple links
- synthesizing conflicting information
- translating content into a plan

This is high effort, especially on mobile.

### 1.2 Ecosystem problem (hard constraint)
If AI answers satisfy intent without returning meaningful value to publishers, the open web’s incentive to produce high-quality content degrades-eventually harming search quality.

### 1.3 Trust problem
AI answers introduce failure modes:
- confident wrong answers
- citation mismatch (citations don’t support the claim)
- freshness failures (stale info in fast-changing topics)

A small number of high-profile incidents can damage Search’s trust moat.

---

## 2) Goals and non-goals
### 2.1 Goals 
**G1 - User success**
- Reduce reformulation rate and time-to-satisfaction on eligible queries.

**G2 - Trust and safety**
- Keep critical error rate below strict threshold; prefer safe fallbacks.

**G3 - Healthy web exchange (explicit value exchange)**
- Ensure AI Answers produce **measurable qualified publisher value** (not just raw clicks).

**G4 - Monetization integrity**
- Preserve ad trust boundaries and commercial intent capture via routing.

### 2.2 Non-goals 
- “Chat-only SERP” or replacing classic ranking universally.
- Solving regulated advice (medical/legal/finance) beyond narrowly-scoped definitional content.
- Publisher licensing/revenue share as a launch requirement.

---

## 3) Scope
### 3.1 In scope
- AI Answer block for **eligible** queries
- Eligibility policy engine (intent + risk + freshness + web-impact)
- Retrieval-grounded generation with citation correctness checks
- **Publisher value exchange UX** (source-first interactions + incentives)
- Follow-ups that preserve original intent
- Safe fallbacks (links-first, vertical modules)
- Instrumentation + evaluation pipeline

### 3.2 Out of scope (initial)
- Personal answers using private user data
- Long multi-turn conversations across intents
- In-answer checkout/transactions

---

## 4) Users and primary use cases
### 4.1 Personas
- **Everyday learner:** wants a clear explanation with sources
- **Decision maker:** wants comparisons and trade-offs
- **Task completer:** wants a step-by-step plan
- **Time-sensitive seeker:** wants up-to-date info

### 4.2 Primary use cases (eligible)
- “How do I…?” multi-step guidance (generic)
- Concept explanations (“what is…”, “why…”) with citations
- Comparisons (“X vs Y”) for non-regulated topics
- “Best practices” queries where sources converge

### 4.3 Excluded (default)
- Medical/legal/finance advice
- Breaking news synthesis (default to sources-first)
- Highly personalized recommendations without explicit user context

---

## 5) Product principles
1) **Answer when it helps; route when intent is vertical; ask when ambiguous; fallback when risky.**
2) **Trust > cleverness.** Prefer sources-first over speculative synthesis.
3) **Design for the web to survive.** Citations must create real publisher value.
4) **Make uncertainty legible.** Timestamp, qualifiers, and conservative language.
5) **Ads integrity is sacred.** Never blur boundaries.

---

## 6) Proposed solution 

### 6.1 Eligibility decision: Answer | Ask | Route | Links-only
For each query, predict:
- intent bucket: Know / Do / Go / Buy
- risk class: low / medium / high
- freshness sensitivity: evergreen / periodic / breaking
- answerability: confidence that retrieved sources support a stable synthesis
- ecosystem impact: likelihood AI block suppresses **qualified** outbound value

**Policy outcomes:**
- **Answer:** show AI block with grounded response + citations
- **Ask:** show clarifying question when query branches materially
- **Route:** show vertical-first (Maps/Shopping/News); AI can assist but not be terminal
- **Links-only:** classic SERP (and modules) without AI synthesis

**v3 default posture (more explicit about web impact):**
- Answer: low/medium-risk Know/Do queries with strong retrieval diversity and support
- Ask: ambiguous “best X” / “should I” where user constraints change the best outcome
- Route: Buy and Go intents; AI summarizes options, pushes actions to vertical modules
- Links-only: high-risk domains + breaking topics + low-support retrieval + **high web-impact risk clusters**

### 6.2 AI Answer block UX (v3 requirements)
**Above the fold:**
- 3–6 bullet summary (scannable)
- 2–5 primary citations visible immediately
- “Explore sources” is explicit and prominent

**Source-first interactions (Web Value Exchange UX):**
- **Claim-to-source affordance:** tapping a bullet reveals the supporting passages + source card (not just a favicon link)
- **Multi-source strip:** when multiple sources support a claim, show them (avoid a single “winner take all” citation)
- **“Continue on source” CTA** on at least one primary card above the fold (A/B measured)
- **Source diversity bias:** prefer showing at least 2 distinct domains above the fold when feasible

**Expandable sections:**
- steps / checklist (Do)
- definitions and key points
- pros/cons (comparisons)
- “What’s uncertain / depends on” (when applicable)

**Trust affordances:**
- “As of <time>” for periodic topics
- citations section-anchored at minimum; claim-anchored where feasible
- “Report an issue” categories: wrong / outdated / unsafe / missing sources / citation mismatch

**Non-dead-end constraint:**
- AI block must not become a terminal experience by default.
- Always provide clear paths to sources and classic results.

### 6.3 Retrieval + grounding + citation correctness
**Retrieval requirements:**
- minimum source count (N) and minimum domain diversity (D)
- source quality thresholds (spam / thin content / AI-generated SEO)
- allow multiple perspectives on contested topics (or fallback)

**Grounding requirements:**
- generation constrained to retrieved content
- citations must support the associated section/claim

**Citation correctness gates (v3 explicit):**
- automated checks: entailment/consistency scoring between claim and cited passage
- coverage checks: each summary bullet must be supported by ≥1 cited passage
- if citation correctness below threshold → fallback to links-only

### 6.4 Freshness handling
- **Evergreen:** allow caching (short TTL) for cost/latency
- **Periodic:** require timestamped sources; show “as of”; bias to freshest high-quality sources
- **Breaking:** default to sources-first modules; synthesis only under very strict constraints

### 6.5 Ads + monetization integrity
- Ads rendered as usual with clear labels
- AI answer must not incorporate sponsored content as “facts”
- Commercial intents route to Shopping/merchant modules; AI can summarize but must encourage comparison actions

---

## 7) Success metrics and guardrails

### 7.1 User success
- Reformulation rate (eligible queries)
- Time-to-satisfying-action (click/dwell/save/call/navigate)
- Session success proxy (reduced pogo-sticking)

### 7.2 Trust + safety
- Critical error rate (severity-weighted) per 10k
- Citation correctness rate (human + automated)
- Freshness incidents (stale/incorrect on periodic/breaking)

### 7.3 Publisher ecosystem health (v3 formalizes a “value exchange scorecard”)
**Primary (must hold):**
- **Qualified outbound value from citations:**
  - long click rate
  - downstream engagement (scroll depth / time)
  - “return-to-SERP quickly” penalty
- **Publisher diversity:** domain concentration / diversity index

**Secondary (watch):**
- Publisher complaint rate / escalation volume
- Topic-cluster content supply health (new high-quality docs created over time)

**Guardrail automation:**
- If qualified outbound value drops below threshold for a query cluster:
  - tighten eligibility to Links-only OR
  - switch to source-first layout variant OR
  - increase above-the-fold source prominence

### 7.4 Revenue integrity
- Buy-intent capture into shopping actions
- Ad trust metrics (complaints, misclicks, surveys)

---

## 8) Functional requirements

### 8.1 Policy engine
- classify intent + risk + freshness
- answerability score from retrieval support
- ecosystem impact score (predict suppression of qualified outbound value)
- category-level kill switches
- query-cluster exclusions and rapid rollback

### 8.2 AI answer generation
- strict JSON output internally
- structured answer sections
- citations: minimum above-the-fold citations + section/claim anchoring
- fallback when:
  - retrieval diversity insufficient
  - citation correctness checks fail
  - domain is high-risk

### 8.3 Observability + feedback
- log decision: Answer/Ask/Route/Links-only and key factors
- log sources retrieved and citations displayed
- log user actions: expand, click citations, view passages, continue to source, return to SERP
- user feedback: wrong/outdated/unsafe/missing sources/citation mismatch

---

## 9) Non-functional requirements
- Latency: AI block must not materially degrade perceived SERP load (progressive render)
- Reliability: degrade gracefully to classic SERP
- Cost controls: caching + eligibility constraints; track $/satisfied-session
- Abuse resistance: detect AI-targeted SEO spam; maintain source quality thresholds

---

## 10) Rollout plan

### Phase 0 - internal + dogfood
- restricted query sets
- heavy human review
- rapid kill switches

### Phase 1 - public (low-risk)
- low/medium-risk Know queries
- limited geo/language (English first)
- strict fallbacks

### Phase 2 - expand carefully
- broaden eligible intents (some Do)
- introduce clarifying questions for ambiguous queries
- iterate source-first UX to increase qualified publisher value

---

## 11) Risks and mitigations
- Hallucinations → grounding thresholds + links-only fallback
- Citation mismatch → claim/section anchoring + automated checks + eval
- Publisher backlash → qualified value guardrails + diversity requirements + source-first UX
- Regulatory scrutiny → transparency, uncertainty communication, audit logs
- Ads trust erosion → strict separation; no sponsored blending

---

## 12) Open questions
- Minimal acceptable **qualified publisher value** threshold per query cluster
- Best UX for uncertainty without confusing users
- Multiple perspectives UX for contested topics
- Breaking news line: ever synthesize, or always sources-first?

---

## 13) Appendix: v3 launch checklist
- [ ] Eligibility policy doc + approved risk taxonomy
- [ ] Citation correctness evaluation harness (automated + human)
- [ ] Breaking-news detection + hard fallback
- [ ] Kill switches per category + incident response runbook
- [ ] Scorecard dashboards (user success, trust/safety, publisher health, revenue)
- [ ] Source-first UX experiments shipped with success criteria
