# Same Case, Different Reasons

Decomposing Content Variability in LLM-Generated Explanations

This project examines whether an LLM consistently presents the same core reasons when repeatedly generating explanations for the same case based on XAI evidence. It also decomposes this variability to identify the stages of the pipeline at which it arises.

---

## Repository Structure

```text
project/
├── data/                  Raw, intermediate, and analysis-ready data (excluded from version control)
├── notebooks/             11 analysis notebooks
├── prompts/               Version-controlled prompts
├── logs/                  Execution records, hyperparameters, and failure logs
├── results/
│   ├── figures/           PNG and PDF, 600 dpi
│   └── tables/            CSV
├── CODING_GUIDE.md        Manual annotation guidelines
├── requirements.txt
├── .env.example
└── README.md
```

---

## Environment Setup

```bash
conda create -n xai-stability python=3.11 -y
conda activate xai-stability

pip install -r requirements.txt
conda install -c conda-forge llvm-openmp -y      # Required on macOS

python -m ipykernel install --user --name xai-stability --display-name "XAI Stability"
```

Select **XAI Stability** as the kernel in Jupyter.

### Version Constraints — Do Not Relax Without Verification

| Package | Constraint | Reason |
|---|---|---|
| `numpy` | `<2.4` | The numba dependency used by shap does not support NumPy 2.4. |
| `lightgbm` | `>=4.5` | Earlier versions pass arguments that were removed in scikit-learn 1.6. |

### macOS

XGBoost and LightGBM require the OpenMP runtime. On conda-forge, the package is named **`llvm-openmp`**, not `libomp`.

```bash
conda install -c conda-forge llvm-openmp
```

### API Keys

```bash
cp .env.example .env
```

Edit `.env` to add your API keys. The notebooks load them using `python-dotenv`. Do not hard-code API keys in notebooks, as the code is intended for public release.

---

## Data

Place the following two public datasets in `data/`.

