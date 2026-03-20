# Vantage — Healthcare Value Intelligence

**A prospective modeling tool for Medicare Advantage plans, ACOs, and risk-bearing provider organizations.**

Vantage models the cross-domain impact of clinical programs and strategic decisions across four interconnected outcome domains — simultaneously. Most analytics platforms model risk adjustment revenue in isolation. Vantage models the full balance sheet: risk revenue *and* audit liability, quality bonus *and* Stars trajectory, medical cost *and* contract economics — then shows how a single intervention cascades through all four.

One surface. One conversation. Four domains.

> **📖 This is the public documentation repository.** It mirrors the README files and documentation from the main Vantage project. Source code, skills, MCP servers, and data loaders are maintained in a private repository.

---

## The Four Outcome Domains

| Domain | What It Models | Business Impact | Key Metrics |
|--------|---------------|-----------------|-------------|
| **Risk Adjustment** | HCC coding, RAF scoring, V24/V28 transition | Base capitation revenue | RAF delta, HCC capture rate, V28 exposure |
| **Quality / Stars** | eCQM/HEDIS measures, Stars ratings, gap closure | Quality bonus revenue + member retention | Measure rates, Stars lift, bonus threshold |
| **Medical Cost / MLR** | TCOC, benchmarks, shared savings/loss, utilization | Medical margin (cost side) | TCOC PMPM, MLR, cost by category |
| **RADV Audit Risk** | Documentation completeness, coding specificity, audit scoring | Audit liability that offsets RAF revenue | Audit risk score, documentation lag, net risk position |

The most valuable insights span all four — a single intervention that improves HCC capture (revenue), quality measures (Stars), reduces utilization (cost), and strengthens documentation (audit resilience).

**Net Risk Position = risk_revenue_delta + radv_exposure_delta**

---

## Getting Started

