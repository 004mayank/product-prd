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
**Status:** v1 - draft
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/github-teardown.md

---

## Context
GitHub’s center of gravity is the **Pull Request as the decision surface**. For teams, velocity and safety depend on one critical loop:

**Branch → PR → Review → Checks → Merge**

When this loop is healthy, teams trust that changes get reviewed quickly, high-quality feedback happens, automation is reliable, and `main` stays safe.

Today, a common failure mode across team sizes is **review latency**: the right people don’t look at the right changes at the right time. The result is slower shipping, more context switching, and reviewer burnout.

This PRD proposes a v1 set of improvements focused on **attention routing for code review**:
- a clearer **Review Inbox** that prioritizes what matters,
- **load-aware reviewer routing** (suggestions + clarity on “why you were requested”),
- lightweight **unblock actions** to keep PRs moving.

---

## 1) Problem statement + why now
### Problem
GitHub’s current review experience often breaks in predictable ways:

1) **Review requests are not a reliable queue**
- A reviewer’s attention is fragmented across notifications, email, Slack, and GitHub.
- Review requests aren’t clearly ranked by urgency/impact.

2) **Routing misses the “right reviewer at the right time”**
- CODEOWNERS ensures ownership but not necessarily responsiveness.
- Teams have uneven load; some reviewers are overloaded while others are underutilized.

3) **Reviewers don’t know “why me?”**
- Requests can feel random or spammy.
- Without context (ownership, recent changes, expertise), reviewers are less likely to engage quickly.

4) **PRs get stuck without an explicit “unblock” mechanism**
- Authors don’t know who is blocking them.
- Reviewers don’t know which PRs are waiting on them vs informational.

**User-observable failure mode:** “My PR sits for hours/days because nobody sees it (or everybody assumes somebody else will review it).”

### Why now
- Automation and checks have become stronger; human review is increasingly the bottleneck.
- Remote/hybrid work increases reliance on async review throughput.
- GitHub already has the primitives (CODEOWNERS, review requests, checks) — the opportunity is to make them work *together* as a trustworthy attention router.

---

## 2) Goals / non-goals
### Goals
1. **Reduce review latency**
- Decrease median and p90 **time-to-first-review** for PRs that request review.

2. **Improve review throughput without increasing burnout**
- Increase reviews completed per active reviewer, while avoiding concentration on the same small set of people.

3. **Make review work more intentional**
- Reviewers can quickly see what’s urgent, what’s high-risk, and what’s blocked.

4. **Preserve governance and quality**
- No weakening of branch protections or required reviews.

### Non-goals
- Replacing CODEOWNERS or protected branches.
- Building a full task management suite inside GitHub.
- Solving notification overload across all GitHub surfaces (this PRD focuses on PR review).

---

## 3) Target users / segments + JTBD
### Primary segments
1. **High-output engineering teams** (frequent PRs, multiple repos).
2. **Platform / infra / security owners** with many cross-team review requests.
3. **Managers / tech leads** who act as review bottlenecks.

### Jobs-to-be-done
- Reviewer: “Show me the PRs that need me *now*, and why.”
- Author: “Help me get the right reviewers quickly, and unblock my PR.”
- Team lead: “Keep review load balanced and the team shipping.”

---

## 4) Success metrics
### Primary
- **Time to first review** (p50 / p90) for PRs with requested reviewers
- **PR open → merge time** (p50 / p90), segmented by repo size and team size

### Secondary
- **Review request acceptance rate** (requested → reviewed within T hours)
- **Load distribution**: Gini coefficient (or similar) of reviews completed per reviewer per week
- **Stuck PR rate**: % PRs with no review after 24h / 48h (configurable)

### Guardrails
- Reviewer overload signals: increases in ignored/dismissed requests, unassigned reviews, or “review fatigue” proxies (rapid LGTM without engagement)
- Negative impact on quality proxies (reverts, hotfixes) — directional only

---

## 5) Proposed solution (v1)
### 5.1 Review Inbox (personal) — a prioritized queue
Create a dedicated **Review Inbox** experience (web + mobile), surfaced as a primary nav item (or as a first-class tab under Pull Requests).

**Default sections (ranked):**
1. **Action required (blocking)**
   - PRs where you are a required reviewer (CODEOWNERS / required reviews)
   - PRs where the author explicitly flagged “unblock me”
2. **High-risk / high-impact**
   - large diffs, sensitive files, security-related labels, failing/flaky checks
3. **Follow-ups**
   - PRs you commented on where new commits were pushed
4. **FYI**
   - optional, lower-priority requests

**Ranking signals (simple v1):**
- required reviewer vs optional
- PR age (time since request)
- check state (failed vs pending vs passed)
- diff size buckets (S/M/L)
- repo/team priority (repo-level setting)

**Key UX detail:** Every item shows **“why it’s here”** (see 5.2).

### 5.2 “Why you were requested” (reduce randomness)
For each review request, display a short explanation banner:
- “You own these paths via CODEOWNERS: `/payments/*`”
- “You recently changed nearby code (last 30 days)”
- “You’re in team `@core-platform` required for this repo”
- “Author requested you explicitly”

This improves trust and engagement, and helps new team members learn ownership.

### 5.3 Load-aware reviewer suggestions (assist authors)
When authors request reviews (PR sidebar):
- Suggest reviewers using CODEOWNERS + recent activity + availability proxy.
- Show a lightweight **load indicator** per suggested reviewer:
  - “Low / Medium / High” based on open requested reviews + recent response times.

**Principles:**
- Suggestions are advisory; CODEOWNERS/required rules still apply.
- Make it easy to add a second reviewer to improve latency.

### 5.4 “Unblock me” (author-driven escalation)
Allow authors to flag a PR as **Needs review** (time-boxed):
- Appears in reviewers’ “Action required” section.
- Optionally pings a secondary channel (team lead) after N hours.

This creates an explicit, socially acceptable escalation path.

---

## 6) Rollout plan
### Phase 1 (v1)
- Ship Review Inbox for individual users (behind feature flag).
- Add “why requested” messaging for common cases (explicit request, CODEOWNERS).
- Add reviewer suggestions + basic load indicator.

### Phase 2 (v1.1)
- Team-level settings: repo priority, escalation rules.
- Better classification of high-risk (sensitive files, security labels).
- Mobile-first improvements.

---

## 7) Risks / trade-offs / open questions
### Risks
- Load indicators can cause social friction (“don’t pick me, I’m high load”).
- Ranking can be gamed (authors marking everything as urgent).
- Additional UI could overwhelm if not kept tight.

### Mitigations
- Keep “unblock me” rate-limited and time-boxed.
- Allow teams to tune what “urgent” means.
- Start with a simple inbox; iterate based on adoption and latency impact.

### Open questions
- Should “Review Inbox” be personal-only, or also provide a **Team Review Queue**?
- What’s the best default for CODEOWNERS-heavy repos (where everyone is technically required)?
- How do we respect privacy while still showing useful load signals?
