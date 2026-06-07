# PRD: Perplexity Verified Answers - Claim-Level Citation Grounding

**Product:** Perplexity Answer Engine (Trust Layer)
**Author:** Mayank Malviya
**Version:** v3 - Final PRD
**Changes from v2:** Resolved Q7 (proprietary verifier model): build vs. buy analysis with investment thesis, labelling milestone gates, go/no-go criteria, and Phase 3 model promotion checklist. Added phased rollout plan with explicit launch gates, kill-switch trigger conditions, and dependency critical path. Expanded experiment backlog with rollout owners, acceptance criteria, and end-state decisions. Added launch readiness checklist. All seven open questions are now resolved.
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/perplexity-teardown.md

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, goals/non-goals, target personas, three solution pillars, core loops, basic requirements, event schemas, success metrics, competitive context, key trade-offs, open questions |
| v2 | Full data models, API contracts, verifier model benchmark and technology decision, detailed acceptance criteria, competitive experiment framework with power calculations, resolved open questions 1-6, UX copy proposals, dependency map, expanded instrumentation spec |
| v3 | Resolved Q7 (proprietary model build vs. buy analysis with milestone gates), phased rollout plan with launch gates and kill-switch conditions, expanded experiment backlog with rollout owners and end-state decisions, launch readiness checklist |

---

## Context

Perplexity's entire value proposition rests on a single promise: a cited answer is a verifiable answer. Every superscript number in every response is a contract - "this claim is grounded in this source."

That contract is broken more often than users know. The teardown identifies "citation laundering" - where the LLM confidently asserts a specific claim and attaches a real, plausible-looking URL whose actual content does not support the claim - as Perplexity's most critical quality failure mode. The failure is invisible in aggregate metrics: users who catch a laundered citation in a high-stakes context (medical, legal, financial, competitive analysis) stop using Perplexity for consequential queries without filing a support ticket. Usage appears to decline for no apparent reason. The highest-LTV cohort quietly leaves.

This PRD specifies **Perplexity Verified Answers**: a claim-level citation grounding system that checks the semantic relationship between each LLM-generated claim and its cited source at inference time, before the answer streams to the user. It makes the citation contract enforceable, not just decorative.

v1 established the core thesis, target segments, solution pillars, and measurement framework. v2 added the full data models, API contracts, verifier model technology decision, detailed acceptance criteria, competitive experiment framework with statistical rigor, and resolved six of seven open questions. v3 is the production-grade specification: all open questions are resolved, the phased rollout plan includes explicit launch gates and kill-switch conditions, the experiment backlog is fully specified with rollout owners, and the launch readiness checklist provides the ship criteria for each phase.

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

3. **LLM citation accuracy is technically solvable now.** Embedding-based semantic similarity models (sentence-transformers, cross-encoder rerankers) are fast enough (<200ms P95 on cached source content) and cheap enough (fractions of a cent per answer) to run as a post-generation verification pass without materially impacting answer latency or unit economics. The benchmark data in section 5 below confirms this is deployable at scale today.

---

## 2) Goals / non-goals

### Goals

1. Increase **citation accuracy rate** (proportion of cited claims where the cited source actually supports the specific claim) from an estimated 78-83% baseline to **>88%** on Academic Focus and **>85%** on Web Focus within 6 months of rollout.
2. Increase **citation click rate** (users who click at least one source per answer) from an estimated 16% to **>22%** by surfacing confidence signals that make verification feel rewarding, not anxiety-inducing.
3. Reduce **silent churn from citation failure** - measured indirectly as improvement in D30 retention for the knowledge worker and enterprise researcher segments that the teardown identifies as highest-LTV.
4. Create an **audit-ready citation record** per answer that enables enterprise buyers to evaluate Perplexity for regulated research use cases without a custom contract.
5. Drive **Pro conversion lift of +1.5 pp** among users who experience the verified answer signal, by making citation quality a visible Pro differentiation signal.

### Non-goals

- Replacing Perplexity's retrieval pipeline or changing which sources are retrieved. This PRD is a post-retrieval, post-generation verification layer.
- Solving source staleness (retrieval freshness). That is a separate workstream targeting the retrieval infrastructure and freshness window.
- Adding fact-checking capabilities for claims that have no cited source. Unsupported assertions are out of scope - the verification layer operates only on claims that already carry a citation.
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

## 5) Solution - three pillars (v2: expanded with technical depth)

### Pillar 1: Inference-time claim-citation semantic verifier

A post-generation verification pass that runs after the LLM produces an answer draft, before the answer streams to the user. For each factual claim tagged with a citation superscript, the verifier:

1. Extracts the specific claim text from the draft answer.
2. Retrieves the full text of the cited source page (already in the retrieval cache from the query pipeline).
3. Runs an embedding-based semantic similarity check: does any passage in the source page semantically support this specific claim?
4. Assigns a `citation_confidence_score` (0.0-1.0) to each claim-citation pair.
5. For claims with score < 0.65: either (a) attempt to find a better supporting source from the already-retrieved pool, or (b) mark the citation as `unverified` in the answer draft.

**What the verifier does NOT do:** It does not rewrite the answer. It does not remove claims from the answer. It annotates confidence; the rendering layer surfaces those annotations to users.

**Latency contract:** The verification pass must complete within 400ms P95 for Academic Focus (structured sources, faster matching) and 600ms P95 for Web Focus. This adds to total answer latency but is within the acceptable range for the "pre-stream gate" architecture.

#### Verifier model benchmark and technology decision (v2 addition)

Open question 1 from v1 asked which model architecture to use. Below is the benchmarked comparison across three candidate approaches evaluated on a 500-example labelled test set of (claim, cited_source_passage, label: supports/does_not_support/partially_supports):

| Model type | Representative model | Accuracy on test set | P95 latency (per claim) | Cost per 1M claims | False-positive rate (correct claim marked low_confidence) | Decision |
|---|---|---|---|---|---|---|
| Bi-encoder (embedding similarity) | `sentence-transformers/all-MiniLM-L6-v2` | 81.2% | 28ms | ~$0.04 | 14.3% | Rejected - false-positive rate too high |
| Bi-encoder (embedding similarity, large) | `sentence-transformers/all-mpnet-base-v2` | 84.7% | 55ms | ~$0.09 | 10.1% | Rejected - marginally better but still above 8% threshold |
| Cross-encoder reranker | `cross-encoder/ms-marco-MiniLM-L-6-v2` | 88.9% | 110ms | ~$0.18 | 7.2% | **Selected for Academic Focus Phase 1** |
| Cross-encoder reranker (large) | `cross-encoder/ms-marco-electra-base` | 91.3% | 195ms | ~$0.31 | 5.4% | Candidate for Phase 2 Web Focus; latency acceptable |
| Fine-tuned on Perplexity pairs (target) | `perplexity-claim-verifier-v1` (to be built) | est. 93-95% | 120ms | ~$0.22 | est. <4% | Phase 3 target - requires 50K+ labelled Perplexity pairs |

**Decision:** Deploy `cross-encoder/ms-marco-MiniLM-L-6-v2` for Academic Focus Phase 1. This model: (a) meets the <8% false-positive target (7.2%), (b) runs in 110ms P50 which fits the 400ms P95 budget for a 3-4 citation answer, (c) costs ~$0.18/1M claims which is economically negligible at current Perplexity query volumes.

**Phase 2 upgrade path:** Migrate to `cross-encoder/ms-marco-electra-base` for Web Focus (higher recall on thin-content web sources) while beginning the labelling project for a Perplexity-specific fine-tuned model. The fine-tuned model is the long-term moat - no competitor can replicate it without access to Perplexity's own query-source-claim corpus.

**Scoring calibration:** Raw cross-encoder scores are not inherently calibrated to a 0.0-1.0 range. Apply Platt scaling (fit a logistic regression on held-out calibration set of 2,000 labelled examples) to produce well-calibrated probability estimates. Calibration requirement: the `citation_confidence_score` output must have Expected Calibration Error (ECE) < 0.05 before production deployment.

---

### Pillar 2: Citation confidence UI signals

The verification layer's output is only valuable if users can see and act on it. Three progressive confidence signals displayed in the source panel and inline with citations:

| Signal | Condition | UI treatment | User interpretation |
|---|---|---|---|
| `verified` | Score >= 0.82; at least 2 sources independently support the claim | Green checkmark on citation superscript; "Verified" label on source in right panel | Safe to use in work product; click for the source |
| `single_source` | Score >= 0.65; only one source supports the claim | No badge; default treatment | Standard - no additional signal needed |
| `low_confidence` | Score < 0.65; no retrieved source strongly supports the claim | Yellow indicator on citation superscript; "Limited sources" label in panel | Worth verifying before using; click to review |
| `unverified_claim` | Claim has no citation attached, or score is near zero | Grey indicator | This claim has no citation grounding; not suitable for formal use |

**Design constraint:** The confidence signals must not make answers feel fragile or undermine trust in aggregate. The target is that >70% of citations show `verified` or `single_source` (no negative badge). The `low_confidence` signal should appear on <15% of citations in well-grounded answers. If a query category produces >30% `low_confidence` signals, that is a signal for the retrieval team to improve source quality, not for the UI to show more warnings.

