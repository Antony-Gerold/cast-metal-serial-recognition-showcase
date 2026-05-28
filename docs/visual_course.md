# Machine Vision Project 2 — Complete Visual Learning Course

## How to Use This Course
This course teaches you every concept in your final report, with visual references to the figures in `report_figures/`. Each module has:
- **📸 Visual**: Which figure to look at
- **🧠 Concept**: The theory explained simply
- **💻 Code**: The actual implementation
- **📝 Report Connection**: Exact section this appears in

---

# Module 1: The Problem — Why Cast Metal Is Hard

## 📸 Look at: `raw_samples.jpg`

You'll see 10 grayscale images of cast metal parts with green boxes around the serial numbers. Notice how the surface texture is rough and noisy — that's the cast metal grain from the manufacturing process.

## 🧠 Why This Is Harder Than Normal OCR

Normal OCR (like scanning a printed page) works because there's a clean separation: black ink on white paper. On cast metal:

1. **Characters are recesses** — a die physically presses into soft metal, creating shallow grooves
2. **Texture is everywhere** — the casting process creates a bumpy surface at roughly the same scale as the character strokes
3. **Low contrast** — the characters and background are the same material (metal), so they reflect light similarly

Think of it like trying to read letters scratched into sandpaper — the scratches and the sandpaper grain look similar to a camera.

## 📸 Look at: `segmentation_failure.png`

This shows part GM 23477043 — one of the hardest parts. The original ROI looks barely readable even to humans. With blur=11, we get 6 blobs instead of 10 characters. With blur=19 (universal), same problem. The texture bumps and character strokes overlap in the frequency domain — no single blur value can separate them.

## 📝 Report Connection
> **Section 1 (Introduction):** "The casting process leaves a rough, textured surface... Separating the character signal from the texture noise turned out to be the central difficulty."

---

# Module 2: The Preprocessing Pipeline

## 📸 Look at: `pipeline_stages.png`

This shows all 6 stages for part 23354374. Follow the image left→right, top→bottom:

### Stage 1: Original ROI
The raw crop from the full image. You can barely see the characters against the texture.

### Stage 2: Gaussian Blur

```python
blurred = cv2.GaussianBlur(gray_roi, (blur_size, blur_size), 0)
```

**What it does:** Smooths the image by averaging each pixel with its neighbors. The kernel size (11, 19, 25, etc.) controls how much smoothing.

**Why we use it:** High-frequency texture (the tiny bumps) gets smoothed away, but the broader character shapes survive because characters span more pixels than individual texture bumps.

**The tradeoff:** Too little blur → texture remains. Too much blur → characters dissolve. This is why different parts need different blur values (9-25).

### Stage 3: CLAHE (Contrast Limited Adaptive Histogram Equalization)

```python
clahe = cv2.createCLAHE(clipLimit=3.0, tileGridSize=(8,8))
enhanced = clahe.apply(blurred)
```

**What it does:** Enhances contrast locally. The image is divided into 8×8 tiles, and each tile gets its own histogram equalization. The "clip limit" (3.0) prevents over-amplifying noise.

**Why we use it:** Different parts of the ROI may have different lighting. CLAHE makes the characters equally visible across the entire region.

**Normal HE vs CLAHE:** Regular histogram equalization treats the whole image equally — if one corner is brighter, it might wash out. CLAHE adapts to local conditions.

### Stage 4: Black Top-Hat Transform

## 📸 Look at: `tophat_explained.png` — THIS IS THE KEY FIGURE

```python
kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (25, 25))
tophat = cv2.morphologyEx(enhanced, cv2.MORPH_BLACKHAT, kernel)
```

**The formula:** `Black Top-Hat = Closing(image) - image`

**Step by step:**
1. **Morphological closing** slides a 25×25 rectangular block across the image. At each position, it finds the maximum in the neighborhood (dilation), then the minimum (erosion). The effect: it "fills in" any dark features smaller than 25×25 pixels.
2. **Subtraction:** The closed image minus the original gives you only what was "filled in" — which is exactly the dark recessed characters!

**Why 25×25?** Character strokes are 5-20 pixels wide. The structuring element must be LARGER than the features you want to extract. 25×25 captures all character strokes.

**Why specifically black top-hat (not white)?** Because stamps create DARK recesses (valleys). The white top-hat extracts bright features. The physics of stamping → dark valleys → black top-hat. This is why the report says the choice was "motivated by physics rather than heuristics."

### Stage 5: Otsu Thresholding

