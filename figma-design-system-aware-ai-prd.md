# PRD: Figma Design-System-Aware AI Generation

**Product:** Figma - Make Designs (library-grounded generation)
**Author:** Mayank Malviya
**Version:** v3 - Final PRD
**Changes from v2:** Added phased rollout plan with quantitative go/no-go gates per phase, kill switches and rollback procedures, launch readiness checklist, two additional experiments in the backlog, all five open questions from section 11 fully resolved, SLO revision history, and a version history table aligned with v3 format.
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/figma-teardown.md

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, goals/non-goals, target personas, solution pillars, core requirements, event schemas, success metrics |
| v2 | Competitive analysis, acceptance criteria with test cases, data model, API contract, experimentation framework (3 experiments), cohort funnel, expanded risk register, open questions partially resolved (Q1-Q3) |
| v3 | Phased rollout plan with go/no-go gates and kill switches, launch readiness checklist, all open questions resolved (Q4-Q5 added), two additional experiments, SLO revision history, final dependency ownership, production instrumentation checklist |

---

## Context

Figma's `Make Designs` AI feature generates UI frames from a text prompt. As of mid-2025, the generated output uses hardcoded colour values and arbitrary font sizes - it ignores the org's published component library entirely. In orgs with a mature design system, this is a net-negative feature: AI-generated frames contain design inconsistencies that cost more time to fix than designing from scratch would have taken.

The teardown identified design-system-aware generation as the P0 AI product priority for Figma. This PRD specifies the feature: **Make Designs that reads your org's published component library and generates frames using instances of your actual components, your actual variables, and your actual text styles.**

---

## 1) Problem statement and why now

### The current failure loop

```
PM or designer writes a prompt ->
Make Designs generates plausible-looking frames ->
frames use hardcoded hex values and generic components ->
designer must audit every node for system compliance ->
audit cost > design-from-scratch cost ->
team bans AI generation for handoff-ready specs
```

**Observable failure modes:**

- A settings page generated from a prompt has 6 shades of grey. The design system has 2. The engineer builds from the spec. QA flags 14 discrepancies.
- A generated card component uses 14px font. The design system's `body/sm` text style is 13px/1.5 line-height. Both are close enough to pass a visual review but break the token linkage engineers rely on.
- A junior designer ships an AI-generated flow without auditing it. The engineer notices that none of the generated buttons match the `Button/primary` component - they are raw rectangles with similar styling. No component link, no variant states, no accessibility properties.

**Why this matters more than any other AI gap:** Figma's enterprise value proposition rests on design systems. The shared library is the product's deepest lock-in mechanism. An AI that generates frames outside the design system actively degrades the system's integrity. Every AI-generated frame that ships to engineering without component compliance adds entropy to the org's design system and erodes trust in both Figma AI and the design system itself.

### Why now

- The AI-to-code threat (v0, Cursor, Lovable) is accelerating. If Figma's AI generation is unreliable for enterprise teams, those teams have a stronger reason to skip the design file entirely and generate code directly. The 18-24 month window to establish AI-in-design leadership is closing.
- Figma already owns the design system data. Every published org library lives in Figma's infrastructure. No competitor has access to this corpus. Using it to ground generation is a data moat that v0 and Cursor cannot replicate.
- The AI add-on revenue model requires demonstrable value per session. Teams that trial AI generation once, find it system-incompatible, and stop using it are not renewing the add-on. Fixing generation quality is a retention and revenue problem, not just a product quality problem.

---

## 2) Goals / non-goals

### Goals

1. Increase the rate of `ai_make_designs_generated` events where `uses_org_library_components = true` from an estimated ~5-10% baseline to **>50% for orgs with at least one published org-scoped library** within 90 days of launch.
2. Reduce the average manual edit distance (% of generated nodes changed post-generation) from an estimated ~70% baseline to **<30%** for orgs using library-grounded generation.
3. Increase Figma AI Weekly Active Users (designers triggering at least one AI generation session per week) by **30% relative** within 90 days of launch, measured for orgs with org-scoped libraries enabled.
4. Achieve a `Make Designs` session outcome of `accepted` or `edited` (not `deleted`) at a rate of **>55%** for library-grounded sessions, vs. estimated ~30% for the current generic generation.

### Non-goals

- Replacing the designer's creative judgment. Library-grounded generation selects from existing components; it does not invent new component patterns.
- Generating animations, motion specs, or interactive prototype connections. Scope is static frame generation only in v1.
- Supporting teams with no published library. Generic generation remains the fallback for Free/Professional orgs without a published org-scoped library.
- Replacing the design systems team's component creation workflow. This feature consumes the library; it does not create or modify it.
- Real-time component creation during generation. If the prompt requests a component type that does not exist in the library, the system uses the closest match or flags the gap - it does not generate a new component.

---

## 3) Target personas

### Primary: Mid-level product designer at an org with a published design system

**Name:** Priya (senior IC, 4 years of Figma experience)
**Context:** Works at a 200-person SaaS company. Her org has a published design system with 140 components, 32 color variables, and 12 text styles. She uses `Make Designs` occasionally for first-draft exploration but always discards the output and starts from components manually because the AI-generated frames don't use her org's components.
**JTBD:** Get a first-draft layout for a new screen type in <5 minutes that she can refine, not rebuild.
**Aha moment for this feature:** She types a prompt, hits Generate, and the output uses her org's actual `Card/content` component, her `semantic/primary` color variable, and her `body/md` text style. She adjusts copy and spacing rather than replacing every element.
**Upgrade/retention signal:** She uses library-grounded generation 3+ times per week as a first-draft tool. Her `ai_make_designs_generated` sessions trend toward `edited` (not `deleted`).

---

### Secondary: Design systems lead at an enterprise org

**Name:** Dan (staff designer, leads a 3-person design systems team at a 600-person company)
**Context:** His team publishes and maintains a 300-component library used by 45 product designers across 6 teams. His biggest complaint about AI generation is that it bypasses the system - junior designers treat AI output as authoritative and ship system-non-compliant frames to engineering.
**JTBD:** Ensure that AI-generated output is always system-compliant, reducing design system violations in shipped product.
**Aha moment for this feature:** He reviews an AI-generated settings flow and sees that every element is a component instance from his library. He sets a team policy: AI generation is permitted for first drafts because it generates system-compliant output.
**Upstream signal:** His team sees a reduction in "please fix this before handoff" design review comments related to system violations.

