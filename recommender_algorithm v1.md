# InsuranceDekho Life Insurance Recommender — Algorithm Documentation

**Version:** v5  
**Type:** Rule-based weighted scoring engine (no ML, fully client-side JavaScript)  
**Plans in database:** 81 retail life insurance plans across 15 insurers

---

## Overview

The engine takes 8 inputs from the agent, runs every plan through a two-stage pipeline (hard filters → weighted scoring), ranks survivors by total score, and surfaces the top 3.

```
8 Agent Inputs
      │
      ▼
┌─────────────────┐
│  Stage 1        │  3 hard filters — plan either passes or is eliminated entirely
│  Hard Filters   │
└────────┬────────┘
         │ surviving plans only
         ▼
┌─────────────────┐
│  Stage 2        │  5 dimensions scored 0–3 each, multiplied by weight, summed
│  Weighted Score │
└────────┬────────┘
         │
         ▼
   Sort descending → Top 3 shown
```

---

## Stage 0 — Inputs Collected

| Input | Field | Values |
|---|---|---|
| Client name | `name` | Free text (personalisation only, not scored) |
| Age | `age` | Bracket midpoints: 25, 35, 45, 55, 65 |
| Annual income | `income` | ₹400K / ₹750K / ₹1.25M / ₹2M / ₹3.75M / ₹7.5M |
| Primary goal | `goal` | `protection` / `wealth` / `retirement` / `child` |
| Risk appetite | `risk` | `conservative` / `moderate` / `aggressive` |
| Desired sum assured | `sa` | ₹5L to ₹5Cr+ (10 options) |
| Annual premium budget | `prem` | Up to ₹6K to ₹5L+ (11 options) |
| Preferred tenure | `tenure` | Bracket midpoints: 7, 15, 25, 35 years |

---

## Stage 1 — Hard Filters (Elimination)

These run before any scoring. A plan that fails any hard filter is removed entirely — it gets no score and will never appear in results.

### Filter 1: Age Eligibility

```
if (client_age < plan.min_age) → ELIMINATE
if (client_age > plan.max_age) → ELIMINATE
```

Every plan in the database has a `min_age` and `max_age`. These come from IRDAI-approved policy eligibility terms. For example, most pension plans require age ≥ 30; some guaranteed savings plans have a max entry age of 55.

**Special rule for pension plans:**
```
if (plan.type === 'pension' AND client_age < 30) → ELIMINATE
```
Pension plans have a hard floor of 30 regardless of what the individual plan's `min_age` states, because annuity structures are not suitable for younger clients.

### Filter 2: Sum Assured Feasibility

```
if (desired_SA > plan.max_sa) → ELIMINATE
```

If the client wants ₹3 Cr cover but a plan's maximum sum assured is ₹1 Cr, it is eliminated. There is no lower SA filter — if the client wants less cover than the plan's minimum, the scoring engine penalises this through the SA Adequacy dimension (see Stage 2) rather than eliminating the plan, since the agent can still recommend a higher SA.

### Filter 3: (Implicit) Plan Pool

Group insurance plans, employer-benefit plans, and micro-insurance plans were excluded at the database curation stage. All 81 plans in the database are retail individual plans only, so no runtime filter is needed for this.

---

## Stage 2 — Weighted Scoring

Every plan that survives the hard filters is scored across 5 dimensions. Each dimension produces a raw score of **0, 1, 2, or 3**. The raw score is multiplied by the dimension's weight to produce a weighted score. All 5 weighted scores are summed to produce the plan's **total score**.

### Scoring Formula

```
Total = (gf × 1.3) + (aff × 1.2) + (sa × 1.0) + (ten × 0.8) + (rf × 0.7)

Maximum possible total = 3 × (1.3 + 1.2 + 1.0 + 0.8 + 0.7) = 3 × 5.0 = 15.0
```

### Weight Rationale

| Dimension | Weight | Why this weight |
|---|---|---|
| Goal fit | 1.3 | Highest — recommending a pension plan to someone who wants pure protection is a fundamental mismatch that no other factor can compensate for |
| Affordability | 1.2 | Second highest — an out-of-budget plan is a null recommendation regardless of how well it fits the goal |
| SA adequacy | 1.0 | Baseline — important but partially controlled by the client's own SA choice |
| Tenure fit | 0.8 | Differentiator within an already viable shortlist |
| Risk fit | 0.7 | Lowest — risk appetite is important but many clients are flexible once they understand the product |

---

### Dimension 1: Goal Fit (`gf`) — Weight 1.3

**Source:** Pre-assigned in plan database as `goal_fit: { protection, wealth, retirement, child }` — each value is 0, 1, 2, or 3.

The goal fit scores for each plan type follow this matrix:

| Plan Type | Protection | Wealth | Retirement | Child |
|---|---|---|---|---|
| Term | 3 | 0 | 0 | 1 |
| Guaranteed/Endowment | 1 | 3 | 2 | 2 |
| ULIP | 1 | 3 | 2 | 2 |
| Pension/Annuity | 0 | 1 | 3 | 0 |
| Child Plan | 1 | 2 | 0 | 3 |
| Par Endowment | 1 | 3 | 2 | 2 |