```python
_, binary = cv2.threshold(tophat, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
```

**What it does:** Automatically finds the best threshold to separate foreground (characters) from background. Otsu's method minimizes the within-class variance — it finds the value where the two intensity peaks are best separated.

**Why Otsu works here:** After the top-hat, we have a bimodal image — dark background (value ≈ 0) and bright characters. Otsu naturally finds the boundary.

### Stage 6: Morphological Opening (Cleanup)

```python
clean_kern = cv2.getStructuringElement(cv2.MORPH_RECT, (3, 3))
cleaned = cv2.morphologyEx(binary, cv2.MORPH_OPEN, clean_kern)
```

**What it does:** Erosion followed by dilation with a 3×3 kernel. Removes small isolated white pixels (noise specks) while preserving larger connected regions (characters).

## 📝 Report Connection
> **Section 3.2 (Preprocessing Pipeline):** "The real breakthrough came from combining CLAHE with the black top-hat transform. I chose the black top-hat specifically because of how stamps physically work."

---

# Module 3: Character Segmentation — What Worked (and What Didn't)

## 📸 Look at: `master_pipeline_results.png`

This shows all 10 parts side-by-side: original ROI on the left, pipeline output with detected blobs on the right.

### Connected Component Analysis

```python
num_labels, labels, stats, centroids = cv2.connectedComponentsWithStats(cleaned)
```

After the pipeline produces a binary image, connected component analysis finds groups of touching white pixels. Each group is a potential character. OpenCV gives us:
- `stats`: bounding box (x, y, width, height) and area for each blob
- `centroids`: center point of each blob

### Filtering: Which Blobs Are Characters?

```python
for i in range(1, num_labels):  # skip 0 (background)
    bx, by, bw, bh, area = stats[i]
    
    # Too short → not a character
    if bh < roi_height * 0.12: continue
    
    # Too wide → probably merged characters
    if bw > roi_width * 0.18: continue
    
    # Too small → noise
    if area < 80: continue
    
    # Too wide relative to height → not character-shaped
    if bw / bh > 2.0: continue
    
    blobs.append((bx, by, bw, bh))
```

### Results: 7/10 Parts Pass

| Parts That Pass | Parts That Fail |
|----------------|----------------|
| 23354374 (8 chars) | GM 23477043 (coarse texture) |
| GM 23301608 (10 chars) | D0CW-51300 (small+noisy) |
| GM 23305982 (10 chars) | 70085119-0300 (very fine) |
| GM 23477043 ✗ | |
| GM 23497667 (10 chars) | |
| 52611-0E110-A (11 chars) | |
| D0CW-51301 (9 chars) | |
| D0CW-51310 (9 chars) | |

The 3 failing parts share a problem: their texture grain is nearly the same size as character strokes. No blur value can separate them.

## 📝 Report Connection
> **Section 3.3 (Character Segmentation Results)** and **Table 2**

---

# Module 4: OCR Baseline — Why Tesseract Fails

## 📸 Look at: `mvp_ocr_test.png`

### The Experiment
We ran Tesseract OCR with 800 configurations:
- 10 images × 16 preprocessing variants × 5 page segmentation modes
- Preprocessing: raw, Otsu, adaptive threshold, CLAHE, Gaussian blur, median blur, morphological operations, Canny edges, Sobel edges, and combinations

### The Result: 0% Exact Match

Not a single configuration correctly read any serial number. The best partial match was 70% on GM 23497667 (the clearest part).

### Why It Fails
Tesseract's neural network was trained on scanned documents — printed text on paper. Cast metal characters have:
- No consistent stroke width (varies with casting pressure)
- No clean background
- Non-standard font (die-stamped)

This isn't a preprocessing problem — no amount of filtering can make metal look like paper. It's a **domain mismatch**.

### Code

```python
import pytesseract

results = []
for img_path in test_images:
    for preprocess in preprocessing_variants:
        processed = preprocess(img_path)
        for psm in [3, 6, 7, 8, 13]:  # Page segmentation modes
            text = pytesseract.image_to_string(processed, config=f'--psm {psm}')
            accuracy = compare_with_ground_truth(text, expected)
            results.append(accuracy)

# Result: max(results) = 0% exact match
```

## 📝 Report Connection
> **Section 3.4 (OCR Baseline Comparison):** "I ran Tesseract OCR with 800 configurations... The result was 0% exact match accuracy."

---

# Module 5: Character-Level Classification — The Overfitting Trap

