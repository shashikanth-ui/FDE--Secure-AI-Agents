# Step-by-Step Operational Runbook: ADK Real Estate Agent & Vertex AI GEAP Autorater Lab

> [!NOTE]
> This runbook provides copy-pasteable, step-by-step operational procedures to provision the environment, run the **Google ADK Real Estate Investment Opportunity Analysis Agent**, execute **Vertex AI GenAI Evaluation Service (GEAP)** multi-pass autoraters, perform **Krippendorff's Alpha ($\alpha$) rubric reliability analysis**, and execute **closed-loop agent hill-climbing**.

---

## Table of Contents

- [Prerequisites & IAM Permissions](#prerequisites--iam-permissions)
- [Phase 1: Environment Bootstrap & Configuration](#phase-1-environment-bootstrap--configuration)
- [Phase 2: Interactive Agent Verification via ADK Web UI](#phase-2-interactive-agent-verification-via-adk-web-ui)
- [Phase 3: Generate ADK Evaluation Traces (`EvalSet`)](#phase-3-generate-adk-evaluation-traces-evalset)
- [Phase 4: Execute GEAP Autoraters (Dry-Run & Multi-Pass Cloud Eval)](#phase-4-execute-geap-autoraters-dry-run--multi-pass-cloud-eval)
- [Phase 5: Statistical Rubric Reliability & Agreement Triage](#phase-5-statistical-rubric-reliability--agreement-triage)
- [Phase 6: Iterative Agent Hill-Climbing Workflow](#phase-6-iterative-agent-hill-climbing-workflow)
- [CLI Command Reference Matrix](#cli-command-reference-matrix)
- [Troubleshooting & Operational Playbook](#troubleshooting--operational-playbook)

---

## Prerequisites & IAM Permissions

Before starting, verify the following requirements in your Google Cloud project:

1. **Google Cloud APIs Enabled**:
   - Vertex AI API (`aiplatform.googleapis.com`)
2. **Required IAM Roles** (for your user or service principal):
   - `roles/aiplatform.user` (Vertex AI User — required for Gemini inference, GEAP `EvalTask`, and Vertex AI Experiments)
   - `roles/storage.objectViewer` / `roles/storage.objectCreator` (optional, if syncing artifacts to/from Cloud Storage)
3. **Runtime Tooling**:
   - Linux / macOS shell (`bash`)
   - Google Cloud SDK (`gcloud`)
   - Python **3.13** (managed via `pyenv` or virtual environment)

---

## Phase 1: Environment Bootstrap & Configuration

### Step 1.1: Authenticate Google Cloud CLI & Application Default Credentials (ADC)

```bash
export PROJECT_ID="your-gcp-project-id"

gcloud config set project "${PROJECT_ID}"
gcloud auth application-default login
gcloud services enable aiplatform.googleapis.com --project="${PROJECT_ID}"
```

### Step 1.2: Install Python 3.13 & Project Dependencies

If Python 3.13 is not yet installed on your workstation or Cloud Shell instance:

```bash
sudo apt-get update && sudo apt-get install -y libreadline-dev

# Install pyenv if not already present
if [ ! -d "$HOME/.pyenv" ]; then
  curl https://pyenv.run | bash
  echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
  echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
  echo 'eval "$(pyenv init -)"' >> ~/.bashrc
fi

export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"

pyenv install -s 3.13
pyenv global 3.13
python3 --version
```

Clone the repository and install pinned dependencies inside a virtual environment (recommended) or active `pyenv` environment:

```bash
git clone https://github.com/shashikanth-ui/FDE--Secure-AI-Agents.git
cd FDE--Secure-AI-Agents

python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
```

### Step 1.3: Configure Runtime Environment Variables (`.env`)

Copy `.env.example` to `.env` and set your Google Cloud Project ID:

```bash
cp .env.example .env
cat << EOF > .env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
EOF
```

> [!CAUTION]
> Never commit `.env` to version control. `.gitignore` is pre-configured to exclude `.env` and `.adk/` session databases.

---

## Phase 2: Interactive Agent Verification via ADK Web UI

Before running batch evaluations, verify that `re_opportunity_analyst` loads its `SkillToolset` (`analyze-opportunity`, `analyze-flood-risk`, `analyze-tax-implications`) and invokes `get_lot_data` and `get_tax_assessment_data` properly.

### Step 2.1: Launch the ADK Developer Web UI

```bash
# If running in Google Cloud Shell:
adk web --allow_origins "regex:https://.*\.cloudshell\.dev"

# If running locally on workstation:
adk web --port 8000
```

### Step 2.2: Run Smoke Test Prompt

1. Open the ADK Web UI in your browser (`http://localhost:8000` or Cloud Shell Web Preview on port `8000`).
2. Select the **`re_analyst`** app from the top-left dropdown.
3. Submit the test prompt:
   ```text
   Please evaluate investment suitability for lot LOT-8472 and generate a report.
   ```
4. **Verification Checklist**:
   - Inspect the **Events / Trace** tab to confirm tool calls to:
     - `load_skill` (`analyze-opportunity`, `analyze-flood-risk`, `analyze-tax-implications`)
     - `get_lot_data(plot_id="LOT-8472")`, `get_lot_data(plot_id="LOT-8471")`, `get_lot_data(plot_id="LOT-8473")`
     - `get_tax_assessment_data(parcel_apn="049-382-011-000")`
   - Confirm the final output strictly follows the 5-section structure:
     1. `# Initial Site Evaluation Memorandum: LOT-8472`
     2. `## Executive Summary`
     3. `## Acquisition Basis and Holding Costs`
     4. `## Zoning and Infrastructure Readiness`
     5. `## Risk Assessment`
     6. `## Recommendation`
5. Press `Ctrl+C` in the terminal when finished with interactive testing.

---

## Phase 3: Generate ADK Evaluation Traces (`EvalSet`)

Generate structured ADK 2.0 `EvalSet` execution traces across all 10 target lots defined in `eval_harness/hillclimb_lots.txt` (`LOT-8472`, `LOT-8471`, `LOT-8473`, `LOT-1000` through `LOT-1006`).

### Step 3.1: Execute the Dataset & Trace Builder

```bash
python3 -m eval_harness.create_re_agent_evalset \
  --lots-file eval_harness/hillclimb_lots.txt \
  --out ./re_analyst/hillclimb_lots.evalset.json \
  --agent-dir ./re_analyst
```

### Expected Output
```text
🚀 Executing 're_analyst' over 10 properties to populate evaluation traces...
-----------------------------------------------------------------
⚡ Starting concurrent execution across 10 properties (max concurrency: 10)...
✅ [1/10] Completed trace for LOT-8472
...
✅ [10/10] Completed trace for LOT-1006

Cleaning up resources...
✅ Successfully generated real estate evaluation dataset for 10 properties -> ./re_analyst/hillclimb_lots.evalset.json
```

---

## Phase 4: Execute GEAP Autoraters (Dry-Run & Multi-Pass Cloud Eval)

### Step 4.1: Validate Prompt Assembly in Dry-Run Mode

Always perform a `--dry-run` first to verify that `eval_harness/rubrics.json` (19 criteria) and the extracted agent responses compile cleanly into `PointwiseMetricPromptTemplate` prompts without incurring Vertex AI API charges:

```bash
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses_dryrun.json \
  --experiment-name rubric-agreement-check \
  --dry-run \
  --k 3
```

Verify record count (`10 responses × 19 criteria = 190 records`) and `"is_dry_run": true`.

### Step 4.2: Dispatch Live $k=3$ Multi-Pass Stochastic Cloud Evaluation

Run 3 independent stochastic evaluation passes across all 19 criteria using Vertex AI GenAI Evaluation Service (`gemini-2.5-flash` autorater judge with `sampling_count=4`):

```bash
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --location us-central1 \
  --autorater-model gemini-2.5-flash \
  --judge-model-sampling-count 4 \
  --k 3
```

> [!TIP]
> If your project has strict regional Vertex AI QPS quotas, append `--evaluation-service-qps 2.0` to throttle request dispatch safely.

---

## Phase 5: Statistical Rubric Reliability & Agreement Triage

Use the statistical analysis suite to evaluate inter-rater reliability across all 19 candidate rubrics and isolate high-signal criteria from noisy or saturated ones.

### Step 5.1: Run Global 19-Criteria Agreement Summary

To analyze either your fresh baseline (`runs/baseline/geap_calls_and_responses.json`) or the reference benchmark log (`runs/geap_calls_and_responses_for_analysis.json`):

```bash
python3 -m eval_harness.all_criteria_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json
```

### Step 5.2: Interpret Global Metrics Table

Apply a three-gate psychometric filter to select trustworthy production autoraters:

1. **Gate 1 — Eliminate Ceiling-Saturated Criteria (`sd report means == 0`)**:
   - `general-tone`, `logical-flow`, `professionalism`, and `readability` score `5.0000` on every report (`sd_report_means = 0`). Even though $\alpha = 1.0$, they fail to distinguish mediocre reports from great ones.
2. **Gate 2 — Eliminate Ambiguous / Low-Agreement Criteria ($\alpha < 0.80$)**:
   - `professional-tone` ($\alpha = 0.498$), `data-density` ($\alpha = 0.499$), `document-formatting` ($\alpha = 0.548$), `report-length` ($\alpha = 0.549$), and `vocabulary-choice` ($\alpha = 0.623$) rely on subjective adjectives (*"plenty of numbers"*, *"fancy jargon"*), causing inconsistent pass-to-pass scoring.
3. **Gate 3 — Select High-Alpha, High-Discrimination Domain Rubrics ($\alpha \ge 0.80$, `sd_report_means > 0.30`)**:
   - **`monthly-tax-holding`**: $\alpha = 0.997$, Pairwise Agreement $= 99.17\%$, `sd_report_means` $= 0.8872$
   - **`flood-zone-flagging`**: $\alpha = 0.897$, Pairwise Agreement $= 94.41\%$, `sd_report_means` $= 0.3357$
   - **`environmental-esa`**: $\alpha = 0.897$, Pairwise Agreement $= 79.10\%$, `sd_report_means` $= 0.7042$

### Step 5.3: Deep-Dive into Single Criterion $N \times k$ Grades Matrices

Inspect report-by-report and pass-by-pass score distributions for each top criterion:

```bash
python3 -m eval_harness.single_criterion_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion monthly-tax-holding

python3 -m eval_harness.single_criterion_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion flood-zone-flagging

python3 -m eval_harness.single_criterion_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion environmental-esa
```

Record the top 3 validated criteria in `best_criteria_submission.txt`:

```bash
printf '%s' 'environmental-esa,flood-zone-flagging,monthly-tax-holding' > best_criteria_submission.txt
```

---

## Phase 6: Iterative Agent Hill-Climbing Workflow

In the baseline audit, two domain-critical rubrics scored near the bottom of the 1–5 scale because the agent lacked explicit analytical procedures:
- `cost-per-acre` (Baseline Hillclimb Score: **1.0000**)
- `neighboring-flood-risk` (Baseline Hillclimb Score: **1.1000**)

Use `run_hillclimb.py` and `explain_criteria_results.py` to systematically diagnose and hill-climb both metrics to **5.0000**.

### Step 6.1: Run Hill-Climbing Iteration (`run_hillclimb.py`)

Each invocation automatically creates the next versioned output folder (`runs/hillclimb_1`, `runs/hillclimb_2`, `runs/hillclimb_3`, ...), generates fresh agent traces (`re_analyst/hillclimb_X.evalset.json`), and dispatches a targeted $k=1$ GEAP evaluation on `cost-per-acre` and `neighboring-flood-risk`:

```bash
python3 -m eval_harness.run_hillclimb
```

### Step 6.2: Diagnose Low-Scoring Judge Rationales

Inspect the LLM judge's exact natural-language explanations for any property scoring $\le 4.0$:

```bash
python3 -m eval_harness.explain_criteria_results \
  --run_geap_eval_out runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json \
  --criterion cost-per-acre \
  --max_score_to_show 4.0

python3 -m eval_harness.explain_criteria_results \
  --run_geap_eval_out runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json \
  --criterion neighboring-flood-risk \
  --max_score_to_show 4.0
```

#### Root Causes Identified from Judge Explanations
1. **`cost-per-acre` Root Cause**: The agent reported `last_sale_price` and `acreage` separately, but never computed `Last sale price / Acreage`, never showed the mathematical derivation formula, and did not highlight cost-per-acre as a core valuation metric in the Executive Summary and Acquisition Basis sections.
2. **`neighboring-flood-risk` Root Cause**: `get_lot_data` returns an `adjacent_parcels` list (e.g., `["LOT-8471", "LOT-8473"]`), but `analyze-flood-risk/SKILL.md` never instructed the agent to call `get_lot_data` for each adjacent lot ID or analyze shared runoff, topography grade differences, soil complexes, and drainage easements.

### Step 6.3: Remediate Agent Domain Skills (`SKILL.md`)

Ensure both `re_analyst/skills/analyze-opportunity/SKILL.md` and `re_analyst/skills/analyze-flood-risk/SKILL.md` declare `adk_additional_tools: [get_lot_data]` in YAML frontmatter and provide deterministic step-by-step analytical instructions:

1. **`re_analyst/skills/analyze-opportunity/SKILL.md`**:
   - Retrieve subject parcel via `get_lot_data`.
   - Compute `Cost per acre = Last sale price / Acreage` formatted to 2 decimal places (`$X.XX/acre`) showing exact inputs and formula.
   - Highlight cost per acre in both **Executive Summary** and **Acquisition Basis and Holding Costs**.
   - Guardrail: If sale price or acreage is missing, invalid, or zero, state that calculation cannot be completed without inventing numbers.
   - Chain execution to `analyze-flood-risk` and `analyze-tax-implications`.

2. **`re_analyst/skills/analyze-flood-risk/SKILL.md`**:
   - Retrieve subject parcel via `get_lot_data` and report exact FEMA `flood_zone`.
   - Read `adjacent_parcels` and invoke `get_lot_data` **separately for EVERY neighboring lot ID**.
   - Report each neighbor's Lot ID, exact FEMA flood zone, topography, soil classification, and drainage/easement attributes.
   - Perform comparative hydrological risk analysis (upslope/downslope grade runoff potential, soil permeability, easement constraints) while distinguishing verified records from inferred hydrological implications.

### Step 6.4: Re-Run Hill-Climbing & Confirm Convergence (`hillclimb_3`)

```bash
python3 -m eval_harness.run_hillclimb
```

### Verified Final Output (`hillclimb_3`)
```text
=========================================
📊 Hill Climbing Results: hillclimb_3
=========================================
 • cost-per-acre             Mean Score: 5.0000
 • neighboring-flood-risk    Mean Score: 5.0000
=========================================
```

Confirm zero remaining failure cases ($\le 4.0$):

```bash
python3 -m eval_harness.all_criteria_analysis \
  --run_geap_eval_out runs/hillclimb_3/geap_calls_and_responses_hillclimb_3.json
```

---

## CLI Command Reference Matrix

| Module | Purpose | Key Arguments |
| :--- | :--- | :--- |
| `eval_harness.create_re_agent_evalset` | Executes ADK agent concurrently over a Lot ID list and saves `.evalset.json` | `--lots-file`, `--out`, `--agent-dir` (default `./re_analyst`), `--eval-set-id` |
| `eval_harness.run_geap_eval` | Compiles GEAP `PointwiseMetric` templates and runs dry-run or $k$-pass Vertex AI evals | `--traces`, `--rubric-file`, `--out`, `-k`/`--k-passes`, `--dry-run`, `--experiment-name`, `--location`, `--autorater-model`, `--judge-model-sampling-count`, `--evaluation-service-qps` |
| `eval_harness.all_criteria_analysis` | Computes Mean, Min/Max, Ranges, SDs, Pairwise Agreement, and Krippendorff's $\alpha$ across all criteria | `--run_geap_eval_out` |
| `eval_harness.single_criterion_analysis` | Displays detailed $N \times k$ Lot-by-Pass Grades Matrix and summary stats for one criterion | `--run_geap_eval_out`, `--criterion` |
| `eval_harness.explain_criteria_results` | Prints LLM judge scores and natural-language rationales filtered by max score threshold | `--run_geap_eval_out`, `--criterion`, `--max_score_to_show` (default `4.0`) |
| `eval_harness.run_hillclimb` | End-to-end versioned runner (`runs/hillclimb_X`) evaluating `cost-per-acre` & `neighboring-flood-risk` | `--lots-file`, `--rubrics-file`, `--agent-dir`, `--runs-dir`, `--qps`, `--autorater-model` |

---

## Troubleshooting & Operational Playbook

### 1. `pyOpenSSL` Error: `"Context has already been used to create a Connection"`
- **Cause**: Concurrent TLS context reuse when `pyOpenSSL` is injected into `urllib3` during multi-threaded Vertex AI `EvalTask` calls.
- **Resolution**: Already mitigated in `run_geap_eval.py` and `run_hillclimb.py` via:
  ```python
  os.environ["GOOGLE_API_USE_MTLS"] = "never"
  os.environ["GOOGLE_API_USE_CLIENT_CERTIFICATE"] = "false"
  ```
  and by instantiating a fresh `EvalTask` per pass. If writing custom wrapper scripts, set both environment variables **before** importing `vertexai` or `google.cloud.aiplatform`.

### 2. Vertex AI `429 Resource Exhausted` / Quota Throttling
- **Cause**: Concurrent agent runs (`max_concurrency=10`) or high-QPS GEAP judge sampling (`sampling_count=4`) exceeding regional quota in `us-central1`.
- **Resolution**:
  - Pass `--evaluation-service-qps 1.5` to `eval_harness.run_geap_eval` or `--qps 1.5` to `eval_harness.run_hillclimb`.
  - Verify `MODEL=gemini-3.5-flash` and `GOOGLE_CLOUD_LOCATION=global` in `.env` for agent generation.

### 3. `Statistical analysis is not supported for dry runs`
- **Cause**: Running `all_criteria_analysis.py` or `single_criterion_analysis.py` against a JSON file generated with `--dry-run`.
- **Resolution**: Re-run `eval_harness.run_geap_eval` **without** `--dry-run` so `responses_from_service` contains live judge scores.

### 4. `alpha` Displays `N/A [Single Pass]` or `nan`
- **Cause**: Krippendorff's Alpha requires $k \ge 2$ evaluation passes (`--k 3` recommended) and non-zero score variance across ratings. Single-pass runs (`run_hillclimb.py` uses `k=1` for rapid iteration speed) intentionally report `N/A [Single Pass]`.
- **Resolution**: Once hill-climbing reaches target mean scores, run `eval_harness.run_geap_eval --k 3` on the final `.evalset.json` to verify multi-pass inter-rater stability.

### 5. `KeyError: 'MODEL'` or `DefaultCredentialsError` on Startup
- **Cause**: Missing `.env` file in working directory or expired Google Cloud ADC token.
- **Resolution**:
  1. Ensure you run `python3 -m eval_harness...` from the repository root containing `.env`.
  2. Refresh credentials with `gcloud auth application-default login`.
