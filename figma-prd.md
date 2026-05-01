# Figma PRD: Engineering Review Activation - Closing the Cross-Functional Handoff Gap

**Version:** v1 - Initial PRD
**Changes from v0:** N/A - initial version
**Source teardown:** figma-teardown.md (v3)
**Lens:** Product Manager - scope is the designer-to-engineer handoff activation problem

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Core problem thesis, user segments, activation funnel analysis, solution pillars, success metrics, non-goals, open questions, key risks |

---

## TL;DR

Figma's D30 retention roughly doubles when a non-designer opens a design file within 7 days of creation. Yet an estimated 50%+ of new files are never opened by an engineer or PM before engineering picks up the work. The problem is structural: the designer can share a link, but Figma has no product motion that creates a compelling reason for an engineer to switch contexts and open that link. This PRD proposes a targeted handoff trigger system - detecting review-ready files, meeting engineers in Linear and GitHub, and giving first-time engineering openers a role-specific onboarding flow - to drive `file_opened_by_non_editor_within_7d_rate` from ~35% to 50%.

---

## 1. Problem

### The failure loop

When a designer finishes a set of screens, the standard handoff today is:

1. Designer shares the Figma file link in a Slack message or Linear comment.
2. Engineer does not click the link - they are mid-sprint, reviewing PRs, or don't see the message as blocking.
3. Engineer starts building from memory, a screenshot, or a verbal brief.
4. Two weeks later, the shipped feature diverges from the design - rework begins.
5. Designer concludes "engineers don't use Figma" and stops sharing design files proactively.
6. Figma file goes stale; team falls back to PDF specs or verbal briefings.
7. Figma loses its value as the shared source of truth - and loses the activation event that drives retention.

This is not an engineer behaviour problem. It is a product problem. Figma has not given engineers a reason to switch contexts that is more compelling than "the designer asked me to look at this." The mechanism that connects "designer finishes work" to "engineer opens file" is entirely informal and entirely outside the product.

### Why this is the right problem to solve

From the teardown analysis:

- Teams where a non-designer opens a file within 7 days show ~2x better M3 retention vs. teams where no non-designer ever opens.
- Non-designer commenting on a file correlates with +50% D30 retention above the "file opened" milestone alone.
- Dev Mode activation by an engineering team within 30 days of a design file being shared correlates with ~85% probability of active Figma use at M6.
- `library_first_consumed_by_second_team` is the long-term retention lock-in event - but it cannot happen if the cross-functional loop never fires in the first place.

The cross-functional activation cliff is the single highest-leverage funnel problem in Figma's PLG motion. Fixing the share flow, improving library onboarding, or shipping AI features will all compound less if engineers do not first build a habit of opening design files.

### Observable signals of the failure today

- Default share behaviour produces a "view only" link pasted into Slack - no notification, no context, no pull toward opening.
- Engineers who open a Figma file for the first time see the design canvas (editor mode), not Dev Mode. Without knowing to switch modes, they get no value from the inspection layer - so they leave.
- The `file_opened_by_non_editor` rate is not surfaced to designers today - they have no visibility into whether their handoff was received.
- There is no "ready for review" state in Figma's product model. A file in active design looks identical to a file ready for engineering handoff.

---

## 2. Goals and non-goals

### Goals

| Goal | Target metric | Timeline |
|---|---|---|
| Increase `file_opened_by_non_editor_within_7d_rate` for new design files | 35% -> 50% (relative +43%) | 2 quarters post-GA |
| Increase first comment by a non-designer on a shared file | 20% -> 30% of files opened by non-designer | 2 quarters post-GA |
| Increase Dev Mode session initiation rate for engineering viewers | 30% -> 45% within 30 days of file creation | 2 quarters post-GA |
| Reduce P50 time from `file_shared` to `file_opened_by_non_editor` | >48h -> <6h | 2 quarters post-GA |
| Increase `devmode_session_started` on first engineering file open | 20% -> 50% | 1 quarter post-GA (Pillar 2 only) |

### Non-goals

- Not redesigning Dev Mode inspection UX or changing its core feature set.
- Not changing the viewer-free model - engineers remain free to open and view files.
- Not building a project management or sprint planning tool - Linear, Jira, and GitHub remain the engineering home.
- Not targeting FigJam (separate product surface with a different activation model).
- Not modifying the design file format, component model, or Auto Layout behaviour.
- Not gating any part of this feature behind a Dev Mode paid tier - activation must precede monetisation.

---

## 3. User segments and personas

