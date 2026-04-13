# PRD: ChatGPT - Outcome-First Guided Sessions

**Product:** ChatGPT (OpenAI)
**Author:** Mayank Malviya
**Status:** v2 - Improved PRD
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/chatgpt-teardown.md

**Version:** v2 - Improved PRD
**Changes from v1:** Added competitive analysis with specific win/loss reasons, detailed requirements with acceptance criteria and edge cases for all five features, full event schemas, instrumentation spec with baselines and directional targets, experimentation framework with three experiments, expanded risk register with likelihood/impact table, and funnel analysis with drop-off hypotheses.

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, goals/non-goals, personas, five-feature solution (outcome templates, iteration coaching, trust UX, workspace capture, handoff), north star metric, MVP scope, risks, open questions |
| v2 | Competitive analysis with specific win/loss reasons, detailed requirements and acceptance criteria, edge cases per feature, event schemas, metrics with baselines and targets, experimentation framework, expanded risk register |

---

## Context

ChatGPT is the highest-distribution AI product in the world with hundreds of millions of users, yet most sessions end without a shippable artifact. The activation funnel breaks at the second turn: users treat the product like search, get a single answer, and leave. Power users who discover iteration stay and pay - but the product does nothing to teach or scaffold this.

This PRD specifies **Outcome-First Guided Sessions**: a structural intervention that moves ChatGPT from "a clever chat box" to a guided work session that starts from outcomes, scaffolds iteration, calibrates trust, and produces handoff-ready artifacts. The feature targets the gap between first response and paid habit - where the largest retention opportunity lives.

v2 is the depth pass. Every requirement has acceptance criteria and at least one edge case. Competitive analysis is grounded in specific observable behaviors, not category labels.

---

## 1) Problem statement + why now

### The core loop failure

```
User types prompt -> gets response -> [copies text and leaves]
                                   OR
                                      [types "make it better" -> marginally better response -> leaves]
```

The loop fails because the product offers no **outcome frame**: no starting constraint, no definition of done, no next-step guidance. The user supplied none of these either - and the system never asked.

**Observable failure modes:**

- User asks "help me write an email to my manager about a promotion" - ChatGPT produces a generic email template with `[Manager Name]` placeholders. The user copies it, edits it manually, and never returns. The product added no scaffolding, never asked about tone, relationship, or stakes.
- User asks "explain how transformers work" - gets a 600-word explanation - reads it - closes the tab. The product never surfaced "want a practice problem?" or "want me to explain the attention mechanism specifically?".
- User is a PM trying to write a PRD - pastes a half-formed spec on turn 1 - gets a rewrite that's 70% right - edits manually for 40 minutes - no export, no structure. The product treated this as a one-shot rewrite, not a multi-step session.

### Funnel breakdown

| Stage | Drop-off hypothesis | Magnitude (est.) |
|---|---|---|
| Acquisition -> First prompt | Fear of blank prompt; lack of starting point | ~20% of new visitors leave without submitting |
| First response -> Second turn | Users treat as search; don't realize iteration is the product | ~50% of free sessions are single-turn |
| Second turn -> Habit | Inconsistent quality, hallucinations, no context reuse | ~60% of users do not return within 7 days |
| Habit -> Paid | Unclear value delta vs. free; tool switching cost seems low | ~5% free-to-paid conversion (estimated) |
| Paid -> Long-term | Trust regressions, context overhead, missing workflows | ~30% annual churn on Plus (estimated) |

### Why now

The product is at an inflection. GPT-4o makes the model strong enough that the bottleneck has shifted from "model quality" to "session quality". Competitors are converging on model performance (Gemini 1.5, Claude 3.5, Llama 3) - the next durable edge is product scaffolding that turns a capable model into a reliable work partner. The window to establish session quality as a moat is 6-12 months.

---

## 2) Goals / non-goals

### Goals

1. Increase **weekly successful outcomes per active user** (North Star) by **25% relative** within 90 days of launch.
2. Increase **activation rate** (new users reaching a successful outcome within first session) by **15pp** - from ~20% estimated baseline to ~35%.
3. Increase **second-turn rate** (% sessions with at least 2 user turns) by **20pp** - from ~50% to ~70%.
4. Increase **artifact export/copy rate** (proxy for outcome) by **30% relative** among users exposed to templates.
5. Reduce **downvote rate** on responses in template-guided sessions by **20% relative** vs. open-ended sessions.

### Non-goals

