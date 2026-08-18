# Limitations

This document collects, in one place, every limitation and non-claim that governs how results from this
project may be described. It mirrors, and should stay consistent with, the manuscript's Limitations
section and `PROJECT_SPEC.md`'s forbidden-claims list — if any of these ever diverge, treat the
manuscript as authoritative for publication text and this file as a convenient repository-level summary.

## Signal model

The injected waveform, $h(t) = A e^{-t/\tau}\sin(2\pi f_{\rm peak} t + \phi)$, is a deliberately minimal,
single-mode phenomenological proxy — not a physically complete post-merger waveform. It does not capture
multi-mode structure, frequency drift, or amplitude modulation present in numerical-relativity
post-merger signals. Every result in this project should be read as a statement about this toy signal
injected into real noise, not as a statement about real post-merger gravitational-wave emission.

## Single detector

All results use LIGO Livingston (L1) data only. No multi-detector coincidence or network-level
significance is computed or claimed anywhere in this project.

## Finite-sample vs. production false-alarm rate

Reported false-positive rates are **finite-sample false-flag rates on constructed injection/noise
windows**, not production gravitational-wave search false-alarm rates estimated over months or years of
detector background. This project does not claim production search sensitivity, and does not claim
coherent-WaveBurst- or PyCBC-level false-alarm-rate calibration.

## Stage-1 interpretation

The Stage-1 statistic (`norm_lambda1`) is validated as sensitive to **local time-frequency organization
beyond total power** — this was established via low power-vs-statistic correlation and a tile-position
shuffle test (Notebooks 3 and 3.5). It is **not** validated as a measure of physical post-merger
coherence, ridge structure, or any other specific physical mechanism. The shuffle test also shows that
real detector noise itself already has non-trivial local time-frequency structure, so any physical
interpretation of Stage 1's sensitivity must remain comparative (more organized than noise) rather than
absolute (only signals show spatial dependence).

The Stage-1 statistic's motivation references spiked-random-matrix and stochastic-block-model
detectability theory as inspiration for the graph-spectral design. **This project does not implement
Christopher Moore's stochastic-block-model community-detection method**, literally or otherwise — that
connection is an analogy, not an implementation, and should always be described as "Moore/BBP-inspired"
or "spiked-matrix-motivated," never as an implementation of that method.

## $\hat\tau$ is a ranking score, not a validated point estimate

Both Stage-2 estimators (nonlinear LSQ, ridge/energy-decay) are validated as **anomaly-ranking scores**
— they discriminate standard from anomalous damping effectively (see the manuscript's ROC results). They
are **not** validated as accurate point estimates of the true physical damping time: the $\hat\tau$-vs-true-$\tau$
recovery scatter (Notebook 4, Figure 7 in the manuscript) shows substantial variance, especially at low
SNR. Do not report a single $\hat\tau$ value as a precise physical measurement.

## No learned-classifier baseline in this version

Neither a simple convolutional neural network nor the frozen-audio-transformer (AST/Mahalanobis)
approach explored in an earlier, abandoned version of this project is included as a baseline here. Both
are explicitly out of scope for this version and would require their own independently fair
training/validation/held-out-test protocol if added in future work. Reintroducing either without that
protocol would be scope creep, not a fair comparison.

## No dark matter or dark-sector detection claim

This project establishes a statistically defensible screening *methodology* for a phenomenological
damping-time anomaly on synthetic, injected signals. It does not claim, and should not be read or cited
as claiming, evidence for dark matter, dark-sector physics, or any specific non-standard energy-loss
mechanism. The anomalous damping times used throughout ($\tau=1,3$ ms) are a phenomenological proxy
category, chosen for methodological development, not a physical prediction.

## Calibration sample sizes

Several calibrated quantities (the Stage-1 null threshold in particular, and both Stage-2 anomaly
thresholds) rely on percentile estimates whose precision is limited by calibration sample size. These are
reported as point values throughout for readability, but should be understood as having residual
sampling uncertainty rather than being exact.
