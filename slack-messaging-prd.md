# PRD: Notification Load Shaping in Slack (Reduce Overload, Improve Response Reliability)

**Product:** Slack (Messaging)
**Author:** Mayank Malviya
**Status:** v1 — execution-ready PRD
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/slack-messaging-teardown.md

---

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

### 4.3 “Needs your response” classification (ranking, not new social norms)
We need a lightweight classifier to label items in digests/catch-up:
- **Needs you:**
  - direct @mention
  - reply to the user in a followed thread
  - message that includes the user’s name + question mark (heuristic)
  - message in a channel where user is the only/one of few active responders (behavioral heuristic)
- **Active:** high activity thread/channel but not clearly addressed to user
- **FYI:** low urgency / broadcast-style updates

This classification powers:
- Digest ranking
- Catch-up view ordering
- Optional “nudge” badge in the sidebar (“2 need you”)

**Important:** We are not introducing a new sender-side “urgency” taxonomy in V1 (no intent picker here) to avoid friction and norm complexity.

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
- `focus_mode_enabled/disabled` (mode, duration, auto_revert)
- `channel_priority_changed` (high_priority on/off)
- `mention_received` + `mention_responded` (response_type=reaction|reply)

Ensure consistent `workspace_id`, `user_id`, `surface`, `channel_id`, `thread_root_message_id`, and time buckets for cohorting.

---

## 6) Experiment / rollout plan
### Unit of experimentation
- **Primary:** user-level randomization (minimizes admin dependency; fastest iteration)
- **Secondary:** workspace-level holdout for spillover checks (optional, later)

### Proposed design
**Phase 0 — Dogfood / internal alpha (1–2 weeks)**
- Enable for Slack-internal workspaces only
- Validate: no critical mention misses; digest UI usable; device consistency

**Phase 1 — A/B test (2–4 weeks)**
- 50/50 user split within eligible workspaces
- Eligibility: users with notification volume above threshold (e.g., ≥K notifications/day) to ensure power
- Treatments:
  - T1: Smart bundling only
  - T2: Smart bundling + focus mode presets
  - Control: current behavior

### Holdout & contamination
- Because message senders are not changed, user-level testing is acceptable.
- Monitor channel-level spillover (if fewer interrupts changes response patterns for teammates).

### Guardrails & stop conditions
- Stop/rollback if:
  - MRT p90 worsens beyond X% for @mentions
  - Critical miss proxies exceed threshold
  - Notification disablement increases significantly

### Rollout
- Gradual ramp post-success: 5% → 25% → 50% → 100% of eligible users
- Default-on for bundling in high-volume cohorts; offer opt-out

---

## 7) Risks, mitigations, open questions
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
