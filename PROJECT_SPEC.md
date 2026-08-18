# PROJECT_SPEC.md
### Detectability-Aware Screening of Post-Merger GW Damping Anomalies in Real Detector Noise
*Checkpoint document — captures project state so work can resume without re-deriving context.*

---

## 1. Scientific goal

Build a **methodologically defensible, modest proof-of-concept** for screening synthetic post-merger
gravitational-wave-like transients for anomalous damping, injected into **real LIGO L1 noise**, using a
**detectability-aware, graph/spectral time-frequency statistic** with an **empirically calibrated** null
distribution — rather than a frozen pretrained-audio-model classifier (the project's original framing,
abandoned after review found it had a manuscript/code mismatch, a wrong PSD formula, an AST input-length
domain mismatch, and pre-existing close prior art in GW-Whisper).

The signal is a **phenomenological proxy**: `h(t) = A·exp(-t/τ)·sin(2π f_peak t + φ)`. Anomalously short
τ is framed as a possible signature of non-standard (including dark-sector-motivated) energy-loss
channels in the post-merger remnant. **This project does not claim dark matter detection, evidence for
dark-sector physics, or any physical detection result.** It is infrastructure + methodology only, as of
this checkpoint.

---

## 2. Current pipeline (as of this checkpoint)

```
Notebook 1  ->  Notebook 2  ->  Notebook 3  ->  [Notebook 4 -- not yet built]
(data/PSD/      (signal gen,    (TF graph,      (Stage 2 physics-anomaly
 whitening)      SNR, inject,    detectability   score, baselines,
                 CQT)            statistic,      final comparison figures)
                                 null calib,
                                 phase diagram)
```

Each notebook is **self-contained** (re-defines whatever it needs from prior notebooks -- `whiten()`,
`compute_snr()`, etc. -- rather than importing). This was a deliberate choice for Colab portability; the
cost is that a fix to a shared function (e.g. `whiten()`) must be propagated by hand to every notebook
that redefines it. Check this explicitly whenever a core function changes.

---

## 3. Notebooks list

