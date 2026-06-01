# PRD: Perplexity Verified Answers - Claim-Level Citation Grounding

**Product:** Perplexity Answer Engine (Trust Layer)
**Author:** Mayank Malviya
**Version:** v1 - Initial PRD
**Changes from v0:** First version.
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/perplexity-teardown.md

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, goals/non-goals, target personas, three solution pillars, core loops, basic requirements, event schemas, success metrics, competitive context, key trade-offs, open questions |

---

## Context

Perplexity's entire value proposition rests on a single promise: a cited answer is a verifiable answer. Every superscript number in every response is a contract - "this claim is grounded in this source."

That contract is broken more often than users know. The teardown identifies "citation laundering" - where the LLM confidently asserts a specific claim and attaches a real, plausible-looking URL whose actual content does not support the claim - as Perplexity's most critical quality failure mode. The failure is invisible in aggregate metrics: users who catch a laundered citation in a high-stakes context (medical, legal, financial, competitive analysis) stop using Perplexity for consequential queries without filing a support ticket. Usage appears to decline for no apparent reason. The highest-LTV cohort quietly leaves.

This PRD specifies **Perplexity Verified Answers**: a claim-level citation grounding system that checks the semantic relationship between each LLM-generated claim and its cited source at inference time, before the answer streams to the user. It makes the citation contract enforceable, not just decorative.

v1 establishes the core thesis, target segments, solution pillars, and measurement framework. v2 will add full data models, API contracts, acceptance criteria, and the competitive experiment framework. v3 will be the production-grade specification with phased rollout plan and resolved open questions.

---

## 1) Problem statement + why now

### The citation trust gap

Perplexity's synthesis pipeline has three distinct failure modes, ordered by frequency and severity:

| Failure mode | Description | User-visible signal | Business impact |
|---|---|---|---|
| **Citation laundering** | LLM asserts a specific fact; cited URL is real but the page does not support the specific claim | User clicks citation, finds it does not say what the answer claims | Permanent trust loss in high-stakes research segment; silent churn |
| **Source staleness** | Retrieved source is outdated; answer presents past state as current | User shares answer with a colleague; colleague corrects with a fresher source | Social embarrassment; "don't trust Perplexity for real-time info" signal spreads |
| **Unsupported assertion** | LLM makes a claim with no citation attached, or cites a low-quality thin-content source | User notices the citation links to a blog post with no primary data | Erosion of confidence; lower citation click rate; lower Pro conversion |

Citation laundering is the most dangerous because it is the hardest to detect passively. A hallucinated statistic attached to a real academic journal URL looks authoritative. A user who does not click through is harmed without knowing it. A user who does click through - and is a high-value knowledge worker who relies on Perplexity for serious research - permanently downgrades their trust in the product.

### Why now

Three forces converge that make this the right moment to invest:

1. **Google AI Overviews quality convergence.** Google is shipping AI-generated answer overviews on the majority of informational queries. If Perplexity cannot demonstrate measurably better citation accuracy than Google's AI answers, the "Perplexity is more trustworthy than Google" positioning collapses. Google's ad-revenue dependency creates a structural quality gap - but only if Perplexity actively defends its trust premium.

2. **Enterprise pipeline is blocked by citation accuracy.** The teardown identifies the "enterprise trust wall" as a primary deal-stall pattern: consultant-segment buyers want auditability, not just answers. A citation verification layer with per-answer audit logs transforms the enterprise sales conversation from "can we trust it?" to "here is the verification record."

3. **LLM citation accuracy is technically solvable now.** Embedding-based semantic similarity models (e.g., sentence-transformers, cohere-rerank) are fast enough (<200ms) and cheap enough (fractions of a cent per answer) to run as a post-generation verification pass without materially impacting answer latency or unit economics. This was not true 18 months ago.

---

## 2) Goals / non-goals

### Goals

