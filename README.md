# KPConv — LiDAR Semantic Segmentation + AV Path Planning

3D point cloud segmentation on SemanticKITTI using KPConv, with A\* path planning and a GT vs Prediction driving simulation.

![AV Simulation](results/simulation.gif)

---

## What this does

```
LiDAR scan → KPConv segmentation → Occupancy grid → A* path → Navigation decision
```

Two vehicles simulate driving through the scene simultaneously — one using ground truth labels, one using model predictions. Where they diverge = where the model gets it wrong.

**91.7% direction agreement** between GT and predicted navigation across 204 validation frames.

---

## Results

| Run | Data | Epochs | mIoU |
|-----|------|--------|------|
| v1 | Subsampled 16% | 200 | 44.7% |
| v2 | Subsampled 16% | 450 | 54–55% ⚠️ hallucinating |
| v3 | Subsampled(Best epoch) | 257 | **58.1%** ✅ |
| v4 | Full dataset | 500 | **56.1%** ✅ |
| Official KPConv paper | Full dataset | 800 | 58.8% |

**Interesting finding:** subsampled data (v4, 257 epochs) outperformed the full dataset run (v5, 500 epochs). See [why below](#why-did-subsampled-data-outperform-full-data).

### AV-Critical Class IoU (v5)

| Class | IoU | Class | IoU |
|-------|-----|-------|-----|
| car | 0.945 | road | 0.887 |
| building | 0.885 | vegetation | 0.873 |
| sidewalk | 0.736 | bicyclist | 0.742 |
| trunk | 0.685 | person | 0.590 |
| bicycle | 0.348 | motorcyclist | 0.001 |

---

## Training Journey

**v1 — First run on subsampled data**
Used stride-based sampling to keep ~16% of SemanticKITTI (3,784 scans). Rare classes like cyclists and pedestrians were oversampled using a smaller stride. Reached 44.7% mIoU at 200 epochs.

**v2 & v3 — Longer training, same data**
Continued to 450 epochs. mIoU hit 54–55% but the model started hallucinating rare classes — predicting cyclists/motorcyclists where none existed. Too confident, not enough rare class examples. Best mIoU 58.1 achieved at epoch 257.

Attempted to add more rare class scans from specific sequences. Didn't improve results, abandoned.

**v5 — Full dataset**
Switched to the complete 23,201 scan dataset. v4 (257 epochs) hit **58.1% mIoU** — surprisingly better than v5 (500 epochs, 56.1%). Model was still converging at 500 epochs and likely needs the full 800 epochs to match the official benchmark.

---

## Dataset Subsampling

SemanticKITTI records at 10Hz — consecutive frames are nearly identical. Subsampling removes redundancy:

```
Train : stride=10, rare_stride=8  →  2,913 scans
Val   : stride=10, rare_stride=6  →    871 scans
Test  : stride=12, rare_stride=10 →  held out
```

`stride=10` keeps every 10th scan. `rare_stride` is a smaller stride applied to scans containing rare classes so they appear more often in training.

To get more data just reduce the strides — `stride 10→6` roughly doubles dataset size.

---

## Key Observations

**Distance-based prediction degradation**
Close points are classified correctly. Far points (>30m) degrade — the model defaults to predicting "car" for sparse distant clusters because that's what most large sparse clusters were during training. This is a known LiDAR limitation.

**Motorcyclist near-zero IoU**
Motorcyclists are extremely rare in SemanticKITTI and visually similar to bicyclists. Even the official benchmark struggles here. Needs either far more training data or class merging.

**Navigation accuracy**
The A\* planner driven by predictions agreed with ground truth **91.7% of the time**. Disagreements mostly occurred on frames where rare classes were misclassified.

---

## Setup

```bash
pip install -r requirements.txt
```

Download SemanticKITTI from [semantic-kitti.org](http://www.semantic-kitti.org/dataset.html) and place at:
```
semkitti_raw/sequences/00–10/
```

Pretrained weights: **[Add Google Drive link here]**
Place at: `kpconv_results/Log_2026-04-05_run1/checkpoints/current_chkp.tar`

---

## Usage

Open `DL_FINAL.ipynb` and run the sections in order:

| Section | What it does |
|---------|-------------|
| 1 — Training | Train or resume KPConv |
| 2 — Visualisation | View predicted vs GT point clouds |
| 3 — Path Planning | A\* navigation on predicted occupancy grid |
| 4 — Simulation | GT vs Prediction side-by-side driving sim |
| 5 — Video | Export simulation as MP4 + GIF |

**To resume training from a checkpoint:**
```python
previous_training_path = 'Log_2026-04-05_run1'
config.max_epoch       = config.max_epoch + 50
config.lr_decays       = {i: 0.984767 for i in range(1, 300)}
config.saving_path     = '/your/drive/path/Log_2026-04-05_run1'
```

---

## What's NOT in this repo

| Item | Size | Where to get it |
|------|------|----------------|
| Model weights | ~500MB | Drive link above |
| val_preds/ (.ply files) | ~400MB | Run inference |
| SemanticKITTI dataset | ~80GB | semantic-kitti.org |
| Simulation HTML files | ~2GB | Run simulation |

---

## FAQ

**Why does the subsampled model (58.1%) beat the full dataset model (56.1%)?**
The subsampled dataset removed near-duplicate consecutive frames (LiDAR at 10Hz barely moves between scans). Fewer but more diverse scans per epoch = better generalisation. The full dataset model also needs more epochs — the official benchmark uses 800, we stopped at 500.

**Why is motorcyclist IoU basically 0%?**
Motorcyclists appear in very few scans across all of SemanticKITTI. The model sees so few examples it never reliably learns the class. Even the official KPConv benchmark has very low motorcyclist IoU. Fix: more training data or merging motorcyclist with bicyclist.

**Why do far away points all get predicted as "car"?**
LiDAR point density drops with distance. At 50m an entire car might have only 10 points — not enough geometry for the model to classify correctly. It defaults to the most common large-cluster class seen in training, which is "car". This is called distance-based prediction degradation.

**Can I use this on a different dataset?**
The architecture is dataset-agnostic but the class definitions, weights, and sampling are tuned for SemanticKITTI. You'd need to retrain with new class labels.

**How do I get more data without downloading the full dataset?**
Reduce the stride values in the subsampling config. `stride 10→6` roughly doubles the training set. `rare_stride 8→4` doubles rare class exposure.

**How does the path planning work?**
Predictions are projected onto a 2D occupancy grid (0.5m/cell). Traversable classes (road, sidewalk, terrain) = free. Obstacle classes (car, building, person, etc.) = blocked. A\* finds the shortest path from the vehicle to a goal 25m ahead. The scene is divided into LEFT/STRAIGHT/RIGHT zones and the zone with the best traversability score wins.

**Why 91.7% navigation agreement and not higher?**
The 8.3% disagreements mostly happen on frames where the model misclassifies an obstacle as road (or vice versa), changing the occupancy grid enough to flip the direction decision. Improving rare class IoU would close most of this gap.

**How long does training take?**
On a Colab A100, roughly 200 epochs ≈ 4–5 hours. Full 800 epochs ≈ 16–20 hours.

---

## References

- [KPConv paper](https://arxiv.org/abs/1904.08889) — Thomas et al., ICCV 2019
- [KPConv-PyTorch](https://github.com/HuguesTHOMAS/KPConv-PyTorch) — original implementation
- [SemanticKITTI](http://www.semantic-kitti.org/) — Behley et al., ICCV 2019

---

*Made as a Deep Learning course project — if you found it useful, give it a ⭐*