| Notebook | Status | Purpose |
|---|---|---|
| `Notebook1_Data_PSD_Whitening.ipynb` | **v13, working** | Fetch real L1 data via GWOSC data-quality segments; 4-block GPS split; PSD estimation (median baseline + percentile-based intermittency detection + envelope across blocks + line-safety inflation); band-limited whitening; extensive sanity checks. |
| `Notebook2_Injection_SNR_CQT.ipynb` | **working, Test A/B/C + CQT fix applied** | Damped-sinusoid generator; matched-filter SNR (`compute_snr`/`scale_to_snr`); padded-whiten-crop injection (`inject_and_whiten`); Test A (synthetic Gaussian, hard assert -- passed); Test B (real noise, diagnostic only); Test C (production pipeline stability -- passed); CQT representation (fmin=500Hz, tuned for a 100ms window). |
| `Notebook3_Graph_Detectability_PhaseDiagram.ipynb` | **v2, working** | Band-restricted, empirically-normalized time-frequency graph; four-tier statistic hierarchy (raw lambda_1 / norm lambda_1 / total power / P_det); null calibration on `calib_block` (N=2000); (SNR, tau) phase diagram on `train_block` (N_PER_CELL=40). First-pass tau=1ms sensitivity finding -- **confirmed** at larger N in Notebook 3.5, see Sec. 10. |
| `Notebook3_5_Confirmation.ipynb` | **working, decision criterion met** | Confirmation-only pass at `N_PER_CELL=125`: reproduces and strengthens Notebook 3's power-vs-graph-statistic gap (decision criterion met -- see Sec. 10); category-wise and fixed-SNR power-vs-lambda_1 correlations; permutation/shuffle sanity test. No new methodology introduced. |
| `Notebook4_Stage2_Damping.ipynb` | **working, first-pass results in hand** | Stage-2 tau-hat estimation (ridge/energy-decay fit, primary; adaptive noise-floor-truncated fit window after a v1 bug -- see Sec. 10) for Stage-1-passing candidates only; baselines (total power, nonlinear damped-sinusoid least-squares fit; simple CNN/AST-Mahalanobis deliberately deferred as optional); standard (5-40ms) vs. anomalous (1/3ms) discrimination, reported both conditionally (Stage-1 passers only) and end-to-end (all injections); `test_block` intentionally not loaded in this notebook. Results: ridge AUC=0.922 (competitive ranking, lower recall than LSQ at the selected operating point:
TPR=0.455 vs LSQ TPR=0.819), LSQ AUC=0.935, power AUC=0.287 (weak/directionally unstable, not simply
"uninformative" -- see Notebook 4.5). See Sec. 10 for full numbers and caveats. |
| `Notebook4_5_HeldoutValidation.ipynb` | **working, held-out results confirm Notebook 4** | Splits
`train_block` into non-overlapping `stage2_calib`/`stage2_val` halves; fits all Stage-2
thresholds/classifiers on `stage2_calib` only, evaluates on `stage2_val` only. Results: ridge AUC=0.915,
lsq AUC=0.934 (both within noise of Notebook 4's in-sample numbers -- no meaningful overfitting found);
power baseline corrected to best_power_auc=0.725 (directionally unstable, not "uninformative" -- see Sec.
10). `test_block` not loaded. **This notebook's output, not Notebook 4's, is what a manuscript should
cite.** Prerequisite for freezing the design is satisfied; explicit sign-off still required before any
`test_block` use. |
| `Notebook5_Final_TestBlock_Evaluation.ipynb` | **being built this session** | The one-time, final
evaluation. Loads `test_block` for the first and only time in this project. Applies the frozen config
(`frozen_config.json`) exactly -- no threshold/model/hyperparameter changes based on test_block results.
Reports Stage-1 pass rate, conditional and end-to-end Stage-2 metrics, per-tau flag rates, AUC and
low-FPR operating-point results for ridge/lsq/power, and a final JSON summary. |

---

## 4. Agreed terminology

- **`psd_block` / `calib_block` / `train_block` / `test_block`** -- the four non-overlapping GPS blocks,
  found via `gwosc.timeline.get_segments('L1_DATA', ...)`, each with a fixed role (Sec. 6).
- **Envelope PSD** -- element-wise maximum of independent per-block PSD estimates across
  `{psd_block, calib_block, train_block}` (`test_block` deliberately excluded).
- **Median baseline** -- the accurate, bias-corrected continuum PSD estimate (gwpy `method='median'`),
  used everywhere EXCEPT at flagged line bins.
- **Persistent line** -- a bin where the median envelope exceeds `PERSISTENT_LINE_FACTOR=5x` its local
  smoothed baseline.
- **Intermittent line** -- a bin where the ratio (99th-percentile-across-segments) / median exceeds
  `INTERMITTENCY_FACTOR=2.5x` the typical such ratio for stationary noise -- catches lines that are quiet
  in the block-averaged median but ring up occasionally.
- **Line-safety inflation** -- flagged bins are set to `max(percentile, median) x LINE_EXTRA_MARGIN(=5)`,
  i.e. multiplying the OBSERVED value, never replacing it with something smaller.
- **Operational PSD** -- the single, final PSD (`freqs`/`psd` in `estimated_psd.npz`) that
  `whiten()`/`compute_snr()` actually use. `psd_raw` (median envelope, uninflated) is diagnostic only.
- **`F_LOW=20 Hz`, `F_HIGH=6000 Hz`** -- the hard whitening/SNR passband. Content outside this range is
  zeroed, never divided by an (unreliable near DC/Nyquist) PSD value.
- **`EDGE_TRIM_S=4.0 s`** -- seconds trimmed from each end of a whole-block whitening operation, or from
  each end of a padded local injection segment, before slicing into 100ms analysis windows. Whitening a
  bare 100ms window on its own reintroduces a circular-convolution edge artifact -- never do this.
- **Matched-filter SNR** (`rho^2 = 4*integral(|h_tilde(f)|^2/S_n(f)) df`, discretized) -- the ONLY SNR
  definition used in this project as of Notebook 2. The old nominal/RMS-based "SNR" from the original
  (pre-review) version of this project is deprecated and must not reappear.
- **Test A / Test B / Test C** -- the three-part SNR validation framework (NB2 Sec. 5): A = synthetic
  Gaussian, hard assert; B = real noise, diagnostic only, no assert; C = production pipeline stability,
  asserts only on finiteness/edge-artifacts/no pathological outliers, never on recovery statistics.
- **Detectability statistic, four-tier hierarchy** (NB3 v2): (1) `raw_lambda1` -- leading eigenvalue
  of a graph built directly from raw (band-restricted) CQT magnitude; **first-pass baseline**, no
  empirical null normalization of the tiles. (2) `norm_lambda1` -- leading eigenvalue of a graph built
  from `Z_pos` (CQT tiles normalized against `calib_block`'s empirical median/MAD, log1p'd, floored at 0)
  *before* the graph is built; **this is the statistic the paper should call "Moore/BBP-inspired" or
  "spiked-matrix-inspired graph-spectral detectability"** -- never "we implement Christopher Moore's
  method" (this project does not implement SBM community detection, literally or otherwise -- the
  connection is the spiked-matrix/BBP detectability-transition analogy only). (3) `total_power_raw` --
  simplest competing **baseline** (sum of in-band CQT magnitude). (4) `P_det(SNR, tau)` -- the **empirical
  detectability result** (fraction of injected windows exceeding a statistic's own calibrated threshold),
  computed for each of (1)-(3), not a per-window statistic itself. Graph edges are built from
  already-normalized tiles (edge weight = product of two tile values), so no separate edge-normalization
  step is needed -- normalizing tiles is sufficient by construction.
- **CONFIRMED (NB3 Sec. 7/7b): what's actually load-bearing.** power-vs-`raw_lambda1` Pearson r=0.173,
  power-vs-`norm_lambda1` r=0.139 (both low -- the graph statistic, in either form, carries substantial
  information beyond total power); `raw_lambda1`-vs-`norm_lambda1` r=0.958 (very high -- normalization is
  close to a monotonic reparametrization of the raw graph statistic's ranking). **The correct claim for
  the paper is that the local time-frequency coherence graph structure itself (leading eigenvalue vs. a
  power sum) is the primary source of the sensitivity gain; empirical null-normalization is a real but
  secondary refinement on top of it, not a separately load-bearing mechanism.** Do not claim
  normalization is "the" contribution -- it is one part of a graph-spectral statistic whose main strength
  is structural (local coherence), not the calibration step alone.
- **Post-merger band mask** -- CQT statistics are computed only on bins within
  `[CQT_BAND_LOW, CQT_BAND_HIGH] = [1500, 4000] Hz`, masked explicitly via CQT bin center frequencies
  (`librosa.cqt_frequencies`), not the full `[500, 8000] Hz` CQT range.
- **Stage 1 / Stage 2** -- Stage 1 = null/noise gate (now planned to be based on the NB3 detectability
  statistic, not the old AST/Mahalanobis approach from the abandoned original design). Stage 2 = physics
  anomaly score (tau estimate vs. standard manifold) -- not yet built (Notebook 4).
- **tau** -- damping time; **f_peak** -- post-merger dominant frequency, drawn `Uniform(1500, 4000)` Hz
  unless otherwise specified.

---

## 5. Strict claims / forbidden claims

**Never claim, in any manuscript draft or figure caption:**
- Detection of, or evidence for, dark matter, dark-sector physics, or any non-standard energy-loss
  channel. tau is a *phenomenological proxy* only.
- That real LIGO noise behaves as stationary Gaussian for matched filtering (Test B falsifies this
  directly -- state the deviation plainly).
- That the project's PSD is the official LIGO-pipeline-calibrated PSD. It is an independently
  constructed, deliberately conservative estimate (median baseline + envelope + line-safety margin) and
  must be described as such.
- Novelty of "pretrained audio transformer for GW" (GW-Whisper predates this) or of "detecting bursts via
  time-frequency structure" in general (excess power / coherent WaveBurst / Q-scan predate this by ~20
  years). Any novelty claim must be scoped narrowly: an RMT/spiked-matrix-motivated detectability
  threshold with a resolution-limited (SNR, tau) phase diagram, calibrated empirically on real noise --
  not the general idea.
- "Foundation model," "low-latency," or "AST understands physics" language -- not applicable to the
  current design (no AST is used) and no runtime has been benchmarked regardless.
- Reintroducing AST/Mahalanobis as a Stage-2 baseline -- reaffirmed, explicitly, when Notebook 4 baselines
  were scoped: the project has pivoted away from that framing entirely, and adding it back now would be
  scope creep requiring its own separate fair train/val/test protocol. If ever revisited, it needs its
  own dedicated section, clearly separated from the classical baselines, not folded in as "just another
  comparison." A simple CNN is optional and, if added, belongs in a similarly separate appendix section,
  after held-out (Notebook 4.5-style) validation is stable -- not required for v1.
- Any result from a component that hasn't been built yet (detectability threshold, phase diagram, Stage
  2 score) -- none of this exists in the paper's evidentiary base until Notebook 3/4 are complete AND
  their outputs pass the required tests below.
