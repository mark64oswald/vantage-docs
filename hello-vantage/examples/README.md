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

**Prompt:**
```
/population 5,000 members in Cook County, Illinois
```

**What Vantage does:** Generates 5,000 synthetic MA members calibrated to Cook County demographics (Census ACS + CMS county data). Assigns HCC conditions at published prevalence rates, generates ~6,000 PA events.

---

### Run a Basic Scenario

**Prompt:**
```
/simulate CDM program for hypertension, 65% engagement, $300K investment
```

**What Vantage does:** Routes the CDM lever through all four domains. Shows BP control improvement (+22 ppt), ER reduction (-35% for HTN urgency), RAF specificity improvement, and net value with confidence range.

---

### Check Status

**Prompt:**
```
/status
```

**What Vantage does:** Shows database connection, reference tables loaded, active population (size, demographics, geography), completed scenarios, and generated outputs.

---

## Risk Adjustment

### V28 Transition Exposure

**Prompt:**
```
What's our V28 transition exposure? Which HCCs are most at risk
and how concentrated is the revenue impact?
```

**What Vantage does:** Scores the population under both V24 and V28, computes per-member RAF deltas, identifies the top 10 HCCs by revenue exposure, and produces a concentration analysis (what % of members drive 80% of the impact).

**Key output:**
- Annual revenue impact (V24 → V28)
- Per-HCC coefficient comparison table
- Concentration decile breakdown
- Generates HTML report to `outputs/`

---

### Coding Program ROI

**Prompt:**
```
Model a coding improvement program — IHA visits plus retrospective
chart review. What's the RAF impact and how does it vary by dual status?
```

**What Vantage does:** Applies two risk-domain effect weights (IHA capture rate: +18%, chart review: +11%). The SDOH modifier adjusts capture rate upward for high-dual populations (+28% yield for dual >35%). Shows per-HCC RAF delta and revenue projection.

---

### RAF Scoring Deep Dive

**Prompt:**
```
Score the population's RAF under both V24 and V28. Show me segment
distribution and which members have the largest delta.
```

**What Vantage does:** Determines payment segments (CNA, CFA, CND, etc.), looks up demographic + HCC coefficients, computes V24 and V28 RAF per member, and identifies concentration risk.

---

## Quality / Stars

### Stars Rating Projection

**Prompt:**
```
What's our current Stars rating based on this population's measure rates?
Which measures are closest to the next cut point?
```

**What Vantage does:** Evaluates 18 HEDIS measures across the five-population model (IPP, denominator, numerator, exclusion, exception), computes measure rates, maps to Stars levels via CMS cut points, and identifies the gap-to-next-star for each measure.

---

### Gap Closure Strategy

**Prompt:**
```
If we could close 60% of diabetes A1C gaps and 40% of breast cancer
screening gaps, what's the Stars impact and quality bonus?
```

**What Vantage does:** Applies measure-specific closure rates, re-projects Stars rating, and calculates quality bonus impact (5% of capitation if Stars crosses 4.0). Shows which measures are most impactful per quality bonus dollar.

---

### CDM Quality Impact

**Prompt:**
```
How does a pharmacist-led CDM program affect BP control, A1C control,
and statin adherence? What's the Stars lift?
```

**What Vantage does:** Applies three quality-domain effect weights (BP control: +22 ppt, A1C: +14 ppt, statin adherence: +9 ppt), adjusted by SVI tier for the population's county. Estimates aggregate Stars lift and quality bonus probability.

---

## Medical Cost

### TCOC Baseline

**Prompt:**
```
What's our total cost of care baseline? Show the breakdown by
cost category and the MLR position.
```

**What Vantage does:** Calculates TCOC from population demographics × RAF × capitation × utilization factors. Breaks into cost categories (inpatient 32%, outpatient 20%, professional 18%, pharmacy 20%, other 10%). Computes MLR and checks 85% compliance.

---

### MSSP Contract Modeling

**Prompt:**
```
Model this population under MSSP Enhanced track with downside risk.
Assume a $1,863 PMPM benchmark. What's our shared savings or loss position?
```

