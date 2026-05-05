# Figma PRD: Engineering Review Activation - Closing the Cross-Functional Handoff Gap

**Version:** v3 - Final PRD
**Changes from v2:** Added explicit launch gates and kill switches per pillar, full instrumentation spec aligned to each rollout phase, SLO revision history, resolved open questions from v2 (baseline validation decisions, Jira roadmap, share-type hypothesis confirmation path), added post-GA monitoring cadence and incident response triggers, updated version history table.
**Source teardown:** figma-teardown.md (v3)
**Lens:** Product Manager - scope is the designer-to-engineer handoff activation problem

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Core problem thesis, user segments, activation funnel analysis, solution pillars, success metrics, non-goals, open questions, key risks |
| v2 | Detailed funnel breakdown, functional requirements + acceptance criteria per pillar, full event schema, experiment backlog, competitive analysis with specific win/loss, edge cases per feature, dependency map, instrumentation spec, resolved open questions |
| v3 | Explicit launch gates and kill switches per pillar, full instrumentation spec aligned to rollout phases, SLO revision history, resolved open questions from v2, post-GA monitoring cadence, incident response triggers |

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

## 4. Activation funnel analysis

### Current state (inferred baseline)

The cross-functional activation funnel has two documented failure points - the share-to-open cliff and the open-to-engage cliff:

| Stage | Event | Inferred rate | Primary failure mode |
|---|---|---|---|
| File created | `file_created` | 100% baseline | - |
| File shared with at least one non-designer | `file_shared` (viewer or dev_mode recipient) | ~55% of new files | Designer does not share proactively because they expect verbal briefing |
| Non-designer opens the file within 7 days | `file_opened_by_non_editor` (within 7d) | ~35% of shared files (~19% of created files) | Engineer gets the link but doesn't prioritise switching contexts |
| Non-designer switches to Dev Mode | `devmode_session_started` after `file_opened_by_non_editor` | ~30% of openers | Engineer opens file in canvas mode; doesn't know Dev Mode exists or how to activate it |
| Non-designer leaves a comment | `comment_created` (commenter_role: viewer or dev_mode) | ~20% of Dev Mode sessions | Engineer copies what they need and leaves without commenting; no prompt to do so |
| Comment triggers a reply from designer | `comment_resolved` or `reply_created` within 48h | ~60% of comments | Designer is responsive; this stage is relatively healthy once comments are created |

### Stage-level interventions this PRD targets

| Funnel stage | Pillar addressing it | Expected lift |
|---|---|---|
| Created -> Shared | Pillar 1 (review-ready prompt encourages explicit share action) | +10pp share rate |
| Shared -> Opened within 7d | Pillar 3 (Linear/GitHub integration puts link in engineering workflow) + Pillar 4 (digest re-engagement) | +15pp open rate |
| Opened -> Dev Mode started | Pillar 2 (engineering-first onboarding overlay) | +20pp Dev Mode activation on first open |
| Dev Mode -> Comment | Pillar 2 step 3 (comment prompt as onboarding CTA) | +10pp comment rate |

### Cohort retention impact (inferred)

| Cross-functional activation milestone | Inferred D30 retention lift | Current cohort size (est.) |
|---|---|---|
| File shared with at least one non-designer | +40% vs. solo-designer files | ~55% of new design files |
| Non-designer opens within 7 days | +35% above "shared" cohort | ~35% of shared files |
| Non-designer leaves a comment | +50% above "opened" cohort | ~20% of opened files |
| Dev Mode session by engineer within 30d | +60% M3 retention for team | ~30% of opener cohort |

---

## 5. Solution pillars - functional requirements and acceptance criteria

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

#### Functional requirements - Pillar 1

| # | Requirement | Priority |
|---|---|---|
| P1-1 | System evaluates all four required heuristic signals on every file save that results in a state change (frame added, comment resolved, prototype link created) | P0 |
| P1-2 | When all required signals are met and no suppression condition applies, show an inline prompt in the top bar within 500ms of the triggering save | P0 |
| P1-3 | Prompt reads: "This file looks ready for engineering review. Start the handoff?" with two CTAs: "Start handoff" and "Not yet" | P0 |
| P1-4 | "Not yet" dismisses the prompt and suppresses re-evaluation for 72 hours from the dismissal timestamp | P0 |
| P1-5 | "Mark as WIP" option (accessible from the "..." overflow on the prompt) suppresses all handoff trigger evaluation for 7 days | P1 |
| P1-6 | Designer confirms via "Start handoff" -> share modal opens pre-populated with Dev Mode access (not view-only) and a required recipient field | P0 |
| P1-7 | After successful share via the handoff flow, the file displays a "Ready for review" label in the project browser visible to all team members | P0 |
| P1-8 | "Ready for review" label is automatically removed if the file is edited (any save event) more than 24 hours after the label was applied, unless the designer re-confirms | P1 |
| P1-9 | Trigger evaluation runs server-side on save events; the inline prompt is served via a WebSocket push, not a client-side poll | P0 |
| P1-10 | Feature is disabled for files owned by solo accounts (no team) - the trigger requires a team context to have meaning | P0 |

#### Acceptance criteria - Pillar 1

| ID | Scenario | Pass condition |
|---|---|---|
| AC-P1-1 | File meets all 4 required signals; no suppression applies | Prompt appears in top bar within 500ms of qualifying save |
| AC-P1-2 | Designer clicks "Not yet" | Prompt disappears; re-evaluation is suppressed for exactly 72h |
| AC-P1-3 | Designer clicks "Start handoff"; selects an engineer recipient; submits | File receives "Ready for review" label; `handoff_triggered` event fires; `file_shared` fires with `share_type: dev_mode` |
| AC-P1-4 | Designer clicks "Start handoff"; closes the share modal without submitting | No label applied; `handoff_triggered` event does NOT fire; prompt re-surfaces on next qualifying save |
| AC-P1-5 | Engineer opens file within 24h of prompt being shown (suppression condition) | Prompt is suppressed; `handoff_trigger_suppressed` event fires with `reason: engineer_already_present` |
| AC-P1-6 | File has already had `handoff_triggered` fire previously | Prompt never re-surfaces for this file; suppression is permanent unless designer resets |
| AC-P1-7 | File is edited 25h after "Ready for review" label is applied | Label is removed; designer receives in-app notification: "Your handoff label was removed because the file was updated. Re-trigger when ready." |

**Edge case 1 - Designer is still iterating:** File meets heuristic criteria (enough frames, no open comments) but work is not actually complete. Mitigation: require explicit confirmation ("Yes, send to engineering") before triggering; suppress re-prompt for 72h; allow "mark as WIP" to disable the trigger for 7 days.

