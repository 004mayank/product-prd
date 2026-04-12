# PRD: Notion AI - Contextual Workspace Intelligence

<p align="center">
  <img
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/Notion.png"
    alt="Notion"
    width="200"
  />
</p>

**Product:** Notion AI  
**Author:** Mayank Malviya  
**Status:** v3 - Final PRD  
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/notion-teardown.md

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, goals/non-goals, target personas, high-level solution pillars, basic requirements, event schemas |
| v2 | Competitive analysis, detailed acceptance criteria with edge cases, full data model, context budget specification, experimentation framework, expanded risk register |
| v3 | Phased rollout plan, launch gates, kill switches, dependency map, SLO definitions, resolved open questions, final experiment backlog, complete instrumentation spec |

---

## Context

Notion AI launched in 2023 as a paid add-on (~$8/user/month) embedded directly inside the workspace. The core promise was AI-powered writing, summarisation, and drafting without leaving the page.

The product is stuck. Notion AI is a **generic LLM wrapper** inside one of the richest knowledge surfaces in the world - and it ignores that knowledge. It knows nothing about your pages, your team's vocabulary, your project history, or how your workspace is structured. Output is grammatically correct and contextually hollow. Users trial it once, then stop.

This PRD specifies **Contextual Workspace Intelligence**: a system that grounds every Notion AI response in the user's own workspace, making output specific, relevant, and worth keeping. v3 is the production-grade specification that serves as direct input to the system architecture.

---

## 1) Problem statement + why now

### The core loop failure

```
Prompt -> Generate -> [Edit heavily] -> Done
         [or]
Prompt -> Generate -> [Discard] -> Write manually
```

The AI loop breaks at the edit step. Root cause: the AI has no context about what matters in this workspace - no page history, no team terminology, no linked data. Every response starts from zero.

**Observable failure modes:**

- "Summarise this page" on a meeting notes doc returns a generic summary that misses the three decisions that actually matter - because the AI does not know which items are decisions vs. discussion noise.
- "Write a product spec" returns a five-section template with placeholder text - because the AI does not know the team's spec format, the product area, or any prior specs.
- "Improve this paragraph" returns a rewritten version with different vocabulary - because the AI does not know the author's voice or the audience.

### Competitive pressure

| Competitor | AI context capability | Threat level |
|---|---|---|
| Coda AI | Reads current doc + referenced packs | High - closing gap fast |
| Confluence AI (Atlassian Intelligence) | Pulls from full Atlassian graph (Jira + Confluence) | High for enterprise |
| Google Workspace Duet AI | Reads Drive files; knows Gmail and Calendar context | Medium - different surface |
| Obsidian + local LLM plugins | Local-first, reads full vault | Low - niche power users |
| Cursor / GitHub Copilot | Workspace-aware for code; sets user expectation | Indirect but real |

The window to own "workspace-aware AI" in productivity is 12-18 months. Notion has a richer page graph than any competitor but is not executing on it.

### Why now

- RAG pipelines are production-ready at Notion's scale. The technical blocker is prioritisation, not research.
- The AI add-on renewal cycle is hitting. Users who trialled at launch are deciding whether to renew. Utility is the only argument - and utility requires context.
- PLG signal: teams using Notion as their OS have the richest possible AI context. Losing them to Coda or Confluence on AI quality is an existential risk to the platform thesis.

---

## 2) Goals / non-goals

### Goals

1. Increase **Notion AI Weekly Active Users** (add-on subscribers triggering at least one AI action per week) by **40% relative** within 90 days of launch.
2. Increase **AI response acceptance rate** (user keeps AI output with <30% character-level edit) from ~25% estimated baseline to **45%**.
3. Increase **AI sessions per user per week** from ~1.2 to **2.5** among active add-on users.
4. Reduce **AI add-on 90-day churn** by **20% relative**.
5. Achieve **page starter acceptance rate >35%** among new blank pages for AI subscribers.

### Non-goals

- Building a sidebar chat interface. This is inline, context-aware AI only.
- Replacing search or the page graph. Context retrieval improves generation quality; navigation is unchanged.
- Fine-tuning a custom Notion model. All generation uses external models (GPT-4o / Claude 3.5) with RAG-injected context.
- Indexing pages the requesting user cannot read. Permission model is strictly respected.
- Real-time collaborative AI output. AI generation is single-user; collaborative editing is out of scope.

---

## 3) Target users / personas + JTBD