## 📸 Look at: `character_samples.png`

### What We Built

From the 7 parts where segmentation succeeded on reference images, we extracted individual character crops (28×28 pixels each) and computed HOG features:

```python
from skimage.feature import hog

def extract_hog(img_28x28):
    features = hog(img_28x28, 
                   orientations=9,        # 9 gradient direction bins
                   pixels_per_cell=(7,7), # each cell is 7×7 pixels
                   cells_per_block=(2,2), # normalize over 2×2 cell blocks
                   block_norm='L2-Hys')
    return features  # 324-dimensional vector
```

### The Numbers

| Stage | Accuracy | What It Means |
|-------|----------|---------------|
| Training data | 57 chars from 7 images | Tiny dataset |
| After augmentation | 1,197 samples | 21× expansion |
| 5-fold CV on augmented | 98.9% | Looks amazing... |
| Deployed on all 1,872 images | 12-18% | ...but it collapsed |

### Why 98.9% → 12%?

The classifier was perfect on clean, well-segmented characters. But when applied to the full dataset:

1. **Segmentation breaks** — ROI coordinates from the reference image don't perfectly align on other images (slight camera position shifts)
2. **Noise blobs pass validation** — if a texture region happens to produce the right number of blobs, they get labeled as "characters" even though they're noise
3. **Only 0.3% of images** (6 out of 1,872) actually produce correct character segmentation

This is the classic **overfitting to test conditions** trap: the model works in controlled settings but fails in the real world.

## 📝 Report Connection
> **Section 3.5-3.6:** "the bootstrapping experiment showed that only 6 out of 1,872 images (0.3%) produce segmentation results where the extracted characters actually spell the correct serial number."

---

# Module 6: Automatic ROI Detection — MSER

## 📸 Look at: `mser_explained.png` and `auto_roi_detection_results.png`

### What Is MSER?

**Maximally Stable Extremal Regions** — an algorithm that finds connected regions whose pixel sets remain stable across many intensity thresholds.

Imagine slowly raising a water level across the image. At each level, some "islands" (connected bright regions) exist. MSER finds islands that stay roughly the same shape across many levels — these are "stable" and likely to be meaningful structures like text characters.

### The Implementation

```python
mser = cv2.MSER_create()
mser.setMinArea(60)    # minimum character size
mser.setMaxArea(5000)  # maximum character size
mser.setDelta(5)       # stability range

# Detect on original AND inverted image
regions_orig, _ = mser.detectRegions(gray)
regions_inv, _ = mser.detectRegions(255 - gray)

# Filter for character-shaped regions
char_candidates = []
for region in all_regions:
    x, y, bw, bh = cv2.boundingRect(region)
    if 8 < bh < 120 and 4 < bw < 80 and bw/bh < 2.5:
        char_candidates.append((x, y, bw, bh))
```

### Spatial Clustering

After finding character candidates, we cluster them to find text lines:

```python
# Grid-based density map (40×25 pixel cells)
grid_w, grid_h = 40, 25
density = np.zeros((h//grid_h + 1, w//grid_w + 1))

for (bx, by, bw, bh, cx, cy) in char_candidates:
    density[cy // grid_h, cx // grid_w] += 1

# Smooth with directional kernel (wider = prefer horizontal text lines)
density_smooth = cv2.GaussianBlur(density, (9, 3), 0)

# Find connected regions above 25% of peak density
threshold = max_density * 0.25
```

### Results

| Part | Recall | IoU | What Happened |
|------|--------|-----|---------------|
| 23354374 | 1.00 | 0.12 | Found SN, but region 8× too large |
| GM 23497667 | 0.80 | 0.45 | Best result, still 2× too large |
| D0CW parts | 1.00 | 0.02-0.04 | Nearly the entire image |

**Average: 94% recall, 14% IoU** — the serial number IS inside the detected region, but so is most of the image. Cast metal texture creates thousands of false "character" detections that inflate the bounding box.

### Why This Justifies Per-Part Enrollment

If auto-detection gives you a region that's 5× too large, the HOG features would be dominated by background texture instead of character patterns. You need a tight, precise ROI. Since auto-detection can't provide that on cast metal, one-time per-part-type calibration (identify ROI on one reference image) is the practical engineering solution.

## 📝 Report Connection
> **Section 3.7 (Automatic ROI Localization Attempt)** and **Table 3**

---

# Module 7: The Holistic Solution — HOG on Whole Regions