**What Vantage does:** Runs TCOC against the benchmark, applies lever effects, computes savings/loss under Enhanced track rules (shared savings rate + loss-sharing), and factors in the quality gate (Stars ≥ 4.0 for maximum savings rate).

---

### MLR by Cohort

**Prompt:**
```
Break down the MLR by age band and dual status. Which cohorts are
above 85% and which have headroom?
```

**What Vantage does:** Decomposes MLR across 4 age bands × 2 dual statuses (8 cohorts), applying age utilization factors (0.70 for under-65 to 1.60 for 85+) and dual cost multiplier (1.20x). Identifies compliance risk by segment.

---

## RADV Audit Risk

### Audit Risk Assessment

**Prompt:**
```
Score our population's RADV audit risk. Which HCCs have the worst
documentation and what's our net risk position?
```

**What Vantage does:** Scores each member-HCC on documentation completeness (encounter-to-doc lag), specificity (high/medium/low based on HCC audit patterns), and composite audit risk (0-1 scale). Aggregates to plan-level exposure delta.

**Key formula:** `Net Risk Position = risk_revenue_delta + radv_exposure_delta`

---

### Coding Program RADV Impact

**Prompt:**
```
If we run a coding improvement program, how much does it reduce our
audit exposure? Show the before/after audit risk distribution.
```

**What Vantage does:** Runs RADV model with and without coding program lever. The coding program improves documentation completeness (+18% specificity) and reduces audit risk score. Shows the delta in average risk and total exposure.

---

## PA Policy Simulation

### PA Relaxation for ACO Partners

**Prompt:**
```
Our hospital partners are threatening to leave the network. What if
we relax PA from 88% to 95% approval for ACO-attributed members?
Show the full cross-domain impact.
```

**What Vantage does:** Runs the PA model with VBC modulation (ACO members get higher approval rates). Shows the cascade:
- **Cost:** +$8-12M gross utilization, but 60% offset by ACO downside risk
- **Risk:** ~840 new HCCs from approved diagnostic imaging
- **Quality:** +0.3-0.6 Stars points from preventive service lift
- **RADV:** Modest documentation specificity improvement
- **Strategic value:** Network retention assessed

---

### PA Tightening Analysis

**Prompt:**
```
What if we tighten PA instead — drop to 80% approval across the board?
What do we save, and what do we lose?
```

**What Vantage does:** Runs the PA model with negative delta (95% → 80%). Cost decreases (utilization drops), but risk revenue decreases (fewer diagnostic workups → fewer HCC captures), quality declines (more treatment abandonment), and RADV worsens (less documentation support).

---

### Service-Category PA

**Prompt:**
```
Relax PA only for imaging and preventive services, but keep it tight
for elective procedures and specialty pharmacy. What's the impact?
```

**What Vantage does:** Runs PA simulation with selective service categories. Shows domain effects only from imaging (risk + RADV) and preventive (quality), while keeping cost-heavy categories (elective, pharmacy) under tight PA.

---

## Combined Levers

### The $2M Question

**Prompt:**
```
We have $2M to invest. Model three options:
1. CDM diabetes + hypertension program ($800K)
2. Coding improvement + provider incentives ($600K)
3. PA relaxation + formulary optimization ($600K)
Compare them and recommend.
```

**What Vantage does:** Runs three multi-lever scenarios, each with different combinations. The comparator normalizes, ranks across 7 dimensions, identifies trade-offs, and synthesizes a weighted recommendation.

---

### Kitchen Sink

**Prompt:**
```
What's the maximum value we could extract from this population?
Activate every lever — CDM for diabetes/HTN/CHF, coding program,
formulary optimization, PA relaxation for ACO, and steerage.
$2M total investment.
```

**What Vantage does:** Runs all available levers simultaneously. Shows the compounding effects (CDM improves quality AND reduces cost; coding improves risk AND reduces RADV; PA improves risk but increases cost with VBC offset). Demonstrates why cross-domain modeling matters — the total isn't the sum of individual lever runs.

---

## Scenario Comparison

### Side-by-Side

**Prompt:**
```
/compare CDM program vs coding program vs PA relaxation
```

