# PRD: Notification Load Shaping in Slack (Reduce Overload, Improve Response Reliability)

**Product:** Slack (Messaging)
**Author:** Mayank Malviya
**Status:** v3 — final PRD (executable spec for “Needs your response” + explainability)
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/slack-messaging-teardown.md

---

## Version history
- **v1 (initial):** Problem framing, core solution (bundling + focus modes + lightweight “needs you” labeling), baseline metrics + basic experiment outline.
- **v2:** Adds **user stories & acceptance criteria**, **rigorous experiment design** (randomization, segmentation, power heuristics, ramp), and a detailed **rollout + kill-switch plan** to safely ship load-shaping.
- **v3 (this revision / final):** Turns **“Needs your response”** into an executable spec: explicit scope boundaries, V1 heuristic classifier + ML progression path, offline + online evaluation plans with targets, failure modes/mitigations, and an **explainability + user-control UX contract**.

## 0) One-liner + context
Build **Notification Load Shaping** to help Slack users stay responsive to what matters (mentions, threads they’re involved in, high-signal channels) while **reducing notification fatigue** in busy workspaces.

Slack’s value compounds when its attention-routing loop is reliable: a message is seen, the right person responds quickly, and work unblocks. Today, many users respond by muting, ignoring, or batch-checking Slack—hurting loop reliability.

---

## 1) Problem statement + why now
### Problem
In high-volume workspaces, Slack notifications become a firehose:
- Too many low-value notifications (ambient channel chatter, noisy threads) trains users to **ignore** notifications.
- Users compensate by muting channels broadly or disabling notifications, which increases missed high-value items.
- Missed or delayed responses reduce team trust in Slack as the “fast lane,” pushing work to DMs, meetings, or external tools.

**User-observable failure mode:** “I can’t tell what needs my attention right now, so I either check everything (time sink) or mute Slack (miss important stuff).”

### Why now
- Slack is increasingly the surface where integrations, incident coordination, and cross-functional updates land; volume is trending up.
- AI summarization and better ranking are becoming table-stakes expectations, but the most immediate win is **delivery control**: send fewer, better notifications.
- Slack’s teardown opportunity tree identifies **Notification overload** as a primary “where Slack can break” risk and **Notification load shaping** as a high-leverage Loop B bet.

---

## 2) Goals / non-goals
### Goals
1. **Reduce notification fatigue** without decreasing responsiveness to high-value asks.
2. Improve Loop B health:
   - Increase **mention response rate**
   - Decrease **median & p90 mention response time**
3. Improve attention efficiency:
   - Reduce “overload sessions” where users must scroll/triage excessively to catch up.
4. Increase team-level outcomes (directional): improve **WAT‑SC** (Weekly Active Teams with Successful Coordination) via more reliable attention routing.

### Non-goals
- Replacing Slack with an email-style inbox (labels/rules/filters as the primary mental model).
- Changing core notification primitives (mentions, per-channel settings) in a way that breaks established norms.
- Solving integration spam in this PRD (belongs to **Integration signal quality** initiative).
- Building a full task management system.

---

## 3) Target users / segments + JTBD
### Primary segments
1. **High-volume ICs** (engineering, support, ops): many channels, frequent thread involvement, time-sensitive mentions.
2. **Managers / leads**: more @mentions, more cross-channel context switching, higher risk of missing asks.
3. **Mobile-heavy users**: notification overload is especially punishing on mobile; triage is harder.

### Jobs-to-be-done (JTBD)
- “When I’m working, help me **stay focused** but still catch things that truly need me.”
- “When I return after being away, help me **catch up quickly** without scanning every channel.”
- “When someone needs me, ensure I **see it and respond** with minimal friction.”

---

## 4) Proposed solution (Notification Load Shaping)
We will shape notification delivery using three tightly-related capabilities:

1) **Smart Bundling for non-mention chatter** (default-on, safe)
2) **Focus Mode presets** (1-tap, reversible)
3) **Action-required classification** to power both bundling and catch-up (ranking)