- Fully autonomous agents across third-party systems without checkpoints - out of scope, different product surface.
- Enterprise permissions, audit logs, shared workspaces - later, dependent on team workflows foundation.
- Perfect factual accuracy - instead ship calibrated confidence UX + verification pathways.
- Real-time collaborative session editing - single-user scope for v2.
- Mobile-first redesign - desktop web is the primary surface for knowledge worker outcomes.

---

## 3) Target users / personas + JTBD

| Persona | Observed behavior | JTBD | Pain today | Success signal |
|---|---|---|---|---|
| **Knowledge worker** (PM / analyst / marketer) | Pastes half-formed briefs; iterates 3-5 turns on good days; frequently abandons without exporting | "Draft a memo I can send to my VP in 20 minutes" | Blank prompt paralysis; output is generic; no export path | Completes template in <5 min; copies or exports artifact; returns within 7 days for next task |
| **Builder** (engineer / data / no-code) | Asks about code + architecture; follows up with "why"; high second-turn rate already | "Debug this error and explain what was wrong so I don't repeat it" | Long threads lose context; hard to export clean code blocks; no session summary | Code block export; explanation retained; returns for next debugging task |
| **Student / learner** | Single-question sessions; rarely iterates; high drop-off after first answer | "Help me understand X so I can do Y" | Doesn't know what to ask next; no practice scaffold | Engages with iteration suggestions; time-on-session increases; self-reports understanding |
| **Ops / support / sales** | Repetitive task pattern; same prompt structure week-over-week | "Draft 10 customer response emails given these 10 tickets" | No reusable template memory; re-enters constraints every session | Saves session as template; reuse rate within 30 days |
| **Curious consumer** | Short sessions; high volume; low iteration; mostly free-tier | "Quick answer to X" | No hook to go deeper; product doesn't distinguish between quick-answer and work-session intent | Distinguishable from knowledge worker sessions via intent classification; not over-scaffolded |

**Activation insight:** The aha moment is a session that produces an artifact the user ships without heavy manual editing. This has never happened for ~80% of users because the product never scaffolded it. The first template-guided session with a copy/export event is the acquisition -> habit conversion moment.

---

## 4) Competitive analysis

| Competitor | Core capability | Where they win | Where ChatGPT wins | Loss risk |
|---|---|---|---|---|
| **Claude.ai (Anthropic)** | Long context, strong reasoning, nuanced tone control | Wins on long-form editing tasks; better at following complex multi-part instructions; growing artifact/Projects feature | Broader tool ecosystem (browsing, image, code execution); stronger brand + distribution; memory | Claude's Projects feature is a direct response to context-friction problem; risk of losing knowledge workers doing deep research |
| **Gemini Advanced (Google)** | Google Workspace integration (Docs, Gmail, Sheets, Calendar); Deep Research | Wins for users already in Google ecosystem; research tasks with source citation; multimodal over Google products | Model quality on reasoning tasks; developer mindshare; custom GPTs ecosystem | Google's workspace integration solves the handoff problem natively; significant enterprise threat |
| **Perplexity** | Real-time web citations; transparent sourcing | Wins on factual queries where trust depends on sources; knowledge workers who need to verify claims | Broader conversational capability; iteration; artifact generation | Perplexity Pro is directly targeting the "research and summarize" use case; trust-through-sourcing is a growing differentiator |
| **GitHub Copilot Chat / Cursor** | IDE-embedded; codebase-aware; inline suggestions | Wins for builders in-editor; codebase context is its moat | General-purpose; non-code use cases; chat flexibility | For builders, in-editor context is decisive; ChatGPT loses code debugging sessions to Cursor in 2024 for users who try both |
| **Microsoft Copilot (M365)** | Deep Office integration (Word, Excel, Outlook, Teams) | Wins for enterprise users creating documents within M365 workflow | Consumer distribution; model quality; iteration flexibility | M365 Copilot solves the handoff-to-doc problem for enterprise users natively; hard to beat with an external product |
| **Notion AI / Coda AI** | Workspace-contextual AI; reads user's own documents | Wins for knowledge workers whose primary artifact surface is Notion/Coda; DB-aware generation | Breadth; not tied to a single tool; can synthesize across contexts | Workspace-native AI is the strongest moat for outcome quality; risk concentrates among PM and ops personas |

