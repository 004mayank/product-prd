# PRD: Review Inbox + Load-Aware Routing in GitHub (Reduce PR Review Latency Without Increasing Reviewer Burnout)

<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/GitHub.png" 
    alt="GitHub" 
    width="200"
  />
</p>

**Product:** GitHub (Pull Requests / Code Review)
**Author:** Mayank Malviya
**Status:** v2 - scoped MVP
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/github-teardown.md

---

## Context
GitHub’s durable edge is the **PR as the decision surface**: diff + discussion + review + checks + merge policy in one place.

But across team sizes, the limiting factor in the shipping loop is often human attention:

**Branch → PR → Review → Checks → Merge**

When review routing fails, teams pay in:
- slower cycle time (PRs waiting idle),
- higher context switching (authors pinging),
- reviewer burnout (everything feels urgent),
- lower trust in the system (“merges are random”).

This PRD proposes an MVP that makes GitHub a better **attention router for code review** by combining:
1) a **Review Inbox** that behaves like a real queue,
2) **explainable routing** (“why you were requested”),
3) **load-aware suggestions** for authors,
4) an explicit, rate-limited **unblock** mechanism.

---

## 1) Problem statement + why now
### Problem
GitHub review breaks in predictable ways:

1) **Review requests are not a reliable queue**
- Requests arrive via mixed channels (GitHub/email/Slack) and are hard to prioritize.
- “Requested reviews” is a list, not a work system (no clear urgency / blocking state).

2) **Ownership routing ≠ responsiveness routing**
- CODEOWNERS answers “who *should* review,” not “who *will* review quickly.”
- Load concentrates on a few senior reviewers.

3) **Low trust in review requests (“why me?”)**
- Reviewers can’t quickly see whether they’re requested because:
  - they own the code,
  - they recently worked in the area,
  - they’re on-call / platform owner,
  - or the author just guessed.

4) **PRs stall without a safe escalation path**
- Authors resort to pings and side channels.
- There’s no first-class “this is blocking” signal with guardrails.

**User-observable failure mode:** “PRs sit until I chase people; reviewers are overwhelmed; nothing feels prioritized.”

### Why now
- Checks and automation have improved; review throughput is the bottleneck.
- Remote/hybrid teams rely more on async review.
- GitHub already owns the primitives (CODEOWNERS, checks, branch rules). The opportunity is to unify them into a trustworthy routing layer.

---

## 2) Goals / non-goals
### Goals
1. **Reduce time-to-first-human-review**
- Improve p50/p90 time from “review requested” → “first substantive review” (comment, approval, or changes requested).

2. **Improve flow without increasing burnout**
- Better load distribution across qualified reviewers.
- Fewer “interruptions” and less pinging outside GitHub.

3. **Make review work legible**
- Reviewers understand what’s blocking, what’s high risk, and why they’re involved.

4. **Preserve governance**
- No bypassing protected branches / required reviewers.

### Non-goals
- A full task manager in GitHub.
- Fixing all notification overload across issues/discussions.
- Perfect ML ranking in v2 (start with simple, explainable rules).

---

## 3) Target users / segments + JTBD
### Segments
1. **High-output teams** (many PRs/day)
2. **Platform/infra/security owners** (cross-repo review load)
3. **Leads** (frequent final-approver bottlenecks)

### JTBD
- Reviewer: “Show me what needs my review now, and why it’s important.”
- Author: “Help me get the right reviewers fast and unblock my PR.”
- Lead: “Keep review work balanced; avoid star reviewers becoming a chokepoint.”

---

## 4) Success metrics
### Primary
- **Time to first review** (p50/p90) for PRs with at least one requested reviewer
- **Idle time**: total time PR spends in “waiting for review” state

### Secondary
- **Acceptance rate**: % requests reviewed within 8h / 24h (team-configurable)
- **Load distribution**: share of reviews done by top 10% reviewers (should decrease)
- **Stuck PR rate**: % PRs with no review after 24h/48h

### Guardrails
- Reviewer fatigue proxies: increases in ignored requests, mass-unsubscribes, rapid approvals without engagement
- Quality proxies (directional): revert rate / hotfix tagging

---

## 5) Proposed solution (v2)

### 5.1 Review Inbox (personal) — a prioritized queue
#### Placement
- Web: top-level nav entry **Review Inbox** (or PRs → “Review Inbox” tab)
- Mobile: prominent “Reviews” tab entry

#### Item model (what shows in the queue)
Each PR row shows:
- repo + PR title
- request type: **Required** vs **Requested** vs **FYI**
- check status: failing/pending/passed
- size bucket: S/M/L (simple heuristic)
- age since request
- **why it’s here** (see 5.2)
- “blocking” badge if the PR is explicitly blocked on the reviewer (see 5.4)