### 4.1 UX: Smart Bundling (non-mention channel chatter)
**Principle:** Mentions remain immediate (with safeguards). Non-mention notifications are bundled into predictable digests so users get fewer interruptions and higher signal.

#### Flow A — On receiving non-mention messages
1. User is a member of channel C.
2. Messages arrive that would normally generate a notification for the user (based on their current settings) but are:
   - not direct @mentions,
   - not from a thread the user follows,
   - not from a channel flagged as “High priority.”
3. Instead of firing individual notifications, Slack groups them into a **bundle window** (e.g., 10–20 minutes, adaptive).
4. At the end of the window, user receives a single digest notification:
   - Title: “Updates in 4 channels”
   - Body: top 2–3 items (ranked), e.g. “#proj-alpha: 3 new messages • #support: 7 new • 2 threads active”
   - CTA buttons: **View digest**, **Snooze bundles for 1h**, **Tune**

#### Flow B — Opening the digest
1. Tapping the digest opens a “Catch up” view:
   - Ranked list of channels/threads with short previews.
   - Clear labels: **Needs you** (see classification), **Active**, **FYI**.
2. User can:
   - Jump into a thread/channel
   - Mark items as “Seen”
   - Promote a channel to **High priority** (future delivery becomes immediate)

#### Key settings/controls
- **Bundling toggle:** On/Off (per user)
- **Bundling window:** Auto (default) / 5m / 15m / 30m
- **Do not bundle:**
  - DMs
  - Direct @mentions
  - Followed threads
  - “High priority” channels

#### Edge cases
- **Incident / war room channels:** Users may want immediate non-mention chatter. Provide per-channel override: “Always notify immediately in this channel.”
- **Low connectivity / mobile battery:** Bundles reduce notification spam—should be net-positive.
- **Quiet hours:** Bundles should respect existing Do Not Disturb; deliver a single catch-up summary upon exit.

---

### 4.2 UX: Focus Mode presets (Deep work / On call / In meetings)
**Principle:** Users need a simple, reversible way to change notification posture as their day context changes.

#### Flow C — Enabling a focus mode
1. User opens profile/status menu → sees **Focus modes** row.
2. Picks a mode:
   - **Deep work:** only DMs + direct @mentions; bundles everything else.
   - **On call:** immediate for incident channels + mentions; bundle the rest.
   - **In meetings:** suppress notifications except VIP list + direct @mentions; deliver a single “meeting catch-up” after.
3. User chooses duration: **30m / 1h / 2h / Until I turn it off**.
4. Slack updates:
   - Presence/status indicator (clear to teammates)
   - Notification rules (client-side + server-side where possible)

#### Key settings/controls
- Focus mode definitions are user-editable (advanced):
  - which channels are immediate
  - whether thread activity is immediate
  - whether keyword alerts break through
- **Safety rails:** Always allow direct @mentions to break through (unless user explicitly disables with a warning).
- **Auto-revert:** Always on by default; remind if mode remains on unusually long.

#### Edge cases
- **Users forget mode is on:** prominent banner + scheduled reminders; “Why am I not getting notifications?” diagnostic.
- **Time zones & quiet hours:** mode duration logic must be consistent across devices.

---

### 4.3 “Needs your response” (NYR) — executable spec (ranking + flagging, not new social norms)
This section defines what **qualifies as “Needs your response”**, how we compute it in V1, how we evaluate it, and the UX/explainability contract that keeps it trustworthy.

#### 4.3.1 Definition
A message or thread state is **Needs your response** if, based on **observable conversation structure and the user’s relationship to the thread**, it is **more likely than not** that *the next productive step requires action from the user* (reply, reaction-as-ack, or a short follow-up).

We treat NYR as a **ranking signal + label** used in:
- digest previews (top items)
- catch-up view ordering
- optional sidebar badge (“N need you”) and/or section

NYR is **not**:
- a sender-defined urgency marker
- a guarantee that something is urgent
- a commitment that Slack will never miss an ask (we still rely on direct mentions/DMs as the safest primitives)