- A literal statistical-physics phase transition (critical exponents, universality) for the detectability
  boundary -- "phase diagram" here means a 2D detection-probability heatmap with a knee, not a physical
  phase transition.
- Claiming the tau=1ms sensitivity finding (Sec. 10) demonstrates physical coherence or post-merger
  ridge structure specifically -- the shuffle test (Sec. 10) shows even pure noise has nontrivial local
  spatial structure, so the defensible claim is comparative ("more organized than noise, especially at
  short tau"), not that the statistic has been shown to measure a specific physical mechanism.
  This finding is now CONFIRMED at N_PER_CELL=125 (Sec. 10) and may be used in a manuscript with the
  exact required phrasing given there -- it is no longer preliminary, but it is still only as strong as
  that specific phrasing, not stronger.

**Always state explicitly when relevant:**
- The signal model is a single-mode toy damped sinusoid, not a physically complete post-merger waveform.
- Detector coverage is L1 only; no H1 coincidence/network analysis.
- All statistical thresholds are calibrated empirically on real noise, precisely because Test B showed
  real noise does not match the ideal Gaussian assumption.

---

## 6. Data split policy

Four non-overlapping GPS blocks, found via `gwosc.timeline.get_segments('L1_DATA', ...)` (never blind
GPS offsets -- this failed once already, landing in a multi-hour non-observing gap), packed sequentially
into verified-good segments with an 8s guard gap between blocks.

| Block | Duration | Role | Used for PSD/envelope? |
|---|---|---|---|
| `psd_block` | 2048s | Noise characterization only | Yes |
| `calib_block` | 1024s | Null-distribution calibration (NB3) | Yes |
| `train_block` | 1024s | Injection sweeps / any learned Stage 2 component | Yes |
| `test_block` | 1024s | **Final evaluation, touched once** | **No -- deliberately excluded** |

**Rule, restated because it's easy to violate by accident:** `test_block` must never contribute to PSD
estimation, line detection, threshold calibration, or any tuning decision. It is read only in a final
"Notebook 4, last section" evaluation, once, and that number is reported as-is.

**NB4-specific split usage:** `test_block` is not loaded at all in Notebook 4 -- not merely "not
tuned on," but physically absent from the notebook's namespace, so it cannot be touched even by accident.
Stage-2 development, threshold calibration (standard-tau tau_hat distribution), and all reported
conditional/end-to-end metrics in Notebook 4 use `train_block` only. `test_block` is loaded and evaluated,
once, only in a future notebook once the full Stage-1+Stage-2 design is frozen -- not before.

**NB3-specific split usage (this checkpoint):** null calibration draws noise-only windows from
`calib_block` ONLY. The (SNR, tau) phase-diagram injection sweep draws its noise component from
`train_block` ONLY -- kept separate from `calib_block` so the calibrated threshold and the swept detection
probabilities are never built from the same specific noise draws.

---

## 7. PSD / SNR conventions

- **PSD estimation** (NB1 Sec. 3, v13): per envelope block, compute both (a) gwpy `method='median'` PSD
  (accurate continuum baseline) and (b) a 99th-percentile-across-Welch-segments PSD (intermittency
  detector, via `scipy.signal.spectrogram` + `np.percentile`, NOT gwpy's median). Take element-wise max
  of each across `{psd_block, calib_block, train_block}`. Flag persistent + intermittent lines (Sec. 4
  definitions) against the median envelope; inflate only flagged bins to
  `max(percentile, median) x 5`. Save as `freqs`/`psd` in `estimated_psd.npz` (`psd_raw` = uninflated
  median envelope, diagnostic; `line_mask` = which bins were flagged).
- **Whitening** (`whiten()`, identical in NB1 and NB2): band-limit hard to `[F_LOW, F_HIGH]`; never
  divide by a PSD value outside that range (DC and near-Nyquist bins are known-unreliable). Whiten a
  **whole long block once**, then trim `EDGE_TRIM_S` from each end, THEN slice into 100ms windows --
  never whiten a bare 100ms window in isolation (circular-convolution edge artifact).
- **Injection** (`inject_and_whiten()`, NB2 Sec. 4): draw a padded local segment
  (`window + 2xEDGE_TRIM_S`) from the target block's OWN raw pool, inject at window center, whiten the
  whole padded segment once, crop back to the 100ms window. Never inject into, then whiten, a bare
  window.
- **Matched-filter SNR**: `rho^2 = 4*df*sum(|h_tilde(f_k)|^2/S_n(f_k))` over the passband,
  `h_tilde = rfft(h)/fs`, `df = fs/N`. Validated correct (Test A, NB2 Sec. 5) -- synthetic Gaussian noise
  built to match this exact convention recovers target SNR with mean/median/std/robust-std all matching
  N(target,1) essentially exactly. **Real noise does NOT satisfy N(target,1)** (Test B: robust_std ~ 11.5)
  -- expected, not a bug, motivates empirical calibration rather than further PSD tuning.
- **Sample rate**: `FS=16384 Hz` used consistently everywhere (an earlier bug had `fetch_open_data`
  silently defaulting to 4096 Hz and being upsampled -- fixed; always pass `sample_rate=FS` explicitly and
  assert on the result rather than resampling silently).

---

## 8. Required tests (re-run whenever the pipeline changes)

**Notebook 1:**
- Block non-overlap assertion (`assert_non_overlapping`).
- Finiteness assertions at every stage (fetch, PSD, whiten, window pools) -- fail loudly at the point of
  the problem, not several cells downstream.
- PSD pathology diagnostic: no bin below `PATHOLOGY_THRESHOLD` inside `[F_LOW, F_HIGH]`.
- Whitening flatness check (independent Welch re-estimate of a whitened long block) -- should be flat
  ~1.0 away from flagged lines, appropriately suppressed (not excess) at flagged lines.
- Gaussianity/Q-Q check -- informative only, real noise is expected to deviate somewhat; large +/-20-sigma
  tails would indicate a real bug (e.g. the DC-bin / near-Nyquist pathologies found earlier).
- Variance check -- expected value is `~ F_HIGH - F_LOW` (Parseval, flat-PSD~1 over the passband), NOT
  O(1) (an earlier version of this check had the wrong expectation -- fixed).

**Notebook 2:**
- Test A (hard assert): synthetic Gaussian recovery ~ N(target, 1).
- Test B (diagnostic, no assert): real-noise recovery statistics, logged for the record.
- Test C (hard assert, but NOT on recovery statistics): production pipeline output is finite,
  edge/interior variance ratio `< 3.0`, no pathological-variance outlier (`max < 50x median`).
- CQT: no librosa `n_fft too large` warnings; example panels should show visible structure across the
  full configured frequency range, not a blank region.

**Notebook 3 (this checkpoint -- required tests defined below, must pass before trusting any output):**
- Detectability statistic is finite for every calibration and sweep window.
- Calibration sample size adequacy: report `N_CALIBRATION`; a chosen false-alarm rate `alpha` needs
  enough samples that the `(1-alpha)` quantile isn't just the single most extreme sample (rule of thumb:
  want several tens of expected exceedances, i.e. `N_CALIBRATION >~ 20/alpha`).
- Threshold stability: bootstrap CI on the calibrated threshold, reported alongside the threshold itself.
- No duplicate/overlapping window draws silently reused between calibration and sweep (`calib_block` vs
  `train_block` separation, Sec. 6).

---

## 9. Paper figures (planned)

1. Estimated ASD vs. analytic design curve (NB1) -- cross-check only, not operational.
2. Whitening flatness + Q-Q diagnostics (NB1) -- appendix/reproducibility material.
3. Test A vs. Test B SNR-recovery comparison (NB2) -- makes the "real noise != ideal Gaussian, hence
   empirical calibration" argument visually, likely a main-text figure given how central this point is.
4. Example CQT panels: pure noise / standard injection / anomalous injection (NB2).
5. Detectability statistic null distribution with calibrated threshold(s) (NB3).
6. **(SNR, tau) phase diagram, three-panel comparison** (total power / raw lambda_1 / norm lambda_1)
   (NB3) -- likely the headline result figure. First-pass run (N_PER_CELL=40) shows raw_lambda1 and
   norm_lambda1 both detecting tau=1ms anomalies far more reliably than total_power_raw at moderate SNR
   (e.g. P_det~1.0 by SNR=12 for the lambda_1 statistics vs. ~0.07 for total power) -- promising, but see
   Sec. 10 before treating this as a confirmed result.
6b. Power-vs-lambda_1 and raw-vs-norm-lambda_1 scatter plots (NB3 Sec. 7/7b) -- the correlation evidence
   underpinning whether lambda_1 is described as adding information beyond excess power.
6c. **Confirmation phase diagram at N_PER_CELL=125** (NB3.5 Sec. 5) -- the number actually worth quoting
   in a manuscript, not NB3's N_PER_CELL=40 first pass.
6d. Fixed-SNR power-vs-lambda_1 correlation (NB3.5 Sec. 7) -- r~0 at matched SNR pooling across tau; the
   sharpest single piece of evidence for the "beyond total power" claim, likely worth a main-text mention.
6e. Shuffle sanity test bar chart (NB3.5 Sec. 8) -- with the noise-also-drops caveat stated in the caption.
7. Stage 2 physics-anomaly-score distributions (NB4, not yet built).
8. Baseline comparison table/ROC: excess power, damped-sinusoid least-squares fit, this method (NB4, not
   yet built).
9. Final `test_block` evaluation numbers, reported once (NB4, not yet built).

---

## 10. Current known issues

- **Edge variance ratio ~1.7-2.0x** in injected windows (Test C / NB2 Sec. 6) -- within the tolerance
  threshold (3.0) but not exactly 1.0. Plausible benign explanation (smaller edge sample -> noisier
  variance estimate) vs. residual real effect not yet distinguished. Watch whether this biases the NB3
  detectability statistic specifically near window edges.
- **Real noise substantially non-Gaussian/non-stationary** in the matched-filter sense (Test B) -- by
  design, this motivates NB3's empirical calibration rather than being "fixed."
- **`test_block` line coverage is indirect**: excluded from PSD/envelope construction by design (Sec. 6),
  so its own potential unique lines are covered only by the general safety margin from the other three
  blocks, not directly measured. A final, one-time sanity check against `test_block` (without using it to
  tune anything) is worth doing at the very end of Notebook 4.
### Notebook 4 Stage-2 result (first pass, `train_block`-only, in-sample) — CORRECTED interpretation

**Bug found and fixed:** the ridge/energy-decay `tau_hat` estimator originally used a fixed 60-frame
(~29ms) fit window regardless of the signal's actual decay time, systematically inflating `tau_hat` for
short tau (worst case true tau=1ms gave estimates of 35-95ms). Fixed by making the fit window adaptive --
it stops once frame power drops into a `calib_block`-calibrated noise floor for 2 consecutive frames.
AUC went from 0.393 (worse than chance) to 0.922 after the fix.

**Ridge vs. LSQ -- do not describe as "comparable" or "on par."** AUC is close (ridge=0.922, lsq=0.935),
but at the actual selected low-FPR operating threshold, conditional TPR differs substantially:
ridge=0.455, lsq=0.819. **Correct phrasing:** "ridge gives competitive ranking-level separation, while
LSQ gives substantially higher recall at the selected low-FPR operating point." AUC alone hides this gap;
always report both.

**Power baseline -- "near/below chance" was an incomplete, potentially misleading description; needs the
enhanced analysis below (Notebook 4.5) before further characterization.** AUC(power)=0.287 does NOT mean
power contains zero information -- it means the assumed direction ("higher power = more anomalous") may
be wrong or unstable for this particular in-sample fit. The correct diagnostic (AUC(power),
AUC(-power), best_power_auc=max of the two, and a logistic-regression-probability AUC) is implemented in
Notebook 4.5 and must be reported before claiming power is "uninformative" rather than "weak and
directionally unstable."

**`tau_hat` is supported as an anomaly-RANKING score, not yet as a precise recovery estimator.** The
recovery scatter shows large variance and outliers, especially for standard tau at low SNR (e.g. true
tau=20ms/SNR=10 gave 10.7ms; true tau=10ms/SNR=10 gave 55.7ms) -- plausibly the adaptive stop
occasionally triggering early on a downward noise fluctuation right after the peak. The current evidence
(AUC=0.922) supports "ridge `tau_hat` ranks candidates by anomalousness well"; it does NOT yet support
"ridge `tau_hat` is an accurate point estimate of the true damping time." Keep these two claims separate
in any manuscript text -- do not use one to justify the other.

**Conditional flag rate by tau (ridge method): tau=1ms=0.704, tau=3ms=0.205**, standard tau (5-40ms)
false-flag rates all under 2%. tau=3ms being much harder than tau=1ms is expected, not a problem -- it
echoes the earlier review's recommendation (from the original, since-abandoned project version) to
describe tau=3ms as "conditional anomaly detection" rather than a uniform flag, since it sits much closer
to the standard-tau boundary. Present tau=1ms and tau=3ms results separately in any manuscript text, not
as a single undifferentiated "anomalous" category.

**Every number above is in-sample on `train_block`** -- fit and evaluated on the same sweep, not
independently held out. **This is acceptable for a development notebook but not sufficient for final
claims** -- see Notebook 4.5 below, which is the required prerequisite before treating any Notebook 4
number as final. `test_block` remains unloaded/untouched throughout (explicit assert guards against
accidental use in both Notebook 4 and 4.5).

### Notebook 4.5 — held-out validation within `train_block` — RESULTS (these are the numbers to cite)

Splits `train_block`'s raw noise pool into two non-overlapping halves (`stage2_calib` / `stage2_val`,
508s each, guard gap between them). All thresholds/classifiers fit on `stage2_calib` ONLY; all reported
metrics computed on `stage2_val` ONLY, no refitting. `test_block` not loaded.