**Strategic conclusion:** ChatGPT's competitive moat is breadth + model quality + distribution. The gap is session scaffolding. Every competitor above has a stronger answer for "help me finish a specific task" within their context. Outcome-first guided sessions are the product's best response to this gap without requiring workspace integration.

---

## 5) Solution - five pillars

### Pillar 1: Outcome templates (persona-aware)

Starter kits that capture 3-6 minimal constraints and produce a structured artifact. Available at session start and via `/template` command mid-session.

**Initial template set (v1):**

| Template | Slots | Output artifact | Primary persona |
|---|---|---|---|
| Write a PRD / product brief | Goal, user problem, success metric, constraints, timeline | Structured PRD with problem/solution/metrics sections | Knowledge worker |
| Executive memo / update | Audience, key decision or ask, context, tone | Formatted memo, 200-400 words | Knowledge worker |
| Debug + explain plan | Error message or description, language/framework, what was tried | Root cause, fix, explanation of why it happened | Builder |
| Study plan + tutoring session | Topic, current understanding, learning goal, time available | Structured study plan + first lesson | Student |
| Project plan / checklist | Project goal, deadline, team size, constraints | Phased checklist with owners and dates | Ops / PM |

### Pillar 2: Iteration coaching

After the first artifact draft, surface 2-3 high-signal follow-up suggestions. Suggestions are dynamic - generated based on the artifact type and current draft content, not hardcoded.

**Example follow-ups for PRD template:**
- "Add alternatives considered and why you chose this approach"
- "Add a rollback or failure scenario"
- "Convert to a one-page exec summary"

Suggestions appear as clickable chips below the response, dismissible, and disappear after being acted on.

### Pillar 3: Trust and confidence UX

AI outputs for factual or advisory tasks include explicit **Assumptions** and **Needs verification** sections. Not applied to creative tasks (email rewrite, brainstorm) where confidence framing is not meaningful.

**Confidence taxonomy:**
- `Assumptions` - constraints the model inferred from incomplete input
- `Needs verification` - claims that are uncertain, time-sensitive, or should be checked against primary sources

Optional per-template: "Verification checklist" - a short list of things to check before shipping the artifact.

### Pillar 4: Lightweight session context

Within a session, store and surface:
- `goal` - user-stated or template-inferred objective
- `constraints` - key limitations (deadline, audience, tone, format)
- `sources` - files or links provided
- `decisions` - user-approved choices made during iteration
- `outputs` - artifacts generated in this session

Displayed as a collapsible sidebar or header bar. Allows the user to edit context mid-session without restarting. Persists for 30 days for logged-in users (opt-in for free users, default-on for Plus).

### Pillar 5: Shippable handoff

Export presets per artifact type:
- Markdown (all templates)
- Clean plain text (email, memo)
- Structured JSON (PRD, project plan) - for integration with external tools

"Copy" button always present. "Export as..." button appears after artifact is accepted (no heavy edit in 60 seconds after generation).

---

## 6) Detailed requirements + acceptance criteria

### Req 1 - Outcome templates

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 1.1 | Show template picker at session start for logged-in users | Template picker visible within 500ms of new session load; shown to 100% of logged-in users in rollout cohort | User pastes a long prompt immediately: auto-dismiss picker; log `template_picker_skipped` |
| 1.2 | Each template collects 3-6 slots before generating | Slot collection flow completes in median <3 user turns; all required slots filled before generation | User skips a required slot: infer from context if possible; if not, mark as `[not specified]` in generation prompt; do not block |
| 1.3 | Template selection available via `/template` mid-session | `/template` command recognized in any turn; shows picker; applies to next generation | User is mid-conversation: warn "this will start a new structured session"; offer to continue in current session instead |
| 1.4 | Template output renders as a structured artifact with distinct sections | Output uses H2 headers and section breaks; not a continuous paragraph | Long templates (>2000 tokens output): paginate with "Show full artifact" expand; do not truncate silently |
| 1.5 | Template picker shows 5 templates + "Start without template" option | All 5 templates listed with one-line description; "Start without template" always visible | New user (first session): surface 3 templates only + "Explore more"; reduces cognitive load |

