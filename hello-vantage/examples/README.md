# Vantage — Conversation Examples

**20+ example prompts organized by domain and complexity.**

Each example shows the exact prompt and what Vantage produces. Start anywhere — examples are independent. For a guided walkthrough, see [Hello, Vantage!](../README.md)

---

## Learning Path

| Level | Focus | Time | Examples |
|-------|-------|------|----------|
| 1. Getting Started | First population, first scenario | 5 min | [Basic](#level-1-getting-started) |
| 2. Single Domain | Deep dive into one domain | 10 min | [Risk](#risk-adjustment), [Quality](#quality--stars), [Cost](#medical-cost), [RADV](#radv-audit-risk) |
| 3. Cross-Domain | Levers that cascade | 10 min | [PA Policy](#pa-policy-simulation), [Combined Levers](#combined-levers) |
| 4. Strategic | Multi-scenario, multi-year, reports | 15 min | [Comparison](#scenario-comparison), [Temporal](#multi-year-projection), [Output](#executive-output) |

---

## Level 1: Getting Started

### Generate a Population

```
/population 5,000 members in Cook County, Illinois
```

Generates 5,000 synthetic MA members calibrated to Cook County demographics.

### Run a Basic Scenario

```
/simulate CDM program for hypertension, 65% engagement, $300K investment
```

Routes the CDM lever through all four domains with confidence ranges.

---

## Risk Adjustment

### V28 Transition Exposure

```
What's our V28 transition exposure? Which HCCs are most at risk?
```

Scores population under V24 and V28, identifies top HCCs by revenue exposure.

### Coding Program ROI

```
Model a coding improvement program — IHA visits plus retrospective chart review.
```

Applies risk-domain effect weights with SDOH dual-status modifier.

---

## Quality / Stars

### Stars Rating Projection

```
What's our current Stars rating? Which measures are closest to the next cut point?
```

Evaluates 18 HEDIS measures, maps to Stars levels, identifies gap-to-next-star.

### Gap Closure Strategy

```
Close 60% of diabetes A1C gaps and 40% of breast cancer screening gaps.
```

Re-projects Stars rating and calculates quality bonus impact.

---

## Medical Cost

### TCOC Baseline

```
What's our total cost of care baseline? Show MLR position.
```

Calculates TCOC by cost category and checks 85% MLR compliance.

### MSSP Contract Modeling

```
Model under MSSP Enhanced track with $1,863 PMPM benchmark.
```

Computes savings/loss under Enhanced track rules with quality gate.

---

## RADV Audit Risk

### Audit Risk Assessment

```
Score our RADV audit risk. What's our net risk position?
```

Scores documentation completeness and specificity across member-HCCs.

`Net Risk Position = risk_revenue_delta + radv_exposure_delta`

---

## PA Policy Simulation

### PA Relaxation for ACO Partners

```
Relax PA from 88% to 95% for ACO-attributed members. Show full cascade.
```

Models cross-domain impact with VBC offset for ACO members.

---

## Combined Levers

### The $2M Question

```
We have $2M to invest. Compare CDM + coding + PA options.
```

Runs multi-lever scenarios, ranks across 7 dimensions, recommends.

---

## Multi-Year Projection

```
Project CDM program forward 3 years with $500K/year investment.
```

Applies CMS rate trends, engagement ramp, population aging, Stars drift. Shows NPV.

---

## Executive Output

```
/report HTML dashboard
/report PPTX deck for the board
```

Self-contained HTML with KPIs and methodology, or board-ready PowerPoint.

---

## Quick Reference

| Category | Example |
|----------|--------|
| Population | `/population 10K in LA County` |
| Risk | `What's our V28 exposure?` |
| Quality | `What's our Stars rating?` |
| Cost | `What's our TCOC baseline?` |
| RADV | `Score our RADV audit risk` |
| PA | `Relax PA from 88% to 95%` |
| Compare | `/compare CDM vs coding vs PA` |
| Projection | `Project forward 3 years` |
| Reports | `/report HTML dashboard` |

---

*Vantage generates synthetic healthcare data for strategic modeling only. No real patient data is used, stored, or transmitted.*