1. Increase **citation accuracy rate** (proportion of cited claims where the cited source actually supports the specific claim) from an estimated 78-83% baseline to **>88%** on Academic Focus and **>85%** on Web Focus within 6 months of rollout.
2. Increase **citation click rate** (users who click at least one source per answer) from an estimated 16% to **>22%** by surfacing confidence signals that make verification feel rewarding, not anxiety-inducing.
3. Reduce **silent churn from citation failure** - measured indirectly as improvement in D30 retention for the knowledge worker and enterprise researcher segments that the teardown identifies as highest-LTV.
4. Create an **audit-ready citation record** per answer that enables enterprise buyers to evaluate Perplexity for regulated research use cases without a custom contract.
5. Drive **Pro conversion lift of +1.5 pp** among users who experience the verified answer signal, by making citation quality a visible Pro differentiation signal.

### Non-goals

- Replacing Perplexity's retrieval pipeline or changing which sources are retrieved. This PRD is a post-retrieval, post-generation verification layer - not a retrieval improvement project.
- Solving source staleness (retrieval freshness). That is a separate workstream targeting the retrieval infrastructure and freshness window.
- Adding fact-checking capabilities for claims that have no cited source. Unsupported assertions are out of scope for v1 - the verification layer operates only on claims that already carry a citation.
- Building a general-purpose hallucination detector. The scope is narrow: does this specific cited URL support this specific claim? Not: is this claim true in the world?
- Changing the streaming architecture. The verification pass runs in a pre-stream gate; it does not require redesigning the streaming token pipeline.

---

## 3) Target users / personas + JTBD

| Persona | Why citation accuracy matters most | Activation trigger for Verified Answers | JTBD |
|---|---|---|---|
| **Knowledge worker (PM, analyst, consultant, journalist)** | Uses Perplexity for briefings, competitive analyses, and memos that go to senior stakeholders. A wrong cited fact in a board deck is a credibility crisis. | First time they click a citation and find it confirms the claim precisely; first time they see a "Verified" confidence signal on a claim they were about to include in a report | "Give me cited claims I can paste directly into work product without needing to verify every one myself" |
| **Enterprise researcher / professional** | Produces formal deliverables (proposals, due diligence reports, client briefs) where citation accuracy is a professional standard. Currently blocked by "can we trust this?" in procurement | Demonstration of per-answer audit log to IT/legal during procurement; "citation verified against source" signal on every answer | "Give me a research tool I can present to IT as enterprise-grade, with a verifiable accuracy record" |
| **Graduate / academic researcher** | Uses Academic Focus mode for literature research. Academic citation standards are strict; a wrongly attributed claim can fail a thesis review | First Academic Focus query where the confidence signal shows "Verified by 4 sources agree" on a key finding they are about to cite | "Give me citations I can include in academic work with confidence they say what Perplexity claims they say" |
| **Developer / technical user** | Uses Perplexity for technical reference, migration guides, and debugging context. A wrong API signature cited from outdated docs wastes hours. | First time the recency + verification signal shows "source from [library docs, version X.Y]" on a code snippet | "Give me technical citations I know are current and accurate, so I do not debug based on wrong information" |

**Segment priority for rollout:** Academic Focus mode first (structured sources, highest citation accuracy expectations, clearest semantic match task), then Web Focus Pro users, then Web Focus free tier.

---

## 4) Insights from teardown

1. **Citation laundering is the primary trust failure, not hallucination.** The teardown is specific: the failure mode is not a made-up URL - it is a real URL attached to a claim the page does not support. This distinction matters for the technical solution: the system does not need to fact-check the claim against world knowledge; it needs to verify the semantic relationship between claim and cited page.

2. **The trust loop is the retention engine for the highest-LTV segment.** Loop D from the teardown (Answer -> Citation -> Trust) describes exactly the reinforcement cycle at stake. Every verified citation builds the habit. Every unverified citation in a high-stakes context breaks the habit permanently. Citation accuracy is not a quality metric - it is a retention metric for the segment that drives 80%+ of Pro revenue.

3. **The enterprise trust wall is primarily a product gap, not a sales gap.** The teardown's edge case 3 shows that enterprise deals stall not on pricing but on auditability: "are queries used to train models?" and "is there a DPA?" A per-answer citation audit log is the product answer to the auditability question in the research context specifically.

