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
| [Setup](#setup) | Python environment, database bootstrap, first run |
| [Slash Commands](#slash-commands) | `/init`, `/population`, `/simulate`, `/compare`, `/report` |
| [Example Analysis](#example-analysis) | End-to-end scenario walkthrough |
| [Architecture](#architecture) | How the domain models connect |
| [Build Plan](docs/vantage-buildplan-v03.html) | Phase-by-phase development roadmap |

### Prerequisites

- Python 3.11+
- Claude Code (development environment and runtime)
- DuckDB (installed via pip)

### Setup

```bash
cd ~/Developer/projects/vantage
pip install -r requirements.txt
```

Bootstrap the database and load reference data:

```
/init
```

This creates the DuckDB database (`data/vantage.duckdb`) with four schemas (`reference`, `population`, `scenario`, `output`), loads CMS crosswalks, RAF coefficients, HEDIS measure specs, Stars cut points, effect weights, and county demographics.

**Optional — CMS MA enrollment enrichment:**

The county demographics loader populates CDC SVI and Census ACS data automatically. To add CMS MA enrollment data (enables the `dual_pct` SDOH modifier for coding programs):

```bash
# Download from CMS (175MB ZIP)
curl -L -o reference-data/geographic/monthly-enrollment-cpsc-december-2025.zip \
  "https://www.cms.gov/files/zip/monthly-enrollment-cpsc-december-2025.zip"
unzip -o reference-data/geographic/monthly-enrollment-cpsc-december-2025.zip \
  -d reference-data/geographic/

# Run enrichment
python3 data/loaders/enrich_county_demographics.py
```

### First Session

Open Claude Code in the project directory:

```
> /population 10,000 members in Los Angeles County
> /simulate CDM diabetes program with 65% engagement, $500K investment
> /compare CDM program vs coding improvement program
> /report HTML dashboard for the comparison
```

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

When processing a user request, Vantage routes to the appropriate skills:

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

**SDOH Modifiers**: Three levers carry population-sensitive adjustments. When a simulation population differs from the national MA average (high SVI, heavy dual mix, rural geography), the base effect weight is adjusted by a tier-based multiplier sourced from published literature.

---

## Prior Authorization — Cross-Domain Lever

PA policy is not a peer domain but a **gating mechanism** whose effects cascade through all four domains. When PA changes are in scope, Vantage models the full cascade:

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

Running `/simulate` with each option and then `/compare` produces:

| Metric | CDM Diabetes | Coding Program | PA Relaxation (88→95%) |
|--------|-------------|----------------|----------------------|
| Risk Revenue | +$450K | +$1.2M | +$680K |
| Quality Bonus | +$890K | — | +$220K |
| Medical Cost | -$1.1M savings | — | +$3.4M increase |
| RADV Exposure | — | +$180K reduced | +$95K reduced |
| **Net Value** | **+$2.4M** | **+$1.4M** | **-$2.4M** |
| ROI | 4.8x | 7.0x | n/a (strategic) |
| Confidence | High | High | Medium |

The comparator ranks by weighted score, identifies trade-offs, and synthesizes a recommendation. The temporal projector extends this over 3-5 years with engagement ramp, CMS rate trending, and NPV.

The report generator produces:
- **HTML dashboard**: Self-contained, inline CSS, four-domain KPIs, methodology section
- **PPTX slide deck**: Board-ready slides with cover, executive summary, domain breakdown, and recommendations

---

## Slash Commands

| Command | Purpose | Example |
|---------|---------|---------|
| `/init` | Bootstrap database, load reference data | `/init` |
| `/population` | Generate synthetic MA population | `/population 10K members in Miami-Dade County` |
| `/simulate` | Run cross-domain scenario | `/simulate CDM program for diabetes, 65% engagement` |
| `/compare` | Compare multiple scenarios | `/compare CDM vs coding vs PA relaxation` |
| `/report` | Generate executive output | `/report HTML dashboard for board presentation` |
| `/status` | Display session state | `/status` |

---

## Population Input

Vantage simulations require a population. Two modes are supported:

| Mode | Source | When to Use |
|------|--------|-------------|
| **Reference-calibrated** | County-level CMS + Census data (3,143 counties) | Exploring a market, benchmarking |
| **Plan-provided** | Customer supplies aggregate statistics | Modeling your actual population |

Every demographic parameter tracks its source (`reference:census_acs`, `reference:cms_county`, `plan_provided`, `synthetic_fallback`) so the user knows exactly what assumptions drove the simulation.

---

## Data Architecture

| Component | Storage | Row Count | Source |
|-----------|---------|-----------|--------|
| HCC Crosswalk (V24 + V28) | `reference.hcc_crosswalk` | 18,510 | CMS ICD-10 → CC → HCC mapping files |
| RAF Coefficients | `reference.raf_coefficients` | ~2,400 | CMS V24/V28 ratebook |
| HCC Hierarchy | `reference.hcc_hierarchy` | ~200 | CMS hierarchy rules |
| HEDIS Measure Specs | `reference.measure_specs` | 18 | NCQA HEDIS specifications |
| Stars Cut Points | `reference.stars_cut_points` | ~90 | CMS Star Ratings methodology |
| Effect Weights | `reference.effect_weights` | 23 | Peer-reviewed literature synthesis |
| PA Criteria Templates | `reference.pa_criteria_templates` | 7 | CMS PA policy guidelines |
| County Demographics | `reference.county_demographics` | 3,143 | Census ACS + CDC SVI + CMS MA enrollment |

**Development**: DuckDB local database (`data/vantage.duckdb`)
**Production**: Abstracted via DataStore interface (future: Databricks, Snowflake)

### Da Vinci FHIR IG Alignment

Vantage schemas are structurally aligned with Da Vinci implementation guides so that future ingestion of real FHIR data is a mapping exercise, not a schema redesign:

| Da Vinci IG | Vantage Table | Alignment |
|-------------|--------------|-----------|
| PAS (Prior Authorization Support) | `population.pa_events` | Event structure mirrors FHIR Claim/ClaimResponse |
| CRD (Coverage Requirements Discovery) | `reference.pa_criteria_templates` | Coverage rules structure |
| DTR (Documentation Templates) | `population.pa_events.da_vinci_dtr_context` | Documentation context |
| DEQM (Data Exchange for Quality) | `output.measure_populations` | Five-population model |
| CDex (Clinical Data Exchange) | `output.member_radv_risk` | Chart review request pattern |

---

## Repository Structure

```
vantage/
├── .claude/                          # Claude Code configuration
│   ├── CLAUDE.md                     # Orchestrator instructions (4 domains + skill routing)
│   └── commands/                     # Slash command definitions
│       ├── init.md                   #   /init — bootstrap database
│       ├── population.md             #   /population — generate synthetic MA pop
│       ├── simulate.md               #   /simulate — run cross-domain scenario
│       ├── compare.md                #   /compare — side-by-side analysis
│       ├── report.md                 #   /report — executive output generation
│       └── status.md                 #   /status — session state display
├── packages/core/src/vantage/
│   └── db/                           # Database abstraction layer
│       ├── schema.py                 #   4 schemas, 27 tables (DDL definitions)
│       ├── init_db.py                #   Bootstrap from CSVs (comment-aware SQL splitter)
│       ├── connection.py             #   ConnectionManager (close-before-write pattern)
│       ├── data_access.py            #   DataStore ABC (platform-agnostic interface)
│       └── duckdb_store.py           #   DuckDB implementation
├── skills/                           # Domain-specific intelligence (11 skills)
│   ├── population-engine/            #   Synthetic MA population generation
│   ├── risk-model/                   #   HCC mapping, RAF scoring, V28 analysis
│   ├── quality-model/                #   eCQM evaluation, Stars projection, gap closure
│   ├── vbc-model/                    #   TCOC calculator, contract modeler, savings projector
│   ├── medical-margin-model/         #   Integrated P&L projection (CFO layer)
│   ├── scenario-runner/              #   Triple-Play cross-domain orchestrator
│   ├── pa-model/                     #   Prior authorization cascade simulation
│   ├── radv-model/                   #   Audit risk scoring (4th domain)
│   ├── comparator/                   #   Multi-scenario comparison and ranking
│   ├── temporal-projector/           #   Multi-year projection with NPV
│   └── report-generator/            #   HTML dashboards + PPTX slide decks
├── reference-data/                   # Source CSVs loaded via /init
│   ├── cms/                          #   HCC crosswalks, RAF coefficients, hierarchy
│   ├── quality/                      #   HEDIS measures, Stars cut points
│   ├── geographic/                   #   County demographics (CMS enrollment on-demand)
│   └── vbc/                          #   MSSP benchmark methodology
├── data/
│   ├── vantage.duckdb                #   Main database (git-ignored, built from CSVs)
│   └── loaders/                      #   Python loaders for reference data
│       ├── load_cms_reference.py     #     HCC crosswalks + RAF coefficients
│       ├── load_county_demographics.py #   Census + CDC SVI + SDOH from HealthSim
│       ├── enrich_county_demographics.py # CMS MA enrollment + Census ACS API
│       ├── load_effect_weights.py    #     23 evidence-based effect weights
│       ├── load_pa_criteria.py       #     PA criteria templates
│       ├── load_measure_specs.py     #     HEDIS measure specifications
│       └── load_stars_cut_points.py  #     Stars rating thresholds
├── tests/                            # 92 tests across 8 files
│   ├── conftest.py                   #   Centralized sys.path + shared fixtures
│   ├── test_data_layer.py            #   ConnectionManager, DuckDBStore, schema init
│   ├── test_data_access.py           #   DataStore interface (uses tmp_store fixture)
│   ├── test_risk_model.py            #   CMS crosswalks, hierarchy, RAF scoring
│   ├── test_quality_model.py         #   Stars calculator, measure evaluation
│   ├── test_vbc_model.py             #   TCOC, MLR, shared savings projection
│   ├── test_medical_margin_model.py  #   P&L projection, ROI, MLR compliance
│   ├── test_phase3_pa_radv.py        #   PA lever, RADV model, Da Vinci alignment
│   └── test_phase4_output.py         #   Comparator, temporal projector, reports
├── outputs/                          # Generated reports (git-ignored)
├── docs/                             # Project documentation
│   ├── vantage-vision.html           #   Product vision and use cases
│   ├── vantage-architecture.html     #   Technical architecture deep dive
│   ├── vantage-buildplan-v03.html    #   Phase-by-phase build plan
│   ├── vantage-execution-guide-v11.html # Execution prompts and test scenarios
│   ├── vantage-effect-weights.html   #   Evidence-based effect weight methodology
│   └── ...                           #   UI mockups, test prompts, web architecture
├── requirements.txt
└── pyproject.toml
```

---

## Testing

```bash
python3 -m pytest tests/ -v
```

**92 tests, 0 skipped**, covering:

| Component | Tests | What's Verified |
|-----------|-------|-----------------|
| Data layer | 11 | ConnectionManager lifecycle, close-before-write, DuckDBStore interface, schema init, bulk insert |
| Risk model | 16 | CMS crosswalk lookups, coefficient validation, HCC hierarchy, RAF scoring, V24/V28 comparison |
| Quality model | 3 | Stars calculator projection, cut-point sensitivity, measure specs |
| VBC model | 4 | TCOC baseline, MLR calculation, MSSP Basic/Enhanced track |
| Medical margin | 8 | P&L projection, ROI, MLR compliance, PMPM view, multi-scenario compare |
| PA + RADV (Phase 3) | 28 | Effect weights, PA criteria, population PA events, RADV scoring, PA cascade, scenario integration, Da Vinci alignment |
| Output (Phase 4) | 16 | Comparator ranking, temporal projection trends, HTML dashboards, PPTX generation |
| Shared fixtures | — | `conftest.py` centralizes sys.path for all 11 skill directories |

### Testing Philosophy

- **When a test fails, the code is wrong, not the test.** No loosening assertions.
- **Effect weights verified against source citations.** Three-point estimates and confidence levels from published literature.
- All population data is **synthetic**. All reference data is **public CMS/Census/CDC**.

---

## Development Phases

| Phase | Focus | Status |
|-------|-------|--------|
| **Phase 1** | Foundation — Risk model (HCC mapping, RAF scoring, V24/V28) | Complete |
| **Phase 2** | Quality + VBC + Medical Margin + Scenario Runner | Complete |
| **Phase 3** | PA Lever + RADV Domain + Da Vinci Alignment | Complete |
| **Phase 4** | Temporal Projection + Comparator + Executive Output | Complete |
| **Data** | Effect Weight Library + County Demographics Enrichment | Complete |

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
| Testing | 92 tests, zero skips, conftest.py | Centralized path setup; no stale test stubs |

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
| [UI Mockup](docs/vantage-ui-mockup.html) | Future web interface design |
| [Web Architecture](docs/vantage-web-architecture-complete.html) | Production deployment architecture |

---

## Lineage

Vantage inherits architectural patterns from [HealthSim](https://github.com/mark64oswald/healthsim-workspace): the hierarchical seed manager for reproducible population generation, the close-before-write DuckDB connection pattern, the SDOH geographic modifier framework, and the skill-based Claude Code development model. The domain content (MA risk adjustment, Stars quality, VBC economics, RADV audit risk) is entirely Vantage-specific.

---

## PHI Safety

- All population data is **synthetic** — generated by the population engine, not sourced from claims
- All reference data is **public** — CMS crosswalks, Census demographics, CDC PLACES, Stars cut points
- When querying any database, queries must not return individual-level PHI
- When in doubt, ask first

---

*Vantage generates synthetic healthcare data for strategic modeling only. No real patient data is used, stored, or transmitted.*
