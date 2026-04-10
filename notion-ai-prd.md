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
**Status:** v1 - Initial PRD  
**Source teardown:** https://github.com/004mayank/product-teardowns/blob/main/notion-teardown.md

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Problem statement, goals/non-goals, target personas, high-level solution pillars |

---

## Context

Notion AI launched in 2023 as a paid add-on (~$8/user/month) that embeds generative AI directly inside the Notion workspace. The core promise: write, summarise, translate, and brainstorm without leaving the page.

The problem is that current Notion AI is a **generic LLM wrapper** inside a rich workspace. It knows nothing about your pages, your team's language, your project history, or your writing style. The output is grammatically correct but contextually hollow - users accept it once, then stop using it.

This PRD is scoped to **making Notion AI contextual**: grounding every AI response in the user's own workspace content so that output is genuinely useful rather than merely passable.

---

## 1) Problem statement + why now

### Problem

Notion AI's core loop breaks at the editing step:

```
Prompt -> Generate -> [Edit heavily] -> Done
```

Users report that AI output requires more editing time than writing from scratch. The root cause is not model quality - it is **lack of context**. The AI does not know:
- What this page is about or who will read it
- What related pages and projects already exist in the workspace
- The team's preferred terminology, tone, or structure
- Prior decisions made in linked databases or meeting notes

**Observable failure mode:** "I asked AI to write a product spec. It wrote a generic template. I had to rewrite everything anyway."

**Observable failure mode 2:** "I used AI Summarise on a meeting notes page. It missed the three decisions that actually mattered because it didn't know what decisions we were tracking."

### Why now

- Notion AI add-on has hit a **retention wall**: users who trial it rarely convert to sustained weekly usage. Add-on churn is the most visible signal.
- RAG (retrieval-augmented generation) is now production-ready at Notion's scale. The technical blocker is engineering prioritisation, not research.
- Competitors are moving fast: Coda AI has workspace context, Confluence AI is pulling from the Atlassian knowledge graph. Notion risks falling behind on the one dimension it is uniquely positioned to win - **your workspace as AI context**.
- The AI add-on is a meaningful revenue line (~$8/user/month). Improving retention from trial to sustained usage is high-leverage on ARPU.

---

## 2) Goals / non-goals

### Goals

1. Increase **AI weekly active users** among add-on subscribers by 40% relative within 90 days of launch.
2. Increase **AI response acceptance rate** (user keeps AI output with <30% edit) from ~25% to ~45%.
3. Increase **AI sessions per user per week** from ~1.2 to ~2.5 among active add-on users.
4. Reduce **AI add-on 90-day churn** by 20% relative.

### Non-goals

- Building a chat interface (ask Notion anything). This PRD is about inline, contextual AI - not a sidebar chatbot.
- Replacing search. AI context is used to improve generation quality, not to surface documents.
- Training a custom Notion model. This uses RAG over existing models (GPT-4o / Claude), not fine-tuning.
- Indexing private pages the user does not have access to. Permissions are strictly respected.

---

## 3) Target users / personas + JTBD

### Primary personas

| Persona | Context | JTBD | AI friction today |
|---|---|---|---|
| **The PM Spec Writer** | Writes specs, roadmaps, stakeholder updates weekly | "Write me a first draft that matches our team's format and references our existing roadmap" | AI writes a generic spec; PM rewrites 80% |
| **The Meeting Notes Taker** | Captures notes in Notion after every meeting | "Summarise this meeting and extract the decisions and action items" | Summary is complete but misses which items are actually decisions vs. discussion |
| **The Knowledge Base Curator** | Maintains team wiki, SOPs, onboarding docs | "Update this SOP to reflect the new process we discussed" | AI edits the SOP without knowing the new process unless manually pasted |
| **The Solo Creator** | Personal workspace: reading lists, journaling, research | "Write a reflection on this article based on my notes" | AI doesn't know the user's notes or reading history |

### Secondary persona

- **The Engineering Lead** writing RFCs and ADRs who wants AI to check consistency with prior architectural decisions.

---

## 4) Insights from teardown

From the Notion teardown v3, the key findings that directly inform this PRD:

1. **Loop D breaks at editing:** The AI loop's failure point is output quality relative to editing cost. Fix the loop by reducing editing cost - which means improving contextual relevance of output.

