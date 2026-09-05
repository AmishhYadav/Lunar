# CLAUDE.md — LUCID

Operating instructions for Claude Code on this repository.

**Read `PROJECT_CONTEXT.md` before doing anything else.** It contains the problem statement, the solution architecture, the reasoning behind every decision, and the scope boundaries. This file governs *how* we work; that file governs *what* we build.

---

## 0. Model and orchestration policy — NON-NEGOTIABLE

**Opus orchestrates. Sonnet codes.**

| Role | Model | Responsibilities |
|---|---|---|
| **Orchestrator** | Opus | Reasoning, architecture decisions, task decomposition, reviewing subagent output, integration, resolving conflicts, deciding what gets built and what gets cut |
| **Implementer** | Sonnet subagents | Writing code, writing tests, refactoring, running experiments, producing files |

### Hard rules

1. **Opus writes no production code.** If Opus finds itself writing an implementation, stop and delegate. Opus may write: task specifications, interface/contract stubs, review comments, and this kind of planning artifact.
2. **Maximum 3 Sonnet subagents in parallel. Never more.** This is for manageability and reviewability, not compute. If a plan needs more than 3 concurrent tasks, it is decomposed wrong — re-sequence it.
3. **Every subagent task must be independently testable.** If two subagents would edit the same file, they do not run in parallel.
4. **Every subagent gets a written contract before it starts:** input types, output types, file paths it owns, files it must not touch, the test that defines "done."
5. **Opus reviews every subagent's output before merging.** No subagent output goes into the pipeline unreviewed.
6. **Opus maintains `PROGRESS.md`** — what is done, what is in flight, which subagent owns what, what is blocked.

### Standard parallel batches

Batches are pre-planned so no two agents collide on files:

```
Batch 1 (parallel, 3):
  A → src/io/          PDS4 parsing, reprojection, tiling
  B → src/relight/     DEM fetch, Lunar-Lambertian renderer, cast shadows
  C → src/matching/    matcher wrappers, RANSAC, bucketing

Batch 2 (parallel, 3):
  D → src/refine/      LSM sub-pixel + covariance
  E → src/eval/        synthetic GT harness, metrics, illumination sweep
  F → src/products/    warping, exports, GCP file

Batch 3 (parallel, 2):
  G → app/             Streamlit demo
  H → experiments/     ablation runner
```

**Critical path:** B (renderer) blocks E (evaluation harness). Build B first, stub the rest. Without the renderer we have neither the method nor the ground truth.

---

## 1. What this project is

A lunar image registration system for SIH 2026, ISRO problem statement on multi-modal, Sun-angle and scale-invariant correspondence for Chandrayaan-2 imagery.

**The one thing that makes it different:** instead of building features invariant to illumination, we physically remove the illumination difference by re-rendering the reference terrain under the source image's exact solar geometry, using a DEM and the Lunar-Lambertian reflectance model.

If any code change makes that claim weaker, less measurable, or less demonstrable, it is the wrong change.

---

## 2. Repository layout

```
lucid/
├── CLAUDE.md
├── PROJECT_CONTEXT.md
├── PROGRESS.md                 # Opus maintains
├── pyproject.toml
├── configs/
│   ├── default.yaml
│   ├── ohrc_nac.yaml           # regime: sub-metre, relight coarse only
│   ├── tmc2_nac.yaml           # regime: 5-10 m, relight fully
│   └── iirs_wac.yaml           # regime: 80-100 m, relight fully
├── data/
│   ├── raw/                    # PDS4 products (gitignored)
│   ├── dem/                    # SLDEM / LOLA / TMC DEM (gitignored)
│   └── cache/                  # reprojected tiles (gitignored)
├── src/lucid/
│   ├── io/                     # PDS4 labels, rasters, reprojection, tiling
│   ├── relight/                # DEM, reflectance model, shadows, transfer
│   ├── matching/               # RoMa v2 / SuperGlue / RootSIFT, RANSAC, bucketing
│   ├── geometry/               # transform models, cross-validated selection
│   ├── refine/                 # LSM sub-pixel + covariance
│   ├── eval/                   # synthetic GT, metrics, sweeps
│   ├── products/               # warping, exports, GCP, maps
│   ├── viz/                    # overlays, plots
│   ├── types.py                # shared dataclasses — the contracts
│   └── pipeline.py             # orchestrator
├── tests/
├── experiments/                # ablation scripts, sweep runners
├── app/                        # Streamlit demo
└── results/                    # run outputs (gitignored)
```

---

## 3. Shared contracts

Every module speaks through `src/lucid/types.py`. **Define these first, before any implementation.** Subagents implement against them and never redefine them.