### Req 2 - Iteration coaching

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 2.1 | Show 2-3 iteration suggestions after first artifact generation | Suggestions appear within 300ms of artifact completion; rendered as clickable chips | User immediately starts typing: hide suggestions without animation to avoid distraction |
| 2.2 | Suggestions are artifact-type specific, not hardcoded | Suggestions change across different template types and different draft content | Very short artifact (<100 tokens): show "expand with more detail" as default suggestion regardless of type |
| 2.3 | Clicking a suggestion triggers generation immediately without a new user prompt | Generation starts within 200ms of chip click; no additional input required | Model cannot complete the suggested action (e.g., "add alternatives" but user gave no context): show single clarifying question; do not silently ignore |
| 2.4 | Suggestions dismissed after being acted on | Acted-on chip is removed from suggestion bar; remaining chips stay | All 3 suggestions dismissed or acted on: suggestion bar disappears; user can trigger via `...` menu |
| 2.5 | Maximum 1 suggestion round per generation | Suggestion bar only appears once per artifact version; refreshes on user-initiated re-generation | User clicks "regenerate": new suggestions generated for new artifact version |

### Req 3 - Trust and confidence UX

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 3.1 | `Assumptions` section appended to factual or advisory outputs | Present in >90% of PRD, debug, project plan template outputs; absent from creative rewrites | Template output is purely creative (email tone rewrite): suppress assumptions section entirely |
| 3.2 | `Needs verification` section lists 1-3 specific claims | Items are specific (not generic "verify before using"); each item is a concrete checkable claim | No uncertain claims identified: omit section entirely; do not render empty section |
| 3.3 | Trust labels do not increase generation latency by more than 200ms P95 | Latency A/B test shows <200ms increase for sessions with trust labels vs. without | Latency spike >400ms: fall back to no trust labels; log `trust_label_fallback` |
| 3.4 | Verification checklist available as optional follow-up via "Add verification checklist" link | Checklist generated within 1 additional generation step; adds 3-7 specific verification actions | Template type is creative: hide "Add verification checklist" link entirely |

### Req 4 - Session context capture

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 4.1 | Session context sidebar populated after first template slot collection | `goal` and `constraints` visible in sidebar within 500ms of first artifact generation | User did not use a template: sidebar shows "Session context: none captured yet" with edit affordance |
| 4.2 | User can edit any context field mid-session | Edit triggers re-generation prompt: "Context updated. Regenerate last artifact with new context?" | User edits context after 10+ turns: warn "editing early context may affect accuracy of later outputs" |
| 4.3 | Session context persists for 30 days for Plus users | Context retrievable from session history; shown in "Continue session" entry point | User downgrades from Plus: context retained for 30 days from downgrade date; then cleared |
| 4.4 | Free users can opt-in to session context persistence | Opt-in prompt shown after first session with context captured; opt-out via Settings | GDPR regions: opt-in required by default; no pre-checked opt-in |

### Req 5 - Shippable handoff

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 5.1 | "Copy" button present on all artifact generations | Renders within 200ms of artifact completion; copies full formatted text including section headers | User copies mid-generation (streaming): copy captures partial artifact; show "copy complete artifact" prompt after generation finishes |
| 5.2 | "Export as..." button appears for structured artifacts (PRD, project plan, debug plan) | Button appears 60 seconds after artifact generated if no heavy editing (>30% character change) | User immediately edits heavily: suppress export button; show only "Copy" |
| 5.3 | Markdown export preserves section structure | H2 headers render as `##`; code blocks render as fenced ` ``` ` blocks; bold renders as `**` | Exported markdown opens in system default app; if no .md handler: copy to clipboard and show "Copied as Markdown" toast |
| 5.4 | Plain text export strips all markdown | No `#`, `**`, or backtick in plain text output | Tables in plain text: convert to tab-separated values; do not drop table data |

---

## 7) Data model

### `session`

```json
{
  "session_id": "uuid",
  "user_id": "uuid",
  "session_type": "template_guided | open_ended",
  "template_id": "prd | memo | debug | study | project_plan | null",
  "goal": "string | null",
  "constraints": {"key": "value"},
  "sources": [{"type": "file | link", "name": "string", "id": "uuid"}],
  "decisions": [{"turn": "int", "decision": "string"}],
  "artifact_ids": ["uuid"],
  "status": "active | completed | abandoned",
  "created_at": "ISO8601",
  "last_active_at": "ISO8601"
}
```

### `artifact`

```json
{
  "artifact_id": "uuid",
  "session_id": "uuid",
  "user_id": "uuid",
  "template_id": "string | null",
  "turn_number": "int",
  "content_tokens": "int",
  "has_assumptions_section": "bool",
  "has_verification_section": "bool",
  "iteration_suggestions": ["string"],
  "export_events": [
    {
      "format": "markdown | plain_text | json",
      "ts": "ISO8601"
    }
  ],
  "outcome": "exported | copied | edited | discarded | in_progress",
  "edit_delta_pct": "float | null",
  "created_at": "ISO8601"
}
```

