# SafePath: VRU Intent & Trajectory Prediction System

**GitHub Repository:** [Insert Public GitHub Link Here]

Demo: https://intent-and-trajectory-prediction-dq.vercel.app/

## Project overview

SafePath is a multimodal trajectory forecasting system for **vulnerable road users (VRUs)** in urban traffic scenes. The project focuses on predicting the future motion and intent of:

- pedestrians
- cyclists

Given a short observed motion history, local map context, and nearby road-user interactions, the system predicts:

- `K=3` future trajectories over 6 seconds
- a confidence score for each trajectory
- an intent label:
  - `crossing`
  - `waiting`
  - `turning`
  - `walking_straight`
- a lightweight risk reading for demo interpretation

This project was built as a hackathon-ready prototype for safer autonomous mobility and explainable road-user behavior forecasting.

## Model architecture

The final model combines trajectory, map, social, and agent-type information.

### Inputs

- Past trajectory history over 4 timesteps at 2 Hz
- Kinematic features: `(x, y, vx, vy, ax, ay)`
- Local `100x100` map patch
- Nearby pedestrians/cyclists within a `10m` radius
- Agent type embedding:
  - pedestrian
  - cyclist

### Architecture Components

- **Transformer trajectory encoder**
  Encodes temporal motion history.

- **CNN map encoder**
  Extracts local spatial context from a grayscale map patch.

- **GAT-style social encoder**
  Models interactions with nearby vulnerable road users. used instead of social pooling.

- **Agent-type embedding**
  Explicitly distinguishes pedestrians from cyclists.

- **Intent classification head**
  Predicts one of 4 intent classes.

- **Endpoint prediction head**
  Predicts `K=3` future endpoint hypotheses.

- **Goal-conditioned multimodal decoder**
  Generates `K=3` full future trajectories conditioned on predicted endpoints.

- **Probability head**
  Assigns confidence to each predicted trajectory mode.

## Dataset used

This project uses the **nuScenes mini** dataset.

- Dataset type: public
- Domain: autonomous driving / urban traffic scenes
- Target agents used in this project:
  - pedestrians
  - cyclists

### Data Preparation

The pipeline reads raw nuScenes JSON tables directly and:

- extracts pedestrian and cyclist tracks
- builds sliding-window trajectory samples
- computes velocity and acceleration features
- extracts local map patches
- collects nearby road users for social context
- converts trajectories into local and heading-aligned coordinates

### Train / Validation Split

This prototype uses a corrected **manual scene-level split** on nuScenes mini.

- Train samples: `1507`
- Validation samples: `807`
- Excluded scene: `scene-1077` because it contains no pedestrian data

There is **no separate test split** in the current prototype.

## Setup & installation instructions

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd "Mobility challenge"
```

### 2. Install dependencies

```bash
python3 -m pip install -r requirements.txt
```

### 3. Prepare the dataset

Place the downloaded nuScenes mini dataset so the JSON tables are available at:

```text
/path/to/v1.0-mini/v1.0-mini/*.json
```

In this local project setup, the code has been used with:

```text
/Users/chaku/Desktop/Mobility challenge/v1.0-mini
```

## How to run the code

### Train the final Phase 4 model

```bash
python3 train_phase4.py \
  --data-root "/path/to/v1.0-mini" \
  --epochs 20 \
  --batch-size 32 \
  --heading-aligned \
  --use-agent-type-embedding \
  --save-dir checkpoints/vru_manualsplit_intent_type
```

### Evaluate the model

```bash
python3 evaluate_phase4.py \
  --checkpoint checkpoints/vru_manualsplit_intent_type/phase4_multimodal_best.pt \
  --data-root "/path/to/v1.0-mini" \
  --split val \
  --batch-size 32
```

### Export predictions

```bash
python3 predict_phase4.py \
  --checkpoint checkpoints/vru_manualsplit_intent_type/phase4_multimodal_best.pt \
  --data-root "/path/to/v1.0-mini" \
  --split val \
  --output outputs/final_predictions_type.json
```

### Generate report assets

```bash
python3 report_phase4.py \
  --checkpoint checkpoints/vru_manualsplit_intent_type/phase4_multimodal_best.pt \
  --data-root "/path/to/v1.0-mini" \
  --split val \
  --output-dir outputs/phase4_report_type_final
```

### Generate visualization examples

```bash
python3 visualize_phase4.py \
  --checkpoint checkpoints/vru_manualsplit_intent_type/phase4_multimodal_best.pt \
  --data-root "/path/to/v1.0-mini" \
  --split val \
  --num-samples 5 \
  --output-dir outputs/visualizations_type_final
```

### Run the demo website locally

First build website data:

```bash
python3 build_demo_data.py \
  --predictions outputs/final_predictions_type.json \
  --data-root "/path/to/v1.0-mini" \
  --split val \
  --heading-aligned \
  --output demo_site/data/demo_data.json
```

Then serve the site:

```bash
cd demo_site
python3 -m http.server 8000
```

Open:

```text
http://127.0.0.1:8000/
```

## Example outputs / results

The final selected model is the **20-epoch type-aware checkpoint** with explicit pedestrian/cyclist embedding.

### Final Validation Results

| Metric | Value |
|---|---:|
| Oracle minADE | 0.8392 |
| Oracle minFDE | 1.6619 |
| Oracle Miss Rate | 0.1809 |
| Top-1 ADE | 1.0384 |
| Top-1 FDE | 2.0585 |
| Top-1 Miss Rate | 0.2714 |
| Intent Accuracy | 0.7720 |

### Result Summary

- The model generates multiple plausible futures instead of a single rigid prediction.
- The top-ranked future improved significantly after adding the pedestrian/cyclist agent-type embedding.
- Intent classification reached **77.2%** validation accuracy on the corrected validation split.
- The demo website presents predictions in plain language and distinguishes pedestrians from cyclists.

## Repository Structure

- `trajectory_baseline/`
  Core dataset and model code.
- `train_phase1.py`, `train_phase2.py`, `train_phase3.py`, `train_phase4.py`
  Training scripts for each development phase.
- `evaluate_phase4.py`
  Evaluation script for the final model.
- `predict_phase4.py`
  JSON export script for predicted trajectories and intent.
- `visualize_phase4.py`
  Saves trajectory visualization PNGs.
- `report_phase4.py`
  Builds report-ready charts and metrics files.
- `build_demo_data.py`
  Converts prediction outputs into website-friendly demo data.
- `demo_site/`
  Static website for interpretable visualization and risk display.

## Important Notes

- Intent labels are **heuristic** and derived from motion patterns, not from direct nuScenes intent annotations.
- The project is designed as a strong prototype and not a production-ready autonomous driving stack.
- The current map patch extraction is a lightweight approximation to keep the system fully runnable without `nuscenes-devkit`.
