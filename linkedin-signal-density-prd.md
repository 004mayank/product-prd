<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/Linkedin.png" 
    alt="Linkedin" 
    width="200"
  />
</p>

# PRD: LinkedIn Signal Density Feed Mode (Reduce Low-Signal Content, Protect Trust)
---

**Product:** LinkedIn (Core Feed + Notifications)
**Author:** Mayank Malviya
**Status:** end-to-end spec (SDS instrumentation), mode UX variants, creator incentives, anti-gaming, and operational playbook
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/linkedin-teardown.md
---

## Executive summary
LinkedIn’s core feed is drifting toward **low-signal, templated, engagement-bait content** (worse with AI), which erodes trust and reduces the value of LinkedIn’s identity graph. Users compensate by muting, hiding, abandoning the feed, or moving their “signal consumption” to newsletters and external communities.

This PRD proposes a first-class **Signal Density Mode**: an explicitly labeled feed mode that prioritizes **credible professional updates** (job changes, product launches, researched insights, data-backed posts) using LinkedIn’s differentiated signals (identity strength, reputation, proof-of-work artifacts) while holding engagement and revenue within guardrails.

The v2 plan makes the initiative measurable (via a concrete SDS spec), shippable (gated components, fallbacks), and safe (bias checks, false-positive budgets, integrity review loops).

---

## 1) Problem statement & why now
### Problem
- The feed increasingly feels like “motivational LinkedIn” vs professional signal.
- Engagement-only ranking naturally optimizes for broad, repeatable templates, not credibility.
- Rising AI-generated content and repost farms make heuristics brittle; users lose confidence.

### Why now
- LinkedIn is investing heavily in creators/newsletters; without a trust counterbalance, consumption quality deteriorates.
- Brand/regulatory scrutiny around misinformation, scams, and professional impersonation is rising.
- LinkedIn already has strong trust primitives (identity verification, role history, network graph) that can be productized into a user-facing control.

---

## 2) Goals / Non-goals
### Goals
1. **Increase signal consumption efficiency** for opted-in users: more credible updates per scroll.
2. **Reduce negative feedback**: lower hide/report rate for opted-in users by **≥15%** vs control.
3. **Increase exposure share for high-quality content**: +10pp impression share for posts meeting “High Signal” criteria.
4. **Protect downstream trust loops**: no degradation in invite acceptance, InMail reply rate, and spam reports (lagging guardrails).

### Non-goals
- Replace the default feed for all users.
- Solve job postings spam or DMs spam directly (we only reduce upstream exposure / route reputation signals).
- Launch a paid tier.

---

## 3) Target users & JTBD
### Primary segments
1. **High-frequency professionals (IC/Leads: eng/PM/ops/design/data)**
   - JTBD: “Give me credible industry updates fast.”
   - Pain: too much fluff; they hide often; value shifts to external sources.

2. **Recruiters / sourcers / hiring managers**
   - JTBD: “Stay on top of meaningful role/company updates.”
   - Pain: miss useful signals; feed clutter reduces reliability.

3. **Creators with proof-based content**
   - JTBD: “Reach an audience that values depth.”
   - Pain: compete with viral low-effort templates; quality doesn’t translate to distribution.

### Secondary segments
- Students / career switchers (may benefit, but risk of over-bias toward established profiles).

---

## 4) Definitions (make it measurable)
### 4.1 What counts as “High Signal” (post-level)
A post is considered **High Signal** if it satisfies at least one **Signal Type** and passes minimum **Trust/Quality thresholds**.

**Signal Types (examples):**
- Job change / promotion / role update (with credible metadata)
- Product launch / feature announcement (with artifact/link/media)
- Case study / lessons learned (with specifics, not generic quotes)
- Data-backed insight (chart, numbers, methodology, or cited sources)
- Hiring / team updates with clarity (role, context, specifics)

**Minimum thresholds (must pass):**
- **Identity strength** above threshold OR **network trust** above threshold
- **Negative feedback risk** below threshold (historical hide/report ratio, spam flags)

### 4.2 Signal Density Score (SDS) - session-level
SDS is a weighted index per session for opted-in users.

**SDS = (Σ w_i * HighSignalImpression_i) / TotalImpressions**, where weights reflect depth.

Recommended weights (v2):
- High-signal impression: **1.0**
- High-signal + dwell > N seconds: **1.5**
- High-signal + meaningful action (save, share, comment with >X chars, click-through to artifact): **2.0**

