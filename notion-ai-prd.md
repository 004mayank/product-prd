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
**Status:** v2 - Improved PRD  
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/notion-teardown.md

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, goals/non-goals, target personas, high-level solution pillars, basic requirements, event schemas |
| v2 | Competitive analysis, detailed acceptance criteria with edge cases, full data model, context budget specification, experimentation framework, rollout phases, expanded risk register |

---

## Context

Notion AI launched in 2023 as a paid add-on (~$8/user/month) embedded directly inside the workspace. The core promise was AI-powered writing, summarisation, and drafting without leaving the page.

The product is stuck. Notion AI is a **generic LLM wrapper** inside one of the richest knowledge surfaces in the world - and it ignores that knowledge. It knows nothing about your pages, your team's vocabulary, your project history, or how your workspace is structured. Output is grammatically correct and contextually hollow. Users trial it once, then stop.

This PRD specifies **Contextual Workspace Intelligence**: a system that grounds every Notion AI response in the user's own workspace, making output specific, relevant, and worth keeping.

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

- "Summarise this page" on a meeting notes doc returns a generic summary that misses the three decisions that were actually made - because the AI does not know which items are decisions vs. discussion noise.
- "Write a product spec" returns a five-section template with placeholder text - because the AI does not know the team's spec format, the product area, or any prior specs.
- "Improve this paragraph" returns a rewritten version with different vocabulary - because the AI does not know the author's voice or the audience.

### Competitive pressure making this urgent

| Competitor | AI context capability | Threat level |
|---|---|---|
| Coda AI | Reads the current doc + referenced packs | High - closing gap fast |
| Confluence AI (Atlassian Intelligence) | Pulls from Atlassian knowledge graph (Jira + Confluence) | High for enterprise |
| Google Workspace Duet AI | Reads Drive files; knows your Gmail and Calendar context | Medium - different surface |
| Obsidian + local LLM plugins | Local-first, reads your full vault | Low - niche power users |
| Cursor / GitHub Copilot | Workspace-aware for code; sets expectation for AI context | Indirect but real |

The window to own "workspace-aware AI" in the productivity space is 12-18 months. Notion is uniquely positioned - richer page graph than any competitor - but is not executing on the advantage.

### Why now

- RAG pipelines are production-ready at Notion's scale. The technical blocker is engineering prioritisation, not research.
- The AI add-on renewal cycle is hitting. Users who trialled at launch are now deciding whether to renew. The only argument for renewal is utility - and utility requires context.
- Bottom-up PLG signal: teams that use Notion as their operating system have the richest possible context for AI. These are Notion's highest-value customers. Losing them to Coda or Confluence on AI quality is an existential risk to the platform thesis.

---

## 2) Goals / non-goals

### Goals

1. Increase **Notion AI Weekly Active Users** (add-on subscribers triggering at least one AI action per week) by **40% relative** within 90 days of launch.
2. Increase **AI response acceptance rate** (user keeps AI output with <30% character-level edit) from ~25% estimated baseline to **45%**.
3. Increase **AI sessions per user per week** from ~1.2 to **2.5** among active add-on users.
4. Reduce **AI add-on 90-day churn** by **20% relative**.
5. Achieve **page starter acceptance rate >35%** among new blank pages for AI subscribers.

### Non-goals

- Building a sidebar chat interface ("ask Notion anything"). This is inline, context-aware AI only - not a conversational agent.
- Replacing search or the page graph. Context retrieval improves generation quality; it does not change how users navigate.
- Fine-tuning or training a custom Notion model. All generation uses external models (GPT-4o / Claude 3.5) with RAG-injected context.
- Indexing or surfacing content from pages the requesting user cannot read. Permission model is strictly respected - no exceptions.
- Real-time collaboration features. AI output is single-user; collaborative editing is out of scope.

---

## 3) Target users / personas + JTBD

### Primary personas