**Ridge and LSQ are stable under held-out validation within the `train_block` split** -- not "perfect
generalization" (this notebook does not test generalization to `test_block` or any other data):

| | Notebook 4 (in-sample) | Notebook 4.5 (held-out) |
|---|---|---|
| ridge AUC | 0.922 | 0.915 |
| lsq AUC | 0.935 | 0.934 |
| ridge conditional TPR (at fitted operating point) | 0.455 | 0.472 |
| lsq conditional TPR (at fitted operating point) | 0.819 | 0.835 |
| tau=1ms conditional flag rate | 0.704 | 0.704 |
| tau=3ms conditional flag rate | 0.205 | 0.242 |

**Correct phrasing: "ridge gives strong ranking-level separation; LSQ gives the strongest low-FPR
operating-point recall."** Not "comparable" or "matching" -- AUC is close (0.915 vs 0.934) but
operating-point recall differs strongly (TPR=0.472 vs 0.835 at FPR~0.01-0.017). Always report AUC and
operating-point TPR/FPR together, never AUC alone.

**Power baseline -- total power is not uninformative; it is weaker and direction-sensitive.** Held-out
`AUC(power)=0.275`, `AUC(-power)=0.725`, `best_power_auc=0.725` -- a real, directionally consistent
signal: lower in-band power is associated with anomalous (short-tau) signals. `best_power_auc=0.725`
indicates real but moderate tau-discriminating information, clearly below ridge/lsq. **A fair
operating-point comparison** (threshold calibrated at the 99th percentile of `stage2_calib`'s own
standard-sample predicted probabilities, targeting ~1% FPR -- NOT the classifier's arbitrary default 0.5
boundary, which gave a degenerate, non-comparable TPR=FPR=0) is implemented in Notebook 4.5 Sec. 4/5 as
`power_operating` -- **RESULT (held-out, confirmed): TPR=0.043, FPR=0.012.** This is the sharpest,
most citable evidence for the Stage-2 claim: `AUC(-power)=0.725` says power carries real, moderate
ranking information, but at the actual low-false-alarm operating point that matters for a screening
pipeline, power recovers essentially nothing (4.3%) while ridge recovers 47.2% and lsq recovers 83.5% at
comparable FPR. Both statements are true and non-contradictory -- a moderate AUC is fully compatible with
near-zero recall at a strict threshold when the anomalous/standard distributions overlap substantially in
their bulk. **Quote both the AUC and the operating-point TPR for power; the operating-point contrast
(0.725 AUC vs. 0.043 TPR) is the number that heads off "why not just use power" in review.**

