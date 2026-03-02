# PRD: LinkedIn New User Activation v1 (Reach “First Career Value” in 7 Days)

**Product:** LinkedIn (Onboarding, Identity/Profile, Network, Feed, Notifications)
**Author:** Mayank Malviya
**Status:** v1 — initial problem framing + MVP requirements

---

## Version history
- **v1 (current):** Define activation as “first career value” and ship an onboarding MVP that gets new users to it within 7 days.

---

## 0) Context snapshot
LinkedIn’s long-term moat is the **identity graph** (who you are + who you know + what you’ve done). But for many new users, the early experience is ambiguous:
- They don’t know what to do first (profile? connect? follow companies? apply jobs?).
- The feed can feel low-signal before the network is built.
- Users hesitate to post or connect because the product is “public + professional”.

This creates an activation gap: users sign up, browse a little, then churn before the graph compounding kicks in.

---

## 1) Problem statement
### Problem
New users fail to experience a clear “career value moment” quickly enough.

Symptoms (typical):
- Day-0 sessions are curiosity-driven (viewing), not intent-driven (building).
- Users skip profile completion because they don’t see immediate payoff.
- Connection requests feel awkward without guidance (“who do I add?”).
- Notifications are noisy and don’t teach the product.

### Why now
- AI content + low-signal posting makes the cold-start feed worse.
- LinkedIn is used by wider cohorts (students, early career, career switchers) who need more scaffolding.
- Growth in job-seeking is cyclical; activation improvements compound across cycles.

---

## 2) Goals / Non-goals (v1)
### Goals
1. **Increase 7-day activation rate** for new signups by **+10% relative** (pilot cohort).
2. Reduce **Day-1 drop-off** (D1 retention) by **+3–5% relative**.
3. Increase completion of “high-signal identity” setup (profile + intent) without increasing user friction.

### Non-goals
- Rebuild the entire onboarding system or replace all ranking.
- Force posting as the primary activation path for every user.
- Change pricing/monetization (Premium, Recruiter) in v1.

---

## 3) North Star definition: “First Career Value”
We define activation as a user achieving at least **one** of the following within **7 days**:

1. **Opportunity value:** receives a relevant inbound (recruiter message / connection request from target role / job alert click that leads to apply intent).
2. **Network value:** connects to ≥5 relevant people *and* has ≥1 meaningful interaction (DM reply, endorsement, comment, or accepted invite from a high-relevance node).
3. **Knowledge value:** follows a curated set and consumes ≥3 high-signal items (posts/newsletters) with positive feedback (“more like this”, saves, dwell).

This is intentionally outcome-based, not “completed onboarding steps.”

---

## 4) Target users & JTBD
### Primary segments (v1)
1. **Early-career / students**
   - JTBD: “Help me look credible and find internships/first roles.”
2. **Job switchers (2–10 years exp.)**
   - JTBD: “Help me explore roles and get discovered without feeling cringe.”
3. **Career returners / switchers**
   - JTBD: “Help me frame my story and connect to the right people.”

### Key pain points
- “I don’t know who to connect with.”
- “My feed is random.”
- “I’m not ready to post.”
- “I don’t know what a good profile looks like.”

---

## 5) Success metrics & guardrails
### Primary outcomes
- **7-day Activation Rate (A7):** % of new users reaching “First Career Value.”
- **D1 / D7 retention:** returning sessions.

### Input metrics
- **High-signal profile completion:** headline + role + location + skills + education/experience (weighted score).
- **Relevant network seeding:** # of accepted invites from high-relevance nodes.
- **Interest/intent capture rate:** % users selecting intent (job seeking / learning / hiring / building audience).

### Guardrails
- Invite spam reports must not increase.
- User-reported “too pushy”/negative onboarding feedback must stay flat.
- Time-to-first-feed must not regress (>10% slower).

---

## 6) Solution overview (MVP)
### Core idea
Replace generic onboarding with a **guided activation path** based on user intent, and make the first week feel like a *sequence* (setup → seed network → get signal → first value), not a set of disconnected screens.

### Pillar A — Intent-first onboarding (30 seconds)
Ask 2 questions upfront:
1. “What brings you to LinkedIn right now?” (choose 1)
   - Find a job
   - Learn from industry
   - Hire / build a team
   - Build my professional presence
