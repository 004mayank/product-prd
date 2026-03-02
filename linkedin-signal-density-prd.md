# PRD: LinkedIn Signal Density Feed Mode (Reduce Low-Signal Content, Protect Trust)

**Product:** LinkedIn (Core Feed + Notifications)
**Author:** Mayank Malviya
**Status:** v1 — initial problem framing + MVP requirements
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/linkedin-teardown.md

---

## Version history
- **v1 (current):** Define the trust problem, goals, metrics, and MVP scope for a "Signal Density" feed mode that prioritizes high-trust, high-signal updates over viral filler.

---

## 0) Context snapshot
LinkedIn’s teardown identified a recurring failure: the feed drifts toward low-signal, generic content, while invites, messaging, and jobs face spam pressure. Users hack around this by doom-scrolling, muting, or abandoning the feed altogether, which weakens every other loop (notifications, invites, monetization). We need a first-class mode that explicitly optimizes for **signal density**—professional relevance, proof, and trust—without tanking engagement or creator incentives.

---

## 1) Problem statement & why now
### Problem
- High-frequency professionals complain that the feed feels like "motivational LinkedIn" instead of credible industry signal.
- When users hide or skip posts, downstream loops (notifications, creator retention, ad quality) degrade.
- Trust pressure is accelerating: AI-generated posts, sales spam, and repost farms are harder to detect with engagement heuristics alone.

### Why now
- LinkedIn is doubling down on creators and newsletters; we need a counterbalance to keep consumption trustworthy.
- Regulatory and brand scrutiny around misinformation + professional scams is rising.
- Feed infrastructure already exposes ranking hooks we can leverage; we just haven’t packaged them into a user-facing control with measurable guardrails.

---

## 2) Goals / Non-goals (v1)
### Goals
1. **Increase feed signal density** for opted-in users: more high-quality professional updates per scroll.
2. **Reduce hides/report rate** for opted-in users by ≥15% vs control while holding session depth flat.
3. **Improve creator quality mix**: increase impressions share for verified, high-reputation authors and posts with proof (work artifacts, data, roles).
4. **Protect trust metrics** that power monetization (invite acceptance, InMail reply rate) by reducing exposure to spammy content upstream.

### Non-goals
- Replace the default feed ranking for all users. V1 is an opt-in (with targeted defaults for power users).
- Police job postings or messaging spam directly (covered by other squads); we only mitigate via content routing.
- Launch new paid tiers. The mode is free and improves baseline experience.

---

## 3) Target users & JTBD
| Segment | JTBD | Pain today |
| --- | --- | --- |
| **Leads & IC experts** (eng/PM/ops) | "Give me credible industry updates without slogging through fluff." | Hide a lot, rely on curated newsletters, spend less time in feed. |
| **Recruiters / sourcers** | "Stay on top of talent/company updates to inform outreach." | Miss meaningful signals; default feed clutters with generic posts. |
| **Creators with proof-based content** | "Reach the right audience that appreciates depth." | Compete with viral low-effort posts; quality doesn’t equal reach. |

---

## 4) Key insights driving the solution
1. **Identity graph quality is LinkedIn’s moat**; we can lean on it to boost credible voices (employment verification, tenure, endorsements, profile completeness).
2. **User intent varies by session**: sometimes users want serendipity; other times they want actionable signal. Giving them a named mode sets expectations.
3. **Notifications are the metronome**: a cleaner feed plus better notifications closes the loop (Graph → Feed → Notification → Return).
4. **Trust needs metrics**: without explicit guardrails (hides, reports, replies), “quality” drifts toward pure engagement again.

---

## 5) Success metrics & guardrails
### Primary outcome
- **Signal Density Score (SDS):** weighted index per session = (professional relevance score + proof signals + dwell on high-depth posts) ÷ total impressions. Target: +20% vs control for opted-in users.

### Supporting metrics
- **Hide/report rate** per 1k impressions: -15% vs control.
- **Revisit rate** (sessions per WAU among opted-in cohort): +5%.
- **Creator quality mix:** share of impressions for verified/credentialed authors + posts with attachments/data: +10pp.

### Guardrails
- Session depth (scrolls per session) must be within ±5% of control.
- Ads revenue impact ≤ -1% for the cohort (tracked via RPM).
- No negative lift on invite acceptance or InMail reply rate for the cohort (lagging guardrail).

---

## 6) Solution overview (MVP scope)
### Pillar A — **Signal Density Mode toggle**
- UI entry: toggle chip above feed (“Signal Density”) with explainer tooltip.
- When on, feed ranking applies additive boosts/penalties:
  - +Boost for posts with verified roles, recent promotions, company moves, data-backed artifacts.
  - +Boost for authors with high reputation (endorsements, low spam reports, high quality feedback).
  - -Penalty for posts with repetitive templates, engagement-bait phrases, low dwell/hide ratio.
- Persist selection per device; allow quick revert.

