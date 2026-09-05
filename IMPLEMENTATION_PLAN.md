# LUCID — Master Implementation Plan

**SIH 2026 · ISRO · Multi-modal, Sun-angle and scale-invariant image correspondence using Chandrayaan-2 optical images (OHRC, TMC-2, IIRS)**

Single executable plan covering Day 0 → submission. Governs `PROJECT_CONTEXT.md` (what) and `CLAUDE.md` (how) into a concrete build order.

---

## 1. Context

`/Users/amish/Lunar` currently holds four documents and no code: `SIH_PROBLEM_STATEMENT.md` (verbatim PS), `PROJECT_CONTEXT.md` (architecture + reasoning), `DATASET_STATUS.md` (data link audit), `CLAUDE.md` (operating rules). The design is sound and the innovation is well-chosen. What does not exist yet is a build order that survives contact with the actual data formats and the actual hardware.

The pitch stands unchanged: **everyone else builds features invariant to illumination; we physically remove the illumination difference by re-rendering the reference terrain under the source image's exact solar geometry.** The delta **D→E** in the ablation ladder is the entire project.

Deadline **30 Sep 2026**. Today **5 Sep 2026**. 25 days.

---

## 2. Review findings — gaps in the current design and their resolutions

The solution was reviewed clause by clause against the PS and against how the data and models actually behave. Ten gaps found. None are fatal; all are resolved below and folded into the phases.

| # | Gap | Why it matters | Resolution (now part of this plan) |
|---|---|---|---|
| **G1** | **Chandrayaan-2 calibrated products are line-scanner geometry, not map-projected.** `PROJECT_CONTEXT.md` §5 Stage 1 says "reproject both to a common map projection" as if both arrive georeferenced. Ames Stereo Pipeline's CH-2 page shows the real route is `isisimport → spiceinit → mapproject` with ~200 GB of SPICE kernels. | Stage 1 is the foundation of everything. If it silently assumes georeferenced input, every downstream stage inherits a wrong assumption. | Introduce a **`GeometryBackend` protocol** with two implementations. Default `LabelCornerBackend`: build a GDAL GCP grid from PDS4 label corner/centre lat-lon, warp with TPS. No ISIS, no kernels, runs today. Residual of tens–hundreds of metres is *exactly what the pipeline exists to remove*. `IsisSpiceBackend` is written as a stub with the same interface, marked **[DEFER]**. |
| **G2** | **The relighting "transfer" operation is never defined.** §5 Stage 2 says "use the pair to transfer the reference into the source's lighting" — the single most important operation in the project has no equation. | Two teammates would implement two different things. Naive `ref × R_src / R_ref` divides by zero in every reference shadow. | Define it as **explicit albedo estimation and re-rendering** with a hard validity mask. Full math in §5, module `relight/transfer.py`. Pixels shadowed in *either* geometry, or below a `cos θi` floor, are excluded from matching and the valid fraction is reported in `metrics.json`. |
| **G3** | **RoMa v2 ships a custom CUDA kernel** for its two-stage refinement. `CLAUDE.md` §4.4 targets an M4 MacBook Pro and says "prefer MPS". These are incompatible. | The primary matcher would not run on the stated demo machine. Discovered late, this kills the demo. | **Matcher ladder, decided.** `EfficientLoFTR` (pure PyTorch, MPS-safe) is the primary local matcher and the demo path. `RoMa v2` runs only as a Colab/Kaggle batch job to produce ablation rung D/E numbers. Both sit behind one `Matcher` protocol so the swap is a config key. |
| **G4** | **The DEM-to-reference frame offset is assumed to be zero.** §5 Stage 2's cross-question answer says LOLA and NAC "share a coordinate system by construction". True for the frame definition; not true for an individual NAC image, which carries residual pointing error before bundle adjustment. | The render lives in the DEM frame, not the reference image's frame. Matching source↔render therefore aligns source to the DEM, leaving an unmeasured offset. | Add **`relight/selfcheck.py`**: register `render(ref_sun_geometry)` against the real reference image. Costs one extra registration, and returns two things for free — the DEM↔reference offset (fed forward as a correction) and a direct **renderer-fidelity metric** (`render_ref_ncc`). This also becomes a slide: proof the physics engine is right. |
| **G5** | **Cast shadows by per-pixel ray-marching is O(N·L).** §5 Stage 2 says "ray-march along the solar azimuth per pixel". On tiled DEMs in Python this dominates runtime. | The renderer is the critical path (`CLAUDE.md` §0). Slow renderer means no illumination sweep, means no money plot. | Replace with the **rotate-and-sweep horizon algorithm**, O(N): rotate the DEM so sun azimuth is along +x, one running-max sweep per row, rotate the mask back. Exact, not approximate. Numba-jitted. Full derivation in §5. |
| **G6** | **Random 70/30 hold-out leaks under spatial autocorrelation.** §5 Stage 8 Tier 2 and `CLAUDE.md` §4.2 specify a random split on inliers. Registration residuals are spatially correlated, so a random hold-out point usually sits ~1 px from a training point. | We are explicitly attacking the baseline for reporting fit residual as accuracy. If our own hold-out is optimistic, that attack is not defensible under questioning. | **Spatially blocked K-fold** (K=5, folds = contiguous KMeans clusters on match coordinates). Report `rmse_holdout_px` from blocked folds, and keep `rmse_fit_px` alongside as `CLAUDE.md` §4.2 requires. Strictly strengthens the argument we are already making. |
| **G7** | **Repeated resampling biases sub-pixel estimates.** Stage 1 reprojects, Stage 2 relights, Stage 6 claims sub-pixel accuracy — each interpolation smooths the signal and pulls estimates toward grid nodes. | Sub-pixel accuracy is the headline claim; a self-inflicted interpolation bias undercuts it. | **Compose transforms, resample once.** Carry the map transform analytically. Final LSM runs on the **source's native grid** against the reference's native grid, with the map transform applied to coordinates, never to pixels. Quantify the residual resampling bias in Tier-1 and report it. |
| **G8** | **Tier-3 evaluation ("compare against LOLA altimetry crossovers") is not executable as written.** LOLA crossovers validate altimetry self-consistency, not 2D image registration. | Tier 3 becomes a stub on demo day, and Stage 8 is marked **[BUILD — CRITICAL]**. | Tier 3 = **(a) cycle-consistency closure error** — register A→B, B→C, C→A and report the loop residual, a true accuracy check requiring no ground truth at all; **(b) registration against LROC NAC DTM orthophotos**, which are already tied to the LOLA frame. Both executable, both independent of our renderer. |
| **G9** | **"Sub-pixel" is ambiguous about which pixel.** Source GSD, reference GSD and common-grid GSD all differ. | A judge will ask. An unlabelled "0.4 px" is exactly the vagueness we are criticising in the baseline. | Define once: **all px units are common-grid pixels**, and **every px figure is reported with its metre equivalent** next to it. Enforced by a `metrics.json` schema test. |
| **G10** | **No calibrated confidence gate.** §8.3 says "low-confidence registrations are flagged, never silently returned", but no rule decides what low means. | `registration_valid` in `metrics.json` has no defined mechanism behind it. | Train a small **confidence calibration model** (M11 below) on Tier-1 synthetic runs: features → P(error < 1 px). Threshold set at a chosen precision. Small, honest, and turns a hand-waved gate into a measured one. |

