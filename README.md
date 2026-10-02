# Same Case, Different Reasons

**Measuring the Fidelity of LLM-Generated Explanations to Their XAI Evidence**

This repository accompanies the paper of the same title. It contains the full
analysis pipeline, the prompts, the per-stage provenance manifests, and the
results tables and figures needed to reproduce the study.

> **Anonymised for double-blind review.** Author names, affiliations, and
> contact details are intentionally omitted. They will be restored in the
> camera-ready version.

---

## What the study does

An explainable-AI (XAI) method such as SHAP or LIME produces a vector of signed
feature attributions. A large language model (LLM) is increasingly used to
rewrite that vector as a short natural-language explanation a non-expert can
read. This study asks whether the generated text **faithfully reports the
attributions it was given**, and whether two explanations of the same case
**cite the same reasons** — and, when they differ, **at which pipeline stage**
the difference is introduced.

Each explanation is decomposed into a set of *(feature, direction, rank)*
claims using an LLM extractor validated against two human coders
(Cohen's κ = 0.99). Content variation is then attributed to an **evidence
stage** (prediction model, attribution method) or a **generation stage** (LLM,
prompt), estimated at the case level with cluster-bootstrap intervals.

**Headline result.** Across 29,700 generated explanations, changing the XAI
method moves explanation content by δ = 0.71 and changing the model by 0.33,
whereas changing the generator moves it by 0.012 and the prompt by 0.020. LLMs
render the supplied evidence with high fidelity; the apparent instability of
their explanations is inherited from the attribution stage, not created in
generation. A constrained prompt removes the residual omission gap and erases
the differences between generators. Isolated LLM-as-a-judge quality scores are
essentially uncorrelated with content variation (ρ = 0.04).

---

## Repository layout

```
.
├── notebooks/          Analysis pipeline, run in numeric order (01–12)
├── prompts/            Version-hashed generation prompts
├── results/
│   ├── figures/        Figures (PNG + PDF, 600 dpi)
│   └── tables/         Result tables (CSV)
├── logs/               Per-notebook run records and provenance
├── data/               Manifests and configuration (see note below)
├── requirements.txt
└── LICENSE
```

### A note on `data/`

The raw public datasets, the trained models, the generated explanation corpus,
and the extracted-claim cache are **not** included here, because of their size
and the redistribution terms of the source data. What is included is enough to
audit the method: the per-stage JSON manifests, the extractor configuration and
its validation summary, the design recommendation, and the prompt texts. The
two public datasets can be obtained from their original sources (see
`notebooks/01_data_prep.ipynb`), after which the pipeline regenerates every
downstream artefact.

---

## Pipeline

The notebooks run in order; each reads the artefacts the previous ones wrote.

| # | Notebook | Purpose |
|---|----------|---------|
| 01 | `data_prep` | Load and clean the two public datasets; fixed train/test split |
| 02 | `model_training` | Train three classifiers per domain under an AUC-parity constraint |
| 03 | `attribution` | Sample cases; compute SHAP and LIME; write the evidence blocks |
| 04 | `claim_parser_dev` | Build the LLM extractor; measure its self-stability |
| 05 | `parser_validation` | Validate the extractor against human coding; decision gates |
| 06 | `power_simulation` | Pre-experiment sample-size and power check |
| 07 | `generation` | Generate the full explanation corpus (29,700) |
| 08 | `content_distance` | Extract claims; compute fidelity and pairwise distances |
| 09 | `delta_analysis` | **Main analysis** — additional variation per pipeline stage |
| 10 | `mixed_effects` | Robustness check (reported as inconclusive) |
| 11 | `quality_stability` | LLM-as-a-judge quality scoring and its blind spot |
| 12 | `fidelity_analysis` | Fidelity intervals, prompt×model interaction, natural experiment |

### Human-in-the-loop step

Between notebooks 04 and 05, a sample of explanations is coded by two
independent annotators and their disagreements adjudicated, to validate the
extractor. The coding guide used for that step is included.

---

## Reproducing the study

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Provide an API key for the generation and extraction steps via a local `.env`
file (not committed):

```
OPENAI_API_KEY=...
```

Then run the notebooks in order. LLM calls are cached per condition, so an
interrupted run resumes without re-issuing completed calls, and a repeated run
regenerates nothing.

---

## Conventions

- All random operations use a fixed seed.
- Figures are grayscale, saved as PNG and PDF at 600 dpi; models are
  distinguished by line style and shading rather than colour.
- Every LLM call is cached by a key over all factors that could change its
  output; model identifiers, prompt hashes, and per-stage counts are logged.

---

## License

Released under the MIT License. See [`LICENSE`](LICENSE).