**Edge case 2 - Engineer is already in the file:** If `file_opened_by_non_editor` fired in the last 24h, suppress the trigger - the handoff has already happened informally. Do not introduce friction into an already-working workflow.

**Edge case 3 - File has been cloned or branched:** If the file is a branch of a main file, the trigger should evaluate against the branch content, not the main file's state. The "Ready for review" label on a branch indicates the branch is ready to be reviewed by engineering before merging - not that the main file is ready. Fire `handoff_triggered` with `context: branch` to distinguish in analytics.

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

#### Functional requirements - Pillar 2

| # | Requirement | Priority |
|---|---|---|
| P2-1 | Overlay appears within 2 seconds of first file open; does not block the file canvas from rendering | P0 |
| P2-2 | "I know Figma, skip this" link is visible in Step 1 without scrolling; click sets `overlay_permanently_dismissed = true` on user record; overlay never shows again | P0 |
| P2-3 | "Switch to Dev Mode" CTA in Step 1 executes the mode switch immediately in the same tab without closing the overlay | P0 |
| P2-4 | "Try it" CTA in Step 2 programmatically selects the topmost interactive element in the first frame and opens the code panel; if no frames exist, step 2 is skipped | P1 |
| P2-5 | Overlay advances to next step automatically if the user takes the suggested action (mode switch, element click, comment open) within 10 seconds | P1 |
| P2-6 | Overlay state persists through page refresh; if user closes the tab mid-overlay, the overlay resumes at the last incomplete step on next open (within 24h) | P1 |
| P2-7 | Overlay fires `overlay_shown`, `overlay_step_completed`, and `overlay_dismissed` events with step number, method (action vs. manual advance), and user context | P0 |
| P2-8 | Overlay is not shown on mobile web - defer to mobile PM for bottom-sheet equivalent; desktop web only for v1 | P0 |

#### Acceptance criteria - Pillar 2

| ID | Scenario | Pass condition |
|---|---|---|
| AC-P2-1 | First-ever Figma file open by a user with no editor history | Overlay appears within 2s; canvas is fully rendered behind overlay |
| AC-P2-2 | User clicks "Switch to Dev Mode" in Step 1 | Dev Mode activates; overlay stays open and advances to Step 2 |
| AC-P2-3 | User clicks "I know Figma, skip this" | Overlay closes; `overlay_dismissed` fires with `method: permanent_skip`; overlay never shown again for this user |
| AC-P2-4 | User opens a file with the "Ready for review" label; user has completed the overlay 45 days ago | Overlay shows (less than 90-day suppression window) |
| AC-P2-5 | User opens a file with the "Ready for review" label; user has completed the overlay 95 days ago | Overlay does NOT show (past 90-day suppression window) |
| AC-P2-6 | User closes the tab on Step 2; reopens the same file within 24h | Overlay resumes at Step 2 |
| AC-P2-7 | User is an active editor (>5 editor sessions in last 30d) | Overlay is suppressed; `overlay_suppressed` fires with `reason: active_editor` |

**Edge case 1 - Experienced engineer who finds this patronising:** "I know Figma, skip this" is prominent in Step 1. One click suppresses permanently. The overlay should feel like a helpful tooltip for someone who got a link and isn't sure what to do with it - not a tutorial for someone who uses Figma daily.

**Edge case 2 - File has no frames or only one frame:** Step 2 ("Click any element to copy its code") has no useful target. Skip Step 2 and advance directly from Step 1 to Step 3. Fire `overlay_step_skipped` with `reason: no_interactive_element` for analytics. This prevents a broken "Try it" experience on empty or stub files.

**Edge case 3 - User opens file in Dev Mode directly (via a deep link with `?mode=dev`):** Overlay should skip Step 1 (already in Dev Mode) and start at Step 2. Detect mode on load; adjust overlay entry point accordingly.

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

#### Functional requirements - Pillar 3

| # | Requirement | Priority |
|---|---|---|
| P3-1 | When `handoff_triggered` fires for a file linked to a Linear issue, Figma's Linear integration creates a sub-task within 60 seconds | P0 |
| P3-2 | Sub-task assignee defaults to the first engineer listed on the Linear issue; if no engineer is listed, the sub-task is unassigned with a label "Needs assignee" | P0 |
| P3-3 | Sub-task link to Figma uses a deep link that opens the file in Dev Mode (`?mode=dev`), not the default editor mode | P0 |
| P3-4 | Sub-task auto-closes when both `file_opened_by_non_editor` and `comment_created` (commenter_role: viewer or dev_mode) fire on the linked file within 7 days of sub-task creation | P1 |
| P3-5 | When a GitHub PR is opened against a branch, Figma checks for a linked Linear issue (via PR body or branch name convention `[LIN-XXX]`) and retrieves any linked Figma file | P0 |
| P3-6 | If a Figma file is found, inject "Design spec: [file name](Dev Mode URL)" into the PR body as the last line of the description, only if the string "figma.com" is not already present in the PR body | P0 |
| P3-7 | PR body injection is performed via the GitHub API using the Figma GitHub app's installation token; injection must not modify any existing line - appended only | P0 |
| P3-8 | If the Figma GitHub app is not installed for the repo, skip injection silently; log `github_injection_skipped` with `reason: app_not_installed` | P0 |
| P3-9 | Org admins can disable Pillar 3 entirely (Linear or GitHub, independently) in the Figma admin panel; disable takes effect within 5 minutes | P1 |
| P3-10 | Per-file opt-out: designer can unlink the Linear issue from the handoff flow to prevent sub-task creation for a specific file | P1 |

#### Acceptance criteria - Pillar 3

| ID | Scenario | Pass condition |
|---|---|---|
| AC-P3-1 | `handoff_triggered` fires; file is linked to a Linear issue with an assigned engineer | Sub-task created on Linear issue within 60s; sub-task link opens file in Dev Mode |
| AC-P3-2 | `handoff_triggered` fires; file has no linked Linear issue | No sub-task created; `linear_subtask_skipped` fires with `reason: no_linked_issue` |
| AC-P3-3 | GitHub PR opened with `[LIN-123]` in the branch name; LIN-123 has a linked Figma file | "Design spec: [file name](Dev Mode URL)" appended to PR body |
| AC-P3-4 | GitHub PR body already contains "figma.com" | Injection skipped; `github_injection_skipped` fires with `reason: figma_link_present` |
| AC-P3-5 | Engineer opens file (fires `file_opened_by_non_editor`) and leaves a comment within 7d | Sub-task auto-closes; `linear_subtask_autoclosed` event fires |
| AC-P3-6 | Engineer opens file but does NOT leave a comment within 7d | Sub-task remains open; no auto-close |
| AC-P3-7 | Org admin disables Linear integration in admin panel | No sub-tasks created for any new handoff triggers in that org; disable effective within 5 min |