**Minor items folded in without needing separate treatment:** per-pixel emission angle varies across a pushbroom swath — we hold it constant per tile and document it; IIRS is demoted to lowest build priority and reached via the TMC-2 chain, never matched directly to NAC; SuperGlue's Magic Leap weights are research-licence, noted in the ablation table.

---

## 3. Decisions locked for this plan

| Decision | Choice | Consequence |
|---|---|---|
| Learned matcher | **EfficientLoFTR primary (local, MPS), RoMa v2 in Colab/Kaggle for ablations** | Demo never depends on CUDA. Ablation rungs D/E still get the strongest matcher. |
| Data access | **ISSDC account approved — real CH-2 data available now** | Real-pair integration starts in Batch 1, not Batch 3. NAC DTM harness still built first because Tier-1 requires it regardless. |
| Georeferencing | **`LabelCornerBackend` (GDAL GCP + TPS) as default; ISIS/SPICE stubbed** | No 200 GB dependency on the critical path. Documented honestly as a stated assumption. |
| Tier-3 evaluation | **Cycle-consistency closure + NAC DTM ortho check** | Executable in the time available; genuinely independent. |

---

## 4. Model inventory — everything that gets built or wrapped

`CLAUDE.md` §0 forbids Opus from writing production code. Every model below is a Sonnet subagent deliverable against an Opus-written contract.

### 4.1 Physical and analytical models — built from scratch

| ID | Model | Module | Definition of done |
|---|---|---|---|
| **M1** | **Lunar-Lambertian reflectance** — `R_LL = A·[2cosθi/(cosθi+cosθe)] + (1−A)·cosθi` | `relight/reflectance.py` | Gaussian hill renders with shading gradient on the correct side for 4 cardinal azimuths; reduces to Lambertian at `A=0`; degenerate `cosθi+cosθe→0` guarded |
| **M2** | **Cast-shadow horizon model** — rotate-and-sweep, O(N) | `relight/shadows.py` | Matches a brute-force ray-march reference on a 256×256 synthetic DEM to within 0 pixels; ≥100× faster; correct for 8 azimuths × 3 elevations |
| **M3** | **Albedo-transfer relighting model** — estimate albedo from reference, re-render under source geometry, emit validity mask | `relight/transfer.py` | Round-trip test: relight ref→src geometry→back to ref recovers original within 2% RMS on valid pixels; valid mask excludes every pixel shadowed in either geometry |
| **M4** | **Transform model family** — similarity (4 dof), affine (6), 2nd-order polynomial (12), thin-plate spline | `geometry/models.py` | Each fits and inverts its own synthetic case exactly; TPS regularisation parameter exposed |
| **M5** | **Affine-photometric LSM (Gruen/Ackermann)** — 6 geometric + 2 radiometric parameters, Gauss-Newton, with normal-equation covariance | `refine/lsm.py` | Recovers a known 0.37 px shift to within 0.05 px (`CLAUDE.md` §5); covariance positive-definite on every converged point; non-converged points rejected, never returned |
| **M6** | **Search-radius prior model** — geodetic position uncertainty → pixel search radius, per sensor and pyramid level | `io/prior.py` | Given 500 m default uncertainty and a coarse GSD, returns a radius that provably contains the true offset on all Tier-1 synthetic cases |
| **M7** | **Confidence calibration model** — logistic regression mapping (match count, inlier ratio, mean σ, render NCC, coverage entropy, Δsun) → P(error < 1 px) | `eval/calibration.py` | Trained on ≥200 Tier-1 runs; reliability diagram plotted; threshold chosen at a stated precision; drives `registration_valid` |