---

### Secondary: PM using Make Designs for quick wireframe alignment

**Name:** Anya (senior PM, non-designer)
**Context:** Uses Figma as a viewer and occasional FigJam contributor. She occasionally uses Make Designs to sketch a rough layout idea before briefing a designer. She does not understand component libraries; she just wants the output to look like the rest of the product.
**JTBD:** Create a rough wireframe that looks like "our product" without needing a designer for the initial sketch.
**Aha moment for this feature:** Her generated wireframe uses the real product's navigation, card, and button components. It looks like a real product screen, not a generic UI mockup.
**Guardrail:** She must not be able to accidentally publish an AI-generated frame to engineering without designer review. This is an out-of-scope social problem for v1 but a risk to flag.

---

## 4) Solution pillars

### Pillar 1: Library context injection at generation time

When a `Make Designs` session is triggered in a file belonging to an org with a published org-scoped library, the generation pipeline receives a structured representation of the library: component names, their variant sets, their visual semantics (inferred from the layer tree and component names), color variables (name + value), and text styles (name + typographic properties).

This structured representation becomes part of the generation context. The model maps each element in the generated layout to the nearest library component before rendering the frame.

**Technical constraint:** Library context is bounded by the generation model's context window. For orgs with very large libraries (>500 components), a relevance filter must be applied: the model receives the 50-100 most semantically relevant components for the given prompt, not the full library. The relevance ranking uses component name embeddings against the prompt.

---

### Pillar 2: Component-first frame generation

Generated frames must use component instances, not raw shapes with similar visual styling. This is the critical quality gate:

- Every button in a generated frame must be an instance of a `Button` component (or the closest available variant).
- Every input field must be an instance of an `Input` component.
- Every card container must be an instance of a `Card` component.
- Color fills must use variable references (`semantic/primary`, `neutral/100`), not hardcoded hex values.
- Text elements must use published text styles (`heading/xl`, `body/md`), not raw font size + weight combinations.

Frames that cannot achieve >80% component coverage from the library (because the prompt requests UI patterns with no library equivalent) must surface a warning to the user before the frame is accepted: "This design uses [N] elements not in your library. Review before handing off."

---

### Pillar 3: Post-generation compliance report

After generation, before the user accepts the frame, display a lightweight compliance panel showing:

- % of nodes that are library component instances (target: >80% for P0 success)
- % of colour values using library variables (target: >90%)
- % of text elements using library text styles (target: >90%)
- List of non-compliant elements with suggested library matches

This transparency lets the designer make an informed accept/edit/discard decision. It also creates a feedback loop: if the designer accepts with low compliance, that is a signal that the library does not cover the prompt's use case - which is actionable for the design systems team.

---

### Pillar 4: Library gap surfacing

When the generation pipeline cannot find a library component for a required element type (e.g., the prompt asks for a data table but the library has no `Table` component), the system:

1. Uses a styled raw frame as a fallback, clearly flagged as "not a library component."
2. Logs the gap as `library_gap_detected` with the element type and prompt context.
3. Surfaces an optional notification to the design systems team: "3 recent AI generation sessions needed a [Table] component that isn't in your library. Consider adding one."

This turns AI generation into a design system feedback mechanism - helping the design systems team prioritise which components to build next based on actual usage signals.

---

## 5) Core requirements

### Functional requirements

| ID | Requirement | Priority | Acceptance criteria |
|---|---|---|---|
| FR-01 | When `Make Designs` is triggered in an org with a published library, the generation pipeline must receive a structured library context | P0 | Library context is injected for 100% of generation sessions in qualifying orgs; latency overhead <2s vs. baseline |
| FR-02 | Generated frames must use component instances for all UI elements with a library match | P0 | >80% of generated nodes are library component instances for prompts covering standard UI patterns (forms, cards, navigation) |
| FR-03 | Generated frames must use color variable references, not hardcoded hex values, for all fill and stroke colours | P0 | >90% of colour values in generated frames are variable references |
| FR-04 | Generated frames must use library text styles for all text elements | P0 | >90% of text nodes in generated frames use library text styles |
| FR-05 | A post-generation compliance panel must appear before the user accepts the frame | P1 | Compliance panel renders within 500ms of frame generation completing; shows % component coverage, % variable coverage, % text style coverage, and list of non-compliant elements |
| FR-06 | Non-compliant elements must be listed with suggested library matches | P1 | Each non-compliant element shows the top 3 nearest library components by semantic similarity |
| FR-07 | Library gap events must be logged when the pipeline cannot find a match for a required element type | P1 | `library_gap_detected` event fires for each unmatched element type per session |
| FR-08 | Generation must fall back to generic (non-library) output for orgs with no published library | P0 | Fallback is seamless; user sees no error; fallback frames are visually indistinguishable from current generic output |
| FR-09 | Generation must work with libraries containing 10 to 500+ components without timeout | P0 | P95 generation latency for orgs with 100-200 component libraries is <8s; for 400-500 component libraries is <12s |

### Non-functional requirements

| ID | Requirement | Threshold |
|---|---|---|
| NFR-01 | Generation end-to-end latency P95 for orgs with published library | <10s |
| NFR-02 | Compliance panel render time | <500ms after frame is placed on canvas |
| NFR-03 | Library context injection failure rate | <0.5% of sessions |
| NFR-04 | Component instance accuracy (correct variant selected for context) | >75% of matched components are the contextually correct variant, not just the correct component type |
| NFR-05 | Privacy: library structure data must not be used to train generation models without explicit org admin consent | Library context used only for in-session generation; opt-in model training flag required |

---

## 6) User journey: happy path

**Actor:** Priya (senior product designer, org has published library with 140 components)