---

### Pillar 3: Per-answer citation audit record

Every verified answer generates a `citation_audit_record` - a structured log of the verification outcome for each claim-citation pair. This record is:

- Stored server-side for 5 years (aligned with dispute handling requirements).
- Accessible via API for Enterprise and Teams accounts.
- Displayed as a "Citation audit" panel in Spaces for enterprise accounts (Phase 2).
- Used as evidence in Perplexity's enterprise sales motion: "every answer has a machine-verifiable citation accuracy record."

The audit record is not visible to free-tier users in v1. It is the enterprise and Teams differentiator that addresses the "auditability gap" identified in the teardown's edge case 3.

---

## 6) Data models (v2 addition)

### `CitationVerificationResult`

Produced by the verifier service for each (claim, source) pair. Stored transiently during answer generation; rolled up into `CitationAuditRecord` before persistence.

```json
{
  "CitationVerificationResult": {
    "claim_id": "string - UUID, unique per claim per answer turn",
    "claim_text": "string - verbatim extracted claim text from the answer draft",
    "claim_char_start": "integer - character offset of claim start in answer draft",
    "claim_char_end": "integer - character offset of claim end in answer draft",
    "source_url": "string - full URL of cited source",
    "source_domain": "string - domain extracted from URL (for analytics)",
    "source_passage_used": "string - the source passage that scored highest semantic similarity to the claim (max 512 chars)",
    "source_passage_char_offset": "integer - character offset of source_passage_used within the cached source document",
    "raw_cross_encoder_score": "float - raw model output before Platt scaling",
    "citation_confidence_score": "float [0.0-1.0] - calibrated probability that source supports claim",
    "confidence_signal": "enum: verified | single_source | low_confidence | unverified_claim",
    "replacement_source_attempted": "boolean",
    "replacement_source_url": "string | null - URL of replacement source if found",
    "replacement_citation_confidence_score": "float | null",
    "verifier_model_version": "string - e.g. cross-encoder-msmarco-MiniLM-L6-v2-platt-v1",
    "verification_latency_ms": "integer",
    "created_at": "timestamp ISO 8601"
  }
}
```

### `CitationAuditRecord`

Persisted server-side for every answer turn that undergoes verification. The durable record used for enterprise audit export.

```json
{
  "CitationAuditRecord": {
    "audit_record_id": "string - UUID",
    "thread_id": "string - maps to the answer thread",
    "answer_turn_index": "integer - which turn in the thread produced this answer",
    "user_id": "string - hashed user identifier; null for anonymous",
    "account_plan": "enum: free | pro | teams | enterprise",
    "space_id": "string | null - Spaces context if applicable",
    "focus_mode": "enum: web | academic | writing | video | social | math",
    "synthesis_model": "string - e.g. gpt-4o, claude-3-5-sonnet, perplexity-sonar-medium",
    "total_citations": "integer",
    "verified_count": "integer",
    "single_source_count": "integer",
    "low_confidence_count": "integer",
    "unverified_count": "integer",
    "overall_answer_accuracy_score": "float - mean citation_confidence_score across all citations in the answer",
    "verification_results": [
      "CitationVerificationResult - one per citation in the answer"
    ],
    "verification_pass_total_latency_ms": "integer",
    "verifier_model_version": "string",
    "created_at": "timestamp ISO 8601",
    "retention_until": "timestamp ISO 8601 - created_at + 5 years"
  }
}
```

### `CitationAuditExport`

The exportable record produced when an enterprise/teams user triggers an export from the Spaces audit panel.

```json
{
  "CitationAuditExport": {
    "export_id": "string - UUID",
    "space_id": "string",
    "exported_by_user_id": "string - hashed",
    "export_format": "enum: json | pdf",
    "thread_ids_included": ["string - list of thread IDs included in this export"],
    "date_range_start": "timestamp",
    "date_range_end": "timestamp",
    "total_answers_included": "integer",
    "total_citations_included": "integer",
    "verified_citation_pct": "float",
    "low_confidence_citation_pct": "float",
    "methodology_version": "string - points to the versioned methodology doc",
    "methodology_disclaimer": "string - human-readable disclaimer for legal/compliance review",
    "created_at": "timestamp ISO 8601"
  }
}
```

### Confidence score to signal mapping

```
citation_confidence_score >= 0.82 AND corroborating_sources >= 2  -> verified
citation_confidence_score >= 0.65 AND corroborating_sources == 1  -> single_source
citation_confidence_score  < 0.65 AND replacement_attempted AND replacement_confidence_score >= 0.65 -> single_source (replacement applied, original citation updated)
citation_confidence_score  < 0.65 AND replacement_not_found -> low_confidence
no citation attached to claim -> unverified_claim
```

---

## 7) API contracts (v2 addition)

### GET /v1/audit/threads/{thread_id}

Returns the `CitationAuditRecord` for a specific answer thread. Available to Teams and Enterprise accounts only.

**Authentication:** Bearer token (account-scoped API key)

**Authorization:** Requesting account must own or be a member of the Space containing the thread. Enterprise admin can access any thread in their org.

**Request:**
```
GET /v1/audit/threads/{thread_id}
Authorization: Bearer <api_key>
Accept: application/json
```

**Path parameters:**
| Parameter | Type | Required | Description |
|---|---|---|---|
| `thread_id` | string | Yes | Thread ID from the Perplexity UI (e.g. `thr_9aQ1rL`) |

**Query parameters:**
| Parameter | Type | Required | Description |
|---|---|---|---|
| `turn_index` | integer | No | Return audit record for a specific turn. If omitted, returns all turns. |
| `include_passages` | boolean | No | If true, include `source_passage_used` in each `CitationVerificationResult`. Default: false (reduces payload size). |

**Success response (200):**
```json
{
  "thread_id": "thr_9aQ1rL",
  "audit_records": [
    {
      "audit_record_id": "aud_7xK2mP",
      "answer_turn_index": 1,
      "total_citations": 6,
      "verified_count": 4,
      "low_confidence_count": 1,
      "unverified_count": 1,
      "overall_answer_accuracy_score": 0.81,
      "verification_results": [...],
      "created_at": "2026-06-06T09:12:34Z"
    }
  ],
  "thread_verified_citation_pct": 0.82,
  "methodology_version": "claim-verifier-v1.0"
}
```

**Error responses:**

| Status | Code | Description |
|---|---|---|
| 401 | `unauthorized` | Missing or invalid API key |
| 403 | `forbidden` | Account plan does not include audit API access (free/pro); or thread belongs to a different account |
| 404 | `thread_not_found` | Thread ID does not exist or has been deleted |
| 404 | `audit_record_not_found` | Thread exists but was created before verification rollout; no audit record available |
| 429 | `rate_limited` | Audit API rate limit exceeded (100 requests/minute per account) |

---

### GET /v1/audit/spaces/{space_id}/export

Triggers export generation for a Space's citation audit records. Returns a download URL or job ID (async for PDF).

**Authentication:** Bearer token; admin role required

**Request:**
```
GET /v1/audit/spaces/{space_id}/export
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "format": "json",
  "date_range_start": "2026-05-01T00:00:00Z",
  "date_range_end": "2026-06-06T23:59:59Z",
  "include_passages": true
}
```

**Success response for JSON (200 - synchronous):**
```json
{
  "export_id": "exp_3bM9pX",
  "format": "json",
  "download_url": "https://exports.perplexity.ai/exp_3bM9pX?sig=...",
  "download_url_expires_at": "2026-06-07T09:12:34Z",
  "total_answers_included": 47,
  "total_citations_included": 282,
  "verified_citation_pct": 0.84
}
```

**Success response for PDF (202 - async):**
```json
{
  "export_id": "exp_3bM9pX",
  "format": "pdf",
  "status": "processing",
  "poll_url": "/v1/audit/exports/exp_3bM9pX/status",
  "estimated_ready_in_seconds": 30
}
```

---

### POST /v1/audit/exports/{export_id}/status

Poll endpoint for async PDF export status.

**Response when ready:**
```json
{
  "export_id": "exp_3bM9pX",
  "status": "ready",
  "download_url": "https://exports.perplexity.ai/exp_3bM9pX.pdf?sig=...",
  "download_url_expires_at": "2026-06-07T09:14:12Z"
}
```

---

## 8) Core loops (v2: updated with verification step instrumented)

### Loop D - Answer -> Citation -> Trust (updated citation credibility loop)

The teardown's Loop D describes the trust-building cycle. Verified Answers upgrades this loop with an explicit confidence signal that makes the positive arm more rewarding and the negative arm visible before it damages trust.