### 4.2 Learned matchers — wrapped, not trained

All behind one `Matcher` protocol in `matching/base.py`. No matcher is fine-tuned in the internal round.

| ID | Model | Backend | Role | Notes |
|---|---|---|---|---|
| **M8** | **EfficientLoFTR** | PyTorch, MPS/CPU/CUDA | **Primary matcher; the demo path** | Pure PyTorch. Runs on the M4. |
| **M9** | **RoMa v2** | CUDA only (custom kernel) | Ablation rungs D/E, best-case numbers | `pip install romav2`; weights auto-download; code MIT, DINOv3 backbone carries its own licence — cite it |
| **M10** | **SuperPoint + SuperGlue** | PyTorch | **The published ISRO baseline (rung B) — must be beaten on its own terms** | Magic Leap weights are research-licence; state this in the ablation table |
| **M11** | **RootSIFT + FLANN + ratio test** | OpenCV | Classical interpretability floor (rung A) | `cv2.SIFT_create()` with L1-normalise-then-sqrt |

### 4.3 Deferred models — promised in the PPT, built only at the Grand Finale

**M12** matcher fine-tuned on rendered lunar data (validated by MARTIAN, MoonAnything) · **M13** full Hapke BRDF with fitted parameters. Both **[DEFER]** per `PROJECT_CONTEXT.md` §8.2. Not in this plan's scope.

---

## 5. The two pieces of mathematics that must be right

Written out here because they are the project, and because G2 and G5 exist precisely because they were not written down.

### 5.1 Relighting transfer (M3) — resolves G2

Sun unit vector from azimuth `az` (clockwise from North) and elevation `el`, in a local East-North-Up frame:

```
l̂ = ( sin(az)·cos(el),  cos(az)·cos(el),  sin(el) )
```

Surface normal from DEM gradients `p = ∂z/∂x`, `q = ∂z/∂y` (metres per metre, so divide by GSD):

```
n̂ = (−p, −q, 1)ᵀ / sqrt(1 + p² + q²)
```

View vector `v̂` from emission angle and look azimuth; nadir `(0,0,1)` when the label does not supply it (documented approximation, constant per tile).

```
cos θi = max(n̂ · l̂, 0)
cos θe = max(n̂ · v̂, ε)
R_LL   = A·[ 2·cos θi / (cos θi + cos θe) ] + (1 − A)·cos θi
```

Render under a given geometry, gated by the shadow mask `S` from M2:

```
R(az, el) = S(az, el) · R_LL(az, el)
```

**The transfer itself.** Estimate albedo from the real reference, then re-render under the source's Sun:

```
A_est    = I_ref_real / max( R(az_ref, el_ref), τ )
I_relit  = A_est · R(az_src, el_src)

valid = S_ref ∧ S_src ∧ (R_ref > τ) ∧ (R_src > τ)
```

`τ` defaults to 0.1 (config). Everything outside `valid` is excluded from matching and the valid fraction goes into `metrics.json`. Both `I_relit` and the source are zero-mean unit-variance normalised per tile before matching, which removes residual radiometric scale without touching structure.

**Why this and not `I_ref × R_src/R_ref`:** algebraically the same on illuminated pixels, but writing it as albedo estimation names the physical quantity, makes the failure mode obvious (albedo is undefined in shadow), and makes the mask non-optional.

### 5.2 Cast shadows by rotate-and-sweep (M2) — resolves G5

Rotate the DEM so the Sun's azimuth lies along +x. Pixel `i` is shadowed iff some pixel `j` further towards the Sun rises above the solar ray:

```
∃ j with x_j > x_i  such that  z_j > z_i + (x_j − x_i)·tan(el)
```

Subtract `x·tan(el)` from both sides and define `g = z − x·tan(el)`. The condition collapses to `g_j > g_i`. So:

1. Rotate DEM by `−az` (bilinear, `reshape=True`).
2. Per row, sweep from high `x` to low `x` keeping a running max of `g`. Pixel shadowed iff `running_max > g_i`.
3. Rotate the boolean mask back; threshold at 0.5.

O(N) instead of O(N·L), exact rather than sampled. Numba-jitted inner loop. Rotation interpolation error is the only approximation and is validated against brute-force ray-marching in the M2 done-test.

---

## 6. Repository skeleton

Extends `CLAUDE.md` §2 with the modules the review added. New files marked ★.