```python
@dataclass(frozen=True)
class Acquisition:
    sensor: str                 # "OHRC" | "TMC2" | "IIRS" | "NAC" | "WAC" | "TC"
    gsd_m: float
    sun_azimuth_deg: float
    sun_elevation_deg: float
    emission_angle_deg: float | None
    footprint_lonlat: tuple     # (min_lon, min_lat, max_lon, max_lat)
    crs: str
    product_id: str

@dataclass
class AlignedPair:
    source: np.ndarray          # reprojected, common CRS + GSD
    reference: np.ndarray
    src_meta: Acquisition
    ref_meta: Acquisition
    crs: str
    gsd_m: float
    transform: Affine           # map transform shared by both
    search_radius_px: float     # prior from geodetic uncertainty
    valid_mask: np.ndarray

@dataclass
class Correspondence:
    src_xy: np.ndarray          # (N, 2) float, sub-pixel
    ref_xy: np.ndarray          # (N, 2) float, sub-pixel
    confidence: np.ndarray      # (N,)
    sigma: np.ndarray | None    # (N, 2, 2) LSM covariance; None before refinement
    is_inlier: np.ndarray       # (N,) bool

@dataclass
class RegistrationResult:
    correspondences: Correspondence
    model_type: str             # "similarity" | "affine" | "poly2" | "tps"
    model_params: dict
    metrics: dict               # see Section 6
    mode: str                   # "relit_full" | "relit_coarse" | "intensity_only" | "blind"
```

**Rule:** a subagent that needs a new field proposes it to Opus. It does not add it unilaterally.

---

## 4. Non-negotiable engineering rules

### 4.1 Physics and correctness

- **Never use a homography as the primary transform model.** Pushbroom sensors over non-planar terrain do not produce projective relationships. Model selection is by cross-validated held-out residual, never fit residual.
- **Never implement Fourier-Mellin or blind rotation/scale search in the default path.** Rotation and scale come from metadata. This is a deliberate architectural decision, not an oversight.
- **Never fit a parabola to a correlation peak for sub-pixel estimation.** Peak-locking biases estimates toward integer positions, which is fatal to our central claim. Use affine-photometric LSM.
- **Uncertainty comes from the LSM normal-equation covariance.** Never from hand-weighted confidence scores.
- **Angles in degrees at I/O boundaries, radians internally.** Convert once at the boundary; document it.
- **Never silently return a low-confidence registration.** Flag it in `RegistrationResult.mode` and `metrics.json`.

### 4.2 Evaluation integrity

- **Every RMSE is labelled `rmse_fit` or `rmse_holdout`.** Never an unlabelled `rmse` key anywhere in the codebase or output.
- Held-out split is 70/30 on inliers, with a fixed seed recorded in `run_config.yaml`.
- Synthetic ground truth is generated by the same renderer used in the method. Document this explicitly — it is a feature, not a leak, because the *matcher* never sees the GT.
- If an ablation rung shows no measurable gain, report it and cut the component. Do not keep components for narrative reasons.

### 4.3 Reproducibility

- Every run writes `run_config.yaml` capturing: git SHA, all config values, random seeds, model checkpoints and versions, input product IDs, DEM source and version.
- All randomness seeded. No exceptions.
- No hardcoded paths. Everything through config.
- Model checkpoints are downloaded by a script, never committed.

### 4.4 Performance

