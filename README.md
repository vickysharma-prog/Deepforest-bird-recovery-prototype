# Recovering Bird Annotations from Historical Airborne Imagery
Prototype for recovering ML-ready annotations from 18,304 historical bird survey screenshots where colored dots were baked directly into images by a point-counting tool. No coordinate data was saved. This prototype extracts dot positions, maps them to original high-resolution photographs, and produces DeepForest-compatible training data.

Data source: twi-aviandata.s3.amazonaws.com (Gulf of Mexico avian monitoring, 2010-2021, post-Deepwater Horizon)

---

## About This Work
This prototype was built over 1.5 months under the mentorship of my uncle and guidance from past GSoC contributors who guided me through the open-source contribution process and research. Before writing any pipeline code, I spent two weeks studying the data — mapping 533K files in the S3 bucket, analyzing 49,204 CSV rows across 60 columns, downloading and measuring 25 diverse images, and testing existing models (SAM 3, DeepForest, GroundingDINO) to understand what was feasible.
The pipeline went through five detector versions, four training experiments, and six approaches that were tested and abandoned. Every failure is documented with root cause analysis. 
The biggest lesson: study the data before building. 40% of my time was measurement and analysis. Every measured parameter directly informed a pipeline design decision. Detection accuracy jumped from 44% (first version, guessed parameters) to 70.8% (final version, measured parameters) without changing the fundamental approach.

---
## Pipeline Output

### The Problem: Annotations Baked Into Screenshots
![Study Images](results/cell1_study_images.png)

Four study images spanning the difficulty range. Left: screenshots with colored dots baked in by the counting tool. Right: original high-resolution photographs (no annotations). The pipeline recovers dot positions from the left and maps them to coordinates on the right.

### Dataset Analysis (49,204 Rows)
![CSV Analysis](results/cell2_forensic_analysis.png)

LAGU (Laughing Gull) dominates at 855K birds. Median image has 61 birds. 18 annotators across 7 years and 5 Gulf Coast states. Every number here informed a pipeline parameter.

### Measured Dot Properties
![Dot Properties](results/cell3_dot_properties.png)

Dot diameter 8.0px median, 6 distinct annotation colors, circularity 0.57. Measured from 1,199 candidate dots across all study images. These measurements set the detection thresholds — nothing was guessed.

### Dots at 8x Zoom
![Zoomed Dots](results/cell3_zoomed_dots.png)

Individual annotation dots at 8x magnification showing actual pixel-level rendering. Red, yellow, green, cyan, blue, and magenta dots are clearly distinct from background at this scale.

### Screenshot Decomposition
![Decomposition](results/cell4_decomposition.png)

3-method consensus boundary detection (grey profile + Sobel edges + color variance) reliably separates the aerial photograph from the dialog box across all screenshot formats.

### Detection Results (4 Study Images)
![Detection](results/cell6_detections_v2.png)

Detection on 4 study images (deliberately hard cases): D: 62%, B: 69%, A: 51%, C: 48%. Left: detected dots overlaid on aerial. Right: expected vs available vs selected per species. Batch of 30 random images achieves 70.8% overall.

### Validation: Are These Real Dots?
![Validation Grid](results/cell7_validation_grid.png)

50 detected dots at 6x zoom from Image D. Top rows (high score): clearly real annotation dots with bright red centers. Bottom rows (low score): includes text watermark false positives ("17", "7Ma", etc.) — this is exactly why the text filter was disabled. Birds sitting in colony rows look like text to the algorithm.

### Spatial Distribution
![Spatial](results/cell7_spatial.png)

Detected dots spread across aerial images, not clustered in one area. Coverage ranges from 52-92% of the 5x5 spatial grid per image.

### Coordinate Mapping (Screenshot to Original)
![Mapping](results/cell8_mapping.png)

Left: dots on screenshot. Right: dots mapped to original high-resolution image. D and C use SIFT homography (0.5px accuracy). B and A use uniform height-based scaling (fallback when SIFT fails).

### Mapped Dots Verification (Zoomed)
![Zoom Verify](results/cell8_zoom_verify.png)

Top 10 mapped dots on original image at high zoom. Red crosshairs mark the mapped center. Bounding boxes show the annotation region. Birds visible at mapped locations.

### Exported Dataset
![Dataset Overview](results/cell9_dataset_overview.png)

3,915 recovered annotations across 21 species in DeepForest CSV format. Species distribution, score distribution, per-image counts, and mapping method breakdown.

### Training Experiment: Full Circle
![Full Circle](results/cell12_full_circle.png)

The complete pipeline story on one image: (1) corrupted screenshot with baked-in dots, (2) 64/93 dots recovered at 69% accuracy, (3) pretrained DeepForest produces 2 high-confidence detections, (4) fine-tuned model — this panel shows the bfloat16-corrupted run (300 false detections at score 1.0). Root cause identified and fixed in subsequent experiments. See [training analysis](docs/training_analysis.md).

### Training Experiment: SIFT-Only (Root Cause Fix)
![SIFT Only](results/cell12d_sift_only.png)

Training only on SIFT-mapped images (0.5px position accuracy, 920 annotations) produced the only improvement over pretrained: +29% max score, 0 to 1 high-confidence detection. This confirmed that position accuracy — not data quantity — drives training quality. With 76% fewer annotations but accurate positions, the model improved. With the full dataset (including ~30px position errors), it got worse.