4. **Academic Focus is the cleanest starting point.** The teardown's Phase 0-1 rollout logic explicitly identifies Academic Focus as the safest starting segment: scholarly papers have structured abstracts and introductions that make semantic matching more reliable than thin-content web pages. The verification model can achieve higher accuracy with lower false-positive rates on academic sources.

5. **Confidence signals reduce anxiety without adding friction.** The teardown identifies confidence indicators (per-claim "4 sources agree" vs. "1 source") as a potential lever for increasing citation click rate and trust. The insight is that most users do not click citations because they feel uncertain about what they will find - a positive confidence signal ("verified") changes the click from "I'm checking because I don't trust this" to "I'm clicking to see the original."

---

## 5) Solution - three pillars

### Pillar 1: Inference-time claim-citation semantic verifier

A post-generation verification pass that runs after the LLM produces an answer draft, before the answer streams to the user. For each factual claim tagged with a citation superscript, the verifier:

1. Extracts the specific claim text from the draft answer.
2. Retrieves the full text of the cited source page (already in the retrieval cache from the query pipeline).
3. Runs an embedding-based semantic similarity check: does any passage in the source page semantically support this specific claim?
4. Assigns a `citation_confidence_score` (0.0-1.0) to each claim-citation pair.
5. For claims with score < 0.65: either (a) attempt to find a better supporting source from the already-retrieved pool, or (b) mark the citation as `unverified` in the answer draft.

**What the verifier does NOT do:** It does not rewrite the answer. It does not remove claims from the answer. It annotates confidence; the rendering layer surfaces those annotations to users.

**Latency contract:** The verification pass must complete within 400ms P95 for Academic Focus (structured sources, faster matching) and 600ms P95 for Web Focus. This adds to total answer latency but is within the acceptable range for the "pre-stream gate" architecture.

### Pillar 2: Citation confidence UI signals

The verification layer's output is only valuable if users can see and act on it. Three progressive confidence signals displayed in the source panel and inline with citations:

| Signal | Condition | UI treatment | User interpretation |
|---|---|---|---|
| `verified` | Score >= 0.82; at least 2 sources independently support the claim | Green checkmark on citation superscript; "Verified" label on source in right panel | Safe to use in work product; click for the source |
| `single_source` | Score >= 0.65; only one source supports the claim | No badge; default treatment | Standard - no additional signal needed |
| `low_confidence` | Score < 0.65; no retrieved source strongly supports the claim | Yellow indicator on citation superscript; "Limited sources" label in panel | Worth verifying before using; click to review |
| `unverified_claim` | Claim has no citation attached, or score is near zero | Grey indicator | This claim has no citation grounding; not suitable for formal use |

**Design constraint:** The confidence signals must not make answers feel fragile or undermine trust in aggregate. The target is that >70% of citations show `verified` or `single_source` (no negative badge). The `low_confidence` signal should appear on <15% of citations in well-grounded answers. If a query category produces >30% `low_confidence` signals, that is a signal for the retrieval team to improve source quality, not for the UI to show more warnings.

### Pillar 3: Per-answer citation audit record

Every verified answer generates a `citation_audit_record` - a structured log of the verification outcome for each claim-citation pair. This record is:

- Stored server-side for 5 years (aligned with dispute handling requirements identified in the Stripe PRD pattern for evidence retention).
- Accessible via API for Enterprise and Teams accounts.
- Displayed as a "Citation audit" panel in Spaces for enterprise accounts (Phase 2).
- Used as evidence in Perplexity's enterprise sales motion: "every answer has a machine-verifiable citation accuracy record."

The audit record is not visible to free-tier users in v1. It is the enterprise and Teams differentiator that addresses the "auditability gap" identified in the teardown's edge case 3.

---

## 6) Core loops (updated for Verified Answers)

### Loop D - Answer -> Citation -> Trust (updated citation credibility loop)

The teardown's Loop D describes the trust-building cycle. Verified Answers upgrades this loop with an explicit confidence signal that makes the positive arm more rewarding and the negative arm visible before it damages trust.

