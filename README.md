# spaceflight-radiation-transcriptomics

Radiation-attribution modeling of NASA GeneLab mouse spaceflight transcriptomes: train a model to predict ionizing-radiation exposure from ground-irradiation RNA-seq, validate it cross-study, then transfer it to spaceflight data to estimate the radiation-attributable fraction of the flight transcriptional signature.

---

## Project status

This README is the working scaffold. Items marked **TODO** depend on the data census/download and are completed locally (the corpus can only be pulled on WSL2; see [Data sources](#data-sources)).

- [ ] Run the corpus census; record selected OSD accessions, ions/doses, tissues, strains, platforms ([Data sources](#data-sources))
- [ ] Resolve the **dose–study confound gate**: does dose vary *within* studies, or only across them? ([Data sources](#data-sources))
- [ ] Fill per-dataset gene-ID namespace + annotation version
- [ ] Fill LODO and within-study results ([Results](#results--eda-summary))
- [ ] Fill transfer attribution result + detection floor ([Results](#results--eda-summary))
- [ ] Mark which modules ended up AI-assisted vs owned ([AI assistance disclosure](#ai-assistance-disclosure))

---

## Problem / question

Spaceflight is not a single stressor — it is a confound bundle of ionizing radiation, microgravity/mechanical unloading, elevated CO₂, altered circadian/light cycles, and launch/reentry stress. A naïve "flight vs. ground" transcriptomic model attributes the whole bundle to one cause. This project isolates the **radiation** component by training on ground-based irradiation experiments (where dose and ion are controlled) and asking how much of the spaceflight signature that radiation model accounts for.

The deliverable is honest multi-study omics engineering, not a differential-expression re-run:

- a **census** of the open mouse RNA-seq corpus by stressor, ion/dose, tissue, strain, platform, and replicate count;
- a **ground radiation prediction model** evaluated **leave-one-dataset-out** (cross-study), not within-study;
- a **transfer/attribution** step that projects spaceflight samples onto the learned radiation axis and reports a radiation-attributable fraction **bounded by an explicit detection floor**.

A well-characterized null — *the flight radiation signal sits below the ground-trained model's detection floor, and the flight signature is dominated by non-radiation components* — is a complete, expected, and defensible result.

> **Scope note:** open-access *Mus musculus* bulk RNA-seq only. No IRB, so human data is out of scope. No new data is generated; this is archival reuse of experiments NASA has already run and published.

---

## Data sources

All data is **gitignored** and must be downloaded locally. The container/CI never holds raw data. Record exact provenance below as you pull it.

**Access surfaces:**

| Surface | URL | Use |
|---|---|---|
| OSDR data repository (browse/search) | https://osdr.nasa.gov/bio/repo | Find studies, confirm open vs. controlled access |
| OSDR biodata API (REST + query) | https://visualization.osdr.nasa.gov/biodata/api/v2/ | Programmatic metadata + counts; merges unnormalized counts across datasets |

**Selected datasets** (fill from the census — exclude legacy microarray; confirm every study is open-access, no lock icon):

| OSD accession | Arm (ground irradiation / spaceflight) | Ion / dose | Tissue | Strain | Platform | Gene-ID namespace | Annotation version | Access date |
|---|---|---|---|---|---|---|---|---|
| _TODO_ | _TODO_ | _TODO_ | _TODO_ | _TODO_ | RNA-seq | _TODO (e.g. Ensembl ENSMUSG)_ | _TODO (e.g. GRCm39)_ | _TODO_ |
| _TODO_ | _TODO_ | _TODO_ | _TODO_ | _TODO_ | RNA-seq | _TODO_ | _TODO_ | _TODO_ |

> **TODO — dose–study confound gate (resolve before any modeling).**
> From the census, state whether radiation dose/ion varies **within** individual ground studies (dose-response designs) or **only across** studies.
> - If dose varies within studies → continuous-dose regression is viable.
> - If dose is confounded with study → a dose model is partly a study classifier and batch correction will erase the signal; fall back to exposure-vs-control or ion-class classification.
> Record the decision and the datasets that carry within-study dose variation here: _TODO_

> **TODO — dose ranges for the detection-floor analysis.** Record the actual ground dose range (Gy) and the flight LEO dose range (mGy): _TODO_

---

## Environment & setup

Runs under WSL2 (Ubuntu). [uv](https://docs.astral.sh/uv/)-managed environment, Python 3.11.

```bash
# from the repo root, inside WSL2
uv sync                 # reproduces the env from pyproject.toml + uv.lock
```

Dependencies (all ship Linux wheels on PyPI; no conda): `requests`, `pandas`, `numpy`, `scikit-learn`, `statsmodels`, `pydeseq2`, `inmoose` (ComBat-seq), `matplotlib`, `seaborn`, `pytest`.

> **TODO — confirm the ComBat-seq import path in `inmoose`** and pin versions once `uv.lock` is committed.

---

## How to run

Intended sequence (exact flags settle as the CLI is built; see TODOs):

```bash
# 1. Census: enumerate the open mouse RNA-seq corpus
uv run python -m src.census --organism "Mus musculus" --assay rna-seq --out data/census.csv

# 2. Assemble: harmonize gene IDs and merge per-study counts into one matrix
uv run python -m src.assemble_counts --arm ground --out data/ground_counts.parquet
uv run python -m src.assemble_counts --arm flight --out data/flight_counts.parquet

# 3. Normalize
uv run python -m src.normalize --in data/ground_counts.parquet --method vst --out data/ground_norm.parquet

# 4. Train + evaluate the ground radiation model (leave-one-dataset-out)
uv run python -m src.train_radiation --in data/ground_norm.parquet --out models/
uv run python -m src.evaluate --model models/ --in data/ground_norm.parquet --out results/lodo/

# 5. Transfer: project flight samples onto the radiation axis
uv run python -m src.transfer --model models/ --flight data/flight_counts.parquet --out results/transfer/
```

> **TODO — finalize CLI flags** once each module's interface is implemented, and update this block to match.

---

## Results / EDA summary

> **TODO — fill after the runs.** Keep commentary honest and calibrated; report nulls as nulls.

- **Corpus census:** _TODO — summary table / counts by arm, ion/dose, tissue, strain; figure in `notebooks/census.ipynb`._
- **Batch & factor structure:** _TODO — count-depth distribution, study/batch separation, factor balance; `notebooks/eda.ipynb`._
- **Ground model — leave-one-dataset-out (headline):** _TODO — metric(s)._
- **Ground model — within-study (optimistic ceiling, labeled as such):** _TODO — metric(s)._
- **Batch correction sensitivity:** _TODO — results with vs. without correction; report the divergence as a finding._
- **Transfer / attribution:** _TODO — where flight samples land on the radiation axis, the radiation-attributable fraction, and the detection floor. State plainly whether flight sits above or below the floor._
- **Figures:** _TODO — `notebooks/results.ipynb`._

---

## Methodology

See [`docs/methodology.md`](docs/methodology.md) for the full write-up: normalization choice, the leave-one-dataset-out rationale, the dose–study confound and how the census resolved it, the batch-correction double-edge, the transfer extrapolation caveat and detection floor, and limitations.

---

## Limitations & honest scope

- **Small per-study n.** Individual missions have few flight animals; statistical power comes only from pooling across studies, which introduces batch structure.
- **Pooled batch effects.** Cross-study pooling confounds biology with study/lab/platform. Results are reported with and without batch correction; correction can remove biology that is confounded with study.
- **Transfer is out-of-distribution.** The ground model is applied to a condition (spaceflight) it was not trained on, and the flight LEO dose (~mGy) is far below the ground training range (~0.1–1.0 Gy). The transfer output is a floor-bounded attribution along the learned radiation axis, **not** a flight-dose measurement.
- **RNA-seq only.** Legacy microarray datasets are excluded for cross-study comparability rather than merged.
- **Mouse only.** No cross-species claim. Single-organism by design.

---

## AI assistance disclosure

The owned, whiteboard-defensible core — gene-ID harmonization, the leave-one-dataset-out split and its leakage guard, the normalization choice, the radiation-axis projection, and the detection-floor estimate — is implemented and explainable from scratch. Library code (API transport, DESeq2-style normalization math, ComBat-seq, scikit-learn estimators) is used as such and labeled.

> **TODO — mark which modules ended up AI-assisted** (e.g., drafted with Claude Code) vs. written/re-derived by hand, per file or per function.

---

## License

MIT. See [`LICENSE`](LICENSE).