---
## What this prototype does
I spent about a month building this. The first two weeks were pure study — mapping the S3 bucket (533K files), analyzing the CSV (49,204 rows, 60 columns), downloading 25 diverse images, measuring dot sizes and colors, testing SAM 3 and DeepForest, and figuring out what was actually feasible before writing any pipeline code.
The pipeline itself went through multiple iterations. Five versions of the detector, four training experiments, six approaches that failed and got thrown out. Everything is documented.
### The pipeline

Screenshot → Decompose → Detect Dots → Validate → Map to Original → Export

1. Split screenshot into aerial photograph and dialog box (3-method consensus boundary detection)
2. Detect colored annotation dots using wide HSV bins with vegetation-adaptive thresholds
3. Match detected color groups to CSV species by count similarity
4. Validate detected positions (98.3% land on high-saturation pixels)
5. Map dot coordinates from screenshot to original using SIFT homography or uniform scaling
6. Export as DeepForest CSV (image_path, xmin, ymin, xmax, ymax, label)

## Results

| Metric | Value |
|--------|-------|
| Images processed | 34 (0 crashes) |
| Detection accuracy | 70.8% |
| Precision | 98.3% |
| False positive rate | 1.7% |
| Annotations recovered | 3,915 |
| Species detected | 21 |
| SIFT mapping success | 57% (0.5px error) |
| SAM 3 habitat classification | 97% |
| Scene captions generated | 34 |
| Processing speed | 11 sec/image |
| Estimated full dataset recovery | ~340,000 annotations |

## What didn't work (and why)

| Approach | Result | Root Cause |
|----------|--------|------------|
| OCR on dialog box | 4-15% precision | Fuzzy matching hits random English words |
| Dialog color clusters | 8% accuracy | Position-based assumption was wrong |
| Narrow HSV bins | 44% accuracy | Real dots have more color variation than expected |
| Text watermark filter | Removed real dots | Birds in rows look like text characters |
| Training on full dataset | 0 high-conf detections | 43% of data has ~30px position error |
| Species-aware boxes | Made it worse | Can't fix positions with better boxes |

### What partially worked

SIFT-only training (using only images with 0.5px-accurate coordinate mapping) produced a +29% improvement in max detection score over pretrained DeepForest. One high-confidence detection appeared. Direction is correct but the model isn't useful yet — needs more SIFT-mapped data or SAM 3 box refinement.

---

## Numbers

**Dataset**: 49,204 CSV rows, 18,304 unique screenshots, 102 species, 18 annotators, 442 colonies, 5 states (LA 71%, TX 15%, AL 8%, FL 6%, MS 1%), years 2010-2021.

**Dot properties** (measured from real data): diameter 5.7-9.6px (median 8.0), area 26-72 px², circularity 0.574 median, anti-aliased 2%, green-on-green contrast 10.4 on worst image.

**Detection on 4 study images** (selected for difficulty — not representative):
- D (Raccoon Island, 538 birds): 62.5%
- B (Gaillard Island, 91 birds): 68.8%
- A (Felicity Island, 1388 birds): 51.3%
- C (North Deer Island, 96 birds, green-on-green): 47.8%

**Detection on 30 batch images** (stratified random — true performance):
- Overall: 70.8%, median 74.2%, range 9-100%
- 3 images at 100%, 11 above 85%, 3 below 30%
- Works across all 7 years and 10 annotators tested

**SAM 3**: 2,387 bird detections total, 593 dots confirmed (15%), 2,012 COPPER (birds SAM found that our pipeline missed). Habitat: marsh grass 45%, vegetation 39%, sandy beach 9%, bare ground 6%.

**Training root cause**: uniform coordinate mapping introduces ~30px position error → training boxes don't align with actual birds → IoU below threshold → model ignores those samples → underfits. SIFT mapping (0.5px error) fixes this.

---

## Repository structure
notebook/prototype_v1.ipynb — Full prototype, 23 sections, runs in Colab (~45 min on T4 GPU)
results/figures/ — Output figures (populated after notebook run)
results/metrics/ — JSON metrics from each pipeline stage
results/annotations/ — Recovered annotations in DeepForest CSV format
docs/learnings.md — 21 learnings from failed and successful approaches
docs/training_analysis.md — Detailed analysis of 4 training experiments

## How to run

Open `notebook/prototype_v1.ipynb` in Google Colab. Set runtime to T4 GPU. Run all cells top to bottom. Takes about 45 minutes. Cells 1-14 run on CPU (data download + pipeline). Cells 15-17 need GPU (SAM 3 + DeepForest training).

## What I learned

The biggest lesson: study the data before building. I spent 40% of my time just measuring things — dot sizes, color distributions, screenshot formats, vegetation percentages, annotator patterns. Every measurement directly informed a pipeline parameter. The detection accuracy jumped from 44% (first version, guessed parameters) to 70.8% (final version, measured parameters) without changing the fundamental approach.

The second lesson: be honest about what doesn't work. OCR looked promising in theory but failed at 4% precision. Training on the full dataset produced zero useful detections. Documenting these failures and understanding why they failed (not just that they failed) turned out to be more valuable than the successes — it points directly to what needs to happen next.

Full list of learnings: [docs/learnings.md](docs/learnings.md)

Training experiment details: [docs/training_analysis.md](docs/training_analysis.md)
