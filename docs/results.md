# Results

All numbers are from the final report ([PDF](../report/MV_Project2_FinalReport.pdf)).

## Headline

| Metric | Value |
|---|---|
| **Test accuracy (auto ROI + HOG + KNN k = 1)** | **96.3%** |
| **GroupKFold cross-validation (k = 1)** | **96.4% ± 0.6%** |
| Images | 1,872 |
| Part types | 10 |
| Train / test split | 1,497 / 375 (80/20) |
| HOG feature length | 3,780 |

## KNN k

| Configuration | Accuracy |
|---|---|
| **KNN k = 1** | **96.3%** |
| KNN k = 3 | 93.9% |
| KNN k = 5 | 92.8% |
| GroupKFold CV (k = 1) | 96.4% ± 0.6% |

k = 1 works best because each image of a part is most similar to other images of the same part. Higher k risks including neighbors from visually similar but different parts. The GroupKFold result shows the accuracy does not depend on a lucky train/test split.

## Per-class results (auto ROI + HOG + KNN k = 1)

| Part | Images | Precision | Recall | F1 |
|---|---|---|---|---|
| 23354374 | 320 | 0.97 | 1.00 | 0.98 |
| 23301608 | 416 | 0.99 | 0.99 | 0.99 |
| 23305982 | 97 | 1.00 | 0.95 | 0.97 |
| 23477043 | 596 | 0.92 | 1.00 | 0.96 |
| 23497667 | 292 | 1.00 | 0.90 | 0.95 |
| 52611-0E110-A | 96 | 1.00 | 0.89 | 0.94 |
| D0CW-51300 | 14 | 1.00 | 0.67 | 0.80 |
| D0CW-51301 | 14 | 1.00 | 0.67 | 0.80 |
| D0CW-51310 | 14 | 1.00 | 1.00 | 1.00 |
| 70085119-0300 | 13 | 1.00 | 0.33 | 0.50 |

Parts with 96 or more images all reach F1 above 0.94. The weaker results come from parts with 13 to 14 images. With only 3 test images per small class, one misclassification drops accuracy by 33 percentage points. Precision is 1.00 for all small classes: when the system predicts one of these parts it is always correct, it just sometimes fails to recognize them.

## All approaches, in the order attempted

| Approach | Method | Result | Key issue |
|---|---|---|---|
| Direct thresholding | Otsu, adaptive | FAIL | Texture has the same intensity |
| Edge detection | Canny, Sobel | FAIL | Texture full of edges |
| Blur + threshold | Gaussian + Otsu | Partial | No perfect kernel size |
| Full pipeline | Blur + CLAHE + top-hat | Works | Characters isolated |
| Character segmentation | Connected components | 2/10 | Too many false blobs |
| Holistic, manual ROI | HOG + KNN k = 1 | 89.3% | Tight ROI limits context |
| Holistic, auto ROI | MSER + HOG + KNN | 96.3% | Extra context helps |

## Robustness check

To check that the 23477043 ROI failure was not inflating accuracy, the system was tested with that part excluded. Accuracy on the remaining 9 parts was 97.3%, higher than the 96.3% with all parts, so 23477043 was the weakest performer, not a source of artificial gain. It is still classified correctly most of the time because the auto ROI captures a consistent region across all 596 of its images, giving KNN a repeatable pattern based on part geometry rather than the serial number.

## Limitations

- Dataset imbalance: the four parts with fewer than 15 images account for most errors. Collecting even 50 more images per underrepresented part would likely improve their accuracy significantly.
- The system cannot identify part types it has not been trained on.