**Band-leakage diagnostic (Notebook 4.5 Sec. 8) result: inconclusive, likely due to a measurement-design
limitation, not a refutation.** Ratios across tau=1..40ms: 0.0312, 0.0320, 0.0319, 0.0324, 0.0328, 0.0336
-- a weak overall upward trend consistent with the hypothesis direction, but not strictly monotonic and
within error bars of each other at `N=40`/tau. Likely cause: `whitened_full_passband_power` sums energy
over the entire `[20,6000]Hz` whitening passband (~5980 Hz), which is overwhelmingly real detector NOISE,
not signal -- diluting whatever genuine signal-bandwidth-leakage effect exists underneath. A cleaner test
would isolate signal-only power (e.g. from the noise-free `h_scaled` component) or subtract each band's
own noise floor before comparing. This does not affect the Sec. 6 power finding itself, which doesn't
depend on this mechanism being confirmed -- it remains an open, low-priority question if the exact
mechanism ever needs pinning down.

**Required baselines for this pass, per the project's explicit, twice-reaffirmed scope decision: total
excess power (default and operating-point variants), inverted power, calibrated logistic power,
nonlinear LSQ fit, ridge/energy-decay estimator only** -- AST/Mahalanobis is not implemented and a simple
CNN remains optional, deferred to a separate appendix section if added at all, after held-out validation.

