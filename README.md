# SpaGCN — Spatial Transcriptomics Clustering with Graph Convolutional Networks

Integrating gene expression, spatial coordinates, and histology imaging to recover
tissue architecture from 10x Visium spatial transcriptomics data — and testing
whether histology actually helps.

## Overview

Spatial transcriptomics captures gene expression *and* where in the tissue that
expression happens, opening the door to methods that use spatial context — not just
gene expression alone — to identify biologically meaningful regions. **SpaGCN**
(Spatial Graph Convolutional Network) does this by building a single graph that fuses
three data modalities before any learning happens:

- **Gene expression** — what's being transcribed at each spot
- **Spatial coordinates** — where each spot physically sits on the tissue
- **Histology image features** — what the tissue looks like under the microscope at
  that spot

This is an **early fusion** approach: rather than clustering on expression alone and
checking histology afterwards, all three signals are combined into one weighted graph
*before* a GCN aggregates information from neighbouring spots. The resulting spot
embeddings are then grouped into clusters using an unsupervised iterative clustering
algorithm — ideally recovering real anatomical tissue layers without ever being told
where those layers are.

## Dataset

10x Visium spatial transcriptomics data from the **dorsolateral prefrontal cortex
(DLPFC)** of postmortem human brain tissue:

| | |
|---|---|
| Spatial spots | 3,431 |
| Genes | 33,538 (before filtering) |
| Ground truth | 7 tissue layers (Layer 1–6 + White Matter) |
| Modalities | Gene expression, spatial coordinates, H&E histology image |

## Preprocessing

- Removed low-quality/non-informative genes (present in fewer than 3 spots)
- Normalised expression per cell and log-transformed
- **18,640 genes × 3,431 spots** remained after filtering
- Data was reloaded fresh at the start of every run to keep comparisons across
  parameter settings fair and independent

## Method — Testing the Role of Histology

The key question this project investigates: **does adding histology information
actually improve clustering?** To test it, SpaGCN was run with three settings of the
histology weighting parameter **s**:

| `s` value | Meaning |
|---|---|
| `s = 0` | No histology — spatial coordinates + gene expression only |
| `s = 1` | Moderate histology contribution |
| `s = 10` | Strong histology dominance |

For each value of `s`:
1. An adjacency matrix was computed from the fused graph
2. The bandwidth parameter `l` was tuned by searching for the value producing a
   target neighbourhood proportion of 0.5
3. SpaGCN was trained with **k-means initialisation** for 200 epochs
   (`tol=5e-3`, `lr=0.05`), targeting **7 clusters** to match the 7 ground-truth layers
4. A **hexagonal refinement** step was applied afterward to smooth cluster boundaries
   (since Visium spots sit on a hexagonal grid)

## Results

### Overall clustering performance

| `s` value | ARI | FMI | AMI |
|---|---|---|---|
| **0** | **0.2912** | **0.4063** | **0.4524** |
| 1 | 0.2335 | 0.3594 | 0.3940 |
| 10 | 0.2301 | 0.3556 | 0.3888 |

**`s = 0` — no histology at all — gave the best clustering performance** across every
metric. Adding histology (`s = 1`) reduced performance, and weighting it even more
heavily (`s = 10`) made it worse still.

### Cluster visualisations

| Setting | Figure |
|---|---|
| `s = 1` vs. ground truth | `figures/clusters_s1.png` |
| `s = 0` vs. ground truth | `figures/clusters_s0.png` |
| `s = 10` vs. ground truth | `figures/clusters_s10.png` |

### Per-layer recovery (% correctly recovered)

| Ground Truth Layer | s = 0 | s = 1 | s = 10 |
|---|---|---|---|
| Layer 1 | **62.63** | 45.33 | 56.40 |
| Layer 2 | 44.49 | 42.91 | 39.37 |
| Layer 3 | 42.82 | 42.82 | 41.75 |
| Layer 4 | 43.31 | 43.70 | 43.70 |
| Layer 5 | 36.98 | 27.73 | 32.82 |
| Layer 6 | 46.92 | **48.21** | 42.69 |
| White Matter | **74.11** | 57.41 | 56.47 |

- **White Matter** was the most reliably recovered layer across all settings — its
  transcriptomic profile is distinct enough from the cortical layers that spatial or
  histology cues added little extra benefit.
- **Layer 1** improved sharply from `s=1` to `s=0` (45.33% → 62.63%).
- **Layer 5** was the hardest to recover under every setting, likely due to overlapping
  gene expression profiles and ambiguous histological appearance at that boundary.

## Discussion

The intuitive expectation going in was that adding histology information should help
— more data, more signal. The results say otherwise: for this DLPFC sample, the
**transcriptomic and spatial signal alone already captured most of the tissue
structure**, and layering histology features on top introduced noise rather than
useful information, most likely because histology intensity patterns don't map
cleanly onto the biological layer boundaries at this resolution.

This is a useful reminder that "more modalities" isn't automatically "better model" —
each added signal needs to earn its place, and the right approach is sample- and
task-dependent rather than universal.

## Repository Structure
SpaGCN-spatial-transcriptomics-clustering/
├── README.md
├── SpaGCN.ipynb            ← full analysis notebook
├── report/
│   └── SpaGCN.pdf           ← written report with all figures and discussion
└── figures/
├── clusters_s0.png
├── clusters_s1.png
└── clusters_s10.png

## Tech Stack

- **Python** — `scanpy`, `SpaGCN`, `torch`, `numpy`, `pandas`, `scipy`
- **Image processing** — `opencv-python`
- **Visualization** — `matplotlib`, `seaborn`

## Data Access

This project uses 10x Visium DLPFC data with ground-truth layer annotations. Paths in
the notebook point to a local dataset directory — update these to your own data
location before running.
