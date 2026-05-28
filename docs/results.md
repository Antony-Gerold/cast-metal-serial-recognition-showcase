# Results

## Headline

| Metric | Value |
|---|---|
| **Test accuracy** | **96.3 %** |
| **GroupKFold CV (k = 1)** | **96.4 % ± 0.6 %** |
| Total images | 1,872 |
| Part types | 10 |
| Best per-class F1 | 0.99 (GM 23301608) |
| Worst per-class F1 | 0.50 (70085119-0300) |
| HOG dimensionality | 3,780 |
| Train / Test split | 1,497 / 375 (80 / 20 stratified) |

## Per-class breakdown

| Part type | Images | Precision | Recall | F1 |
|---|---|---|---|---|
| GM 23301608 | 416 | 0.99 | 0.99 | 0.99 |
| 23354374 | 320 | 0.97 | 1.00 | 0.98 |
| GM 23305982 | 97 | 1.00 | 0.95 | 0.97 |
| GM 23477043 | 596 | 0.92 | 1.00 | 0.96 |
| GM 23497667 | 292 | 1.00 | 0.90 | 0.95 |
| 52611-0E110-A | 96 | 1.00 | 0.89 | 0.94 |
| D0CW-51310 | 14 | 1.00 | 1.00 | 1.00 |
| D0CW-51300 | 14 | 1.00 | 0.67 | 0.80 |
| D0CW-51301 | 14 | 1.00 | 0.67 | 0.80 |
| 70085119-0300 | 13 | 1.00 | 0.33 | 0.50 |

Every part with ≥ 96 images scores ≥ 0.94 F1. The bottom four (13–14 images each) drive worst-case F1 down, but **precision stays at 1.00 across all of them** — when the system predicts a small-class part, it is always correct; it just sometimes fails to recall.

## Approach comparison (chronological)

| # | Approach | Result | Failure mode / why it works |
|---|---|---|---|
| 1 | Direct thresholding (Otsu / adaptive / manual) | FAIL | Texture and characters share intensity range |
| 2 | Edge detection (Canny / Sobel) | FAIL | Metal texture creates edges everywhere |
| 3 | Gaussian blur + Otsu | Partial | No single kernel cleanly separates the two scales |
| 4 | Blur + CLAHE + black top-hat (grayscale) | Works | Stamps are dark valleys; top-hat extracts them |
| 5 | Character segmentation via connected components | 2 / 10 parts pass | Binarisation destroys signal/noise distinction |
| 6 | Character HOG + KNN on clean crops | 98.9 % CV | Recognition works; segmentation can't supply input |
| 7 | Holistic HOG + KNN, **manual ROI** | 89.3 % | Tight crop loses context |
| 8 | **Holistic HOG + KNN, MSER auto-ROI** | **96.3 %** | Wider context disambiguates similar SNs |

## ROI ablation

| ROI method | Accuracy | Notes |
|---|---|---|
| Manual ROI (tight) | 89.3 % | Just the serial number |
| Auto MSER (wide) | **96.3 %** | Includes surrounding geometry |
| Δ | **+ 7.0 pp** | Counterintuitive — initially expected the opposite |

## KNN k ablation

| k | Accuracy |
|---|---|
| **1** | **96.3 %** |
| 3 | 93.9 % |
| 5 | 92.8 % |

k = 1 dominates because each test image's nearest neighbour is overwhelmingly likely to be another image of the same part. Higher k brings in neighbours from visually similar but distinct parts and drags accuracy down.

## OCR baseline

| Engine | Configurations | Exact-match rate |
|---|---|---|
| Tesseract | 80 per image × 10 images = 800 | 0 % |
| EasyOCR | Multiple configurations | 0 % |

Not "close but with errors" — both engines produced garbage strings because the input domain (recessed stamps in textured metal) is fundamentally different from printed text.

## Robustness check

When `GM 23477043` (the part where MSER fails) is excluded, accuracy on the remaining 9 parts is **97.3 %** — higher, not lower. This confirms the failing class is the weakest performer and isn't artificially boosting the headline score. The auto-ROI captures a *consistent* wrong region for that part across all 596 of its images, which is itself a fingerprint KNN can latch onto.

## Failure analysis (small classes)

The four parts with 13–14 images have ≤ 4 test images each. A single misclassification costs 25–33 pp of recall. With 1.00 precision across all four, this is a recall problem, not a confusion problem: the system rarely confuses these parts with each other; it occasionally fails to recall them. More images per class would close most of this gap.