| Resource | Description |
|----------|-------------|
| [Hello, Vantage!](hello-vantage/) | Guided walkthrough — zero to board-ready in 15 minutes |
| [Conversation Examples](hello-vantage/examples/) | 20+ example prompts organized by domain and complexity |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    User Conversation                     │
│         /population  /simulate  /compare  /report        │
├─────────────────────────────────────────────────────────┤
│                   Scenario Runner                        │
│          Triple-Play Cross-Domain Orchestrator            │
│    ┌──────────┬──────────┬──────────┬──────────┐        │
│    │   Risk   │ Quality  │   Cost   │   RADV   │        │
│    │  Model   │  Model   │  Model   │  Model   │        │
│    └────┬─────┴────┬─────┴────┬─────┴────┬─────┘        │
│         │          │          │          │               │
│    ┌────┴──────────┴──────────┴──────────┴─────┐        │
│    │          PA Lever (Cross-Domain)           │        │
│    │    Cascades through all four domains        │        │
│    └───────────────────────────────────────────┘        │
├─────────────────────────────────────────────────────────┤
│  Temporal Projector  │  Comparator  │  Report Generator  │
│  1-5 year NPV        │  Rank & rec  │  HTML + PPTX       │
├─────────────────────────────────────────────────────────┤
│  Population Engine  │  Medical Margin Model (CFO Layer)  │
│  Synthetic MA pop   │  P&L, ROI, MLR, break-even         │
├─────────────────────────────────────────────────────────┤
│                     Data Layer                           │
│  DuckDB (dev) │ DataStore ABC │ Effect Weights │ SDOH    │
│  4 schemas    │ Future: DBX   │ 23 weights     │ 3 mods  │
└─────────────────────────────────────────────────────────┘
```

### Skill Routing

| User Intent | Primary Skill | Supporting Skills |
|---|---|---|
| Generate/load population | population-engine | — |
| HCC mapping, RAF scoring | risk-model | population-engine |
| RADV exposure, audit risk | radv-model | population-engine |
| Quality gap analysis | quality-model | population-engine |
| Contract modeling, TCOC | vbc-model | risk-model, quality-model |
| Medical margin / P&L | medical-margin-model | risk-model, quality-model, vbc-model |
| PA policy simulation | pa-model | all domain models |
| Run scenario / what-if | scenario-runner | all domain models |
| Multi-year projection | temporal-projector | scenario-runner |
| Compare alternatives | comparator | scenario-runner |
| Executive output | report-generator | comparator or scenario-runner |

---

## Intervention Levers

Vantage models how intervention levers produce effects across domains. Each lever has evidence-based effect weights with three-point estimates (base, P10, P90) and confidence levels.

| Lever | Risk | Quality | Cost | RADV | SDOH Modifier |
|-------|------|---------|------|------|---------------|
| **CDM** (chronic disease management) | HCC specificity | BP/A1C control, adherence | ER reduction, TCOC | — | SVI tier |
| **Coding Program** | HCC capture rate, RAF delta | — | — | Audit risk reduction, doc specificity | Dual % tier |
| **PA Policy** | New HCC from imaging | Preventive measure lift | Utilization elasticity, VBC offset | Doc specificity from diagnostics | — |
| **Formulary** | — | Medication adherence | Pharmacy PMPM reduction | — | — |
| **Steerage** | — | — | Site-of-service cost reduction | — | Urban/rural |
| **Provider Contract** | — | — | — | Coding accuracy incentive | — |
| **V28 Response** | — | — | — | Documentation quality | — |

---

## Prior Authorization — Cross-Domain Lever

PA policy is not a peer domain but a **gating mechanism** whose effects cascade through all four domains:

```
PA approval rate change (e.g., 88% → 95%)
├── → Cost:    +1.8% utilization per 1pp increase (VBC offset for ACO members)
├── → Risk:    0.33 new HCCs per approved diagnostic imaging
├── → Quality: +0.06pp preventive measure completion lift
└── → RADV:    +0.12 documentation specificity improvement
```

---

## Example Analysis

**Scenario**: *"We have $2M to invest. Should we fund a CDM diabetes program, a coding improvement initiative, or relax PA for ACO partners?"*

| Metric | CDM Diabetes | Coding Program | PA Relaxation (88→95%) |
|--------|-------------|----------------|----------------------|
| Risk Revenue | +$450K | +$1.2M | +$680K |
| Quality Bonus | +$890K | — | +$220K |
| Medical Cost | -$1.1M savings | — | +$3.4M increase |
| RADV Exposure | — | +$180K reduced | +$95K reduced |
| **Net Value** | **+$2.4M** | **+$1.4M** | **-$2.4M** |
| ROI | 4.8x | 7.0x | n/a (strategic) |
| Confidence | High | High | Medium |

---

## Documentation

| Document | Description |
|----------|-------------|
| [Hello, Vantage!](hello-vantage/) | Guided walkthrough — zero to board-ready in 15 minutes |
| [Conversation Examples](hello-vantage/examples/) | 20+ example prompts organized by domain and complexity |
| [Vision](docs/vantage-vision.html) | Product vision, use cases, conversation examples |
| [Architecture](docs/vantage-architecture.html) | Technical architecture, skill routing, data model |
| [Build Plan](docs/vantage-buildplan-v03.html) | Phase-by-phase roadmap with checklists |
| [Execution Guide](docs/vantage-execution-guide-v11.html) | Step-by-step prompts and test scenarios |
| [Effect Weights](docs/vantage-effect-weights.html) | Evidence-based methodology and source citations |

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Runtime | Claude Code | Conversational interface matches how strategy leaders think — "what if we..." |
| Database | DuckDB (dev), DataStore ABC (prod) | Zero-config local development; swap to Databricks/Snowflake via interface |
| Effect weights | Literature-sourced, three-point estimates | Honest about uncertainty; every weight cites published evidence |
| SDOH modifiers | Population-sensitive tier multipliers | Same intervention has different effects in different communities |
| RADV as 4th domain | Model both revenue and audit liability | Most platforms ignore audit exposure — modeling both is the differentiator |
| PA as lever, not domain | Gating mechanism that cascades | Elevating PA to domain status would conflate inputs with outputs |
| Da Vinci alignment | Structural, not implementation | Future FHIR data ingestion is a mapping exercise, not a schema redesign |

---

## Lineage

Vantage inherits architectural patterns from [HealthSim](https://github.com/mark64oswald/healthsim-workspace): the hierarchical seed manager for reproducible population generation, the close-before-write DuckDB connection pattern, the SDOH geographic modifier framework, and the skill-based Claude Code development model.

---

## PHI Safety

- All population data is **synthetic** — generated by the population engine, not sourced from claims
- All reference data is **public** — CMS crosswalks, Census demographics, CDC PLACES, Stars cut points
- When querying any database, queries must not return individual-level PHI

---

*Vantage generates synthetic healthcare data for strategic modeling only. No real patient data is used, stored, or transmitted.*