**Score lookup:**
```
gf = plan.goal_fit[client_goal]
// e.g. if client_goal = 'protection' and plan is a Term plan → gf = 3
// e.g. if client_goal = 'protection' and plan is a Pension plan → gf = 0
```

---

### Dimension 2: Affordability (`aff`) — Weight 1.2

**Logic:** Two conditions must both be true for any score above 0:

1. The client's stated premium budget must meet or exceed the plan's minimum annual premium (`plan.annual_premium_approx[0]`)
2. The premium-to-income ratio must be within acceptable bounds

```
if (client_prem < plan.minimum_premium) → aff = 0  // can't afford entry point

else:
  ratio = client_prem / client_income
  if (ratio ≤ 0.08)  → aff = 3  // premium ≤ 8% of income — comfortable
  if (ratio ≤ 0.15)  → aff = 2  // premium 8–15% of income — manageable
  if (ratio ≤ 0.25)  → aff = 1  // premium 15–25% of income — stretched
  if (ratio > 0.25)  → aff = 0  // premium > 25% of income — unaffordable
```

**Why this approach:** The 8/15/25% thresholds are standard financial planning benchmarks for insurance premium burden. A household spending more than 25% of income on a single insurance policy is financially over-exposed regardless of plan quality.

---

### Dimension 3: SA Adequacy (`sa`) — Weight 1.0

**Logic:** Uses the Human Life Value (HLV) method — the ratio of desired sum assured to annual income.

```
hlv = desired_SA / annual_income

if (hlv ≥ 15)  → sa = 3  // ≥15× income — ideal
if (hlv ≥ 10)  → sa = 2  // 10–15× income — adequate
if (hlv ≥ 5)   → sa = 1  // 5–10× income — partial
if (hlv < 5)   → sa = 0  // < 5× income — insufficient
```

**Special overrides:**
```
if (plan.type === 'pension') → sa = 2  // forced to 2 — pension plans are corpus-based,
                                        // SA metric is less relevant
if (plan.type === 'child')   → sa = max(computed_sa, 1)  // minimum 1 — child plans
                                                           // always have some SA relevance
```

**Why HLV:** HLV (Human Life Value) is the standard actuarial approach used by IRDAI and financial planners to determine adequate life coverage. The rule of thumb of 10–15× annual income accounts for income replacement, outstanding liabilities, and inflation over a policy term.

---

### Dimension 4: Tenure Fit (`ten`) — Weight 0.8

**Logic:** Measures how closely the client's preferred tenure falls within the plan's allowed tenure range.

```
[tMin, tMax] = plan.tenure_range  // e.g. [10, 40] for a term plan

if (preferred_tenure is within [tMin, tMax]):
  tenDiff = 0
else:
  tenDiff = min(|preferred_tenure − tMin|, |preferred_tenure − tMax|)
  // distance to nearest edge of the plan's valid range

if (tenDiff ≤ 5)   → ten = 3
if (tenDiff ≤ 10)  → ten = 2
if (tenDiff ≤ 15)  → ten = 1
if (tenDiff > 15)  → ten = 0
```

**Example:**  
Client wants 20-year tenure. Plan allows 5–40 years. `tenDiff = 0` → `ten = 3`.  
Client wants 7-year tenure. Plan allows 10–30 years. `tenDiff = |7−10| = 3` → `ten = 3`.  
Client wants 5-year tenure. Plan allows 20–40 years. `tenDiff = |5−20| = 15` → `ten = 1`.

---

### Dimension 5: Risk Fit (`rf`) — Weight 0.7

**Source:** Pre-assigned in plan database as `risk_fit: { conservative, moderate, aggressive }` — each value is 0, 1, 2, or 3.

Default risk fit assignment by plan type:

| Plan Type | Conservative | Moderate | Aggressive |
|---|---|---|---|
| Term | 3 | 3 | 2 |
| Guaranteed/Endowment | 3 | 2 | 0 |
| ULIP | 0 | 2 | 3 |
| Pension | 3 | 2 | 0 |
| Par Endowment | 2 | 3 | 1 |
| Child (ULIP-based) | 1 | 3 | 2 |
| Child (Guaranteed) | 3 | 2 | 0 |

**Score lookup:**
```
rf = plan.risk_fit[client_risk]
// e.g. conservative client + ULIP plan → rf = 0
// e.g. aggressive client + Guaranteed plan → rf = 0
```

---

## Stage 3 — Ranking and Output

```javascript
// 1. Score all 81 plans
const results = PLANS
  .map(plan => scorePlan(plan, clientProfile))  // returns null if hard-filtered
  .filter(Boolean)                               // remove null (eliminated) plans
  .sort((a, b) => b.total - a.total)            // descending by total score
  .slice(0, 3);                                  // top 3 only
```