```
lucid/
├── CLAUDE.md · PROJECT_CONTEXT.md · SIH_PROBLEM_STATEMENT.md · DATASET_STATUS.md
├── PROGRESS.md                     ★ Opus maintains (CLAUDE.md §0 rule 6)
├── pyproject.toml · .gitignore · README.md
├── configs/
│   ├── default.yaml · ohrc_nac.yaml · tmc2_nac.yaml · iirs_wac.yaml
│   └── colab_romav2.yaml           ★ CUDA-only ablation config
├── scripts/
│   ├── fetch_dem.py                ★ SLDEM2015 / LOLA polar tile fetch + mosaic
│   ├── fetch_nac_dtm.py            ★ NAC DTM site fetch (ortho GeoTIFF + DTM)
│   ├── fetch_wac.py                ★ WAC global mosaic subset
│   ├── fetch_checkpoints.py        ★ matcher weights (never committed)
│   └── ingest_ch2.py               ★ PRADAN product → normalised local layout
├── data/{raw,dem,ref,cache}/       gitignored
├── src/lucid/
│   ├── types.py                    contracts — CLAUDE.md §3, extended per §7 below
│   ├── io/       pds4.py · geometry_backend.py ★ · reproject.py · tiling.py · prior.py ★
│   ├── relight/  dem.py · reflectance.py · shadows.py · transfer.py · selfcheck.py ★
│   ├── matching/ base.py · eloftr.py · romav2.py · superglue.py · rootsift.py
│   │             · tilematch.py ★ · ransac.py · bucketing.py
│   ├── geometry/ models.py · selection.py (spatially blocked CV ★)
│   ├── refine/   lsm.py · covariance.py
│   ├── eval/     synthgt.py · metrics.py · sweep.py · closure.py ★ · calibration.py ★
│   ├── products/ warp.py · gcp.py · package.py
│   ├── viz/      overlays.py · plots.py
│   └── pipeline.py
├── tests/                          one per module, each <5 s on synthetic data
├── experiments/  ablation.py · money_plot.py · runtime_bench.py
├── app/          streamlit_app.py
└── results/                        gitignored
```

---

## 7. Contract changes to `types.py`

`CLAUDE.md` §3 says a subagent proposes new fields to Opus rather than adding them. These are proposed here, up front, so Batch 1 never blocks:

```python
# Acquisition — add
look_azimuth_deg: float | None      # for the view vector; None → nadir assumed
geometry_backend: str               # "label_corner" | "isis_spice"

# AlignedPair — add
valid_relight_mask: np.ndarray | None   # §5.1 validity; None when mode != relit_*
dem_ref_offset_px: tuple[float,float]   # from relight/selfcheck.py (G4)
render_ref_ncc: float | None            # renderer fidelity (G4)

# RegistrationResult — add
gate_failures: list[str]                # which quality gate failed (CLAUDE.md §6)
confidence_calibrated: float            # M7 output, P(error < 1 px)
```

Rationale is recorded per field so a reviewer can trace each one to a gap in §2.

---

## 8. Build phases

Batches follow `CLAUDE.md` §0: **Opus orchestrates and reviews, Sonnet subagents write all code, maximum 3 in parallel, never two agents on the same file.**

---

### Phase 0 — Foundations · Day 1 · Opus solo, no subagents

Opus may write specifications and contract stubs (`CLAUDE.md` §0 rule 1). No implementation.

| Task | Output |
|---|---|
| 0.1 | `pyproject.toml`, `.gitignore`, `git init`, package skeleton with empty modules |
| 0.2 | **`src/lucid/types.py` complete** — every dataclass from `CLAUDE.md` §3 plus §7 additions. Nothing else starts until this is frozen. |
| 0.3 | `configs/default.yaml` + three regime configs. Every threshold named here, zero hardcoded constants downstream. |
| 0.4 | `PROGRESS.md` initialised — task table, owners, blockers |
| 0.5 | Three subagent contracts written for Batch 1: inputs, outputs, files owned, files forbidden, the test that defines done |
| 0.6 | `scripts/fetch_nac_dtm.py` spec + one NAC DTM site chosen and downloaded manually to unblock everything |

**Why the NAC DTM site first:** it delivers an orthorectified NAC image *and* its DTM in the *same frame*, as GeoTIFF, with no login and no ISIS. That single download unblocks M1–M3, M5 and all of Tier-1 before any Chandrayaan-2 file is touched.

**Gate:** `python -c "from lucid.types import *"` succeeds; `pytest tests/test_types.py` passes.

---

### Phase 1 — Batch 1 · Days 2–6 · 3 Sonnet subagents in parallel

`CLAUDE.md` §0 critical path: **B (renderer) blocks E (evaluation). B is the priority.**

#### Agent A — `src/lucid/io/` + `scripts/ingest_ch2.py`, `scripts/fetch_*.py`

| Task | Detail |
|---|---|
| A1 | `pds4.py` — parse PDS4 XML via `pds4_tools`, populate `Acquisition`. Handle OHRC, TMC-2, IIRS label variants and the CH-2 Local Data Dictionary namespace. Sun azimuth/elevation, incidence, emission, corner lat-lon, GSD, product ID. |
| A2 | `geometry_backend.py` — **G1 resolution.** `GeometryBackend` protocol; `LabelCornerBackend` builds a GDAL GCP grid from label corners and warps with TPS; `IsisSpiceBackend` raises `NotImplementedError` with a pointer to the ASP route. |
| A3 | `reproject.py` — equirectangular for \|lat\|<60°, polar stereographic for \|lat\|≥60°. **Single composed resample only (G7).** Both rasters onto one shared grid at one GSD. |
| A4 | `tiling.py` — 1024×1024 tiles, 25% overlap, tile-pair generation from footprint intersection dilated by the search radius. Never load a full strip (`CLAUDE.md` §4.4). |
| A5 | `prior.py` — **M6.** Position uncertainty → pixel search radius per sensor and per pyramid level. |
| A6 | The three fetch scripts + `ingest_ch2.py` normalising a PRADAN download into `data/raw/`. |