## 📸 Look at: `hog_explained.png` and `roi_samples.png`

### The Key Insight

If you can't segment individual characters reliably, don't try. Instead, classify the **entire serial number region as one image**. Each part type has a unique serial number, so each region has a unique visual pattern — a "fingerprint."

### What Is HOG?

**Histogram of Oriented Gradients** computes edge directions in local patches:

## 📸 Look at: `hog_explained.png` — Study this carefully

```python
from skimage.feature import hog

# Resize ROI to standard size
roi_resized = cv2.resize(preprocessed_roi, (128, 64))

# Extract HOG features
features = hog(roi_resized,
               orientations=9,        # 9 direction bins (0°-180°)
               pixels_per_cell=(8,8), # each cell is 8×8 pixels
               cells_per_block=(2,2), # normalize across 2×2 blocks
               block_norm='L2-Hys')

# Result: 3,780-dimensional feature vector
```

**How HOG works, step by step:**
1. **Compute gradients:** For each pixel, calculate horizontal and vertical intensity changes. This gives magnitude (how strong the edge is) and direction (which way it points).
2. **Divide into cells:** The 128×64 image is divided into 8×8 pixel cells → 16×8 = 128 cells.
3. **Histogram per cell:** Each cell gets a histogram with 9 bins representing edge directions (0°, 20°, 40°, ..., 160°). Pixels vote for their direction bin, weighted by their magnitude.
4. **Block normalization:** 2×2 groups of cells are normalized together for lighting invariance. This is why it's robust to brightness changes.
5. **Concatenate:** All normalized histograms are concatenated into one long vector.

**Why HOG works for this problem:**
- Characters are defined by their edges (strokes going up, down, diagonal)
- Different serial numbers have different edge patterns
- HOG captures these patterns regardless of exact brightness or contrast
- The 3,780 dimensions capture enough detail to distinguish 10 part types

### KNN Classification

```python
from sklearn.neighbors import KNeighborsClassifier

# Train: HOG features from all 1,872 images
knn = KNeighborsClassifier(n_neighbors=1, metric='euclidean')
knn.fit(X_train, y_train)  # X: 3780-dim vectors, y: part labels

# Predict: find the most similar training image
prediction = knn.predict(X_test)
```

**Why k=1?** Each test image should be most similar to images of the same part type. The nearest neighbor (k=1) directly captures this. Higher k values (3, 5) actually hurt accuracy because they bring in confusing neighbors from similar-looking parts (like the three D0CW variants).

**Why KNN over SVM?** On this dataset, KNN k=1 (92.3%) outperformed SVM-RBF (89.1%) and SVM-Linear (88.5%). KNN also has a practical advantage: you can explain each classification by showing the most similar training image, which helps operators trust the system.

## 📝 Report Connection
> **Section 3.8 (Holistic ROI Classification):** "The holistic pipeline crops the serial number ROI... resizes to a standard 128×64 pixels, extracts HOG features from the entire region."

---

# Module 8: Evaluation — How to Test Honestly

## 📸 Look at: `eval_protocols.png`

### Protocol 1: 80/20 Stratified Split → 92.3%

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42)
```

**What it tests:** Can the model classify images it hasn't seen, from part types it HAS seen?
**Limitation:** Only one random split — could get lucky or unlucky.

### Protocol 2: 5-Fold Stratified CV → 92.3% ± 1.2%

```python
from sklearn.model_selection import StratifiedKFold, cross_val_score

skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(knn, X, y, cv=skf)
```

**What it tests:** Same question, but repeated 5 times with different splits. The ±1.2% tells us the model is stable.
**Limitation:** Images from the same part type (and potentially the same capture session) could appear in both train and test.

### Protocol 3: GroupKFold → 91.9% ± 1.6% (FAIREST)

```python
from sklearn.model_selection import GroupKFold

