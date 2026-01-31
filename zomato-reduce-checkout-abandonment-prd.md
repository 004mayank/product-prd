<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/zomato-logo.png" 
    alt="Zomato logo" 
    width="200"
  />
</p>


# PRD: Reduce Checkout Abandonment in Zomato Ordering Flow

**Product:** Zomato - Food Delivery  
**Author:** Mayank Malviya 
**Status:** v1 - Problem framing & solution proposal

---

## 1. Problem Statement

A significant percentage of users abandon the ordering flow during checkout,
after adding items to the cart but before placing the order- https://github.com/004mayank/product-teardowns/blob/main/zomato-search-to-checkout-teardown.md

This drop-off directly impacts:
- Orders per MAU (North Star)
- Revenue
- User trust and repeat behavior

Based on teardown insights, the primary contributors appear to be **late price revelation,
fee shock, and offer confusion**.

---

## 2. Goals & Non-Goals

### Goals
- Reduce cart → order drop-offs
- Improve price expectation clarity
- Increase completed orders per session

### Non-Goals
- Redesigning the entire checkout flow
- Changing platform fee structure
- Optimizing delivery partner logistics

---

## 3. User Personas

### Primary Persona
**Urban frequent food-ordering user**
- Orders 2-5 times per week
- Price-aware but convenience-driven
- Sensitive to unexpected charges

### Secondary Persona
**First-time or low-frequency user**
- Higher trust and price anxiety
- More likely to abandon on friction

---

## 4. User Journey (Scope)

Menu Exploration  
↓  
Add to Cart  
↓  
Cart Review  
↓  
Checkout  
↓  
Order Placed

---

## 5. Key Insights (from Teardown)

- Platform and delivery fees are surfaced late
- Offer application logic is unclear
- Final price often differs from menu price expectations
- Users reassess value only at checkout

---

## 6. Proposed Solutions

### Solution 1: Early Price Transparency
- Show estimated total cost (including fees) in cart
- Display fee breakdown before checkout CTA

### Solution 2: Offer Clarity
- Auto-apply best available offer
- Clearly explain why an offer was applied or not

### Solution 3: Checkout Confidence Nudges
- Reinforce ETA accuracy
- Highlight “no additional charges after this step”

---

## 7. Success Metrics

### Primary Metric
- **Cart → Order Conversion Rate**

### Secondary Metrics
- Orders per MAU
- Checkout abandonment rate
- Average order value (guardrail)

---

## 8. Experimentation Plan

### Experiment 1: Early Fee Disclosure
- Variant A: Current experience
- Variant B: Fee-inclusive cart summary
- Expected impact: Lower abandonment

### Experiment 2: Offer Auto-Application
- Variant A: Manual offer selection
- Variant B: Best offer auto-applied
- Expected impact: Faster checkout completion

---

## 9. Risks & Trade-offs

| Risk | Mitigation |
|----|-----------|
| Lower AOV | Treat AOV as guardrail metric |
| Fee visibility deters users | Test phrasing and framing |
| Increased UI clutter | Progressive disclosure |

---

## 10. Open Questions

- Should fee transparency be shown even earlier (menu page)?
- Do first-time users behave differently than repeat users?
- How does Gold subscription change sensitivity?

---

## 11. Rollout Plan

- 10% experiment cohort
- Monitor for 2 weeks
- Roll out winning variants incrementally

---

## 12. Final Takeaway

Reducing checkout abandonment is less about removing fees
and more about **aligning user expectations earlier in the journey**.

This PRD prioritizes trust and clarity over short-term revenue optimization.

