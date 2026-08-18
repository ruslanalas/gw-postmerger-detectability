# Data Split Policy

This project's credibility depends entirely on this split discipline being followed without exception.
Four non-overlapping real-noise segments are used, located via GWOSC's data-quality timeline (not a
blind offset from a reference GPS time — real detector duty cycles are well below unity, and hour-scale
non-observing gaps are common).

| Segment | Duration | Role | Used for PSD envelope? | Used for threshold/model fitting? | Used for final evaluation? |
|---|---|---|---|---|---|
| `psd_block` | 2048 s | PSD / noise characterization | Yes | No | No |
| `calib_block` | 1024 s | Stage-1 null calibration | Yes | Yes (Stage-1 threshold, ridge noise floor) | No |
| Training segment — **`stage2_calib`** half | 508 s | Stage-2 threshold / classifier fitting | Yes¹ | Yes (Stage-2 thresholds, power-baseline classifier) | No |
| Training segment — **`stage2_val`** half | 508 s | Held-out Stage-2 validation | Yes¹ | No — evaluation only, no refitting | No |
| `test_block` | 1024 s | **Final evaluation** | No | No | **Yes — loaded once, after design freeze** |

¹ Both halves were part of the single training segment used, as a whole, in the PSD envelope
(Notebook 1) — that envelope was computed *before* the segment was later split (Notebook 4.5) into
`stage2_calib`/`stage2_val` for Stage-2 held-out validation specifically.

## What `test_block` is excluded from, explicitly

`test_block` is excluded from:
- PSD estimation (the operational PSD is an envelope across `psd_block`, `calib_block`, and the training
  segment only)
- Stage-1 null calibration
- Stage-2 threshold/classifier fitting
- Any model selection, hyperparameter tuning, or design decision of any kind

`test_block` is used for exactly one thing: the final, one-time evaluation in Notebook 5, after the
complete design (Stage-1 statistic and threshold, both Stage-2 estimators and their thresholds, and the
power baseline's classifier and operating threshold) was frozen and recorded in `frozen_config.json`.

## Why this matters

Every number reported anywhere in this project *before* Notebook 5 is, by construction, an in-sample or
held-out-within-training-data result — useful for development and for building confidence in the design,
but not an independent test of it. The Notebook 5 result is the only number in this entire project that
was computed on data the design had never seen in any form. This is why `PROJECT_SPEC.md` and the
manuscript both insist on stating clearly, wherever results are discussed, which of these two categories
a given number belongs to.