- OHRC strips are ~1.08 gigapixels (12000 × 90148 in ISRO's own paper). **Never load a full strip into memory.** Tile with overlap, stream, and use the geodetic prior to skip non-overlapping tile pairs.
- Every matcher call runs on a tile, never a full image.
- Target: full OHRC↔NAC registration in under 10 minutes on an M4 MacBook Pro. If a design cannot hit that, flag it to Opus before implementing.
- On Apple Silicon prefer MPS; provide a CPU fallback. Verify numerical parity between backends on a fixed test case.

### 4.5 Code standards

- Python 3.11+, type hints on every public function.
- Pure functions where possible; I/O isolated to `src/lucid/io/` and `src/lucid/products/`.
- No module imports from `pipeline.py`. Dependencies point inward.
- Every module has a `tests/test_<module>.py` with at least one synthetic-data test that runs in under 5 seconds.
- Docstrings state units. `gsd_m`, `sun_azimuth_deg`, `shift_px` — units in the name or the docstring, always.

---

## 5. Definition of done, per module

A subagent's task is complete only when all boxes are ticked.

**`io/`** — Parses a real PDS4 label into `Acquisition`. Reprojects two rasters into a shared CRS/GSD. Round-trip test: reproject → inverse-reproject → error < 0.01 px. Tiling covers the full valid mask with correct overlap.

**`relight/`** — Renders a DEM under a given sun az/el with Lunar-Lambertian reflectance and cast shadows. Test: a synthetic Gaussian hill renders with the shadow on the correct side for four cardinal sun azimuths. Test: rendering under identical sun geometry as a real image gives NCC > 0.3 against that image.

**`matching/`** — Each backend (RoMa v2, SuperGlue, RootSIFT) returns a valid `Correspondence`. Test: on an image pair with a known synthetic affine, ≥80% of returned matches are inliers. Bucketing raises coverage entropy without dropping mean confidence below the quality floor.

**`geometry/`** — Model selection picks `similarity` for a synthetic similarity pair and does not over-select `tps`. Test: cross-validated residual rises when a deliberately overfitting model is forced.

**`refine/`** — On a synthetic pair with a known 0.37 px shift, LSM recovers it to within 0.05 px. Covariance is positive-definite for every converged point. Non-converged points are rejected, not returned with garbage.

**`eval/`** — Produces all metrics in Section 6. Generates the illumination sweep with configurable Δ solar elevation. Tier-1 accuracy on a synthetic pair matches the injected shift.

**`products/`** — `gcps.txt` is accepted by `gdalwarp -tps` without error. All files in the result package are written and non-empty.

---

## 6. Required metrics in `metrics.json`

```json
{
  "rmse_fit_px": 0.0,
  "rmse_holdout_px": 0.0,
  "rmse_synthetic_gt_px": 0.0,
  "inlier_count": 0,
  "total_matches": 0,
  "inlier_ratio": 0.0,
  "coverage_fraction": 0.0,
  "spatial_entropy_normalized": 0.0,
  "mean_sigma_px": 0.0,
  "subpixel_improvement_px": 0.0,
  "delta_sun_elevation_deg": 0.0,
  "delta_sun_azimuth_deg": 0.0,
  "mode": "relit_full",
  "model_type": "affine",
  "runtime_s": 0.0,
  "peak_memory_mb": 0.0,
  "registration_valid": true
}
```

`rmse_synthetic_gt_px` is `null` when no synthetic GT applies. `registration_valid` is `false` when any quality gate fails — and when it is false, the caller must be able to see *which* gate failed.

---

## 7. Do NOT build

These are cut with reasons. If a subagent proposes them, Opus rejects and cites this section.

| Item | Reason |
|---|---|
| Fourier-Mellin transform | Rotation and scale come from metadata |
| PC-SIFT | Unvalidated; SIFT/RootSIFT is the honest classical baseline |
| LightGlue | Redundant given RoMa v2 and SuperGlue branches |
| DINOv2 as a separate stage | That combination *is* RoMa; use the checkpoint |
| Learned + classical "match fusion" | Complexity without demonstrated gain; simple fallback only |
| MLflow / Weights & Biases | `run_config.yaml` + `metrics.json` is sufficient at this scale |
| Docker | Not needed for the demo; costs setup time |
| FastAPI service layer | Streamlit demo is sufficient |
| Full Hapke BRDF with fitted parameters | Deferred to finale; Lunar-Lambertian first |
| Homography as primary model | Wrong physics for pushbroom over non-planar terrain |

---

## 8. Working rhythm for Opus

Each cycle:

1. **Read `PROGRESS.md`.** Know what is done and in flight.
2. **Decide the next 1–3 tasks.** Never more than 3 in parallel; never two touching the same file.
3. **Write each task contract:** inputs, outputs, files owned, files forbidden, the test that defines done, and the relevant section of `PROJECT_CONTEXT.md`.
4. **Dispatch to Sonnet subagents.**
5. **Review returned code against the contract and Section 4.** Reject anything that violates a non-negotiable rule, however well-written.
6. **Integrate, run the test suite, update `PROGRESS.md`.**
7. **Re-check scope.** If a task has drifted toward something in Section 7, stop it.

When a decision is genuinely ambiguous, Opus asks Amish rather than guessing. Architecture and scope decisions are his; implementation is delegated.

---

## 9. Anti-drift checks

Run these mentally before every merge:

- **Does the innovation still work?** Can we still show the relit pair side by side and the Δ-solar-elevation plot? If a refactor breaks that demo path, revert it.
- **Is any accuracy claim untraceable?** Every number in the PPT must trace to Tier-1 or Tier-2 evaluation, in code, reproducibly.
- **Has scope crept?** Compare against Section 7 and `PROJECT_CONTEXT.md` §8.
- **Is the delta D→E still measurable?** That single ablation step is the entire project. It must remain isolatable at all times.
- **Would an ISRO/SAC judge recognize this as derivative?** They wrote arXiv:2509.04775. If a component duplicates their CLAHE/PCA/histogram-matching approach without a measured improvement, it is not a contribution.

---

## 10. Success criteria

The project succeeds if, on demo day, we can:

1. Register a real OHRC↔NAC pair end to end and export the full result package.
2. Show `rmse_holdout_px` and `rmse_fit_px` side by side, honestly labelled.
3. Show the relit reference next to the source and have the audience see they now look like the same image.
4. Show the Δ-solar-elevation plot where our curve stays flat and the baseline curves collapse.
5. Answer every question in `PROJECT_CONTEXT.md` §11 without hedging.
6. Hand an ISRO analyst `gcps.txt` and have `gdalwarp -tps` work first try.

Anything that does not serve one of these six is not a priority.