1. Priya opens a new file in her org's project space. She is in a meeting and needs a rough first draft of a new "User Settings > Notifications" screen to align with her PM before the meeting ends.
2. She clicks an empty frame area and types: "Settings page for notification preferences - toggles for email, push, and in-app notifications, grouped by category, with a save button."
3. `Make Designs` generates a frame in ~6 seconds. A compliance panel appears beneath the frame showing: **Component coverage: 92% | Color variables: 100% | Text styles: 94%**.
4. She sees the generated frame uses her org's `Toggle/active` and `Toggle/inactive` variant components, her `semantic/primary` variable for the active state, and her `body/md` text style for option labels.
5. The panel flags one non-compliant element: the section divider is a raw line, not a `Divider` component. The suggested match is `Divider/horizontal/full` from her library. She clicks "Replace" and the system swaps it.
6. She accepts the frame. It lands on her canvas as a fully editable Figma frame with all library links intact.
7. She adjusts the copy, removes one category section, and shares the prototype link in Slack in under 4 minutes. PM and engineer open the link and immediately recognise the components from the live product.

**Key events fired:** `ai_make_designs_generated` (uses_org_library_components: true, outcome: edited), `compliance_panel_viewed`, `non_compliant_element_replaced`

---

## 7) User journey: edge case 1 - prompt requests a UI pattern not in the library

**Actor:** Priya, same org. She prompts: "Data table with sortable columns and row-level checkboxes, with a bulk action toolbar."

1. Generation completes. Compliance panel shows: **Component coverage: 34% | Color variables: 100% | Text styles: 88%**.
2. The panel surfaces a warning: "3 element types in this design are not in your library: Table, Checkbox row, Bulk action bar. These are shown as styled frames, not components."
3. The frame still uses library components for buttons, typography, and colour. The non-library elements are visually consistent but raw shapes.
4. Priya accepts the frame knowing it needs design systems review. She tags the frame with a `needs-component-review` annotation before sharing.
5. The `library_gap_detected` event fires 3 times (once per unmatched element type), and the design systems team's Figma notification shows: "3 AI sessions in the past 7 days needed a Table component."

**PM implication:** Library gap events are the design system team's feature request queue. In orgs where this surfaces consistently, it drives library expansion - increasing the quality ceiling for future AI generations and reinforcing the library as the system of record.

---

## 8) User journey: edge case 2 - org with a sparse library (Free/Professional tier)

**Actor:** Solo designer at a 12-person startup on the Professional plan. No published library. She uses `Make Designs` to get a first wireframe draft.

1. She triggers `Make Designs`. The system detects no published library in her team or file.
2. Generation proceeds with the current generic model (no library context injection). Output is visually plausible but uses hardcoded styles.
3. No compliance panel appears (no library to compare against). Generation UX is identical to the current product.
4. She accepts the frame and uses it as a rough wireframe for stakeholder alignment - appropriate use of generic generation at this team's maturity level.

**PM implication:** Library-grounded generation does not degrade the experience for teams without a library. It adds a new capability layer for orgs that have invested in their design system - incentivising library creation for teams that want AI generation quality.

---

## 9) Event schemas (v1 additions)

```json
{
  "event": "ai_make_designs_generated",
  "properties": {
    "file_id": "string",
    "org_id": "string",
    "prompt_length": "integer",
    "frames_generated": "integer",
    "library_context_injected": "boolean",
    "library_component_count": "integer",
    "uses_org_library_components": "boolean",
    "component_coverage_pct": "float",
    "variable_coverage_pct": "float",
    "text_style_coverage_pct": "float",
    "outcome": "accepted | edited | deleted",
    "time_to_outcome_seconds": "integer",
    "generation_latency_ms": "integer"
  }
}
```

```json
{
  "event": "compliance_panel_viewed",
  "properties": {
    "file_id": "string",
    "generation_session_id": "string",
    "component_coverage_pct": "float",
    "non_compliant_element_count": "integer",
    "time_spent_on_panel_seconds": "integer",
    "panel_dismissed": "boolean"
  }
}
```

```json
{
  "event": "non_compliant_element_replaced",
  "properties": {
    "file_id": "string",
    "generation_session_id": "string",
    "original_element_type": "string",
    "replacement_component_id": "string",
    "was_suggested_replacement": "boolean"
  }
}
```

```json
{
  "event": "library_gap_detected",
  "properties": {
    "org_id": "string",
    "generation_session_id": "string",
    "missing_element_type": "string",
    "prompt_context": "string",
    "gap_count_this_week": "integer"
  }
}
```

---

## 10) Success metrics

### North Star for this feature

**% of `ai_make_designs_generated` sessions with `uses_org_library_components = true` and `outcome != deleted`** (i.e., library-grounded generations that the designer keeps)

Target 90 days post-launch: >45% of all `Make Designs` sessions in orgs with published org-scoped libraries.

### Leading indicators

| Metric | Baseline (estimated) | Target (90 days) | Why it matters |
|---|---|---|---|
| `component_coverage_pct` mean for library-grounded sessions | N/A (feature does not exist) | >80% | Core quality signal: are generations actually using the library? |
| `outcome = accepted or edited` rate for library-grounded sessions | N/A | >55% | Acceptance proxy: designers are keeping AI output, not discarding |
| `compliance_panel_viewed` -> `panel_dismissed without replacing` rate | N/A | <30% | If designers are not engaging with the panel, it is not adding value or is too friction-heavy |
| `library_gap_detected` events per org per month | N/A | Track and surface to design systems teams | Measures which component types are missing from the library; drives library roadmap |

### Guardrail metrics

| Metric | Threshold | Risk |
|---|---|---|
| `ai_make_designs_generated` volume (all orgs) | Must not decline MoM after launch | Feature must not reduce overall AI generation usage by adding friction |
| Generation latency P95 for orgs with library | Must stay <10s | Performance regression would cause users to abandon generation before it completes |
| `outcome = deleted` rate for library-grounded sessions | Must not exceed 45% | If deletion rate is high, component coverage or variant selection quality is poor |
| Design systems team NPS (surveyed at 30 days) | Must not decline vs. pre-launch | If design systems leads feel AI generation undermines their library, the feature is harmful to the org |

---

## 11) Risks and open questions

### Risks

1. **Library context window overflow:** Orgs with 500+ components exceed current LLM context limits. The relevance filtering approach (top 50-100 components by semantic similarity to the prompt) is an approximation. If the relevance ranking selects wrong components, generation quality drops.

   *Mitigation:* Use a two-pass approach - first pass generates a component type manifest from the prompt (e.g., "Button, Input, Card, Toggle"); second pass fetches the exact matching components from the library by name/semantic match.