| File Name | Source | Number of Rows |
|---|---|---:|
| `credit_risk_dataset.csv` | [Kaggle: Credit Risk Dataset](https://www.kaggle.com/datasets/laotse/credit-risk-dataset) | 32,581 |
| `telco_churn.csv` | [Kaggle: Telco Customer Churn](https://www.kaggle.com/datasets/blastchar/telco-customer-churn) | 7,043 |

The original Telco file is named `WA_Fn-UseC_-Telco-Customer-Churn.csv` and should be renamed. Alternatively, update the `DATASETS` configuration in Notebook 01 to match your file names.

**Important:** Several datasets on Kaggle are named “credit risk dataset.” Use the version published by `laotse`. The correct dataset has a `loan_status` target encoded as `0` and `1`. If the target contains text labels such as `Fully Paid`, you have a different dataset.

---

## Execution Order

| # | Notebook | Purpose | Main Output |
|---|---|---|---|
| 01 | `data_prep` | Preprocessing and train/test split | `feature_registry.json` |
| 02 | `model_training` | Train three models and establish AUC equivalence | `model_*.joblib` |
| 03 | `attribution` | Stratified case sampling and SHAP/LIME computation | `evidence_blocks.json` |
| 04 | `claim_parser_dev` | Pilot generation, extractor development, and stability assessment | `manual_coding_sheet.csv` |
| 05 | `parser_validation` | Validation against human annotations and gate assessment | `validation_summary.json` |
| 06 | `power_simulation` | Preliminary sample-size assessment | `design_recommendation.json` |
| 07 | `generation` | Main experiment: approximately 33,600 generations | `generation_corpus.parquet` |
| 08 | `content_distance` | Claim extraction and pairwise distance computation | `content_distances.parquet` |
| 09 | `delta_analysis` | **Primary analysis:** excess variability relative to baseline | `delta_manifest.json` |
| 10 | `mixed_effects` | Robustness checks | `robustness_manifest.json` |
| 11 | `quality_stability` | RQ5: Blind spots in quality metrics | `quality_scores.parquet` |

### Human Annotation Step

**Human annotation** is required between Notebooks 04 and 05. This involves reading explanation sentences and recording their content in a table—not writing code. Provide annotators with `CODING_GUIDE.md`.

```text
1. Notebook 04 generates data/manual_coding_sheet.csv.
2. Two annotators independently complete the coder1_* and coder2_* columns.
3. Run Notebook 05 to generate data/adjudication.csv.
4. Adjudicate disagreements.
5. Rerun Notebook 05 to assess whether the validation gate is passed.
```

**Do not run Notebook 07 unless the validation gate in Notebook 05 has been passed.**

Annotators do not need domain expertise. English reading comprehension and the ability to follow the guidelines are sufficient; a 30-minute training session can serve as an initial introduction. Automated extraction results are stored separately in `coding_auto_reference.csv` and are not shown to annotators.

---

## Claim Extractor

The extractor is **LLM-only**. A rule-based parser was initially used alongside it but was subsequently abandoned.

One-hot-encoded feature names such as `Contract_Two_year` appear in explanations as natural-language expressions such as “a two-year contract.” Capturing these expressions with rules required repeated adjustments to dataset-specific synonym dictionaries based on the pilot data. These adjustments effectively **tailored the tool to the data on which it would later be validated**. Even with these adjustments, recall remained at roughly one-third of the supplied features. The LLM extractor identifies nearly all of them without such tuning.

The trade-off is the **loss of fully deterministic reproducibility**. Even at temperature 0, an LLM does not guarantee identical outputs. Because this study concerns reproducibility, we address this limitation in two ways:

1. **Frozen extraction cache** — `data/claims_cache/`. Once claims have been extracted, subsequent runs reuse the same results. Releasing this cache allows third parties to reproduce the analyses using identical claim data.
2. **Measurement of the extractor’s own reproducibility** — Notebook 04 repeats extraction three times for each of 50 items and computes feature agreement and direction agreement (`tab09_extractor_stability`). Notebook 08 treats this estimate as a **lower bound on measurement error** and reports it alongside the observed repeated-generation baseline (`tab37_baseline_versus_measurement_error`).

The second measure is particularly important. If the observed baseline is comparable to the extractor’s own instability, the variability cannot be attributed to the generator. Notebook 08 automatically flags this situation.

---

## Conventions

### Figures

- Use a grayscale seaborn style, without captions embedded in the figures.
- Save both PNG and PDF versions at 600 dpi.
- Use the naming pattern `fig{number}_{description}`.
- Distinguish models by line style rather than color to support black-and-white printing.

### Tables

- Save as CSV with `utf-8-sig` encoding.
- Use the naming pattern `tab{number}_{description}`.

### Caching

- Save LLM outputs to files for each experimental condition; skip calls when cached outputs already exist.
- Editing a prompt changes its hash and triggers new generation.
- To regenerate outputs, delete the corresponding block directory.

### Reproducibility

- Fix the random seed at 42.
- Record model and LLM versions, temperature, and execution timestamps in `logs/`.

---

## Experimental Blocks

| Block | Factors Varied | Number of Generations |
|---|---|---:|
| A | Predictive model × XAI method | 5,400 |
| A' | Same as A, in the churn domain | 5,400 |
| B | LLM × prompt | 10,800 |
| C | Direct comparison of the two effects | 4,800 |
| D | No changes to conditions (τ = 1.0) | 3,600 |
| D-0 | No changes to conditions (τ = 0) | 3,600 |
| **Total** | | **33,600** |

Notebook 07 runs the blocks in the following order: D → D-0 → A → A' → B → C. Lower-cost blocks are run first to detect pipeline issues early.

---

## Core Analytical Framework

**Baseline:** The content distance between explanations generated repeatedly without changing any experimental condition.

**Excess variability (Δ):** The additional content distance, relative to baseline, when one condition is changed.

```text
ΔD = D(changed condition) − D(repeated-generation baseline)
```

Δ is **computed at the case level, after which its distribution is examined**. It is not calculated by subtracting overall means. Confidence intervals are obtained using a cluster bootstrap with cases as the resampling unit.

This framework avoids the need for an absolute acceptance threshold, such as a specified percentage, because conclusions are based on relative comparisons.

---

## Important Caveats

- **Extractor validity is foundational to the study.** Do not proceed if κ in Notebook 05 fails to meet the required threshold.
- **The extractor’s own instability sets the baseline noise floor.** Check `tab37` in Notebook 08. If the reported ratio is below 2, explicitly state that the generator’s contribution cannot be isolated.
- **Cases with no shared features, for which distance is undefined,** are automatically excluded, even though they represent the most strongly divergent cases. Report the missingness rate alongside the mean distance.
- **Negative Δ values are not errors.** They can occur under conditions that suppress variability, such as constrained prompts. Do not truncate them; report their proportion.
- **The comparison domain is used only in Block A.** The study therefore cannot establish cross-domain generalizability of LLM or prompt effects.
- Notebook 10 is a supplementary analysis. If it fails to converge, report that failure. Notebook 09 provides the primary results.
- In the paper, **distinguish validity from reliability**. Notebook 05 measures whether the extractor interprets text like a human annotator (validity); Notebook 04 measures whether it interprets the same text consistently across repeated readings (reliability).