**Edge case 1 - File not linked to any Linear issue:** Pillar 3 does not fire. The designer is prompted in the share modal to link to a Linear issue if they want the workflow integration. Linking is optional - surfaced as a value-add, not required. `linear_subtask_skipped` fires with `reason: no_linked_issue`.

**Edge case 2 - Linear issue has multiple engineers assigned:** Sub-task assignee is the first engineer listed (by Linear's assignment order). If the linear issue has no engineers (only PMs or designers), sub-task is created unassigned with `needs_assignee: true` flag visible in the sub-task description.

**Edge case 3 - PR is opened before the design is marked "Ready for review":** GitHub injection still fires based on the linked Figma file state at PR open time. The file does not need to have a `handoff_triggered` event - the presence of a linked file is sufficient. This handles teams that do their own informal handoff before the formal trigger.

---

### Pillar 4 - "Designs waiting for your review" digest

**Problem it solves:** Engineers who were shared a design file but didn't open it have no re-engagement path. The Slack message is buried; there is no follow-up product touch from Figma.

**What it does:** A weekly (configurable) email and in-app notification digest for engineering viewers showing design files that have been shared with them and have not been opened or commented on.

**Digest content:**
- Subject: "3 designs are waiting for your review" (number is accurate, not inflated).
- Each entry: file thumbnail (first frame), file name, designer name, time since sharing, and a "Review in Dev Mode" CTA (deep link to the file in Dev Mode, not editor mode).
- Grouped by project or team.
- Maximum 5 files per digest - if more than 5 are pending, show the 5 most recently shared.

#### Functional requirements - Pillar 4

| # | Requirement | Priority |
|---|---|---|
| P4-1 | Digest is computed weekly (Monday 9am in recipient's timezone) by querying `file_shared` events where recipient has no subsequent `file_opened_by_non_editor` or `comment_created` event | P0 |
| P4-2 | Digest includes a maximum of 5 files per recipient; ranked by recency of `file_shared` event (most recent first) | P0 |
| P4-3 | Each file entry in the digest renders a thumbnail (first frame of the file, cached at 400x300px) with fallback to a file icon if thumbnail generation fails | P0 |
| P4-4 | "Review in Dev Mode" CTA uses a deep link that opens the file in Dev Mode; link includes a UTM parameter `utm_source=digest` for attribution | P0 |
| P4-5 | Unsubscribe link is visible in the email footer; one-click, no confirmation dialog required | P0 |
| P4-6 | Users can configure digest cadence (weekly, daily, bi-weekly, off) in Notification Preferences; default is weekly | P1 |
| P4-7 | No digest is sent if a recipient has zero pending files that week | P0 |
| P4-8 | If the digest unsubscribe rate exceeds 15% in the first 30 days (measured across all recipients), the digest feature is automatically paused pending a content and cadence review | P0 |
| P4-9 | In-app notification counterpart to the email digest: "3 designs need your review" banner in the Figma home screen, shown on Monday login if the email has not been clicked | P1 |

#### Acceptance criteria - Pillar 4

| ID | Scenario | Pass condition |
|---|---|---|
| AC-P4-1 | User has 3 pending design files (shared but not opened) at Monday 9am | Digest email sent; subject: "3 designs are waiting for your review" |
| AC-P4-2 | User has 0 pending files | No digest sent that week |
| AC-P4-3 | User has 7 pending files | Digest contains the 5 most recently shared; oldest 2 are excluded |
| AC-P4-4 | User clicks "Review in Dev Mode" in the digest | File opens in Dev Mode; `digest_cta_clicked` event fires with `utm_source: digest` |
| AC-P4-5 | User clicks "Unsubscribe" in the email footer | User is unsubscribed immediately; no confirmation required; `digest_unsubscribed` event fires |
| AC-P4-6 | 30-day unsubscribe rate exceeds 15% | Digest sending is automatically paused; ops team receives an alert |

**Edge case 1 - Designer is also the engineer on the same team:** A user who is both an editor and listed as an engineering reviewer on some files could receive a digest about their own files. Mitigation: suppress digest entries for files where the recipient is the file's primary creator (owner).

**Edge case 2 - Engineer has already reviewed the file via an informal channel (opened from Slack link, but `file_opened_by_non_editor` did not fire due to a tracking gap):** The digest will incorrectly include this file. This is a known limitation of inferring review state from events. Mitigation: add a "Mark as reviewed" action directly in the digest email - a single click that fires `digest_marked_reviewed` and suppresses the file from future digests. No Figma login required to mark as reviewed (use an unguessable token in the URL).

---

## 6. Event schema

All new events introduced by this PRD. Field names use snake_case. Events fire server-side where possible to avoid client-side ad blocker interference.

```json
// Handoff trigger prompt shown to designer
{
  "event": "handoff_trigger_shown",
  "properties": {
    "file_id": "string",
    "team_id": "string",
    "org_id": "string",
    "trigger_reason": "frame_count | prototype_connected | comments_resolved | all_criteria_met",
    "frame_count": "integer",
    "has_prototype": "boolean",
    "open_comment_count": "integer",
    "ts": "ISO8601"
  }
}
```

```json
// Designer completes the handoff trigger (shares file via the handoff flow)
{
  "event": "handoff_triggered",
  "properties": {
    "file_id": "string",
    "team_id": "string",
    "org_id": "string",
    "recipient_count": "integer",
    "recipient_roles": ["dev_mode", "viewer"],
    "linked_linear_issue": "boolean",
    "context": "main_file | branch",
    "ts": "ISO8601"
  }
}
```

```json
// Handoff trigger prompt suppressed (engineer already present, or repeat trigger)
{
  "event": "handoff_trigger_suppressed",
  "properties": {
    "file_id": "string",
    "team_id": "string",
    "reason": "engineer_already_present | prior_handoff_exists | wip_marked | 72h_cooldown",
    "ts": "ISO8601"
  }
}
```

```json
// Engineering onboarding overlay shown
{
  "event": "overlay_shown",
  "properties": {
    "file_id": "string",
    "user_id": "string",
    "trigger_reason": "first_ever_open | lapsed_90d | ready_for_review_file",
    "starting_step": "integer",
    "ts": "ISO8601"
  }
}
```

```json
// Overlay step completed or skipped
{
  "event": "overlay_step_completed",
  "properties": {
    "file_id": "string",
    "user_id": "string",
    "step_number": "integer",
    "method": "action_taken | manual_advance | auto_advance",
    "skipped": "boolean",
    "skip_reason": "string | null",
    "ts": "ISO8601"
  }
}
```

```json
// Overlay dismissed at any step
{
  "event": "overlay_dismissed",
  "properties": {
    "file_id": "string",
    "user_id": "string",
    "dismissed_at_step": "integer",
    "method": "close_button | permanent_skip | completed_all_steps",
    "ts": "ISO8601"
  }
}
```

```json
// Linear sub-task created for design review
{
  "event": "linear_subtask_created",
  "properties": {
    "file_id": "string",
    "linear_issue_id": "string",
    "subtask_id": "string",
    "assignee_present": "boolean",
    "ts": "ISO8601"
  }
}
```

```json
// Linear sub-task auto-closed after engineer review
{
  "event": "linear_subtask_autoclosed",
  "properties": {
    "file_id": "string",
    "subtask_id": "string",
    "days_to_close": "integer",
    "triggered_by": "file_opened_and_commented",
    "ts": "ISO8601"
  }
}
```

```json
// GitHub PR description injection
{
  "event": "github_pr_injected",
  "properties": {
    "file_id": "string",
    "repo_id": "string",
    "pr_number": "integer",
    "linear_issue_id": "string",
    "ts": "ISO8601"
  }
}
```

```json
// Digest email sent
{
  "event": "digest_sent",
  "properties": {
    "recipient_user_id": "string",
    "file_count": "integer",
    "cadence": "daily | weekly | biweekly",
    "ts": "ISO8601"
  }
}
```

```json
// Digest CTA clicked
{
  "event": "digest_cta_clicked",
  "properties": {
    "recipient_user_id": "string",
    "file_id": "string",
    "days_since_share": "integer",
    "utm_source": "digest",
    "ts": "ISO8601"
  }
}
```

---

## 7. Key metrics

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
| `devmode_session_started` on first-ever file open by engineer | Rate among users with `trigger_reason: first_ever_open` in overlay | ~20% | >50% | 2 |
| Engineer comment rate on design files within 14 days | `comment_created` with `commenter_role: viewer or dev_mode` | ~15% | >25% | 1 + 3 |
| P50 time from `file_shared` to `file_opened_by_non_editor` | Time delta between events | >48h | <6h | 3 |
| Digest email open rate | Email analytics | N/A (new) | >25% | 4 |
| Linear "Design review" sub-task completion rate | `linear_subtask_autoclosed` / `linear_subtask_created` | N/A (new) | >60% | 3 |
| Overlay completion rate (all 3 steps) | `overlay_dismissed` with `method: completed_all_steps` | N/A (new) | >40% of overlays shown | 2 |

### Guardrail metrics

| Metric | Threshold | Why it matters |
|---|---|---|
| Time-to-first-frame for designers in the activation funnel | Must not increase >5% vs. pre-launch baseline | Handoff prompts and "ready for review" nudges must not interrupt design flow |
| Digest unsubscribe rate in first 30 days | Must not exceed 15% - trigger a content/cadence review if breached | Digest is additive value, not notification spam |
| `file_shared` rate per designer per week | Must not decline - new prompts should not create friction that reduces sharing | If designers share less, the funnel narrows regardless of downstream improvements |
| Designer NPS for the share flow (in-product survey, n >= 200) | Must not decline >5 points vs. pre-launch baseline | Designer sentiment protects the top of the funnel |
| `handoff_trigger_shown` -> `handoff_triggered` conversion rate | Must reach >35% within 30 days of launch | If < 35%, prompt copy or placement requires iteration |

---

## 8. Competitive analysis

| Competitor | Their handoff model | Where they win | Figma's gap vs. them | Figma's structural advantage |
|---|---|---|---|---|
| **Zeplin** | Dedicated handoff tool; engineers get a separate inspection app with annotations, styleguides, and comment threads; PM defines "section" which maps to a feature | Purpose-built engineer-first UI; sections model makes each handoff an explicit workflow unit with its own review state; engineers treat Zeplin as the spec home, not a design tool they're visiting | Zeplin has explicit "handoff" as a first-class concept; a Zeplin section can be approved, pending, or needs changes; Figma has no equivalent state machine | Figma owns the design file - Zeplin requires a sync step and a second tool. Fixing the handoff trigger closes most of the gap without requiring a tool switch. Teams can stay in one product. |
| **Notion + Figma embed** | PMs embed Figma prototypes in Notion specs; engineers read the spec doc with the Figma embed as a secondary artefact | Context-rich: the Notion doc has the why (problem, goals, metrics) and the what (Figma embed); engineers can comment on the Notion doc without ever opening Figma | Figma has the design but not the context; Notion has the context but not the inspectable design; the Figma embed is view-only with no Dev Mode | Pillar 3's Linear integration creates a Notion-equivalent "context layer" inside the engineering workflow (Linear), without requiring a PM to manually embed or maintain a separate document. |
| **Storybook / Code Connect** | Engineers review component behaviour in code; design-to-code link is established post-build | Post-build review catches production gaps; Storybook is trusted by engineering because it is the real component | Storybook is post-build (when it's too late to change the design); Figma Dev Mode + Code Connect is pre-build spec but the loop rarely closes unless engineers already know Dev Mode | Dev Mode + Code Connect is the bridge between design spec and production code. Pillar 2 activates engineers in Dev Mode before the Storybook habit forms - the sequence matters. |
| **Linear + screenshots (status quo)** | Teams paste screenshots of Figma frames into Linear comments as the handoff artefact | Zero friction for the designer; screenshot in Linear is immediately visible to engineers in their existing workflow | Screenshots degrade: they go stale when the design changes; have no inspection layer; don't link back to the source of truth | Pillar 3 replaces the screenshot paste with a live Dev Mode link inside the same Linear ticket. No extra tool, no new URL to remember - the link is in the ticket the engineer already has open. |
| **Figma's own DevMode (current free tier)** | Engineers get free inspection access but no workflow integration; they know Dev Mode only if the designer or a colleague told them about it | Free; no seat purchase needed; powerful inspection when the engineer finds it | Dev Mode adoption is organic and accidental; there is no systematic product motion that tells engineers "this exists and you should use it for this file" | Pillar 2 is the product motion. This PRD activates Dev Mode through the onboarding overlay rather than expecting engineers to discover it on their own. |

**Win condition vs. Zeplin:** Figma wins if the handoff trigger + "Ready for review" label makes the Figma workflow feel as intentional as Zeplin's sections model, without requiring a tool switch. If Figma can own the "design is ready" moment, Zeplin's reason for existing weakens.

**Loss condition vs. Zeplin:** Figma loses if the review-ready heuristic fires too frequently (false positives) and designers stop trusting it. If the prompt becomes noise, designers will continue to use informal Slack links - and Zeplin's explicit "publish to Zeplin" workflow will feel more controlled.

**Win condition vs. Linear + screenshots:** Pillar 3 wins if the GitHub PR injection and Linear sub-task creation reach >60% task-completion rate within 90 days. If engineers close the design review sub-task with a comment in Figma, the loop is closed - screenshots become demonstrably inferior.

**Loss condition vs. Linear + screenshots:** Pillar 3 loses if the Linear sub-task is perceived as PM micromanagement of engineering review. If engineers mark the sub-task "done" without actually opening the Figma file, the metric is gamed and the underlying problem persists.

---

## 9. Experiment backlog

| Experiment | Hypothesis | Primary metric | Guardrail | Priority | Minimum detectable effect | Power |
|---|---|---|---|---|---|---|
| Review-ready prompt placement: top bar vs. right panel | Top bar placement has higher designer visibility but may feel more intrusive than a right-panel nudge | `handoff_trigger_shown` -> `handoff_triggered` conversion rate | `time_to_first_frame` must not increase | P0 - required before GA | 5pp conversion difference | 80% at n=2,000 per arm |
| Overlay with "Switch to Dev Mode" CTA vs. text-only tooltip | Interactive CTA (executes mode switch) drives higher Dev Mode adoption than a text tooltip that describes Dev Mode | `devmode_session_started` within first 10 min of file open for overlay cohort vs. control | `overlay_dismissed` with `method: permanent_skip` rate must not exceed 30% | P0 - required before GA | 10pp Dev Mode activation difference | 80% at n=1,000 per arm |
| Linear sub-task creation: always-on vs. opt-in | Always-on sub-task creation drives higher `file_opened_by_non_editor_within_7d` rate vs. opt-in which reaches fewer engineers | `file_opened_by_non_editor_within_7d_rate` for Linear-linked files | Linear issue comment rate must not decline (sub-task must not create noise) | P1 | 8pp open-rate difference | 80% at n=1,500 per arm |
| Digest cadence: weekly vs. twice-weekly | Twice-weekly digest drives higher `file_opened_by_non_editor` rate by reducing time between share and re-engagement | `digest_cta_clicked` rate per digest sent | Unsubscribe rate must stay <15% | P1 | 3pp click-rate difference | 80% at n=3,000 per arm |
| Dev Mode deep link vs. editor link in digest CTA | Deep links to Dev Mode drive higher `devmode_session_started` rate than links to editor mode (which is the default) | `devmode_session_started` within 5 min of digest CTA click | `file_opened_by_non_editor` rate must not decline (deep link must not break file access) | P0 | 15pp Dev Mode session rate difference | 80% at n=800 per arm |
| "Ready for review" label visibility: project browser only vs. project browser + in-file top bar | Showing the label in the file top bar as well as the project browser increases engineer awareness and reduces time-to-first-open | P50 time from `handoff_triggered` to `file_opened_by_non_editor` | Designer edit flow duration must not increase | P2 | 4h P50 time difference | 80% at n=1,000 files |

---

## 10. Dependency map

| Pillar | Dependency | Owner | Blocking? | Risk |
|---|---|---|---|---|
| Pillar 1 (review-ready trigger) | Figma file save event pipeline (server-side) | Platform infra | Yes | Event volume at scale; trigger evaluation must not introduce save latency |
| Pillar 1 | Share modal redesign (Dev Mode default, recipient required field) | Design | Yes | PM/Design alignment on share modal changes |
| Pillar 2 (overlay) | Dev Mode entry point API (programmatic mode switch) | Dev Mode team | Yes | Dev Mode team owns the mode toggle; cross-team coordination required |
| Pillar 2 | User attribute store (overlay suppression flags) | Identity/auth team | Yes | Requires `overlay_permanently_dismissed` and `overlay_last_shown_at` fields on user record |
| Pillar 3 (Linear integration) | Existing Figma-Linear integration (v2 of the integration) | Partnerships/integrations | Yes | Sub-task creation requires Linear webhook capability; must validate with Linear's API |
| Pillar 3 (GitHub integration) | GitHub App for Figma (PR body write permissions) | Partnerships/integrations | Yes | Figma's GitHub app must have `pull_requests: write` scope; may require users to re-authorize the app |
| Pillar 4 (digest) | Email delivery infrastructure (transactional email) | Growth/comms infra | Yes | Requires suppression list management and bounce handling |
| Pillar 4 | Thumbnail generation service (first frame as image) | Media/rendering | No | Fallback to file icon if thumbnail fails; not blocking |
| All pillars | Event pipeline and analytics warehouse | Data infra | No | Analytics are required for experiment measurement but not for feature function |

---

## 11. Rollout plan

### Phase 0 - Internal dogfood (weeks 1-4)

**Scope:** Figma's own product and design teams only.
**Goal:** Validate that heuristic trigger does not interrupt design flow; validate that overlay renders correctly across file types.

**Launch gate - exit Phase 0:**
- Zero designer-reported interruptions to design flow from the trigger prompt (tracked via internal feedback channel).
- Overlay shown rate >80% of qualifying first-opens (measured against internal usage logs).
- `handoff_trigger_shown` -> `handoff_triggered` conversion rate >25% in internal cohort (lower bar than GA target; internal team is aware of the feature).
- P1-9 server-side push latency confirmed <200ms on internal load.

**Kill switches active:**
- Per-file feature flag: any Figma engineer can toggle off the trigger for a specific file in <1 minute.
- Pillar 2 overlay can be disabled globally via ops config without a code deploy.

**Do not proceed if:** Trigger prompt fires on files that the internal team classifies as WIP in a post-hoc review of 20+ trigger events. Re-evaluate heuristic thresholds before Phase 1.

---

### Phase 1 - Closed beta, 5% of Professional teams (weeks 5-10)

**Scope:** 5% of Professional-tier teams randomised by team ID. Holdout: 5% of Professional teams receiving no treatment (pure control). Remaining 90%: no exposure.

**Goal:** Measure `file_opened_by_non_editor_within_7d_rate` lift vs. holdout; validate digest unsubscribe rates below 15%.

**Launch gate - exit Phase 1 to Phase 2:**

| Gate | Target | Measurement window |
|---|---|---|
| `file_opened_by_non_editor_within_7d_rate` lift vs. holdout | >5pp absolute lift | Week 6-10 (after 4 weeks of exposure) |
| Digest unsubscribe rate | <15% of digest recipients | First 30 days of digest sending |
| Designer NPS for share flow (in-product survey, n >= 100) | Not decline >5 points vs. pre-launch baseline | Weeks 8-10 |
| `file_shared` rate per designer per week in treatment vs. holdout | Must not decline >2pp | Week 6-10 |
| P1-9 trigger push latency P99 | <500ms | Continuous monitoring weeks 5-10 |

**Per-pillar kill switches:**

| Pillar | Kill switch condition | Action |
|---|---|---|
| Pillar 1 (trigger) | `handoff_trigger_shown` rate exceeds 3x per file per week in any team cohort (false-positive flood) | Disable trigger evaluation for that team; ops alert within 15 min |
| Pillar 2 (overlay) | `overlay_dismissed` with `method: permanent_skip` rate exceeds 40% of overlays shown | Pause overlay; review copy and placement before resuming |
| Pillar 3 (Linear) | Linear API error rate for sub-task creation exceeds 5% over any 1-hour window | Disable Linear sub-task creation; switch to silent logging mode; alert integrations team |
| Pillar 3 (GitHub) | GitHub PR injection failure rate exceeds 3% (failed API calls / total injection attempts) | Disable GitHub injection; log `github_injection_disabled` with reason |
| Pillar 4 (digest) | Unsubscribe rate exceeds 15% within first 14 days of sending | Auto-pause digest sending; alert growth ops; do not resume without content review |

**Rollback procedure:** All four pillars can be disabled independently via ops config flags. No code deploy required. Estimated time to full rollback for any single pillar: <5 minutes. Full rollback of all pillars: <10 minutes.

---

### Phase 2 - Expansion, 25% of Professional and Organisation tiers (weeks 11-16)

**Scope:** 25% of Professional and Organisation tier teams. A/B experiments for prompt placement and overlay CTA variant run within this cohort.

**Goal:** Validate at scale; run A/B experiments on prompt placement and overlay CTA variant.

**Launch gate - exit Phase 2 to Phase 3:**

| Gate | Target | Measurement window |
|---|---|---|
| P50 time from `file_shared` to `file_opened_by_non_editor` | <12h (interim target on path to 6h) | Weeks 13-16 |
| Designer NPS for share flow (n >= 200) | Not decline >3 points vs. pre-launch baseline | Weeks 13-16 |
| `file_opened_by_non_editor_within_7d_rate` trend | Consistent week-over-week increase in treatment cohort vs. holdout | Weeks 11-16 (trend, not point-in-time) |
| Pillar 3 Linear sub-task completion rate | >50% (sub-tasks auto-closed within 7d of creation) | Weeks 13-16 |
| P0 experiments concluded | Both Phase-1-required experiments (prompt placement + overlay CTA) have conclusive results | Before Phase 3 launch |

**Additional kill switch - Phase 2 only:**
- If `file_shared` rate declines >3% week-over-week for 2 consecutive weeks in the treatment cohort vs. holdout, pause Pillar 1 trigger expansion and investigate. This is the canary metric for designer friction.

---

### Phase 3 - GA, all Professional and Organisation tiers (weeks 17+)

**Scope:** All Professional and Organisation tiers. Free tier gets Pillar 2 only (overlay) - no trigger, no integrations, no digest.

**Why Free tier gets Pillar 2 only:** Free tier files are disproportionately solo projects or prototypes - the cross-functional handoff problem is less acute. Pillar 2 (overlay) benefits any engineer opening any file and costs nothing to extend. Pillars 1, 3, and 4 require team context and integration setup, which is a poor fit for Free tier solo accounts.

**GA monitoring cadence:**
- Weekly: `file_opened_by_non_editor_within_7d_rate` vs. pre-launch baseline; digest unsubscribe rate; `file_shared` rate per designer.
- Monthly: Designer NPS for share flow; P50 time from `file_shared` to `file_opened_by_non_editor`; Linear sub-task completion rate.
- Quarterly: Full A/B experiment review; North Star metric progress vs. 50% target; SLO compliance check.

**Incident response triggers (post-GA):**
- If `file_shared` rate drops >5% relative in any 7-day rolling window vs. 4-week pre-launch average: P1 incident, Pillar 1 disable within 30 minutes.
- If digest unsubscribe rate climbs above 20% in any 14-day window: auto-pause digest, P2 incident.
- If `handoff_trigger_shown` fires >5x per file per week for any org cohort: P1 incident, trigger rate-limiting deployed within 2 hours.
- If Linear API sub-task creation error rate exceeds 10% for 30 consecutive minutes: auto-disable Pillar 3 Linear integration, P1 alert.

---

## 12. Key risks

| Risk | Likelihood | Severity | Mitigation |
|---|---|---|---|
| Review-ready heuristic fires when file is still a WIP | High | Medium | Require explicit designer confirmation; 72h suppression after dismissal; "Mark as WIP" escape hatch |
| Engineering onboarding overlay is perceived as patronising | Medium | Medium | Prominent "I know Figma, skip this" in step 1; permanent suppression flag; show only on first open or first file with "Ready for review" label |
| Linear/GitHub integrations break existing PR templates or issue workflows | Medium | High | Additive-only changes to PR bodies; no overwriting of existing content; rollout to opt-in teams first; one-click disable per org |
| Digest email contributes to notification fatigue and increases unsubscribe rate from all Figma emails | Medium | High | Weekly default (not daily); aggressive unsubscribe UX; strict volume cap at 5 files per digest; pause if unsubscribe rate > 15% |
| Dev Mode paid tier is introduced before engineers build an engagement habit | High | High | Engineering activation (`devmode_session_started` rate > 40% for engineering teams) must be a prerequisite gate before any Dev Mode paid tier launch |
| Pillar 3 scope creep into project management territory (competing with Linear) | Low | High | Strict non-goal enforcement: Figma creates one sub-task and one PR annotation, nothing more. No sprint boards, no issue creation, no status sync beyond the design review event. |
| GitHub PR body injection fails silently for repos using strict PR template enforcement | Medium | Medium | Pre-check for PR template validation rules before injecting; if template enforcement detected, skip injection and log `github_injection_skipped` with `reason: template_enforcement` |

---

## 13. Open questions - resolved from v1

**Q1: What is the actual `file_opened_by_non_editor_within_7d_rate` baseline today?**

Decision: The 35% estimate is the working baseline for experiment power calculations. Instrument `file_opened_by_non_editor` with a `days_since_file_created` field. In Phase 0 dogfood, measure the actual rate against the holdout. If actual baseline differs materially from 35% (e.g., >45% or <25%), re-run experiment power calculations before Phase 1 launch. A higher baseline means a harder uplift target; a lower baseline means more room for improvement and a potentially easier success gate.

**Q2: Does the Figma-Linear integration already support sub-task creation?**

Decision: The current Figma-Linear integration (as of 2025) supports file-to-issue linking and syncs design file thumbnails into Linear issues. Sub-task creation is net-new functionality requiring the Figma integration to make a Linear API call via the Linear GraphQL `createIssue` mutation with the parent issue ID. This is feasible with Linear's API but requires a partnership engineering allocation. Estimated effort: 3-4 weeks of integration work. Confirmed as a blocking dependency for Pillar 3.

**Q3: What is the P50 time from `file_shared` to `file_opened_by_non_editor` in the current dataset?**

Decision: Treat >48h as the working baseline for the P50 time metric. Instrument this delta from Phase 0 dogfood. If actual P50 is already <24h for a subset of teams (e.g., teams already using Linear), Pillar 3 should focus on the segment where P50 is >48h - likely teams using Slack-only sharing - where the impact is highest.

**Q4: Is there a meaningful difference in activation rate between named-recipient shares vs. link-only shares?**

Decision: Hypothesis is yes - named-recipient shares likely have >50% 7-day open rate while link-only shares are below 20%. Instrument `file_shared` with `share_type: named_recipient | link_only`. If the hypothesis is confirmed, Pillar 1's explicit recipient requirement in the handoff share modal is the highest-leverage intervention (shifting shares from link-only to named-recipient). If the hypothesis is wrong (open rates are similar), the focus shifts entirely to meeting engineers in their workflow (Pillar 3) regardless of share type.

**Q5: What % of engineering teams with an active Figma file have ever initiated a Dev Mode session?**

Decision: The ~30% estimate is the working baseline for Pillar 2 targeting. Phase 0 dogfood will instrument `devmode_session_started` cohorted by `file_opened_by_non_editor` trigger reason. If >50% of engineers who open a "Ready for review" file already initiate Dev Mode without the overlay, Pillar 2's target cohort narrows to first-time Figma users only - and the overlay does not need to show for experienced engineers opening labelled files.

**Q6: Can the review-ready detection heuristic be validated before building the full trigger system?**

Decision: Yes. Before building the server-side trigger, run a manual audit on a sample of 50 design files where `devmode_session_started` fired within 7 days of file creation. Apply the heuristic retroactively: how many of those files would have qualified as "review-ready" by the heuristic? If >70% of files that led to an engineering session qualified under the heuristic, the signal is validated. If <50% qualified, the heuristic needs additional signals (e.g., design annotation layer present, component coverage >80%) before building the trigger.

**Q7: Is there a Jira integration roadmap for Pillar 3?**

Decision: Jira is in scope for v2 of Pillar 3, not v1. Jira's API is more complex (project/issue hierarchy, multiple auth models), and the Figma-Jira integration surface is thinner than Figma-Linear. v1 targets Linear-first because Linear is the primary project management tool for product-led companies (Figma's core customer segment). Jira coverage is tracked as a follow-on item post-GA.

---

## 14. Open questions - resolved from v2

**Q8: Should the 35% baseline be validated before setting Phase 1 success gates?**

Decision: Yes - the 35% baseline is directional and based on public teardown inference, not internal Figma data. Phase 0 must produce an actual measured baseline for Professional-tier teams before Phase 1 success gates are locked. The Phase 1 gate of ">5pp lift vs. holdout" is robust to a range of actual baselines (works whether actual is 25% or 45%), but if the actual baseline is already above 48%, the 50% target requires re-scoping to a relative lift goal (e.g., +10% relative) rather than an absolute target.

**Q9: What happens if the share-type hypothesis (Q4) is disproved - named vs. link-only open rates are similar?**

Decision: If Phase 1 data shows that named-recipient and link-only shares have equivalent 7-day open rates (within 3pp), Pillar 1's recipient-required field in the handoff modal becomes lower-priority UX friction with no proven benefit. In that case, remove the required field in Phase 2 and shift the modal focus to Dev Mode access default (which remains valuable regardless of share type). Retain the `share_type` instrumentation for longitudinal analysis.

**Q10: Does Pillar 2 (overlay) show a meaningful retention difference between engineers who complete all 3 steps vs. those who dismiss at Step 1?**

Decision: Instrument this as a cohort split from Phase 1 onward. Track `devmode_session_started` rate within 7 days of file open for three cohorts: (a) overlay completed all 3 steps, (b) overlay dismissed at step 1-2, (c) control (no overlay). If cohort (a) shows >15pp higher `devmode_session_started` rate vs. cohort (b), the overlay is delivering meaningful behaviour change - not just exposure. If cohorts (a) and (b) are within 5pp, the overlay's step sequence adds minimal value over mere exposure to any overlay, and a simplified 1-step version should be tested in Phase 2.

**Q11: What is the right auto-close logic for the Linear sub-task if the engineer opens the file but leaves no comment?**

Decision: Do not auto-close on file open alone. The sub-task closes only when both `file_opened_by_non_editor` AND `comment_created` fire on the linked file within 7 days. This maintains the signal integrity of the "review complete" event - an engineer who opens and immediately closes the file without engaging has not completed the review. The 7-day window balances urgency (engineers should review promptly) against sprint realities (7 days covers a typical sprint cycle). If the auto-close rate for "opened + commented within 7d" falls below 40%, shorten the window to 5 days and measure again.

**Q12: Should the digest "Mark as reviewed" token-in-URL approach be scoped out of v1 due to security concerns?**

Decision: The unguessable token approach is acceptable for v1. The token grants a single action (suppress a specific file from future digests for a specific user) and has no write access to the Figma file itself. Token length of 32 bytes (256 bits) from a cryptographically secure random source is sufficient. Token expiry: 30 days from digest send date. Log token redemption events (`digest_marked_reviewed`) for abuse monitoring. If token abuse is detected (bulk redemption patterns), rotate tokens and invalidate outstanding tokens without requiring user re-authentication.

---

## 15. Full instrumentation spec

This section specifies which events are instrumented in each rollout phase, the responsible data owner, and the validation approach.

### Phase 0 instrumentation (internal dogfood - weeks 1-4)

**Goal:** Establish actual baselines; validate event firing behaviour.

| Event | Phase 0 requirement | Validation approach |
|---|---|---|
| `handoff_trigger_shown` | Fire on every qualifying save; include all defined properties | Manual spot-check: trigger 5 test files through heuristic criteria; confirm event fires in analytics warehouse within 60s |
| `handoff_triggered` | Fire only after designer submits via the handoff share modal | Confirm event does NOT fire if designer opens share modal and cancels |
| `handoff_trigger_suppressed` | Fire with correct `reason` field on each suppression path | Test all 4 suppression reasons in QA environment; confirm reason field accuracy |
| `file_opened_by_non_editor` | Existing event - add `days_since_file_created` field | Backfill validation: confirm new field is present in 100% of events fired in Phase 0 |
| `overlay_shown` | Fire on every qualifying first-open; include `trigger_reason` | Confirm overlay shown event fires before canvas interaction events in the session |
| `overlay_step_completed` | Fire per step with correct `method` field | Test all three advance methods (action_taken, manual_advance, auto_advance) in QA |
| `overlay_dismissed` | Fire with correct `method` field including `permanent_skip` | Confirm `overlay_permanently_dismissed` flag is set on user record after permanent skip |

**Phase 0 baseline measurements (to be captured before Phase 1 launch):**
- Actual `file_opened_by_non_editor_within_7d_rate` for Professional-tier internal files (current state, no treatment).
- Actual P50 time from `file_shared` to `file_opened_by_non_editor` for internal teams.
- Actual `devmode_session_started` rate within 24h of `file_opened_by_non_editor` for internal engineering files.

### Phase 1 instrumentation (5% closed beta - weeks 5-10)

**Goal:** Measure treatment vs. holdout; validate Pillar 3 event accuracy.

| Event | Phase 1 additions | Responsible owner |
|---|---|---|
| `linear_subtask_created` | Add to production pipeline; fire on every successful sub-task creation | Integrations team |
| `linear_subtask_skipped` | Fire with `reason` field for every suppression | Integrations team |
| `linear_subtask_autoclosed` | Fire when both conditions met within 7d window | Integrations team |
| `github_pr_injected` | Fire on every successful PR body injection | Integrations team |
| `github_injection_skipped` | Fire with `reason` field for every skip | Integrations team |
| `digest_sent` | Fire on every digest batch send; include `file_count` | Growth/comms infra |
| `digest_cta_clicked` | Fire server-side on token redemption (not client-side pixel); include `days_since_share` | Growth/comms infra |
| `digest_unsubscribed` | Fire immediately on unsubscribe link click | Growth/comms infra |

**Holdout instrumentation:** The 5% holdout cohort must have identical event instrumentation to the treatment cohort except for the absence of the feature. This ensures the baseline `file_opened_by_non_editor_within_7d_rate` measurement is comparable.

**Weekly data review in Phase 1:**
- Every Monday: treatment vs. holdout `file_opened_by_non_editor_within_7d_rate` (7-day rolling window).
- Every Monday: digest unsubscribe rate (cumulative since first send).
- Every Monday: `file_shared` rate in treatment vs. holdout (guardrail).

### Phase 2 instrumentation (25% expansion - weeks 11-16)

**Goal:** A/B experiment instrumentation; full guardrail dashboard live.

| Event / addition | Phase 2 requirement |
|---|---|
| Experiment arm assignment | Add `experiment_arm` field to `handoff_trigger_shown` and `overlay_shown` events; values: `top_bar | right_panel` and `cta_interactive | text_only` |
| `digest_marked_reviewed` | Fire on token redemption; include `days_since_share` and `token_age_days` |
| Designer NPS survey | In-product survey fires after `handoff_triggered` (3-day delay); `nps_response` event with `score: integer`, `feature: share_flow` |
| Share modal exit without submit | Fire `handoff_modal_abandoned` event if share modal is opened from the trigger but closed without submitting |

**Guardrail dashboard (live by Phase 2 week 1):**
- Real-time alert if `file_shared` rate drops >3% week-over-week in treatment cohort.
- Real-time alert if any single Pillar 3 API error rate exceeds 5% over a 1-hour rolling window.
- Daily digest unsubscribe rate chart with 15% threshold line visible.

### Phase 3 (GA) instrumentation

**Goal:** Production-grade monitoring; automated incident triggers.

| Monitor | Trigger threshold | Response |
|---|---|---|
| `file_shared` rate 7-day rolling | Drops >5% vs. 4-week pre-launch average | P1 incident; Pillar 1 disable within 30 min |
| Digest unsubscribe rate 14-day rolling | Exceeds 20% | Auto-pause digest; P2 incident |
| `handoff_trigger_shown` rate per file | Exceeds 5x per week for any org cohort | P1 incident; trigger rate-limiting deployed within 2h |
| Linear API sub-task creation error rate 30-min rolling | Exceeds 10% | Auto-disable Pillar 3 Linear; P1 alert |

---

## 16. SLO revision history

| Metric | v1 target | v2 target | v3 target | Rationale for change |
|---|---|---|---|---|
| `file_opened_by_non_editor_within_7d_rate` (North Star) | >50% within 3 quarters | >50% within 2 quarters | >50% within 2 quarters; measured as 4-week rolling average post-GA (not point-in-time) | v3 adds measurement methodology to prevent single-week spikes masking a lower trend |
| Handoff trigger push latency | <500ms (P99) | <500ms (P99) | <500ms (P99) in Phase 1-2; <300ms (P99) at GA scale | Tightened for GA to account for higher event volume; baseline <200ms confirmed in Phase 0 required |
| Linear sub-task creation latency | Within 60 seconds | Within 60 seconds | Within 60 seconds (P95); alert if >120s for any sub-task | Added P95 definition and alert threshold; a 60s average is meaningless if P95 is 5 minutes |
| Digest unsubscribe rate | <15% in first 30 days | <15% in first 30 days | <15% in first 30 days; <10% at steady state (months 3+) | Added steady-state target; 15% is an acceptable launch rate but should converge down as list becomes self-selected engaged users |
| Overlay shown to Dev Mode session rate | N/A | >40% of overlays shown complete all 3 steps | >40% completion rate; >50% `devmode_session_started` within 24h of overlay shown | v3 adds downstream behaviour metric (Dev Mode session) not just overlay completion |
| P50 time from `file_shared` to `file_opened_by_non_editor` | >48h -> <6h | >48h -> <6h | Interim target <12h at Phase 2 exit; <6h at 2 quarters post-GA | Added interim target to create a checkable waypoint at Phase 2 rather than waiting 2 quarters |
| Designer NPS for share flow | Must not decline >5 points | Must not decline >5 points | Must not decline >5 points in Phase 1; must not decline >3 points at GA | Tightened at GA; early pilots have higher designer self-selection and tolerance; GA population is more representative |

---

*This is the final PRD version. All open questions from v1 and v2 are resolved. The next document in this pipeline sequence is the system architecture for Engineering Review Activation.*