```
User receives answer
  -> Sees citation superscript with `verified` badge
  -> Clicks citation with anticipation of confirmation, not anxiety
  -> Source page confirms claim precisely
  -> Trust in Perplexity increases (positive reinforcement, not relief)
  -> Next query: user does not second-guess the answer before using it

vs. pre-Verified Answers:

User receives answer
  -> Sees citation superscript (no confidence signal)
  -> Clicks citation to check (anxiety-driven)
  -> Source page does NOT support the claim
  -> Trust in Perplexity permanently damaged for high-stakes queries
```

**Loop metric:** `citation_click_rate` segmented by `citation_confidence` level. Hypothesis: `verified`-badged citations show higher click rate (positive curiosity) while `low_confidence` citations also show higher click rate (warranted caution) vs. current unmodified baseline.

**Key events:**

```json
{
  "event": "citation_verified",
  "properties": {
    "thread_id": "thr_9aQ1rL",
    "answer_turn_index": 1,
    "citation_position": 2,
    "citation_confidence_score": 0.91,
    "confidence_signal_shown": "verified",
    "verification_source_domain": "nature.com",
    "verification_latency_ms": 310
  }
}

{
  "event": "citation_low_confidence_shown",
  "properties": {
    "thread_id": "thr_9aQ1rL",
    "answer_turn_index": 1,
    "citation_position": 4,
    "citation_confidence_score": 0.54,
    "original_source_domain": "medium.com",
    "replacement_source_found": false
  }
}

{
  "event": "citation_clicked_with_signal",
  "properties": {
    "thread_id": "thr_9aQ1rL",
    "citation_position": 2,
    "confidence_signal_shown": "verified",
    "time_since_answer_completed_ms": 8200,
    "user_plan": "pro"
  }
}
```

---

### Loop E - Enterprise buyer -> Audit demo -> Deal close (new enterprise trust loop)

```
Enterprise buyer evaluates Perplexity Teams
  -> IT asks: "How do we know the citations are accurate?"
  -> Sales demo: opens a research thread in Spaces
  -> Shows citation audit panel: per-claim confidence scores, source excerpts
  -> IT: "This is a verifiable accuracy record - acceptable for research use"
  -> Deal advances to DPA review (separate workstream)
  -> Team plan activated
```

**Loop metric:** Enterprise deal progression rate past "citation accuracy" objection in sales pipeline. Tracked via CRM stage data (not a product event - requires Sales Ops instrumentation).

---

## 7) Funnels

### Academic Focus user -> Verified Answers activation

| Stage | Observable signal | Target |
|---|---|---|
| Uses Academic Focus mode | `focus_mode_selected: academic` | >40% of graduate/academic segment |
| Sees first `verified` badge on a citation | `citation_verified` event with `confidence_signal_shown: verified` | 100% of Academic Focus answers post-rollout |
| Clicks a `verified`-badged citation | `citation_clicked_with_signal` where `confidence_signal = verified` | >30% citation click rate on `verified` citations |
| Returns to Academic Focus within 7 days | `session_started` with Academic Focus in first query | D7 return rate >35% for Academic Focus users who saw `verified` signals |
| Upgrades to Pro | `upgrade_initiated` within 30 days | +2 pp lift vs. Academic Focus users without `verified` signals |

### Enterprise buyer -> Audit record evaluation

| Stage | Observable signal | Target |
|---|---|---|
| Teams trial requested | `teams_trial_created` | Baseline; no change from Verified Answers rollout in v1 |
| Audit panel accessed during trial | `citation_audit_panel_opened` in Spaces | >60% of active Teams trial accounts |
| IT/legal stakeholder added to Space | `space_member_invited` with role `admin` or `it_reviewer` | >30% of Teams trials that access audit panel |
| Trial converts to paid Teams | `teams_subscription_activated` | +3 pp lift vs. Teams trials without audit panel access |

---

## 8) User journeys

### Happy path - Knowledge worker citing verified claims in a report