#### Default sections (ranked)
1) **Blocking me**
- I’m required and not yet reviewed, OR
- author triggered “Unblock” targeting me/team

2) **High risk / high impact**
- failing required checks
- touches sensitive paths (CODEOWNERS critical paths)
- large PRs (L) waiting > threshold

3) **Follow-ups**
- I reviewed/commented and new commits landed

4) **FYI / low priority**
- optional requests, watch-based items

#### Ranking (MVP, explainable)
Rank score = weighted sum of:
- required-reviewer (highest weight)
- age since request
- check state (failing required checks boosts priority; “all green” can be lower)
- PR size (larger boosts priority over time to avoid starvation)
- repo priority (repo setting: normal/high)

**Explicitly not in v2:** complex ML, individual performance scoring.

#### Key interactions
- **Review now** → opens PR in review mode
- **Snooze** (1h / until tomorrow) with reason prompt (optional)
- **Not me** (routes back to author with suggested alternatives; see 5.3)
- **Mark reviewed** (auto when review submitted)


### 5.2 “Why you were requested” (explainability)
Show one primary reason + optional secondary reasons:
- “CODEOWNERS: you own `/payments/*`”
- “Required by branch rules (team `@platform-owners`)”
- “You changed nearby code recently”
- “Author requested you”

Principle: reduce the feeling of randomness and teach ownership.


### 5.3 Load-aware reviewer suggestions (author assist)
When the author clicks **Request reviewers**:

**Inputs**
- CODEOWNERS candidates (paths touched)
- past reviewers for this area (recent N PRs)
- recent activity (commits/reviews in last 30 days)

**Outputs**
- Suggested reviewers with:
  - “Match reason” (owner/recent)
  - **Load indicator**: Low/Med/High

**Load indicator (v2 heuristic)**
- open review requests assigned to the reviewer (rolling)
- median time-to-first-review for that reviewer (rolling)

**Important constraints**
- Load is bucketed (no exact numbers) to reduce social pressure.
- If branch rules require a specific team/owner, suggestions cannot violate required constraints.


### 5.4 “Unblock” (safe escalation)
Add a first-class escalation action for authors:

**Trigger**
- PR has waited > X hours since review request (default 4h; configurable by org/repo)

**Action**
- Author clicks **Unblock review**
- Select scope:
  - “Unblock this PR” (pings requested reviewers)
  - “Escalate to team lead” (if configured)

**Guardrails**
- rate-limited per author (e.g., 3/day)
- cooldown per PR (e.g., once per 6h)
- org policy can disable

**Effect**
- PR gets a “Blocking” badge in Review Inbox for targeted reviewers
- optional: one additional notification (digest-friendly)


### 5.5 Team-level view (optional, but high leverage)
If we ship one team surface in v2, ship a minimal **Team Review Queue**:
- shows PRs waiting on the team’s required reviewers
- highlights aging items and load hotspots

(If this is too much scope, keep as v2.1.)

---

## 6) Detailed user journeys
### Journey A — Reviewer: start day
1. Opens Review Inbox.
2. Sees “Blocking me” at top.
3. Clicks a PR and completes review.
4. PR disappears from Blocking section automatically.

### Journey B — Author: request review
1. Opens PR, clicks request reviewers.
2. Picks suggested reviewers with “owner” reasons + low/med load.
3. Submits; reviewers see “why requested.”

### Journey C — Author: stalled PR
1. PR waiting > threshold.
2. “Unblock review” appears.
3. Author triggers unblock; reviewers get a targeted escalation.

---

## 7) Instrumentation plan
Track event funnel:
- review_requested → inbox_impression → inbox_click → review_submitted

Key cuts:
- required vs optional
- repo size, org size
- CODEOWNERS-heavy vs not

Also log:
- snooze usage (too much snooze may indicate overload)
- “Not me” usage (routing quality)
- unblock usage (rate limits, effectiveness)

---

## 8) Rollout plan
### Phase 0 — Internal dogfood
- 1–2 orgs, limited repos, feature flag

### Phase 1 — MVP
- Personal Review Inbox
- “Why requested”
- Reviewer suggestions with load buckets
- Unblock (rate-limited)

### Phase 2 — Expansion
- Team Review Queue (if not shipped)
- Repo/org tuning knobs
- Better sensitive-path classification

---

## 9) Risks / trade-offs / open questions
### Risks
- Load indicators can create social dynamics (“don’t pick me”).
- Authors might overuse unblock.
- Ranking mistakes can hide important work.

### Mitigations
- Bucketed load only, and allow opt-out.
- Strong unblock rate limits + policy controls.
- Always show required-review items prominently; don’t over-optimize ranking.

### Open questions
- Should we distinguish “substantive review” vs “LGTM drive-by” in metrics?
- What’s the minimal viable “sensitive path” configuration for v2?
- How should this integrate with email/notifications digests to reduce spam?