2. “What roles/topics are you focused on?” (choose 3–5 tags)

Store as `member_intent` and `interest_tags` for downstream personalization.

### Pillar B — Profile MVP that feels worth it
Instead of “Complete your profile (80%)”, present a **value-backed checklist**:
- Add role + company OR “Open to work” (unlocks better recommendations)
- Add 3 skills (unlocks better job matches)
- Add 1 proof item (project link, portfolio, featured link) (unlocks credibility badge)

Add a “See examples” drawer with 2–3 good headlines per segment.

### Pillar C — Network seeding with social comfort
Create a guided “Start with people you already know” flow:
- Import from contacts (optional) + show *why* (“to find colleagues/classmates”) and allow granular permission.
- Recommend 15 people in 3 buckets:
  1) People you likely know
  2) Role-models in your target domain
  3) Communities (companies/schools)

For each recommended person, show a **1-line reason** (“Worked at X”, “Same college”, “PM at Y”).

### Pillar D — First-week feed + notifications tuned for learning
For the first 7 days, create a **New User Feed Mode**:
- Heavier bias toward high-signal explainers, career advice with proof, and role-relevant posts/newsletters.
- More aggressive downranking of engagement-bait.

Notifications become a **coach**, not a slot machine:
- Day 0: “3 people to connect with”
- Day 1: “1 community to follow”
- Day 2: “1 high-signal newsletter for your role”
- Day 3–7: “1 small action” (comment on a post, message a connection, save a job)

---

## 7) Detailed requirements (v1)
### 7.1 Onboarding entry + flow
- Trigger: new signup OR returning user with low-activation score.
- Flow length: ≤ 6 screens; target completion time < 2 minutes.
- Must be skippable, but skipping sets a reminder CTA.

### 7.2 Data + personalization
- Persist:
  - `member_intent` (enum)
  - `interest_tags` (list)
  - `activation_path_state` (step completion)
- Use these to drive:
  - People recommendations (graph + similarity)
  - Feed ranking (cold-start boosts)
  - Notifications (sequenced prompts)

### 7.3 UI components
- Activation Home (in-app card)
  - Shows next best action (NBA)
  - Shows “why it matters”
  - Shows progress toward first value (not % profile)

### 7.4 Anti-abuse
- Invite sending limits tighter for first 72 hours.
- Higher scrutiny on repeated invite actions.
- Easy “Don’t suggest this person” and “Not relevant” feedback.

### 7.5 Experimentation + rollout
- A/B test at signup level.
- Start with 5% of new signups in 1–2 markets.
- Rollout gated by:
  - A7 uplift
  - Invite abuse guardrails
  - Support tickets/negative feedback

---

## 8) User stories & acceptance criteria
1. **As a job switcher**, I can select “Find a job” and 3–5 role tags, and within my first session I see 10 relevant jobs and 10 relevant people to connect with.
   - Acceptance: ≥70% of users see at least 15 recommendations above a relevance threshold.

2. **As a student**, I can create a credible headline with examples, and add at least one proof item.
   - Acceptance: proof item attach rate +15% vs control.

3. **As a new user**, my feed in week 1 feels useful, and I can give feedback to improve it.
   - Acceptance: hide rate down, “more like this” up, dwell on high-signal posts up.

---

## 9) Risks & mitigations
| Risk | Impact | Mitigation |
| --- | --- | --- |
| More steps reduces signup completion | Growth hit | Keep flow short, skippable, progressive disclosure |
| Invite spam increases | Trust hit | Tight invite limits + abuse detection + friction on bursts |
| Bias toward “known networks” entrenches privilege | Equity risk | Include “role-models” + communities; ensure diverse recommendations |
| Cold-start personalization is wrong | Bad early experience | Easy feedback + rapid re-personalization within 1 session |

---

## 10) Open questions (v1 → v2)
1. Should activation be personalized by cohort (student vs experienced) with different “first value” targets?
2. Should we create a lightweight “starter content pack” (curated newsletters, creators) per role?
3. How do we integrate posting without making it feel performative (e.g., private drafts, templates, feedback)?
4. Can we unify this with trust/scoring work (e.g., “signal density” mode) without overcomplicating onboarding?