2. **Variant selection errors:** The library may have 12 Button variants. Selecting the wrong one (e.g., `Button/secondary/disabled` instead of `Button/primary/default`) is technically "using a library component" but produces incorrect output.

   *Mitigation:* Include variant semantic metadata in the library context (e.g., "primary = main CTA; secondary = supporting action; ghost = low-emphasis"). Measure `correct variant selected` accuracy separately from `component type matched` accuracy.

3. **Library staleness:** If the designer's file is using a library that has pending updates not yet accepted, the generation pipeline may use outdated component definitions.

   *Mitigation:* Always use the latest published library version for context injection, regardless of whether the file has accepted the latest update.

4. **Privacy and IP concerns at enterprise:** Some enterprise orgs treat their component library as proprietary IP. They may not want library structure data transmitted to Figma's AI infrastructure in a form that could influence model training.

   *Mitigation:* Library context is used only for in-session generation and is not retained after session end unless the org admin has opted in to model training. Make this explicit in the privacy settings page.

### Open questions - resolved in v3 (see Section 20)

All five open questions are now resolved. See Section 20 for decisions and rationale.

---

---

## 12) Competitive analysis

### How competitors handle AI generation

| Tool | AI generation approach | Design system awareness | Figma's position |
|---|---|---|---|
| **v0 (Vercel)** | Generates production React/Tailwind code from a prompt | No design system concept - generates from design tokens only if supplied | v0 routes around the design file entirely; not a direct competitor for this feature |
| **Framer AI** | Generates responsive web layouts; can be published as production sites | No component library linkage; generates with Framer's built-in component set only | Closer competitor - Framer AI is system-unaware but targets a different user (web publishers, not product design teams) |
| **Galileo AI** | Generates high-fidelity UI mockups from text prompts | Claims Figma library import but component matching is inconsistent and requires manual cleanup | Direct competitor - Galileo AI is attempting exactly this feature; Figma's advantage is owning the library data natively |
| **Uizard** | AI wireframe generation from sketches or text | No library integration; generates from a generic component set | Targets early-stage wireframing; not in the enterprise design system space |
| **Spellbook / Diagram (Figma plugin)** | Figma-native plugin for AI generation inside the canvas | Has partial library awareness via plugin API - can read file components but not org-scoped libraries | Closest existing approach; limited by plugin API access compared to native Figma infrastructure |
| **Adobe Firefly in XD** | Adobe XD was discontinued; Firefly is focused on image generation | Not applicable | Not a threat |

**Key insight:** No competitor has native access to org-scoped library data at the infrastructure level. Galileo AI and Diagram plugins are the closest, but both work via surface-level API access - they can read visible components in a file, not the full published org library with variant metadata and variable bindings. Figma's infrastructure advantage is the moat here. The feature must be shipped before a well-funded competitor builds the same capability via API and erodes Figma's AI narrative.

### Competitive win/loss scenarios

**Figma wins** when:
- An enterprise team with a mature design system evaluates AI generation tools. No external tool has access to their org library; Figma does. The output quality gap is decisive.
- A design systems lead evaluates whether to allow AI generation in their org. Library-grounded generation with a compliance panel is the only AI tool that makes this decision easy to say yes to.

**Figma loses** when:
- A startup with no design system evaluates Make Designs vs. v0. v0 generates production code; Make Designs generates a Figma frame. If the startup doesn't have a designer to refine the frame, v0 wins by delivering the final artifact.
- An org uses a non-Figma design system (e.g., Storybook as the source of truth, with Figma as a secondary tool). Library context injection requires the primary system to live in Figma.

---

## 13) Data model - library context injection

The generation pipeline requires a structured representation of the org library. This is the schema for the context payload sent to the generation model at session time.

```json
{
  "library_context": {
    "org_id": "string",
    "library_file_id": "string",
    "library_version": "string",
    "generated_at": "ISO8601 timestamp",
    "components": [
      {
        "component_id": "string",
        "name": "string",
        "category": "string",
        "description": "string",
        "variants": [
          {
            "variant_id": "string",
            "name": "string",
            "properties": {
              "type": "string",
              "state": "string",
              "size": "string"
            },
            "semantic_label": "string"
          }
        ],
        "usage_frequency": "integer"
      }
    ],
    "color_variables": [
      {
        "variable_id": "string",
        "name": "string",
        "collection": "string",
        "value_light": "hex string",
        "value_dark": "hex string",
        "semantic_role": "string"
      }
    ],
    "text_styles": [
      {
        "style_id": "string",
        "name": "string",
        "font_family": "string",
        "font_size": "integer",
        "font_weight": "integer",
        "line_height": "float",
        "semantic_role": "string"
      }
    ],
    "relevance_scores": {
      "component_id": "float"
    }
  }
}
```

**Context size management:**
- Full library payload for a 200-component org: estimated ~40-80KB before compression.
- Token budget for library context in generation model: ~8,000 tokens (out of ~32,000 total context).
- For orgs with >200 components: apply relevance filtering to top 80 components by cosine similarity between component name embeddings and the prompt embedding.
- `usage_frequency` is included to bias relevance ranking toward components the org actually uses.

---

## 14) API contract sketch (internal)

The library context injection service exposes a single endpoint consumed by the Make Designs generation pipeline.

```
POST /internal/ai/library-context

Request:
{
  "org_id": "string",
  "file_id": "string",
  "prompt": "string",
  "max_components": 80
}

Response:
{
  "library_context": { ...schema above... },
  "context_size_tokens": "integer",
  "was_truncated": "boolean",
  "truncation_method": "relevance_filter | none",
  "latency_ms": "integer"
}
```

**SLOs for this service:**
- P50 latency: <300ms
- P95 latency: <800ms
- Availability: 99.9% (same as core generation pipeline)
- Cache TTL: 10 minutes per org library version (library context is stable between publishes)

---

## 15) Acceptance criteria with test cases

### FR-01: Library context injection

**Happy path:**
- Given: Org has a published org-scoped library with 50 components and 20 variables.
- When: Designer triggers `Make Designs` with prompt "Settings page with toggles."
- Then: `library_context_injected = true` in generation event; latency overhead vs. non-library generation is <2s.

