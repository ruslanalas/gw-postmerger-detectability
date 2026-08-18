# Reproduction Guide

This project's notebooks must be run **in order**. Later notebooks depend on artifacts saved by earlier
ones (e.g. the estimated PSD, the raw noise pools, the frozen calibration values) — they are not
independent scripts that can be run in any sequence or skipped.

## Notebook order and purpose

| Order | Notebook | Purpose |
|---|---|---|
| 1 | `Notebook1_Data_PSD_Whitening.ipynb` | Fetches real LIGO L1 data (via GWOSC data-quality segments, not blind GPS offsets), estimates the PSD, and builds/validates the whitening pipeline. |
| 2 | `Notebook2_Injection_SNR_CQT.ipynb` | Damped-sinusoid signal generator; matched-filter SNR definition and validation (Test A: synthetic Gaussian control; Test B: real-noise diagnostic); injection machinery; CQT representation. |
| 3 | `Notebook3_Graph_Detectability_PhaseDiagram.ipynb` | Builds the time-frequency graph and the `norm_lambda1` detectability statistic; empirical null calibration; first-pass (SNR, τ) phase diagram; power-vs-statistic correlation checks. |
| 3.5 | `Notebook3_5_Confirmation.ipynb` | Confirmation-only pass at a larger sample size: reproduces Notebook 3's phase diagram, adds category-wise and fixed-SNR correlation diagnostics, and the tile-position shuffle test. No new methodology introduced. |
| 4 | `Notebook4_Stage2_Damping.ipynb` | Stage-2 development: the nonlinear LSQ and ridge/energy-decay $\hat\tau$ estimators, the calibrated power baseline, and in-sample discrimination results. |
| 4.5 | `Notebook4_5_HeldoutValidation.ipynb` | Splits the training noise pool into `stage2_calib`/`stage2_val`; fits every Stage-2 threshold/classifier on `stage2_calib` only and evaluates, without refitting, on `stage2_val`. This is the required prerequisite before freezing the design. |
| 5 | `Notebook5_Final_TestBlock_Evaluation.ipynb` | **The one-time final evaluation.** Loads `test_block` for the first and only time in the entire project, applies `frozen_config.json` exactly, and reports the final results. |
| 6 | `Notebook6_Statistical_Reporting.ipynb` | Statistical packaging only — **no new experiments**. Loads Notebook 5's already-saved results and adds Wilson 95% confidence intervals and the final paper tables. |

## Governing rules (enforced in code, not just convention)

1. **`test_block` must not be loaded before Notebook 5.** Notebooks 1–4.5 do not load it at all; several
   notebooks assert on the block name to make this a hard failure rather than a silent possibility.
2. **`frozen_config.json` governs the final evaluation exactly.** Notebook 5 re-derives every calibration
   quantity (Stage-1 threshold, ridge noise floor, power-baseline operating threshold) from
   `calib_block`/`stage2_calib` and **asserts** the result matches `frozen_config.json` before proceeding
   — a mismatch stops the notebook rather than silently continuing.
3. **No threshold may be changed after Notebook 5 has run.** If `test_block` performance differs from the
   Notebook 4.5 held-out validation, that difference is reported as-is; it does not trigger a return to
   tuning, in this project or in any reproduction of it.
4. **`Notebook6` performs zero new experiments.** It only re-summarizes already-saved Notebook 5 output.
   If you want to reproduce Notebook 6's tables, you must have already run Notebook 5 (or have its saved
   `nb5_artifacts/` available).

## Running the notebooks

Each notebook is self-contained (Colab-runnable top to bottom) and begins with its own `!pip install`
cell for anything beyond the base scientific-Python stack. See `requirements.txt` /
`environment.yml` if running locally instead of on Colab.

Artifacts (saved `.npz`/`.json`/plot files) from each notebook should be placed in the matching
`artifacts/nbX_artifacts/` folder. Large raw-array files (e.g. cached raw noise pools) are better hosted
on the project's Zenodo archive than committed to Git history directly — see the note in the top-level
`README.md` and the `.gitignore` policy comment.

## Runtime expectations

No GPU is required anywhere in this project. Notebooks 3, 3.5, 4.5, and 5 are the most
compute-intensive, due to their calibration sample sizes and (SNR, τ) sweep grids — each notebook's own
configuration cell exposes the relevant sample-size parameters (`N_CALIBRATION`, `N_PER_CELL`, etc.) if
you need to trade off runtime against statistical precision for a quick local test run.