#### 4.3.2 Scope boundaries (what qualifies / does not)
**Qualifies (high-confidence):**
1. **Direct @mention to the user** in a channel message.
2. **DMs** to the user (or small group DM where the user is addressed).
3. **Thread reply** that directly addresses the user when:
   - the user is explicitly mentioned in the reply, OR
   - the reply is in a thread the user follows / has participated in recently (see “thread state” features).
4. **Explicit question directed at the user** using simple, observable cues (heuristics): user name/handle + question mark, or short interrogatives (“can you…”, “could you…”, “do you know…”) in proximity to the user reference.
5. **Ownership/role cues** where the user is the obvious responder based on prior behavior in that channel/thread (e.g., the user has been the primary responder for similar asks in that channel recently).

**Does NOT qualify (by default):**
- **Broadcast updates** without a directed ask (release notes, FYIs, announcements).
- Messages that contain a question but are **not addressed** to the user and occur in a large channel.
- Threads where the user was mentioned historically but the conversation has moved on and another person is actively responding.
- Automated/integration messages that look like asks but are machine-generated (handled as a separate initiative; we may down-rank them heuristically).
- “Loud” channels with high activity where directed asks are ambiguous; these should fall into **Active** rather than NYR.

**Ambiguous cases (allowed, but must be explainable and correctable):**
- “Can someone…” questions in channels where the user typically responds.
- Incident/war-room chatter where response responsibility is shared (may be NYR only when the user is @mentioned / on-call mode engaged / channel set to immediate).

#### 4.3.3 Output labels and contract
We expose a three-bucket label on catch-up items:
- **Needs your response** (NYR)
- **Active** (high activity / context-building)
- **FYI** (low urgency, informational)

**Contract:**
- NYR labeling must be **high precision** (avoid crying wolf). If unsure, classify as **Active**.
- “Why” must be available for any NYR label shown to the user (see §4.3.7).
- Users can correct it (see §4.3.8); corrections feed future ranking.

#### 4.3.4 Feature list (grounded in observable behavior)
We intentionally restrict to features that can be derived from message metadata, thread graph, and user interaction history—no proprietary org signals.

**A) Message-level signals**
- mention type: direct @mention of user; @here/@channel; user name token (if visible)
- DM vs channel message
- punctuation/structure: presence of question mark; short interrogative patterns (regex-based)
- request verbs (lightweight pattern set): “please”, “can you”, “could you”, “need”, “review”, “approve”, “take a look”
- message length and link/file presence (often FYI vs ask; used cautiously)
- sender relationship: frequent dyad interactions with user (prior reply rate), recency of 1:1 interactions

**B) Channel metadata signals**
- channel size (member count bucket)
- channel priority setting (user-set “High priority”; “Always immediate”)
- channel recent volume (messages/hour) and burstiness
- channel type: public/private; DM vs group DM

**C) Thread state signals**
- is message a thread reply vs root
- whether user follows the thread
- whether user has posted in the thread (participation)
- time since user’s last contribution in thread
- whether someone else has responded after the ask (may de-escalate NYR)
- reply count growth after ask (if multiple responders, likely Active)

**D) User behavior signals (per-user personalization, bounded + interpretable)**
- user’s historical response likelihood to sender/channel (reply or reaction within X hours)
- user’s typical responsiveness by time-of-day/day-of-week (used for ranking, not for suppressing directed asks)
- explicit user feedback signals: “Not relevant”, “Snooze”, “Always immediate in this channel”

#### 4.3.5 Baseline V1 heuristic approach (ship-ready)
V1 uses a **rule-based scoring model** that produces:
- `nyr_score ∈ [0,1]`
- `nyr_label ∈ {NYR, Active, FYI}` with conservative thresholds

**High-level algorithm (illustrative):**
1. **Hard positives (NYR = true):**
   - direct @mention to user → NYR
   - DM to user → NYR
2. **Thread-directed asks:**
   - if thread reply AND (user mentioned OR thread followed/participated) THEN add score
3. **Question / request cues:**
   - if (user referenced) AND (question/request pattern) THEN add score
4. **De-escalation rules (reduce false positives):**
   - if channel is large + message is not a direct mention + ask is generic (“anyone”, “someone”) → cap score
   - if another user has already replied meaningfully after the ask (within short window) → reduce score
   - if sender is high-volume in that channel and user rarely responds → reduce score