1. PM is building a competitive analysis on AI search tools for an investor update.
2. Opens Perplexity, types: "What is Perplexity's citation accuracy compared to Google AI Overviews?"
3. Answer streams; inline citations show green `verified` badges on two claims: "Perplexity displays 4-8 inline citations per answer" and "Google AI Overviews shows 3-5 source chips."
4. PM clicks the `verified` badge on the first claim. Source panel highlights the exact passage from the cited TechCrunch article that supports the specific metric.
5. PM pastes the claim and citation URL directly into the investor deck with confidence - no separate tab-checking required.
6. PM's colleague reads the deck and sees the inline citations. Asks: "How did you verify these?" PM explains: "Perplexity's answers are source-verified now."

**Value delivered:** PM saved 15+ minutes of tab-verification per sourced claim. Colleague becomes a referral path for a new Pro account.

---

### Edge case 1 - Academic citation with `low_confidence` signal correctly preventing propagation of a wrong claim

**Scenario:** Graduate student researching protein folding asks Perplexity about a specific AlphaFold metric.

**With Verified Answers:**
1. Perplexity retrieves 7 sources; LLM generates an answer with the claim "AlphaFold achieves 92.4% accuracy on CASP14 targets [3]."
2. Verifier checks claim against citation [3] - a Nature paper. The paper reports 92.4% accuracy on a specific subset, not all CASP14 targets.
3. Verifier assigns `citation_confidence_score: 0.61` (claim overgeneralises the finding from the source).
4. Answer shows a yellow `low_confidence` indicator on citation [3].
5. Student clicks citation [3] - reads the Nature paper; sees the correct scoped claim.
6. Student uses the correctly scoped claim ("92.4% accuracy on TBM targets in CASP14") in their thesis, not the overgeneralised version.
7. Verifier has prevented the propagation of a subtly wrong claim that would have been invisible without the signal.

**PM implication:** The value of `low_confidence` is not that it blocks the answer - it is that it directs the high-value user's attention to exactly the citation that needs human review. This is the right division of labour between AI and expert user.

---

### Edge case 2 - False positive `low_confidence` signal on a valid claim

**Scenario:** Technology journalist asks about a specific product launch date. Perplexity retrieves a press release and correctly states the launch date.

**Failure pattern:**
1. Verifier checks claim "Perplexity launched Spaces in February 2024 [2]" against citation [2], a press release.
2. The press release is structured as a PR boilerplate with the date in a footer, not in the body content where the semantic model primarily looks.
3. Verifier assigns `citation_confidence_score: 0.62` - borderline `low_confidence` due to structural mismatch, not semantic mismatch.
4. Yellow indicator appears on a correct, well-sourced claim.
5. Journalist clicks through; finds the date is correct; loses confidence in the confidence signal itself.

**PM implication:** False-positive `low_confidence` signals are corrosive to the signal's credibility. The verifier must be tuned to avoid false positives on structural mismatches (dates in footers, numbers in tables, proper nouns in headlines). The acceptance criteria for the verifier model must include: false-positive rate on manually-labelled "correct citation" test set < 8%. This is a model quality requirement, not a product design requirement.

---

### Edge case 3 - Pro user in enterprise trial who needs audit record for procurement

**Scenario:** A management consultant running a Perplexity Teams trial needs to present the citation audit record to the client firm's IT team for approval.

**With Verified Answers:**
1. Consultant runs a research session on competitive landscape in Spaces; all citations show `verified` badges.
2. Opens the citation audit panel from Spaces settings.
3. Exports the per-session audit record as a structured JSON (for IT) and a human-readable PDF (for legal).
4. IT reviews the record: "Each claim has a confidence score, a source URL, and a timestamp. The scoring methodology is documented. This is acceptable for enterprise research use."
5. Legal reviews: "This is a machine-generated accuracy record, not a human editorial guarantee. Acceptable for research tools with the right disclaimer."
6. Procurement unlocks. Team plan signed.

**PM implication:** The audit export format matters as much as the audit record itself. IT wants structured data. Legal wants a human-readable summary with a methodology disclaimer. The audit panel must ship with both export formats on day one of enterprise beta.

---

## 9) Basic requirements

### Req 1 - Claim-citation semantic verifier (inference time)