We also compute:
- **Low-signal exposure rate** = LowSignalImpressions / TotalImpressions
- **Regret rate** = (hide + “not interested” + report) / TotalImpressions

---

## 5) Success metrics & guardrails
### Primary outcomes
- **SDS:** +20% vs control among opted-in cohort.
- **Regret rate:** -15% vs control.

### Secondary outcomes
- **Revisit rate:** +5% sessions/WAU among opted-in cohort.
- **High-signal impression share:** +10pp.
- **Quality creator distribution:** +10pp share of impressions to creators meeting Trust thresholds.

### Guardrails (must hold)
- Session depth (scrolls/session) within **±5%**.
- Ads revenue impact (RPM) for cohort: **≥ -1%**.
- No negative lift on: invite acceptance, InMail reply rate (lagging).
- **Fairness guardrails:** maintain representation of “emerging experts” (see §11).

---

## 6) Solution overview
### Pillar A - Signal Density Mode (user-facing)
- A distinct, named mode in feed UI: **“Signal Density”**.
- Explicitly communicates tradeoff: “More credible updates, less viral filler.”
- One-tap exit (revert to standard feed).

### Pillar B - Trust Vector + High-Signal classifier
- **Trust Vector**: author/post composite score used as input to ranking.
- **High-Signal classifier**: ML model (or staged heuristics → ML) to label content type and depth.

### Pillar C - Explainability + feedback loops
- Mode-specific “Why am I seeing this?”
- Mode-specific feedback actions to improve future ranking.

---

## 7) Detailed product requirements
### 7.1 Entry points & UX
**Entry points:**
- Above feed: chip/toggle next to existing controls.
- Settings: Feed preferences → “Signal Density”.

**First-run education:**
- Coach mark: “Try Signal Density when you want credible professional updates.”
- Brief explanation (2 lines) + “Learn more”.

**Persistence:**
- Persist server-side per user; client caches state.
- Default **OFF** in v2; can be prompted to try via experiment.

**Mode indicators:**
- Visible label at top while active.
- Optional subtle banner for first 3 sessions.

### 7.2 Ranking integration (technical behavior)
Introduce `signal_density_multiplier` applied post-core rank in the pipeline.

**Score = BaseRankScore * signal_density_multiplier**

`signal_density_multiplier` is computed from:
- Trust Vector (author reputation, identity strength)
- High-Signal probability
- Negative feedback risk
- Diversity constraints (avoid monotony/over-concentration)

**Target range:** 0.7–1.3 (configurable).

**Gating:**
- Each component feature-flagged and ablatable:
  - `sd_trust_vector_on`
  - `sd_high_signal_classifier_on`
  - `sd_negative_feedback_penalty_on`
  - `sd_emerging_expert_boost_on`

**Fallback:**
- If Trust Vector or classifier unavailable, revert to base feed and show non-alarming notice: “Showing standard feed right now.”

### 7.3 Trust Vector spec (inputs)
**Author-level signals (slow-moving):**
- Identity verification status (employment verification, phone/email)
- Tenure + consistency of work history
- Endorsements/recommendations quality signals
- Historical report rate, spam flags, blocks

**Post-level signals (fast-moving):**
- Early dwell, hides, “not interested”, reports
- Attachment/link presence + type
- Novelty / repetition score (template detection)
- Engagement quality (comments length distribution; not raw likes)

**Storage:**
- `quality_author_profile` keyed by member ID
- `quality_post_profile` keyed by post URN

**SLA:**
- Author profile: daily batch
- Post profile: near-real-time stream; 5-minute freshness target

### 7.4 High-Signal classifier
**Phase 1 (ship fast):** rules + weak model
- Identify obvious low-signal templates (generic motivational patterns).
- Detect proof artifacts (links, attachments, numbers, citations).

**Phase 2:** supervised ML
- Train on a labeled dataset: “High signal / medium / low” with rubric.
- Calibrate thresholds to minimize false positives against known quality creators.

### 7.5 Feedback & explainability
**Why this post (in mode):** show 1–2 top reasons when confidence is high:
- “Data-backed insight”
- “Verified role”
- “From your industry/network”

**Feedback actions:**
- “More like this”
- “Off-topic”
- “Feels spammy”

All feedback must be logged with `mode_state=signal_density`.

