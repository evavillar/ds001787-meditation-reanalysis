# Reanalysis of Brandmeyer & Delorme (2018): EEG during meditation vs. mind wandering

## Overview

Independent reanalysis, not peer reviewed. The null result in non-experts replicates, but the expert theta and alpha effects do not, and removing the aperiodic background reveals no hidden state effect.

Original study: Brandmeyer & Delorme (2018; full reference below). Data: OpenNeuro [ds001787](https://openneuro.org/datasets/ds001787) (v1.1.1).

The goal is to reproduce the paper’s EEG result: the within-subject difference in theta (4–7 Hz) and alpha (9–11 Hz) power between meditation and mind-wandering probes, tested separately in expert and non-expert meditators. Behavioural analyses, the theta–alpha correlation and the group × condition interaction are out of scope.

## Pipeline

1.  **Probe alignment.** Probe times and answers (Q1 meditation depth, Q2 mind-wandering depth, Q3 drowsiness) are parsed from the behavioural text files and aligned to the EEG probe triggers (event 128) with a per-session offset.
2.  **Preprocessing (ICA fit).** 1 Hz high-pass, 50 Hz notch, bad channels detected with PyPREP and interpolated, average reference. Extended Infomax ICA (99% variance) on the continuous data. ICLabel removes components labelled eye blink, muscle or heart beat with p \> 0.8. This 1 Hz copy is used only to fit ICA; the analysed data are filtered at 2 Hz (step 3).
3.  **Epoching.** 2 Hz high-pass (6th-order Butterworth IIR, zero-phase), 50 Hz notch, same bad channels interpolated, average reference, ICA applied. Epochs from −10.05 to −0.05 s before Q1.
4.  **Trial classification (as in the paper).** Q1 \> Q2 = meditation, Q1 \< Q2 = mind wandering, ties discarded. Subjects with no trials in one condition are excluded.
5.  **Epoch rejection.** Global peak-to-peak threshold per session (autoreject `get_rejection_threshold`).
6.  **Time–frequency to spectral power (as in the paper).** Morlet wavelets, 100 log-spaced frequencies from 1 to 128 Hz, cycles rising linearly from 3 to 60. Power averaged over −8.55 to −1.55 s, which yields spectral power, then averaged within each band, converted to dB and averaged over epochs per subject and condition.
7.  **Statistics (as in the paper).** Paired comparison (one-sample test on meditation − mind-wandering differences) at each channel. Correction by cluster-based permutation (Maris & Oostenveld, 2007; 2048 permutations, cluster-forming threshold two-tailed p \< .05, about 4.5 neighbours per channel) and by Benjamini–Yekutieli FDR.

## Deviations from the original methods

| Step                | Paper                                                                    | This reanalysis                                                                                                                                                                                                                                                                                                                             |
|---------------------|--------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Bad channels        | Manual removal (mean 6, range 0–20 per subject)                          | PyPREP detection + interpolation (mean 4.1, range 0–8 per session)                                                                                                                                                                                                                                                                          |
| ICA fit             | Epoched, 2 Hz-filtered data after bad-epoch removal                      | Continuous, 1 Hz-filtered data (paper-style fit tested as robustness check 3)                                                                                                                                                                                                                                                               |
| Component selection | Visual (blinks, lateral eye movements, temporal muscle; 1–5 per subject) | ICLabel, p \> 0.8; heart-beat components also removed (mean 3.2, median 2, range 0–13 per session)                                                                                                                                                                                                                                          |
| Epoch rejection     | Manual removal of epochs with paroxysmal activity, before ICA            | Automated global peak-to-peak threshold, after ICA; on average 99–100% of epochs kept in both conditions (minimum 80%)                                                                                                                                                                                                                      |
| Line noise          | Not reported                                                             | 50 Hz notch                                                                                                                                                                                                                                                                                                                                 |
| Sample              | 24 participants                                                          | sub-013 and sub-014 excluded: their EEG files match subject 14’s behavioural log, so identity cannot be confirmed. Session 01 of sub-007 excluded: it shows a comb-like spectral artefact (regular dips about 1.9 Hz apart from 1 to 128 Hz, in all channels) that filtering or ICA cannot remove; sub-007 is analysed with session 02 only |
| Sessions            | Not reported                                                             | Sessions pooled per subject                                                                                                                                                                                                                                                                                                                 |
| Power units         | EEGLAB normalisation                                                     | MNE’s Morlet normalisation differs from EEGLAB’s, so absolute dB values are not comparable with the paper’s Figure 4. Differences between conditions are broadly comparable, because a constant scaling factor cancels when one condition is subtracted from the other in dB                                                                |

Final sample: 12 experts, 8 non-experts. Subjects 017 and 018 were excluded for having no trials in one condition; the paper also excluded two subjects for this reason, but it does not say which ones. All four excluded subjects (013, 014, 017, 018) are non-experts, so the expert group keeps the paper’s full size.

Epochs per subject in the analysis (meditation + mind wandering, ties excluded): mean 30.4 (SD 10.8, range 18–62). The paper reports mean 38.1 (range 21–64) clean epochs, possibly counted before ties were removed, so the two figures are not directly comparable.

## Results

|                    | Paper                           | This reanalysis                                                            |
|--------------------|---------------------------------|----------------------------------------------------------------------------|
| Experts, theta     | Frontal cluster, p \< .02       | No significant cluster (smallest cluster p = .74); no channel survives FDR |
| Experts, alpha     | Somatosensory cluster, p \< .02 | No significant cluster (smallest cluster p = .18); no channel survives FDR |
| Non-experts, theta | No significant difference       | No clusters formed; no channel survives FDR                                |
| Non-experts, alpha | No significant difference       | No clusters formed; no channel survives FDR                                |

The null result in non-experts replicates. The expert effects do not.

In experts, alpha t-values were positive at every channel (t from 0.01 to 2.93), which is the direction the paper reports (meditation \> mind wandering). But the effect is spread across the scalp rather than concentrated over somatosensory cortex, and it does not survive correction. Expert theta peaked at t = 2.27, barely above the cluster-forming threshold.

## Robustness checks

Each check changes one element of the main pipeline and keeps everything else fixed. Check 3 is the exception, because in the paper’s procedure the timing of rejection is tied to the ICA input.

Smallest cluster p per analysis (“—” = no cluster formed). r = correlation of the 64-channel t-map with the main analysis.

| Check                 | What changes                                                                                                                                                  | Experts theta | Experts alpha | Non-experts theta | Non-experts alpha | FDR  | r with main (range) |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------|---------------|-------------------|-------------------|------|---------------------|
| Main analysis         | —                                                                                                                                                             | .74           | .18           | —                 | —                 | None | —                   |
| 1\. Adjacency         | Delaunay-triangulation neighbours (`find_ch_adjacency`) instead of the ~4.5-neighbour distance threshold                                                      | .74           | .18           | —                 | —                 | None | 1.00                |
| 2\. ICLabel threshold | Components removed at p \> 0.5 instead of 0.8 (tests over-cleaning)                                                                                           | .71           | —             | .38               | —                 | None | .84–.96             |
| 3\. Paper-style ICA   | Bad epochs rejected before ICA; ICA fit on the clean 2 Hz epochs                                                                                              | .66           | .19           | —                 | —                 | None | .88–.96             |
| 4\. No ICA            | ICA not applied (tests under-cleaning)                                                                                                                        | .65           | .18           | —                 | —                 | None | .91–.98             |
| 5\. Minimum epochs    | Subjects with fewer than 3 epochs in either condition removed: sub-007 and sub-009 (1 mind-wandering epoch each; added after the notebook 02 quality control) | .69           | .07           | —                 | —                 | None | .64–1.00            |

The conclusion holds across all six analyses: no significant meditation vs. mind-wandering difference in either band or group.

- **ICA did not create or hide an effect.** The result with no ICA at all (check 4) is essentially the same as with ICA, and so is the result with a lower component-selection threshold (check 2) or a paper-style decomposition (check 3).
- **Check 2 is a test of over-cleaning.** Lowering the threshold to 0.5 removed more components in 31 of 37 sessions.
- **The topographies are stable, not only the p-values.** Every preprocessing check’s t-map (checks 1–4) correlates at least .84 with the main analysis. The lowest values are non-expert theta under the 0.5 threshold (.84) and expert theta with paper-style ICA (.88). Check 5 correlates less (down to .64) because it changes the sample, not the preprocessing.
- **The neighbourhood definition is not the cause.** Check 1 gives the same clusters.
- **Removing subjects with almost no mind-wandering epochs strengthens expert alpha but not to significance.** Without sub-007 and sub-009, expert alpha moves closer (smallest cluster p .07; t positive at every channel, 1.08 to 2.64; t at FCz 2.09), but no cluster reaches p \< .05 and no channel survives FDR. Expert theta is not close to significance (smallest cluster p .69). Non-expert tests are unchanged.
- **The expert direction is consistent but weak.** Theta at FCz is t = 2.0 to 2.3 in every preprocessing variant (1.4 in check 5), and expert alpha t-values are almost entirely positive. Both match the paper’s direction (meditation \> mind wandering), but neither survives correction in any analysis.

## Interpretation and limitations

- The divergence from the paper cannot be attributed to the preprocessing choices tested here: bad-epoch handling, ICA input, component-selection threshold, removing ICA altogether, or neighbourhood definition.
- **Remaining differences that could explain it:**
  - **Sample.** The expert group has the same size as in the paper (12), so a smaller sample cannot explain the missing expert effects. The four excluded subjects only reduce the non-expert group, from 12 to 8. With 12 subjects, a cluster test still has limited power for small effects, but the paper’s own analysis had the same constraint.
  - **Epoch rejection.** The automated threshold kept almost all epochs, while the paper removed epochs with paroxysmal activity by hand. Some noisy epochs the authors excluded may therefore remain here.
  - **Session pooling.** Several participants were recorded in two or three sessions. The paper does not say how sessions were combined. Here, epochs from all sessions were pooled into a single average per subject and condition, so participants with more sessions contribute more epochs to their own estimate, but each participant counts once in the group tests.
  - **Unreported details of the original statistics:** cluster-forming threshold, number of permutations, test statistic.
- This reanalysis **cannot distinguish a smaller true effect from a false positive** in the original study.

## Aperiodic analysis (notebook 02)

The question: does the meditation vs. mind-wandering comparison change once the aperiodic (1/f-like) background is removed from the spectrum? The analysis uses the epochs of the main pipeline. Robustness checks are not repeated, since the replication result did not depend on preprocessing.

### Spectra and bridge check

specparam needs linearly spaced frequencies, so spectra were recomputed with Welch’s method (2 s Hann windows, 50% overlap, 0.5 Hz resolution, 1–60 Hz) instead of the log-spaced wavelets of notebook 01.

To check that the method change alone does not alter the result, the theta/alpha test of notebook 01 was rerun on the Welch spectra. It reproduces the null: no significant cluster and no FDR channel in either group or band. Alpha t-values are close to the wavelet results. Theta t-values are higher (maximum t 2.83 vs 2.27 in experts; 2.93 vs 1.71 in non-experts), plausibly because Welch separates 4–7 Hz more sharply than short wavelets do. Results after specparam are therefore compared against the Welch results, not against notebook 01.

### Software

Spectral parameterization with specparam 2.0.0rc7 (installed with `pip install specparam==2.0.0rc7`). According to its documentation, results are equivalent to the stable fooof 1.1 for the same model and settings.

### Pre-specified settings

These settings were fixed before fitting any subject-level spectra. Choices 1 and 2 were based only on the grand-average spectrum, which pools both conditions.

1.  **Fitting range: 3–40 Hz.**
    - The lower limit is above the influence of the 2 Hz high-pass filter.
    - The upper limit is below the 50 Hz notch and the 52 Hz anti-aliasing low-pass recorded in the file header. That filter’s gradual roll-off attenuates the top of the range by about 2 dB, equally in both conditions.
2.  **Aperiodic mode: `knee`, with the knee estimated freely.** On the grand average, the knee model fitted better than `fixed` (R² .997 vs .989; mean absolute error 0.026 vs 0.055), especially at 3–7 Hz, where a straight background line overshot the spectrum.
3.  **Peak settings as in Donoghue et al. (2020) for 64-channel BioSemi EEG:** `peak_width_limits = [1, 6]`, `max_n_peaks = 6`, `min_peak_height = 0.05`, `peak_threshold = 1.5`.
4.  **Quality control:** fits more than 2.5 SD from the mean on R² or error are flagged and counted, not excluded.
5.  **Unit of fitting:** the epoch-averaged spectrum per subject × condition × channel.
6.  **Primary outcomes:** periodic theta (4–7 Hz) and periodic alpha (9–11 Hz).
    - Each is the mean of the aperiodic-removed (flattened) log spectrum within the band, expressed in dB.
    - Tests are the same as in notebook 01: paired cluster permutation test and Benjamini–Yekutieli FDR, run separately per group and band.
7.  **Descriptive only:**
    - the meditation − mind-wandering differences in aperiodic offset, exponent and knee frequency (knee^(1/exponent));
    - the percentage of fits with a detected peak in each band, per condition.

### Quality control and sensitivity analysis

- **Failed fits.** Two fits failed entirely (sub-018, mind wandering, C6; sub-023, meditation, O1). sub-018 is not tested, because it has no meditation epochs. The value for sub-023 was replaced with the mean of the same subject and condition across channels, following Donoghue et al. (2020).
- **Negative knees.** 160 of 2,688 fits (6.0%) had a negative knee, mostly in sub-002 and sub-007. Their fits were still good (median R² 0.973). Knee frequency is undefined for these fits; they are reported, not excluded.
- **Flagged fits.** 133 of 2,688 fits were more than 2.5 SD from the mean on R² or error. 92 of these come from two subject × condition averages that each rest on a single epoch: sub-009, mind wandering (47 flagged; 1 epoch versus 61 meditation epochs) and sub-007, mind wandering (45 flagged; 1 epoch versus 19 meditation epochs). The next-lowest counts are 3 mind-wandering epochs (sub-004, sub-008).
- **Sensitivity analysis.** Added after quality control and before any tests were run: the four tests are repeated without subjects with fewer than 3 epochs in either condition, which removes sub-007 and sub-009, in this notebook and as an additional robustness check of the replication (notebook 01). The main analyses keep both, following the original exclusion rule (no trials in a condition).

### Results

**Periodic theta and alpha (primary tests).** No significant meditation vs. mind-wandering difference in either group or band (no cluster p \< .05, no FDR channel). Compared with total power from the same Welch spectra:

| Test               | Total power (Welch): min cluster p / max t / t at FCz | Periodic power (specparam): min cluster p / max t (channel) / t at FCz |
|--------------------|-------------------------------------------------------|------------------------------------------------------------------------|
| Experts, theta     | .31 / 2.83 / 2.47                                     | .59 / 2.39 (F5) / 0.31                                                 |
| Experts, alpha     | .19 / 2.86 / 1.05                                     | .36 / 2.58 (P9) / −0.09                                                |
| Non-experts, theta | .55 / 2.93 / 0.18                                     | .27 / 3.83 (FT7) / 0.06                                                |
| Non-experts, alpha | no cluster / 0.37 / −1.61                             | no cluster / 1.15 (T8) / −0.79                                         |

The non-significant frontal-midline theta tendency in experts’ total power (meditation \> mind wandering, the direction reported in the paper) disappears once the aperiodic background is removed.

**Sensitivity analysis without sub-007 and sub-009 (both experts).** Still no significant result in any test. Expert theta: smallest cluster p .27, max t 2.63, t at FCz 0.59. Expert alpha: smallest cluster p .059, max t 3.42, t at FCz 0.50; close to the threshold, but no cluster reaches p \< .05 and no channel survives FDR. Non-expert tests are unchanged.

**Aperiodic parameters (descriptive).** The meditation − mind-wandering differences in offset, exponent and knee frequency are small relative to their between-subject SD in both groups, at FCz and across the whole head (channel median). Knee frequency is undefined for negative knees. 16 fits had a knee frequency above 40 Hz, outside the fitting range (sub-018, 019, 023); this is why whole-head summaries use the median across channels.

**Post-hoc explanatory descriptive: theta split into periodic and aperiodic parts at FCz.** Added after the primary tests. In log power, total theta = periodic theta + background theta, where the background is reconstructed from each fit’s aperiodic parameters over 4–7 Hz. Meditation − mind-wandering differences in dB:

| Group               | Total: mean (SD), d_z | Periodic: mean (SD), d_z | Background: mean (SD), d_z |
|---------------------|-----------------------|--------------------------|----------------------------|
| Experts (n = 12)    | 0.67 (0.88), 0.75     | 0.07 (0.83), 0.09        | 0.59 (1.23), 0.48          |
| Non-experts (n = 8) | −0.45 (1.27), −0.36   | 0.02 (0.93), 0.02        | −0.47 (1.97), −0.24        |

In both groups, almost all of the total-theta condition difference at FCz sits in the aperiodic background, and the periodic difference is close to zero. The background difference shows up although offset and exponent show none on their own: in knee mode the parameters trade off, so the background level within 4–7 Hz is a better summary than any single parameter. These are descriptive, post-hoc values from one channel, with large between-subject variability.

**Detected peaks (descriptive).** Percentage of fits with a peak centred in each band, tested subjects only:

| Group       | Condition      | Theta peak, all channels | Alpha peak, all channels | Theta peak, FCz | Alpha peak, FCz |
|-------------|----------------|--------------------------|--------------------------|-----------------|-----------------|
| Experts     | Meditation     | 24.5%                    | 88.8%                    | 41.7% (5/12)    | 83.3%           |
| Experts     | Mind wandering | 19.9%                    | 88.2%                    | 41.7% (5/12)    | 66.7%           |
| Non-experts | Meditation     | 5.3%                     | 87.1%                    | 0% (0/8)        | 100%            |
| Non-experts | Mind wandering | 4.5%                     | 90.4%                    | 0% (0/8)        | 87.5%           |

- **Alpha peaks** are present in most fits in both groups and conditions.
- **Theta peaks** are rare in non-experts. In experts they are detected in about a fifth to a quarter of fits overall, and in about 40% of fits at FCz.
- **The difference between conditions is small in both groups.** The larger contrast is between groups, in both conditions, which suggests a more trait-like than state-like feature. Group differences were not tested, and this pattern is descriptive only.
- **Most detected theta peaks are not at the band edge.** Detected theta peaks are centered mostly between 5.5 and 6.5 Hz rather than near 7 Hz, which suggests that most of them are distinct theta peaks rather than the lower flank of the alpha peak. This cannot be fully ruled out for participants with slow or broad alpha.

### Known limitations of these choices

- **The knee falls outside the validated range.** The grand-average knee is about 19.5 Hz (knee parameter about 7 × 10⁶, exponent 5.3), far beyond the knee values in the original validation simulations (0–150). The knee-mode aperiodic parameters are therefore not interpreted on their own, and are not comparable with fixed-mode exponents in the literature.
- **The knee-mode exponent is steeper than the data.** It overstates the 20–40 Hz slope (about 5.3 vs about 3.7 read from the spectrum), and the model compensates with peaks at the upper edge of the range (33 and 40 Hz). This mainly affects the fit above 20 Hz, not the background under 4–11 Hz.
- **Periodic theta may partly reflect alpha.** Periodic theta is the mean of the flattened spectrum over 4–7 Hz, so it includes any power above the background in that band, not only detected theta peaks. In the grand average, alpha rises from about 5 Hz. This partly reflects averaging participants with different alpha frequencies, but alpha’s lower flank may still contribute to periodic theta in some participants. The paper’s band was kept for comparability, and no alternative band is tested.
- **Periodic theta can be negative.** In the grand average, the spectrum lies slightly below the fitted background at about 4–6 Hz, so periodic theta is expected to be close to zero or slightly negative.

## Repository structure

- `01_Meditation_reanalysis.ipynb`: preprocessing, main analysis and robustness checks 1–5.
- `02_Meditation_aperiodic.ipynb`: Welch spectra, bridge check, specparam fits, tests and descriptives.
- `requirements.txt`: Python packages and versions.
- `LICENSE`: MIT licence.

Raw EEG, preprocessed data and intermediate files (.fif, .npz) are not included. The raw data are openly available on OpenNeuro.

## Requirements

openneuro-py, mne, mne-bids, pyprep, mne-icalabel (with torch as backend), autoreject, specparam (2.0.0rc7), numpy, pandas, scipy, matplotlib.

Developed and run in Google Colab (Python 3.13). A `requirements.txt` with these packages is included in the repository root.

## Running the notebooks

- `01_Meditation_reanalysis`: main analysis and robustness checks 1–5. `02_Meditation_aperiodic`: spectra, bridge check, specparam, tests and descriptives. Notebook 02 reads the main epochs saved by notebook 01.
- Both notebooks were run in Google Colab with intermediate files on Google Drive. Change the folder paths at the top of each notebook to your own location.
- The data are downloaded from OpenNeuro (ds001787, v1.1.1) by notebook 01 and are not included in this repository.
- The slow steps (ICA, epoching) save their outputs, so later cells can be rerun without recomputing them.

## Note on scope

This reanalysis uses an automated pipeline in place of the original manual cleaning, so a divergence from the published results may reflect preprocessing differences that could not be reproduced from the shared data. It is not a re-examination of the original authors’ analysis, and it was only possible because they made their data openly available.

## AI assistance

The code was written by the author with the assistance of Claude Opus 5.5 (Anthropic), which was also used to compare the pipeline against the methods of Brandmeyer & Delorme (2018), debug it, restructure the robustness checks and draft this README. The author made all analytic decisions and takes responsibility for the content.

## Key references

- Brandmeyer, T., & Delorme, A. (2018). Reduced mind wandering in experienced meditators and associated EEG correlates. *Experimental Brain Research, 236*(9), 2519–2528. https://doi.org/10.1007/s00221-016-4811-5 (published online 2016)
- Maris, E., & Oostenveld, R. (2007). Nonparametric statistical testing of EEG- and MEG-data. *Journal of Neuroscience Methods, 164*(1), 177–190.
- Benjamini, Y., & Yekutieli, D. (2001). The control of the false discovery rate in multiple testing under dependency. *The Annals of Statistics, 29*(4), 1165–1188.
- Donoghue, T., et al. (2020). Parameterizing neural power spectra into periodic and aperiodic components. *Nature Neuroscience, 23*(12), 1655–1665.
- Bigdely-Shamlo, N., et al. (2015). The PREP pipeline: standardized preprocessing for large-scale EEG analysis. *Frontiers in Neuroinformatics, 9*, 16.
- Pion-Tonachini, L., Kreutz-Delgado, K., & Makeig, S. (2019). ICLabel: An automated electroencephalographic independent component classifier, dataset, and website. *NeuroImage, 198*, 181–197.
- Lee, T.-W., Girolami, M., & Sejnowski, T. J. (1999). Independent component analysis using an extended infomax algorithm for mixed subgaussian and supergaussian sources. *Neural Computation, 11*(2), 417–441.
- Jas, M., et al. (2017). Autoreject: Automated artifact rejection for MEG and EEG data. *NeuroImage, 159*, 417–429.
- Gramfort, A., et al. (2013). MEG and EEG data analysis with MNE-Python. *Frontiers in Neuroscience, 7*, 267.
- Appelhoff, S., et al. (2019). MNE-BIDS: Organizing electrophysiological data into the BIDS format and facilitating their analysis. *Journal of Open Source Software, 4*(44), 1896.

## Author

Eva Villar, evagarciavillar12@gmail.com