| Persona | JTBD | Current behaviour | Activation blocker |
|---|---|---|---|
| Product designer | Ship features that look and behave as designed; avoid rework cycles from misinterpretation | Creates file, shares link in Slack or Linear comment, then waits | No visibility into whether the engineer opened the file; no mechanism to prompt re-engagement if they didn't |
| Frontend engineer | Build features that match the spec without extra back-and-forth; avoid rework cycles | Opens Figma only when explicitly blocked on a measurement or asset; switches to Dev Mode only if they already know it exists | No workflow-native reason to open Figma proactively; Figma link in Slack competes with 40+ other messages; does not know Dev Mode exists or what it offers |
| Product manager | Review and approve designs before engineering picks up the ticket; catch UX problems before build cost is incurred | Reviews in Slack screenshots or opens Figma once and doesn't return; marks the Linear ticket as "approved" without a Figma-native review event | No notification when designs are updated after initial share; no single "review this" moment in the product |
| Design systems lead | Ensure all shipped UI uses current design system components; prevent design debt from entering production via unchecked handoffs | Reviews files ad hoc when flagged; has no scalable way to audit which files going to engineering are design-system-compliant | No visibility into the pipeline of files approaching handoff; no way to intercept a non-compliant file before it reaches engineering |

---

## 4. Solution pillars

### Pillar 1 - Review-ready signal and handoff trigger

**Problem it solves:** There is no explicit "ready for engineering review" state in Figma. Files transition from design to handoff invisibly. Engineers cannot tell whether a Figma link is a WIP sketch or a finished spec.

**What it does:** Detect when a design file crosses a review-ready threshold and prompt the designer to initiate a formal handoff, generating a higher-signal notification for engineers than an informal Slack link.

**Review-ready detection heuristic (v1):**

| Signal | Criteria | Weight |
|---|---|---|
| Frame count | >= 3 frames on the primary page | Required |
| Prototype connectivity | At least one prototype connection between frames | Required |
| Recent activity | File edited in the last 72h | Required |
| Open comments | Zero unresolved comments (or all comments resolved) | Required |
| Previous handoff | No prior `handoff_triggered` event on this file | Suppression |
| Engineer already present | No `file_opened_by_non_editor` event in last 24h | Suppression |

When all required criteria are met and no suppression conditions apply:
- Surface an inline prompt in the top bar: "This file looks ready for engineering review. Start the handoff?"
- Prompt appears once per file; can be dismissed with "Not yet" (re-surfaces after 72h of additional edits).
- If the designer confirms, the share modal pre-populates with "Dev Mode" access (not view-only) and prompts for a specific recipient.
- File gets a visible "Ready for review" label in the project browser, visible to all team members.

**Edge case 1 - Designer is still iterating:** File meets heuristic criteria (enough frames, no open comments) but work is not actually complete. Mitigation: require explicit confirmation ("Yes, send to engineering") before triggering; suppress re-prompt for 72h; allow "mark as WIP" to disable the trigger for 7 days.

**Edge case 2 - Engineer is already in the file:** If `file_opened_by_non_editor` fired in the last 24h, suppress the trigger - the handoff has already happened informally. Do not introduce friction into an already-working workflow.

---

### Pillar 2 - Engineering-first onboarding overlay for first-time openers

**Problem it solves:** Engineers who open a Figma file for the first time see the design canvas in editor mode. Without knowing that Dev Mode exists, they get no actionable value - they copy a screenshot and leave. The product does not explain what they are supposed to do.

**What it does:** When an engineer opens a Figma file for the first time (or first time in 90 days), show a 3-step onboarding overlay that is specific to the engineering use case - not a generic "welcome to Figma" tour.

**Overlay content (3 steps, dismissable at any point):**

Step 1: "Switch to Dev Mode to see measurements, CSS, and design tokens."
- CTA: "Switch to Dev Mode" (executes the mode switch immediately).
- Sub-text: "Free for engineering viewers."