### 7.6 Mode variants (Lite vs Strict) + scheduling
To avoid a single “one size fits all” toggle, this PRD supports two internally-configurable variants under the same user-facing label.

- **Signal Density (Lite)** : modest suppression of low-signal templates + boosts for proof/credibility.
  - Intended for broad rollout.
  - Multiplier range: 0.85–1.2
- **Signal Density (Strict)** (pilot): aggressively prioritize proof-of-work + reduce reshared/templated content.
  - Intended for power users + recruiters.
  - Multiplier range: 0.7–1.3

**Optional scheduling (pilot):** “Enable during work hours” (Mon–Fri, 9–6 local). Persisted as a preference; ranking reads `mode_state` + `schedule_state`.

### 7.7 Proof-of-work incentives (make the good path easy)
We should not only demote low-signal content; we must increase the supply of high-signal content.

**Creation-time nudges (soft):**
- If the draft matches a low-signal template, show a non-blocking prompt: “Add proof to improve reach in Signal Density mode.”
- Suggested add-ons: attachment, link, numbers, “what changed + why” bullets.

**Badges (read-side):**
- “Proof of work” badge when artifact confidence is high (attachment/link + depth cues).
- “Verified employment” badge when identity strength is high.

**Creator analytics (later):** add a small insight: “X% of your impressions came from Signal Density mode.” (guarded; avoid shadow-ban narratives).

### 7.8 Anti-gaming protections (integrity by design)
Signal Density will be a new target for spam. Add explicit anti-gaming constraints.

- **Template farms:** detect repeated n-gram/structure across accounts; apply a penalty even if they add superficial artifacts.
- **Artifact spoofing:** downweight low-quality or irrelevant attachments/links (mismatch between text topic and link domain; repeated link domains across farm).
- **Engagement pods:** incorporate network-collusion heuristics into Trust Vector (ablation-gated; integrity-owned).
- **Rate limits:** cap consecutive boosted impressions from a single author to avoid exploitation.

### 7.9 Notifications & digests (close the loop)
Notifications are the re-entry lever for this mode.

- Add an experiment-only digest module: “Today’s high-signal updates” with 3–5 items from the mode’s ranking.
- Push notification eligibility rules:
  - Only for opted-in users.
  - Only for items above a high-confidence threshold (Trust Vector + High-Signal).
  - Frequency capped (e.g., 1/day).

### 7.10 Operational requirements (kill switches + reviewer queues)
- **Kill switches:**
  - global: disable mode ranking multiplier
  - component-level: disable Trust Vector, classifier, negative-feedback penalty, emerging-expert boost
- **False-positive budget:** aim <5% demotion among a curated “known quality” creator list.
- **Reviewer queue:** weekly sampling of:
  - top demotions (what got suppressed)
  - top promotions (what got boosted)
  - appeal reports from creators

---

## 8) User stories & acceptance criteria (updated)
1. **As a staff PM**, when I enable Signal Density, I see credible job/company/product updates quickly.
   - Acceptance: In experimentation, **≥2 of first 5 posts** meet High Signal definition in mode bucket.

2. **As a recruiter**, I can improve the mode by giving “More like this” on a strong post.
   - Acceptance: Within next 10 impressions, at least 1 similar High Signal post appears (similarity recall target ≥0.6).

3. **As a quality creator**, I understand why my post is getting distribution.
   - Acceptance: If Trust Vector + High Signal are above threshold, show a badge/label; tooltip states top drivers.

4. **As any user**, I can exit mode instantly with no confusion.
   - Acceptance: Toggle off returns to standard feed in <200ms; state persists.

---

## 9) Experiment design
### 9.1 Cohorts
- Dogfood: employees + creator advisory board (~5k)
- Beta: power users (≥5 sessions/week) + recruiters segment

### 9.2 Experiment structure
- Randomized controlled experiment with:
  - Control: standard feed
  - Treatment A: mode ON with Trust Vector only
  - Treatment B: mode ON with Trust + classifier + feedback loop

### 9.3 Analysis plan
- Evaluate SDS, regret rate, and guardrails weekly.
- Slice by segment, geography, and “emerging expert” cohort.
- Monitor false positives: high-quality creators demoted.

---

## 10) Launch plan & milestones
1. **Week 0–2: Build/verify metrics + logging**
   - SDS computation pipeline
   - Dashboard and alerts