5. **Bucketing:**
   - NYR if `score ≥ 0.75`
   - Active if `0.35 ≤ score < 0.75`
   - FYI if `< 0.35`

**Why heuristics first:**
- deterministic + debuggable
- easy to gate with safety rails (“if unsure → Active”)
- provides labeled data + user feedback to bootstrap ML later

#### 4.3.6 Progression path to ML (optional, after V1 proves value)
Once V1 ships and generates reliable training data (with user corrections), we can move to a supervised model.

**ML scope (incremental, not a rewrite):**
- Replace parts of the heuristic score with a model score while keeping:
  - hard-positive rules (direct @mentions, DMs)
  - conservative thresholds
  - explainability mapping (“top factors”)

**Candidate models:**
- V2: logistic regression / gradient-boosted trees using the feature set above (fast, interpretable)
- V3+: lightweight text embedding model for message intent (only if privacy + performance constraints allow)

**Training labels (observable + scalable):**
- positive label if user replies or reacts (ack) within a working-time window (e.g., 4 business hours) AND the message had directed cues
- negative label if user sees the message (open/click/thread view) but does not respond within window
- incorporate user feedback (“Not relevant”) as strong negatives

#### 4.3.7 Offline evaluation (NYR quality) + targets
We evaluate NYR as a **binary classification** at a chosen threshold and also as a **ranking problem**.

**Datasets:**
- sample of messages/threads from eligible high-volume users (stratified by channel size, platform, role proxy)
- exclude purely automated integration channels if feasible (or track as separate slice)

**Metrics:**
- **Precision@NYR** (primary): of items labeled NYR, % that receive a user response within window
- **Recall@NYR** (secondary): of items that truly required response, % labeled NYR
- **Calibration:** reliability curve for `nyr_score` (predicted vs observed response rate)
- **Ranking quality:** NDCG@K for catch-up lists (K=5,10)

**Target thresholds (V1 ship targets; tune in alpha):**
- Precision@NYR ≥ **0.70** overall; ≥ **0.60** in large channels
- Recall@NYR ≥ **0.50** (acceptable to miss ambiguous asks; they should land in Active)
- Calibration: top score bucket (≥0.8) observed response rate ≥ **0.75**

*(These are intentionally conservative; trust is the gating constraint.)*

#### 4.3.8 Online evaluation plan (product impact + trust)
NYR affects both **attention routing** and **user trust**. We evaluate it separately from bundling when possible.

**Online metrics (NYR-specific):**
- **Time-to-first-response** for NYR-labeled items (median, p90) vs control ordering
- **Catch-up action rate:** % of digest opens leading to action on a NYR item within 5 minutes
- **Trust/quality signals:**
  - rate of “Not relevant” on NYR items (lower is better)
  - rate of “Why am I seeing this?” opens followed by disabling/opt-out (watch for spikes)
  - net opt-out from NYR labels / sidebar badge

**Hypotheses:**
- NYR ranking reduces time-to-action on true asks without increasing total interruptions.

**Experiment approach:**
- within treated bundling cohorts, randomize **ordering + label visibility**:
  - A: NYR labels shown + NYR-based ranking
  - B: NYR-based ranking only (no label)
  - C: baseline ranking (control)

#### 4.3.9 Failure modes + mitigations
1. **False positives (“cry wolf”)**
   - Mitigation: high precision threshold; default uncertain items to Active; easy “Not relevant” feedback; show clear “why”.
2. **Bias toward loud senders / high-volume channels**
   - Mitigation: cap score in large channels unless directly addressed; sender-volume normalization; per-channel de-escalation.
3. **Incident channels / war-room dynamics**
   - Mitigation: if channel is set “Always immediate” or user is in On-call focus mode, treat channel chatter as Active (not NYR) unless directly addressed; avoid flooding NYR.
4. **Reinforcing responsiveness inequality** (model learns “user responds to X, so rank X higher”)
   - Mitigation: personalization is bounded; always keep structural cues (mentions/questions) dominant; monitor slices by sender type and channel size.