| ID | Requirement | Priority |
|---|---|---|
| 1.1 | Verifier runs as a post-generation pass before streaming begins; does not change claim text | P0 |
| 1.2 | Verifier assigns `citation_confidence_score` (0.0-1.0) to each claim-citation pair using embedding-based semantic similarity | P0 |
| 1.3 | Verification latency: <400ms P95 for Academic Focus; <600ms P95 for Web Focus | P0 |
| 1.4 | For claims with score < 0.65: verifier first attempts to find a replacement source from the already-retrieved pool before marking `low_confidence` | P1 |
| 1.5 | False-positive rate (correct claims marked `low_confidence`) must be < 8% on the manually-labelled test set | P0 |
| 1.6 | Verifier degrades gracefully: if verification service is unavailable, answer streams without confidence signals (no user-visible failure) | P0 |

### Req 2 - Citation confidence UI signals

| ID | Requirement | Priority |
|---|---|---|
| 2.1 | `verified` badge (score >= 0.82, 2+ agreeing sources) displayed as a green visual indicator on the citation superscript | P0 |
| 2.2 | `low_confidence` indicator (score < 0.65) displayed as a yellow visual indicator | P0 |
| 2.3 | Clicking a confidence badge opens the source panel with the specific supporting passage highlighted | P1 |
| 2.4 | Confidence signals must not reduce source panel click rate for `single_source` citations vs. current baseline | P0 (guardrail) |
| 2.5 | `verified` badge tooltip on hover: "Claim verified - the cited source directly supports this statement" | P2 |

### Req 3 - Citation audit record

| ID | Requirement | Priority |
|---|---|---|
| 3.1 | `citation_audit_record` created server-side for every answer that undergoes verification; retained for 5 years | P0 |
| 3.2 | Audit record fields: `claim_text`, `citation_url`, `citation_confidence_score`, `verification_timestamp`, `source_passage_excerpt`, `model_version_used` | P0 |
| 3.3 | Audit record accessible via REST API for Enterprise and Teams accounts | P1 |
| 3.4 | Audit panel in Spaces (Enterprise and Teams): per-session audit record with export to JSON and PDF | P1 |
| 3.5 | Free-tier users: no audit record access; answer confidence signals only | P0 |

---

## 10) Event schemas

### `citation_verification_completed`

Emitted once per answer after the full verification pass completes, before streaming begins.

```json
{
  "event": "citation_verification_completed",
  "properties": {
    "thread_id": "thr_9aQ1rL",
    "answer_turn_index": 1,
    "focus_mode": "academic",
    "total_citations": 6,
    "verified_count": 4,
    "low_confidence_count": 1,
    "unverified_count": 1,
    "verification_pass_latency_ms": 312,
    "verifier_model_version": "claim-verifier-v1.0",
    "replacement_source_found_count": 1,
    "user_plan": "pro"
  }
}
```

### `citation_signal_rendered`

Emitted when a confidence signal is displayed to the user in the answer.

```json
{
  "event": "citation_signal_rendered",
  "properties": {
    "thread_id": "thr_9aQ1rL",
    "answer_turn_index": 1,
    "citation_position": 2,
    "signal_type": "verified",
    "citation_confidence_score": 0.91,
    "source_domain": "nature.com",
    "focus_mode": "academic"
  }
}
```

### `citation_audit_panel_opened`

Emitted when an Enterprise or Teams user accesses the audit panel in Spaces.

```json
{
  "event": "citation_audit_panel_opened",
  "properties": {
    "space_id": "spc_5nT8vW",
    "thread_id": "thr_9aQ1rL",
    "opener_plan": "teams",
    "opener_role": "admin",
    "total_verified_in_session": 14,
    "total_low_confidence_in_session": 2,
    "audit_export_triggered": false
  }
}
```

### `citation_audit_exported`

Emitted when a user exports the audit record from the Spaces audit panel.

```json
{
  "event": "citation_audit_exported",
  "properties": {
    "space_id": "spc_5nT8vW",
    "export_format": "pdf",
    "total_answers_included": 7,
    "total_verified_citations": 34,
    "exporter_plan": "teams",
    "is_procurement_context": null
  }
}
```

