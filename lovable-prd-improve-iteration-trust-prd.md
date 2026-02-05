
<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/Lovable.png" 
    alt="Zomato logo" 
    width="200"
  />
</p>

# PRD: Improve Iteration Trust & Retention in Lovable

**Product:** Lovable  
**Author:** Mayank Malviya  
**Status:** Final  
**Audience:** Product, Design, Engineering  

---

## 1. Problem Statement

Lovable enables users to generate functional applications using natural language prompts.  
While first-time usage often feels magical, a significant portion of users fail to
confidently iterate or return to build a second app.

Observed problems:
- Output quality can vary across iterations, reducing confidence
- Failures are often opaque, offering little guidance on recovery
- Users hesitate to invest time after an imperfect first attempt

As a result, Lovable risks being perceived as a **novelty tool**
rather than a **reliable creation platform**.

---

## 2. Target Users

**Primary users**
- Non-technical builders
- Indie hackers and Product Managers
- Early-stage founders exploring ideas

**User mindset**
- High curiosity and experimentation
- Low tolerance for unexplained failure
- Needs reassurance before deeper time investment

---

## 3. Goals & Success Metrics

### Primary Goal
Increase user confidence to iterate and return.

### Success Metrics
- **Primary:** % of users who complete at least one successful iteration after their first app preview  
- **Secondary:** % of users who create a second app within 7 days  

### Guardrail Metrics
- Generation latency
- Failed generation rate
- Support tickets related to confusion or broken outputs

---

## 4. Non-Goals

This initiative explicitly does **not** aim to:
- Improve raw model intelligence
- Support production-grade deployments
- Add advanced developer customization

The focus is on **product experience**, not model capability.

---

## 5. Key User Moments to Improve

1. First app preview  
2. First failed or degraded iteration  
3. Prompt refinement attempt  
4. Decision to start a second app  

These moments disproportionately influence trust and long-term retention.

---

## 6. Proposed Solutions

### 6.1 Iteration Confidence Indicators
- Visually signal the expected impact of a change:
  - Minor refinement
  - Structural change

**Hypothesis:**  
Users iterate more confidently when they understand the risk and scope of changes.

---

### 6.2 Failure Transparency & Recovery
- Clear failure categories (e.g., unsupported pattern, ambiguity)
- Suggested next-step prompts when generation fails

**Hypothesis:**  
Failures with guidance feel recoverable rather than discouraging.

---

### 6.3 Prompt Scaffolding for Common Patterns
- Optional templates for common app types
- Fully editable, not restrictive

**Hypothesis:**  
Reducing prompt ambiguity increases successful iteration rates.

---

### 6.4 Second-App Nudge
- Post-completion prompt encouraging users to try a second app
- Option to reuse structure or patterns from the first app

**Hypothesis:**  
Lowering cognitive reset increases repeat creation.

---

## 7. Risks & Trade-offs

| Risk | Mitigation |
|----|----|
| Reduced flexibility | Keep scaffolds optional |
| UI complexity | Progressive disclosure |
| Overconfidence in system | Clear limitation messaging |

---

## 8. Rollout Plan (High-Level)

1. Internal testing with guided prompts
2. Limited cohort rollout
3. Measure iteration and second-app metrics
4. Gradual expansion based on results

---

## 9. Open Questions

- How much predictability is sufficient without harming creativity?
- When should the system refuse a request versus attempt a low-quality output?
- How can limitations be communicated without discouraging exploration?

---

## 10. PM Summary

For AI-native products like Lovable, retention is driven less by first success
and more by **confidence to continue**.

This PRD prioritizes:
- Trust over novelty
- Recoverability over perfection
- Habit formation over demos