gkf = GroupKFold(n_splits=5)
# groups = image IDs, ensuring no image leaks across folds
scores = cross_val_score(knn, X, y, groups=image_ids, cv=gkf)
```

**What it tests:** Same question, but guarantees no image appears in both train and test. This is the most honest evaluation.
**Why it matters:** Without GroupKFold, augmented or near-duplicate images could leak information.

### Protocol 4: Leave-One-Part-Out → 0%

```python
# Train on 9 part types, test on the 1 held-out type
# Result: 0% — can't recognize what you've never seen
```

**What it tests:** Can the model identify a completely new part type?
**Answer:** No. This is expected and correct — the system is an **identification** system (like face recognition), not a **reading** system. You must enroll a part type before identifying it.

## 📸 Look at: `confusion_matrix_v8.png`

The confusion matrix shows where errors happen. Notice:
- Large parts (GM 23477043: 596 images) are nearly perfect
- D0CW variants (14 images each) confuse each other — their serial numbers differ by ONE digit

## 📝 Report Connection
> **Section 4.2 (Holistic Classification Results)** and **Table 5**

---

# Module 9: The Full Pipeline Code

Here's the complete working code that generates classification results:

```python
import cv2
import numpy as np
from sklearn.neighbors import KNeighborsClassifier
from sklearn.model_selection import StratifiedKFold, GroupKFold, cross_val_score
from skimage.feature import hog
import os

# ============================================================
# CONFIGURATION: One entry per part type
# ROI coordinates identified from a single reference image
# ============================================================
parts_config = {
    '23354374':      {'roi': (330,310,890,430),  'blur': 11},
    '23301608':      {'roi': (240,680,930,800),  'blur': 25},
    '23305982':      {'roi': (210,575,730,690),  'blur': 19},
    '23477043':      {'roi': (170,135,920,285),  'blur': 11},
    '23497667':      {'roi': (210,830,960,964),  'blur': 17},
    '52611-0E110-A': {'roi': (50,425,1200,550),  'blur': 15},
    'D0CW-51300':    {'roi': (540,365,960,420),  'blur': 25},
    'D0CW-51301':    {'roi': (440,100,900,160),  'blur': 19},
    'D0CW-51310':    {'roi': (470,170,870,225),  'blur': 9},
    '70085119-0300': {'roi': (275,550,740,620),  'blur': 25},
}

def preprocess_roi(gray_roi, blur_size):
    """Full preprocessing pipeline."""
    blurred = cv2.GaussianBlur(gray_roi, (blur_size, blur_size), 0)
    clahe = cv2.createCLAHE(clipLimit=3.0, tileGridSize=(8,8))
    enhanced = clahe.apply(blurred)
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (25, 25))
    tophat = cv2.morphologyEx(enhanced, cv2.MORPH_BLACKHAT, kernel)
    return tophat

