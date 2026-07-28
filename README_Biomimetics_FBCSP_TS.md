# FBCSP-TS: Robust Motor Imagery BCI Classification Under Signal Degradation

Official implementation of **"Robust Motor Imagery–Brain–Computer Interface Classification in Signal Degradation: A Multi-Window Ensemble Approach"**
*Biomimetics*, vol. 10, no. 12, p. 832, 2025.

[![Paper](https://img.shields.io/badge/DOI-10.3390%2Fbiomimetics10120832-blue)](https://doi.org/10.3390/biomimetics10120832)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

---

## Overview

**FBCSP-TS** (Filter Bank Common Spatial Pattern with Time Segmentation) extends FBCSP by adding a temporal dimension: EEG trials are split into overlapping time segments, FBCSP features are extracted independently from each segment, and segment-level predictions are merged via **soft voting**.

The motivation is deployment realism. Mobile, wearable, and low-cost EEG systems operate at reduced sampling frequencies, which erodes the μ/β oscillatory structure that motor imagery decoding depends on. By accumulating spatial → spectral → temporal features step by step, FBCSP-TS retains usable accuracy far deeper into the degradation curve than CSP or FBCSP.

**Key points**

- Cumulative feature design: CSP (spatial) → FBCSP (+spectral) → FBCSP-TS (+temporal)
- Grid search over 64 temporal configurations (window 0.5–4.0 s × moving 0.5–4.0 s)
- Systematic degradation study: 25 sampling rates from 250 Hz down to 10 Hz
- Soft voting across segments for resilience to transient noise and local artifacts
- External validation on an independent dataset recorded at a different sampling rate

---

## Results

### Degradation robustness (BCI Competition IV Dataset 2a)

| Method | Median accuracy | Performance retained down to |
|---|---|---|
| CSP | 0.58 | 160 Hz |
| FBCSP | 0.65 | 160 Hz |
| **FBCSP-TS** | **0.68** | **110 Hz** |

A paired *t*-test with Bonferroni correction confirms that accuracy at 110 Hz is not significantly different from 250 Hz for FBCSP-TS.

Optimal temporal parameters: **window length 3.5 s, moving length 0.5 s**.

### External validation (BCI Competition IV Dataset 1, 100 Hz)

| Subject | CSP acc | CSP κ | FBCSP acc | FBCSP κ | FBCSP-TS acc | FBCSP-TS κ |
|---|---|---|---|---|---|---|
| a | 0.542 | 0.095 | 0.785 | 0.570 | **0.840** | 0.680 |
| b | 0.527 | 0.061 | 0.534 | 0.070 | **0.644** | 0.290 |
| c | 0.487 | −0.019 | **0.715** | 0.430 | 0.705 | 0.410 |
| d | 0.502 | 0.001 | **0.860** | 0.720 | 0.805 | 0.610 |
| e | 0.472 | −0.042 | 0.865 | 0.730 | **0.909** | 0.820 |
| f | 0.577 | 0.161 | 0.865 | 0.730 | **0.875** | 0.750 |
| g | 0.472 | −0.030 | 0.835 | 0.670 | **0.890** | 0.779 |
| **mean** | 0.511 | 0.032 | 0.779 | 0.560 | **0.809** | **0.619** |

FBCSP-TS achieves the best accuracy on 5 of 7 subjects, including all four non-synthetic subjects.

---

## Datasets

Both datasets are publicly available from the [BCI Competition IV archive](https://www.bbci.de/competition/iv/).

| | **Dataset 2a** (training) | **Dataset 1** (external validation) |
|---|---|---|
| Subjects | 9 healthy | 7 (4 real: a, b, f, g / 3 synthetic: c, d, e) |
| Classes | 4 (left hand, right hand, foot, tongue) | 2 of {left hand, right hand, foot} |
| Trials | 288 (72 per class) | 200 (100 per class) |
| Channels | 22 Ag/AgCl | 59 |
| Sampling rate | 250 Hz | 100 Hz (downsampled version used) |
| Trial shape | (288, 22, 1000) | (200, 59, 1000) |

> Datasets are **not** redistributed here. Download them and place them under `data/`.

---

## Repository structure

```
.
├── data/
│   ├── bcic_iv_2a/           # training dataset (not tracked)
│   └── bcic_iv_1/            # external validation set (not tracked)
├── src/
│   ├── preprocessing.py      # 8–32 Hz Butterworth, 4 s trial windowing
│   ├── resampling.py         # MNE FFT-based resampling, 10–250 Hz
│   ├── csp.py                # baseline CSP
│   ├── fbcsp.py              # 7-subband filter bank + MI feature selection
│   ├── fbcsp_ts.py           # proposed method (time segmentation + soft voting)
│   ├── classifier.py         # SVM (RBF), 5-fold CV
│   └── stats.py              # paired t-test with Bonferroni correction
├── experiments/
│   ├── run_segmentation_grid.py
│   ├── run_degradation_sweep.py
│   └── run_external_validation.py
├── notebooks/
│   ├── 01_tsne_visualization.ipynb
│   ├── 02_segmentation_heatmap.ipynb
│   └── 03_confusion_matrices.ipynb
├── results/
├── requirements.txt
└── README.md
```

---

## Installation

```bash
git clone https://github.com/<user>/<repo>.git
cd <repo>
conda create -n fbcsp-ts python=3.10
conda activate fbcsp-ts
pip install -r requirements.txt
```

Main dependencies: `numpy`, `scipy`, `mne==1.9.0`, `scikit-learn`, `matplotlib`, `seaborn`.

---

## Usage

### 1. Preprocessing

Band-pass Butterworth 8–32 Hz (μ and β bands, ocular noise suppressed), 4 s MI-related trial window.

```bash
python src/preprocessing.py --dataset bcic_iv_2a --out data/processed
```

### 2. Sampling-frequency degradation sweep

Resampling uses MNE-Python's frequency-domain (FFT interpolation with built-in anti-aliasing) method, applied along the time axis:

```
N_resampled = N_original × (f_target / f_original)
```

```bash
python experiments/run_degradation_sweep.py \
    --methods csp fbcsp fbcsp_ts \
    --rates 10:250:10
```

### 3. Temporal segmentation grid search

```bash
python experiments/run_segmentation_grid.py \
    --window-range 0.5 4.0 --window-step 0.5 \
    --moving-range 0.5 4.0 --moving-step 0.5
```

Produces the 8 × 8 accuracy heatmap; the peak (0.61 on Dataset 2a) occurs at window 3.5 s / moving 0.5 s.

### 4. External validation

```bash
python experiments/run_external_validation.py \
    --window 3.5 --moving 0.5 --dataset bcic_iv_1
```

---

## Method configuration

| Component | Setting |
|---|---|
| Band-pass (preprocessing) | 8–32 Hz Butterworth |
| CSP baseline band | 7–35 Hz |
| Filter bank | 4–8, 8–12, 12–16, 16–20, 20–24, 24–28, 28–32 Hz |
| Feature selection | mutual-information based |
| Classifier | SVM, RBF kernel |
| Validation | 5-fold cross-validation |
| Segment fusion | soft voting (mean predicted probability) |
| Metrics | accuracy, Cohen's kappa |

Note on the grid: when window length + moving length exceeds the 4 s trial, no further segmentation is possible, so those cells reproduce the single-window result.

---

## Citation

```bibtex
@article{lee2025fbcspts,
  title   = {Robust Motor Imagery--Brain--Computer Interface Classification in Signal Degradation: A Multi-Window Ensemble Approach},
  author  = {Lee, Dong-Geun and Lee, Seung-Bo},
  journal = {Biomimetics},
  volume  = {10},
  number  = {12},
  pages   = {832},
  year    = {2025},
  doi     = {10.3390/biomimetics10120832}
}
```

---

## Funding

Supported by the Bisa Research Grant of Keimyung University, 2024 (project number 20240421).

## License

Code released under the MIT License. The associated article is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

## Contact

Dong-Geun Lee — Department of Medical Informatics, Keimyung University School of Medicine, Daegu, Republic of Korea
Corresponding author: Seung-Bo Lee
