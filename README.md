<div align="center">

# TwinSplat

### Adaptive 3DGS-SLAM-Driven Incremental Online Geometric Digital Twinning Complex Indoor Built Environments

Ye Yuan<sup>1</sup> · Long Chen<sup>1,✉</sup> · Qiuchen Lu<sup>2</sup> · Thomas Shiu Tong Ng<sup>1</sup> · Hongyang Li<sup>3</sup> · Shanjing Zhou<sup>2</sup>

<sup>1</sup> City University of Hong Kong &nbsp;·&nbsp; <sup>2</sup> University College London &nbsp;·&nbsp; <sup>3</sup> Hohai University

<sup>✉</sup> Corresponding author `longchen@cityu.edu.hk`

<br>

![Task](https://img.shields.io/badge/Task-3DGS--SLAM%20%C2%B7%20Digital%20Twin-8A2BE2)
![Journal](https://img.shields.io/badge/Journal-Advanced%20Engineering%20Informatics-1f6feb)
![Paper](https://img.shields.io/badge/Paper-Accepted-brightgreen)
![Dataset](https://img.shields.io/badge/Dataset-ReplicaCAD%20%2B%20Real%20Scene-2ea44f)
![Code](https://img.shields.io/badge/Code-Coming%20Soon-orange)

</div>

---

## Overview

Digital twins of indoor built environments must stay synchronized with the physical space as furniture is rearranged, equipment is replaced, and rooms are reconfigured between inspections. Existing 3D reconstruction pipelines either **re-scan and rebuild from scratch** (prohibitively expensive) or **naively fine-tune** the model, which triggers *catastrophic forgetting* and *visual tearing* in unchanged regions.

**TwinSplat** is an adaptive **3D Gaussian Splatting-SLAM** framework for *incremental online* geometric digital twinning. Given a pre-built baseline 3DGS model and a new RGB-D scan stream from routine inspection, it detects **only the regions that actually changed** and edits **only the Gaussian primitives responsible for those changes**, keeping the rest of the scene intact as new data streams in.

<div align="center">
<img src="assets/flowchart.png" width="85%">
<br>
<sub>Pose alignment → SSIM change detection and keyframe selection → Gaussian contribution evaluation → adaptive point-cloud editing.</sub>
</div>

---

## Main Contributions

1. **Four-Stage Adaptive 3DGS-SLAM Framework.** A unified pipeline of *visual alignment*, *SSIM-based change detection*, *parallel Gaussian contribution evaluation*, and *adaptive point-cloud editing* that prevents visual tearing and mitigates catastrophic forgetting during incremental updates.

2. **Selective Gaussian Modification.** Instead of updating the whole model, only the Gaussians tied to detected local changes are edited. This cuts average keyframe mapping time by **70%** and total per-frame time by **57%** (synthetic) or **21.5%** (real world), with **no** loss in reconstruction quality.

3. **Incremental Updating with Geometric Preservation.** Streaming data is integrated continuously while unchanged geometry is protected, and about **70%** of the final Gaussian primitives are preserved from the baseline model, keeping the scene complete even under partial scan coverage.

---

## How It Works

**Compare first, then edit only what changed.** Each incoming frame is rendered from the current model at the tracked pose and compared with the new capture. A sliding-window **SSIM** detector masks the pixels that differ, traces them back to the Gaussians responsible, and re-optimizes only those while leaving everything else frozen. That selectivity is what keeps each update cheap and free of catastrophic forgetting.

<div align="center">
<table>
<tr>
<td align="center"><b>① Baseline render</b></td>
<td align="center"><b>② Change detected</b></td>
<td align="center"><b>③ Updated model</b></td>
</tr>
<tr>
<td><img src="assets/step1_baseline.png" width="250"></td>
<td><img src="assets/step2_mask.png" width="250"></td>
<td><img src="assets/step3_updated.png" width="250"></td>
</tr>
</table>
<sub>The baseline still renders the moved plant (①). SSIM flags that region (②, 6.45%). Editing only those Gaussians matches the new capture (③, SSIM 0.881).</sub>
</div>

<br>

Contribution scoring and editing happen inside the Gaussian rasterizer, reusing the same CUDA backward pass as standard 3DGS training, so the added cost stays small.

<div align="center">
<img src="assets/cuda_gaussian.png" width="80%">
<br>
<sub>The SSIM difference mask drives a per-Gaussian contribution score that selects which primitives to edit, all inside CUDA.</sub>
</div>

### Updating across a sequence

Across a long capture, only the frames that actually change trigger an edit, so the moved plant, shelf, and bicycle are integrated region by region while the rest of the scene stays intact.

<div align="center">
<img src="assets/process_frames_1-130.png" width="90%">
<br>
<sub>Frames 1–130. Rows from top: baseline render, change mask, updated model, ground truth.</sub>
</div>

---

## Key Results

### Handling changes without forgetting

When new scans cover only part of a room, naively updating with MonoGS corrupts previously reconstructed areas. TwinSplat updates the changed furniture while preserving the static background.

<div align="center">
<table>
<tr>
<td align="center"><b>TwinSplat (Ours)</b></td>
<td align="center"><b>Update with MonoGS</b></td>
</tr>
<tr>
<td><img src="assets/rpd_upd_render_frame_0070.png" width="360"></td>
<td><img src="assets/monogs_render_frame_0070.png" width="360"></td>
</tr>
<tr>
<td align="center"><sub>Static regions preserved, change integrated</sub></td>
<td align="center"><sub>Catastrophic forgetting and artifacts</sub></td>
</tr>
</table>
</div>

### Novel-view rendering on a real scene

In a real laboratory scan, rendering the baseline model from a novel viewpoint exposes rendering voids where geometry is incomplete. The adaptive update fills these regions and renders cleanly.

<div align="center">
<table>
<tr>
<td align="center"><b>TwinSplat (Ours)</b></td>
<td align="center"><b>Baseline (no update)</b></td>
</tr>
<tr>
<td><img src="assets/Rpd_update_GS.jpg" width="360"></td>
<td><img src="assets/no_update_GS.jpg" width="360"></td>
</tr>
</table>
</div>

### Quantitative comparison on ReplicaCAD test views

| Method | Test SSIM ↑ | Test PSNR ↑ | ATE RMSE [m] ↓ | Avg. Time [s] ↓ |
|:--|:--:|:--:|:--:|:--:|
| **TwinSplat (Ours)** | **0.7542** | **19.48** | 0.1054 | 1.162 |
| Update w/ MonoGS | 0.6537 | 13.98 | 0.1109 | 2.730 |
| Rebuild w/ SplaTAM | 0.6834 | 13.39 | **0.0086** | 5.988 |
| Rebuild w/ Photo-SLAM | 0.7339 | 15.14 | 0.0400 | 0.040 * |
| Rebuild w/ GS-ICP-SLAM | 0.5810 | 9.04 | 1.0764 | 0.085 * |

<sub>\* C++ or hybrid C++/Python multi-process implementations; all Python baselines run single-thread for a fair comparison.</sub>

- Against the **MonoGS baseline**, TwinSplat improves test-view SSIM by **15%** and lowers average processing time by **57%** on ReplicaCAD.
- In the **real-world case study**, it reaches SSIM **0.84** and PSNR **23.37 dB**, a **5%** SSIM gain and **21.5%** less per-frame time than the classic MonoGS updating pipeline, with **zero** catastrophic-forgetting or visual-tearing failures across all test frames.

---

## Source Code

> 🚧 **Source code will be released here soon.** We are cleaning up and documenting the implementation for public release. Star ⭐ / watch 👀 this repository to be notified.

Planned release.
- [ ] Core adaptive 3DGS-SLAM pipeline (tracking, change detection, adaptive editing)
- [ ] Pretrained baseline models and example scans
- [ ] Evaluation scripts (SSIM, PSNR, ATE) and configs
- [ ] Setup instructions and dependencies

---

## Citation

If you find this work useful, please consider citing it. Volume and page details will be added once the article is in press.

```bibtex
@article{yuan2026twinsplat,
  title   = {Adaptive 3DGS-SLAM-Driven Incremental Online Geometric Digital
             Twinning Complex Indoor Built Environments},
  author  = {Yuan, Ye and Chen, Long and Lu, Qiuchen and Ng, Thomas Shiu Tong
             and Li, Hongyang and Zhou, Shanjing},
  journal = {Advanced Engineering Informatics},
  year    = {2026},
  note    = {Accepted}
}
```

---

## Acknowledgements

This work builds upon the open-source community, in particular
[MonoGS](https://github.com/muskie82/MonoGS),
[3D Gaussian Splatting](https://github.com/graphdeco-inria/gaussian-splatting),
[SplaTAM](https://github.com/spla-tam/SplaTAM),
[Photo-SLAM](https://github.com/HuajianUP/Photo-SLAM), and
[GS-ICP-SLAM](https://github.com/Lab-of-AI-and-Robotics/GS_ICP_SLAM).
We thank the authors for releasing their code.