def extract_holistic_features(img_path, roi, blur_size, pad=20):
    """Extract HOG features from the whole serial number region."""
    img = cv2.imread(img_path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    x1, y1, x2, y2 = roi
    
    # Add padding for robustness to slight position shifts
    h, w = gray.shape
    x1 = max(0, x1 - pad)
    y1 = max(0, y1 - pad)
    x2 = min(w, x2 + pad)
    y2 = min(h, y2 + pad)
    
    crop = gray[y1:y2, x1:x2]
    preprocessed = preprocess_roi(crop, blur_size)
    resized = cv2.resize(preprocessed, (128, 64))
    
    features = hog(resized, orientations=9, pixels_per_cell=(8,8),
                   cells_per_block=(2,2), block_norm='L2-Hys')
    return features  # 3,780 dimensions

# ============================================================
# PROCESS ALL IMAGES
# ============================================================
X_all = []  # HOG feature vectors
y_all = []  # Part type labels
g_all = []  # Image group IDs (for GroupKFold)

dataset_root = '/path/to/IndustrialDigitRecognition'
for part_name, config in parts_config.items():
    part_dir = os.path.join(dataset_root, part_name)
    for img_file in sorted(os.listdir(part_dir)):
        if not img_file.endswith('.jpg'): continue
        features = extract_holistic_features(
            os.path.join(part_dir, img_file),
            config['roi'], config['blur'])
        X_all.append(features)
        y_all.append(part_name)
        g_all.append(img_file)

X = np.array(X_all)
y = np.array(y_all)
groups = np.array(g_all)

# ============================================================
# EVALUATE
# ============================================================
knn = KNeighborsClassifier(n_neighbors=1, metric='euclidean')

# 1. Stratified K-Fold
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(knn, X, y, cv=skf)
print(f"5-Fold Stratified: {scores.mean()*100:.1f}% ± {scores.std()*100:.1f}%")

# 2. GroupKFold (no image leakage)
gkf = GroupKFold(n_splits=5)
scores_g = cross_val_score(knn, X, y, groups=groups, cv=gkf)
print(f"GroupKFold:        {scores_g.mean()*100:.1f}% ± {scores_g.std()*100:.1f}%")
```

---

# Module 10: Understanding the Results & What to Say in Your Presentation

## 📸 Look at: `project_journey.png`

### The Story Arc (6-minute presentation)

**Minute 1 — The Problem:**
"Here's a raw image. See how the texture and characters look similar? That's the core challenge."
→ Show `raw_samples.jpg`

**Minute 2 — OCR Fails:**
"I tried 800 Tesseract configurations. Zero percent. It's a domain mismatch — OCR expects printed text, not metal stamps."
→ Show `mvp_ocr_test.png`

**Minute 3 — Preprocessing Breakthrough:**
"The black top-hat transform works because stamps create physical recesses. The math matches the physics."
→ Show `tophat_explained.png` and `pipeline_stages.png`

**Minute 4 — Character Approach & Failure:**
"Segmentation works on reference images (98.9% classifier accuracy) but fails at scale (12-18%) because noise blobs pass count validation."
→ Show `segmentation_failure.png`

**Minute 5 — Auto-Detection & Holistic Solution:**
"I tried automatic ROI detection with MSER — 94% recall but only 14% IoU. Too imprecise. So I classify the whole region as an image — each serial number is a visual fingerprint."
→ Show `auto_roi_detection_results.png` and `hog_explained.png`

**Minute 6 — Results:**
"92.3% accuracy across 1,872 images, validated with GroupKFold to prevent data leakage. Main limitation: dataset imbalance — parts with 14 images score lower than parts with 596."
→ Show `confusion_matrix_v8.png`

### Key Questions the Professor Might Ask

**Q: "Why not use deep learning?"**
A: "No annotated character-level dataset existed. The holistic approach achieves 92.3% without any labeling beyond folder names. Deep learning (e.g., U-Net for segmentation) would be a good future direction if annotated data becomes available."

**Q: "Why are the ROI coordinates manual?"**
A: "I attempted automatic detection using MSER and morphological analysis — it achieves 94% recall but only 14% IoU because cast metal texture creates thousands of false character detections. Per-part enrollment from a single reference image is standard industrial practice for fixed inspection stations."

**Q: "Why does k=1 beat k=3 and k=5?"**
A: "For part identification, each image should be closest to other images of the same part. Higher k brings in confusing neighbors, especially from visually similar parts like the three D0CW variants that differ by only one digit."

**Q: "What does the 0% leave-one-part-out mean?"**
A: "It means the system can't identify a part type it has never seen — which is expected and correct. This is an identification system (like fingerprint matching), not a reading system. New parts must be enrolled first."

**Q: "Why did character-level fail at scale?"**
A: "The segmentation, not the classifier, was the bottleneck. Only 0.3% of images produced correct character extraction. ROI coordinates from reference images don't perfectly align across hundreds of captures, and Otsu thresholding on noisy metal produces random blobs."

---

# Quick Reference: Key Numbers for the Report

| Metric | Value |
|--------|-------|
| Total images | 1,872 (from 10 part types) |
| OCR accuracy | 0% (800 experiments) |
| Character segmentation | 7/10 parts pass |
| Character classifier (clean) | 98.9% CV |
| Character classifier (full data) | 12-18% |
| Auto ROI recall | 94% |
| Auto ROI IoU | 14% |
| **Holistic ROI accuracy** | **92.3%** |
| Holistic GroupKFold | 91.9% ± 1.6% |
| HOG feature dimensions | 3,780 |
| Parts tested | 10 of 12 (requirement: ≥8) |
| Blur universal best | 19 (4/10 pass) |
| Blur per-part best | 9-25 (7/10 pass) |

---

# Glossary of Key Terms

| Term | Simple Explanation |
|------|-------------------|
| **Black Top-Hat** | Extracts dark features (recesses) smaller than the structuring element |
| **CLAHE** | Local contrast enhancement that adapts to different image regions |
| **HOG** | Captures edge direction patterns — like a "shape fingerprint" |
| **KNN** | Classifies by finding the most similar training example |
| **MSER** | Finds stable connected regions across intensity thresholds |
| **Otsu** | Automatically picks the best threshold for binarization |
| **Morphological Opening** | Erosion→dilation; removes small noise specks |
| **Morphological Closing** | Dilation→erosion; fills in small holes |
| **Connected Components** | Finds groups of touching pixels in a binary image |
| **IoU** | Intersection over Union — measures how well two boxes overlap (1.0 = perfect) |
| **GroupKFold** | Cross-validation that prevents data leakage between groups |
| **Structuring Element** | The shape used for morphological operations (our: 25×25 rectangle) |
