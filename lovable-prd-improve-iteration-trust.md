# PRD: Improve Iteration Trust & Retention in Lovable

**Product:** Lovable  
**Author:** Mayank Malviya  
**Status:** v1 - Problem framing & solution proposal  
**Audience:** Product, Design, Engineering  

---

## 1. Problem Statement

Lovable enables users to generate functional applications using natural language prompts.  
While first-time usage often feels magical, many users fail to meaningfully iterate or return to build a second app.

Key observed problems:
- Users lose confidence when outputs are inconsistent or degrade unexpectedly
- Failures are often opaque, offering little guidance on recovery
- Users hesitate to invest time after an imperfect first attempt

As a result, Lovable risks being perceived as a **novelty tool** rather than a **reliable creation platform**.

---

## 2. Who Is This For

**Primary users**
- Non-technical builders
- Indie hackers & PMs prototyping ideas
- Early-stage founders exploring concepts

**User mindset**
- High curiosity
- Low tolerance for unexplained failure
- Needs reassurance before deeper investment

---

## 3. Goals & Success Metrics

### Primary Goal
Increase user confidence to iterate and return.

### Success Metrics
- **Primary:** % of users who perform ≥1 successful iteration after first app preview  
- **Secondary:** % of users who create a second app within 7 days  

### Guardrail Metrics
- Generation latency
- Support tickets related to failed generations
- Abandonment after first failed output

---

## 4. Non-Goals (Explicit)

- Improving raw model intelligence
- Supporting production-grade deployments
- Adding advanced developer customization

This PRD focuses on **product experience**, not model capability.

---

## 5. Key User Moments to Improve

1. First app preview  
2. First failed or degraded generation  
3. Prompt refinement attempt  
4. Decision to start a second app  

These moments disproportionately influence trust and retention.

---

## 6. Proposed Solutions

### 6.1 Iteration Confidence Indicators
- Visually differentiate:
  - “Minor refinement”
  - “Structural change”
- Signal expected impact before generation

**Hypothesis:**  
Users iterate more when they understand the risk of change.

---

### 6.2 Failure Transparency & Recovery
- Clear error categories (unsupported pattern, ambiguity, overload)
- Suggested next-step prompts when generation fails

**Hypothesis:**  
Failures with guidance feel recoverable, not discouraging.

---

### 6.3 Prompt Scaffolding for Common App Patterns
- Lightweight templates for:
  - CRUD apps
  - Landing pages
  - Internal tools
- Editable, not restrictive

**Hypothesis:**  
Reducing prompt ambiguity increases successful iteration rates.

---

### 6.4 Second-App Nudge
- Post-completion prompt:
  > “Want to try a second app using what you learned?”

- Offer to reuse structure or patterns

**Hypothesis:**  
Lowering the cognitive reset increases second-app creation.

---

## 7. Risks & Trade-offs

| Risk | Mitigation |
|----|----|
| Reduced flexibility | Keep scaffolds optional |
| UI complexity | Progressive disclosure |
| Over-trusting system | Set clear expectations on limitations |

---

## 8. Rollout Plan (High-Level)

1. Internal testing with guided prompts
2. Limited cohort rollout
3. Measure iteration & second-app metrics
4. Gradual expansion

---

## 9. Open Questions

- How much predictability is “enough” before creativity suffers?
- Should Lovable prioritize confidence over surprise?
- When should the system refuse vs attempt a low-quality generation?

---

## 10. PM Takeaway

For AI-native products like Lovable, retention is driven less by first success and more by **confidence to continue**.

This PRD prioritizes:
- Trust over novelty
- Recoverability over perfection
- Habit over demo appeal
