# Recovering Bird Annotations from Historical Airborne Imagery
Prototype for recovering ML-ready annotations from 18,304 historical bird survey screenshots where colored dots were baked directly into images by a point-counting tool. No coordinate data was saved. This prototype extracts dot positions, maps them to original high-resolution photographs, and produces DeepForest-compatible training data.

Data source: twi-aviandata.s3.amazonaws.com (Gulf of Mexico avian monitoring, 2010-2021, post-Deepwater Horizon)

---

## Pipeline Output Samples

### Study Images (Screenshot vs Original)
![Study Images](results/cell1_study_images.png)

### Data Analysis (49,204 rows)
![CSV Analysis](results/cell2_forensic_analysis.png)

### Dot Properties (Measured from Real Data)
![Dot Properties](results/cell3_dot_properties.png)

### Dot Detection Results (4 Study Images)
![Detection](results/cell6_detections_v2.png)

### Validation — 98.3% of Dots Are Real
![Validation](results/cell7_validation_grid.png)

### Coordinate Mapping (Screenshot → Original)
![Mapping](results/cell8_mapping.png)

### Full Circle: Corrupted → Recovered → Detector
![Full Circle](results/cell12_full_circle.png)

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

### What actually worked

| What | Result |
|------|--------|
| Processing 34 images end-to-end | 0 crashes |
| Count accuracy (detected / expected) | 70.8% overall |
| Position precision (saturation check) | 98.3% of dots are real |
| False positive rate | 1.7% |
| SIFT coordinate mapping | 57% of images, 0.5 pixel error |
| SAM 3 habitat classification | 97% of images classified |
| Total annotations recovered | 3,915 across 21 species |

### What didn't work (and why)

| Approach | Result | Why it failed |
|----------|--------|---------------|
| OCR on dialog box | 4-15% precision | Fuzzy matching hits random English words |
| Dialog color clusters → species | 8% accuracy | Position-based matching assumption was wrong |
| Narrow HSV color bins | 44% accuracy | Too restrictive for real-world color variation |
| Text watermark filter | Removed real dots | Birds in colony rows look like text to the algorithm |
| Training on full dataset | 0 high-conf detections | 43% of data has ~30px position error from uniform mapping |
| Species-aware box sizes | Made it worse | Can't fix wrong positions with better boxes |

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
