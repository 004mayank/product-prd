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
**Status:** v3 - routing + capacity + team queue
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

This PRD proposes a v3 that makes GitHub a better **attention router for code review** by combining:
1) a **Review Inbox** that behaves like a real queue,
2) **explainable routing** (“why you were requested”),
3) **capacity-aware + load-aware routing** (suggestions + optional auto-routing),
4) an explicit, rate-limited **unblock** escalation ladder,
5) a minimal **Team Review Queue** for accountability and SLAs.

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

## 5) Proposed solution (v3)

v3 keeps the v2 MVP surfaces, but upgrades GitHub from “show my requests” to a real **routing + capacity system**:
- **Capacity-aware**: reviewers can signal availability and GitHub can respect it.
- **Auto-routing (with guardrails)**: GitHub can request the *right* people by default, not just suggest them.
- **Team-level accountability**: teams get a queue + SLAs, not just individuals.

### 5.0 Principles (what makes it trustworthy)
1) **Explainable by default**: every request has a reason.
2) **Never violate governance**: branch rules/CODEOWNERS are hard constraints.
3) **Respect capacity**: route work to available reviewers; reduce interrupts.
4) **Safe escalation**: unblock exists, but is rate-limited and policy-controlled.

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
- **capacity context** (e.g., “routed to you: low load”) where appropriate
- “blocking” badge if the PR is explicitly blocked on the reviewer (see 5.5)

#### Default sections (ranked)
1) **Blocking me**
- I’m required and not yet reviewed, OR
- author triggered “Unblock” targeting me/team

2) **Due soon (team SLA)**
- items approaching a configured SLA (e.g., 8h/24h) for teams/orgs that opt in

3) **High risk / high impact**
- failing required checks
- touches sensitive paths (CODEOWNERS critical paths)
- large PRs (L) waiting > threshold

4) **Follow-ups**
- I reviewed/commented and new commits landed

5) **FYI / low priority**
- optional requests, watch-based items

#### Ranking (v3: still explainable)
Rank score = weighted sum of:
- required-reviewer (highest weight)
- SLA urgency (if configured)
- age since request
- check state (failing required checks boosts priority; “all green” can be lower)
- PR size (larger boosts priority over time to avoid starvation)
- repo priority (repo setting: normal/high)

**Explicitly not in v3:** opaque black-box ranking. If ML is introduced later, it must ship with explanations + guardrails.

#### Key interactions
- **Review now** → opens PR in review mode
- **Snooze** (1h / until tomorrow) with optional reason
- **Not me** (routes back to author + offers reroute suggestions)
- **Mark reviewed** (auto when review submitted)


### 5.2 “Why you were requested” (explainability)
Show one primary reason + optional secondary reasons:
- “CODEOWNERS: you own `/payments/*`”
- “Required by branch rules (team `@platform-owners`)”
- “You changed nearby code recently”
- “You’re on-call for `Payments` (this week)”
- “Auto-routed: lowest load among qualified reviewers”
- “Author requested you”

Principle: reduce the feeling of randomness, and make routing auditable.


### 5.3 Load-aware reviewer suggestions (author assist) → plus auto-routing
When the author clicks **Request reviewers**:

**Inputs**
- CODEOWNERS candidates (paths touched)
- required teams/users from branch rules
- past reviewers for this area (recent N PRs)
- recent activity (commits/reviews in last 30 days)
- reviewer capacity signals (see 5.4)

**Outputs**
- Suggested reviewers with:
  - “Match reason” (owner/recent/on-call)
  - **Load indicator**: Low/Med/High (bucketed)
  - “Expected response” bucket (Fast/Normal/Slow) derived from rolling medians (also bucketed)

**Auto-routing (opt-in)**
- For repos/orgs that enable it, GitHub can **auto-request** 1–2 reviewers that satisfy constraints.
- Authors can override, but overrides are captured as a signal (“I chose X instead of suggestion Y”).

**Important constraints**
- Load is bucketed (no exact numbers) to reduce social pressure.
- Auto-routing cannot remove required reviewers; it only fills the “requested” slots.


### 5.4 Reviewer capacity signals (new in v3)
Give reviewers lightweight, explicit control over routing:
- **Availability**: Available / Focus mode / Away (OOO)
- **Max queue**: soft cap on open review requests (e.g., 5)
- **Working hours** (optional): local-time window to avoid night pings

**How GitHub uses this**
- Avoid routing new optional requests to “Away” reviewers.
- Prefer “Available” + low-load reviewers for auto-routing.
- If only required owners are available, keep governance and show “capacity constrained” to the author.


### 5.5 “Unblock” (safe escalation) → escalation ladder
Add a first-class escalation action for authors:

**Trigger**
- PR has waited > X hours since review request (default 4h; configurable)

**Action**
- Author clicks **Unblock review**
- Select scope:
  1) “Unblock this PR” (targets currently requested reviewers)
  2) “Reroute to next-best reviewers” (if enabled and constraints allow)
  3) “Escalate to team lead/on-call” (if configured)

**Guardrails**
- rate-limited per author (e.g., 3/day)
- cooldown per PR (e.g., once per 6h)
- org policy can disable or restrict to required-review PRs only

**Effect**
- PR gets a “Blocking” badge in Review Inbox for targeted reviewers
- one additional notification (digest-friendly)
- optional: auto-create a team-queue item if a team SLA is enabled


### 5.6 Team Review Queue (v3: ship, not optional)
A minimal **Team Review Queue** for teams configured in branch rules/CODEOWNERS:
- shows PRs waiting on the team’s required reviewers
- highlights aging items, SLA breaches, and capacity constraints
- supports assignment suggestions (not mandates) for team triage

This is the missing layer that turns review from a personal notification problem into a team flow system.

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