---

## 11) Success metrics

### North Star

**Verified Answer trust score:** Weekly-active Pro and Teams users who experience at least one `verified`-badged citation AND return within 7 days. Rationale: this cohort has experienced the positive trust loop, not just the feature. A `verified` badge that no one clicks or acts on is UX decoration. A `verified` badge that changes behaviour (return rate, downstream citation use) is a trust-loop compound investment.

### Input metrics (leading indicators)

| Metric | Estimated baseline | Target (6 months post-rollout) | Instrumentation event | Priority |
|---|---|---|---|---|
| Citation accuracy rate (Academic Focus) | ~80-83% | >88% | `citation_verification_completed`, `verified_count / total_citations` | P0 |
| Citation accuracy rate (Web Focus) | ~75-80% | >85% | Same | P0 |
| Citation click rate on `verified`-badged citations | Not measured (new signal) | >30% | `citation_clicked_with_signal` where `signal_type: verified` | P0 |
| Citation click rate on `low_confidence` citations | Not measured | >45% (expected: warning prompts review) | `citation_clicked_with_signal` where `signal_type: low_confidence` | P1 |
| `low_confidence` citation rate per answer | Not measured | <15% of all citations | `low_confidence_count / total_citations` per `citation_verification_completed` | P0 (guardrail) |
| Enterprise Teams trials that open audit panel | 0 (new feature) | >60% of active Teams trials | `citation_audit_panel_opened` | P1 |
| D7 retention for Academic Focus users exposed to `verified` signals | ~28% (estimated Academic Focus baseline) | >35% (+7 pp) | D7 session return for cohort | P0 |

### Guardrail metrics

| Metric | Threshold | Risk if breached |
|---|---|---|
| Answer latency P95 (time to first token, Academic Focus) | Must not increase >400ms vs. pre-rollout baseline | Verification pass overhead makes Academic Focus feel slower than Google Scholar; competitive regression |
| Answer latency P95 (time to first token, Web Focus) | Must not increase >600ms vs. pre-rollout baseline | Slows primary use case; potential D7 retention harm |
| False-positive `low_confidence` rate (correct citations marked warning) | <8% on manual test set; <12% in production sampling | False positives erode confidence signal credibility; users ignore all signals |
| Source panel click rate overall (all citation types) | Must not decline >2 pp vs. pre-rollout baseline | Confidence signals create over-confidence; users stop verifying even when they should |
| Pro churn rate (monthly) | Must not increase during rollout window | If `low_confidence` signals feel like indictments of answer quality, users downgrade trust in the whole product |

---

## 12) Competitive context

| Competitor | Citation approach | Verified Answers advantage | Risk |
|---|---|---|---|
| **Google AI Overviews** | Source chips displayed below synthesised answer; no per-claim citation; no confidence signal | Perplexity's inline per-claim citation + confidence signal is a demonstrably more rigorous attribution model | Google has 10x the engineering resources; could ship a similar verification layer within 12 months |
| **ChatGPT (with web search)** | Inline footnote links; no claim-level verification; known hallucination rate on synthesised web content | ChatGPT's browsing is a feature; Perplexity's verified citation is a product commitment | OpenAI could ship confidence signals quickly; their advantage would be breadth of deployment (100M+ users) |
| **Bing Copilot** | Source panel with excerpts; no claim-level confidence signals | Similar to ChatGPT - no claim-level verification; Perplexity has meaningful head start | Bing has access to real-time web index and could add verification infrastructure |
| **Perplexity vs. academic search (Google Scholar, PubMed)** | Academic search returns raw papers; no synthesis; no citations needed (user reads the paper) | Verified Answers closes the "I trust Scholar more than Perplexity" objection for academic segment | Academic search has no synthesis step; the comparison is only relevant for users who want both synthesis AND accuracy |