The top 3 plans by total score are presented as swipeable cards. Ties are broken by the order plans appear in the database (which reflects their historical booking volume rank — higher-volume plans appear earlier).

---

## Check Tags (UI Logic)

Each result card shows green checkmark tags — these are not part of scoring, they are a UI-level summary generated after scoring:

```
if (gf ≥ 2)  → show "Goal match"
if (aff ≥ 2) → show "Within budget"
if (rf ≥ 2)  → show "Risk match"
if (ten ≥ 2) → show "Tenure match"
```

Threshold is ≥ 2 (not 3) so that partial matches still surface as positive signals rather than being silent.

---

## Plan Database Schema

Each of the 81 plans is stored as a JSON object with the following structure:

```json
{
  "id": "hdfc_sanchay_plus",
  "name": "Sanchay Plus",
  "insurer": "HDFC Life",
  "type": "guaranteed",

  "goal_fit": {
    "protection": 1,
    "wealth": 3,
    "retirement": 2,
    "child": 2
  },

  "risk_fit": {
    "conservative": 3,
    "moderate": 2,
    "aggressive": 0
  },

  "min_age": 30,
  "max_age": 60,
  "min_sa": 200000,
  "max_sa": 50000000,
  "tenure_range": [10, 45],
  "annual_premium_approx": [25000, 500000],
  "smoker_allowed": true,

  "key_features": [
    "Guaranteed returns",
    "Multiple payout options",
    "Whole life income option",
    "Immediate income variant"
  ]
}
```

**Fields used in scoring:**
- `goal_fit` → Dimension 1 (goal fit)
- `annual_premium_approx[0]` → Dimension 2 (affordability floor check)
- `max_sa` → Hard Filter 2 (SA feasibility)
- `min_age`, `max_age` → Hard Filter 1 (age eligibility)
- `tenure_range` → Dimension 4 (tenure fit)
- `risk_fit` → Dimension 5 (risk fit)
- `type` → Used in SA adequacy overrides (Dimension 3) and IRR display

**Fields used in UI only (not scored):**
- `name`, `insurer` → Display
- `key_features` → Feature chips on result card
- `min_sa` → Not currently used in scoring (only `max_sa` is filtered)
- `smoker_allowed` → Collected in DB but smoker/non-smoker filtering not yet activated

---

## Worked Example

**Client profile:** Age 35, Income ₹15L, Goal: Protection, Risk: Conservative, SA: ₹1 Cr, Premium: ₹25K/yr, Tenure: 20 yrs

**Plan being scored:** ICICI iProtect Smart (Term plan)  
- `min_age`: 18, `max_age`: 65, `max_sa`: ₹10 Cr, `tenure_range`: [5, 40]  
- `goal_fit.protection` = 3, `risk_fit.conservative` = 3  
- `annual_premium_approx` = [8000, 50000]

**Hard filters:**
- Age: 35 is within [18, 65] → PASS
- SA: ₹1 Cr < ₹10 Cr max → PASS

**Scoring:**

| Dimension | Calculation | Raw Score |
|---|---|---|
| Goal fit | `plan.goal_fit['protection']` = 3 | 3 |
| Affordability | ₹25K ≥ ₹8K min ✓; ratio = 25000/1500000 = 1.67% ≤ 8% | 3 |
| SA adequacy | HLV = 10000000/1500000 = 6.67× → ≥5 but <10 | 1 |
| Tenure fit | 20 yrs within [5, 40] → tenDiff = 0 | 3 |
| Risk fit | `plan.risk_fit['conservative']` = 3 | 3 |

**Total:**
```
(3 × 1.3) + (3 × 1.2) + (1 × 1.0) + (3 × 0.8) + (3 × 0.7)
= 3.9 + 3.6 + 1.0 + 2.4 + 2.1
= 13.0 / 15.0
```

**Check tags shown:** Goal match ✓, Within budget ✓, Risk match ✓, Tenure match ✓  
*(SA adequacy scored 1, below threshold of 2, so no "SA adequate" tag)*

---

## Known Limitations and Improvement Areas

| Limitation | Impact | Suggested fix |
|---|---|---|
| Premium ranges are approximate | Affordability dimension is indicative, not exact | Integrate live premium API per plan |
| Smoker status collected but not filtered | Smokers may see non-smoker plans | Add smoker pool split as a hard filter |
| No existing cover gap logic | Client with ₹50L existing cover asking for ₹1 Cr gets same score as one with zero cover | Add `net_sa = desired_sa − existing_cover` and use that in SA adequacy |
| `min_sa` not enforced | Agent could select ₹5L SA for a plan requiring ₹25L minimum | Add lower SA hard filter |
| Goal fit scores are manually assigned | Subjective at the margins | Validate against actual policy sales conversion data |
| Tie-breaking by DB order | Plans appearing earlier in the database (higher historical volume) win ties | Consider adding an explicit `insurer_quality_score` as a tiebreaker |
| No recency/availability check | A plan might have been withdrawn or modified | Connect to InsuranceDekho product master for live status |