**Edge case - no library:**
- Given: File belongs to a Free-tier team with no published library.
- When: Designer triggers `Make Designs`.
- Then: `library_context_injected = false`; generation proceeds with generic model; no error shown; UX identical to current.

**Edge case - library with 500+ components:**
- Given: Org has 520 components in their published library.
- When: Designer triggers `Make Designs` with prompt "Dashboard with charts and filters."
- Then: `was_truncated = true`; context contains top 80 components by relevance; generation completes within 12s P95.

---

### FR-02: Component-first frame generation

**Happy path:**
- Given: Library contains `Button/primary/default`, `Input/text/default`, `Card/content`.
- When: Prompt is "User onboarding form with name, email, and submit button."
- Then: Generated frame contains Button instance (variant: primary/default), Input instances, Card wrapper instance. Component coverage: >80%.

**Edge case - prompt requests an unlibrary-able pattern:**
- Given: Library has no `DataTable` component.
- When: Prompt is "Data table with sortable columns."
- Then: Table rendered as raw frame (not a component); compliance panel shows warning "Table not in your library"; `library_gap_detected` event fires.

**Edge case - wrong variant selected:**
- Given: Library has `Button/primary/default` and `Button/primary/disabled`.
- When: Prompt is "Form with a disabled submit button."
- Then: Generated button uses `Button/primary/disabled` variant (not default). Variant semantic metadata must be used, not just component type matching.

---

### FR-05: Compliance panel

**Happy path:**
- When: Frame generation completes.
- Then: Compliance panel renders within 500ms; shows component coverage %, variable coverage %, text style coverage %, and a list of non-compliant elements.

**Edge case - 100% compliant frame:**
- When: Generated frame achieves 100% component, variable, and text style coverage.
- Then: Compliance panel shows a "Looks good" state with no non-compliant list; panel is dismissible with a single click.

**Edge case - designer dismisses panel immediately:**
- When: Designer dismisses compliance panel without reviewing.
- Then: `panel_dismissed = true` in `compliance_panel_viewed` event; no blocking gate; designer can accept frame.

---

## 16) Experimentation framework

### Experiment 1: Compliance panel - default visible vs. opt-in

**Hypothesis:** Designers who see the compliance panel by default make better accept/edit/discard decisions and show higher `component_coverage_pct` in accepted frames than designers who must opt in to view it.

**Setup:**
- Control: Compliance panel is opt-in (collapsed by default; "View compliance" link).
- Treatment: Compliance panel is open by default after generation.

**Primary metric:** `component_coverage_pct` mean for accepted frames (treatment vs. control).
**Secondary metric:** `outcome = deleted` rate (if forced visibility increases deletion, the feature is adding friction without value).
**Guardrail:** `ai_make_designs_generated` volume must not decline in treatment group (panel must not deter generation attempts).
**Duration:** 4 weeks; minimum detectable effect: 5 percentage points in coverage.
**Rollout:** 50/50 split on qualifying orgs (published library, Professional+ tier).

---

### Experiment 2: Library gap digest - real-time notification vs. weekly summary

**Hypothesis:** Design systems leads who receive real-time notifications about library gaps act on them faster (add missing components within 14 days) than leads who receive a weekly summary.

**Setup:**
- Control: Weekly digest email summarising `library_gap_detected` events from the past 7 days.
- Treatment: Real-time Figma notification when 3+ sessions in 7 days detect the same missing component type.

**Primary metric:** Time from first `library_gap_detected` event to new component published in the library (for the missing type).
**Secondary metric:** Design systems lead opt-out rate from notifications (treatment must not exceed 15% opt-out).
**Guardrail:** Total `library_gap_detected` events must not increase (would suggest the feature is creating new gaps, not surfacing existing ones).
**Duration:** 8 weeks (library updates are slow; need longer window to measure).

---

### Experiment 3: Compliance gate - hard gate vs. soft warning for low-coverage frames

**Hypothesis:** A soft warning ("Low design system coverage - review before sharing") on frames with <50% coverage reduces non-compliant frames handed off to engineering, without increasing `outcome = deleted` rate.

**Setup:**
- Control: No gate or warning; designer can accept any frame regardless of coverage.
- Treatment: Frames with <50% component coverage show a yellow "Low coverage" badge on the canvas, visible to all file viewers.

**Primary metric:** % of frames accepted with <50% coverage that are subsequently shared with a Dev Mode user (proxy for engineer handoff of non-compliant frames).
**Secondary metric:** `outcome = deleted` rate for low-coverage generations (must not increase more than 10% relative - we want designers to edit, not discard).
**Guardrail:** Overall `ai_make_designs_generated` volume must not decline.

---

### Experiment 4: Prompt pre-flight - component type manifest preview before generation

**Hypothesis:** Showing designers a 2-second preview of which component types the AI intends to use ("I'll use Button, Card, Toggle from your library") before starting generation increases `outcome = accepted` rate by reducing surprise when the frame appears.

**Setup:**
- Control: Current flow - generation starts immediately after prompt submission.
- Treatment: A 2-second "planning" step displays a bulleted list of component types the AI identified in the prompt. Designer can cancel or proceed.

**Primary metric:** `outcome = accepted` rate (not edited, not deleted) - hypothesis is that transparency reduces regret-driven deletion.
**Secondary metric:** Time-to-outcome (must not increase by more than 5 seconds in treatment - the 2s preview is net positive only if it reduces downstream editing time).
**Guardrail:** `ai_make_designs_generated` submission rate must not drop (the preview step must not deter generation attempts).
**Duration:** 3 weeks; 50/50 split on orgs with published libraries.
**Kill condition:** If `outcome = deleted` rate increases more than 5% relative in treatment, the pre-flight preview is adding anxiety, not clarity - kill immediately.

---

### Experiment 5: Variant confidence labelling in the compliance panel

**Hypothesis:** When the compliance panel shows a confidence score for each variant selection ("Button/primary/default - 87% confident this is the right variant"), designers fix more variant mismatches pre-handoff, reducing `correct variant selected` rework in engineering.

**Setup:**
- Control: Compliance panel shows component instance with no confidence score.
- Treatment: Compliance panel shows component instance + a confidence percentage for the variant choice. Variants below 70% confidence are highlighted in amber.