**What Vantage does:** Loads all three scenario results, normalizes to PMPM, ranks across Risk/Quality/Cost/RADV/Net/ROI/Break-Even, identifies trade-offs, and produces a weighted recommendation.

---

### Sensitivity Check

**Prompt:**
```
How does the CDM recommendation change if engagement drops from 65% to 40%?
What about in a rural county with low SVI?
```

**What Vantage does:** Re-runs with adjusted parameters. Low engagement scales all domain effects proportionally. Rural county triggers the steerage SDOH modifier (ASC cost reduction drops 75%) and CDM SVI modifier adjusts upward (low SVI = higher engagement).

---

## Multi-Year Projection

### Three-Year NPV

**Prompt:**
```
Project the CDM program forward 3 years with $500K annual investment.
Apply standard CMS rate trending and engagement ramp.
```

**What Vantage does:** Projects Year 0 results through Years 1-3, applying:
- CMS rate trend: +3%/year (capitation increases)
- Medical cost trend: +6%/year (costs inflate)
- Engagement ramp: 40% → 60% → 70% (program matures)
- Population aging: +1 year/year (more conditions)
- Stars cut-point drift: -2%/year (measures get harder)

Shows year-by-year table with cumulative NPV (5% discount rate) and break-even year.

---

### Five-Year Strategic View

**Prompt:**
```
Show me a 5-year projection with aggressive engagement (70% year 1,
85% by year 3) and higher CMS rate growth (4%). What's the NPV?
```

**What Vantage does:** Accepts custom trend overrides and engagement ramp. Projects through 5 years with the adjusted assumptions.

---

## Executive Output

### HTML Dashboard

**Prompt:**
```
/report HTML dashboard of the CDM scenario with the 3-year projection
```

**What Vantage does:** Generates a self-contained HTML file with:
- Cover section (plan name, population, confidence)
- Executive summary KPIs (Net Value, Risk, Quality, Cost, RADV, ROI)
- Domain breakdown cards with lever detail
- Multi-year projection table with NPV
- Methodology and assumptions section
- Vantage-branded footer

---

### PPTX Board Deck

**Prompt:**
```
/report PPTX deck of the three-scenario comparison for the board
```

**What Vantage does:** Generates a PowerPoint file with:
- Title slide (dark background, teal accents)
- Recommendation slide with scenario overview cards
- Dimension-by-dimension comparison table

---

### Audience-Specific

**Prompt:**
```
Generate an HTML report focused on MLR compliance for the actuarial team.
Show the cohort-level MLR breakdown and cost category decomposition.
```

**What Vantage does:** Tailors the report content to the actuarial audience — emphasizes MLR by age band × dual status, cost category waterfall, benchmark comparison, and compliance headroom.

---

## Quick Reference: All Prompts

### Population
```
/population 10K members in LA County
/population 5,000 members with 30% dual, avg age 75, 2.0 avg RAF
/population 20K in rural Alabama — use Barbour County reference data
```

### Risk
```
What's our V28 exposure?
Score RAF under both V24 and V28
Model a coding program — IHA + chart review
```

### Quality
```
What's our Stars rating?
Which measures are closest to the next star?
Model CDM impact on BP control and A1C
```

### Cost
```
What's our TCOC baseline?
Model under MSSP Enhanced with $1,863 benchmark
Break down MLR by age and dual status
```

### RADV
```
Score our RADV audit risk
Which HCCs have the worst documentation?
How much does coding reduce our audit exposure?
```

### PA
```
What if we relax PA from 88% to 95% for ACO members?
What if we tighten PA to 80%?
Relax PA for imaging and preventive only
```

### Comparison
```
/compare CDM vs coding vs PA
Compare with 40% engagement instead of 65%
```

### Projection
```
Project forward 3 years with $500K/year investment
Show 5-year NPV with custom engagement ramp
```

### Reports
```
/report HTML dashboard
/report PPTX deck for the board
/report comparison report
```

---

*Vantage generates synthetic healthcare data for strategic modeling only. No real patient data is used, stored, or transmitted.*