| Persona | Workspace behaviour | JTBD | Pain today | Success signal |
|---|---|---|---|---|
| **PM Spec Writer** | Creates 3-5 docs/week; heavy linked DB use; references prior specs | "Draft a spec that follows our team format and references the Q2 roadmap" | Generic output; must manually import context from 3-4 other pages | Accepts AI draft with <20% edit; prior spec referenced correctly |
| **Meeting Notes Taker** | Captures notes for 5-10 meetings/week; consistent template; Decisions DB | "Summarise this meeting and populate our Decisions database" | Summary ignores DB schema; misses what was actually decided | Summary maps to Decisions DB properties; action items correctly attributed |
| **Knowledge Base Curator** | Maintains 50-200 page wiki; enforces structure | "Rewrite this SOP to match our current process" | AI doesn't know the current process unless manually pasted | AI pulls process from linked pages; output is structurally consistent |
| **Engineering Lead** | Writes RFCs and ADRs; references past decisions | "Check whether this RFC conflicts with prior architectural decisions" | AI has no knowledge of prior ADRs; cannot detect conflicts | AI retrieves relevant ADRs; surfaces potential conflicts explicitly |
| **Solo Creator** | Personal workspace: reading lists, journals, project notes | "Write a weekly review based on my notes from this week" | AI doesn't know the user's notes or prior reviews | Weekly review references actual notes from current week's pages |

**Activation insight:** The aha moment is the first accepted draft that references real workspace content. Until then users compare Notion AI to ChatGPT. After that, they evaluate it against not having it. The product must engineer this moment in the first session.

---

## 4) Insights from teardown

1. **Loop D breaks at the edit step.** Reducing editing time by 50% requires contextual output, not a smarter model.
2. **The linked DB view is the aha moment.** AI must be DB-aware - able to read and write to linked databases (e.g., populate a Decisions DB from meeting notes).
3. **Entropy at scale is a product gap.** AI that surfaces the authoritative version or flags staleness turns a retention risk into a retention driver.
4. **The blank-page cliff is the top activation drop-off.** Workspace-contextual starters convert the blank page from a blocker into a fast start.
5. **AI add-on renewal depends on utility.** The first trial is curiosity-driven. Renewal is utility-driven. Context is the conversion mechanism.

---

## 5) Solution - three pillars

### Pillar 1: Page Context Injection

Every AI generation is preceded by injecting the current page's full content (blocks, properties, linked page summaries) into the context window.

**Context injected (priority order, subject to token budget):**
1. Current page title and breadcrumb path
2. All existing text blocks (truncated at 3,000 tokens for large pages)
3. Database properties if page is a DB entry (typed properties as structured JSON)
4. Summaries of pages explicitly `@mentioned` (up to 3 pages, 400 tokens each)

### Pillar 2: Workspace Context Retrieval (RAG)

For write/draft/summarise commands, run semantic search against the workspace index and inject top-3 relevant pages as additional context.

**Retrieval pipeline:**
1. Embed prompt + page title using `text-embedding-3-small`
2. Query workspace vector index scoped to user-readable pages
3. Return top-3 by cosine similarity; exclude current page
4. Extractive summarise each to 600 tokens
5. Inject with source attribution; show Sources disclosure below output

**Token budget:**

| Context layer | Max tokens | Priority |
|---|---|---|
| System prompt + instructions | 800 | Fixed |
| Current page content | 3,000 | Highest |
| DB properties | 400 | High |
| Linked pages via @mention (3 x 400) | 1,200 | Medium |
| RAG retrieved pages (3 x 600) | 1,800 | Medium |
| User prompt | 500 | Fixed |
| Generation budget | 2,000 | Fixed |
| **Total** | **~9,700** | Within GPT-4o 128k window |

### Pillar 3: AI-Powered Page Starter

When a blank page is created, offer a contextual structure suggestion based on:
- Page title (if entered)
- Parent section type (meeting notes, projects, wiki, personal)
- 3 most recently created pages in the same section

The starter inserts heading blocks only - giving structure without presuming content.

---

## 6) Detailed requirements + acceptance criteria

### Req 1 - Page context injection

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 1.1 | Inject page title, breadcrumb, and all text blocks before every generation | AI references page title in >80% of test cases | >500 blocks: truncate to 3,000 tokens, append `[page truncated]` marker |
| 1.2 | Inject all DB property values when page is a DB entry | AI references at least one property in >70% of DB entry test cases | Empty properties injected as null; formula properties as computed value |
| 1.3 | Inject summaries of up to 3 `@mentioned` pages | Linked summaries present in context; permission verified per page | No read permission: skip silently; do not expose page existence |
| 1.4 | Context injection adds <800ms to generation P95 latency | Alert fires if P95 exceeds 1,000ms | Timeout >1,200ms: fall back to no injection; log `ai_context_fallback` |