## 11. FROZEN DESIGN (status: ACTIVE — confirmed and locked)

**The design below is frozen as of this checkpoint. No threshold, model, hyperparameter, or
preprocessing choice listed here may be changed based on `test_block` results.** The canonical machine-
readable copy is `frozen_config.json`, loaded exactly by `Notebook5_Final_TestBlock_Evaluation.ipynb` --
if this section and that file ever disagree, `frozen_config.json` is authoritative.

- **Stage 1:** score = `norm_lambda1`; threshold = **55.6439**; `alpha=0.01`; calibrated on `calib_block`.
- **Stage 2, primary:** nonlinear LSQ `tau_hat`; anomaly threshold = **3.472 ms**; fit on `stage2_calib`.
- **Stage 2, secondary:** ridge/energy-decay `tau_hat`; anomaly threshold = **4.365 ms**; fit on
  `stage2_calib`; reported alongside LSQ, never as a replacement for it. **No further tuning of the ridge
  estimator's parameters is permitted**, in this or any future notebook.
- **Power baseline:** logistic regression on `total_power_raw`, fit on `stage2_calib` only; operating
  threshold = **0.3656** (probability). Report `AUC(power)`, `AUC(-power)`, `AUC(logistic)`, AND the
  low-FPR operating-point TPR/FPR together -- never AUC alone (see Sec. 10 for why: AUC=0.725 but
  operating-point TPR=0.043 in the Notebook 4.5 held-out run).