**Primary metric:** Rate at which designers click "swap variant" for amber-highlighted instances (proxy for low-confidence variant corrections pre-handoff).
**Secondary metric:** Dev Mode session depth after accepting library-grounded frames - specifically whether engineers are raising variant mismatch comments on accepted frames.
**Guardrail:** Compliance panel dismiss rate must not increase (confidence labels must not make the panel feel noisier or harder to parse).
**Duration:** 4 weeks.

---

## 17) Dependency map

| Dependency | Team owner | Status | Blocking phase | SLO |
|---|---|---|---|---|
| Library context injection service (new backend service) | AI platform team | Not started | Phase 0 | P95 latency <800ms; availability 99.9% |
| Component semantic metadata indexing (variant labels, usage frequency) | Design systems infrastructure team | In progress | Phase 0 | Index freshness: updated within 5 minutes of library publish |
| `library_context_injected` field in generation event pipeline | Data / analytics team | Not started | Phase 0 | Event delivery lag <30s; zero dropped events |
| Compliance panel UI component | Design tooling team | Not started | Phase 0 | Render latency <500ms post-generation |
| `library_gap_detected` event + notification pipeline | Data + notification team | Not started | Phase 1 | Gap event deduplication window: 7 days per org per element type |
| Privacy controls for library data (opt-in model training flag) | Privacy / legal team | In review | Phase 0 | Legal sign-off required before any Phase 0 sessions |
| Relevance filtering model for large libraries | AI research team | Prototype exists | Phase 1 | Relevance filter P95 latency <200ms; >75% recall for top-3 relevant components |
| Code Connect parity for AI-generated components | Dev Mode team | Not started | Phase 2 | Generated component instances must link to Code Connect entries if available |

---

## 18) Expanded risk register

| Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|
| Library context window overflow for 500+ component orgs | High | High - generation quality degrades for largest enterprise customers | Two-pass generation: prompt -> component type manifest -> targeted library fetch | AI platform |
| Wrong variant selected (e.g., disabled button for active CTA) | Medium | High - designer ships incorrect state to engineering | Variant semantic metadata in context; measure and target >75% correct variant rate | AI research |
| Library context injection latency adds >3s to generation P95 | Medium | Medium - users abandon generation if it feels slower | Cache library context with 10-min TTL; warm cache on file open for qualifying orgs | AI platform |
| Privacy backlash from enterprise over library data in AI pipeline | Low | High - enterprise orgs turn off AI generation entirely | Explicit opt-in for model training; clear in-product disclosure; library context used only for in-session generation | Privacy / legal |
| Design systems leads resist feature because it "locks" AI to current library | Low | Medium - reduces enterprise adoption | Compliance panel shows gaps; gap notifications drive library expansion; framed as "AI that improves your design system" | PM / DS relations |
| Galileo AI or similar ships design-system-aware generation before Figma GA | Medium | High - loses first-mover narrative | Accelerate Phase 0 to 4 weeks by parallelising backend service and UI work | PM / engineering |
| `non_compliant_element_replaced` rate is low (designers ignore suggestions) | Medium | Low - feature works but compliance panel is not helping | A/B test compliance panel visibility (Experiment 1); if <20% replacement rate, reduce panel prominence and focus on coverage score only | PM |
| Multi-library conflict causes wrong component set injected | Low | High - entire generation session uses wrong brand system | Enforce library priority order: org > team > file; log `library_scope_used` in every generation event | AI platform |

---

## 19) Open questions - resolved (v2)

**Q1: What is the right threshold for the compliance warning - 80% or configurable?**

Decision: 80% is the default threshold for surfacing the non-compliant elements list. Org admins can adjust this threshold in the Design Systems settings page (range: 50%-95%). This gives design systems leads control without making it a per-designer setting (which would lead to inconsistent standards). The compliance panel always shows the actual coverage percentage regardless of the threshold - the threshold only controls whether the non-compliant list is shown prominently or collapsed.

**Q2: Compliance panel - visible by default or opt-in?**

Decision: Run Experiment 1 to determine default. Pre-launch default: visible (open) for first 10 generation sessions per user, then collapsed by default. This mirrors how Figma introduces other contextual panels (e.g., the auto-layout suggestion prompt). After 10 sessions, the user has seen the panel enough to use it intentionally. Re-open trigger: if `component_coverage_pct` is below the org threshold, force-open regardless of user preference.

**Q3: Multi-library orgs - which library takes precedence?**

Decision: Priority order for context injection: (1) org-scoped library, (2) team-scoped library, (3) file-local components. If org library and team library have a component with the same name, org library takes precedence. This mirrors how Figma resolves library conflicts in the existing product. For orgs with multiple org-scoped libraries (e.g., one for web, one for mobile), the user selects the active library in the generation panel before triggering Make Designs - defaults to the most recently updated org library.

---

## 20) Open questions - resolved (v3)

**Q4: Should library gap signals be surfaced to design systems leads in real time (as Figma notifications) or in a weekly digest?**

Decision: Neither exclusively. The right delivery mode depends on the gap's signal strength. Implement a tiered notification model:

- **Tier 1 - Immediate notification:** A single missing component type appears in 5+ generation sessions within 24 hours. This is a strong signal of an active team need. Surface as a Figma bell notification with a link to the gap report. Threshold is high enough to suppress false positives from a single PM experimenting with prompts.
- **Tier 2 - Weekly digest:** Any gap that crosses the 3-session threshold in 7 days but not the Tier 1 threshold. Batched into a weekly email to the design systems lead with a table of missing component types, session count, and a sample prompt.
- **Opt-out:** Design systems leads can downgrade Tier 1 to Tier 2 for any gap type they have already triaged ("we know we don't have a Table component; stop reminding us"). This prevents notification fatigue for known gaps.

Run Experiment 2 to validate whether Tier 1 real-time notifications materially shorten time-to-component vs. the Tier 2 weekly digest. If the difference is less than 7 days (i.e., real-time and weekly lead to the same outcome within the sprint cycle), simplify to weekly-only to reduce notification surface complexity.

**Q5: Is the "correct variant selected" accuracy measurable without ground truth labels?**

Decision: Yes, via three observable proxy signals, with explicit limitations.