2. **The aha moment is the linked database view:** Users who create a linked DB view have dramatically better retention. AI should be aware of and able to reference linked databases - not just the current page text.

3. **Entropy at scale is a real problem:** Large workspaces have stale, duplicate, and inconsistent pages. Contextual AI that knows the workspace can help surface the authoritative version and reduce entropy.

4. **AI add-on is an upsell that must earn its keep at renewal:** The first AI trial is driven by curiosity. Renewal is driven by whether the AI saved meaningful time. Context is the mechanism that converts novelty into utility.

5. **Blank-page problem is the top activation blocker:** AI-powered page starters (using context from similar pages the user has written) are a high-leverage solution.

---

## 5) Solution overview - three pillars

### Pillar 1: Page Context Awareness

Before generating any response, the AI reads the current page's content, metadata (database properties if the page is a DB entry), and any explicitly linked pages via `@mention`.

**What changes:** AI output references the actual page - its title, existing sections, properties, linked pages - rather than generating from scratch.

**Example:** User asks AI to "write an executive summary" on a page titled "Q3 Roadmap - Mobile". AI reads the existing roadmap content and writes a summary that references the actual initiatives, not a generic template.

---

### Pillar 2: Workspace Context Retrieval (RAG)

For higher-value generation tasks (spec writing, summarisation, drafting), the AI retrieves the top-K most relevant pages from the workspace via semantic search and injects them as context.

**What changes:** AI knows about your team's prior decisions, existing docs, and project history before generating.

**Example:** User asks AI to "draft an onboarding guide for new engineers." AI retrieves the existing engineering handbook, the team norms page, and the last onboarding doc - and writes a guide that is consistent with them, not a generic template.

**Scope:** Only pages the requesting user has read access to. No cross-permission leakage.

---

### Pillar 3: AI-Powered Page Starter

When a user creates a blank page, a contextual prompt appears: "Start with AI?" AI suggests a structure based on:
- The page's parent (what section of the workspace it lives in)
- The page's title (if entered)
- Similar pages the user has created before

**What changes:** Reduces blank-page paralysis (the top activation drop-off cliff identified in teardown).

**Example:** User creates a new page inside the "Meeting Notes" section titled "Design Review - Apr 10". AI suggests a structure matching prior meeting notes in that section: attendees, agenda, decisions, action items.

---

## 6) Detailed requirements (v1 scope)

### Req 1 - Page context injection

| # | Requirement | Acceptance criteria |
|---|---|---|
| 1.1 | Before every AI generation, inject current page title, all existing text blocks, and database properties into the context window | AI output references page title and at least one existing content element in >80% of test cases |
| 1.2 | If page is a database entry, inject property values (status, owner, date, tags) as structured context | AI output for DB entry pages references at least one property in >70% of test cases |
| 1.3 | Inject content from pages explicitly `@mentioned` on the current page (up to 3 linked pages, truncated to 500 tokens each) | Linked page content is visible in AI context; user permission is checked before injection |
| 1.4 | Context injection must add <800ms to generation latency at P95 | Measured in production via latency instrumentation |

### Req 2 - Workspace retrieval (RAG)

| # | Requirement | Acceptance criteria |
|---|---|---|
| 2.1 | For generation commands (write, draft, summarise), retrieve top-3 semantically similar pages from the workspace | Retrieval runs within 500ms at P95; results are scoped to pages the user can read |
| 2.2 | Retrieved pages are injected as summarised context (max 800 tokens per retrieved page) | Total context window addition from retrieval does not exceed 2,400 tokens |
| 2.3 | User can see which pages were used as context via a "Sources" disclosure below AI output | Disclosure is collapsed by default; expandable; each source links to the source page |
| 2.4 | Retrieval index is updated within 24 hours of a page being created or materially edited | Index freshness SLA: 24h for new/edited pages |
| 2.5 | Retrieval respects workspace permissions strictly: never inject content from pages the requesting user cannot read | Zero permission leakage in security audit; tested with cross-space and guest-restricted pages |

### Req 3 - AI page starter