### Pillar B — **Author & content trust scoring**
- Build a composite **Trust Vector** per author/post from existing signals:
  1. **Identity strength** (verification signals, tenure, endorsements).
  2. **Behavior health** (report rate, spam/"don’t know" feedback, authenticity heuristics).
  3. **Content depth** (attachments, data tables, experience tags, job change proof).
- Expose the score internally to ranking; show lightweight badges (“Verified employment”, “Data-backed insight”) when confidence is high.

### Pillar C — **Feedback loops & explainability**
- Inline "Why am I seeing this?" tailored for Signal Density mode.
- One-tap feedback options: “More like this”, “Off-topic”, “Feels spammy”.
- Capture mode-specific hides to retrain scoring.

---

## 7) Detailed requirements
1. **Mode activation UX**
   - Placement: just below story bar / post composer.
   - State persistence: per user per client; default off except for pilot cohorts (see rollout).
   - Empty state: explain benefits, link to blog/FAQ.
2. **Ranking integration**
   - Introduce `signal_density_multiplier` (0.7–1.3) applied post-core rank.
   - Feature gates for each trust component to allow ablation.
3. **Trust Vector pipeline**
   - Daily batch job to compute long-term author reputation.
   - Near-real-time stream (Kafka) for per-post signals (dwell, hides, reports) with 5-minute SLA.
   - Storage: Pin in `quality_author_profile` table keyed by member ID.
4. **Instrumentation**
   - Log `mode_state`, `signal_density_score`, `exposure_reason`, `feedback_event`.
   - Build Looker dashboard with cohort filters, before/after comparisons, guardrail alerts.
5. **User education**
   - In-feed coach mark on first activation; email digest call-out for opted-in users.
6. **Safety**
   - Fail-safe: if Trust Vector unavailable, fall back to base feed with banner “Showing standard feed”.
   - Abuse monitoring: integrity team gets weekly report of false positives (good posts demoted) via reviewer queue.

---

## 8) User stories & acceptance criteria
1. **As a staff PM**, when I switch to Signal Density mode, my feed should highlight job changes, product launches, and authored insights from credible peers within 2 scrolls.
   - *Acceptance:* ≥2 of the first 5 posts must pass a reputation + proof threshold in experimentation buckets.
2. **As a recruiter**, I can tap “More like this” on a high-signal post and see similar posts within the next 10 impressions.
   - *Acceptance:* Similarity recall ≥0.6 for embedding-based retrieval; logged event for follow-up.
3. **As a quality creator**, I should see a badge that explains why my post is featured in Signal Density mode.
   - *Acceptance:* Badge appears when Trust Vector > threshold; tooltip states “Based on verified role + engagement quality.”
4. **As any user**, I can exit the mode instantly, and my choice persists until I change it.
   - *Acceptance:* Preference stored client-side + server-side; toggling off reverts with <200ms latency.

---

## 9) Launch plan
1. **Tech readiness (Week 0–3)**
   - Build Trust Vector pipeline (batch + stream), offline evaluation vs labeled datasets.
   - Add ranking flag + logging.
2. **Dogfood (Week 4–5)**
   - Employees + Creator Advisory Board (~5k users).
   - Qual study on perceived quality, UI clarity.
3. **Beta (Week 6–9)**
   - 5% of power users (≥5 sessions/week, >50 connections) auto-prompted to try mode.
   - Monitor guardrails daily; ability to remote-disable boosts.
4. **GA criteria**
   - SDS +20%, hides -15%, no negative impact on session depth or ads RPM.
   - Integrity sign-off on false positive rate (<5% high-quality posts wrongly degraded).
5. **Post-launch**
   - Evaluate making mode sticky/default for certain segments (e.g., recruiters) if metrics hold.
   - Feed AI team explores ML classifier to replace heuristic penalties (scope for v2).

---

## 10) Dependencies & owners
- **Feed ranking**: implement multiplier, logging, experiment buckets.
- **Integrity systems**: share reputation scores, provide abuse review queue.
- **Notifications**: optionally highlight Signal Density posts in digests (stretch goal).
- **Data infra**: ensure Kafka topics + Looker dashboards exist before beta.

---

## 11) Risks & mitigations
| Risk | Impact | Mitigation |
| --- | --- | --- |
| Creators perceive mode as "shadow banning" | Quality creators churn | Add creator-facing badge + analytics; provide appeals channel. |
| Bias toward well-established profiles | Diversity of voices drops | Include "emerging expert" boost (engagement growth + endorsements) so newcomers can win. |
| Ads relevance mismatch | Revenue dip | Coordinate with Ads to exclude boosted/demoted inventory until targeting verified. |
| Mode confusion (“Did LinkedIn change my feed?”) | Support load | Explicit toggle UI + banner; easy revert + explanation. |

---

## 12) Open questions for v1 → v2
1. Should Signal Density eventually replace default ranking during work hours for certain cohorts?
2. How do we treat long-form content (newsletters, articles) inside the mode—inline preview or link-out?
3. What are the right incentives for creators to tag posts with proof (e.g., template prompts, attachments)?
4. Can we tie invites/messages gating to the Trust Vector to create a shared reputation platform?