**Done:** round-trip reproject → inverse → error < 0.01 px (`CLAUDE.md` §5). Real PDS4 label parses into `Acquisition`. Tiling covers the valid mask with correct overlap. **Forbidden:** everything outside `io/` and `scripts/`.

#### Agent B — `src/lucid/relight/` ← **CRITICAL PATH, highest priority**

| Task | Detail |
|---|---|
| B1 | `dem.py` — load SLDEM2015 / LOLA polar / NAC DTM, mosaic tiles, reproject **onto the exact common grid** from A3, fill voids |
| B2 | `reflectance.py` — **M1**, §5.1 exactly |
| B3 | `shadows.py` — **M2**, rotate-and-sweep per §5.2, Numba-jitted, validated against brute-force ray-march |
| B4 | `transfer.py` — **M3**, albedo estimation + re-render + validity mask per §5.1 |
| B5 | `selfcheck.py` — **G4.** Register `render(ref geometry)` against the real reference; return `dem_ref_offset_px` and `render_ref_ncc` |

**Done (`CLAUDE.md` §5 plus review additions):** Gaussian hill shadows fall on the correct side for 4 cardinal azimuths · M2 matches brute-force ray-march exactly and is ≥100× faster · rendering under a real image's own Sun geometry gives NCC > 0.3 against it · relight round-trip recovers the original within 2% RMS on valid pixels. **Forbidden:** everything outside `relight/`.

#### Agent C — `src/lucid/matching/`

| Task | Detail |
|---|---|
| C1 | `base.py` — `Matcher` protocol: `match(img_a, img_b, mask_a, mask_b) → Correspondence` |
| C2 | `rootsift.py` — **M11**. OpenCV SIFT, L1-normalise then sqrt, FLANN, Lowe ratio |
| C3 | `eloftr.py` — **M8, primary.** Device selection `mps → cuda → cpu`, tile-level batching |
| C4 | `superglue.py` — **M10.** SuperPoint + SuperGlue, licence noted in the docstring |
| C5 | `romav2.py` — **M9.** `pip install romav2`; raise a clear error on non-CUDA rather than failing obscurely |
| C6 | `tilematch.py` — **new.** Run any matcher over tile pairs, accumulate matches in map coordinates, KD-tree dedup at 1 px radius keeping highest confidence |
| C7 | `ransac.py` — MAGSAC++ / USAC via OpenCV, seeded |
| C8 | `bucketing.py` — grid cells, best-per-cell **subject to a mandatory quality floor** (`PROJECT_CONTEXT.md` §8.3 — the floor always overrides the quota); report occupancy fraction and normalised entropy |

**Done:** every backend returns a valid `Correspondence` · on a known synthetic affine ≥80% of returned matches are inliers · bucketing raises coverage entropy without dropping mean confidence below the floor. **Forbidden:** everything outside `matching/`.

**Phase 1 gate (Opus review, `CLAUDE.md` §0 rule 5):** all three merged, full test suite green, `PROGRESS.md` updated. Reject any output violating §4 however well written.

---

### Phase 2 — Batch 2 · Days 6–11 · 3 Sonnet subagents in parallel

#### Agent D — `src/lucid/refine/` + `src/lucid/geometry/`

| Task | Detail |
|---|---|
| D1 | `refine/lsm.py` — **M5.** 8 parameters (6 affine + brightness + contrast), 31×31 patch, Gauss-Newton, bicubic resample, converge at \|Δshift\| < 0.01 px |
| D2 | Rejection gates: >15 iterations · σ₀ above threshold · normal-matrix condition number > 1e8 · drift > 3 px from the matcher's initial estimate. Rejected points are dropped, never returned with garbage. |
| D3 | `refine/covariance.py` — `σ₀² (AᵀA)⁻¹`, extract the 2×2 shift block. **Uncertainty comes from here and nowhere else** (`CLAUDE.md` §4.1). |
| D4 | **G7:** LSM operates on native grids with map transforms applied to coordinates, not to pixels |
| D5 | `geometry/models.py` — **M4**, four transform models |
| D6 | `geometry/selection.py` — **G6.** Spatially blocked K-fold (K=5, KMeans clusters on match xy). Select by mean held-out RMSE with the 1-SE rule preferring the simpler model. Emit `rmse_fit_px` alongside. |

**Done:** known 0.37 px shift recovered within 0.05 px · covariance positive-definite on every converged point · selection picks `similarity` for a synthetic similarity pair and does not over-select `tps` · held-out residual rises when a deliberately overfitting model is forced. Cross-check against `skimage.registration.phase_cross_correlation(upsample_factor=100)` on synthetic shifts.

#### Agent E — `src/lucid/eval/`