**Competitive moat from Verified Answers:** The `citation_audit_record` is a unique enterprise differentiator. None of the above competitors expose per-answer, per-claim citation verification records via API. For enterprise buyers who need auditability, this is a point of unique differentiation that Google, ChatGPT, and Bing do not have - and cannot quickly replicate because it requires both a verification layer AND an enterprise data model that most consumer AI tools have not invested in.

---

## 13) Key trade-offs

| Trade-off | Current bias (v1) | Cost | Alternative considered |
|---|---|---|---|
| Verification latency vs. answer quality | Accept up to 600ms added latency for Web Focus | Some users may perceive a slower answer; competitive disadvantage if Google closes the speed gap | Run verification asynchronously after streaming starts; show confidence signals as they complete rather than gating the stream. Risk: answer arrives before signals; late signal appearance feels like a glitch |
| Confidence signal granularity vs. user trust in the signal | Three-level signal (`verified`, `single_source`, `low_confidence`) | More levels = more information but more cognitive load; risk of "confidence signal fatigue" | Binary signal (verified / not verified). Rejected: too blunt; `single_source` citations (one good source) should not be penalised with a "not verified" label |
| Verification on every answer vs. targeted verification | Academic Focus first; Web Focus in Phase 2 | Users on Web Focus in Phase 1 do not get confidence signals; inconsistent experience across modes | Verify all answers from day one. Rejected: false-positive rate on thin Web content too high for v1; risks undermining signal credibility before model is tuned |
| Audit record as Pro feature vs. all users | Pro and above for audit record; confidence signals for free tier | Free users get better citation confidence UX but no auditability; reduces the Pro differentiation argument for this specific feature | Audit records for all users. Rejected: storage cost and privacy surface area; enterprise auditability is a specific value prop for the Teams/Enterprise tier, not a mass-market feature |
| Automatic source replacement for `low_confidence` vs. user notification | Attempt replacement first; fall back to `low_confidence` signal | Replacement may substitute a marginally better source but miss the user's preferred original source | Only notify; never replace. Rejected: if a better source exists in the retrieved pool, replacing silently improves accuracy with no UX cost - the user can still click through to see the replacement |

---

## 14) Open questions (for v2 to resolve)

1. **Verifier model selection:** Should the claim-citation semantic verifier use a general-purpose embedding model (sentence-transformers, OpenAI ada-002 embeddings), a cross-encoder reranking model (cohere-rerank, BGE-reranker), or a fine-tuned model specifically trained on Perplexity's (claim, citation, label) pairs? The choice affects latency, cost, and accuracy. v2 must include a benchmarked comparison.

2. **Verification latency budget allocation:** The 400ms and 600ms latency targets are estimates. The actual acceptable latency will depend on user research with the Academic Focus and Web Focus segments. Does 400ms added latency to an Academic query feel acceptable if it produces a `verified` badge? Needs user testing data before v2 acceptance criteria are set.

3. **False-positive rate calibration:** The 8% false-positive target is a starting hypothesis. The right threshold depends on the population of "borderline correct" citations - claims that are technically accurate but where the source only weakly supports the specific wording. What is the actual distribution of citation confidence scores on a labelled test set of Perplexity answers? This requires a labelling project in Phase 0.

4. **`low_confidence` UX messaging:** Does showing a yellow indicator create anxiety (users doubt all of Perplexity's answers) or correct calibration (users know which claims need extra review)? This is a UX research question that affects the copy and visual design of the signal. v2 must include a proposed A/B test design.

5. **Audit record privacy model:** Citation audit records contain the full text of user queries and the cited claim text. For users on free and Pro plans (who do not have the audit record feature), how long does Perplexity retain the raw verification output? Is the `citation_confidence_score` stored against the user account, or anonymised? This requires a data retention and privacy review before v2.

6. **Multi-model verification:** Pro users can switch synthesis models (GPT-4o, Claude, Gemini). Does the verifier need to behave differently for claims generated by different synthesis models? Hypothesis: Claude-generated answers may have different citation laundering patterns than GPT-4o-generated answers. v2 should include a segmented analysis of `citation_confidence_score` distributions by synthesis model.

---

*All metrics are directional estimates based on publicly observable product behaviour and industry benchmarks - not internal Perplexity data.*
