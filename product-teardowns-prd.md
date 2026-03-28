# PRD: Product Teardowns as a Repeatable “Insight Engine” (Improve Consistency, Discoverability, and PRD Conversion)

**Product:** Product Teardowns (GitHub repository)
**Author:** Mayank Malviya
**Status:** v1 - draft
**Repo:** https://github.com/004mayank/product-teardowns

---

## Context
`product-teardowns` is a growing set of teardown documents that explain **how real products work** (user journeys, systems, UX patterns, growth loops, trade-offs). It already creates value for readers, and it also acts as an upstream input into the downstream repo: `product-prd` (turning insights into execution-ready PRDs).

Today, the content quality is strong, but the “product” around the content is still light: it’s not maximally discoverable, not consistently structured, and doesn’t reliably convert first-time readers into repeat readers (or into PRD readers).

This PRD defines a v1 approach to make `product-teardowns` a **repeatable insight engine** that:
- makes every teardown easier to read and reference,
- makes it easier to discover older teardowns,
- increases conversion into `product-prd` (where insights become decisions and plans).

---

## 1) Problem statement + why now
### Problem
1) **Inconsistent structure → harder scanning + reuse**
- Different teardowns have different sectioning depth, naming, and “where to find things.”
- Readers can’t quickly answer: “Where is the architecture? Where are the key loops? What are the risks?”

2) **Low discoverability inside and outside GitHub**
- GitHub repo browsing is not an optimal reading experience.
- There isn’t a strong index that supports browsing by theme (e.g., notifications, messaging, checkout), surface (mobile/web), or “what you’ll learn.”

3) **Weak conversion to the next artifact (PRDs)**
- The repository doesn’t consistently point readers to the matching PRD (when it exists), or explain the “so what” (how to apply the insight).
- There is no explicit path: *read teardown → save/follow → read PRD → discuss/iterate*.

### Why now
- The content set is now large enough that navigation and consistency matter.
- Teardowns are being produced repeatedly; small improvements compound.
- The `product-prd` repo is active; we can create a clear loop from teardown → PRD.

---

## 2) Goals / non-goals
### Goals
1. Make each teardown **easy to skim, deep-dive, and reference**.
2. Make the collection **easy to browse** (by product and by theme).
3. Increase **conversion**:
   - first-time reader → repeat reader (stars/watches)
   - teardown reader → PRD reader (click-through)
4. Reduce authoring friction: new teardown creation should feel **templated and fast**.

### Non-goals
- Building a full publishing platform (payments, auth, subscriptions).
- SEO-optimized long-form blog operations.
- Heavy tooling that increases writing overhead.

---

## 3) Target users / segments + JTBD
### Primary segments
1. **PMs / founders / operators** learning from real products.
2. **Engineers** who want mental models of systems + trade-offs.
3. **Students / career switchers** building product intuition.

### Jobs-to-be-done
- “Help me quickly understand *how this product works end-to-end*.”
- “Help me learn reusable patterns (loops, UX primitives, system components).”
- “Help me apply insights to my own product decisions.”

---

## 4) Success metrics (v1)
### Topline
- **Unique readers** (proxy: GitHub repo traffic views + clones)
- **Retention** (returning visitors; proxy: stars + watchers growth)

### Conversion
- **Teardown → PRD click-through rate**
  - proxy: link clicks if available; otherwise use qualitative + GitHub traffic comparisons on PRD repo.

### Content quality / consistency
- % teardowns that:
  - follow the template
  - include “Key takeaways”
  - include architecture/loops section
  - link to PRD (if exists)

---

## 5) Proposed solution (v1)
### 5.1 A consistent teardown template (default structure)
Add a standard structure to every teardown (with flexibility for product specifics):

1. **TL;DR** (what you’ll learn)
2. **Product surface + primary user journey**
3. **Core loops** (growth / retention / coordination loops)
4. **System / architecture** (high-level components + data flow)
5. **UX mechanics** (ranking, notifications, batching, inventory, trust primitives)
6. **Key trade-offs** (latency vs relevance, safety vs engagement, etc.)
7. **Failure modes** (where it breaks)
8. **Opportunities** (what to improve; ideas)
9. **Key takeaways** (portable lessons)
10. **Related PRD(s)** (links into `product-prd`)

This improves scan-ability and makes cross-teardown comparisons easier.

### 5.2 A strong repo-level index
Create / maintain a single “start here” index that supports two browsing modes:

- **By product** (Slack, Uber, WhatsApp, etc.)
- **By theme/pattern** (notifications, messaging, checkout, search, ranking, trust)

Each index entry should include:
- short 1-line descriptor (“what you’ll learn”)
- links: teardown, PRD (if any), and any diagrams/assets

### 5.3 Explicit “PRD conversion” hooks
For each teardown, add an explicit section:

- **“If you want to go from insight → execution”**
  - link to PRD(s)
  - what problem the PRD targets
  - what’s intentionally out-of-scope

This makes the relationship between repos legible and encourages deeper reading.

### 5.4 Lightweight authoring workflow (reduce friction)
- Add a `TEMPLATE.md` (or `templates/teardown-template.md`) for new teardowns.
- Add a short checklist:
  - hero image present
  - TL;DR present
  - takeaways present
  - PRD link added (if exists)
  - file naming consistent

Optionally (nice-to-have): a small script to scaffold a new teardown file with the template.

---

## 6) Rollout plan
### Phase 1 (v1)
1. Add template + checklist.
2. Add index improvements.
3. Update 3–5 most-read teardowns to match template.
4. Add PRD cross-links for teardowns that already have PRDs.

### Phase 2 (v1.1)
- Backfill remaining teardowns.
- Add theme tags and improve navigation.

---

## 7) Risks / trade-offs / open questions
### Risks
- Over-templating can reduce expressive analysis quality.
- Time spent on structure might reduce time spent on insight.

### Trade-offs
- GitHub-native reading vs a dedicated site: v1 stays GitHub-native.

### Open questions
- Should we standardize image naming and dimensions for consistent rendering?
- Should “by theme” tags live in each markdown file (frontmatter) or only in the index?
- What is the minimal set of analytics proxies we’re comfortable using (GitHub traffic vs external tooling)?
