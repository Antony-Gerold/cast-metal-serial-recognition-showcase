# Methodology

Everything here follows the final report ([PDF](../report/MV_Project2_FinalReport.pdf)), in the order the problem was worked through.

## 1. Dataset

| Property | Value |
|---|---|
| Full dataset | 1,885 grayscale images across 12 part types |
| Used in this project | 1,872 images across 10 part types |
| Class balance | Six parts have 96 to 596 images; four parts have only 13 or 14 images each |
| Excluded | `20871905` and `BK21-64842-A-B` (13 images total), too few for reliable training and testing |

Each part type has a unique serial number stamped into its surface.

## 2. Region of interest

### 2.1 Manual ROI calibration

Open one reference image per part, note the pixel coordinates of the serial number corners, and hard code them. Coordinates were defined for all 10 parts this way. The downside is that every new part type needs human setup.

### 2.2 Automatic ROI with MSER

MSER (Maximally Stable Extremal Regions) [6] finds regions that stay stable across a range of intensity thresholds. Three steps:

1. MSER finds candidate regions on both the original and the inverted image. Candidates are filtered by size and aspect ratio to keep character-shaped blobs.
2. A density grid clusters the candidates spatially. The densest cluster is assumed to contain the serial number.
3. A bounding box with some padding is drawn around the cluster.

Across all 10 parts, the auto ROI achieved an average recall of 90% and an average IoU of 0.13 against the manual ROI. For 9 of 10 parts it captured the serial number region, with 3 to 10 times more background than needed. For part 23477043 the MSER candidates clustered in the wrong area and missed the serial number entirely. The auto ROI was still used for all parts, since 9 of 10 is a reasonable rate for an unsupervised method.

## 3. Preprocessing

### Attempt 1: direct thresholding (fail)

Otsu, adaptive, and manual thresholds all failed. The metal texture contains dark and light spots in the same intensity range as the character strokes.

### Attempt 2: edge detection (fail)

Canny and Sobel both failed. The textured surface is full of edges, and character edges are drowned out.

### Attempt 3: Gaussian blur before thresholding (partial)

Kernel sizes from 3 to 31 were tested. Small kernels (3 to 7) leave too much texture; large kernels (23+) start degrading the characters. There is a usable range around 11 to 15, but no single kernel perfectly separates texture from characters.

### Black top-hat transform

A die presses into soft metal and creates recessed valleys that reflect less light and appear darker than the surrounding surface. The black top-hat transform [1] extracts exactly these dark features:

$$T_{black}(f) = (f \bullet b) - f$$

A morphological closing fills the valleys, and subtracting the original leaves only what was filled in, which is the characters. A 25×25 rectangular structuring element was used because character strokes are roughly 5 to 20 pixels wide and the element must be larger than the features being extracted. CLAHE [4] (clipLimit = 3.0, tileGridSize = 8×8) is applied before the top-hat to normalize local contrast under uneven lighting.

### Complete preprocessing pipeline

1. Gaussian blur with a per-part kernel size (9 to 25)
2. CLAHE
3. Black top-hat, 25×25 kernel
4. Otsu thresholding [5]
5. Morphological opening, 3×3 kernel, to remove small noise specks

The Otsu step loses information: the grayscale top-hat output keeps the intensity difference between strong character edges and weak texture edges, and binarization removes it.

## 4. Character segmentation (fail)

Connected components on the binary output, filtered by height (>12% of ROI height), width (<18% of ROI width), area (>80 pixels), and aspect ratio (<2.0).

On one reference image per part, only 2 of 10 parts produced blob counts within ±2 of the expected character count. The other 8 produced too many blobs from residual texture noise.

From the parts where segmentation worked, 11 labeled 28×28 character crops were extracted. Most were corrupted by texture noise. Even with data augmentation, a KNN classifier trained on these samples cannot work at scale because the input quality is too poor. Segmentation, not the classifier, is the bottleneck: after Otsu thresholding, characters and texture both become white pixels.

## 5. Holistic classification (final)

Each of the 10 part types has a unique serial number, so the serial number region produces a unique visual pattern even when individual characters cannot be isolated. The whole preprocessed ROI is treated as one image.

**Features.** HOG [2] on the grayscale top-hat output (blur, CLAHE, black top-hat, no thresholding), resized to 128×64, with 9 orientations, 8×8 pixel cells, and 2×2 cell blocks. This gives a 3,780-dimensional feature vector per image.

**Classifier.** KNN [3]. The 1,872 images are split 80/20 into 1,497 training and 375 test images.

**Manual vs automatic ROI.**

| ROI method | Accuracy | Notes |
|---|---|---|
| Manual ROI (tight) | 89.3% | Only the serial number region |
| Auto ROI (MSER, wide) | 96.3% | Serial number plus surrounding context |

The wider detection area captures context around the serial number (screw holes, casting edges, part geometry). The system is therefore partly identifying parts by their surrounding structure rather than purely by the serial number. This makes the classifier more robust, especially for the D0CW parts whose serial numbers differ by a single digit.

## 6. Tools

Python, with OpenCV [7] for image processing, scikit-image [8] for HOG, scikit-learn [9] for KNN and evaluation, and Matplotlib for figures.

Bracketed numbers refer to [references.md](references.md).