| Persona | Workspace behaviour | JTBD | Pain today | Success signal |
|---|---|---|---|---|
| **PM Spec Writer** | Creates 3-5 docs/week; heavy use of linked DB views; references prior specs constantly | "Draft a spec that follows our team format and references the Q2 roadmap" | Generic output; must manually import context from 3-4 other pages | Accepts AI draft with <20% edit; references prior spec correctly |
| **Meeting Notes Taker** | Captures notes for 5-10 meetings/week; uses a consistent template; has a Decisions database | "Summarise this meeting and populate our Decisions database" | Summary ignores the Decisions DB schema; misses what was actually decided | Summary maps to Decisions DB properties; action items are correctly attributed |
| **Knowledge Base Curator** | Maintains 50-200 page wiki; enforces structure; spots stale pages | "Rewrite this SOP to match our current process" | AI doesn't know the current process unless it's pasted manually | AI pulls current process from linked pages; output is structurally consistent |
| **Engineering Lead** | Writes RFCs and ADRs; references past decisions; owns onboarding docs | "Check whether this RFC conflicts with any prior architectural decisions" | AI has no knowledge of prior ADRs; cannot detect conflicts | AI retrieves relevant ADRs and surfaces potential conflicts explicitly |
| **Solo Creator** | Personal workspace with reading lists, journals, project notes | "Write a weekly review based on my notes from this week" | AI doesn't know the user's notes, projects, or prior reviews | Weekly review references actual notes from the current week's pages |

### Activation insight

The aha moment for contextual AI is the **first accepted draft that references real workspace content**. Until that moment, users evaluate Notion AI against ChatGPT. After that moment, they evaluate it against not having it. The product must engineer this moment in the first session.

---

## 4) Insights from teardown

Key findings from Notion teardown v3 that directly shape this PRD:

**1. Loop D breaks at the edit step - the fix is context, not model quality.**
The AI loop (Prompt -> Generate -> Edit -> Done) is structurally broken because the edit burden is too high. Reducing editing time by 50% requires contextual output, not a smarter model.

**2. The linked database view is the aha moment - AI must be DB-aware.**
Users with linked DB views retain at significantly higher rates. AI that can read and write to linked databases (summarise meeting notes into a Decisions DB, populate a task DB from a spec) is the highest-leverage contextual feature.

**3. Entropy at scale is a product gap - contextual AI can address it.**
Large workspaces accumulate stale, duplicate, and inconsistent pages. AI that surfaces the authoritative version, flags staleness, or consolidates duplicates turns a retention risk into a retention driver.

**4. The blank-page cliff is the top activation drop-off - AI starters directly address it.**
Guided page starters using workspace context (not generic templates) convert the blank page from a blocker into a fast start.

**5. AI add-on renewal depends on utility - utility requires context.**
The first trial is curiosity-driven. Renewal is utility-driven. Context is the mechanism that converts novelty into weekly usage.

---

## 5) Solution - three pillars

### Pillar 1: Page Context Injection

Every AI generation is preceded by injecting the current page's full content (blocks, properties, linked page summaries) into the context window. This is the minimum viable context layer - no retrieval required, no latency risk.

**Context injected (in order of priority, subject to token budget):**
1. Current page title and URL path (workspace/section/page)
2. All existing text blocks on the page (truncated at 3,000 tokens for large pages)
3. Database properties if the page is a DB entry (all typed properties as structured JSON)
4. Summaries of pages explicitly `@mentioned` in the current page (up to 3 pages, 400 tokens each)

---

### Pillar 2: Workspace Context Retrieval (RAG)

For higher-value commands (write, draft, summarise, improve), run a semantic search against the workspace index and inject the top-3 most relevant pages as additional context.

**Retrieval pipeline:**
1. Embed the user's prompt + current page title using text-embedding-3-small
2. Query the workspace vector index (scoped to pages the user can read)
3. Return top-3 results by cosine similarity; filter out the current page
4. Summarise each retrieved page to 600 tokens using extractive summarisation
5. Inject as structured context with source attribution

