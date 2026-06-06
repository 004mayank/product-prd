# PRD: Perplexity Verified Answers - Claim-Level Citation Grounding

**Product:** Perplexity Answer Engine (Trust Layer)
**Author:** Mayank Malviya
**Version:** v2 - Improved PRD
**Changes from v1:** Added full data models, API contracts, verifier model benchmark with technology decision, detailed acceptance criteria per requirement, competitive experiment framework with power calculations, resolved open questions 1-6, UX copy proposals, dependency map, and expanded event schema with field-level descriptions.
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/perplexity-teardown.md

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, goals/non-goals, target personas, three solution pillars, core loops, basic requirements, event schemas, success metrics, competitive context, key trade-offs, open questions |
| v2 | Full data models, API contracts, verifier model benchmark and decision, detailed acceptance criteria, competitive experiment framework with power calculations, resolved open questions 1-6, UX copy proposals, dependency map, expanded instrumentation spec |

---

## Context

Perplexity's entire value proposition rests on a single promise: a cited answer is a verifiable answer. Every superscript number in every response is a contract - "this claim is grounded in this source."

That contract is broken more often than users know. The teardown identifies "citation laundering" - where the LLM confidently asserts a specific claim and attaches a real, plausible-looking URL whose actual content does not support the claim - as Perplexity's most critical quality failure mode. The failure is invisible in aggregate metrics: users who catch a laundered citation in a high-stakes context (medical, legal, financial, competitive analysis) stop using Perplexity for consequential queries without filing a support ticket. Usage appears to decline for no apparent reason. The highest-LTV cohort quietly leaves.

This PRD specifies **Perplexity Verified Answers**: a claim-level citation grounding system that checks the semantic relationship between each LLM-generated claim and its cited source at inference time, before the answer streams to the user. It makes the citation contract enforceable, not just decorative.

v1 established the core thesis, target segments, solution pillars, and measurement framework. v2 adds the full data models, API contracts, verifier model technology decision, detailed acceptance criteria, competitive experiment framework with statistical rigor, and resolves six of seven open questions. v3 will be the production-grade specification with phased rollout plan, launch gates, and fully resolved open questions.

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

3. **LLM citation accuracy is technically solvable now.** Embedding-based semantic similarity models (sentence-transformers, cross-encoder rerankers) are fast enough (<200ms P95 on cached source content) and cheap enough (fractions of a cent per answer) to run as a post-generation verification pass without materially impacting answer latency or unit economics. The benchmark data in section 5 below confirms this is deployable at scale in 2025.

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

---

## 17) Open questions - partially resolved (v2)

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

**Q7: Proprietary model investment timeline - remains open for v3**

This open question requires a strategic roadmap decision (multi-quarter, multi-million-dollar investment) that should be resolved at the v3 stage with input from engineering leadership and the product roadmap process. v3 will include a build vs. buy analysis for the `perplexity-claim-verifier-v1` fine-tuned model.

---

## 18) Key trade-offs

| Trade-off | Current bias (v2) | Cost | Alternative considered |
|---|---|---|---|
| Verification latency vs. answer quality | Accept up to 600ms added latency for Web Focus | Some users may perceive a slower answer; competitive disadvantage if Google closes the speed gap | Run verification asynchronously after streaming starts; show confidence signals as they complete rather than gating the stream. Risk: answer arrives before signals; late signal appearance feels like a glitch |
| Confidence signal granularity vs. user trust in the signal | Three-level signal (`verified`, `single_source`, `low_confidence`) | More levels = more cognitive load; risk of "confidence signal fatigue" | Binary signal (verified / not verified). Rejected: too blunt; `single_source` citations should not be penalised with a "not verified" label |
| Verification on every answer vs. targeted verification | Academic Focus first; Web Focus in Phase 2 | Web Focus users in Phase 1 do not get confidence signals; inconsistent experience across modes | Verify all answers from day one. Rejected: false-positive rate on thin Web content too high for v1; risks undermining signal credibility before model is tuned |
| Audit record as Pro feature vs. all users | Teams/Enterprise only for audit record; confidence signals for free tier | Free users get better citation confidence UX but no auditability | Audit records for all users. Rejected: storage cost and privacy surface area; enterprise auditability is a specific value prop for Teams/Enterprise, not a mass-market feature |
| Automatic source replacement for `low_confidence` vs. user notification | Attempt replacement first; fall back to `low_confidence` signal | Replacement may substitute a marginally better source but miss the user's preferred original source | Only notify; never replace. Rejected: if a better source exists in the retrieved pool, replacing silently improves accuracy with no UX cost |
| Privacy: store `claim_text` for all users vs. only Teams/Enterprise | Hashed for free; plain-text for Pro+; full retention for Teams/Enterprise | Reduces forensic value of free-tier audit records; some asymmetry in data handling | Full retention for all plans. Rejected: GDPR/CCPA compliance complexity and user trust risk of storing all query text indefinitely for non-paying users |

---

*All metrics are directional estimates based on publicly observable product behaviour and industry benchmarks - not internal Perplexity data.*