| # | Requirement | Acceptance criteria |
|---|---|---|
| 3.1 | When a new blank page is created, show an "AI can help you start this" prompt within 1.5s of page load | Prompt appears for blank pages only; does not appear if page already has >1 block |
| 3.2 | AI suggests a structure (list of section headings) based on page title and parent section | Structure suggestion is relevant to the title in >75% of user ratings in qualitative test |
| 3.3 | User can accept (inserts suggested structure as heading blocks), dismiss (removes prompt permanently for this page), or ignore (prompt fades after 10s) | All three states work correctly; dismiss is persisted per page |
| 3.4 | Page starter is available to all Notion AI subscribers; shown max once per new page | Not shown to non-AI users; not re-shown after dismiss or accept |

---

## 7) Success metrics + instrumentation plan

### North Star
**Notion AI Weekly Active Users (add-on subscribers who trigger at least one AI action per week)**

### Input metrics

| Metric | Baseline (est.) | Target (90 days post-launch) | Instrumentation event |
|---|---|---|---|
| AI response acceptance rate | ~25% | 45% | `ai_response_accepted` / `ai_response_discarded` |
| AI sessions per user per week | ~1.2 | 2.5 | `ai_session_started`, `ai_session_ended` |
| Page starter acceptance rate | n/a (new) | >35% | `ai_page_starter_accepted` / `ai_page_starter_dismissed` |
| Sources disclosure opened | n/a (new) | >20% of AI responses | `ai_sources_disclosure_opened` |

### Guardrail metrics

| Metric | Constraint |
|---|---|
| AI generation P95 latency | Must stay <5s (contextual injection adds latency; must not degrade UX) |
| Permission leakage incidents | Zero tolerance |
| AI add-on 30-day trial conversion | Must not decrease (contextual AI should improve, not hurt, conversion) |
| Page load P95 for pages with AI blocks | Must not increase by >200ms vs. baseline |

### Event schemas

```json
// ai_response_accepted
{
  "event": "ai_response_accepted",
  "user_id": "uuid",
  "workspace_id": "uuid",
  "page_id": "uuid",
  "ai_command": "write | summarise | draft | translate | improve",
  "context_sources": ["page_content", "linked_pages", "workspace_retrieval"],
  "retrieved_page_count": 3,
  "edit_delta_pct": 15,
  "ts": "ISO8601"
}

// ai_page_starter_accepted
{
  "event": "ai_page_starter_accepted",
  "user_id": "uuid",
  "workspace_id": "uuid",
  "page_id": "uuid",
  "parent_section_type": "meeting_notes | project | wiki | personal",
  "suggestion_heading_count": 5,
  "ts": "ISO8601"
}

// ai_sources_disclosure_opened
{
  "event": "ai_sources_disclosure_opened",
  "user_id": "uuid",
  "page_id": "uuid",
  "source_page_count": 3,
  "ts": "ISO8601"
}
```

---

## 8) Risks + mitigations (v1)

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Context injection causes hallucination about workspace content | Medium | High | RAG results must be clearly attributed; Sources disclosure shows exactly what was used; add disclaimer copy |
| Retrieval latency degrades generation P95 beyond 5s | Medium | High | Cache top-K retrieval results per user per session; set hard timeout of 400ms for retrieval; fall back to page-only context on timeout |
| Permission leakage via RAG | Low | Critical | Retrieval is scoped at query time to pages the requesting user can read; verified in security audit before launch |
| Users don't notice Sources disclosure / don't trust AI | Medium | Medium | Run qualitative usability test on disclosure design before launch; consider "AI used your workspace" prominent callout |
| Page starter feels intrusive and increases blank-page anxiety | Low | Medium | Prompt auto-fades after 10s; dismiss is permanent for that page; A/B test prompt copy and timing |

---

## 9) Open questions

1. Should workspace retrieval be opt-in or opt-out? Opt-in reduces adoption but increases trust; opt-out is more powerful but may alarm privacy-conscious users.
2. What is the right context window budget split between current page, linked pages, and retrieved pages?
3. Should the Sources disclosure be shown by default (expanded) or collapsed? Collapsed reduces UI noise but may reduce trust.
4. How do we handle very large pages (>500 blocks) in context injection - truncate, summarise, or chunk?
5. For the page starter, should AI suggest full section structure or just the first heading to reduce overwhelm?

---

*All metrics are directional estimates based on public information and observable UX patterns, not internal Notion data.*