| Task | Detail |
|---|---|
| E1 | `synthgt.py` — **Tier 1.** Take a real image, apply a known sub-pixel shift and a known relighting via M1–M3. Exact per-pixel truth. Document explicitly that the matcher never sees the GT (`CLAUDE.md` §4.2). |
| E2 | `metrics.py` — every key in `CLAUDE.md` §6, plus `valid_relight_fraction`, `render_ref_ncc`, `dem_ref_offset_m`. **Schema test enforcing G9:** no unlabelled `rmse` key anywhere, and every px value paired with its metre equivalent. |
| E3 | `closure.py` — **Tier 3(a), G8.** Cycle consistency: A→B, B→C, C→A; report loop residual. No ground truth needed. |
| E4 | **Tier 3(b), G8** — registration against NAC DTM orthophotos in the LOLA frame |
| E5 | `sweep.py` — illumination sweep, Δ solar elevation 0°→60° in 5° steps, configurable scenes and methods |
| E6 | `calibration.py` — **M7.** Logistic regression on ≥200 Tier-1 runs; reliability diagram; threshold at a stated precision; drives `registration_valid` and `gate_failures`. |

**Done:** all §6 metrics produced · Tier-1 accuracy matches the injected shift · sweep runs end to end at reduced resolution · reliability diagram generated.

#### Agent F — `src/lucid/products/` + `src/lucid/viz/`

| Task | Detail |
|---|---|
| F1 | `warp.py` — warp the source into the reference frame with the selected model |
| F2 | `gcp.py` — **the detail that turns a demo into a tool.** GDAL-format GCP export. Done means `gdalwarp -tps` accepts `gcps.txt` **first try, no edits**. |
| F3 | `package.py` — the full `PROJECT_CONTEXT.md` §5 Stage 7 result directory, including `run_config.yaml` with git SHA, seeds, checkpoint versions, product IDs and DEM version |
| F4 | `viz/overlays.py` — match overlay, inlier/outlier split, registration overlay, difference image, per-point error ellipses from M5 covariance |
| F5 | `viz/plots.py` — **the money plot**, coverage/accuracy tradeoff curve, reliability diagram |

**Done:** every file in the result package written and non-empty · `gdalwarp -tps` succeeds unmodified · error ellipses render from real covariance, not from confidence scores.

**Phase 2 gate:** `pipeline.py` runs a NAC-DTM pair end to end and emits a complete result package.

---

### Phase 3 — Batch 3 · Days 11–15 · 2 Sonnet subagents in parallel

#### Agent G — `app/streamlit_app.py`

Implements the 13-step demo flow of `PROJECT_CONTEXT.md` §10 exactly, in order. The two steps that must be visually unmistakable:

- **Step 5** — render under reference Sun vs render under source Sun, side by side: *"this is the illumination difference, made explicit"*
- **Step 6** — relit reference next to the source: *"now they look like the same picture"*

Runs on the M4 with EfficientLoFTR. Results cached so the live demo never waits on a matcher.

#### Agent H — `experiments/`

| Task | Detail |
|---|---|
| H1 | `ablation.py` — rungs **A→H** from `PROJECT_CONTEXT.md` §7, resumable, one result row per rung |
| H2 | `money_plot.py` — 5 scenes × 13 Δel values × 3 methods ≈ 195 registrations |
| H3 | `runtime_bench.py` — runtime and peak memory per stage; verifies the `CLAUDE.md` §4.4 target of full OHRC↔NAC under 10 minutes on the M4 |
| H4 | `configs/colab_romav2.yaml` + a Colab notebook running rungs D and E with **M9 RoMa v2** on CUDA, writing metrics back as JSON |

---

### Phase 4 — Real data, ablations, the money plot · Days 15–21 · Opus + 1–2 subagents

| Day | Work |
|---|---|
| 15–16 | Pull 3 real CH-2 footprints from PRADAN with confirmed NAC/WAC overlap (verify overlap in QuickMap first). Run `ingest_ch2.py`. **`LabelCornerBackend` meets a real OHRC label for the first time — expect and budget for iteration here.** |
| 16–17 | End-to-end on real **OHRC↔NAC**, then **TMC-2↔NAC**. IIRS↔WAC via the TMC-2 chain last, and only if time allows. |
| 17–18 | Run the full ablation ladder A→H locally; Colab job for the RoMa v2 rungs. **Isolate the D→E delta as a single number.** |
| 18–19 | Generate the money plot. **This is the deliverable that decides the round.** |
| 19–20 | Tier 3: cycle closure across three overlapping products + NAC DTM ortho check |
| 20–21 | Freeze results. Every number in the PPT traced to a Tier-1 or Tier-2 artefact on disk (`CLAUDE.md` §4.2, §9). |

**Honest-reporting rule, non-negotiable:** if a rung shows no measurable gain, report it and cut the component (`CLAUDE.md` §4.2). No component survives for narrative reasons.

---

### Phase 5 — Deliverables · Days 21–25 · Opus + Amish