### Req 2 - Workspace retrieval (RAG)

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 2.1 | Retrieve top-3 relevant pages for write/draft/summarise commands | Retrieval <500ms P95; excludes current page; all results user-readable | Timeout >600ms: skip RAG, proceed page-only; log `ai_rag_timeout` |
| 2.2 | Inject retrieved pages as extractive summaries (max 600 tokens each) | Total RAG injection does not exceed 1,800 tokens | Pages <200 tokens: inject full content without summarisation |
| 2.3 | Show "Sources" disclosure below every RAG-enabled output | Titles + links; collapsed default; opens on click; renders within 200ms | 0 results: no disclosure shown |
| 2.4 | Vector index updated within 24h of creation or material edit (>50 token change) | Alert fires if median lag exceeds 24h | Deleted pages: remove from index within 1h; archived: flag as archived, keep indexed |
| 2.5 | Retrieval scoped strictly to user-readable pages | Zero permission leakage in security test suite | Guests: scoped to their specific granted pages only |
| 2.6 | User can disable workspace retrieval in AI settings | Per-workspace toggle; on by default for existing subscribers; off by default for new workspaces | Subscription lapses: toggle auto-disabled; re-enabled on renewal |

### Req 3 - AI page starter

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 3.1 | Show "Start with AI" prompt on blank page within 1.5s of load | Appears for AI subscribers with 0 blocks only | Empty title: show "What will this page be about?"; defer structure suggestion until title entered |
| 3.2 | Generate structure based on title + parent section + sibling pages | Relevant headings in >75% of qualitative ratings; 3-7 headings | <3 sibling pages: use title-only inference |
| 3.3 | Accept inserts headings; dismiss is permanent per page; ignore fades after 10s | All three states handled; dismiss persisted as `ai_starter_dismissed: true` | User types before fade: auto-dismiss; do not interrupt |
| 3.4 | Shown max once per page; AI subscribers only | Re-shown only via `/AI start` command | Add-on lapses: suppress prompt; no paywall shown on page |

### Req 4 - Context-aware DB population

| ID | Requirement | Acceptance criteria | Edge case |
|---|---|---|---|
| 4.1 | Offer to populate empty DB properties after generation on DB entry pages | "Fill properties" button shown if >2 properties empty and AI output contains candidates | All properties filled: no button shown |
| 4.2 | Extract and map property values to correct property types | >80% accuracy for text, date, select in internal testing | Multi-select and relation: suggest only, never auto-populate |

---

## 7) Data model

### `ai_generation_request`

```json
{
  "request_id": "uuid",
  "user_id": "uuid",
  "workspace_id": "uuid",
  "page_id": "uuid",
  "command": "write | summarise | draft | improve | translate | fix_spelling | page_starter",
  "prompt_text": "string",
  "context": {
    "page_title": "string",
    "page_breadcrumb": ["string"],
    "page_block_count": "int",
    "page_token_count": "int",
    "db_properties": {"key": "value"},
    "linked_page_ids": ["uuid"],
    "retrieved_page_ids": ["uuid"],
    "context_token_total": "int",
    "rag_enabled": "bool",
    "fallback_triggered": "bool"
  },
  "model": "gpt-4o | claude-3-5-sonnet",
  "status": "pending | generating | completed | failed | fallback",
  "created_at": "ISO8601",
  "completed_at": "ISO8601",
  "latency_ms": "int"
}
```

### `ai_generation_outcome`

```json
{
  "outcome_id": "uuid",
  "request_id": "uuid",
  "user_id": "uuid",
  "workspace_id": "uuid",
  "page_id": "uuid",
  "outcome": "accepted | discarded | partially_accepted",
  "edit_delta_pct": "float",
  "time_to_outcome_ms": "int",
  "sources_disclosure_opened": "bool",
  "db_properties_populated": "bool",
  "ts": "ISO8601"
}
```

### `workspace_vector_index_entry`

```json
{
  "page_id": "uuid",
  "workspace_id": "uuid",
  "embedding_vector": "[float_1536]",
  "content_hash": "sha256",
  "token_count": "int",
  "is_archived": "bool",
  "is_deleted": "bool",
  "last_indexed_at": "ISO8601",
  "page_updated_at": "ISO8601",
  "index_version": "int"
}
```

---

## 8) Success metrics + instrumentation

### North Star
**Notion AI Weekly Active Users** - add-on subscribers who trigger at least one AI action per week.

