# Multi-Template Fastener Detector

A Python pipeline for **automatically detecting and locating fastener symbols** (bolts, screws, rivets, etc.) in engineering drawings using scale-invariant template matching, morphological image augmentation, and DBSCAN clustering.

---

## Overview

Given a grayscale engineering drawing and a folder of fastener symbol templates, this tool finds every occurrence of each symbol in the drawing — even when symbols appear at different scales. Rather than relying on a single template match, it generates augmented crops of each template, runs template matching across multiple scales and image variants, clusters the raw detections with DBSCAN, and refines them through a multi-stage statistical filtering pipeline before outputting final bounding boxes and center points.

---

## Repository Structure

```
Multi-Template-Matcher/
│
├── Multi_Template_Matcher.py       # Entry point — runs the full pipeline for all templates
├── Processor.py                    # Core detection pipeline for a single template
├── Cleanup.py                      # Cross-template deduplication using DBSCAN
│
└── Utils/
    ├── Hyperparameters.py          # All tunable parameters and directory paths
    ├── Template_Matching.py        # Scale-invariant template matching + Display function
    ├── Morphological_Operations.py # Binarization and morphological preprocessing
    └── Random_Image_Crop.py        # Random crop augmentation of templates
```

### File Descriptions

| File | Purpose |
|------|---------|
| `Multi_Template_Matcher.py` | Iterates over all templates, runs `Processor` on each, runs `Cleanup`, then calls `Display` |
| `Processor.py` | Full single-template pipeline: augment → match → cluster → refine → return mean points |
| `Cleanup.py` | Removes duplicate detections across different template types using DBSCAN |
| `Template_Matching.py` | `invariantMatchTemplate` — matches at every scale in range; `Display` — plots results |
| `Morphological_Operations.py` | Otsu binarization + erosion/opening/closing/dilation to produce sharp and eroded drawing variants |
| `Random_Image_Crop.py` | Generates N random crops of the template, split into sharp/eroded/normal subsets |
| `Hyperparameters.py` | Single source of truth for all parameters, thresholds, and file paths |

---

## Requirements

- Python 3.8+
- Dependencies:

```bash
pip install opencv-python numpy scikit-learn scipy tqdm torchvision Pillow matplotlib
```

| Package | Purpose |
|---------|---------|
| `opencv-python` | Image I/O, binarization, morphological ops, template matching |
| `numpy` | Array operations |
| `scikit-learn` | DBSCAN clustering, z-score, train/test split |
| `scipy` | Z-score for scale outlier removal |
| `tqdm` | Progress bars during template matching |
| `torchvision` | `RandomCrop` transform for template augmentation |
| `Pillow` | PIL image conversion for torchvision |
| `matplotlib` | Result visualization and saving |

---

## How to Run

1. Place your engineering drawing at `Data/Input/Drawings/drawing_6.png`
2. Place your template images (one per fastener type) in `Data/Input/Templates/Template_Set_1/`
3. Run:

```bash
python Multi_Template_Matcher.py
```

The script will process every template in the folder, clean up overlapping detections, and display/save results to `Data/Output/`.

---

## Pipeline

### 1. Preprocessing (`Morphological_Operations.py`)
Each drawing and template is binarized using **Otsu thresholding**, then three variants of the drawing are produced — normal, sharp (dilated), and eroded — to improve detection robustness across line weight variations.

### 2. Template Augmentation (`Random_Image_Crop.py`)
`N = 90` random crops of the preprocessed template (at `55%` of original size) are generated. These are split into three subsets — sharp, eroded, and normal — matched against their respective drawing variants.

### 3. Scale-Invariant Template Matching (`Template_Matching.py`)
`invariantMatchTemplate` sweeps scales from `30%` to `150%` at `1%` intervals, running OpenCV template matching (`TM_CCORR_NORMED` by default) at each scale. Only matches above the `matched_thresh = 0.983` threshold are kept, and redundant nearby detections are removed.

### 4. DBSCAN Clustering + Refinement (`Processor.py`)
Raw detections are clustered spatially with DBSCAN (`eps=35`, `min_samples=5`). Clusters then go through three refinement stages:
- **Accuracy check** — drops clusters whose best match score falls more than `0.01` below the global best
- **Scale variance check** — removes points with z-score ≥ `1.5` within a cluster, then drops clusters with scale std dev ≥ `9`
- **Mean scale check** — drops clusters whose mean scale deviates more than `4` units from the highest-confidence detection's scale

The surviving clusters are reduced to a single mean point each, carrying: `[x, y, sample_count, scale, cross_corr_coeff]`.

### 5. Cross-Template Cleanup (`Cleanup.py`)
After all templates are processed, `Cleanup` runs a second DBSCAN pass (`eps=20`, `min_samples=2`) across all detections from all templates combined, retaining only the highest-sample-count detection per cluster — removing cases where multiple templates matched the same symbol.

### 6. Display & Save (`Template_Matching.py → Display`)
Results are plotted on the drawing with per-template colors, bounding boxes, and center points, and saved to `Data/Output/final_boxes.png` and `Data/Output/final_points.png`.

---

## Configuration (`Hyperparameters.py`)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `drawing_dir` | `Data/Input/Drawings/drawing_6.png` | Path to input drawing |
| `main_template_dir` | `Data/Input/Templates/Template_Set_1/` | Folder of template images |
| `method` | `TM_CCORR_NORMED` | OpenCV template matching method |
| `matched_thresh` | `0.983` | Minimum match score threshold |
| `scale_range` | `[30, 150]` | Scale sweep range (%) |
| `num_crop_images` | `90` | Number of augmented template crops |
| `crop_scale` | `0.55` | Crop size as fraction of template |
| `eps_main` | `35` | DBSCAN epsilon for detection clustering |
| `eps_cleanup` | `20` | DBSCAN epsilon for cross-template cleanup |
| `accuracy_thres` | `0.01` | Max allowed drop from best match score |
| `zscore_thres` | `1.5` | Z-score cutoff for scale outlier removal |
| `std_dev_thres` | `9` | Max allowed scale std dev within cluster |
| `mean_scale_thres` | `4` | Max allowed mean scale deviation |

---

## Output

All outputs are saved to `Data/Output/`:

| File | Description |
|------|-------------|
| `final_boxes.png` | Drawing with bounding boxes around each detected fastener |
| `final_points.png` | Drawing with center points of each detection, color-coded by type |
| `Data/Input/Augmented/` | Saved augmented template crops (for inspection) |

---

## Supported Template Matching Methods

`TM_CCORR_NORMED`, `TM_CCOEFF`, `TM_CCOEFF_NORMED`, `TM_CCORR`, `TM_SQDIFF`, `TM_SQDIFF_NORMED` — configurable in `Hyperparameters.py`.

---

## References

- OpenCV Template Matching: https://docs.opencv.org/3.4/d4/d86/group__imgproc__filter.html
- Ester et al. (1996) — DBSCAN: *A Density-Based Algorithm for Discovering Clusters*