**Token budget allocation:**

| Context layer | Max tokens | Priority |
|---|---|---|
| System prompt + instructions | 800 | Fixed |
| Current page content | 3,000 | Highest |
| DB properties (if applicable) | 400 | High |
| Linked pages via @mention (3 x 400) | 1,200 | Medium |
| RAG retrieved pages (3 x 600) | 1,800 | Medium |
| User prompt | 500 | Fixed |
| Generation budget | 2,000 | Fixed |
| **Total** | **~9,700** | Within GPT-4o 128k window |

---

### Pillar 3: AI-Powered Page Starter

When a user creates a blank page, a non-intrusive contextual prompt offers to generate a structure based on:
- The page title (if entered before opening)
- The parent section (meeting notes, projects, wiki, personal)
- The 3 most recently created pages in the same section (structural pattern matching)

The starter inserts heading blocks (not content) - giving structure without presuming content.

---

## 6) Detailed requirements + acceptance criteria

### Req 1 - Page context injection

| ID | Requirement | Acceptance criteria | Edge case handling |
|---|---|---|---|
| 1.1 | Inject current page title, breadcrumb path, and all text blocks before every generation | AI output references page title in >80% of spot-check test cases; breadcrumb is injected even for untitled pages ("Untitled" as placeholder) | Pages with >500 blocks: truncate to first 3,000 tokens, append "... [page truncated]" marker to context |
| 1.2 | Inject all DB property values when page is a database entry | AI references at least one property value in >70% of DB entry generation test cases | Empty properties injected as null; formula properties injected as computed value, not formula string |
| 1.3 | Inject summarised content from up to 3 `@mentioned` pages | Linked page summaries appear in AI context; user read permission verified at injection time | If user lacks read permission on a linked page, skip that page silently - do not error; do not expose page existence |
| 1.4 | Context injection adds <800ms to generation P95 latency | Measured in production; alert fires if P95 exceeds 1,000ms | On timeout (>1,200ms), fall back to no context injection; log fallback event as `ai_context_fallback` |

### Req 2 - Workspace retrieval (RAG)

| ID | Requirement | Acceptance criteria | Edge case handling |
|---|---|---|---|
| 2.1 | Retrieve top-3 semantically similar pages from workspace for write/draft/summarise commands | Retrieval completes within 500ms at P95; results exclude the current page; all results are readable by the requesting user | If retrieval times out (>600ms), skip RAG and proceed with page-only context; log as `ai_rag_timeout` |
| 2.2 | Inject retrieved pages as extractive summaries (max 600 tokens each) | Total RAG context injection does not exceed 1,800 tokens regardless of page length | Very short pages (<200 tokens): inject full content without summarisation |
| 2.3 | Display "Sources" disclosure below every AI output that used RAG context | Disclosure shows page titles + links; collapsed by default; opens on click; present within 200ms of output render | If retrieval returned 0 results, do not show Sources disclosure |
| 2.4 | Workspace vector index updated within 24h of page creation or material edit (>50 token change) | Index freshness monitored; alert fires if median lag exceeds 24h | Deleted pages: remove from index within 1h of deletion; archived pages: keep in index but flag as archived |
| 2.5 | Retrieval scoped strictly to pages the requesting user can read | Zero permission leakage verified in dedicated security test suite; tested with guest users, cross-space members, and private pages | Guest users: retrieval scoped to the specific pages they have been granted access to only |
| 2.6 | User can disable workspace retrieval from AI settings | Toggle is off-by-default for new workspaces; on-by-default for existing subscribers on migration | Per-workspace toggle, not per-user; workspace owner controls the default |

### Req 3 - AI page starter