| Day | Work |
|---|---|
| 21–22 | PPT. Slide 2 is the three-challenge decomposition from `PROJECT_CONTEXT.md` §2. The money plot is the centre. The DEM-resolution limitation table from §5 Stage 2 goes in **as-is, not hidden** — and now the G1 georeferencing assumption is stated alongside it. |
| 22–23 | Demo video following §10, plus a live-run dry run on the M4 with the network off |
| 23–24 | Rehearse every §11 judge question. Add the three the review surfaced: *"is your DEM co-registered to your reference?"* (answer: measured, `render_ref_ncc` and `dem_ref_offset_px`), *"is your hold-out actually independent?"* (answer: spatially blocked, not random), *"your CH-2 products aren't map-projected — how did you georeference them?"* (answer: label-corner GCPs, stated assumption, residual is what the pipeline removes) |
| 24–25 | Buffer. Submit. |

---

## 9. Tool execution reference

Every command the build runs, in order.

### Environment
```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install numpy scipy rasterio pyproj shapely pds4-tools \
            opencv-contrib-python scikit-image scikit-learn numba \
            torch torchvision kornia matplotlib pyyaml tqdm streamlit pytest
gdalinfo --version                      # confirm GDAL via rasterio wheels
python -c "import torch; print(torch.backends.mps.is_available())"
```

### Data acquisition
```bash
python scripts/fetch_nac_dtm.py --site <SITE> --out data/ref/      # ortho GeoTIFF + DTM, no login
python scripts/fetch_dem.py --bbox <minlon,minlat,maxlon,maxlat> --source sldem2015 --out data/dem/
python scripts/fetch_wac.py --bbox <...> --out data/ref/
python scripts/ingest_ch2.py --zip <PRADAN_download.zip> --out data/raw/     # ISSDC account approved
python scripts/fetch_checkpoints.py --models eloftr,superglue                # never committed
```

### Development loop
```bash
pytest tests/ -x -q                                   # after every subagent merge
pytest tests/test_relight.py -v                       # critical path
python -m lucid.pipeline --config configs/tmc2_nac.yaml --source <src> --reference <ref> --out results/run_001
gdalinfo results/run_001/registered_image.tif
gdalwarp -tps -r bilinear results/run_001/... check.tif   # F2 done-test, must pass first try
```

### Experiments
```bash
python experiments/ablation.py    --config configs/default.yaml --out results/ablation/
python experiments/money_plot.py  --scenes 5 --delta-el 0:60:5 --methods raw_sg,clahe_sg,lucid --out results/money/
python experiments/runtime_bench.py --config configs/ohrc_nac.yaml
```

### Colab / Kaggle — RoMa v2 arm only
```bash
pip install romav2                                    # weights auto-download; MIT code, DINOv3 licence noted
python experiments/ablation.py --config configs/colab_romav2.yaml --rungs D,E --out /content/results/
```

### Demo
```bash
streamlit run app/streamlit_app.py
```

---

## 10. Verification — how we know it works

### Per-module (blocks merge)
`CLAUDE.md` §5 definitions of done, plus the review's additions: M2 exact against brute-force ray-march · M3 round-trip within 2% RMS · `render_ref_ncc` reported on every real pair · `metrics.json` schema test rejecting any unlabelled `rmse` key (G9).

### End-to-end
1. **Synthetic** — inject a known 0.37 px shift and a known 30° Δ solar elevation into a NAC DTM ortho; the pipeline recovers the shift within 0.05 px.
2. **Real, no ground truth** — real TMC-2↔NAC pair; report `rmse_fit_px` and blocked `rmse_holdout_px` side by side; hand `gcps.txt` to `gdalwarp -tps` and confirm it runs unmodified.
3. **Independent** — cycle closure across three overlapping products; loop residual reported with no ground truth involved.
4. **The claim itself** — the money plot. Curve 3 stays flat past 30° Δ solar elevation where curves 1 and 2 collapse. **If this figure does not materialise, the central claim is not supported and we say so.**

### The six success criteria
`CLAUDE.md` §10 is the acceptance test. Anything not serving one of those six is not a priority.

---

## 11. Risk register

| Risk | Likelihood | Mitigation, already in the plan |
|---|---|---|
| `LabelCornerBackend` residual too large on real OHRC | Medium | Coarse pyramid stage sized from M6's prior; Phase 4 budgets iteration days; ISIS backend interface already stubbed |
| SLDEM streak artifacts corrupt the render | Medium | Render used only for coarse initialisation; gated on `render_ref_ncc`; falls back to intensity matching (`PROJECT_CONTEXT.md` §11) |
| OHRC 10–20× DEM/GSD ratio limits relighting | **Known and accepted** | Documented in the §5 Stage 2 table. OHRC uses relighting for coarse alignment only, then fine-matches the real pair. **Show the table, do not hide it.** |
| EfficientLoFTR underperforms SuperGlue on lunar imagery | Low | Rung C isolates it; if SuperGlue wins we use SuperGlue and say so — D→E is the claim, not the matcher choice |
| Money plot does not separate the curves | Low, high impact | Detected at Day 18–19 with 6 days of buffer; the honest fallback is the Tier-1/Tier-2 accuracy table plus the measured D→E delta |
| Runtime misses the 10-minute target | Medium | `runtime_bench.py` from Batch 3; tile budget and pyramid depth are config keys, tunable without code change |

---

## 12. What this plan does not build