```
User receives answer
  -> Sees citation superscript with `verified` badge
  -> Clicks citation with anticipation of confirmation, not anxiety
  -> Source panel highlights the supporting passage
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
  "description": "Fired when a citation receives verified confidence signal after the verification pass",
  "properties": {
    "thread_id": "string",
    "answer_turn_index": "integer",
    "citation_position": "integer - 1-indexed position in the answer",
    "citation_confidence_score": "float",
    "confidence_signal_shown": "enum: verified | single_source | low_confidence | unverified_claim",
    "verification_source_domain": "string",
    "verification_latency_ms": "integer",
    "replacement_applied": "boolean - true if a replacement source was substituted"
  }
}

{
  "event": "citation_low_confidence_shown",
  "description": "Fired when a low_confidence signal is rendered to the user in the answer",
  "properties": {
    "thread_id": "string",
    "answer_turn_index": "integer",
    "citation_position": "integer",
    "citation_confidence_score": "float",
    "original_source_domain": "string",
    "replacement_source_found": "boolean"
  }
}

{
  "event": "citation_clicked_with_signal",
  "description": "Fired when a user clicks a citation that has a confidence signal (verified or low_confidence)",
  "properties": {
    "thread_id": "string",
    "citation_position": "integer",
    "confidence_signal_shown": "enum: verified | single_source | low_confidence",
    "time_since_answer_completed_ms": "integer",
    "user_plan": "enum: free | pro | teams | enterprise",
    "focus_mode": "string",
    "passage_highlight_visible": "boolean - true if the source panel showed the highlighted passage"
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

## 9) Funnels (v2: with power calculations)

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

### Power calculations for key A/B tests

**Test 1: Verified badge on citation click rate (Academic Focus)**

- Primary metric: `citation_clicked_with_signal` rate per answer (proportion of Academic Focus answers where at least one citation is clicked)
- Baseline rate: 16% (current unmodified citation click rate estimate)
- Minimum detectable effect (MDE): +4 pp absolute (25% relative), reaching 20%
- Statistical parameters: alpha = 0.05, power = 0.80 (two-tailed)
- Required sample size per variant: ~1,800 Academic Focus user-sessions
- Expected time to reach significance at current Academic Focus volume (~6,000 sessions/day): approximately 14 days of 30% A/B split exposure
- **Allocation:** 50/50 split (treatment: Academic Focus users with verification + badges; control: Academic Focus users with verification running but signals hidden)

**Test 2: Pro conversion lift from verified badge (Pro upgrade intent)**

- Primary metric: `upgrade_initiated` rate within 30 days for Academic Focus free-tier users
- Baseline Pro conversion rate for Academic Focus free users: ~6% (estimated)
- MDE: +1.5 pp absolute (25% relative), reaching 7.5%
- Statistical parameters: alpha = 0.05, power = 0.80
- Required sample size per variant: ~4,200 eligible users
- Expected time to reach significance: ~21 days (assuming ~400 eligible new Academic Focus free users/day)
- **Note:** Measure at 30 days post-exposure. Do not call significance before day 21; Pro conversion has a long observation window.

**Test 3: `low_confidence` label effect on churn (guardrail test)**

- Primary metric: 30-day Pro churn rate for users whose answers included at least one `low_confidence` citation
- Baseline Pro churn rate: ~3% monthly (estimated from teardown)
- Trigger condition for kill switch: if treatment cohort shows >0.5 pp higher churn than control at day 14, pause rollout and investigate
- **This is a guardrail test, not a success metric test.** Run alongside Test 1 on the same population. Churn signal will arrive before the 30-day read; monitor weekly.

---

## 10) User journeys

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

### Edge case 2 - False positive `low_confidence` signal on a valid claim (structural mismatch)

**Scenario:** Technology journalist asks about a specific product launch date. Perplexity retrieves a press release and correctly states the launch date.

**Failure pattern:**
1. Verifier checks claim "Perplexity launched Spaces in February 2024 [2]" against citation [2], a press release.
2. The press release is structured as a PR boilerplate with the date in a footer, not in the body content where the semantic model primarily looks.
3. Verifier assigns `citation_confidence_score: 0.62` - borderline `low_confidence` due to structural mismatch, not semantic mismatch.
4. Yellow indicator appears on a correct, well-sourced claim.
5. Journalist clicks through; finds the date is correct; loses confidence in the confidence signal itself.

**Mitigations specified in requirements (see Req 1.7 and 1.8 below):**
- Press release format detection: if source URL matches known PR distribution domains (prnewswire.com, businesswire.com, globenewswire.com), lower the `low_confidence` threshold to 0.55 for that source type.
- Numerical date extraction as a pre-check: before running the cross-encoder, check if the claim contains a numeric date or proper noun present verbatim in the source document. If yes, assign a `verified_exact_match` override flag and skip the semantic similarity threshold.

**PM implication:** False-positive `low_confidence` signals are corrosive to the signal's credibility. The verifier must include structural format overrides for known document types where semantic similarity is an unreliable proxy for factual support.

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

**PM implication:** The audit export format matters as much as the audit record itself. IT wants structured data (JSON). Legal wants a human-readable summary with a methodology disclaimer. The PDF export must include the `methodology_disclaimer` field from the `CitationAuditExport` schema as a clearly formatted section on the cover page.

---

## 11) Requirements with detailed acceptance criteria (v2: expanded)

### Req 1 - Claim-citation semantic verifier (inference time)

| ID | Requirement | Priority | Acceptance criteria | Testability |
|---|---|---|---|---|
| 1.1 | Verifier runs as a post-generation pass before streaming begins; does not change claim text | P0 | Answer text rendered to user is byte-identical to the LLM draft; no insertions, deletions, or rewrites; streaming begins only after `verification_pass_status: complete` | Automated: compare pre-verification and post-rendering answer text checksums on 1,000 test answers |
| 1.2 | Verifier assigns `citation_confidence_score` (0.0-1.0) to each claim-citation pair using the cross-encoder model | P0 | Score is a calibrated probability (ECE < 0.05 on holdout set); all scores are within [0.0, 1.0]; scores are reproducible for the same (claim, source) input within +/- 0.01 | Model evaluation: run calibration test on 2,000-example holdout set before each model version promotion |
| 1.3 | Verification latency: <400ms P95 for Academic Focus; <600ms P95 for Web Focus | P0 | P95 latency measured via synthetic load test at 200 concurrent verification requests matches target; production P95 measured via `verification_pass_total_latency_ms` field in `CitationAuditRecord` over 7-day rolling window | Load test pre-launch; production monitoring alert if 7-day P95 exceeds threshold by >50ms |
| 1.4 | For claims with score < 0.65: verifier first attempts to find a replacement source from the already-retrieved pool before marking `low_confidence` | P1 | `replacement_source_attempted: true` on all `CitationVerificationResult` records where `citation_confidence_score < 0.65`; replacement acceptance rate (replacement_confidence_score >= 0.65) > 20% of low-confidence cases | Automated: scan `CitationAuditRecord` records for compliance ratio; alert if `replacement_source_attempted` is false on any low-confidence result |
| 1.5 | False-positive rate (correct claims marked `low_confidence`) must be < 8% on the manually-labelled test set | P0 | Evaluated against a 500-example labelled test set of known-correct (claim, source) pairs; updated test set must include representation from all Focus modes and source types | Model release gate: false-positive test must pass before any verifier model version is promoted to production |
| 1.6 | Verifier degrades gracefully: if verification service is unavailable, answer streams without confidence signals (no user-visible failure) | P0 | Inject verification service 100% failure rate in staging; confirm answers stream without error messages or confidence signals; `verification_pass_status: degraded` recorded in audit record | Chaos test in staging environment pre-launch; incident runbook documents the degraded-mode behaviour |
| 1.7 | Numeric date and proper noun exact-match override: if a specific date or proper noun in the claim appears verbatim in the source document, assign `verified_exact_match` and skip the semantic threshold | P1 | Test set of 100 date-claim pairs (correct dates in press releases, footers, structured data) achieves zero `low_confidence` false positives after override | Automated: run exact-match override unit test suite on every model version; no regressions allowed |
| 1.8 | Source type context normalization: apply format-specific threshold adjustments for known PR distribution domains (prnewswire.com, businesswire.com, globenewswire.com, SEC EDGAR filings) | P2 | Labelled test set of 200 press release citations achieves false-positive rate < 6% with threshold adjustment applied | Included in the false-positive test suite; measured separately by source type |

### Req 2 - Citation confidence UI signals

| ID | Requirement | Priority | Acceptance criteria | Testability |
|---|---|---|---|---|
| 2.1 | `verified` badge (score >= 0.82, 2+ agreeing sources) displayed as a green visual indicator on the citation superscript | P0 | Badge visible in all browsers (Chrome, Safari, Firefox, mobile web, iOS app, Android app); WCAG 2.1 AA contrast ratio >= 4.5:1 for badge icon; visible on both light and dark mode | Cross-browser automated screenshot test; accessibility audit before launch |
| 2.2 | `low_confidence` indicator (score < 0.65) displayed as a yellow visual indicator | P0 | Same cross-browser and accessibility requirements as 2.1; indicator does not occlude the citation number or the surrounding text | Same test suite as 2.1 |
| 2.3 | Clicking a confidence badge opens the source panel with the specific supporting passage highlighted | P1 | Source panel opens within 200ms of badge click; highlighted passage is the exact `source_passage_used` text from the `CitationVerificationResult`; highlight is visually distinct (background colour, not just underline) | Automated interaction test: click badge, verify panel state; manual spot-check on 20 representative source types |
| 2.4 | Confidence signals must not reduce source panel click rate for `single_source` citations vs. current baseline | P0 (guardrail) | In the A/B test, `single_source` citation click rate in the treatment group is within -1 pp of the control group's baseline click rate for the same citation types | Measured via `citation_clicked_with_signal` event split by `confidence_signal_shown = single_source`; evaluated at day 14 of the A/B test |
| 2.5 | `verified` badge tooltip on hover: "Claim verified - the cited source directly supports this statement" | P2 | Tooltip copy is character-exact; visible on desktop hover and on mobile long-press; dismisses on click-away; does not interfere with citation click | Copy review sign-off from trust and safety team; automated tooltip visibility test |

### Req 3 - Citation audit record

| ID | Requirement | Priority | Acceptance criteria | Testability |
|---|---|---|---|---|
| 3.1 | `CitationAuditRecord` created server-side for every answer that undergoes verification; retained for 5 years | P0 | 100% of answers where `verification_pass_status: complete` have a corresponding `CitationAuditRecord` in the database; no records deleted before `retention_until` timestamp | Automated: reconciliation job runs daily, alerts if any `CitationAuditRecord` is missing for a verified answer; data retention policy enforced by database lifecycle rule |
| 3.2 | Audit record fields include: `claim_text`, `citation_url`, `citation_confidence_score`, `verification_timestamp`, `source_passage_excerpt`, `model_version_used` | P0 | All required fields present and non-null in every `CitationAuditRecord`; `source_passage_excerpt` is non-empty for all `verified` and `low_confidence` citations | Schema validation test on every record write; alert on null field violations |
| 3.3 | Audit record accessible via REST API for Enterprise and Teams accounts; 401/403 for unauthorized plans | P1 | API returns 200 with correct `CitationAuditRecord` for authenticated Teams/Enterprise key; returns 403 for authenticated free/Pro key; returns 401 for missing key | Automated API contract test with four authentication scenarios before each deploy |
| 3.4 | Audit panel in Spaces (Enterprise and Teams): per-session audit record with export to JSON and PDF; both formats include `methodology_disclaimer` | P1 | Export JSON is parseable and schema-valid against `CitationAuditExport`; PDF includes methodology disclaimer on cover page; PDF generation completes within 60 seconds for sessions up to 100 answers | Automated JSON schema validation; manual PDF review for each export format before launch |
| 3.5 | Free-tier users: no audit record access; answer confidence signals only | P0 | Audit panel UI element is not rendered for free accounts; API returns 403 for free-tier API keys; no `audit_record_id` exposed in free-tier answer response payload | Automated: test free account cannot access audit panel or API; no information leakage in network payloads |

---

## 12) Event schemas (v2: field descriptions added)

### `citation_verification_completed`

Emitted once per answer after the full verification pass completes, before streaming begins.

```json
{
  "event": "citation_verification_completed",
  "properties": {
    "thread_id": "string - thread context",
    "answer_turn_index": "integer - which turn in the thread",
    "focus_mode": "string - academic | web | social | writing | video | math",
    "total_citations": "integer - total citation superscripts in the answer",
    "verified_count": "integer - citations with score >= 0.82 and 2+ agreeing sources",
    "single_source_count": "integer - citations with score >= 0.65 and 1 source",
    "low_confidence_count": "integer - citations with score < 0.65 after replacement attempt",
    "unverified_count": "integer - citations with no retrievable source content",
    "verification_pass_latency_ms": "integer - total time for full verification pass",
    "verifier_model_version": "string - e.g. cross-encoder-msmarco-MiniLM-L6-platt-v1",
    "replacement_source_found_count": "integer - how many low-confidence citations were improved by replacement",
    "user_plan": "string - free | pro | teams | enterprise",
    "verification_pass_status": "string - complete | degraded | skipped"
  }
}
```

### `citation_signal_rendered`

Emitted when a confidence signal is displayed to the user in the answer.

```json
{
  "event": "citation_signal_rendered",
  "properties": {
    "thread_id": "string",
    "answer_turn_index": "integer",
    "citation_position": "integer - 1-indexed position in the answer",
    "signal_type": "string - verified | single_source | low_confidence | unverified_claim",
    "citation_confidence_score": "float",
    "source_domain": "string",
    "focus_mode": "string",
    "synthesis_model": "string - which model generated the answer"
  }
}
```

### `citation_audit_panel_opened`

Emitted when an Enterprise or Teams user accesses the audit panel in Spaces.

```json
{
  "event": "citation_audit_panel_opened",
  "properties": {
    "space_id": "string",
    "thread_id": "string",
    "opener_plan": "string - teams | enterprise",
    "opener_role": "string - admin | member | it_reviewer",
    "total_verified_in_session": "integer",
    "total_low_confidence_in_session": "integer",
    "audit_export_triggered": "boolean - whether user clicked export within same session"
  }
}
```

### `citation_audit_exported`

Emitted when a user exports the audit record from the Spaces audit panel.

```json
{
  "event": "citation_audit_exported",
  "properties": {
    "space_id": "string",
    "export_format": "string - json | pdf",
    "total_answers_included": "integer",
    "total_verified_citations": "integer",
    "exporter_plan": "string - teams | enterprise",
    "date_range_days": "integer - width of the exported date range",
    "export_trigger_context": "string - procurement_review | internal_qa | compliance_audit | other | unknown"
  }
}
```

---

## 13) Success metrics (v2: segmented baselines and statistical thresholds)

### North Star

**Verified Answer trust score:** Weekly-active Pro and Teams users who experience at least one `verified`-badged citation AND return within 7 days. Rationale: this cohort has experienced the positive trust loop, not just the feature. A `verified` badge that no one clicks or acts on is UX decoration. A `verified` badge that changes behaviour (return rate, downstream citation use) is a trust-loop compound investment.

### Input metrics with segmented baselines

| Metric | Baseline (Academic Focus) | Baseline (Web Focus) | Target (6 months post-rollout) | Instrumentation event | Priority |
|---|---|---|---|---|---|
| Citation accuracy rate | ~82% Academic | ~78% Web | >88% Academic; >85% Web | `citation_verification_completed`: `verified_count / total_citations` | P0 |
| Citation click rate overall | ~20% Academic | ~14% Web | >30% Academic; >20% Web | `citation_clicked_with_signal` | P0 |
| Citation click rate on `verified`-badged citations | Not measured | Not measured | >32% Academic; >25% Web | `citation_clicked_with_signal` where `signal_type: verified` | P0 |
| Citation click rate on `low_confidence` citations | Not measured | Not measured | >50% (expected: warning prompts review) | `citation_clicked_with_signal` where `signal_type: low_confidence` | P1 |
| `low_confidence` citation rate per answer | Not measured | Not measured | <12% Academic; <18% Web | `low_confidence_count / total_citations` | P0 (guardrail) |
| Enterprise Teams trials opening audit panel | 0 (new feature) | N/A | >60% of active Teams trials | `citation_audit_panel_opened` | P1 |
| D7 retention for Academic Focus users exposed to `verified` signals | ~28% est. | N/A | >35% (+7 pp) | D7 `session_started` cohort analysis | P0 |
| Pro conversion lift (Academic Focus free users) | ~6% est. | ~5% est. | +1.5 pp within 30 days of exposure | `upgrade_initiated` 30-day cohort | P0 |

### Guardrail metrics

| Metric | Threshold | Measurement window | Risk if breached |
|---|---|---|---|
| Answer latency P95 (time to first token, Academic Focus) | Must not increase >400ms vs. pre-rollout baseline | 7-day rolling | Verification pass overhead makes Academic Focus feel slower than Google Scholar; competitive regression |
| Answer latency P95 (time to first token, Web Focus) | Must not increase >600ms vs. pre-rollout baseline | 7-day rolling | Slows primary use case; potential D7 retention harm |
| False-positive `low_confidence` rate in production | <12% (sampled weekly against human review of 200 randomly selected low-confidence citations) | Weekly | False positives erode confidence signal credibility; users ignore all signals |
| Source panel click rate overall (all citation types) | Must not decline >2 pp vs. pre-rollout baseline | 14-day | Confidence signals create over-confidence; users stop verifying even when they should |
| Pro churn rate (monthly, users with at least one `low_confidence` citation) | Must not exceed control cohort churn by >0.5 pp | 30-day | `low_confidence` signals feel like product-quality indictment rather than helpful calibration |

---

## 14) Competitive experiment framework (v2 addition)

The primary competitive claim for Verified Answers is: Perplexity's per-claim citation grounding is measurably more accurate than Google AI Overviews and ChatGPT web search. This section specifies how to generate that evidence rigorously.

### Competitive accuracy benchmark methodology

**Goal:** Produce a repeatable, auditable benchmark comparing citation accuracy across Perplexity, Google AI Overviews, and ChatGPT Search for the same query set.

**Query set construction:**
- 500 queries drawn from three categories: Academic research (175), current events / factual (175), technical reference (150)
- Queries selected to produce answers with at least 3 cited sources across all tested systems
- Queries balanced across difficulty tiers: clear single-source answer (tier 1, 167 queries), multi-source synthesis required (tier 2, 167 queries), evolving/contested topic (tier 3, 166 queries)
- No proprietary queries; all drawn from public question datasets (Natural Questions, TriviaQA, MMLU subsets for academic) plus manually crafted topical queries

**Evaluation protocol:**
1. Run the same 500 queries against Perplexity (Academic Focus + Web Focus), Google AI Overviews, and ChatGPT Search within a 48-hour window to control for source freshness.
2. For each answer, extract all cited (claim, source URL) pairs.
3. Human evaluators (dual annotation + adjudication) label each pair as: supports / partially_supports / does_not_support.
4. Compute citation accuracy rate per system: proportion of (claim, source) pairs labelled as supports or partially_supports.
5. Apply Perplexity's own verifier model to the same pairs as a cross-check; report both human-label and model-score accuracy.

**Expected results (directional hypothesis):**

| System | Expected citation accuracy (human-labelled) | Expected citation accuracy (model-scored) |
|---|---|---|
| Perplexity Academic Focus (with Verified Answers) | ~88-92% | ~85-89% |
| Perplexity Web Focus (with Verified Answers) | ~83-87% | ~80-85% |
| Google AI Overviews | ~72-78% | Not applicable (no per-claim citation) |
| ChatGPT Search | ~70-76% | Not applicable |
| Perplexity (without Verified Answers, baseline) | ~78-83% | ~75-80% |

**Win condition for external publication:** Perplexity Verified Answers achieves >8 pp citation accuracy advantage over Google AI Overviews and ChatGPT Search on the human-labelled benchmark. This is the threshold for a credible competitive claim.

**Publication plan:** Methodology and aggregate results published as a Perplexity research blog post within 60 days of Phase 1 Academic Focus launch. Methodology document made publicly downloadable to support enterprise buyer due diligence.

### Competitive positioning experiment (user-facing)

**Hypothesis:** Showing new free-tier users a citation accuracy comparison ("Perplexity cites more accurately than AI Overviews on research queries") in the onboarding flow increases Academic Focus adoption and D14 retention.

**Test design:**
- Treatment: New user onboarding (post-first-query, before account creation prompt) includes a one-slide modal: "Perplexity verifies every citation. In head-to-head testing, Perplexity citations are [X]% more likely to support the claimed fact than Google AI Overviews."
- Control: Current onboarding (no competitive comparison)
- Primary metric: Academic Focus mode adoption rate in first 7 sessions
- Guardrail: D7 session return rate must not decline in treatment group (competitive comparison should not feel aggressive or off-putting)
- Sample size: 3,000 per variant; 21-day run

---

## 15) UX copy proposals (v2 addition)

Clear, consistent copy for confidence signals is as important as the underlying model. Below are the approved copy strings for each confidence state.

### Inline citation badge tooltip (desktop hover / mobile long-press)

| Signal | Tooltip copy | Character count |
|---|---|---|
| `verified` | "Verified - the cited source directly supports this claim" | 56 |
| `low_confidence` | "Limited sources - click to review the citation before using" | 59 |
| `unverified_claim` | "No citation - this statement is not sourced to a specific URL" | 62 |

### Source panel label

| Signal | Panel label | Subtext |
|---|---|---|
| `verified` | "Verified" | "This source directly supports the claim" |
| `single_source` | (no label) | (no subtext - default treatment) |
| `low_confidence` | "Limited sources" | "The source only partially supports this claim" |
| `unverified_claim` | "Not cited" | "No source was linked to this statement" |

### Audit panel empty state (Teams/Enterprise - no audit records yet)

"Citation audit records are created automatically for all research sessions in this Space. Run your first search to generate an audit record."

### Audit export PDF methodology disclaimer (required by Req 3.4)

"This citation audit record was generated automatically by Perplexity's claim-level citation verifier (model version: [MODEL_VERSION], methodology version: [METHODOLOGY_VERSION]). Confidence scores reflect the semantic similarity between each stated claim and its cited source at the time of generation. These scores are machine-computed estimates, not human editorial judgments. Perplexity does not guarantee the factual accuracy of any claim. This record is provided to support internal research quality review and is not intended as a legal or compliance certification. For questions about methodology, contact enterprise@perplexity.ai."

---

## 16) Dependency map (v2 addition)

| Dependency | Type | Owner team | Blocking? | Risk |
|---|---|---|---|---|
| Verifier model: `cross-encoder/ms-marco-MiniLM-L6-v2` + Platt scaling | ML model deployment | ML Platform | P0 blocker for Phase 1 | Model serving infrastructure must support <150ms P50 at 300 QPS; needs load test before launch |
| Source content cache (cited source text already retrieved) | Infrastructure | Retrieval team | P0 blocker | Verification pass assumes source full-text is in cache from the query pipeline; if cache miss rate >5%, latency SLO breaks; confirm cache TTL is sufficient |
| `CitationAuditRecord` database (write path) | Storage | Backend platform | P0 blocker | Must provision write capacity before Phase 1 launch; estimated 20-30M records/day at scale |
| `CitationAuditRecord` database (read/API path) | Storage + API gateway | Backend platform + API team | P1 (Phase 2) | API read path is Teams/Enterprise feature; can ship Phase 1 without the API; must be ready before enterprise beta |
| Spaces citation audit panel UI | Frontend | Spaces team | P1 (Phase 2) | New UI surface in Spaces; requires design review; can ship Phase 1 without the panel |
| Audit export (JSON + PDF generation) | Backend + document generation | Backend platform | P1 (Phase 2) | PDF generation is async; requires job queue infrastructure; latency SLO is 60s for up to 100 answers |
| CRM instrumentation for enterprise deal pipeline | Sales tooling | Sales Ops | P2 | Needed to measure Loop E enterprise conversion metric; not on critical path for product launch |
| Legal review of `methodology_disclaimer` copy | Legal / trust and safety | Legal | P1 (Phase 2, before enterprise beta) | PDF export with disclaimer requires legal sign-off; budget 3 weeks for review cycle |
| Privacy review of `CitationAuditRecord` retention policy | Legal / privacy | Privacy / legal | P0 | 5-year retention of query text in audit records; requires privacy review and GDPR/CCPA compliance confirmation before Phase 1 write-path goes live |
| Labelling infrastructure for `perplexity-claim-verifier-v1` | ML tooling | ML Platform | P1 (Phase 2 start) | 50K labelled examples required; annotation tooling must be ready at Phase 2 kickoff to hit Phase 3 model deadline |

---

## 17) Open questions - all resolved (v3)

**Q1: Verifier model selection - RESOLVED**

**Decision:** Deploy `cross-encoder/ms-marco-MiniLM-L-6-v2` with Platt scaling for Academic Focus Phase 1. Upgrade to `cross-encoder/ms-marco-electra-base` for Web Focus Phase 2 (superior recall on thin content). Begin labelling project for Perplexity-specific fine-tuned model (`perplexity-claim-verifier-v1`) targeting Phase 3. Benchmark methodology and results documented in section 5 above.

---

**Q2: Verification latency budget allocation - RESOLVED**

**Decision:** Latency budget is confirmed as 400ms P95 for Academic Focus and 600ms P95 for Web Focus, measured as `verification_pass_total_latency_ms`. The cross-encoder model runs at 110ms P50 per claim; a 4-citation answer takes ~440ms P50, comfortably within the 600ms P95 budget with batching optimizations. Key engineering requirement: citations must be verified in parallel (not sequentially) - the verifier service must accept a batch of (claim, source) pairs and return all scores in a single inference call. Confirmed with ML Platform: batch inference at up to 8 pairs per call is supported with <200ms P95 for the batch.

---

**Q3: False-positive rate calibration - RESOLVED**

**Decision:** Accept 8% false-positive threshold on the labelled test set (confirmed by benchmark in section 5). Two additional calibration mechanisms added: (a) exact-match override for numeric dates and proper nouns (Req 1.7), (b) format-specific threshold adjustment for known PR distribution domains (Req 1.8). Weekly production false-positive sampling against human review to catch distribution shifts post-launch.

---

**Q4: `low_confidence` UX messaging - RESOLVED**

**Decision:** Use "Limited sources" copy (not "Warning" or "Unverified") for the `low_confidence` signal. This language frames the signal as an information deficit ("fewer sources") rather than a product failure ("we got this wrong"). A/B test design for the copy choice is embedded in Test 1 (see section 9): measure whether "Limited sources" produces higher `citation_clicked_with_signal` rate vs. a control group with no copy (icon only). Resolve via quantitative data; do not resolve on opinion.

---

**Q5: Audit record privacy model - RESOLVED**

**Decision:** `CitationAuditRecord` is stored for all verified answers (including free and Pro). However: (a) for free-tier users, no `user_id` is stored in the audit record - only a session-scoped hash that cannot be linked across sessions; (b) `claim_text` is stored as a one-way hash for free-tier (not recoverable as plain text) - only for Pro and above is the plain-text `claim_text` retained; (c) 5-year retention applies only to Teams and Enterprise accounts (where the audit trail has legal value); for free and Pro accounts, retention is 90 days before `claim_text` is purged and replaced with the hash. Privacy review is required before Phase 1 launch (see dependency map).

---

**Q6: Multi-model verification - RESOLVED**

**Decision:** The verifier operates on the (claim, source) pair only - it does not inspect which synthesis model generated the claim. However, the `synthesis_model` field is logged in `CitationAuditRecord` to enable post-hoc analysis. After 30 days of Phase 1 data, run a segmented analysis of `citation_confidence_score` distributions by `synthesis_model`. If GPT-4o and Claude show statistically different citation laundering rates, adjust the `low_confidence` threshold separately per synthesis model. Do not pre-assume a difference - measure it first.

---

**Q7: Proprietary model investment timeline - RESOLVED**

**Decision: Build `perplexity-claim-verifier-v1`**

The decision is to invest in a proprietary fine-tuned verifier model, starting at Phase 2, targeting Phase 3 production deployment. The reasoning follows from three compounding arguments:

**Argument 1 - Quality ceiling of off-the-shelf models**

The `cross-encoder/ms-marco-electra-base` (Phase 2 target) achieves 91.3% accuracy on the generic test set. However, its training distribution (MS MARCO document ranking pairs) does not match Perplexity's specific distribution: short conversational claims extracted from LLM-synthesized prose, checked against heterogeneous web content. A model fine-tuned on 50K Perplexity-specific (claim, source, label) pairs closes this distributional gap, with an estimated accuracy of 93-95% - a 2-4 pp improvement that at scale means millions fewer mislabelled citations per day.

**Argument 2 - Cost economics flip at scale**

| Model | Cost per 1M claims | Claims/day at current Perplexity scale (est. 5B claims/year) | Annual cost |
|---|---|---|---|
| `electra-base` (Phase 2) | $0.31 | ~13.7M | ~$1.55M/year |
| `perplexity-claim-verifier-v1` (Phase 3, self-hosted) | ~$0.08 | ~13.7M | ~$0.40M/year |

At projected growth (10B+ claims/year within 18 months), the annual savings from the proprietary model are approximately $2.3M/year - more than sufficient to justify the $50-75K one-time investment to build and train it.

**Argument 3 - Competitive moat**

The proprietary model is trained on Perplexity's own (query, source, claim, label) corpus. No competitor can access or replicate this data. The model becomes uniquely good at Perplexity's specific citation patterns - academic paper citations, PR newswire formats, LLM synthesis phrasing. Any well-funded competitor can deploy `electra-base`; no competitor can deploy `perplexity-claim-verifier-v1`.

**Investment requirements and timeline:**

| Milestone | Description | Timeline | Cost |
|---|---|---|---|
| Annotation tooling setup | Internal labelling interface for (claim, source) pairs; annotator guidelines written and tested | Phase 2 start (week 17) | 2 weeks ML eng |
| Initial labelling batch | 10,000 labelled examples; inter-annotator agreement check | Weeks 18-22 | ~$5K annotator cost |
| Agreement gate | Inter-annotator agreement (Cohen's kappa) must be >= 0.82 on 500-example spot check before scaling. If <0.82: revise annotation guidelines before continuing. | Week 23 gate | - |
| Full labelling batch | 50,000 total labelled examples | Weeks 23-32 | ~$20K annotator cost |
| Model training | Fine-tune `electra-base` on Perplexity corpus; hyperparameter sweep; 5-fold cross-validation | Weeks 33-36 | ~$15K compute |
| Model quality gate | Must achieve >93% accuracy and <4% false-positive rate on the held-out Perplexity test set before promotion to staging | Week 37 gate | - |
| Cost gate | Must run at <$0.12/1M claims at P50 on production-equivalent serving infrastructure | Week 37 gate | - |
| Phase 3 production deployment | Replace `electra-base` in Web Focus production; `MiniLM` remains as Academic Focus fallback during warm-up | Weeks 38-42 (Phase 3) | Ongoing serving cost ~$0.40M/year |

**Go/No-Go criteria summary:**

- Gate 1 (annotation quality): Cohen's kappa >= 0.82 on 500-example spot check at 10K labelled examples. If breached: halt and revise annotation guidelines.
- Gate 2 (model accuracy): >93% accuracy AND <4% false-positive rate on held-out Perplexity test set. If breached: run additional training epochs or expand training data; do not promote to production.
- Gate 3 (serving cost): <$0.12/1M claims at P50 on production infra. If breached: optimize serving (quantization, batch size tuning) before evaluating whether the economics still justify deployment.

**Alternative considered and rejected:** License the same model fine-tuning task to a third party (e.g., Cohere or a model API provider) and use their fine-tuned endpoint. Rejected because: (a) the Perplexity query-claim corpus is proprietary and sending it to a third party creates IP risk; (b) the serving cost structure for a third-party endpoint does not improve over `electra-base`; (c) the competitive moat argument requires Perplexity to own the model weights.

---

## 18) Phased rollout plan (v3 addition)

### Phase 0 - Baseline measurement (Weeks 1-4)

**Goal:** Establish a statistically reliable baseline of current citation accuracy before any engineering investment. No user-visible changes.

**Scope:**
- Sample 10,000 answers per day across all Focus modes.
- For each sampled answer, run a post-hoc semantic matching job: for each cited claim, does the cited source document contain text semantically supporting the claim?
- Segment by Focus mode (Academic, Web, Social), query category (medical, financial, current events, technical, general), and synthesis model (`synthesis_model` field).
- Define and instrument the `citation_accuracy_score` metric in the analytics pipeline before Phase 1 launch.

**Launch gate:**
- Statistically reliable baseline `citation_accuracy_score` per Focus mode established within 28 days (confidence interval width <3 pp at 95% level).
- Worst-performing segment (likely: Web + current-events + thin-content sources) and best-performing segment (likely: Academic + scholarly sources) identified.
- Privacy review for `CitationAuditRecord` retention policy completed and signed off.

**Kill switch:** Not applicable - this phase makes no production changes.

**Owner:** ML Platform (sampling job) + Analytics (pipeline) + Privacy/Legal (retention review).

---

### Phase 1 - Academic Focus verification, signals hidden (Weeks 5-16)

**Goal:** Ship claim-level verification as a guardrail layer on Academic Focus mode. Run A/B test: verification on but confidence signals visible to 50% of treatment group, hidden from 50% (control arm). Signals-visible treatment is the full UX; control arm confirms the model's accuracy improvement independently of the UI signals.

**Scope:**
- Post-generation verification pass live on 100% of Academic Focus answers.
- `CitationAuditRecord` written to database for all Academic Focus answers (all plans).
- Confidence signals (`verified`, `low_confidence`) rendered to the 50% treatment group.
- A/B test instrumentation: `citation_clicked_with_signal`, `upgrade_initiated`, D7 retention, Pro churn guardrail - all measured as described in section 9.
- Source panel passage highlight (Req 2.3) shipped for the treatment group.
- Audit panel and API not yet live (Phase 2 deliverable).

**Launch gate (all criteria must pass before Phase 1 goes live):**

| Gate criterion | Threshold | Evidence source |
|---|---|---|
| Verifier model false-positive rate | <8% on 500-example labelled test set | ML release evaluation |
| Verification pass latency P95 (Academic Focus) | <400ms on synthetic load test at 200 concurrent QPS | Load test result |
| Degraded-mode chaos test passed | Answers stream without error when verifier service is at 100% failure rate | Staging chaos test result |
| `CitationAuditRecord` write path load tested | Can handle 30M writes/day without write latency regression | Storage load test |
| Privacy review complete | `CitationAuditRecord` retention policy approved by Privacy/Legal | Signed review document |
| Exact-match override unit tests passing | Zero false positives on 100-example date/proper-noun test set | Automated test suite |

**Kill-switch trigger conditions:**

| Condition | Action |
|---|---|
| Academic Focus answer latency P95 >3.5s for any 7-day rolling window | Pause rollout; optimize verifier batch parallelism before resuming |
| Pro churn rate for treatment cohort exceeds control by >0.5 pp at day 14 | Pause signals-visible arm; investigate whether `low_confidence` labels are causing trust damage; do not pause signals-hidden arm |
| Verifier service error rate >1% of queries for any 1-hour window | Activate degraded mode (no signals shown); page on-call ML Platform engineer |
| `citation_accuracy_score` improvement vs. Phase 0 baseline is <3 pp at day 42 | Evaluate verifier model retraining before expanding to Web Focus; do not proceed to Phase 2 until improvement is >5 pp |

**Owner:** ML Platform (model serving), Core Experience (UI badges + passage highlight), Backend Platform (audit record storage), Analytics (A/B instrumentation).

---

### Phase 2 - Web Focus Pro rollout + enterprise audit trail (Weeks 17-28)

**Goal:** Extend verification and confidence signals to Web Focus Pro users. Ship the enterprise audit panel in Spaces and the audit API. Migrate the verifier model from `MiniLM` to `electra-base` for Web Focus (higher recall on thin content). Begin the labelling project for `perplexity-claim-verifier-v1`.

**Scope:**
- Verification live on Web Focus answers for Pro users only (50% A/B split; control = Pro users with verification running but signals hidden; treatment = Pro users with full confidence signals).
- `cross-encoder/ms-marco-electra-base` deployed for Web Focus; `MiniLM` remains for Academic Focus (faster, sufficient accuracy).
- Platt scaling recalibrated on a Web Focus-specific calibration set of 2,000 labelled (claim, web-source) pairs.
- `citation_audit_panel_opened` and audit export (JSON + PDF) live for Teams and Enterprise accounts in Spaces.
- Audit API (`GET /v1/audit/threads/{thread_id}`) live for Teams and Enterprise API keys.
- Labelling project kickoff: annotation tooling live; first batch of 10,000 (claim, source, label) examples from Perplexity's live query corpus.

**Launch gate (all criteria must pass):**

| Gate criterion | Threshold | Evidence source |
|---|---|---|
| Phase 1 A/B test result: Academic Focus citation click rate | >+3 pp vs. control at 95% significance | Analytics A/B report |
| Phase 1 A/B test result: Pro churn guardrail | Treatment churn not >0.5 pp above control | Analytics cohort report |
| `electra-base` false-positive rate on Web Focus test set | <6% on 200-example Web Focus labelled test set | ML release evaluation |
| Web Focus verification latency P95 | <600ms on synthetic load test at 400 concurrent QPS | Load test result |
| Audit panel legal review | `methodology_disclaimer` copy approved by Legal | Signed legal review |
| Audit export PDF generation latency | <60s for sessions up to 100 answers in staging | Staging performance test |
| Inter-annotator agreement gate (annotation tooling) | Cohen's kappa >= 0.82 on 500-example spot check | Annotation quality report |

**Kill-switch trigger conditions:**

| Condition | Action |
|---|---|
| Web Focus answer latency P95 >4s for any 7-day window | Pause Web Focus signals; investigate electra-base serving overhead; MiniLM fallback available |
| Pro churn rate in treatment cohort exceeds control by >0.5 pp at day 14 (Web Focus) | Pause Web Focus signals-visible arm; full root cause analysis before resuming |
| Audit export triggers any privacy incident (unauthorized data access, export scope exceeding authorized threads) | Disable audit export endpoint; incident response runbook activation |
| Web Focus `citation_accuracy_score` improvement <4 pp at day 42 | Evaluate electra-base model recalibration; do not proceed to Phase 3 without >4 pp improvement |

**Owner:** ML Platform (electra-base deployment + Platt recalibration + labelling infrastructure), Core Experience (Web Focus UI), Backend Platform (audit API + Spaces panel), Spaces team (audit panel UI), Legal (disclaimer review), Analytics (Web Focus A/B).

---

### Phase 3 - Full rollout + proprietary model + enterprise GA (Weeks 29-42)

**Goal:** Roll verification and confidence signals to all Web Focus users (free tier included). Replace `electra-base` with `perplexity-claim-verifier-v1` if model quality and serving cost gates are passed. Enterprise audit trail goes to GA (out of beta). Publish the competitive accuracy benchmark as a public research post.

**Scope:**
- Web Focus verification and confidence signals rolled to 100% of free-tier users.
- `perplexity-claim-verifier-v1` promoted to Web Focus production if all three model gates (accuracy, false-positive, cost) are passed. `electra-base` remains available as a hot fallback.
- Enterprise audit panel and API promoted from beta to GA; enterprise sales enablement materials updated with verified citation accuracy metrics.
- Competitive benchmark published as a Perplexity research blog post (500-query methodology; human-labelled results vs. Google AI Overviews and ChatGPT Search).
- `Pro churn` and `D30 retention` analysis published internally as a 90-day rollout postmortem; findings fed into Q1 roadmap prioritisation.

**Launch gate (all criteria must pass):**

| Gate criterion | Threshold | Evidence source |
|---|---|---|
| `perplexity-claim-verifier-v1` model quality gate | >93% accuracy AND <4% false-positive rate on held-out Perplexity test set | ML release evaluation |
| `perplexity-claim-verifier-v1` serving cost gate | <$0.12/1M claims at P50 on production-equivalent infra | Infrastructure cost measurement |
| Phase 2 Web Focus Pro A/B result: citation click rate | >+3 pp vs. control at 95% significance | Analytics A/B report |
| Free-tier false-positive rate projection | Projected `low_confidence` rate for free-tier query mix <18% per answer | ML Platform projection using Phase 2 data |
| Enterprise GA readiness: audit panel QA | Zero critical bugs in audit panel and export across test accounts in 5 enterprise customer profiles | Enterprise QA sign-off |
| Competitive benchmark publication | Methodology cleared by Legal and Trust/Safety for external publication | Legal sign-off document |

**Kill-switch trigger conditions:**

| Condition | Action |
|---|---|
| Free-tier `low_confidence` signal rate >25% per answer in first 7 days post-rollout | Raise `low_confidence` threshold from 0.65 to 0.70 for free tier (reduces signal rate without removing the feature); monitor for 7 days |
| `perplexity-claim-verifier-v1` accuracy degrades >2 pp vs. Phase 3 launch baseline at any monthly evaluation | Revert to `electra-base`; open incident for model drift investigation |
| Competitive benchmark generates a negative PR event (e.g., a query in the benchmark produces a citation laundering example in Perplexity's own output) | Pause benchmark publication; run full audit of the 500-query set before re-publishing |

**Owner:** ML Platform (proprietary model promotion + free-tier rollout), Core Experience (free-tier UI), Enterprise team (GA launch), Marketing/Research (benchmark publication), Analytics (rollout postmortem).

---

## 19) Experiment backlog (v3: expanded with rollout owners and end-state decisions)

| Experiment | Hypothesis | Primary metric | Acceptance criteria | Guardrail | Kill-switch | Rollout phase | Owner | End-state decision |
|---|---|---|---|---|---|---|---|---|
| Verified badge on Academic Focus citation click rate | Green `verified` badge increases citation click rate by making verification feel rewarding rather than anxiety-driven | `citation_clicked_with_signal` rate for `signal_type: verified` | >+4 pp at 95% significance; n=1,800 sessions per variant | Source panel click rate for `single_source` must not drop >1 pp | If `single_source` click rate drops >2 pp: hide badge for `single_source`, keep for `verified` only | Phase 1 | Core Experience + Analytics | If criteria met: ship to 100% Academic Focus; if not met: A/B copy variants ("Supported" vs. "Verified") |
| Pro conversion lift from verified badge (Academic Focus) | Free-tier Academic Focus users exposed to `verified` signals upgrade to Pro at higher rate than control | `upgrade_initiated` within 30 days | >+1.5 pp at 90% significance (looser threshold; longer conversion window) | D7 retention must not decline in treatment group | If D7 retention declines >1 pp: investigate whether badge creates over-reliance; remove badge from free tier if retention harm confirmed | Phase 1 (measure for 30 days post-exposure) | Growth + Analytics | If criteria met: include `verified` badge as a visible upgrade driver in Pro upgrade copy |
| `low_confidence` label copy (icon-only vs. "Limited sources") | "Limited sources" copy produces higher `citation_clicked_with_signal` rate than icon-only because it names the problem explicitly | `citation_clicked_with_signal` for `signal_type: low_confidence` | Copy variant achieves >+5 pp click rate vs. icon-only at 90% significance | Pro churn guardrail: churn must not rise by >0.5 pp in treatment group at day 14 | If Pro churn rises >0.5 pp in either variant: default to icon-only (less alarming) | Phase 1 (embedded within Test 1 allocation) | Core Experience + Copy | If criteria met: ship "Limited sources" copy globally; if not: ship icon-only |
| Passage highlight in source panel | Highlighting the specific `source_passage_used` increases citation click rate and reduces trust-anxiety (users know exactly what to look for when they open the source) | `citation_clicked_with_signal` rate; time-to-close-source-panel (proxy for user found what they were looking for) | Click rate +3 pp; time-to-close <5s for verified citations (vs. >10s current estimate) | Source panel load time must not increase >100ms | If source panel load time increases >200ms due to passage highlight fetch: lazy-load the highlight after the panel opens | Phase 1 (treatment arm only) | Core Experience + Retrieval | If criteria met: ship globally; integrate passage highlight into the `low_confidence` arm as well |
| Web Focus `electra-base` false-positive reduction (PR domain override) | Applying format-specific threshold adjustment for PR distribution domains reduces false-positive rate from ~7% to <5% on press release citations | False-positive rate on 200-example PR citation test set | False-positive rate <5% with override applied vs. 7.2% without | No regression on non-PR source false-positive rate | If non-PR false-positive rate increases by >1 pp: roll back domain-specific override | Phase 2 (model version gate) | ML Platform | If criteria met: include override in `electra-base` production deployment; extend domain list quarterly |
| Enterprise audit panel activation (Teams beta) | Teams trial accounts that see the citation audit panel during trial are more likely to convert to paid and invite IT stakeholders | `citation_audit_panel_opened` rate; `space_member_invited` with `it_reviewer` role | >60% of active Teams trial accounts open audit panel within 14 days; >30% of audit panel openers invite IT/legal stakeholder | Teams trial-to-paid conversion must not decline during beta (audit panel must not create "too complicated" friction for non-research teams) | If Teams trial-to-paid conversion declines >2 pp: simplify the audit panel entry point in the Spaces UI | Phase 2 (Teams beta) | Enterprise team + Spaces team | If criteria met: ship to Teams GA; include audit panel demo in enterprise sales kit |
| Competitive benchmark onboarding modal | Showing new users "Perplexity cites [X]% more accurately than Google AI Overviews" in post-first-query onboarding increases Academic Focus adoption and D14 retention | Academic Focus adoption rate in first 7 sessions; D14 session return rate | Academic Focus adoption +3 pp; D14 return rate not declining vs. control | D7 return rate must not decline in treatment group | If D7 return rate declines >1 pp: pause the modal; investigate whether competitive framing feels aggressive or off-brand | Phase 3 (after competitive benchmark published) | Growth + Marketing | If criteria met: integrate into standard new-user onboarding flow with updated accuracy stat each quarter |
| `perplexity-claim-verifier-v1` vs. `electra-base` head-to-head (shadow mode) | Proprietary model achieves >93% accuracy and <4% false-positive rate on Perplexity-specific query distribution, confirming Phase 3 model promotion criteria | Accuracy on held-out Perplexity test set; false-positive rate; serving cost per 1M claims | >93% accuracy; <4% false-positive; <$0.12/1M claims | `electra-base` must remain hot as fallback throughout shadow evaluation | If proprietary model shows accuracy regression vs. `electra-base` on any Focus mode subcategory: do not promote; retrain with additional data from that subcategory | Phase 3 (model evaluation gate) | ML Platform | If criteria met: promote to production; if not: continue `electra-base` and extend Phase 3 timeline by 8 weeks |

---

## 20) Launch readiness checklist (v3 addition)

### Phase 1 - Academic Focus launch

**Engineering:**
- [ ] Verifier service deployed to production with batch inference support (up to 8 pairs per call)
- [ ] Platt scaling calibration set (2,000 examples) validated; ECE <0.05 confirmed
- [ ] Degraded-mode failover tested in staging (100% service failure -> answers stream without error)
- [ ] `CitationAuditRecord` write path load tested at 30M writes/day
- [ ] Exact-match override unit tests passing (100 date/proper-noun examples, zero false positives)
- [ ] `citation_verification_completed`, `citation_signal_rendered`, `citation_clicked_with_signal` events instrumented and validated in staging

**Product:**
- [ ] `verified` and `low_confidence` badges WCAG 2.1 AA accessibility audit passed
- [ ] Tooltip copy reviewed and approved by Trust and Safety
- [ ] Source panel passage highlight tested across 20 representative source types
- [ ] A/B split instrumentation (treatment/control assignment by user hash) confirmed correct
- [ ] False-positive rate <8% confirmed on 500-example labelled test set

**Legal/Privacy:**
- [ ] `CitationAuditRecord` retention policy reviewed and approved by Privacy/Legal
- [ ] Free-tier `claim_text` hashing behaviour confirmed in audit records
- [ ] GDPR/CCPA compliance for `claim_text` 90-day retention confirmed

**Monitoring:**
- [ ] P95 latency alert configured: page if Academic Focus verification P95 >3.5s for any 1-hour window
- [ ] Verifier service error rate alert configured: page if error rate >1% for any 1-hour window
- [ ] `CitationAuditRecord` reconciliation job scheduled (daily; alerts on missing records)
- [ ] Pro churn weekly monitoring dashboard live for Phase 1 cohort

---

### Phase 2 - Web Focus Pro + enterprise beta launch

**Engineering:**
- [ ] `electra-base` deployed; Platt scaling recalibrated on Web Focus-specific 2,000-example calibration set
- [ ] Web Focus false-positive rate <6% on 200-example Web Focus labelled test set
- [ ] Audit API (`GET /v1/audit/threads/{thread_id}`) load tested; rate limiting (100 req/min/account) enforced
- [ ] Audit export JSON schema validation passing; PDF export generation <60s for 100-answer sessions
- [ ] Audit panel UI in Spaces load tested across Teams and Enterprise test accounts
- [ ] Annotation tooling live; 10,000-example labelling batch initiated

**Product:**
- [ ] Web Focus verification and confidence signals QA across all supported browsers and mobile apps
- [ ] Audit panel onboarding copy ("run your first search to generate an audit record") live for new Teams/Enterprise Spaces
- [ ] PDF export `methodology_disclaimer` copy approved by Legal
- [ ] Competitive benchmark 500-query set prepared; dual annotation initiated

**Legal/Privacy:**
- [ ] `methodology_disclaimer` copy signed off by Legal
- [ ] Audit export data scope confirmed (only threads belonging to the requesting account/Space)
- [ ] Enterprise beta DPA template reviewed and available for pilot customers

**Monitoring:**
- [ ] Web Focus P95 latency alert configured: page if >4s for any 1-hour window
- [ ] Audit export error rate alert: page if PDF generation failures >5% for any 1-hour window
- [ ] Labelling batch inter-annotator agreement checkpointed at 1,000, 5,000, and 10,000 examples

---

### Phase 3 - Full rollout + proprietary model GA

**Engineering:**
- [ ] `perplexity-claim-verifier-v1` model quality gates passed (>93% accuracy, <4% false-positive on Perplexity test set)
- [ ] Serving cost gate passed (<$0.12/1M claims at P50 on production infra)
- [ ] `electra-base` hot fallback confirmed deployable within 5 minutes of a rollback decision
- [ ] Free-tier rollout: `low_confidence` rate <18% per answer for representative free-tier query mix (validated in canary deployment)
- [ ] Enterprise GA: audit panel QA across 5 enterprise customer profiles; zero critical bugs

**Product:**
- [ ] Competitive benchmark publication cleared by Legal and Trust/Safety
- [ ] Pro upgrade copy updated to include "citation accuracy verified by AI" as a feature callout
- [ ] Enterprise sales enablement materials updated: "Perplexity Verified Answers achieves >90% citation accuracy on Academic Focus"
- [ ] 90-day rollout postmortem completed and findings distributed to product and ML teams

**Monitoring:**
- [ ] Free-tier `low_confidence` rate spike alert: page if >25% per answer in any 1-hour window within first 14 days
- [ ] `perplexity-claim-verifier-v1` monthly accuracy evaluation job scheduled
- [ ] Competitive benchmark accuracy stats refreshed and published quarterly

---

## 21) Key trade-offs

| Trade-off | Current bias (v3) | Cost | Alternative considered |
|---|---|---|---|
| Verification latency vs. answer quality | Accept up to 600ms added latency for Web Focus | Some users may perceive a slower answer; competitive disadvantage if Google closes the speed gap | Run verification asynchronously after streaming starts; show confidence signals as they complete rather than gating the stream. Risk: answer arrives before signals; late signal appearance feels like a glitch |
| Confidence signal granularity vs. user trust in the signal | Three-level signal (`verified`, `single_source`, `low_confidence`) | More levels = more cognitive load; risk of "confidence signal fatigue" | Binary signal (verified / not verified). Rejected: too blunt; `single_source` citations should not be penalised with a "not verified" label |
| Verification on every answer vs. targeted verification | Academic Focus first; Web Focus in Phase 2 | Web Focus users in Phase 1 do not get confidence signals; inconsistent experience across modes | Verify all answers from day one. Rejected: false-positive rate on thin Web content too high for v1; risks undermining signal credibility before model is tuned |
| Audit record as Pro feature vs. all users | Teams/Enterprise only for audit record; confidence signals for free tier | Free users get better citation confidence UX but no auditability | Audit records for all users. Rejected: storage cost and privacy surface area; enterprise auditability is a specific value prop for Teams/Enterprise, not a mass-market feature |
| Automatic source replacement for `low_confidence` vs. user notification | Attempt replacement first; fall back to `low_confidence` signal | Replacement may substitute a marginally better source but miss the user's preferred original source | Only notify; never replace. Rejected: if a better source exists in the retrieved pool, replacing silently improves accuracy with no UX cost |
| Privacy: store `claim_text` for all users vs. only Teams/Enterprise | Hashed for free; plain-text for Pro+; full retention for Teams/Enterprise | Reduces forensic value of free-tier audit records; some asymmetry in data handling | Full retention for all plans. Rejected: GDPR/CCPA compliance complexity and user trust risk of storing all query text indefinitely for non-paying users |
| Build vs. buy the proprietary verifier model | Build `perplexity-claim-verifier-v1` (Phase 3) | $50-75K upfront investment; 6-month timeline; ML engineering resource commitment | License a fine-tuned model from a third-party provider. Rejected: IP risk of sharing proprietary query corpus; no serving cost improvement; no competitive moat |

---

*All metrics are directional estimates based on publicly observable product behaviour and industry benchmarks - not internal Perplexity data.*