5. **Gaming** (senders add question marks or name tokens)
   - Mitigation: require multiple signals (address + question + relationship/thread state); rate-limit repeated low-quality asks from same sender/channel via de-escalation.
6. **Cold start** for new users/channels
   - Mitigation: fall back to global heuristics; require direct address for NYR; gradually incorporate behavior.

#### 4.3.10 Explainability / UX contract + user controls
**Explainability surfaces (minimum viable):**
- On each NYR-labeled item in Catch-up: a “Why” affordance that opens a short explanation.
- Explanations must be **specific, non-creepy, and actionable** (avoid opaque phrasing).

**Standard explanation templates (examples):**
- “You were @mentioned.”
- “You replied in this thread recently.”
- “This message asks you a question (mentions your name).”
- “You often respond in #support.” *(only if we can phrase it safely; otherwise omit behavioral phrasing and stick to thread/channel facts)*

**User controls to correct the system (must exist wherever NYR is shown):**
- **Not relevant** (demote this item; reduces similar future NYR candidates)
- **Snooze NYR** (mute NYR badge/digest prominence for 1h / today)
- **Always show NYR from this channel** / **Never show NYR from this channel** (bounded channel-level preference)
- **Undo** for any of the above (short window)

**Data/behavior contract:**
- User feedback must affect ranking within a short time window (target: within 24h; ideally immediate for the same channel).
- We do not expose private scores; we expose **reasons** and **controls**.

---

---

## 4.4) User stories & acceptance criteria
### User stories
1. **Deep work (reduce interruptions, don’t miss asks)**
   - As a high-volume IC, when I enable **Deep work** for 1 hour, I want Slack to stop interrupting me for ambient channel chatter, while still surfacing messages that clearly need me.
2. **Incident/war-room (keep channel hot)**
   - As an on-call engineer, when I am in an incident channel, I want **all activity in that channel** to alert immediately (even if not a mention) so I can track the situation in real time.
3. **Cross-device catch-up (single source of truth)**
   - As a mobile-heavy user, when I receive bundled digests on my phone and later open Slack on desktop, I want a consistent “catch up” state so I don’t re-triage the same items twice.
4. **Predictability + trust**
   - As any user, when Slack bundles or suppresses notifications, I want to understand **what changed and why**, and be able to override it quickly.

### Acceptance criteria (ship gates)
**A. Correctness / safety**
- **Mentions are never bundled by default.** Direct @mentions and DMs must remain immediate unless the user explicitly turns this off in advanced settings with a clear warning.
- **Per-channel “Always immediate” override works across devices** within ≤60 seconds (config propagation).
- **No silent loss:** every notification that would have been delivered in control is either delivered immediately or represented in a digest/catch-up surface (auditable in UI and logs).

**B. UX requirements (minimum viable)**
- Digest notification includes: (1) count of channels/threads, (2) top ranked preview(s), (3) CTA to **View digest**.
- Catch-up view clearly labels at least three buckets: **Needs you / Active / FYI**, and allows a 1-tap jump into the underlying thread/channel.
- Explainability: a “Why am I seeing this?” affordance exists for bundled items and for “Needs you” labeling.

**C. Cross-device consistency**
- A digest opened on device A is reflected as “seen/opened” on device B (eventual consistency acceptable, target p95 ≤5 minutes).
- Focus mode state and remaining duration are consistent across devices (p95 ≤60 seconds).

**D. Measurable outcomes (must hold in experiment before ramping beyond 25%)**
- **Mention response rate** non-inferior vs control (define non-inferiority margin, e.g., −1% absolute).
- **Mention response time p90** does not regress beyond pre-defined guardrail (e.g., +5% relative).
- Opt-out rate for bundling remains below threshold (e.g., <8% of treated eligible users in first 2 weeks).

---

## 5) Success metrics + instrumentation
Aligning to teardown V2’s Loop B + overload metrics.

### Primary success metrics
1. **Mention response rate (MRR):** % of @mention messages that receive a reply or reaction from the mentioned user within T hours (e.g., 4h working window; also measure 24h).
2. **Mention response time (MRT):** median and p90 time from @mention to first response by mentioned user.
3. **Notification mute/disable rate:**
   - % users who mute additional channels per week
   - % users who disable notifications / uninstall mobile push permissions

