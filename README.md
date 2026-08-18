# Detectability-Aware Screening of Post-Merger GW Damping Anomalies

**A Graph-Spectral Approach with Empirical Null Calibration**

This repository contains the full code, notebooks, frozen configuration, and manuscript for a
proof-of-concept pipeline that screens synthetic post-merger gravitational-wave-like transients,
injected into real LIGO Livingston (L1) noise, for anomalously short damping times.

> **This is a methodology proof-of-concept, not a detection claim.** No gravitational-wave signal has
> been detected. No evidence for dark matter or dark-sector physics is presented or claimed anywhere in
> this work. See [Limitations](#limitations) below and `docs/notes_on_limitations.md` for the full,
> explicit list of what this project does *not* claim.

---

## Plain-language summary

Neutron star mergers may briefly leave behind a hot, rapidly oscillating remnant before collapsing to a
black hole or settling into a stable star. If this remnant loses energy faster than expected — for
example, through exotic (e.g. dark-sector) physics — its gravitational-wave "ringing" might damp out
unusually quickly. Finding such a signal in real detector noise is hard, partly because short,
high-frequency transients are easy to confuse with instrumental glitches, and partly because real
detector noise does not behave like the clean, idealized noise most detection-threshold formulas assume.

This project builds and rigorously validates a two-stage screening pipeline for a simplified
("toy") version of that signal, injected directly into real LIGO data, with every threshold calibrated
empirically on real noise and frozen *before* a single final, held-out evaluation.

## Scientific motivation

Post-merger gravitational-wave emission from binary neutron star coalescences may probe matter at
extreme densities, and deviations in the effective damping time have been proposed as phenomenological
probes of non-standard energy-loss channels. This project does not model that physics in detail — it
uses a single-mode damped sinusoid as a deliberately minimal proxy signal — and instead focuses on
building and validating the *statistical methodology* needed to screen for such anomalies in real,
non-Gaussian, non-stationary detector noise. See the manuscript (`manuscript/main.tex`) for the full
scientific framing, related work, and citations.

## Pipeline overview

```
real LIGO L1 noise
  -> PSD estimation / whitening
  -> toy damped-sinusoid injections, h(t) = A exp(-t/tau) sin(2*pi*f_peak*t + phi)
  -> Stage 1: graph-spectral detectability gate  (score: norm_lambda1)
  -> Stage 2: damping-time scoring                (LSQ tau_hat, ridge/energy-decay tau_hat)
  -> frozen design (frozen_config.json)
  -> one-time held-out test_block evaluation
```

**Stage 1 — graph-spectral detectability gate.** Each candidate window's whitened Constant-Q
representation is turned into a local time-frequency coherence graph; the leading eigenvalue of that
graph (`norm_lambda1`) is used as a detectability statistic, with its null distribution calibrated
empirically on real L1 noise rather than assumed from a closed-form Gaussian model (validated directly:
real noise does *not* satisfy that assumption — see Notebook 2).

**Stage 2 — damping-time scoring.** For candidates that pass Stage 1, an effective damping time
$\hat\tau$ is estimated two independent ways — a classical nonlinear least-squares fit (primary) and an
interpretable, time-frequency-native ridge/energy-decay fit (secondary) — and compared against a
threshold calibrated on the standard ($\tau=5$–$40$~ms) damping-time manifold. A calibrated total-power
logistic-regression baseline is included for comparison.

Every threshold, model, and hyperparameter governing the final evaluation is recorded exactly in
[`frozen_config.json`](frozen_config.json) and was fixed *before* the held-out test segment was ever
loaded.

## Final frozen `test_block` results

Evaluated once, after the design was frozen, on a held-out real-noise segment untouched during any prior
development (see `docs/data_split.md`).

**Stage-1 pass rate:** 1623/1800 = 90.2% (see `docs/data_split.md` and Notebook 5 for the per-$\tau$
breakdown — this rate is strongly $\tau$-dependent, not uniform).

| Method | Conditional TPR | Conditional FPR | AUC | Notes |
|---|---|---|---|---|
| Nonlinear LSQ (primary) | 0.879 | 0.012 | 0.943 | Strongest Stage-2 discriminator |
| Ridge/energy-decay (secondary) | 0.486 | 0.016 | 0.930 | Strong ranking separation, lower operating-point recall than LSQ |
| Power (calibrated logistic, baseline) | 0.054 | 0.010 | 0.726 (AUC(-power)) | Weaker, direction-sensitive; poor recall at the strict operating point despite moderate AUC |

Full results, including end-to-end metrics, per-$\tau$ breakdowns, and Wilson 95% confidence intervals,
are in the manuscript (Tables I–II) and in `artifacts/nb5_artifacts/` / `artifacts/nb6_artifacts/`.

## Repository structure

```
gw-postmerger-detectability/
├── README.md                    # this file
├── LICENSE
├── CITATION.cff
├── requirements.txt
├── environment.yml              # optional conda equivalent
├── frozen_config.json           # canonical, machine-readable frozen design (single source of truth)
├── PROJECT_SPEC.md              # full project history, terminology, forbidden claims, known issues
│
├── notebooks/                   # run in this order -- see docs/reproduction.md
│   ├── Notebook1_Data_PSD_Whitening.ipynb
│   ├── Notebook2_Injection_SNR_CQT.ipynb
│   ├── Notebook3_Graph_Detectability_PhaseDiagram.ipynb
│   ├── Notebook3_5_Confirmation.ipynb
│   ├── Notebook4_Stage2_Damping.ipynb
│   ├── Notebook4_5_HeldoutValidation.ipynb
│   ├── Notebook5_Final_TestBlock_Evaluation.ipynb
│   └── Notebook6_Statistical_Reporting.ipynb   # stats/table packaging only, no new experiments
│
├── artifacts/                   # saved notebook outputs (npz/json/pdf) -- see note below
│   ├── nb1_artifacts/ ... nb6_artifacts/
│
├── manuscript/
│   ├── main.tex
│   └── figures/                 # the 8 figure PDFs used in the manuscript
│
└── docs/
    ├── reproduction.md          # exact notebook order, dependencies, and frozen-design rules
    ├── data_split.md            # the four-block split, and exactly what each block may/may not touch
    └── notes_on_limitations.md  # full, explicit list of this project's limitations and non-claims
```

**Note on `artifacts/`:** these folders should contain each notebook's saved `.npz`/`.json`/plot outputs.
If any folder is large (the raw noise-pool `.npy` arrays in particular can be sizeable), see the policy
in `docs/reproduction.md` — large raw arrays belong on the Zenodo archive, not in the Git history.

## How to reproduce

See [`docs/reproduction.md`](docs/reproduction.md) for the full walkthrough. In short: run the notebooks
in the numbered order above, in Google Colab or a local Jupyter environment with the packages in
`requirements.txt` installed. **`test_block` must never be loaded before Notebook 5**, and no threshold
in `frozen_config.json` may be changed after Notebook 5 has run — both rules are enforced in code
(assertions), not just convention.

## Requirements

See [`requirements.txt`](requirements.txt). Core dependencies: `numpy`, `scipy`, `matplotlib`,
`scikit-learn`, `librosa`, `gwpy`, `gwosc`.

## Runtime notes

All notebooks were developed and run on Google Colab (standard CPU runtime, no GPU required). Notebook 3
(null calibration + phase-diagram sweep) and Notebook 4.5/5 (calibration re-derivation + evaluation
sweep) are the most compute-intensive, each taking on the order of several minutes to tens of minutes
depending on the exact sample-size settings used (`N_CALIBRATION`, `N_PER_CELL` — see each notebook's own
config cell).

## Data source

All real detector data is LIGO Livingston (L1) strain, fetched at runtime from the
[Gravitational Wave Open Science Center](https://www.gw-openscience.org) via `gwpy`/`gwosc` — no raw
strain data is stored in this repository.

## Manuscript

The full manuscript, including complete methodology, related-work discussion, and limitations, is in
[`manuscript/main.tex`](manuscript/main.tex) (compile with `revtex4-2`, `aps`, `prd` style). A compiled
PDF will be added here once available.

## Zenodo archive

A tagged release of this repository is archived on Zenodo:
**DOI: [10.5281/zenodo.21993621](https://doi.org/10.5281/zenodo.21993621)**

## Citation

If you use this code or methodology, please cite via [`CITATION.cff`](CITATION.cff), or:

```bibtex
@misc{alaskarov2026detectability,
  author = {Alaskarov, Ruslan},
  title  = {Detectability-Aware Screening of Post-Merger Gravitational-Wave Damping Anomalies
            in Real Detector Noise: A Graph-Spectral Approach with Empirical Null Calibration},
  year   = {2026},
  doi    = {10.5281/zenodo.21993621},
  url    = {https://github.com/ruslanalas/gw-postmerger-detectability}
}
```

## Limitations

Full details in [`docs/notes_on_limitations.md`](docs/notes_on_limitations.md). In brief:

- The injected signal is a **single-mode toy waveform**, not a physically complete post-merger model.
- **LIGO Livingston (L1) only** — no multi-detector coincidence.
- Reported false-positive rates are **finite-sample false-flag rates on constructed windows**, not a
  production search's false-alarm rate calibrated over months/years of background.
- $\hat\tau$ is validated as an **anomaly-ranking score**, not as a precise point estimate of the true
  physical damping time.
- **No learned-classifier (CNN/AST) baseline** is included in this version.
- Stage 1's statistic is validated as sensitive to local time-frequency organization beyond total power
  — it is **not** validated as a measure of physical post-merger coherence specifically.
- **No dark matter or dark-sector detection claim** is made anywhere in this work.

## Related, earlier prototype

An earlier version of this project explored a frozen pretrained-audio-transformer approach. That
framing was abandoned after technical review (see the manuscript's Introduction for why) and is preserved
only for historical transparency at
[`gw-postmerger-ast`](https://github.com/ruslanalas/gw-postmerger-ast) — **it is not the current or
recommended version of this project.**

## License

See [`LICENSE`](LICENSE) (MIT).

## Contact

Ruslan Alaskarov — ruslanalas@gmail.com — [ORCID: 0009-0006-1030-3031](https://orcid.org/0009-0006-1030-3031)
