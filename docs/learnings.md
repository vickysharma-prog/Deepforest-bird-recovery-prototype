# What I Learned Building This Prototype

21 things I discovered, got wrong, or figured out the hard way over about a month of work. Each one changed how the pipeline works.

---

## Getting the data right

**1. total_birds is per-row, not per-image.** The CSV has one row per species per image. An image with 538 birds and 7 species has 7 rows. I was picking study images by looking at individual row values instead of summing — got completely wrong images the first time. Fixed by grouping by screenshot path and summing.

**2. DottingAreaNumber is just a counter.** I initially thought multiple screenshots might cover different sub-areas of one original image, which would mean I'd need to stitch them together. Turns out it's just a sequential ID (0, 1, 2...). Each original has exactly one screenshot. Saved me from building an unnecessary merging module.

**3. LAGU dominates, not BRPE.** Laughing Gull is 32.4% of all birds. Brown Pelican gets more press but is only 14.7% in this dataset. Changed which species I optimized for.

**4. Most images are easy.** Median is 61 birds per image. 65% have fewer than 100. Only 2% have over 1000. My four study images (96-1388 birds, 7-13 species) were deliberately hard — not representative of what the pipeline would typically face.

---

## Things I tried that didn't work

**5. OCR on the dialog box.** Spent a day on this. Tesseract with upscaling and adaptive thresholding, fuzzy matching species codes with Levenshtein distance ≤ 2. Problem: "THE" matches "TRHE" (Tricolored Heron), "AND" matches "CANG" (Canada Goose), any 3-4 letter word matches something. 4% precision on the best image, 4% on the worst. The CSV has the same information with 100% accuracy, so I stopped.

**6. Color analysis from the dialog.** Tried extracting colored regions from the dialog box and matching them to species by vertical position. Assumed colors would appear in the same order as species in the CSV. They don't. 1 out of 12 mappings was correct (8%). Switched to count-based matching instead — match the largest detected color group to the most common species, second-largest to second-most-common, and so on. Not perfect but functional.

**7. Text watermark filter.** Some screenshots have colony names printed in color on the aerial photograph. I built a filter to detect and remove horizontally-aligned, regularly-spaced colored dots (text characters). Problem: birds in colonies also sit in rows at similar y-coordinates. The filter removed actual bird annotations. Disabled it entirely. The aspect ratio filter (skip anything wider than 3:1) catches actual text lines without touching bird colonies.

**8. Narrow color bins.** Started with tight HSV ranges (mean ± 5 hue units per color). Got 44% detection accuracy. Widened to cover entire color regions (e.g., green = H 36-82 instead of H 55-65). Jumped to 58%. Annotation dots have more color variation than I expected — fading, JPEG compression, background influence.

**9. Adaptive threshold tuning.** Built a system that starts narrow and widens if detected count is too low compared to CSV expected count. Same result as just using wide bins from the start. The iteration added complexity without benefit.

---

## Technical things that matter

**10. Red hue wraps around.** In OpenCV's HSV, hue goes 0-180. Red sits at both ends: 0-10 and 170-180. Taking the arithmetic mean of [5, 175] gives 90 (cyan — completely wrong). Had to implement circular mean using sin/cos components. Shows up as a 5-line function but took a while to debug.

**11. Count-based color-to-species matching.** The pipeline detects color groups (how many red dots, how many blue dots, etc.) and matches them to CSV species by count similarity. It's a greedy assignment: biggest detected group → most common species. Sometimes it gets the color name wrong (calls cyan "blue") but the species label is usually right because the matching is based on counts, not color names.

**12. Study image accuracy ≠ pipeline accuracy.** The four study images average 52.3% accuracy. The batch of 30 random images averages 70.8%. I almost reported 52.3% as the pipeline's performance — that would have been misleading. The study images were picked to be hard.

**13. Uniform scale_x is wrong for coordinate mapping.** When SIFT fails, I fall back to uniform scaling. Initially used scale_x = original_width / aerial_width. But the aerial region in the screenshot shows a subregion of the original — it doesn't span the full width. Scale_x gave 23-39% horizontal stretch. The fix: use scale_y (original_height / aerial_height) for both axes. Heights match because the aerial view isn't cropped vertically.

**14. The pipeline works across all years and annotators.** I was worried that the annotation software might have changed between 2010 and 2021, or that different annotators might use different dot styles. Tested on 7 years and 10 annotators. No systematic failures. The format is consistent.

---

## Training was harder than expected

**15. bfloat16 from SAM 3 breaks DeepForest.** SAM 3 enables a global bfloat16 autocast. Even after deleting the SAM 3 model and freeing GPU memory, the autocast stays active. DeepForest training under bfloat16 produces all scores = 1.0 (garbage). Took a while to figure out — the training "succeeds" with no errors, but the model is useless. Fix: explicitly disable autocast before training.

**16. 80×80 boxes cause IoU problems.** My training annotations use fixed 80×80 pixel bounding boxes. The model learns to predict boxes closer to actual bird size (~106×105). During training, the IoU between an 80×80 ground truth box and a 106×105 predicted box is low enough that the model doesn't get credit for correct detections.

**17. Better boxes don't help if positions are wrong.** After discovering the box size issue, I switched to species-aware sizes (BRPE=110×100, LAGU=60×55). Results got worse, not better. The real problem: 43% of my training data uses uniform coordinate mapping with ~30px position error. A perfectly-sized box centered 30 pixels away from the actual bird still has low IoU.

**18. Position accuracy is everything.** Training only on SIFT-mapped images (0.5px accuracy, 920 annotations) improved the model by 29% over pretrained. Training on the full dataset (3,851 annotations including ~30px-error data) made the model worse than pretrained. Less data with accurate positions beats more data with inaccurate positions.

**19. The model isn't working yet, but the direction is clear.** SIFT-only training produced one high-confidence detection and a 29% improvement in max score. That's not a working detector. But it proves the recovered annotations can teach a model — the limitation is position accuracy, not the recovery approach itself. SAM 3 box refinement or more SIFT-mapped images should close the gap.
