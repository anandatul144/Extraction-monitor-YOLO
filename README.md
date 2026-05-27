# Extraction-Monitor-YOLO

**Researcher:** Atul — ZF Friedrichshafen AG  
**Goal:** Detect surrogate model extraction attempts against a YOLO11-based perception API  
**Deadline:** November 2026  
**Status:** Phase 0 — Baseline established, attack reproduction next

---

## Problem Statement

An attacker with physical access to a production AV system (greybox threat model) can query the onboard perception API repeatedly, collect `(image → bbox+conf+class)` responses, and train a surrogate model. That surrogate is then used to craft adversarial patches against traffic signs or VRUs — without ever accessing model weights.

This project builds a monitor that detects extraction attempts from API query traffic patterns alone, agnostic to attack method and API output format.

---

## Threat Model

```
ATTACKER HAS                        ATTACKER LACKS
────────────────────────────────────────────────────
Physical vehicle (AV platform)      Model weights
Inference API access                Training data
Full response: bbox + conf + class  Gradients
Unlimited query budget              Source code
Physical test environment           Architecture details
```

---

## Attack Methods Under Study

```
ID    METHOD        PAPER                      API NEEDED     QUERY PATTERN
──────────────────────────────────────────────────────────────────────────────
A1    Knockoff Nets Orekondy, CVPR 2019        scores/labels  High-volume diverse
                                                              images; no spatial
                                                              continuity

A2    ZOO           Chen et al., CCS 2017      conf scores    Burst of perturbed
      (Zeroth-Order                                           versions of same image;
      Optimisation)                                           systematic pixel edits

A3    HSJA          Chen et al., 2020          hard labels    Binary search rhythm;
      (HopSkipJump)                            only           queries cluster near
                                                              decision boundary

A4    MEAOD         Li et al., arXiv 2312      bbox + conf    Domain-relevant images
                    .14677 (2023)              (OD-specific)  selected via active
                                                              learning; ~10k budget
```

---

## Detection Signals (per attack)

```
SIGNAL                  A1 Knockoff   A2 ZOO    A3 HSJA   A4 MEAOD
──────────────────────────────────────────────────────────────────
pHash distance (high)        ✓                              partial
pHash distance (low)                      ✓        ✓
Query burst rate                          ✓        ✓
Session volume (high)        ✓                              ✓
Class distribution shift     ✓                              ✓
Conf score variance                       ✓        ✓
Spatial discontinuity        ✓                              partial
```

pHash alone is insufficient for A1/A4 — these require combined signals.

---

## Measured Baselines (normal dashcam traffic)

```
Metric                        Value
─────────────────────────────────────
Total normal queries          37,086
Video source                  640×360 @ 30fps dashcam
Mean delta_ms (inter-query)   6.6ms (~150 queries/sec)
Mean detections/frame         4.4

pHash Hamming distances:
  Consecutive frames          min=0  max=28  mean=1.1
  Random scenes               min=4  max=38  mean=30.3

Inferred thresholds:
  ZOO / HSJA                  < 4    (perturbed same image)
  Normal scene variation      4–38
  Knockoff                    > 33   ⚠ overlaps normal → needs multi-signal
```

Note: consecutive max=28 reflects scene cuts in the source video, not noise.

---

## Repository Structure

```
YOLO-Monitoring/
├── models/
│   └── yolo11n.pt                  # YOLO11 nano weights
├── data/
│   ├── dash-cam-video.mp4          # normal baseline traffic source
│   └── gettyimage.jpg              # single image test
├── logs/
│   └── query_log_v2.jsonl          # append-only query log (monitor training data)
├── notebooks/
│   ├── 00_yolo_verify.ipynb        # YOLO install and first inference check
│   ├── 01_api_wrapper.ipynb        # v1 wrapper (superseded)
│   ├── 02_api_wrapper_v2_fixed.ipynb  # ← current wrapper (use this)
│   ├── 03_attack_zoo.ipynb         # [TODO] ZOO attack reproduction
│   ├── 04_attack_knockoff.ipynb    # [TODO] Knockoff Nets reproduction
│   ├── 05_attack_hsja.ipynb        # [TODO] HSJA reproduction
│   └── 06_attack_meaod.ipynb       # [TODO] MEAOD reproduction
└── README.md
```

---

## Log Schema

Each line in `query_log_v2.jsonl` is one API query:

```json
{
  "query_id":     18544,
  "session_id":   "user_normal",
  "timestamp":    1779797434.65,
  "delta_ms":     6.7,
  "source":       "normal_video",
  "md5":          "7de8ef72...",
  "phash":        "8289ddc1...",
  "n_detections": 8,
  "latency_ms":   6.6,
  "detections": [
    {"class": 2, "class_name": "car", "conf": 0.8031, "bbox": [305.29, 158.15, 369.25, 206.05]}
  ]
}
```

**Field notes:**
- `session_id` — groups queries by caller; each attack notebook uses a unique id
- `delta_ms` — null for first query in a session; used for burst detection
- `md5` — exact identity check (duplicate detection)
- `phash` — perceptual similarity (Hamming distance between queries)
- `source` — ground truth label for monitor training; never used as input feature

---

## Research Phases

```
Phase 0  Now – Jun 2026   Reproduce A1–A4; populate attack traffic in log
Phase 1  Jun – Jul 2026   Characterise signatures; extract monitor features
Phase 2  Jul – Aug 2026   Build monitor prototype (sliding window classifier)
Phase 3  Sep – Oct 2026   Harden against evasion; tune TPR/FPR
Phase 4  Oct – Nov 2026   Validate, benchmark, document
```

---

## Environment

```
OS:      Ubuntu 26.04 LTS
Python:  3.11.15
PyTorch: 2.12.0+cu130
GPU:     NVIDIA RTX 3060 (12GB VRAM) — cuda:0
YOLO:    Ultralytics 8.4.54 (yolo11n)
```

**Setup:**
```bash
python3.11 -m venv .py311yolo
source .py311yolo/bin/activate
pip install ultralytics opencv-python imagehash Pillow pandas
```

---

## Standards Context

Findings map to:

| Standard | Relevance |
|---|---|
| ISO 21434 | Cybersecurity — model extraction = TARA threat |
| ISO/PAS 8800 | AI safety — perception robustness requirements |
| ISO 21448 (SOTIF) | Functional safety — triggered by adversarial inputs |