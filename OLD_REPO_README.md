# ⚠️ Archived / Superseded Prototype

**This repository is preserved for historical transparency only. It is not the current or recommended
version of this project, and its results should not be cited.**

## What this was

This repository contains an early prototype exploring a **frozen, pretrained audio-spectrogram
transformer (AST)** as a time-frequency feature extractor, combined with a two-stage Mahalanobis
metric-learning framework, for post-merger gravitational-wave anomaly screening.

## Why it was abandoned

On technical review, this approach was found to have several serious issues:

- A manuscript/code mismatch (the frozen AST backbone was pretrained on ~10 s audio clips and applied
  here to 100 ms windows, dominated by padding).
- An incorrect analytic detector-noise power spectral density (PSD) formula.
- Insufficient accounting for close prior art applying pretrained audio transformers to
  gravitational-wave data (see [GW-Whisper](https://arxiv.org/abs/2412.20789)).

Rather than patch these individually, the project was rebuilt from the data-processing layer up around
a different, more defensible statistical framework.

## Where the current project lives

**The current, validated, and citable version of this project is here:**

➡️ **[github.com/ruslanalas/gw-postmerger-detectability](https://github.com/ruslanalas/gw-postmerger-detectability)**

That repository uses a graph-spectral detectability statistic with an empirically calibrated null
distribution on real LIGO noise, a two-stage Stage-1/Stage-2 design frozen before a one-time held-out
evaluation, and is the basis for the associated manuscript and Zenodo archive
(DOI: [10.5281/zenodo.21993621](https://doi.org/10.5281/zenodo.21993621)).

Nothing in this archived repository should be cited or reused without reading the current repository's
manuscript first, which explains in its Introduction exactly why this earlier approach was abandoned.