| ID | Requirement | Acceptance criteria | Edge case handling |
|---|---|---|---|
| 3.1 | Show contextual "Start with AI" prompt on blank page within 1.5s of page load | Prompt appears for AI subscribers on pages with 0 blocks; does not appear on pages with >0 blocks | If page title is empty: show generic prompt "What will this page be about?"; do not attempt structure suggestion until title is entered |
| 3.2 | Generate structure suggestion based on page title + parent section + recent sibling pages | Suggested headings are contextually relevant to page title in >75% of internal qualitative ratings; minimum 3 headings, maximum 7 | If parent section has <3 sibling pages: use title-only inference for structure suggestion |
| 3.3 | Accept inserts heading blocks; dismiss removes prompt permanently for that page; ignore fades after 10s | All three states are correctly handled; dismiss persisted in page metadata as `ai_starter_dismissed: true` | If user starts typing before prompt fades: dismiss prompt automatically; do not interrupt typing |
| 3.4 | Page starter shown maximum once per page; only to AI add-on subscribers | Re-shown if user manually triggers it via `/AI start`; not shown to free or Plus users without AI add-on | If AI add-on subscription lapses: suppress prompt; do not error or show paywall on the page itself |

### Req 4 - Context-aware DB population

| ID | Requirement | Acceptance criteria | Edge case handling |
|---|---|---|---|
| 4.1 | When AI generates content on a DB entry page, offer to populate empty DB properties with extracted values | "Fill properties" button appears after generation if >2 DB properties are empty and AI output contains candidate values | If DB properties are all filled: do not show "Fill properties" button |
| 4.2 | AI extracts property values from generated content and maps them to correct property types | Extraction accuracy >80% for text, date, and select properties in internal testing | For multi-select and relation properties: show extracted values as suggestions; do not auto-populate without confirmation |

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
    "context_token_total": "int"
  },
  "model": "gpt-4o | claude-3-5-sonnet",
  "status": "pending | generating | completed | failed | fallback",
  "created_at": "ISO8601",
  "completed_at": "ISO8601"
}
```

### `ai_generation_outcome`

```json
{
  "outcome_id": "uuid",
  "request_id": "uuid",
  "user_id": "uuid",
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
  "embedding_vector": "[float]",
  "content_hash": "string",
  "token_count": "int",
  "is_archived": "bool",
  "is_deleted": "bool",
  "last_indexed_at": "ISO8601",
  "page_updated_at": "ISO8601"
}
```

---

## 8) Success metrics + instrumentation

### North Star
**Notion AI Weekly Active Users** - add-on subscribers who trigger at least one AI action per week.

### Input metrics (leading, owned by this team)

| Metric | Baseline (est.) | 90-day target | Event |
|---|---|---|---|
| AI response acceptance rate | ~25% | 45% | `ai_generation_outcome.outcome = accepted` |
| AI sessions per user per week | ~1.2 | 2.5 | `ai_session_started` count per user per week |
| Page starter acceptance rate | n/a | >35% | `ai_page_starter_accepted` |
| RAG context used rate | n/a | >60% of write/draft/summarise commands | `ai_generation_request.retrieved_page_ids.length > 0` |
| Sources disclosure opened rate | n/a | >20% of RAG-enabled responses | `ai_sources_disclosure_opened` |
| DB property population acceptance | n/a | >25% of DB entry generations | `ai_generation_outcome.db_properties_populated = true` |

### Guardrail metrics (must not regress)

| Metric | Constraint | Owner |
|---|---|---|
| AI generation P95 latency | <5s end-to-end | Platform |
| RAG retrieval P95 latency | <500ms | ML Infra |
| Context injection fallback rate | <5% of sessions | Platform |
| Permission leakage incidents | Zero | Security |
| AI add-on 30-day trial conversion | Must not decrease vs. pre-launch baseline | Growth |
| Page load P95 (pages with AI blocks) | Must not increase >200ms vs. baseline | Web Perf |

### Instrumentation events

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
```

---

## 9) Experimentation strategy

### Experiment 1 - RAG context vs. page-only context (primary)

**Hypothesis:** AI responses grounded in workspace RAG context produce a higher acceptance rate than page-only context.

