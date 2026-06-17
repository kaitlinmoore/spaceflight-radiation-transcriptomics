# CLAUDE.md — Spaceflight Radiation Transcriptomics

## What this is
Train an ML model to predict ionizing-radiation exposure from open NASA GeneLab
mouse bulk RNA-seq (ground-irradiation studies), evaluate it cross-study, then
transfer it to spaceflight datasets to estimate the radiation-attributable
fraction of the flight transcriptional signature. Stretch: interpretability
layer, additional analog arms (unloading/CO2), tissue/ion resolution.

## Standing principles (do not violate)
- Defensible without AI assistance. The owned core — gene-ID harmonization, the
  leave-one-dataset-out split, the normalization choice, the radiation-axis
  projection, the detection-floor estimate — must be something I can rebuild from
  scratch and explain at a whiteboard. AI-assisted code is fine, but every
  non-trivial piece comes with a plain-language explanation in comments or
  docs/methodology.md.
- Explain as you build. For each transform, model, and metric, record the
  alternatives considered and why this choice won.
- Honest framing. Never claim more than the code supports. Separate what I
  implemented from what a library does, flag AI-assisted sections, state
  limitations plainly. The transfer result is reported with its detection floor;
  a null is reported as a null, not buried.
- Plans before execution. Propose a plan and wait for approval before large
  changes, restructures, or deleting work. I decide.
- Tests where logic is testable. The API parsing, gene-ID harmonization, counts
  assembly, and LODO split logic get pytest tests against committed synthetic
  fixtures. Test the seams I own, not sklearn/torch internals.

## Repo conventions
- Keep raw and derived data out of git. Commit code, configs, synthetic
  fixtures, and docs only.
- README carries exact OSD accessions, API endpoints, gene-ID namespace and
  annotation version per dataset, and access dates.
- docs/methodology.md explains approach, choices, and evaluation.

## Annotation & gene-ID hygiene   [replaces genome-build hygiene]
- Counts have no CHROM:POS join, so the build hazard is different but real:
  datasets may be processed against different mouse annotations (GRCm39 vs
  GRCm38/mm10) and exported in different gene-ID namespaces (Ensembl ENSMUSG,
  Entrez, MGI symbol).
- Detect and record the gene-ID namespace and annotation version per dataset
  from file/assay metadata. Do not assume.
- Harmonize to a single namespace (prefer stable Ensembl gene IDs) before
  merging counts. Never merge on gene SYMBOL across datasets silently — symbols
  are non-unique and version-drifting.
- Report the gene drop rate from harmonization. Silent gene loss is the omics
  analog of the cross-build join bug; log it the way the VCF repo logs liftover
  drop rate.

## Project-specific instructions
- Evaluation: leave-one-dataset-out is the headline. A sample from a held-out
  study must NEVER appear in training. splits.py enforces this; test it.
- Dose-study confound (gate before modeling): confirm from the census that
  dose/ion varies WITHIN studies, not only across them. If dose is confounded
  with study, a dose model is partly a study classifier and batch correction
  removes the signal. Fallback: model exposure-vs-control within study, or ion
  class, instead of continuous dose. Decide from the census; do not assume.
- Batch correction is double-edged: correcting study effects can erase biology
  confounded with study. Report results with AND without correction; treat the
  divergence as a finding, not noise.
- Transfer honesty: ground doses (~0.1-1.0 Gy) are ~100-1000x the flight LEO
  dose (~mGy). Predicting a flight "dose" is extrapolation below the training
  range and is NOT a defensible point estimate. Frame transfer as projection
  onto the learned radiation axis and as an attributable-fraction estimate
  bounded by an explicit detection floor. Report the floor; if flight sits below
  it, say so plainly.
- Small-n / high-dim: hundreds of samples, ~tens of thousands of genes.
  Regularized linear (elastic net) and tree baselines first; justify any deep
  model against those baselines. No deep model for its own sake.

## Environment
Run under WSL2 (Ubuntu). uv-managed env (pyproject.toml + uv.lock), Python 3.11.
Python deps: requests, pandas, numpy, scikit-learn, statsmodels, pydeseq2
(normalization), inmoose (ComBat-seq batch correction), matplotlib, seaborn,
pytest. All ship Linux wheels on PyPI; no conda.