- **Primary proxy - post-acceptance variant swap rate:** After accepting a library-grounded frame, if the designer immediately swaps a variant within the first 60 seconds of the editing session (before making other changes), that is a strong signal the AI selected the wrong variant. Track this as `variant_swapped_immediately` (window: <60s post-accept). This is not a perfect signal - designers sometimes swap variants by preference, not by correction - but at population scale it is a reliable quality indicator.
- **Secondary proxy - Dev Mode comment rate on variant elements:** When an engineer in Dev Mode leaves a comment on a component instance from an AI-generated frame, the comment is a proxy for a variant mismatch or compliance issue. Track `devmode_comment_on_ai_generated_component` events; analyse whether comment rate is higher for AI-generated instances vs. manually placed instances.
- **Long-term ground truth path:** In Phase 2 of the rollout, add an optional thumbs-up/thumbs-down micro-feedback on each variant in the compliance panel ("Is this the right variant?"). Collect labelled data for 8 weeks, use it to fine-tune the variant selection model, and measure whether `variant_swapped_immediately` rate decreases in the post-fine-tuning cohort vs. pre-fine-tuning cohort.

**Accuracy target:** >75% of variant selections are the contextually correct variant, measurable via `variant_swapped_immediately` rate below 15% within 60 seconds of frame acceptance. Establish this baseline in Phase 0 dogfood before Phase 1 beta launch.

---

## 21) Phased rollout plan

### Phase 0 - Internal dogfood (weeks 1-4)

**Goal:** Validate that the library context injection pipeline works end-to-end without breaking existing Make Designs behaviour. Establish accuracy baselines for component coverage, variant selection, and latency.

**Scope:**
- Enable for Figma's internal design team only (estimated 150-300 designers).
- Library context injected when user triggers Make Designs in a file with a published org library.
- Compliance panel shown post-generation.
- All 4 generation events instrumented and logging to the internal analytics pipeline.
- Generic generation remains the fallback for any session where context injection fails.

**Go/no-go criteria to advance to Phase 1:**

| Criterion | Threshold | Source |
|---|---|---|
| `library_context_injected = true` rate for qualifying sessions | >98% | `ai_make_designs_generated` event |
| Library context injection service P95 latency | <800ms | Service metrics |
| Generation end-to-end P95 latency (library-grounded sessions) | <10s | `generation_latency_ms` field |
| `component_coverage_pct` mean across all internal sessions | >70% | `ai_make_designs_generated` event |
| `outcome = deleted` rate for library-grounded sessions | <50% | Same event |
| Zero critical bugs (incorrect library applied, privacy data leakage) | 0 P0 incidents | Incident log |

**Kill switch:** Feature flag per org. If any criterion is missed or a P0 incident occurs, disable library context injection for all sessions within 30 minutes; revert to generic generation with no user-facing error.

**Rollback plan:** Library context injection is additive to the generation pipeline - removing it reverts seamlessly to the existing generic generation path. No data migration required. All events continue firing with `library_context_injected = false`.

---

### Phase 1 - Closed beta: enterprise design systems teams (weeks 5-10)

**Goal:** Validate output quality with enterprise orgs that have mature, high-component-count libraries. Collect qualitative signal from design systems leads on compliance panel utility and gap notification relevance.

**Scope:**
- Invite 10-15 enterprise orgs with published org-scoped libraries containing >100 components and >20 variables.
- Manual selection: prioritise orgs with active Make Designs usage in the past 30 days (ensures we are testing with engaged users, not cold-start testers).
- Show "Generated with your design system" label on AI-generated frames.
- Run Experiment 2 (gap notification) as a 50/50 split within this cohort.
- Collect structured qualitative feedback via an in-Figma survey (5 questions, triggered 7 days post-enable): "Did the AI use the right components?", "Was the compliance panel useful?", "Did you change fewer elements than with generic generation?", "Would you use this for real handoff work?", "Net Promoter Score for this feature."

**Go/no-go criteria to advance to Phase 2:**

| Criterion | Threshold | Source |
|---|---|---|
| `component_coverage_pct` mean for beta orgs | >78% | Event data |
| `outcome = accepted or edited` rate for library-grounded sessions | >50% | Event data |
| `variant_swapped_immediately` rate (<60s post-accept) | <20% | Event data |
| Survey: "Did the AI use the right components?" positive response rate | >60% | In-product survey |
| Survey: "Would you use this for real handoff work?" positive response rate | >40% | In-product survey |
| Generation latency P95 for orgs with 200-400 component libraries | <12s | Service metrics |
| No increase in `ai_make_designs_generated` volume drop (orgs must not reduce generation attempts) | Volume must not decline >10% after feature enable | Event data |

**Kill switch:** Per-org disable flag. If any beta org reports generation quality regressions or privacy concerns, disable for that org within 2 hours. If >3 orgs report the same regression category, pause the entire Phase 1 cohort and investigate.

**Auto-kill triggers:**
- If `component_coverage_pct` mean drops below 60% for any org cohort after 2 weeks, pause and audit the relevance filtering model.
- If generation latency P95 exceeds 15s for any library size category, pause and audit the context injection service.

---

### Phase 2 - Opt-in GA: Professional+ orgs with published libraries (weeks 11-16)

**Goal:** Make library-grounded generation available to all Professional and Organisation tier orgs that have a published shared library. Feature is opt-in at the org admin level to give design systems leads control over adoption timing.

**Scope:**
- All Professional and Organisation tier orgs with at least one published shared library see a banner: "Your design system is now available in Make Designs - enable library-grounded generation."
- Org admins can enable or disable the feature in the AI settings page.
- Free users see no change (generic generation only).
- Compliance panel defaults to open for first 10 sessions per user, then auto-collapses (per Q2 resolution).
- Experiment 1 (compliance panel default visibility) runs across this population as a 50/50 split.
- Experiment 4 (prompt pre-flight) runs across 20% of this population.
- Library gap notifications go live per the Tier 1/Tier 2 model from Q4 resolution.

**Go/no-go criteria to advance to Phase 3:**

