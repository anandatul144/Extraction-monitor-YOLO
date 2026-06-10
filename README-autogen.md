# Experiment Log (autogen)

This file tracks experiment attempts, hyperparameter changes, dead ends,
and ideas across the project. It is supplementary to
[README.md](README.md) — the human-facing README should stay clean and
describe the *current* state; this file keeps the history of *how we got
here* and *what to try next*, so context isn't lost between sessions.

Newest entries at the top.

---

## 2026-06-10 — A2 (ZOO) Run 002: SPSA (gradient-magnitude-scaled) — FAILED

**Setup:** same target (car, conf=0.918, 32×32 patch), switched to SPSA
random-direction finite differences, writing to
`logs/attacks/attack_zoo_001.jsonl`.

**Hyperparameters:** `DELTA=16.0`, `LR=16.0`, `MAX_ITER=300`
(2 queries/iteration, 600 total).

**Result:**
- 300 iterations, 600 queries
- Confidence: 0.9179 → 0.9177 (Δ = 0.00023)
- Patch pixels changed by mean 0.6, max 5 (out of 255) — **sub-pixel
  updates per iteration**.
- `success: false` — nowhere near `CONF_TARGET=0.20`.

**Diagnosis:** the update rule `adv_img -= LR · grad_est` where
`grad_est = (conf_plus - conf_minus)/(2·DELTA) · direction` is mis-scaled.
With `conf_diff ≈ 0.001` and `LR/(2·DELTA) = 0.5`, each iteration's update
is ~0.0005 — about 1/2000th of a pixel level. The gradient *direction* is
roughly correct (confidence trended down), but gradient-magnitude scaling
makes the step size proportional to a tiny, noisy confidence delta, so it
never accumulates to a meaningful perturbation within budget. Bumping `LR`
by raw multiplication doesn't fix this robustly — the right fix is to
decouple step size from gradient magnitude entirely.

**Next attempt (Run 003):** switch to a **greedy fixed-step search**
(SimBA-style): each iteration, try `+DELTA·d` and `-DELTA·d` for a random
direction `d`; if either reduces confidence, commit that *full* `±DELTA`
step; otherwise discard and try a new direction. Guarantees real
pixel-level progress on every successful iteration instead of sub-pixel
drift.

---

## 2026-06-10 — A2 (ZOO) Run 001: per-pixel finite differences — FAILED

**Setup:** `notebooks/03_attack_zoo.ipynb`, target = highest-confidence
"car" detection (conf=0.918) in `data/gettyimage.jpg`, 32×32 patch centered
on bbox, per-pixel finite-difference gradient estimation.

**Hyperparameters:** `DELTA=1.0`, `LR=2.0`, `MAX_ITER=50`
(2 × 32² = 2,048 queries/iteration).

**Result:**
- 50 iterations, 307,200 queries
- Confidence: 0.9179 → 0.9171 (Δ = 0.0008)
- **Effectively no movement** — well within measurement noise.

**Diagnosis:** ±1/255 single-pixel perturbations are below YOLO11's
confidence sensitivity floor. The finite-difference gradient estimate was
dominated by noise, and `LR × noise` produced negligible per-iteration
updates. The per-pixel approach also burns the query budget extremely fast
(2,048 queries/iteration) for no benefit.

**Side effects of this run:**
- `logs/query_log_v2.jsonl` grew from 24MB (37,086 lines) to 437MB
  (325,743 lines) — the 288,657 attack queries were written straight into
  the committed baseline log.
- A re-run of `notebooks/02_api_wrapper_v2_fixed.ipynb` (to check log state)
  regenerated a second 37,086-line baseline with new timestamps, written to
  `logs/query_log_v2-1.jsonl` — diverging from the original committed
  baseline.

**Cleanup performed:**
- Reverted `logs/query_log_v2.jsonl` and
  `notebooks/02_api_wrapper_v2_fixed.ipynb` to the last clean commit
  (af7c4f2), discarding both the 288K attack-query rows and the duplicate
  baseline re-run.
- Deleted `logs/query_log_v2-1.jsonl` and the duplicate `model/` directory
  (kept `models/`, which matches the path used in notebooks).
- Added `logs/attacks/*` to `.gitignore`.

**Next attempt (Run 002):** switch to SPSA (random-direction finite
differences) — `DELTA=16.0`, `LR=16.0`, `MAX_ITER=300`, 2 queries/iteration
(600 total instead of 307,200). Attack queries now write to
`logs/attacks/attack_zoo_001.jsonl` (gitignored), never to the committed
baseline log.

**Open questions for Run 002:**
- Does a 32×32 patch at the bbox center actually overlap a region YOLO's
  classifier head is sensitive to, or would an edge/corner of the bbox
  perturb confidence faster?
- Is `LR=16` too aggressive (visible/unrealistic perturbation) or still too
  weak? Watch `conf_log` trend over the first ~20 iterations before
  committing to a full 300-iteration run.
- If SPSA also stalls, consider: larger patch, multiple random directions
  averaged per iteration (reduces variance at the cost of more queries), or
  attacking a lower-confidence detection (closer to the 0.25 threshold,
  less perturbation needed to push below it).

---

## Ideas backlog

- A1 (Knockoff Nets): need a diverse image pool (not just dashcam frames) —
  check if a public driving dataset subset is feasible to query against.
- A3 (HSJA): hard-label-only attack — current wrapper always returns
  conf+bbox, need a "hard label mode" toggle to simulate this API surface.
- A4 (MEAOD): ~10k query budget, active-learning image selection — largest
  scope item, defer until A1–A3 signatures are characterized.
- Monitor feature extraction (Phase 1): once 2-3 attack types have clean
  logged signatures, build a notebook that computes sliding-window features
  (pHash distance distribution, query rate, session volume, class
  distribution) per session and compares attack vs. normal.