### Secondary metrics
- **Delivered-notification open rate:** opens/clicks on delivered notifications (overall and digest-specific).
- **Digest engagement:** % digests opened; time to first action from digest.
- **Overload proxy:** % sessions with scroll/triage behavior above threshold (e.g., “scroll depth” or “unread clearing time”).
- **WAT‑SC (directional):** team-level coordination outcomes (see teardown §5.2) should trend up in treated cohorts.

### Guardrails
- **Critical miss rate:** increase in “missed urgent” proxies, e.g. @mentions not seen within X minutes (read receipt equivalent where available) or significantly increased MRT p90.
- **DM deflection:** increase in DM share of messages (users fleeing channels).
- **Unsubscribe backlash:** opt-out rate for bundling/focus modes; negative feedback signals.

### Instrumentation (events to log)
New/updated events (illustrative names):
- `notification_delivered` (props: type=mention|dm|bundle_digest|thread|channel, client, mode)
- `notification_opened`
- `bundle_digest_created` (count of items, channels, ranking features)
- `bundle_digest_opened`
- `catchup_item_clicked` (entity=channel|thread|message)
- `nyr_labeled` (score_bucket, label, reasons=[...], entity=message|thread)
- `nyr_why_opened` (entity, label)
- `nyr_feedback` (action=not_relevant|snooze|always_show|never_show, scope=message|thread|channel, undo_used)
- `focus_mode_enabled/disabled` (mode, duration, auto_revert)
- `channel_priority_changed` (high_priority on/off)
- `mention_received` + `mention_responded` (response_type=reaction|reply)

Ensure consistent `workspace_id`, `user_id`, `surface`, `channel_id`, `thread_root_message_id`, and time buckets for cohorting.

---

## 6) Experiment design (rigorous)
### 6.1 Unit of randomization + eligibility
- **Primary unit:** **user-level** randomization **within workspace**, because the feature changes *delivery* not *sender behavior*.
- **Eligibility:** users above a notification-volume threshold (e.g., p50+ of notifications/day, or ≥K/day) AND with at least N @mentions/week so mention metrics are powered.
- **Stratification (recommended):** assign users within strata to reduce variance:
  - platform: mobile-heavy vs desktop-heavy
  - volume: medium vs very-high (e.g., p50–p90 vs p90+)
  - role proxy: manager/leads vs IC (if available) or “sends many mentions” vs “receives many mentions”

### 6.2 Treatment arms
- **Control:** current notification behavior.
- **T1 (Bundling):** Smart Bundling for non-mention chatter + catch-up view.
- **T2 (Bundling + Focus):** T1 + Focus Mode presets (Deep work / On call / In meetings).

*(Optional later)* **T3 (Bundling + Focus + stronger ranking):** gated behind separate classifier iteration; avoid mixing algorithm changes with delivery changes in the first powered test.

### 6.3 Ramp + duration
- **Phase 0 — Dogfood / internal alpha (1–2 weeks):**
  - Validate: no critical mention misses; digest UX; cross-device state consistency.
  - Confirm logging completeness for all primary metrics + guardrails.
- **Phase 1 — External beta (1–2 weeks):**
  - **5% ramp** of eligible users (with opt-out) to validate guardrails at small scale.
- **Phase 2 — Powered A/B/n (2–4+ weeks):**
  - Ramp to **25% / 25% / 50%** (T1/T2/Control) or **33/33/34** depending on desired compare.
  - Run long enough to cover weekday cycles and “return after absence” patterns (at least 2 full work weeks).

### 6.4 Power / sample size heuristics (practical)
Because the primary outcome is “response reliability,” we power on **Mention Response Rate (MRR)**.
- Let baseline MRR be *p0* (measure from pre-period; typical might be 0.55–0.75 depending on cohort).
- Choose a minimum detectable effect **MDE** that is *meaningful and safe*, e.g. **+1.0–2.0% absolute** improvement, and a **non-inferiority margin** for safety (e.g., not worse than −1% absolute).
- Approximate per-arm sample size using a two-proportion heuristic:
  - \(n \approx 16 \cdot p(1-p) / \text{MDE}^2\) for 80% power at ~5% alpha (order-of-magnitude planning).
