# Using Brain Waves to Make a Robot Arm Move

A complete, end-to-end pipeline that decodes motor imagery intent from scalp EEG and translates classified mental commands into physical robot arm motion in a physics simulation. Raw EEG recordings are cleaned, epoched, and classified using Riemannian geometry and a support vector machine. The classified output drives a KUKA iiwa 7-DOF arm in PyBullet in real time.

## Intent

Conventional robot training pipelines rely on kinematic data — motion capture, joint encoders, or visual demonstrations — which require physical hardware, controlled environments, and significant setup cost. This project demonstrates that clinically available EEG data, recorded non-invasively during imagined movement, can serve as an alternative training signal. The methodology decodes four motor imagery classes (left hand, right hand, feet, tongue) from nine subjects using a cross-subject generalisation protocol, with no subject-specific hardware or calibration required beyond a small number of additional trials. The core claim is not accuracy — it is that the methodology is viable, reproducible, and accessible in a way that kinematic pipelines are not.

## Pipeline

```
Raw EEG (.gdf)
      │
      ▼
Continuous Preprocessing
  · 1 Hz high-pass FIR          — baseline stabilisation for ICA
  · ICA (15 components)         — automated EOG artifact rejection
  · 8–30 Hz bandpass FIR        — SMR / motor band isolation
  · Common Average Reference    — spatial noise attenuation
      │
      ▼
Epoching & Artifact Scrubbing
  · 0.5s–3.5s window post-cue  — excludes visual evoked potential
  · ±1000 sample artifact zone  — expert-flagged trial rejection
      │
      ▼
Riemannian Geometry + SVM
  · Ledoit-Wolf covariance estimation  (22×22 SPD matrix per trial)
  · Tangent space projection           (curved manifold → Euclidean)
  · Linear SVM (C=1.0)
  · LOSO cross-validation              (9 folds, one subject held out per fold)
      │
      ▼
Dynamic Calibration
  · Few-shot adaptation: 3–15 trials per class injected from test subject
  · Iterative until accuracy ≥ 65% or shot budget exhausted
      │
      ▼
PyBullet Robot Arm Simulation
  · KUKA iiwa 7-DOF, URDF physics
  · Majority-vote buffer (5 epochs) — command smoothing
  · 4 classes → 4 distinct joint velocity commands
```

## Results

| Protocol | Mean Accuracy | Std |
|---|---|---|
| Generalised LOSO (zero calibration) | 36.20% | ±10.58% |
| Dynamic Calibration (≤15 shots/class) | 51.67% | ±11.74% |

Chance level for 4-class classification is 25%. Both protocols exceed chance across all nine subjects. Subject-level variance is high, which is consistent with published cross-subject motor imagery literature on this dataset.

## Pipeline QA — Subject A08T

The figure below shows the signal state at each stage of the pipeline for a randomly selected subject, plus the Riemannian feature space the classifier operates in.

![Pipeline QA](assets/pipeline_qa_A08T.png)

**Stage 1 — Raw EEG:** Broadband signal with visible high-amplitude transients from eye blinks and saccades propagating across frontal channels.

**Stage 2 — FIR Filtered:** Slow drift and high-frequency noise removed. Signal amplitude compressed into the 8–30 Hz motor band. EOG transients partially attenuated but not eliminated by spectral filtering alone.

**Stage 3 — ICA Cleaned:** EOG components identified via cross-correlation with mapped EOG channels and mathematically subtracted. Traces are markedly smoother with sharp blink deflections visibly removed.

**Stage 4 — Mean Covariance Matrix:** The 22×22 Ledoit-Wolf shrinkage covariance matrix averaged across all trials. Strong diagonal dominance confirms variance is concentrated in individual channels rather than diffuse noise. Off-diagonal structure reflects genuine spatial correlation between anatomically adjacent electrodes.

**Stage 4 — Tangent Space Scatter:** Each point is one trial projected from the Riemannian manifold into 2D via PCA. PC1 (28.9% variance) separates left hand (blue) from right hand (pink) — the contralateral SMR suppression pattern the classifier relies on. Feet (green) and tongue (orange) cluster in the positive PC1 region, consistent with their distinct spatial ERD patterns. The 2D projection represents a lower bound on separability — the SVM operates in the full 253-dimensional tangent space.

## Simulation Output

<!-- Replace with GIF or video link once recorded -->
*Simulation video coming soon. Run the final cell with PyBullet GUI enabled to reproduce.*

The arm demonstrates four movement classes driven by classified EEG intent:

| Class | Motor Imagery | Joint | Movement |
|---|---|---|---|
| 1 | Left hand | Base (joint 0) | Rotate left |
| 2 | Right hand | Base (joint 0) | Rotate right |
| 3 | Both feet | Elbow (joint 3) | Extend up |
| 4 | Tongue | Base + Elbow | Retract both |

Evaluation data is sorted by class before inference to simulate sustained, continuous motor intent — the arm holds each movement direction in blocks rather than switching randomly, which reflects how a real BCI user would perform a sustained imagery task.

## Environment Setup

```bash
conda env create -f environment.yml
conda activate eeg-bci
```

## Dataset

BCI Competition IV Dataset 2a is required to run the preprocessing stages. Download from the official BCI Competition IV archive and place `.gdf` files in a `raweegdata/` directory.

> Brunner, C., Leeb, R., Müller-Putz, G., Schlögl, A., & Pfurtscheller, G. (2008). *BCI Competition 2008 – Graz Data Set A*. Institute for Knowledge Discovery, Graz University of Technology.

## Status

- [x] Preprocessing pipeline
- [x] Epoching & artifact scrubbing
- [x] Riemannian geometry + SVM classification
- [x] Dynamic calibration
- [x] PyBullet simulation
- [x] Pipeline QA visualisation
- [ ] Simulation video
- [ ] Literature review & citations
- [ ] White paper