### `template_slot_fill`

```json
{
  "event_id": "uuid",
  "session_id": "uuid",
  "user_id": "uuid",
  "template_id": "string",
  "slot_name": "string",
  "fill_method": "user_typed | inferred | skipped",
  "turn_number": "int",
  "ts": "ISO8601"
}
```

---

## 8) Event schemas

```json
// session_started
{
  "event": "session_started",
  "session_id": "uuid",
  "user_id": "uuid",
  "trigger": "template_picker | slash_command | direct_prompt",
  "template_id": "string | null",
  "ts": "ISO8601"
}

// template_selected
{
  "event": "template_selected",
  "session_id": "uuid",
  "user_id": "uuid",
  "template_id": "prd | memo | debug | study | project_plan",
  "entry_point": "session_start | slash_command | mid_session",
  "ts": "ISO8601"
}

// artifact_generated
{
  "event": "artifact_generated",
  "session_id": "uuid",
  "artifact_id": "uuid",
  "user_id": "uuid",
  "template_id": "string | null",
  "turn_number": "int",
  "has_trust_labels": "bool",
  "suggestion_count": "int",
  "generation_latency_ms": "int",
  "ts": "ISO8601"
}

// artifact_exported
{
  "event": "artifact_exported",
  "session_id": "uuid",
  "artifact_id": "uuid",
  "user_id": "uuid",
  "export_format": "markdown | plain_text | json | clipboard_copy",
  "time_to_export_ms": "int",
  "ts": "ISO8601"
}

// iteration_suggestion_acted
{
  "event": "iteration_suggestion_acted",
  "session_id": "uuid",
  "artifact_id": "uuid",
  "user_id": "uuid",
  "suggestion_index": "int",
  "suggestion_text": "string",
  "action": "clicked | dismissed",
  "ts": "ISO8601"
}

// trust_label_expanded
{
  "event": "trust_label_expanded",
  "session_id": "uuid",
  "artifact_id": "uuid",
  "user_id": "uuid",
  "section": "assumptions | needs_verification | verification_checklist",
  "ts": "ISO8601"
}

// template_picker_dismissed
{
  "event": "template_picker_dismissed",
  "session_id": "uuid",
  "user_id": "uuid",
  "dismiss_method": "x_button | start_without | direct_prompt_typed",
  "ts": "ISO8601"
}
```

---

## 9) Success metrics + instrumentation

### North Star
**Weekly successful outcomes per active user** - sessions ending with `artifact_exported` or `artifact.outcome = copied` within 7 days, per weekly active user.

### Input metrics

| Metric | Baseline (est.) | 90-day target | Instrumentation event |
|---|---|---|---|
| Activation rate (new user completes outcome in first session) | ~20% | 35% | `artifact_exported` or copy in session 1 |
| Second-turn rate (sessions with >=2 user turns) | ~50% | 70% | `turn_count >= 2` per session |
| Template adoption rate (among logged-in users) | 0% (new feature) | 30% of new sessions | `template_selected` / `session_started` |
| Artifact export/copy rate (template sessions) | 0% (new feature) | 50% of template sessions | `artifact_exported` per `session_type = template_guided` |
| Iteration suggestion acted rate | 0% (new feature) | 40% of template sessions have >=1 suggestion clicked | `iteration_suggestion_acted.action = clicked` |
| Trust label expansion rate | 0% (new feature) | 20% of factual sessions open Assumptions or Needs verification | `trust_label_expanded` per eligible artifact |

### Guardrail metrics

| Metric | Constraint | Owner |
|---|---|---|
| Generation P95 latency (template sessions) | Must not exceed baseline +500ms | Platform |
| Trust label fallback rate | <5% of eligible sessions | ML |
| Template slot abandonment rate | <30% of started template flows | Product |
| Downvote rate (template sessions) | Must not increase vs. open sessions | Quality |
| Free-to-paid conversion rate | Must not decrease | Growth |
| Session abandonment rate | Must not increase vs. pre-launch baseline | Retention |

### SLO definitions

| SLO | Target | Measurement window | Alert threshold |
|---|---|---|---|
| Session context API availability | 99.5% | Rolling 7 days | <99.0% triggers incident |
| Template generation availability | 99.5% | Rolling 7 days | <99.0% falls back to open session |
| Artifact export availability | 99.9% | Rolling 7 days | <99.5% pages on-call |