Step 2: "Click any element to copy its code."
- Highlights the code panel in Dev Mode.
- CTA: "Try it" (focuses the code panel on the first frame's primary element).

Step 3: "Leave a comment if something looks different from what you built."
- Highlights the comment tool.
- CTA: "Got it - start reviewing."

**Trigger conditions:**
- First-ever file open by this user across all Figma files (new engineer to Figma).
- OR: first file open in 90+ days (lapsed engineer).
- OR: first open of a file that has the "Ready for review" label (regardless of prior Figma experience).

**Suppression conditions:**
- User is an editor or design-mode active user (skip - they know Figma).
- User has completed this overlay in the last 90 days (show at most once per quarter).
- User explicitly clicked "I know Figma, skip this" (permanent suppression flag on user record).

**Edge case - Experienced engineer who finds this patronising:** Show "I know Figma, skip this" prominently in step 1. One click suppresses permanently. Do not re-show. The overlay should feel like a helpful tooltip for someone who got a link and isn't sure what to do with it - not a tutorial for someone who uses Figma daily.

---

### Pillar 3 - Linear and GitHub workflow integration for design review requests

**Problem it solves:** Engineers live in Linear and GitHub, not Slack. A Figma link buried in a Slack message competes with 40 other messages and has no relationship to the engineer's current task. A Figma link surfaced inside the task the engineer is already working on is 10x higher-signal.

**What it does:**

**Linear integration (build on existing Figma-Linear integration):**
- When a designer marks a file as "Ready for review" via the Pillar 1 trigger, and the file has a linked Linear issue, auto-create a "Design review" sub-task on that issue assigned to the lead engineer on the ticket.
- Sub-task description: "Review design in Figma before implementation: [file link - opens in Dev Mode]."
- Sub-task auto-closes when `file_opened_by_non_editor` fires and is followed by a `comment_created` event within 7 days.

**GitHub integration (new):**
- When a PR is opened against a branch whose associated Linear issue has a linked Figma file, surface a collapsible "Design spec" section in the PR description with a thumbnail and a direct Dev Mode link.
- Template injection: adds one line to the PR body - "Design spec: [Figma file - Dev Mode]" - if the field is not already present.
- This is additive to the PR template; it never overwrites existing content.

**Scope for v1:** Linear and GitHub only. Jira and Notion are explicitly out of scope for v1 (noted as v2 targets in the open questions).

**Edge case - File not linked to any Linear issue:** Pillar 3 does not fire. The designer is prompted in the share modal to link to a Linear issue if they want the workflow integration. No forced linking - it is optional and surfaced as a value-add.

---

### Pillar 4 - "Designs waiting for your review" digest

**Problem it solves:** Engineers who were shared a design file but didn't open it have no re-engagement path. The Slack message is buried; there is no follow-up product touch from Figma.

**What it does:** A weekly (configurable) email and in-app notification digest for engineering viewers showing design files that have been shared with them and have not been opened or commented on.

**Digest content:**
- Subject: "3 designs are waiting for your review" (number is accurate, not inflated).
- Each entry: file thumbnail (first frame), file name, designer name, time since sharing, and a "Review in Dev Mode" CTA (deep link to the file in Dev Mode, not editor mode).
- Grouped by project or team.
- Maximum 5 files per digest - if more than 5 are pending, show the 5 most recently shared.

**Configuration:**
- Default: weekly, sent Monday morning in the recipient's timezone.
- User can change to: daily, bi-weekly, or off.
- Unsubscribe is prominent and one-click - no confirmation required.
- If there are zero pending files, no digest is sent that week.

**Guardrail:** If the digest unsubscribe rate exceeds 15% in the first 30 days, pause the digest and audit content relevance before re-enabling.

---

## 5. Key metrics

### North Star metric

`file_opened_by_non_editor_within_7d_rate` - the percentage of new design files (created by a user with editor access) where at least one non-designer (viewer or Dev Mode role) opens the file within 7 calendar days of file creation.

**Baseline (inferred):** ~35%
**Target:** >50% within 2 quarters of full rollout.
**Rationale:** This is the single best leading indicator of cross-functional activation and the strongest predictor of D30 and M3 retention. Every other metric in this PRD is an input to or guardrail on this number.

### Input metrics (leading indicators)

| Metric | Event | Baseline (inferred) | Target | Pillar driving it |
|---|---|---|---|---|
| % of new files receiving a "Ready for review" label | `handoff_triggered` | ~0% (feature doesn't exist) | >40% of files with >= 3 frames | 1 |
| Engineer first Dev Mode session rate within 30 days | `devmode_session_started` after `file_opened_by_non_editor` | ~30% | >45% | 1 + 2 |
| `devmode_session_started` on first-ever file open by engineer | Rate among users with `is_first_figma_open = true` | ~20% | >50% | 2 |
| Engineer comment rate on design files within 14 days | `comment_created` with `commenter_role = viewer or dev_mode` | ~15% | >25% | 1 + 3 |
| P50 time from `file_shared` to `file_opened_by_non_editor` | Time delta between events | >48h | <6h | 3 |
| Digest email open rate | Email analytics | N/A (new) | >25% | 4 |
| Linear "Design review" sub-task completion rate | Sub-task closed with `comment_created` present | N/A (new) | >60% of auto-created sub-tasks | 3 |

### Guardrail metrics

| Metric | Threshold | Why it matters |
|---|---|---|
| Time-to-first-frame for designers in the activation funnel | Must not increase >5% vs. pre-launch baseline | Handoff prompts and "ready for review" nudges must not interrupt design flow |
| Digest unsubscribe rate in first 30 days | Must not exceed 15% - trigger a content/cadence review if breached | Digest is additive value, not notification spam |
| `file_shared` rate per designer per week | Must not decline - new prompts should not create friction that reduces sharing | If designers share less, the funnel narrows regardless of downstream improvements |
| Designer NPS for the share flow (in-product survey, n >= 200) | Must not decline >5 points vs. pre-launch baseline | Designer sentiment protects the top of the funnel |

---

## 6. Key risks

| Risk | Likelihood | Severity | Mitigation |
|---|---|---|---|
| Review-ready heuristic fires when file is still a WIP | High | Medium | Require explicit designer confirmation; 72h suppression after dismissal; "Mark as WIP" escape hatch |
| Engineering onboarding overlay is perceived as patronising | Medium | Medium | Prominent "I know Figma, skip this" in step 1; permanent suppression flag; show only on first open or first file with "Ready for review" label |
| Linear/GitHub integrations break existing PR templates or issue workflows | Medium | High | Additive-only changes to PR bodies; no overwriting of existing content; rollout to opt-in teams first; one-click disable per org |
| Digest email contributes to notification fatigue and increases unsubscribe rate from all Figma emails | Medium | High | Weekly default (not daily); aggressive unsubscribe UX; strict volume cap at 5 files per digest; pause if unsubscribe rate > 15% |
| Dev Mode paid tier is introduced before engineers build an engagement habit | High | High | Engineering activation (`devmode_session_started` rate > 40% for engineering teams) must be a prerequisite gate before any Dev Mode paid tier launch |
| Pillar 3 scope creep into project management territory (competing with Linear) | Low | High | Strict non-goal enforcement: Figma creates one sub-task and one PR annotation, nothing more. No sprint boards, no issue creation, no status sync beyond the design review event. |

---

## 7. Competitive context

| Competitor | Their handoff model | Figma's gap vs. them | Figma's advantage |
|---|---|---|---|
| Zeplin | Dedicated handoff tool; engineers get a separate inspection app with annotations, styleguides, and comment threads | Zeplin has a purpose-built engineer-first UI; its "project" model makes design-to-engineering handoff an explicit workflow step | Figma has the design file - Zeplin requires a sync step. Fixing the handoff trigger closes most of the gap without requiring a separate tool. |
| Notion + Figma embed | PMs embed Figma prototypes in Notion specs; engineers read the spec doc | The design source of truth is fragmented - Notion has the context, Figma has the design | Figma's Linear integration creates the same "spec-adjacent" surface inside the engineering workflow tool, without requiring a PM to manually embed |
| Storybook / Code Connect | Engineers review component behaviour in code, not in design | Storybook is post-build review; Figma is pre-build spec. These should be complementary but feel disconnected | Dev Mode + Code Connect is the bridge - but only if engineers are first activated in Figma before the Storybook habit forms |
| Linear + screenshots | Teams paste screenshots of Figma frames into Linear comments as the handoff artefact | Absolutely the status quo for many teams - screenshots degrade in resolution, go stale, and have no inspection layer | The Pillar 3 Linear integration replaces the screenshot with a live Dev Mode link inside the same Linear ticket |

---

## 8. Open questions (v1)

1. What is the actual `file_opened_by_non_editor_within_7d_rate` baseline today? The 35% estimate is inferred from retention correlation data; internal Figma analytics would set a precise baseline and determine how aggressive a 50% target is.
2. Does the existing Figma-Linear integration already support the "ready for review" sub-task creation, or is this net-new infrastructure?
3. What is the P50 time from `file_shared` to `file_opened_by_non_editor` in the current dataset? This directly determines whether Pillar 3 (meeting engineers in their tools) or Pillar 4 (digest re-engagement) is the higher-leverage bet.
4. Is there a meaningful difference in activation rate between files shared with a specific named recipient vs. files shared via "anyone with the link"? If named-recipient shares already activate at >50%, the problem is concentrated in link-only shares.
5. What % of engineering teams with an active Figma file have ever initiated a Dev Mode session? If this is already >50%, Pillar 2's target needs to be re-baselined.
6. Can the review-ready detection heuristic be validated with a lightweight user study (5-10 designer pairs reviewed with the heuristic applied to their real files) before building the full trigger system?
7. Is there a Jira integration roadmap that would allow Pillar 3 to cover the significant segment of engineering teams not using Linear?

---

*All baselines are directional estimates inferred from public Figma retention research, teardown analysis, and industry benchmarks. Internal Figma analytics should be used to validate or correct these before v2 requirements are finalised.*
