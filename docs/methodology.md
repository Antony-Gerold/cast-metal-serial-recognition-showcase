# Methodology

## 1. Dataset

| Property | Value |
|---|---|
| Total images used | 1,872 (of 1,885 available) |
| Part types | 10 (of 12 — two excluded for too few images) |
| Resolution | 1288 × 964, grayscale |
| Class balance | Heavily imbalanced — 13 to 596 images per class |
| Excluded | `20871905`, `BK21-64842-A-B` (13 images total) |

## 2. Region-of-Interest detection

### 2.1 Manual ROI baseline

For each part type, the serial-number bounding box is hand-annotated on one reference image and reused for all images of that part. Trivial to implement; requires human setup per new part type.

### 2.2 Automatic ROI via MSER

[Matas et al., 2004] Maximally Stable Extremal Regions detects connected pixel sets that stay stable across a range of intensity thresholds. Three-stage pipeline:

1. **Candidates.** Run MSER on both the original ROI and its inverse to capture both dark-on-light and light-on-dark characters. Filter by size and aspect ratio to keep character-shaped blobs only.
2. **Cluster.** Build a 2-D density grid over candidate centroids; the densest cluster is assumed to enclose the serial number.
3. **Box.** Fit a padded bounding box around the cluster.

**Results across the 10 parts:** average recall **0.90**, average IoU **0.13** against the manual ROI. The auto box is consistently 3–10× larger than the true serial-number region. One part (`GM 23477043`) fails completely — MSER candidates cluster in the wrong area.

This was initially flagged as a problem. It later proved not to be — see Section 5.

## 3. Preprocessing evolution

### Attempt 1 — Direct thresholding (FAIL)

Otsu, adaptive, and manual fixed-threshold binarisation. All three fail because texture bumps occupy the same intensity range as character pixels — thresholding cannot separate signal from noise when both are at the same value.

### Attempt 2 — Edge detection (FAIL)

Canny and Sobel on the raw ROI. Texture creates edges everywhere; character edges have no distinguishing magnitude. Drowned out.

### Attempt 3 — Gaussian blur before thresholding (PARTIAL)

Texture is high-frequency; characters are mid-frequency. Blur should suppress texture preferentially. Tested kernels 3 → 31. Small kernels leave too much texture; large kernels erode characters. No single kernel size cleanly separates the two scales. Blur helps but is insufficient on its own.

### Attempt 4 — Black top-hat (BREAKTHROUGH)

The physical insight: stamps are recessed valleys created by a die pressed into soft metal. Recessed valleys reflect less light → they are dark depressions in the surface. The black top-hat transform extracts exactly that class of feature:

$$T_{black}(f) = (f \bullet b) - f$$

where $f$ is the input image and $b$ is a flat structuring element. The morphological closing $f \bullet b$ slides $b$ over the surface and fills the valleys; subtracting the original leaves only what was filled.

**Structuring element:** 25 × 25 rectangle. Sized larger than character strokes (5–20 px) but smaller than the largest texture artefacts. Combined with CLAHE (`clipLimit = 3.0`, `tileGridSize = 8 × 8`) for local contrast normalisation under uneven lighting.

### Final preprocessing chain

```
Gaussian blur (per-part k = 9..25) → CLAHE → Black top-hat (25×25)
```

Two versions exist downstream:

- **Binarised version** (`+ Otsu + morphological opening`): used for the character-segmentation attempt
- **Grayscale version**: used for the final holistic classifier

## 4. Character-level approach (FAIL)

Goal: connected-component analysis on the binarised pipeline output → labelled character crops → HOG + KNN classifier.

**Component filters:** height > 12 % of ROI height, width < 18 % of ROI width, area > 80 px, aspect ratio < 2.0.

**Reference-image test:** only 2 of 10 parts produced a blob count within ±2 of the expected character count. The other 8 produced too many blobs from residual texture noise.

**Extracted crops:** 11 labelled 28 × 28 character samples across 7 classes (the parts where segmentation worked). Most crops are visually corrupted — texture noise survives binarisation and contaminates the character shape.

**Diagnosis:** the classifier is not the bottleneck (HOG + KNN reaches 98.9 % CV on clean character data with augmentation). The bottleneck is producing clean crops. **Binarisation is the culprit:** in the grayscale top-hat output, characters are bright and texture is dim. After Otsu, both become white pixels, and connected components can no longer distinguish them.

## 5. Holistic classification (FINAL)

Insight: each of the 10 part types has a *unique* serial number, so its serial-number region has a unique visual fingerprint regardless of whether individual characters are isolable.

**Feature extraction:** HOG [Dalal & Triggs, 2005] with 9 orientations, 8 × 8 pixel cells, 2 × 2 cell blocks, on the 128 × 64 resized **grayscale top-hat** output (no binarisation). 3,780-dimensional feature vector per image.

The critical choice is feeding HOG the grayscale top-hat rather than the binarised version. Grayscale preserves the intensity gap between strong character edges and weak texture edges — exactly what HOG is designed to integrate over.

**Classifier:** k-nearest neighbours, k = 1, Euclidean distance. k = 1 outperforms k = 3 and k = 5 because every test image's nearest neighbour is overwhelmingly likely to be another image of the same part.

**ROI choice — counterintuitive result:**

| ROI | Test accuracy |
|---|---|
| Manual (tight) | 89.3 % |
| Auto MSER (wide) | **96.3 %** |

The wider MSER region captures surrounding casting geometry (screw holes, edges, part contour). For the D0CW parts whose serial numbers differ by one digit, this extra signal is what disambiguates them. The system identifies the part not purely by the serial number but by the *combination* of serial-number region and surrounding context.

## 6. Evaluation

| Protocol | Accuracy | Note |
|---|---|---|
| 80/20 stratified split (k = 1) | **96.3 %** | Headline test number |
| GroupKFold cross-validation (k = 1) | **96.4 % ± 0.6 %** | No image leakage; fairest protocol |
| KNN k = 3 | 93.9 % | |
| KNN k = 5 | 92.8 % | |
| Accuracy excluding the failing class | 97.3 % | Confirms 23477043 is the weakest, not propping up the overall score |

The small gap between stratified and GroupKFold accuracy indicates the model is genuinely learning part-level visual patterns rather than memorising per-image artefacts.

## 7. Report-style notes (why this graded 9.7/10)

ENGG\*6100 (Prof. Medhat Moussa) rewards:

- Chronological *tried → failed → learned → pivoted* narrative
- Concrete numbers at every step
- Comparison tables across approaches (here: Table 5 in the report)
- Physical reasoning for design choices (top-hat motivated by stamp physics)
- External references beyond course materials
- Figures showing each processing stage

The 0.3-point deduction was on Discussion depth — limitations could have been pushed harder, with more external literature cited. This calibration informed subsequent projects.