2. **Week 2–4: Integrate multiplier + feature gating**
   - Ablations enabled
   - Fallback behavior

3. **Week 4–5: Dogfood**
   - Qual study on perceived quality + confusion

4. **Week 6–9: Beta (5%)**
   - Prompt eligible users to try mode
   - Daily monitoring of guardrails

5. **GA criteria**
   - SDS +20%, regret -15%, guardrails hold
   - Integrity sign-off on false positives (<5% high-quality posts wrongly degraded)

---

## 11) Fairness, bias, and “Emerging Expert” support
**Risk:** mode over-boosts established profiles (rich-get-richer).

**Mitigation:** add an **Emerging Expert boost** feature gate:
- Criteria: rapidly improving engagement quality + endorsements growth + low negative feedback
- Provide modest boost capped by diversity constraints

**Monitoring:**
- Representation of creators by tenure bands, connection counts, and region
- Creator satisfaction survey for demotion disputes

---

## 12) Dependencies & owners
- **Feed ranking:** multiplier integration, gating, experiment buckets
- **Integrity systems:** reputation signals, abuse review queue, enforcement hooks
- **Notifications:** optional: promote high-signal posts in digests (stretch)
- **Data infra:** streams, storage tables, SDS pipelines, Looker dashboards
- **Comms/legal:** user education copy + policy alignment

---

## 13) Risks & mitigations (expanded)
| Risk | Impact | Mitigation |
| --- | --- | --- |
| “Shadow-banning” perception | Creator churn | Explainability, badge, appeals process, transparency in criteria |
| Bias toward established profiles | Diversity loss | Emerging expert boost + diversity constraints + monitoring |
| Revenue impact | RPM drops | Guardrail + incremental rollout + ads inventory coordination |
| Confusion about feed changes | Support load | Explicit mode labeling + easy exit + first-run education |
| Model errors / false positives | Trust harm | Human review queue, false-positive budget, gated rollout |

---

## 14) Appendix - SDS instrumentation & computation notes
### 14.1 Required events
Log a single canonical event for every feed impression in the mode pipeline.

**Event: `feed_impression`** (per impression)
- `member_id` (hashed)
- `session_id`
- `post_urn`
- `mode_state` (standard|signal_density)
- `mode_variant` (lite|strict)
- `rank_score_base`
- `signal_density_multiplier`
- `rank_score_final`
- `high_signal_label` (high|medium|low|unknown)
- `high_signal_confidence` (0–1)
- `trust_vector_score` (0–1)
- `neg_feedback_risk` (0–1)
- `reasons[]` (top N strings: verified_role, proof_artifact, industry_match, etc.)

**Event: `feed_action`** (per action)
- `session_id`, `post_urn`, `mode_state`, `action_type`
- `action_type` ∈ {dwell, click_link, save, share, comment, hide, not_interested, report, follow_author}
- `dwell_ms` (for dwell)

**Event: `mode_toggle`**
- `member_id`, `timestamp`, `from_state`, `to_state`, `entry_point` (chip|settings|prompt)
- `schedule_enabled` (bool)

### 14.2 SDS computation (reference)
Compute SDS per session and per member-day.

- `HighSignalImpression` = (`high_signal_label` == high) AND (`high_signal_confidence` ≥ threshold) AND (`trust_vector_score` ≥ threshold)
- Session SDS = Σ weights / total_impressions
  - weight = 1.0 for HighSignalImpression
  - +0.5 if dwell_ms ≥ N
  - +1.0 if meaningful action (save/share/comment>threshold/click_link)

Also compute (and dashboard):
- regret rate = (hide + not_interested + report) / impressions
- low-signal exposure rate (label==low)
- concentration = top-author impression share (anti-farm signal)

### 14.3 Dashboards + alerts (minimum)
- SDS trend + regret trend (daily)
- guardrails (depth, RPM) (daily)
- false-positive audit set performance (weekly)
- fairness slices (tenure, region, connection count) (weekly)

---

## 15) Open questions
1. Should Signal Density become a *default* for specific segments (recruiters, hiring managers) with an easy escape hatch?
2. Should we expose a user-facing slider (More signal ↔ More variety), or keep variants internal to avoid complexity?
3. How do we handle newsletters/long-form: boost only when “new information density” is detected, or treat as a separate lane?
4. Can Trust Vector be safely reused to gate invitations/DMs without creating opaque “social credit” concerns (policy + UX implications)?