- **Excluded from v1, explicitly:** simple CNN, AST/Mahalanobis. Neither is scope for the main paper.
- **Band-leakage diagnostic is explanatory only, not load-bearing** -- inconclusive result (Sec. 10);
  must not be cited to justify, adjust, or interpret any threshold above.

**Only `Notebook5_Final_TestBlock_Evaluation.ipynb` may load `test_block`, and only once.** No prior
notebook in this project has touched it. If `test_block` performance differs from the `stage2_val`
held-out validation (Notebook 4.5), that difference is reported honestly -- it does not trigger a return
to tuning.

## 12. FINAL RESULT — `test_block` evaluation complete (terminal result of the experimental phase)

**Status: DONE. This is the project's reportable Stage-1/Stage-2 result.** `test_block` has been loaded
and evaluated exactly once, using the frozen config exactly, with all calibration artifacts (Stage-1
threshold, ridge noise floor, power operating threshold) re-derived from `calib_block`/`stage2_calib` and
verified to match `frozen_config.json` before evaluation. No threshold was adjusted based on `test_block`.

**Comparison to `stage2_val` (Notebook 4.5): no degradation, several metrics mildly better --** every
delta fell within the notebook's own "no notable difference" band (<0.10):

| metric | stage2_val | test_block |
|---|---|---|
| ridge AUC | 0.915 | 0.930 |
| lsq AUC | 0.934 | 0.943 |
| ridge conditional TPR / FPR | 0.472 / 0.010 | 0.486 / 0.016 |
| lsq conditional TPR / FPR | 0.835 / 0.017 | 0.879 / 0.012 |
| power conditional TPR / FPR | 0.043 / 0.012 | 0.054 / 0.010 |
| tau=1ms flag rate | 0.704 | 0.736 |
| tau=3ms flag rate | 0.242 | 0.237 |

Rule 9 ("if worse, report honestly, do not tune") was not triggered -- nothing tuned, nothing needed to be.

**New finding, not stated explicitly in any earlier notebook: Stage-1 pass rate is strongly
tau-dependent, and favorably so for this project's goal.** `test_block` pass rate by tau:
`1ms=0.987, 3ms=0.997, 5ms=0.973, 10ms=0.937, 20ms=0.853, 40ms=0.663`. Among standard-tau signals, pass
rate falls monotonically as tau increases -- a tau=40ms signal is rejected by Stage-1 a third of the time,
while both anomalous tau values pass almost always. **Mechanism (consistent with everything established
about `norm_lambda1` since Notebook 3):** the statistic rewards energy concentrated into few, spatially
compact, bright tiles (product-of-neighboring-tile-magnitudes graph construction). A long-tau signal
spreads the same SNR-normalized total energy across more time frames than a short-tau signal, so each
tile is dimmer and the eigenvalue is lower -- at matched total SNR, not because the signal is weaker
overall. **Must be stated as both a limitation and a favorable property in any manuscript:**
(a) limitation -- sensitivity to standard, astrophysically-expected long-tau signals is measurably weaker,
a third of genuine tau=40ms candidates never reach Stage-2, which matters for the end-to-end numbers;
(b) favorable, unengineered alignment -- this bias happens to favor exactly the short-tau (anomalous)
signals this project cares about detecting, and was not designed in.

