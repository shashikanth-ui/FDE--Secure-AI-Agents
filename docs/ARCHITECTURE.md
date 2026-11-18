# Architecture & Detailed Workflows: Secure AI Real Estate Analyst & GEAP Autorater Framework

> [!IMPORTANT]
> This document details the end-to-end system architecture, runtime component topology, multi-skill execution sequences, stochastic LLM-as-a-Judge evaluation pipelines, statistical inter-rater reliability analytics, and closed-loop agent hill-climbing workflows implemented in this repository.

---

## Table of Contents

1. [Architectural Overview](#1-architectural-overview)
2. [System Component Topology](#2-system-component-topology)
3. [Workflow 1: Multi-Skill Agent Investigation & Memorandum Generation](#3-workflow-1-multi-skill-agent-investigation--memorandum-generation)
4. [Workflow 2: Concurrent ADK EvalSet Trace Generation](#4-workflow-2-concurrent-adk-evalset-trace-generation)
5. [Workflow 3: Vertex AI GEAP Stochastic Multi-Pass Evaluation](#5-workflow-3-vertex-ai-geap-stochastic-multi-pass-evaluation)
6. [Workflow 4: Inter-Rater Reliability & Rubric Selection Triage](#6-workflow-4-inter-rater-reliability--rubric-selection-triage)
7. [Workflow 5: Closed-Loop Agent Hill-Climbing Optimization](#7-workflow-5-closed-loop-agent-hill-climbing-optimization)
8. [Data Contracts & Schema Specifications](#8-data-contracts--schema-specifications)
9. [Security, Guardrails & Reliability Engineering](#9-security-guardrails--reliability-engineering)

---

## 1. Architectural Overview

Evaluating domain-specific AI agents requires two tightly coupled capabilities:
1. **Grounded Agentic Execution**: A modular AI agent built on **Google Agent Development Kit (ADK 2.4.0)** (`re_opportunity_analyst`) that dynamically loads specialized domain skills (`SkillToolset`) and invokes deterministic data-retrieval tools (`FunctionTool`) without hallucinating financial or hydrological facts.
2. **Statistically Rigorous Autorater Governance**: An automated evaluation harness (`eval_harness/`) backed by **Vertex AI GenAI Evaluation Service (GEAP)** that executes $k$ stochastic LLM-as-a-Judge passes per rubric, measures inter-rater agreement via **Krippendorff's Alpha ($\alpha$)** and pairwise agreement, filters out noisy or ceiling-saturated rubrics, and drives iterative **hill-climbing** improvements on agent skills.

### Core Technology Stack

| Layer | Component | Implementation / Version | Purpose |
| :--- | :--- | :--- | :--- |
| **Agent Framework** | Google ADK | `google-adk==2.4.0` | `Agent`, `App`, `Runner`, `InMemorySessionService`, `SkillToolset`, `FunctionTool` |
| **Agent Model** | Vertex AI Gemini | `gemini-3.5-flash` (configurable via `.env`) | Generates structured 5-section Initial Site Evaluation Memoranda |
| **Autorater Service** | Vertex AI GEAP | `google-cloud-aiplatform[evaluation]==1.160.0` | `EvalTask`, `PointwiseMetric`, `PointwiseMetricPromptTemplate`, `AutoraterConfig` |
| **Judge Model** | Vertex AI Gemini Judge | `gemini-2.5-flash` (`sampling_count=4`) | Pointwise rubric scoring (1-5 Likert scale) + natural-language rationales |
| **Statistical Engine** | Reliability Analytics | `krippendorff-aleph-alpha==0.1.17`, `pandas==3.0.3` | Interval Krippendorff's $\alpha$, pairwise % agreement, intra/inter-report $\sigma$ |
| **Domain Data Store** | Structured JSON Stores | `lots_database.json`, `tax_database.json` | County parcel records, FEMA flood zones, adjacent parcels, APN tax assessments |

---

## 2. System Component Topology

The following high-level architectural diagram illustrates how the **ADK Agent Runtime**, **Domain Data Layer**, **Evaluation & Hill-Climbing Harness**, **Vertex AI Cloud Services**, and **Statistical Analytics Engine** interact across execution boundaries.

```mermaid
flowchart TB
    subgraph OperatorLayer ["Operator & Developer Layer"]
        CLI["CLI Entrypoints & ADK Web UI"]
        LotsTxt["Target Lot Manifest (eval_harness/hillclimb_lots.txt)"]
        RubricsJson["19 Rubric Definitions (eval_harness/rubrics.json)"]
    end

    subgraph AgentRuntime ["ADK 2.4 Agent Runtime (re_analyst/)"]
        App["ADK App & Runner (InMemorySessionService)"]
        RootAgent["re_opportunity_analyst (Agent + System Instruction)"]
        SkillSet["SkillToolset (Dynamic Progressive Skill Loader)"]

        subgraph DomainSkills ["Domain Skills (re_analyst/skills/)"]
            SkillOpp["analyze-opportunity (Cost-Per-Acre & Orchestration)"]
            SkillFlood["analyze-flood-risk (Subject + Adjacent Parcel Hydrology)"]
            SkillTax["analyze-tax-implications (APN Tax & Carrying Costs)"]
        end

        subgraph FunctionTools ["Deterministic FunctionTools (re_analyst/tools/)"]
            LotTool["get_lot_data(plot_id)"]
            TaxTool["get_tax_assessment_data(parcel_apn)"]
        end
    end

    subgraph DataStore ["Local Domain Data Layer (tools_data/)"]
        LotsDB[("lots_database.json (County & GIS Records)")]
        TaxDB[("tax_database.json (Assessor Tax Records)")]
    end

    subgraph CloudAI ["Google Cloud Vertex AI"]
        GeminiAgent["Vertex AI Gemini API (gemini-3.5-flash)"]
        subgraph GEAPService ["Vertex AI GenAI Evaluation Service (GEAP)"]
            EvalTaskAPI["EvalTask Orchestrator (Experiment Run Tracking)"]
            JudgeLLM["Autorater Judge Model (gemini-2.5-flash, sampling_count=4)"]
        end
    end

    subgraph EvalHarness ["Evaluation & Hill-Climbing Harness (eval_harness/)"]
        DatasetBuilder["create_re_agent_evalset.py (Async Semaphore=10)"]
        EvalSetFile[("ADK EvalSet JSON (*.evalset.json)")]
        GeapRunner["run_geap_eval.py (GeapEvalRunner + Template Assembler)"]
        HillclimbRunner["run_hillclimb.py (Versioned Iteration Orchestrator)"]
        CallLogs[("GEAP Calls & Responses JSON (runs/*/geap_calls_and_responses*.json)")]
    end

    subgraph AnalyticsLayer ["Statistical Reliability & Diagnostics Engine"]
        MetricsCalc["metrics_calc.py (Krippendorff Alpha + Variance Stats)"]
        AllCrit["all_criteria_analysis.py (Global 19-Criteria Summary)"]
        SingleCrit["single_criterion_analysis.py (N x k Grades Matrix)"]
        ExplainCrit["explain_criteria_results.py (Low-Score Judge Rationales)"]
    end

    CLI --> DatasetBuilder
    CLI --> HillclimbRunner
    LotsTxt --> DatasetBuilder
    LotsTxt --> HillclimbRunner
    DatasetBuilder --> App
    App --> RootAgent
    RootAgent <--> GeminiAgent
    RootAgent --> SkillSet
    SkillSet --> SkillOpp
    SkillSet --> SkillFlood
    SkillSet --> SkillTax
    SkillOpp --> LotTool
    SkillFlood --> LotTool
    SkillTax --> TaxTool
    LotTool --> LotsDB
    TaxTool --> TaxDB

    App --> EvalSetFile
    EvalSetFile --> GeapRunner
    RubricsJson --> GeapRunner
    HillclimbRunner --> DatasetBuilder
    HillclimbRunner --> GeapRunner
    GeapRunner <--> EvalTaskAPI
    EvalTaskAPI <--> JudgeLLM
    GeapRunner --> CallLogs

    CallLogs --> MetricsCalc
    MetricsCalc --> AllCrit
    MetricsCalc --> SingleCrit
    CallLogs --> ExplainCrit
    ExplainCrit -. "Diagnose Failures & Refine SKILL.md" .-> DomainSkills
```

---

## 3. Workflow 1: Multi-Skill Agent Investigation & Memorandum Generation

When a user or evaluation runner submits a property identifier (e.g., `LOT-8472`), `re_opportunity_analyst` executes a structured multi-skill investigation before drafting the standardized five-section memorandum.

### Key Architectural Mechanics
- **Progressive Disclosure via `SkillToolset`**: Rather than stuffing every analytical rule into the root `SYSTEM_INSTRUCTION`, domain procedures are modularized into `SKILL.md` files with YAML frontmatter declaring `adk_additional_tools`.
- **Cross-Parcel Graph Traversal**: In `analyze-flood-risk`, the agent inspects `adjacent_parcels` returned from the subject property's record and invokes `get_lot_data` for **every neighboring lot ID** to compare FEMA flood zones, topography slopes, soil complexes, and easements.
- **Cross-Database Join via `parcel_apn`**: `get_lot_data` resolves a `plot_id` (e.g., `LOT-8472`) to its county Assessor's Parcel Number (`049-382-011-000`), which `analyze-tax-implications` passes to `get_tax_assessment_data` to retrieve official assessed land values and actual 2025 tax bills.
- **In-Memory Caching**: Both `lot_tool.py` and `tax_tool.py` maintain module-level singleton caches (`_LOTS_DB_CACHE`, `_TAX_DB_CACHE`) to eliminate redundant disk I/O during concurrent batch runs.

```mermaid
sequenceDiagram
    autonumber
    actor Caller as User / EvalSet Runner
    participant Agent as re_opportunity_analyst (ADK Agent)
    participant Gemini as Vertex AI (gemini-3.5-flash)
    participant Skills as SkillToolset (SKILL.md Loader)
    participant LotTool as get_lot_data (lots_database.json)
    participant TaxTool as get_tax_assessment_data (tax_database.json)

    Caller->>Agent: "Please evaluate investment suitability for lot LOT-8472 and generate a report."
    Agent->>Gemini: System Instruction + User Prompt + SkillToolset Declarations

    Note over Agent,Skills: Step 1: Opportunity Valuation & Cost-Per-Acre
    Gemini->>Skills: Load skill "analyze-opportunity"
    Skills-->>Gemini: Return opportunity instructions + bind get_lot_data
    Gemini->>LotTool: get_lot_data(plot_id="LOT-8472")
    LotTool-->>Gemini: JSON {acreage: 1.2, last_sale_price: 450000, parcel_apn: "049-382-011-000", flood_zone: "Zone AE", adjacent_parcels: ["LOT-8471", "LOT-8473"], ...}
    Note right of Gemini: Compute Cost/Acre = $450,000 / 1.2 = $375,000.00/acre

    Note over Agent,LotTool: Step 2: Subject & Neighboring Parcel Hydrological Risk Analysis
    Gemini->>Skills: Load skill "analyze-flood-risk"
    Skills-->>Gemini: Return flood-risk instructions (mandates querying all adjacent_parcels)
    par Query Every Neighboring Parcel
        Gemini->>LotTool: get_lot_data(plot_id="LOT-8471")
        LotTool-->>Gemini: JSON {flood_zone: "Zone AE (High Risk)", topography: "0-2% grade", soil: "Urban Land-Inwood", easement: "None"}
    and
        Gemini->>LotTool: get_lot_data(plot_id="LOT-8473")
        LotTool-->>Gemini: JSON {flood_zone: "Zone X (Minimal Risk)", topography: "2-5% grade", soil: "Urban Land-Inwood", easement: ...}
    end
    Note right of Gemini: Compare subject Zone AE vs LOT-8471 (Zone AE) & LOT-8473 (Zone X upslope 2-5% grade runoff analysis)

    Note over Agent,TaxTool: Step 3: Official APN Tax Assessment & Holding Cost Analysis
    Gemini->>Skills: Load skill "analyze-tax-implications"
    Skills-->>Gemini: Return tax instructions + bind get_tax_assessment_data
    Gemini->>TaxTool: get_tax_assessment_data(parcel_apn="049-382-011-000")
    TaxTool-->>Gemini: JSON {tax_year: 2025, annual_tax_amount: 23600.0, total_assessed_value: 856000, payment_status: "Current"}

    Note over Agent,Caller: Step 4: Synthesize Strictly Formatted 5-Section Memorandum
    Gemini-->>Agent: Final Markdown Report (Exec Summary, Acquisition Basis, Zoning/Infra, Risk Assessment, Recommendation)
    Agent-->>Caller: Persist Session Events & Return Final Memorandum
```

---

## 4. Workflow 2: Concurrent ADK EvalSet Trace Generation

`eval_harness/create_re_agent_evalset.py` automates batch agent execution over a target list of Lot IDs and serializes full multi-turn tool/skill telemetry in ADK 2.0 `EvalSet` schema format.

```mermaid
flowchart LR
    A["Input Manifest (hillclimb_lots.txt)"] --> B["Parse Valid Lot IDs (Strip blanks & '#' comments)"]
    B --> C["Build EvalCase Objects (Prompt + ConversationScenario)"]
    C --> D["Dynamic Agent Import (utils.load_app_from_dir)"]
    D --> E["Initialize InMemorySessionService & ADK Runner"]
    E --> F["Concurrent Execution Pool (asyncio.Semaphore = 10)"]
    F --> G["Capture Session Events & Convert via EvaluationGenerator"]
    G --> H["Graceful Async Client & Runner Teardown"]
    H --> I["Serialize Pydantic EvalSet (*.evalset.json)"]
```

### Key Implementation Details
1. **Clean Dynamic Module Loading (`utils.load_app_from_dir`)**: Temporarily injects the parent directory into `sys.path` to import `<agent_dir>.agent.app`, then immediately removes the path entry in a `finally` block to prevent cross-module namespace pollution.
2. **Bounded Async Concurrency**: Uses `asyncio.Semaphore(max_concurrency=10)` across `runner.run_async(...)` coroutines so 10 properties are analyzed in parallel without overwhelming Vertex AI QPS quotas.
3. **Native Telemetry Conversion**: Converts raw session events into structured ADK `Invocation` records via `EvaluationGenerator.convert_events_to_eval_invocations(session.events)` and clears `conversation_scenario` once populated.
4. **Connection Pool Cleanup**: Explicitly closes `runner.close()` and underlying `api_client.aio.aclose()` to avoid dangling asyncio transport warnings on Python 3.13.

---

## 5. Workflow 3: Vertex AI GEAP Stochastic Multi-Pass Evaluation

`eval_harness/run_geap_eval.py` (`GeapEvalRunner`) bridges ADK `EvalSet` outputs to **Vertex AI GenAI Evaluation Service (GEAP)**, supporting both offline `--dry-run` prompt verification and live $k$-pass stochastic cloud evaluations.

```mermaid
flowchart TD
    Start(["Start GeapEvalRunner"]) --> LoadTraces["Extract {eval_id, response} from EvalSet JSON"]
    LoadTraces --> LoadRubrics["Load Criteria & 1-5 Scales from rubrics.json"]
    LoadRubrics --> FilterCheck{"target_criterion_ids specified?"}
    FilterCheck -- "Yes (Hillclimb Mode)" --> SubsetCriteria["Filter to Target Criteria (e.g. cost-per-acre, neighboring-flood-risk)"]
    FilterCheck -- "No (Full Audit)" --> AllCriteria["Retain All 19 Criteria"]

    SubsetCriteria --> BuildTemplates["Build PointwiseMetricPromptTemplate & PointwiseMetric"]
    AllCriteria --> BuildTemplates

    BuildTemplates --> SDKFix["Apply SDK Workaround: template.variables = template._get_variables()"]
    SDKFix --> DryRunCheck{"--dry-run flag set?"}

    DryRunCheck -- "Yes" --> AssemblePrompts["Assemble Exact Prompts via template.assemble(response=...)"]
    AssemblePrompts --> MarkDry["Set is_dry_run = true"]

    DryRunCheck -- "No (Live Cloud Eval)" --> InitVertex["Initialize Vertex AI & AutoraterConfig (gemini-2.5-flash, sampling_count=4)"]
    InitVertex --> LoopPasses["Loop eval_pass_idx = 1 .. k_passes"]
    LoopPasses --> NewEvalTask["Instantiate Fresh EvalTask (Prevents SSL Context Reuse Bug)"]
    NewEvalTask --> DispatchCloud["eval_task.evaluate(experiment_run_name='pass-i')"]
    DispatchCloud --> ParseTable["Extract {criterion}/score & {criterion}/explanation from metrics_table"]
    ParseTable --> MorePasses{"More passes remaining?"}
    MorePasses -- "Yes" --> LoopPasses
    MorePasses -- "No" --> MergeResults["Couple Assembled Prompts with k-Pass Responses"]

    MarkDry --> SaveJSON["Write Output JSON (runs/*/geap_calls_and_responses*.json)"]
    MergeResults --> SaveJSON
    SaveJSON --> End(["Complete"])
```

### Two-Level Sampling Architecture
- **Intra-Pass Judge Sampling (`judge_model_sampling_count=4`)**: Inside each GEAP cloud call, `AutoraterConfig` instructs the Vertex AI evaluation backend to sample the judge model (`gemini-2.5-flash`) 4 times and aggregate a point score and explanation.
- **Inter-Pass Stochastic Replication (`-k / --k-passes`)**: `execute_cloud_evaluation` executes $k$ independent `EvalTask` passes (`pass-1`, `pass-2`, ..., `pass-k`) logged under a timestamped Vertex AI Experiment (`{experiment_name}-{YYYYMMDD-HHMMSS}`), enabling empirical measurement of rater variance across passes.

---

## 6. Workflow 4: Inter-Rater Reliability & Rubric Selection Triage

Before using an LLM autorater to optimize an agent, engineers must verify that the rubric itself is **consistent across passes** and **discriminative across agent outputs**. `metrics_calc.py`, `all_criteria_analysis.py`, and `single_criterion_analysis.py` implement a complete psychometric evaluation pipeline over the $N \text{ (lots)} \times k \text{ (passes)}$ grades matrix.

### Statistical Metrics Computed (`metrics_calc.py`)

| Metric | Formula / Definition | Diagnostic Purpose |
| :--- | :--- | :--- |
| **`mean`** | Global mean across all $N \times k$ ratings | Measures current agent capability level on a 1–5 scale |
| **`sd_report_means`** | $\text{SD}_i(\bar{x}_{i,\cdot})$ across $N$ property reports | Measures **discriminative power** (spread between strong and weak reports). Zero means ceiling/floor saturation! |
| **`sd_pass_means`** | $\text{SD}_j(\bar{x}_{\cdot,j})$ across $k$ evaluation passes | Detects systematic pass-to-pass drift |
| **`pairwise %_agree`** | $\frac{\sum c(c-1)}{n(n-1)}$ averaged across $N$ items | Measures exact score agreement across repeated passes on the same report |
| **`alpha` (Krippendorff's $\alpha$)** | $1 - \frac{D_o}{D_e}$ (Interval metric via `krippendorff-aleph-alpha`) | Chance-corrected inter-rater reliability. $\alpha \ge 0.80$ indicates high reliability; $\alpha \approx 0.50$ indicates noisy/ambiguous rubrics |

### Rubric Reliability Triage Decision Workflow

```mermaid
flowchart TD
    Input["19 Candidate Rubrics Evaluated Across N Reports x k Passes"] --> CheckSpread{"sd_report_means > 0? (Non-Zero Variance Across Reports)"}

    CheckSpread -- "No (sd_report_means = 0, All Scores = 5)" --> CeilingBucket["Ceiling / Saturated Criteria (general-tone, logical-flow, professionalism, readability)"]
    CeilingBucket --> RejectCeiling["Exclude from Optimization Target: Zero discriminative signal despite 100% agreement"]

    CheckSpread -- "Yes" --> CheckAlpha{"Krippendorff's Alpha >= 0.80 & Actionable Domain Rubric?"}

    CheckAlpha -- "No (Alpha 0.49 - 0.67: Vague / Subjective Wording)" --> NoisyBucket["Low-Reliability / Subjective Criteria (professional-tone: 0.498, data-density: 0.499, report-length: 0.549, vocabulary-choice: 0.623)"]
    NoisyBucket --> RejectNoisy["Rewrite Rubric Anchors Before Using as Automated Gate"]

    CheckAlpha -- "Yes (Alpha >= 0.80 + Clear Quantitative/Domain Anchors)" --> GoldBucket["Production-Grade Autorater Rubrics"]
    GoldBucket --> Top3["Top Validated Rubrics (best_criteria_submission.txt): 1. monthly-tax-holding (alpha=0.997, PA=99.17%) 2. environmental-esa (alpha=0.897, PA=79.10%) 3. flood-zone-flagging (alpha=0.897, PA=94.41%)"]
```

### Empirical Baseline Audit Results (19 Criteria)

| Category | Criteria Examples | Mean | `sd_report_means` | Pairwise Agree | Krippendorff's $\alpha$ | Verdict |
| :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| **Gold-Standard Discriminative** | `monthly-tax-holding`<br/>`flood-zone-flagging`<br/>`environmental-esa` | 2.2764<br/>4.8447<br/>4.2772 | 0.8872<br/>0.3357<br/>0.7042 | 99.17%<br/>94.41%<br/>79.10% | **0.997**<br/>**0.897**<br/>**0.897** | **Selected** (`best_criteria_submission.txt`) — High $\alpha$, crisp objective anchors, strong signal |
| **Ceiling-Saturated** | `general-tone`, `logical-flow`, `professionalism`, `readability` | 5.0000 | 0.0000 | 100.00% | 1.000 | **Excluded** — Saturated at 5/5; provides no gradient for improvement |
| **Subjective / Noisy** | `professional-tone`<br/>`data-density`<br/>`document-formatting` | 4.9958<br/>4.9979<br/>4.8361 | 0.0164<br/>0.0118<br/>0.1488 | 99.17%<br/>99.58%<br/>75.22% | 0.498<br/>0.499<br/>0.548 | **Excluded** — Vague wording (*"plenty of numbers"*) causes chance-level $\alpha$ |
| **Hill-Climbing Targets** | `cost-per-acre`<br/>`neighboring-flood-risk` | 1.3771<br/>1.0316 | 0.2163<br/>0.1768 | 58.97%<br/>100.00% | 0.565<br/>1.000 | **Targeted in Hill-Climbing** — Agent severely underperforming (~1.0/5.0) due to missing skill logic |

---

## 7. Workflow 5: Closed-Loop Agent Hill-Climbing Optimization

`eval_harness/run_hillclimb.py` and `eval_harness/explain_criteria_results.py` implement a disciplined, version-controlled optimization loop focused on lifting low-performing capabilities (`cost-per-acre` and `neighboring-flood-risk`) across 10 edge-case lots (`hillclimb_lots.txt`).

```mermaid
stateDiagram-v2
    [*] --> BaselineRun: Execute run_hillclimb.py (hillclimb_1)
    BaselineRun --> DiagnoseFailures: Scores = 1.00 (cost-per-acre) & 1.10 (neighboring-flood-risk)

    state DiagnoseFailures {
        [*] --> InspectRationales: Run explain_criteria_results.py (--max_score_to_show 4.0)
        InspectRationales --> RootCause1: Finding 1 - Agent omits Cost/Acre formula & valuation prominence
        InspectRationales --> RootCause2: Finding 2 - Agent ignores adjacent_parcels array in get_lot_data
    }

    DiagnoseFailures --> RefineSkills: Edit SKILL.md Instructions (Iteration 2 -> Iteration 3)

    state RefineSkills {
        [*] --> UpdateOpportunity: Update analyze-opportunity/SKILL.md (Mandate Sale Price / Acreage formula + 2-decimal formatting + Exec Summary & Acquisition Basis placement + zero/missing guardrails)
        UpdateOpportunity --> UpdateFloodRisk: Update analyze-flood-risk/SKILL.md (Mandate separate get_lot_data call for EVERY adjacent_parcels ID + comparative FEMA zone, topography, soil & easement hydrology analysis)
    }

    RefineSkills --> ValidationRun: Re-run run_hillclimb.py (hillclimb_3)
    ValidationRun --> TargetAchieved: Mean Score = 5.0000 / 5.0000 across all 10 lots!
    TargetAchieved --> [*]
```

### Empirical Hill-Climbing Progression Across Versioned Runs

| Iteration | EvalSet Trace Artifact | GEAP Results Artifact | `cost-per-acre` Mean | `neighboring-flood-risk` Mean | Key Engineering Change |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **`hillclimb_1`** (Baseline) | `re_analyst/hillclimb_1.evalset.json` | `runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json` | `1.0000` | `1.1000` | Unmodified starter skills (no cost-per-acre math; single-parcel flood lookup only) |
| **`hillclimb_2`** (Intermediate) | `re_analyst/hillclimb_2.evalset.json` | `runs/hillclimb_2/geap_calls_and_responses_hillclimb_2.json` | `1.2000` | `1.0000` | Diagnostic trace inspection verifying `SkillToolset` tool bindings (`adk_additional_tools: [get_lot_data]`) |
| **`hillclimb_3`** (Optimized) | `re_analyst/hillclimb_3.evalset.json` | `runs/hillclimb_3/geap_calls_and_responses_hillclimb_3.json` | **`5.0000`** | **`5.0000`** | Explicit formula, section placement, missing-value guardrails in `analyze-opportunity/SKILL.md` + mandatory per-neighbor `get_lot_data` loop & hydrological comparison in `analyze-flood-risk/SKILL.md` |

---

## 8. Data Contracts & Schema Specifications

### 8.1 Property Record Schema (`tools_data/lots_database.json`)
Keyed by `plot_id` (e.g., `"LOT-8472"`):
```json
{
  "plot_id": "LOT-8472",
  "parcel_apn": "049-382-011-000",
  "acreage": 1.2,
  "zoning": "R-3 (Multifamily)",
  "flood_zone": "Zone AE (High Risk)",
  "last_sale_price": 450000,
  "estimated_annual_taxes": 8500,
  "environmental_status": "Phase I ESA - Clean",
  "utility_access": "City Water & Sewer at boundary",
  "topography": "0-2% grade (Flat)",
  "adjacent_parcels": ["LOT-8471", "LOT-8473"],
  "soil_classification": "Urban Land-Inwood complex",
  "easement_type": "Public Utility (Rear 5ft)"
}
```

### 8.2 Tax Assessment Schema (`tools_data/tax_database.json`)
Keyed by `parcel_apn` (e.g., `"049-382-011-000"`):
```json
{
  "parcel_apn": "049-382-011-000",
  "plot_id": "LOT-1011",
  "tax_year": 2025,
  "annual_tax_amount": 23600.0,
  "assessed_land_value": 856000,
  "assessed_improvement_value": 0,
  "total_assessed_value": 856000,
  "tax_jurisdiction": "County Assessor Office",
  "payment_status": "Current - Paid in Full"
}
```

### 8.3 Rubric Definition Schema (`eval_harness/rubrics.json`)
```json
{
  "criterion_id": "cost-per-acre",
  "name": "Acquisition Cost-Per-Acre Calculation",
  "criterion": "Derives and states the exact mathematical acquisition cost-per-acre based on the last sale price and acreage.",
  "rating_scale": {
    "1": "Cost-per-acre calculation is entirely omitted or mathematically incorrect.",
    "2": "Alludes to land cost relative to size, but does not provide a numerical figure or formula.",
    "3": "Mentions cost-per-acre roughly or without explicit derivation formula.",
    "4": "Correctly computes cost-per-acre with clear derivation, but does not benchmark or emphasize it as a core valuation metric.",
    "5": "Accurately calculates and highlights exact cost-per-acre as a core valuation metric."
  }
}
```

### 8.4 GEAP Call & Response Collection Schema (`runs/*/geap_calls_and_responses*.json`)
```json
{
  "eval_id": "LOT-8472",
  "criterion_id": "cost-per-acre",
  "criterion_name": "Acquisition Cost-Per-Acre Calculation",
  "prompt_made_to_service": "...rendered PointwiseMetricPromptTemplate...",
  "responses_from_service": [
    {
      "eval_pass_index": 1,
      "score": 5.0,
      "explanation": "The report accurately calculates the cost-per-acre ($450,000 / 1.2 acres = $375,000.00/acre)..."
    }
  ]
}
```

---

## 9. Security, Guardrails & Reliability Engineering

1. **Credential & Secret Isolation**:
   - Runtime authentication relies exclusively on **Google Cloud Application Default Credentials (ADC)** (`gcloud auth application-default login`), avoiding static service account JSON keys.
   - Environment configuration (`.env`) and local SQLite session databases (`.adk/session.db`) are excluded via `.gitignore`; `.env.example` provides sanitized configuration templates.
2. **Anti-Hallucination & Epistemic Guardrails in Agent Skills**:
   - `analyze-opportunity/SKILL.md` enforces: *"If sale price or acreage is missing, invalid, or zero, state that the calculation cannot be completed. Never invent values."*
   - `analyze-flood-risk/SKILL.md` enforces: *"Distinguish verified property data from potential hydrological implications. Never claim a specific runoff direction without supporting data."*
3. **Concurrency & Transport Hardening**:
   - **mTLS Bypass for Concurrent Vertex SDK Calls**: Both `run_geap_eval.py` and `run_hillclimb.py` set `GOOGLE_API_USE_MTLS="never"` and `GOOGLE_API_USE_CLIENT_CERTIFICATE="false"` prior to importing Vertex AI libraries, preventing `pyOpenSSL`'s `"Context has already been used to create a Connection"` race condition during multi-threaded `EvalTask` execution.
   - **Per-Pass `EvalTask` Instantiation**: `GeapEvalRunner.execute_cloud_evaluation` constructs a fresh `EvalTask` object on every pass iteration to avoid stateful connection-pool corruption.
   - **Vertex AI SDK PromptTemplate Initialization Fix**: Explicitly populates `template.variables = template._get_variables()` to work around an upstream inheritance bug in `_MetricPromptTemplate` so `.assemble(response=...)` renders audit logs deterministically.
   - **HTTP Retry Backoff**: `re_opportunity_analyst` configures `types.HttpRetryOptions(initial_delay=1, attempts=3)` on the `Gemini` model wrapper for resilient transient error recovery.