- Prefer **sequential monitoring with conservative thresholds** (e.g., weekly looks) to catch harm early without over-optimizing p-values.

### 6.5 Analysis plan (what to look at)
- **Primary comparisons:**
  - T1 vs Control (does bundling help without harming mentions?)
  - T2 vs T1 (incremental value of focus presets)
- **Segmentation cuts (pre-registered):**
  - mobile-heavy vs desktop-heavy
  - very-high volume vs medium-high volume
  - time zones / working hours
  - “incident channel override” users vs non-users
- **Spillover checks:**
  - workspace-level aggregates (response times, DM share) to detect second-order effects.

### 6.6 Guardrails + stop rules (operational)
Stop / rollback or freeze ramp if any trigger hits for 24h+:
- **MRT p90** for direct @mentions worsens by **>5% relative** vs control (or crosses an absolute SLO).
- **Critical miss proxy** increases by **>X%** (define: @mentions not opened within Y minutes during working hours).
- **Notification permissions disablement** increases by **>Z%** vs control.
- **Opt-out / “Always immediate” override spikes** beyond threshold (trust regression signal).

---

## 7) Rollout & kill-switch plan
### Rollout principles
- Ship behind **server-controlled feature flags** with per-user assignment and per-workspace allowlisting.
- Separate flags so we can disable specific components without tearing down the whole system:
  - `bundling_enabled`
  - `catchup_view_enabled`
  - `focus_modes_enabled`
  - `incident_channel_immediate_override_enabled`
  - `ranking_labels_enabled`

### Ramp plan (post-experiment)
1. **0% (dark launch):** decisioning + logging on, user-visible changes off. Validate event quality + latency.
2. **5% → 10%:** eligible users only, default-on bundling with clear opt-out.
3. **25%:** only if acceptance criteria D (non-inferiority + guardrails) holds.
4. **50%:** expand eligibility threshold downward (include more “medium volume” users) only after stability.
5. **100% of eligible users:** then consider defaulting on for all users in high-volume workspaces.

### Kill-switch (what, who, how fast)
- **What we can shut off:**
  - Immediate: disable bundling decisioning (revert to baseline delivery) and/or hide catch-up UI.
  - Scoped: disable for a single workspace, platform, app version, or cohort.
- **Who can activate:** on-call product/eng via standard Slack incident process (document owner + backup).
- **Target time-to-mitigate:**
  - server-side flag flip reflected to clients **p95 ≤5 minutes**
  - if client caching exists, force refresh on next notification evaluation.

### Rollback verification checklist
- Mention notifications confirmed immediate in treated cohort (spot-check + logs).
- Bundles cease generating (`bundle_digest_created` drops to ~0).
- Any queued digests are either delivered as immediate notifications or dropped with user-visible explanation (choose consistent policy; do not silently lose).

---

## 8) Risks, mitigations, open questions
### Risks
1. **Bundling delays time-sensitive non-mentions** (e.g., incident chatter).
2. **Users lose trust** if they feel Slack is “hiding messages.”
3. **Cross-device inconsistency** (mobile vs desktop) creates confusion.
4. **Over-personalization** makes behavior hard to predict (“Why did I get this notification now?”).

### Mitigations
- Clear per-channel override: “Always immediate” for chosen channels.
- Transparent UI: digest shows what was bundled and why; easy jump to raw stream.
- Consistent rules engine shared across clients; server-side decisioning where possible.
- Explainability hooks: “Delivered as digest because: not a mention + not high priority + deep work mode on.”

### Open questions
1. What is the safest default bundling window (fixed vs adaptive) across roles/time zones?
2. What is the best “incident channel” detection method (manual vs heuristic vs integration tag) without scope creep into integration initiative?
3. What is the most reliable overload proxy we can instrument publicly/consistently across clients?
4. Should “High priority channels” be suggested automatically based on user behavior, or only user-set?