**LSQ vs. ridge, now resolved with real data specifically on the hard case:** at tau=3ms (the harder
anomalous case, closer to the standard boundary), lsq conditional flag rate = 0.826 vs. ridge = 0.237 --
a much larger gap than at tau=1ms (0.932 vs. 0.736, both good). LSQ's standard-tau false-flag rate is
0.000 for tau=10/20/40ms and only 0.041 for tau=5ms -- very clean specificity. This is the clearest
evidence yet for reporting LSQ as primary and ridge as secondary, concentrated exactly where
discrimination is hardest.

**Power baseline, confirmed on held-out test data:** AUC(power)=0.274, AUC(-power)=AUC(logistic)=0.726,
operating-point TPR=0.054 at FPR=0.010 -- the AUC-vs-operating-point gap (0.726 vs. 0.054) reproduces
Notebook 4.5's finding almost exactly, confirming it is a real property of this baseline, not an artifact
of the validation split.

**Experimental phase of this project is complete.** Remaining work is manuscript writing (per Notebook 5's
closing section and `PROJECT_SPEC.md` Sec. 5/9), not further experimentation, tuning, or notebook
development, unless a manuscript-review process identifies a specific new question.

- **Line-detection thresholds are heuristic**: `LINE_FACTOR=5`, `INTERMITTENCY_FACTOR=2.5`,
  `LINE_EXTRA_MARGIN=5` were chosen reasonably but not derived from first principles or
  sensitivity-tested. Should be justified or stress-tested before being treated as final in a paper.
- **Node-value normalization is now implemented** (NB3 v2, superseding the earlier "raw magnitude,
  unnormalized" caveat): `norm_lambda1` is built from `Z_pos`, tiles normalized against `calib_block`'s
  empirical median/MAD before the graph is built. `raw_lambda1` (no normalization) is retained
  deliberately as the tier-1 baseline, not removed. Section 7's power-vs-lambda_1 correlation check
  (Pearson r=0.17 raw / 0.14 normalized, both "moderate/low") argues against `raw_lambda1` being
  redundant with total power, though this should be re-checked as the sweep grid grows.
### ACCEPTED Stage-1 interpretation (final, as of the Notebook 3.5 confirmation run)

**Central validated Stage-1 claim, the only phrasing to use in manuscript text:**

> "The graph-spectral statistic captures local organization of time-frequency excess beyond total
> integrated power."

**Do not claim** lambda_1 measures physical post-merger coherence. It may capture compactness,
localization, connected high-energy clusters, or ridge-like organization -- these remain
indistinguishable candidate explanations, not something this project has shown.

**Confirmed decision criterion (Notebook 3.5, `N_PER_CELL=125`):** SNR at which P_det first reaches
>=0.9 for tau=1ms -- `total_power_raw`: not reached even at SNR=15 (top of tested grid); `raw_lambda1`:
SNR=10; `norm_lambda1`: SNR=8. The classical baseline never catches up within the tested range while both
graph statistics saturate by SNR=8-10.

**The shuffle test (Notebook 3.5 Sec. 8) is the strongest evidence for the central claim**, stronger than
the correlation checks: randomizing CQT tile positions (preserving the value set, hence total power
exactly) while rebuilding the graph produces large, highly significant lambda_1 drops for every category
(Wilcoxon p=1.6e-11 throughout) -- and *especially* for injected signals: noise 54-64%, tau=40ms 69-70%,
tau=10ms 82%, tau=1ms 86-87% (largest of all). This directly demonstrates lambda_1 is structure-sensitive,
not merely power-sensitive, since the power-preserving shuffle alone is enough to collapse it.

Fixed-SNR correlation (pooling across tau at SNR=8/10/12: r=0.08/0.01/-0.05, essentially zero) is
corroborating evidence for the same claim, but the shuffle test is the primary evidence to cite.

**Two caveats carried forward, both still open:**
1. `N_CALIBRATION=1500` is marginal for alpha=0.01 (~15 expected exceedances, below the ~20 rule of
   thumb) -- exact threshold values should be tightened with a larger calibration run before being quoted
   precisely in a table (Notebook 4 recalibrates with a larger `N_CALIBRATION` for its own Stage-1 gate).
2. Noise itself also shows a substantial shuffle-test drop -- real whitened noise's own CQT already has
   non-trivial local spatial structure. **The claim must stay comparative** (injected structure, especially
   short tau, is *more* spatially organized than noise), never absolute (only signals show spatial
   dependence).
- **No multi-detector data** -- L1 only, no H1 coincidence.
- **Signal model is a toy** -- single-mode damped sinusoid, not a physically complete post-merger
  waveform; must be caveated explicitly in any manuscript.
- **Related-work citations still owed**: GW-Whisper, coherent WaveBurst / excess power, glitch-anomaly
  autoencoders, post-merger CNN detection work -- all identified in the earlier review, not yet written
  into any manuscript text (no manuscript text has been drafted yet as of this checkpoint).
- **Notebook 4 does not exist yet** -- Stage 2 scoring, classical baselines, and the final `test_block`
  evaluation are all still to be built.