---

## 10) Experimentation strategy

### Experiment 1 - Template picker at session start (primary)

**Hypothesis:** Users who are shown a template picker at session start have a higher first-session artifact export rate than users who see only the blank prompt.

| Parameter | Value |
|---|---|
| Type | A/B (50/50 split) |
| Randomisation unit | User (consistent across sessions) |
| Control | Current blank-prompt session start |
| Treatment | Template picker shown at session start |
| Primary metric | Artifact export/copy rate in first session |
| Secondary metric | Second-turn rate; D7 return rate |
| Guardrail | Session abandonment rate must not increase; generation P95 latency must not exceed baseline +500ms |
| Min detectable effect | +10pp export rate (0% -> 10%) |
| Estimated runtime | 3 weeks (90% power, ~2M new sessions/week) |
| Holdout | 10% excluded for long-run retention measurement |

### Experiment 2 - Iteration suggestions placement

**Hypothesis:** Clickable iteration suggestion chips below the artifact produce higher second-turn rates than no suggestions, without degrading artifact quality perception.

| Parameter | Value |
|---|---|
| Type | A/B/C (3 variants) |
| Variants | (A) No suggestions (control), (B) Static suggestions (hardcoded per template), (C) Dynamic suggestions (generated per artifact) |
| Primary metric | Second-turn rate |
| Secondary metric | Downvote rate on iterated artifacts; suggestion click-through rate |
| Guardrail | Generation latency for variant C must not exceed variant B +800ms |
| Estimated runtime | 4 weeks |

### Experiment 3 - Trust label opt-in vs. default-on

**Hypothesis:** Default-on trust labels increase downvote rate reduction vs. opt-in, because the majority of users who benefit from them would not actively enable them.

| Parameter | Value |
|---|---|
| Type | A/B/C |
| Variants | (A) No trust labels (control), (B) Trust labels default-on, (C) Trust labels opt-in via "Show assumptions" link |
| Primary metric | Downvote rate on factual template sessions |
| Secondary metric | Trust label expansion rate; session completion rate |
| Guardrail | Trust label fallback rate <5%; no latency regression |
| Estimated runtime | 3 weeks |

---

## 11) Risks + mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Templates feel like forms; reduce conversational quality perception | High | High | Keep slots to 3 required max; infer from prompt; allow "skip" on all optional slots; A/B test slot count |
| Trust labels become boilerplate; users stop reading | Medium | Medium | Use dynamic content per artifact, not generic disclaimers; measure expansion rate; remove if <5% open rate at 60 days |
| Context capture creates privacy concern (users paste sensitive data) | Medium | High | Clear in-session disclosure; opt-in for free users; no storage of raw slot fills beyond session; GDPR/CCPA review pre-launch |
| Iteration suggestions feel pushy; users dismiss all | Medium | Medium | Cap at 3; auto-hide after 30s inactivity; measure dismiss rate; remove feature if dismiss rate >70% |
| Cost to serve increases with template-structured generations | Low | Medium | Templates add ~400 tokens overhead per session; monitor cost per session delta; gate on Plus tier if cost materially increases |
| Template adoption is low (users prefer open chat) | High | Medium | This is expected - target is 30% adoption, not 100%; treat open sessions as the default and templates as an opt-in power feature |
| Competitors ship similar features before launch | Medium | Medium | Competitive feature timing is not a launch gate; ship and iterate; our moat is session quality + model quality combined |
| Session context persistence increases security attack surface | Low | High | Encrypt at rest; scope context to session owner only; no cross-user context sharing in v1; security review pre-Phase 1 |

---

## 12) Open questions

- What is the minimum slot count before template adoption drops below 20%? (Experiment 1 will inform, but pre-launch user research needed.)
- For builders (code debugging template), is the in-chat template pattern the right surface, or should this be a separate "debug mode" within the code execution environment?
- Does session context persistence create a meaningful retention lift for free users sufficient to justify the opt-in complexity?
- What is the right "definition of done" trigger for showing the "Export as..." button - 60 seconds of inactivity, or an explicit "I'm done" signal?
- How should templates interact with existing Custom Instructions and memory? Does template context take precedence, or should it compose?

---

*All metrics are directional estimates based on public information and observable UX patterns, not internal OpenAI data.*
