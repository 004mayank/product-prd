# ChatGPT — PRD v1

> Ref teardown: https://github.com/004mayank/product-teardowns/blob/main/chatgpt-teardown.md
>
> Version: v1
> Status: Draft
> Owner: PM
> Last updated: 2026-04-08

## 1) Summary
ChatGPT’s core value is **helpfulness at speed**: compressing “time-to-clarity” for knowledge work through drafting, synthesis, and iterative refinement.

This v1 PRD focuses on the highest-leverage product direction implied by the teardown: moving from “a clever chat box” to a **guided, outcome-first work session** with improved trust calibration and reduced context friction.

## 2) Problem
### 2.1 User problems
1) **Blank prompt problem**
- Many users don’t know what to ask or how to ask it.

2) **Iteration isn’t learned**
- Users treat ChatGPT like search; they don’t realize follow-ups are the superpower.

3) **Context friction**
- Getting good outputs requires supplying constraints/files; long threads get messy.

4) **Trust & calibration**
- Hallucinations and inconsistent quality erode confidence.
- Users can’t easily tell what to rely on vs verify.

5) **Agentic expectation mismatch**
- Users ask for “do it end-to-end,” but the system may stall, ask too many questions, or produce non-shippable output.

### 2.2 Why now
The product is now a bundle of loops (chat + tools + multimodal + memory/workspaces). Competitive advantage increasingly depends on **outcomes**, not just answers.

## 3) Goals / Non-goals
### 3.1 Goals
- Increase **Weekly successful outcomes per active user**.
- Increase activation: more new users reach a “surprisingly useful” outcome quickly.
- Improve trust calibration (lower “wrong”/downvotes; reduce “had to verify elsewhere”).
- Reduce context overhead for returning users.

### 3.2 Non-goals (v1)
- Fully autonomous, no-checkpoint agents across third-party systems.
- Enterprise team features (permissions, audit logs) — later.
- Perfect factuality; instead ship **better confidence UX + verification pathways**.

## 4) Users & use cases
### 4.1 Personas (from teardown)
- Knowledge worker (PM/analyst/marketer/founder)
- Builder (engineer/data/no-code)
- Student/learner
- Ops/support/sales
- Curious consumer

### 4.2 Primary v1 use cases (broad + artifact-driven)
- Draft a document: memo, PRD, email, plan.
- Produce structured thinking: options, trade-offs, decision doc.
- Debug/implement: code generation, debugging plan, architecture outline.
- Learn: explanation + examples + practice plan.

## 5) North Star & metrics
### 5.1 North Star (candidate)
**Weekly successful outcomes per active user**

Operational definition (instrumentable proxies):
- Session ends with a **shippable artifact** (copy/save/export event)
- Or explicit “done/satisfied” feedback
- Or return-to-project behavior within 7 days (for workspace-backed sessions)

### 5.2 Input metrics
Activation
- % new users reaching “successful outcome” within first N minutes

Engagement quality
- Iteration depth to success (median turns)
- Tool adoption rate (files/browsing/images/voice) where relevant

Trust
- Downvote rate
- “Had to verify elsewhere” survey rate
- Safety incident rate / refusal calibration

Retention
- Return within 7 days

## 6) Product principles (v1)
- **Outcome-first**: start from what the user wants to ship.
- **Guided, not gated**: structure helps but doesn’t feel like a form.
- **Trust is a UX**: separate assumptions vs facts; show uncertainty.
- **Context should compound**: reuse goals/files/decisions across sessions.
- **Checkpoints over autonomy**: clarify, draft, review, finalize.

## 7) Proposed v1: Outcome-first guided sessions
### 7.1 Feature set
#### A) Outcome templates (persona-aware)
Provide a small set of “starter kits” that capture minimal constraints and produce a structured artifact.

Initial templates (v1):
1. **Write a PRD / product brief**
2. **Executive summary / memo**
3. **Debug plan / implementation plan**
4. **Study plan / tutoring session**
5. **Project plan / checklist**

Each template includes:
- Required slots (3–6): goal, audience, constraints, success metrics
- Optional: files/links, tone, format
- Definition of done checklist

#### B) Iteration coaching
After first draft, suggest 2–3 high-signal follow-ups:
- Tighten scope
- Add alternatives
- Add risks/mitigations
- Convert to final format

#### C) Trust & confidence UX
Add explicit sections to outputs:
- **Assumptions** (what the model inferred)
- **Confident** (high certainty)
- **Needs verification** (uncertain or potentially wrong)

Optional: “verification checklist” for factual outputs.

#### D) Lightweight workspace/context capture
Within a session (and optionally across sessions), store:
- Goal
- Constraints
- Sources (files/links)
- Decisions (user-approved)
- Outputs (artifacts)

Purpose: reduce re-explaining and thread bloat.

#### E) Shippable handoff
- Copy/export presets: Markdown + clean headings
- Consistent formatting for docs/tools

### 7.2 User journey (happy path)
1. User enters outcome: “Help me ship X”
2. System selects a template (or user picks)
3. Ask minimal questions (3–6)
4. Generate v1 artifact
5. Show assumptions + suggested next steps
6. User iterates; export/copy final

## 8) MVP scope
Ship:
- 5 templates (above)
- Trust labels (Assumptions / Needs verification)
- Iteration suggestions
- Export: Markdown (copy/export)
- Basic instrumentation for activation/outcome proxies

## 9) Risks & mitigations
- **Templates feel like forms** → keep minimal; allow “skip”; infer from prompt.
- **Confidence UX becomes trust theater** → start conservative; evaluate with user feedback.
- **Context capture privacy concerns** → clear controls; avoid storing sensitive data by default.
- **Cost to serve** → limit heavy tool routing; cache drafts where possible.

## 10) Rollout plan
- Internal beta with power users
- Metrics focus: activation, outcome proxy events, trust feedback
- Iterate template set based on retention + export/copy frequency

## 11) Open questions
- Which 3–5 workflows drive the majority of paid retention?
- What trust UI improves confidence without slowing users down?
- What’s the minimum viable “workspace” that users understand?