### Input metrics

| Metric | Baseline (est.) | 90-day target | Instrumentation event |
|---|---|---|---|
| AI response acceptance rate | ~25% | 45% | `ai_generation_outcome.outcome = accepted` |
| AI sessions per user per week | ~1.2 | 2.5 | `ai_session_started` count / user / week |
| Page starter acceptance rate | n/a | >35% | `ai_page_starter_accepted` |
| RAG context used rate | n/a | >60% of write/draft/summarise | `retrieved_page_ids.length > 0` |
| Sources disclosure opened rate | n/a | >20% of RAG responses | `ai_sources_disclosure_opened` |
| DB property population rate | n/a | >25% of DB entry generations | `db_properties_populated = true` |

### Guardrail metrics

| Metric | Constraint | Owner |
|---|---|---|
| AI generation P95 latency | <5s end-to-end | Platform |
| RAG retrieval P95 latency | <500ms | ML Infra |
| Context injection fallback rate | <5% of sessions | Platform |
| Permission leakage incidents | Zero | Security |
| AI add-on 30-day trial conversion | Must not decrease | Growth |
| Page load P95 (pages with AI blocks) | Must not increase >200ms | Web Perf |

### SLO definitions

| SLO | Target | Measurement window | Alert threshold |
|---|---|---|---|
| AI generation availability | 99.5% | Rolling 7 days | <99.0% pages 24h alert |
| RAG retrieval availability | 99.0% | Rolling 7 days | <98.5% triggers fallback-only mode |
| Index freshness (median lag) | <24h | Rolling 24h | >36h lag triggers reindex job |

### Full instrumentation event schemas

```json
// ai_session_started
{
  "event": "ai_session_started",
  "user_id": "uuid",
  "workspace_id": "uuid",
  "page_id": "uuid",
  "trigger": "slash_command | ask_ai_button | page_starter | keyboard_shortcut",
  "ts": "ISO8601"
}

// ai_generation_outcome
{
  "event": "ai_generation_outcome",
  "outcome_id": "uuid",
  "request_id": "uuid",
  "user_id": "uuid",
  "workspace_id": "uuid",
  "page_id": "uuid",
  "command": "write | summarise | draft | improve | translate | page_starter",
  "outcome": "accepted | discarded | partially_accepted",
  "context_sources_used": ["page_content", "db_properties", "linked_pages", "rag_retrieval"],
  "retrieved_page_count": 3,
  "edit_delta_pct": 18.5,
  "sources_disclosure_opened": false,
  "db_properties_populated": false,
  "time_to_outcome_ms": 12400,
  "ts": "ISO8601"
}

// ai_context_fallback
{
  "event": "ai_context_fallback",
  "user_id": "uuid",
  "workspace_id": "uuid",
  "page_id": "uuid",
  "fallback_reason": "injection_timeout | rag_timeout | permission_error",
  "ts": "ISO8601"
}

// ai_page_starter_dismissed
{
  "event": "ai_page_starter_dismissed",
  "user_id": "uuid",
  "workspace_id": "uuid",
  "page_id": "uuid",
  "dismiss_method": "button | escape | typing",
  "ts": "ISO8601"
}

// ai_rag_timeout
{
  "event": "ai_rag_timeout",
  "user_id": "uuid",
  "workspace_id": "uuid",
  "page_id": "uuid",
  "retrieval_duration_ms": 623,
  "fallback_applied": true,
  "ts": "ISO8601"
}
```

---

## 9) Experimentation strategy

### Experiment 1 - RAG context vs. page-only context (primary)

**Hypothesis:** AI responses grounded in workspace RAG context produce a higher acceptance rate than page-only context.

| Parameter | Value |
|---|---|
| Type | A/B (50/50 split) |
| Randomisation unit | User (consistent per user across sessions) |
| Control | Page-only context injection |
| Treatment | Page context + RAG retrieval (top-3 pages) |
| Primary metric | AI response acceptance rate |
| Guardrail | Generation P95 latency must not exceed 5s |
| Min detectable effect | +8pp (25% -> 33%) |
| Estimated runtime | 4 weeks (90% power) |
| Holdout | 10% excluded from both variants for long-run measurement |

### Experiment 2 - Page starter placement

**Hypothesis:** Inline placement (where first block appears) outperforms top-of-page banner in acceptance rate.

| Parameter | Value |
|---|---|
| Type | A/B/n (3 variants) |
| Variants | (A) No starter (control), (B) Top-of-page banner, (C) Inline first-block position |
| Primary metric | Page starter acceptance rate |
| Secondary metric | D7 retention for new users who saw starter |
| Guardrail | Page load P95 must not increase |
| Estimated runtime | 3 weeks |