Unchanged from `CLAUDE.md` §7 and `PROJECT_CONTEXT.md` §8.1. **CUT:** Fourier-Mellin · PC-SIFT · LightGlue · DINOv2 as a separate stage · MLflow · W&B · Docker · FastAPI · homography as the primary model · learned/classical match fusion beyond simple fallback. **DEFER (M12, M13):** matcher fine-tuning on rendered lunar data · full Hapke BRDF · IIRS full-cube band chaining · DFSAR extension.

If a subagent proposes any of these, Opus rejects and cites the section.

---

## 13. Anti-drift check — run before every merge

From `CLAUDE.md` §9, with one addition from this review:

- Does the innovation still work — can we still show the relit pair and the Δ-solar-elevation plot?
- Is any accuracy claim untraceable to a Tier-1 or Tier-2 artefact on disk?
- Has scope crept toward §12?
- **Is the D→E delta still isolatable?** That single ablation step is the entire project.
- Would an ISRO/SAC judge recognise this as derivative of arXiv:2509.04775?
- **★ New:** is every gap G1–G10 still resolved, or has a refactor quietly reintroduced one? The three most fragile are **G2** (validity mask silently dropped), **G6** (someone "simplifies" blocked CV back to a random split), and **G7** (an extra resample creeping into the LSM path).

---

## 14. Ownership

Per `CLAUDE.md` §0 and §8: **Opus** decides architecture and scope, writes contracts, reviews every subagent output before merge, maintains `PROGRESS.md`, and never writes production code. **Sonnet subagents** write all implementation, maximum 3 in parallel, never two on the same file, each against a written contract with a test that defines done. Genuinely ambiguous architecture and scope decisions go to **Amish**, not guessed.

---

## 15. Sources consulted during the review

The gaps in §2 were not found by reading the design documents alone — G1, G3 and part of G8 came from verifying the external dependencies the design leans on. Recorded here so a teammate can re-check them rather than take them on trust.

| Claim in §2 | Verified against |
|---|---|
| RoMa v2 exists, weights auto-download, code is MIT but the DINOv3 backbone carries its own licence | [github.com/Parskatt/RoMaV2](https://github.com/Parskatt/RoMaV2) · [pypi.org/project/romav2](https://pypi.org/project/romav2/) · [arXiv:2511.15706](https://arxiv.org/abs/2511.15706) |
| **G3** — RoMa v2's refinement stage uses a custom CUDA kernel, so it will not run on Apple MPS | Same repository and paper (decoupled two-stage matching-then-refinement with a custom CUDA kernel) |
| **G1** — Chandrayaan-2 calibrated products are line-scanner geometry; exact map projection needs `isisimport → spiceinit → mapproject` with ~200 GB of SPICE kernels | [Ames Stereo Pipeline — Chandrayaan-2](https://stereopipeline.readthedocs.io/en/latest/examples/chandrayaan2.html) |
| LRO NAC EDRs likewise need `lronac2isis → spiceinit → lronaccal → lronacecho → cam2map` before they are georeferenced | [LROC NAC Processing Guide (ASU)](https://www.lroc.asu.edu/files/DOCS/LROC_NAC_Processing_Guide.pdf) · [USGS ISIS lronac2isis](https://isis.astrogeology.usgs.gov/9.0.0/Application/presentation/Tabbed/lronac2isis/lronac2isis.html) |
| **The Phase 0 unlock** — LROC NAC DTM products ship orthorectified NAC images *and* the DTM as GeoTIFF in the same frame, no login and no ISIS required | [PDS LROC NAC_DTM README](https://pds.lroc.asu.edu/data/LRO-L-LROC-5-RDR-V1.0/LROLRC_2001/DATA/SDP/NAC_DTM/MRECRISIUM1/NAC_DTM_MRECRISIUM1_README.TXT) · [LROC RDR browse](https://data.lroc.im-ldi.com/lroc/view_rdr/SHAPEFILE_NAC_DTMS) |
| SLDEM2015 tiles are directly downloadable (512 ppd, ~59 m/px, ±60° lat, 30° lat × 45° lon tiles) | [PDS Geosciences SLDEM2015](https://pds-geosciences.wustl.edu/lro/lro-l-lola-3-rdr-v1/lrolol_1xxx/data/sldem2015/) · [imbrium.mit.edu](https://imbrium.mit.edu/) · [PGDA product 54](https://pgda.gsfc.nasa.gov/products/54) |
| PDS4 labels are readable from Python without ISIS | [pds4_tools](https://github.com/Small-Bodies-Node/pds4_tools) · [user manual](https://pdssbn.astro.umd.edu/tools/pds4_tools_docs/current/user_manual.html) |
| CH-2 optical payloads (TMC-2, OHRC, IIRS) use PDS4 with a mission Local Data Dictionary; OHRC ships raw + calibrated only, TMC-2 also ships derived products | [LPSC 2023 abstract 1041](https://www.hou.usra.edu/meetings/lpsc2023/pdf/1041.pdf) · [PRADAN CH-2 FAQ](https://pradan.issdc.gov.in/ch2/faq.xhtml) |

**Link corrections carried forward from `DATASET_STATUS.md`:** the PS's LROC download URL `lroc.im.-ldi.com` is a typo for `lroc.im-ldi.com`; `quickmap.lroc.im-ldi.com` works as given; no SELENE URL was supplied by the PS and none has been invented here.