| Criterion | Threshold | Source |
|---|---|---|
| Opt-in rate among eligible orgs | >30% within 4 weeks of banner launch | Product analytics |
| `component_coverage_pct` mean across all opted-in orgs | >75% | Event data |
| `outcome = accepted or edited` rate for library-grounded sessions (all opted-in orgs) | >50% | Event data |
| Free -> Professional conversion rate (guardrail - must not be hurt by the feature) | Must stay within 5% of pre-launch 30-day baseline | Revenue metrics |
| Generation latency P95 across all opted-in orgs | <10s | Service metrics |
| Design systems lead NPS (surveyed at day 30 post-enable) | NPS >20 | In-product survey |
| `library_context_injection_failure_rate` | <0.5% | Service metrics |

**Kill switch:** Org-level feature flag. Org admins can disable at any time from the AI settings page. Global kill: if `library_context_injection_failure_rate` exceeds 2% across the full population, disable library injection globally and fall back to generic generation within 15 minutes.

---

### Phase 3 - Default-on GA: all tiers and community libraries (week 17+)

**Goal:** Design-system-aware generation is the default for every Make Designs session that has access to any library - org, team, file-local, or Figma community library (Material Design, iOS, etc.). Free users get the benefit of community libraries as a context source.

**Scope:**
- Library context injection is on by default for all tiers. Org admins can opt out in settings.
- Community template libraries (Figma Material Design, iOS Design System) are used as context for Free users when no custom library exists.
- "AI used these components" provenance panel is shown inline in the accepted frame, visible to all file collaborators.
- Figma publishes a public quality benchmark: "% of Make Designs outputs requiring zero manual component replacements in library-enabled sessions" - target >75%.
- Variant confidence labelling (Experiment 5) runs across 30% of the Phase 3 population.

**Per-pillar kill switches (independent):**
- Variable mapping: disable via `disable_variable_injection` flag if variable conflicts cause incorrect token application in >5% of sessions.
- Component matching: disable via `disable_component_matching` flag if component misidentification rate exceeds 15% as measured by `variant_swapped_immediately` rate.
- Community library fallback: disable via `disable_community_library_context` flag if community library components degrade output quality for custom-brand orgs (measured by `outcome = deleted` rate increasing >10% in Free user cohort vs. Phase 2 baseline).

---

## 22) Launch readiness checklist

This checklist must be signed off by the accountable owner before Phase 2 GA launch. All items are blocking unless explicitly marked as non-blocking.

### Engineering readiness

- [ ] Library context injection service deployed to production with 99.9% availability SLO; load tested at 3x Phase 1 peak volume (AI platform team)
- [ ] Context injection cache layer (10-minute TTL per org library version) validated under concurrent write load (AI platform team)
- [ ] Relevance filtering model for >200 component libraries benchmarked: >75% recall for top-3 components in 50 benchmark prompts (AI research team)
- [ ] Component semantic metadata index freshness verified: updated within 5 minutes of a library publish event in staging (Design systems infra team)
- [ ] All 4 new analytics events (`ai_make_designs_generated` additions, `compliance_panel_viewed`, `non_compliant_element_replaced`, `library_gap_detected`) emitting correctly in production with zero dropped events in 48-hour canary (Data team)
- [ ] Feature flags configured for per-org enable/disable; global kill switch tested in staging (Engineering)
- [ ] Fallback to generic generation triggers correctly when `library_context_injected = false`; verified via chaos injection test (Engineering)

### Legal and privacy readiness

- [ ] Privacy review complete: library context data used only for in-session generation; no retention after session end (Privacy team)
- [ ] Org admin consent flow for model training opt-in shipped to staging and reviewed by legal (Privacy / Legal)
- [ ] Data processing addendum (DPA) language updated for enterprise customers reflecting library data handling (Legal)
- [ ] Enterprise privacy FAQ page updated with library context handling documentation (Privacy / Legal)

### Design and product readiness

- [ ] Compliance panel passes accessibility audit: WCAG 2.1 AA compliance for all interactive elements (Design tooling team)
- [ ] Compliance panel copy reviewed and approved by content design (no em-dashes; clear language for non-designer users) (PM / Content design)
- [ ] "Generated with your design system" label and provenance panel reviewed by brand (Design)
- [ ] Edge cases tested manually: 0-component library, 500+ component library, multi-library org, library with no variables, library with pending unpublished updates (QA)
- [ ] User-facing error states defined for: context injection failure, generation timeout, compliance panel render failure (Design / Engineering)

### Go-to-market readiness

- [ ] Release notes drafted and reviewed (PM)
- [ ] Design systems community blog post approved for publish date aligned with Phase 2 launch (Marketing)
- [ ] CS team briefed on feature scope, common design systems lead questions, and escalation path for enterprise orgs reporting component quality issues (CS)
- [ ] Support documentation published: "How Make Designs uses your design system," "How to configure library-grounded generation," "Understanding the compliance panel" (Support)

### Monitoring readiness

- [ ] Real-time dashboard live: `component_coverage_pct` mean, `outcome = deleted` rate, `library_context_injection_failure_rate`, generation latency P50/P95/P99 (Data team)
- [ ] PagerDuty alerts configured: `library_context_injection_failure_rate` > 1% (warn), >2% (critical); generation latency P95 > 12s (warn), >15s (critical) (Engineering)
- [ ] Week 1 post-launch review meeting scheduled with AI platform, data, and PM leads (PM)

---

## 23) SLO revision history

| Metric | v1 target | v2 target | v3 target | Rationale for change |
|---|---|---|---|---|
| Library context injection P95 latency | Not specified | <800ms | <800ms; P50 <300ms | Added P50 target; most sessions benefit from the faster path |
| Generation end-to-end P95 (library sessions) | <8s | <10s | <10s for standard; <12s for 400-500 component orgs | Split by library size; large-library orgs have a separate SLO |
| `library_context_injection_failure_rate` | Not specified | <0.5% | <0.5% (warn at 1%); global kill at 2% | Added explicit alert and kill thresholds |
| Compliance panel render latency | Not specified | <500ms | <500ms; degraded-mode fallback at 800ms (show score only, no list) | Added graceful degradation path |
| Component instance accuracy (correct variant) | Not specified | >75% | >75%; measured via `variant_swapped_immediately` <15% | Made measurement method explicit |
| `outcome = accepted or edited` rate | Not specified | >55% | >55% in Phase 1 beta; >50% at Phase 2 GA scale (larger population includes lower-intent users) | Separated beta target from GA target |

---

*All metrics are directional targets. Baselines are estimated from publicly observable product behaviour, not internal Figma data.*