### Experiment 3 - Sources disclosure as trust signal

**Hypothesis:** Showing which workspace pages were used increases acceptance rate by building trust.

| Parameter | Value |
|---|---|
| Type | A/B |
| Control | No sources disclosure |
| Treatment | Sources disclosure (collapsed by default) |
| Primary metric | AI response acceptance rate |
| Secondary metric | Sources disclosure open rate |
| Estimated runtime | 3 weeks |

---

## 10) Phased rollout plan

### Phase 0 - Internal (week 1-2)
- Deploy to Notion employees only
- Validate context injection latency, RAG retrieval P95, permission model
- Fix any critical bugs; establish baseline acceptance rate on internal dogfood

**Launch gate:** Context injection fallback rate <5%; zero permission leakage in security audit

### Phase 1 - Limited beta (week 3-4)
- Roll out to 5% of AI add-on subscribers (randomised by workspace)
- Run Experiment 1 (RAG vs. page-only) at full split within this cohort
- Monitor guardrail metrics daily

**Launch gate:** AI generation P95 <5s; no increase in workspace error rate; acceptance rate trending positive vs. control

### Phase 2 - Expanded rollout (week 5-8)
- Roll out to 50% of AI add-on subscribers
- Launch page starter (Experiment 2) within this cohort
- Begin Sources disclosure experiment (Experiment 3)

**Launch gate:** Experiment 1 primary metric statistically significant at 95% confidence with positive direction; no guardrail regression

### Phase 3 - Full rollout (week 9+)
- 100% of AI add-on subscribers
- Ship winning variants from all three experiments
- Begin monitoring 90-day churn cohort

**Kill switch:** Feature flag `contextual_ai_enabled` per workspace; RAG can be disabled independently via `rag_retrieval_enabled` flag without disabling page context injection

---

## 11) Dependencies

| Dependency | Team | Status | Risk |
|---|---|---|---|
| Workspace vector index infrastructure | ML Infra | In progress | High - on critical path; no RAG without this |
| Permission-scoped retrieval API | Platform | Planned | High - security requirement; blocks Phase 1 |
| Page block content API (for injection) | Core Editor | Available | Low - existing API, minor additions needed |
| DB property injection schema | Database team | Planned | Medium - needed for Req 1.2 and Req 4 |
| Sources disclosure UI component | Design | In design | Low - UI only; not on critical path |
| AI settings toggle (opt-out) | Settings team | Not started | Medium - needed before Phase 2 |

---

## 12) Risks + mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| RAG injects stale content | Medium | High | Show "last updated" on Sources; alert if index lag >48h; re-index on material edit |
| Permission leakage via RAG | Low | Critical | ACL enforcement at query time; dedicated security test suite; pentest before Phase 1 |
| Retrieval latency degrades P95 beyond 5s | Medium | High | Hard 600ms timeout; fallback to page-only; cache per-user top-K per session |
| AI hallucinates workspace content | Medium | High | Sources disclosure shows grounding; copy: "Based on your workspace pages" |
| Page starter increases blank-page anxiety | Low | Medium | Auto-dismiss after 10s; permanent dismiss per page |
| Index build cost at scale | Low | High | Incremental indexing on edit events; dedup to skip re-embedding identical content |
| Workspace retrieval opt-out confusion | Medium | Medium | Clear toggle in AI settings; educational modal on first RAG response |

---

## 13) Open questions - resolved

| Question | Resolution |
|---|---|
| Opt-in vs. opt-out for workspace retrieval | **Opt-out by default** for existing subscribers; opt-in for new workspaces. Reduces adoption risk while protecting trust for new users. |
| Context window budget split | Current allocation (3,000 / 1,800 / 1,200) is v1 hypothesis. Experiment 1 will validate which layer drives most acceptance rate lift. Revisit at Phase 2 gate. |
| DB property population scope | **Text, date, and select only for v1.** Multi-select and relation are suggestions-only due to incorrect linking risk. |
| Large page truncation strategy | **Truncate from the bottom** (keep intro and most recent content visible to injection). Add `[page truncated]` marker. Revisit with extractive summarisation in v2. |
| Index SLA for high-frequency editors | **24h SLA maintained for v1.** Pages edited in the current session get a session-level cache refresh to reduce staleness for the most active users. |

---

*All metrics are directional estimates based on public information and observable UX patterns, not internal Notion data.*