| Parameter | Value |
|---|---|
| Type | A/B (50/50 split) |
| Randomisation unit | User (not session - must be consistent per user) |
| Control | Page-only context injection |
| Treatment | Page context + RAG retrieval (top-3 pages) |
| Primary metric | AI response acceptance rate |
| Guardrail | Generation P95 latency must not exceed 5s |
| Minimum detectable effect | +8 percentage points (25% -> 33%) |
| Estimated runtime | 4 weeks (to reach 90% power at expected traffic) |
| Holdout | 10% holdout excluded from both variants for long-term measurement |

### Experiment 2 - Page starter placement + copy

**Hypothesis:** A page starter prompt positioned inline (where the first block would appear) outperforms a top-of-page banner in acceptance rate.

| Parameter | Value |
|---|---|
| Type | A/B/n (3 variants) |
| Variants | (A) No starter (control), (B) Top-of-page banner, (C) Inline first-block position |
| Primary metric | Page starter acceptance rate |
| Secondary metric | D7 retention for new users who saw starter |
| Guardrail | Page load P95 must not increase |
| Estimated runtime | 3 weeks |

### Experiment 3 - Sources disclosure (trust signal)

**Hypothesis:** Showing sources (which workspace pages were used) increases acceptance rate by building trust in AI output.

| Parameter | Value |
|---|---|
| Type | A/B |
| Control | No sources disclosure |
| Treatment | Sources disclosure (collapsed by default) |
| Primary metric | AI response acceptance rate |
| Secondary metric | Sources disclosure open rate |
| Estimated runtime | 3 weeks |

---

## 10) Risks + mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| RAG injects stale content (page indexed before last edit) | Medium | High | Show "last updated" timestamp on Sources; alert if index lag >48h; re-index on every material edit |
| Permission leakage via RAG - user sees content from inaccessible page | Low | Critical | Retrieval enforces read ACL at query time; dedicated security test suite with cross-permission scenarios; pentest before launch |
| Retrieval latency degrades generation P95 beyond 5s | Medium | High | Hard timeout at 600ms for retrieval; fallback to page-only context on timeout; cache per-user top-K for session |
| AI hallucinates or misattributes workspace content | Medium | High | Sources disclosure shows exactly what was used; copy: "Based on your workspace pages"; do not present AI output as factual about workspace without grounding |
| Page starter increases blank-page anxiety for users who don't want AI | Low | Medium | Auto-dismiss after 10s; permanent dismiss per page; respect "disable AI suggestions" workspace setting |
| Users don't understand opt-out for workspace retrieval | Medium | Medium | Clear toggle in AI settings with plain-language explanation; one-time educational modal on first RAG-powered response |
| Index build cost at scale (millions of pages) | Low | High | Incremental indexing on edit events only; coarse deduplication to skip re-embedding identical content |

---

## 11) Open questions

1. **Opt-in vs. opt-out for workspace retrieval:** Opt-in protects trust but limits adoption; opt-out maximises utility but may create privacy concerns for enterprise admins. Recommendation: opt-out default, with clear disclosure and per-workspace admin control.
2. **Context window budget split:** Current allocation (3,000 page / 1,800 RAG / 1,200 linked) is a hypothesis. Needs empirical testing - which context layer drives the most acceptance rate lift?
3. **DB property population scope:** Auto-populating text and date properties is relatively safe. Auto-populating relation properties (which link to other DB entries) carries higher risk of incorrect linking. Scoping to text/select/date only for v1?
4. **Large page truncation strategy:** Truncate from the end (keep intro), truncate to most recent (keep recent context), or use extractive summarisation? Each trades off differently.
5. **Index update SLA for high-frequency editors:** Power users editing 20+ pages/day create high index churn. Is 24h SLA acceptable for these users, or do we need near-real-time indexing for pages edited in the current session?

---

*All metrics are directional estimates based on public information and observable UX patterns, not internal Notion data.*
