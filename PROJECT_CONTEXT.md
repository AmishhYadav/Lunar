# LUCID — Lunar Correspondence by Illumination Decoupling

**SIH 2026 · ISRO · Problem Statement: Multi-modal, Sun-angle and Scale-invariant Image Correspondence using Chandrayaan-2 Optical Images (OHRC, TMC-2, IIRS)**

> **One-line pitch:** We don't teach the matcher to ignore the Sun — we put the Sun in the same place first.

---

## 0. Document purpose

This is the single source of truth for the project. It states what we are building, why each decision was made, what we deliberately rejected, how we will measure success, and what we will *not* do. Anyone (human or agent) joining the project reads this file first.

Status legend used throughout:
- **[BUILD]** — must exist and work for the internal round
- **[DEFER]** — promised in the PPT, built only if we reach the Grand Finale
- **[CUT]** — explicitly out of scope, do not implement

---

## 1. The problem, stated plainly

ISRO has Chandrayaan-2 imagery. Each image has an approximate location on the Moon from spacecraft telemetry, but not an exact one. They need it pinned onto a trusted reference map (LRO NAC / SELENE TC / LROC WAC) with sub-pixel accuracy, with match points spread evenly across the frame, plus quantitative quality metrics.

Why it matters downstream: without accurate co-registration you cannot build seamless mosaics, cannot compare an OHRC image from 2020 against one from 2023 for change detection, and cannot overlay IIRS mineral spectra onto high-resolution terrain. Every derived product inherits the alignment error.

### 1.1 Required deliverables (directly from the PS)

| PS requirement | Where we satisfy it |
|---|---|
| Generic software solution for correspondence | Stage 1–6 pipeline, CLI + demo UI |
| Sub-pixel accuracy | Stage 6 (affine-photometric least-squares matching) |
| Uniform distribution of match points | Stage 5 (bucketed selection with quality floor) |
| Registered product | Stage 7 outputs (warped raster + transform) |
| Corresponding match points | `correspondence_pairs.csv`, GDAL GCP export |
| Evaluation metric (RMSE, inlier count, inlier ratio, etc.) | Stage 8 three-tier evaluation protocol |
| Handles illumination variation | **Stage 2 — the innovation** |
| Handles viewpoint variation | Stage 1 (map projection) + Stage 4 (local model) |
| Handles scale variation | Stage 1 (common GSD by construction) |

---

## 2. The core insight

**The three named challenges are not the same kind of problem. Treating them as one problem is why every existing pipeline looks identical.**

| Challenge | What it actually is | How we solve it |
|---|---|---|
| **Scale variation** | Bookkeeping. Both products carry selenographic coordinates and GSD in PDS4 metadata. | Reproject both to a common map projection at a common GSD. Scale ratio becomes exactly 1.0. Solved analytically. |
| **Viewpoint variation** | Mostly bookkeeping, partly physics. Map projection removes the global geometric difference. What survives is terrain parallax — a rim 500 m above the plain viewed at 20° vs 5° emission angle shifts by ~200 m. | Map projection + a *local* transform model. This residual is terrain-dependent, which is precisely why a single global homography can never fix it. |
| **Illumination variation** | Pure physics. Metadata cannot save you. Same crater at low vs high Sun looks like two different objects; shadows flip sides, bright rims go dark. | **Physically remove it by relighting.** This is where all our innovation goes. |

Put this decomposition on slide 2 of the PPT. It demonstrates that we understood the problem rather than pattern-matching to "image registration = features + RANSAC."

---

## 3. The innovation (our USP)

### 3.1 What everyone else does

Every published approach — including ISRO's own baseline — tries to build a **description of the terrain that is invariant to lighting**: CLAHE, histogram matching, shadow normalization, gradient features, RIFT2's radiation-insensitive descriptors, SuperGlue's learned robustness.

### 3.2 What we do instead

We **remove** the illumination difference before matching.

The reasoning chain:
1. We know the shape of the terrain — LOLA, SLDEM2015, CH-2 TMC DEM.
2. We know exactly where the Sun was for each image — it is in the metadata (sun azimuth, sun elevation).
3. Given shape + Sun position + a reflectance model, computing what the surface *should* look like is a solved physics problem.
4. Therefore: render the reference terrain under the Chandrayaan-2 image's **exact** solar geometry.

Now both images have shadows falling the same way. The matcher's job collapses from *"recognize this crater despite the lighting"* to *"find the shift between two nearly identical pictures."*

**Analogy for judges:** everyone else is building a translator that compares an English document against a Hindi one. We translate both into the same language first.

### 3.3 Why this is defensible as novel

Physically-based rendering for lunar image matching is established in **terrain-relative navigation for landers**:
- A 2026 two-stage lunar polar landing system renders LROC NAC + LOLA DEM at 0.95°–1.75° solar elevation and achieves NCC matching peaks exceeding 0.35 in all cases, up to 0.69.
- On Mars, **MARTIAN** pretrains LoFTR on Blender-rendered HiRISE terrain; removing that rendered pretraining costs 11.2% Acc@5m overall and up to 26.4% on hard terrain.
- On the Moon, **MoonAnything** (MMSys '26) renders real DEMs with the Hapke BRDF and shows fine-tuning on a restricted subset yields transferable, geometry-aware representations that generalize to unseen terrain (Tycho).

**Nobody has applied this to cross-sensor orbital co-registration of Chandrayaan-2 against LRO.** The published ISRO/SAC baseline on exactly these image pairs uses CLAHE, PCA, histogram matching and shadow correction — the invariance approach.

That is our gap, and it is a gap we can name out loud with citations. Far stronger than claiming vague novelty.

### 3.4 The second half of the innovation (costs nothing extra)

**The same renderer becomes the measuring stick.** Take one real image, apply a known sub-pixel shift and a known change of Sun angle → we now have exact ground truth per pixel.

One physics engine serves as both the method and the evaluation harness. This solves the single biggest risk in any registration project: the absence of ground truth.

---

## 4. Prior art we must beat and cite

### 4.1 The baseline written by our own reviewers

**Makharia, Singla, Amitabh, Dube, Sharma — "Comparative Evaluation of Traditional and Deep Learning Feature Matching Algorithms using Chandrayaan-2 Lunar Data" (arXiv 2509.04775, Sept 2025).**

Authors 2–4 are from **Signal and Image Processing Area, Space Applications Centre, ISRO, Ahmedabad**. This is almost certainly the group that authored our problem statement. Read this paper before writing any slide.

What they did: evaluated SIFT, ASIFT, AKAZE, RIFT2, SuperGlue on OHRC–NAC, IIRS–WAC, DFSAR–SELENE in equatorial and polar regions. Preprocessing: georeferencing, resolution resampling, intensity normalization, CLAHE, inversion, morphological dilation, PCA, histogram matching, shadow normalization, log transform.

Their reported results (OHRC–NAC):

| Algorithm | RMSE X (px) | RMSE Y (px) | Time (s) |
|---|---|---|---|
| SuperGlue (equatorial) | 0.6249 | 0.5718 | 3.809 |
| RIFT2 | 1.5033 | 1.1888 | 36.881 |
| ASIFT | 1.9946 | 1.6345 | 809.82 |
| SIFT | 3.6096 | 5.9558 | 678.20 |
| AKAZE | 3.1189 | 4.7096 | 737.17 |
| SuperGlue (polar) | 0.9234 | 0.7586 | 4.643 |

SIFT, ASIFT, AKAZE and RIFT2 **all failed entirely** on OHRC–NAC polar. Their conclusion: methods needing no explicit preprocessing outperform preprocessed classical methods on both accuracy and cost.

**Two exploitable weaknesses:**
1. **Their RMSE is fit residual, not accuracy.** They identify matching control points, apply the derived transform, and average the squared distances — on the same points used to fit the transform. Keep four points and it goes to zero.
2. **No performance-vs-solar-angle curve exists** for this data. The PS title says *Sun angle invariant*; ISRO put that word there because they know it is the failure mode. Nobody has published the curve.

### 4.2 Why we do not hand-build "DINOv2 → LoFTR"

That combination **is RoMa** (CVPR 2024): frozen pretrained DINOv2 coarse features combined with specialized ConvNet fine features forming a precisely localizable feature pyramid. It is already superseded — **RoMa v2** (arXiv 2511.15706, Nov 2025) upgrades to DINOv3, consistently outperforms all prior matchers on MegaDepth and ScanNet, beats MASt3R/DUSt3R/VGGT on the benchmark demanding accurate sub-pixel correspondence, at 1.7× RoMa's throughput.

Hand-assembling a worse version of a downloadable checkpoint is effort spent going backwards.

### 4.3 Other prior art that is NOT our innovation

| Component | Actually is | Treat as |
|---|---|---|
| Adaptive match grid | Bucketing / ANMS — standard in bundle adjustment since the 1990s | Requirement implemented correctly |
| Phase correlation + Fourier-Mellin | Reddy & Chatterji, 1996 | **[CUT]** — we know rotation/scale from metadata |
| Coarse-to-fine pyramid | 1980s | Standard practice |
| Illumination normalization via CLAHE/Retinex | Exactly the ISRO baseline's approach | Ablation baseline, not our method |

---

## 5. Pipeline — stage by stage

### Stage 1 — Metadata parsing and geodetic priming **[BUILD]**

**What:** Parse PDS4 XML labels for corner coordinates, sun azimuth, sun elevation, incidence/emission angle, GSD, sensor ID. Reproject source and reference into a common map projection at a common GSD.
- Equirectangular for |lat| < 60°
- Polar stereographic for |lat| ≥ 60°

**Why:** Scale ratio becomes 1.0 and rotation ≈ 0 **by construction**. The remaining unknown is a bounded translation of tens to a few hundred metres. This makes everything downstream tractable.

**Rejected — Fourier-Mellin:** it exists to recover rotation and scale blindly. We already know both from metadata. Keeping it costs compute and adds a failure mode to rediscover information sitting in a text file.

**Fallback (metadata absent):** the PS says "generic software solution," so degraded mode must exist. Downsample both heavily → run a dense matcher globally → recover a coarse similarity transform → rejoin the main pipeline at Stage 2. Documented degraded path, never the default.

**Outputs:** `AlignedPair` object — both rasters in a shared CRS/GSD, plus a normalized metadata schema, plus a search-radius prior in pixels.

---

### Stage 2 — Illumination equalization by relighting **[BUILD — THE INNOVATION]**

**What:** Fetch the DEM over the footprint. Render it twice with the **Lunar-Lambertian reflectance model** — once under the source's solar geometry, once under the reference's. Use the pair to transfer the reference into the source's lighting.

**Reflectance model:**
```
R_LL = A · [ 2·cos(θi) / (cos(θi) + cos(θe)) ] + (1 − A)·cos(θi)

cos(θi) = n̂ · l̂        (incidence)
cos(θe) = n̂ · v̂        (emission)
n̂ = (−p, −q, 1)ᵀ / sqrt(1 + p² + q²)
p = ∂z/∂x,  q = ∂z/∂y
```

**Why Lunar-Lambertian and not full Hapke:** it is the standard model for airless bodies, has one tunable albedo parameter, and is roughly twenty lines of code. Full Hapke requires parameters we would have to fit. For *matching* we need shadow and shading **structure** to be right, not absolute radiometry. Start simple; upgrade to Hapke only if ablation shows it helps.

**Cast shadows:** ray-march along the solar azimuth per pixel to compute a binary shadow mask. This is what makes the render actually look like the Moon, and it is the part CLAHE can never reproduce.

**Cross-question — isn't this circular? Doesn't the DEM need to be registered to the reference already?**
No. LOLA and SLDEM are tied to the LRO/LOLA geodetic frame — the same frame NAC and WAC reference products live in. DEM and reference share a coordinate system by construction. The only unknown is where Chandrayaan-2 was. LOLA is laser altimetry, so its geolocation does not depend on image matching at all.

**Cross-question — why keep the reference image at all? Why not match source directly to the render?**
This shapes the architecture. Matching source ↔ render is cleaner, because the render already lives in the reference frame, so a match to it *is* a match to the reference. But the render only contains detail the DEM contains. Therefore:

> **Match against the render for the COARSE stage. Match against the real reference for the FINE stage.**

The render solves the hard global search where illumination kills you; the real reference supplies fine detail once we are already close.

**The honest limitation — relighting quality is bounded by DEM resolution relative to image GSD:**

| Pair | Image GSD | Best available DEM | Ratio | Relighting works? |
|---|---|---|---|---|
| IIRS ↔ LROC WAC | 80 / 100 m | SLDEM2015 ~59 m | ~1× | **Yes, fully** |
| TMC-2 ↔ NAC / SELENE TC | 5 / 10 m | CH-2 TMC DEM 10 m; LOLA polar 5 m | ~1–2× | **Yes** |
| OHRC ↔ NAC | 0.25 / 0.5 m | NAC DTMs 2–5 m, select sites only | 10–20× | **Coarse stage only** |

For OHRC the render cannot reproduce the small craters carrying the fine matching signal. So OHRC uses relighting to nail coarse alignment, then does fine matching on the real pair — by which point we are within a few pixels, where local illumination differences matter far less.

**Do not hide this table. Show it.** Judges trust a team that knows exactly where its method stops working more than one claiming universal success.

**Optional cheap addition for the ablation:** since sun azimuth is known for both images, rotate image gradient fields into a **sun-relative frame** before matching. Handles the shading component for free. Does **not** handle shadow occlusion — present as an approximation, not a fix.

---

### Stage 3 — Correspondence **[BUILD]**

**What:** Run a modern dense matcher on tiles of the illumination-equalized pair.

| Branch | Model | Role |
|---|---|---|
| Primary | RoMa v2 (or EfficientLoFTR if compute-bound) | Our matcher |
| Baseline | SuperGlue + SuperPoint | ISRO's published best — we must beat it on its own terms |
| Control | RootSIFT | Classical interpretability floor |

**Tiling matters more than expected.** An OHRC strip in ISRO's own paper is **12000 × 90148 px = 1.08 gigapixels**. Tile with overlap; use the Stage 1 geodetic prior to only match tiles that actually overlap. Without the prior we would match every tile against every tile.

**Regime routing — never match IIRS directly to NAC.** Chain up the resolution ladder:
- OHRC (0.25 m) ↔ NAC (0.5–2 m)
- TMC-2 (5 m) ↔ NAC / SELENE TC (10 m)
- IIRS (80 m) ↔ LROC WAC (100 m) or a TMC-2-derived basemap

A 160:1 ratio across different physical signals is not a matching problem, it is a chaining problem.

**IIRS band selection:** use a reflectance-dominated **short SWIR** band. Avoid anything beyond ~3 µm where thermal emission dominates and the signal is no longer terrain shading. Register one band; apply the transform to the full cube.

---

### Stage 4 — Geometric verification and model selection **[BUILD]**

**What:** RANSAC to reject outliers, then fit a transform. Try in order: similarity → affine → 2nd-order polynomial → piecewise / thin-plate spline.

**Selection rule: cross-validated residual on held-out points, NOT fit residual.**

**Why this rule:** fit residual always improves as you add parameters — a TPS with enough control points fits noise perfectly. Cross-validated residual *rises* when overfitting starts, so it actually tells you when to stop. This is the concrete implementation of "simplest model that fits," which most proposals state and never operationalize.

**Why NOT a homography:** pushbroom sensors over curved, non-planar terrain do not produce projective relationships. A homography assumes a flat plane and a pinhole camera; neither holds. It is the wrong physics — and it is what the published baseline uses, so departing from it is a point we make explicitly.

---

### Stage 5 — Uniform match distribution **[BUILD]**

**What:** Divide the valid footprint into cells. Select the best points per cell subject to a **mandatory quality floor**. Report occupancy fraction and normalized entropy of the point distribution.

**Why we do NOT claim this as innovation:** bucketing and adaptive non-maximal suppression have been standard in bundle adjustment for 30 years. Claiming it invites a judge to say "that's just bucketing." Call it a requirement implemented correctly; let relighting be the innovation.

**Cross-question — does forcing uniformity hurt accuracy?**
Yes, if you force points into empty cells regardless of quality. So the quality floor is non-negotiable, and we report coverage and accuracy **together** with a tradeoff curve in the ablation. Showing we measured the tradeoff is worth more than pretending it does not exist.

---

### Stage 6 — Sub-pixel refinement **[BUILD]**

**What:** Affine-photometric **least-squares matching** (LSM, Gruen/Ackermann) per point. For each match, solve a small optimization for the continuous shift, local affine warp, and brightness/contrast scaling that makes the two patches identical.

**Why LSM and not parabola-fitting the correlation peak:** peak-fitting suffers **peak-locking** — estimates get pulled toward integer pixel positions. That is fatal when sub-pixel accuracy is the entire claim. LSM also solves brightness and contrast as free parameters, exactly what is needed across sensors.

**The free bonus:** LSM's normal equations yield a **covariance per point**. That is a real, derived error bar — it is what turns the uncertainty map from decoration into something meaningful. Never compute uncertainty by hand-weighting confidence scores.

**Sanity check:** cross-validate against `skimage.registration.phase_cross_correlation(upsample_factor=100)` on synthetic shifts.

---

### Stage 7 — Warping and product generation **[BUILD]**

Outputs (one directory per run):
```
result/
├── registered_image.tif          # source warped into reference frame
├── correspondence_pairs.csv      # src_x, src_y, ref_x, ref_y, conf, sigma_x, sigma_y
├── source_points.csv
├── reference_points.csv
├── gcps.txt                      # GDAL-compatible GCP file  ← see below
├── transform.json                # model type + parameters + selection evidence
├── metrics.json                  # all metrics, all three evaluation tiers
├── confidence_map.tif
├── residual_map.tif
├── uncertainty_map.tif           # derived from LSM covariance
├── match_overlay.png
├── registration_overlay.png
└── run_config.yaml               # full reproducibility record
```

**`gcps.txt` is the detail that converts a demo into a tool.** Export match points in GDAL GCP format so an ISRO analyst can run `gdalwarp -tps` themselves. Nobody else will do this.

---

### Stage 8 — Evaluation protocol **[BUILD — CRITICAL]**

Three tiers, **all reported side by side**.

| Tier | Method | What it proves |
|---|---|---|
| **1. Synthetic ground truth** | Apply a known sub-pixel shift + known relighting to a real image. Exact truth per pixel. | True accuracy; measures our sub-pixel resolution floor |
| **2. Held-out RMSE** | Fit transform on 70% of inliers, report residual on the other 30%. **Report fit residual next to it.** | Honest accuracy; makes the baseline's methodological gap visible without attacking anyone |
| **3. Independent geodetic check** | Compare against LOLA altimetry crossovers | Absolute, external validation |

Plus the PS-mandated set: inlier count, inlier ratio, coverage fraction, spatial uniformity (entropy), runtime, peak GPU memory, failure rate.

**Reporting rule:** every RMSE number in the PPT is labelled either `RMSE_fit` or `RMSE_holdout`. Never an unlabelled "RMSE."

---

## 6. The demo that decides the round

**One plot.**

- **X axis:** Δ solar elevation between source and reference (0° → 60°)
- **Y axis:** registration success rate and true RMSE (from Tier-1 synthetic GT)
- **Three curves:**
  1. Raw intensity + SuperGlue *(the published ISRO baseline)*
  2. CLAHE + SuperGlue *(their preprocessing)*
  3. **Our relit pipeline**

Generate the illumination sweep **synthetically** so we are not dependent on hunting for real multi-date pairs.

If curve 3 stays flat past 30° Δ solar elevation where 1 and 2 collapse, that single figure is the entire pitch.

**Why this framing wins:** we are not claiming "our RMSE is lower." We are claiming *"everyone's method works on easy pairs; we work on the pairs the PS was actually written about."* The PS title says **Sun angle invariant**. No published paper shows a performance-vs-solar-angle curve for this data.

---

## 7. Ablation ladder

Each rung must produce a measurable delta in robustness, accuracy, coverage, runtime, or failure rate. If a rung shows no gain, we say so and cut the component.

| ID | Configuration |
|---|---|
| A | RootSIFT + RANSAC (classical floor) |
| B | CLAHE + SuperGlue + RANSAC (**= ISRO published baseline**) |
| C | Geodetic priming + SuperGlue + RANSAC |
| D | Geodetic priming + RoMa v2 + RANSAC |
| E | D + **relighting** ← the innovation isolated |
| F | E + bucketed uniform selection |
| G | F + LSM sub-pixel refinement |
| H | G + cross-validated model selection (full system) |

The delta **D → E** is the entire project. Present it as a single number and a single plot.

---

## 8. Scope control

### 8.1 CUT — do not implement, do not mention in the PPT

Fourier-Mellin · PC-SIFT · LightGlue · DINOv2 as a separate stage · MLflow · Weights & Biases · Docker · homography as the primary transform model · any "match fusion" between learned and classical branches beyond simple fallback.

Each is scope we cannot defend and do not need.

### 8.2 DEFER — promise in the PPT, build only at the Grand Finale

- Fine-tuning a matcher on rendered lunar data (the natural extension; validated by MARTIAN and MoonAnything)
- Full IIRS hyperspectral band-chaining across the cube
- DFSAR / radar extension
- Full Hapke BRDF with fitted parameters
- FastAPI service layer

### 8.3 Non-negotiable quality floors

- Quality threshold in Stage 5 always overrides coverage quotas
- Every accuracy claim traceable to Tier-1 or Tier-2 evaluation
- Low-confidence registrations are **flagged**, never silently returned

---

## 9. Three-week build plan

Portal deadline **30 September 2026**. Internal round before that. Team of 6 (min. one female member, per SIH rules).

| Days | Task | Owner |
|---|---|---|
| 1–2 | PDS4 parser, reprojection to common map frame, tiling | A |
| 2–4 | Lunar-Lambertian renderer, DEM fetch, cast shadows, relighting module | B |
| 3–5 | Matcher wrappers (RoMa v2 / SuperGlue / RootSIFT), RANSAC, bucketing | C |
| 5–7 | LSM sub-pixel, covariance, uncertainty map | D |
| 6–9 | Synthetic GT harness, illumination sweep, all metrics | E |
| 9–12 | Streamlit demo, result package, GCP export | F |
| 12–16 | Ablations, the money plot, PPT and video | All |

**Critical-path dependency:** E (evaluation harness) depends on B (renderer). Build B first and stub the rest. Without the renderer we have neither the method nor the ground truth.

---

## 10. Demo flow (for the video and live presentation)

1. Select source (OHRC / TMC-2 / IIRS) + reference image
2. Show parsed metadata — sun az/el, GSD, footprint
3. Show both reprojected into common map frame → **"scale problem, solved analytically"**
4. Show the DEM over the footprint
5. Show the render under reference sun geometry vs render under source sun geometry — side by side → **"this is the illumination difference, made explicit"**
6. Show the relit reference next to the source → **"now they look like the same picture"**
7. Show correspondences, RANSAC inliers vs outliers
8. Show bucketed uniform selection with coverage/entropy numbers
9. Show sub-pixel refinement before/after with per-point error ellipses
10. Show final registered overlay, difference image
11. Show `metrics.json` — RMSE_fit **and** RMSE_holdout side by side
12. **Show the money plot**
13. Export result package + `gcps.txt`

---

## 11. Anticipated judge questions and our answers

**"SuperGlue already gets 0.6 px. What's left?"**
That number is fit residual computed on the same points used to fit the transform, on one hand-picked equatorial pair. It is not accuracy. And there is no published curve of performance versus solar-angle difference. The PS asks for Sun-angle invariance, which means ISRO knows the hard cases are elsewhere.

**"Rendering-based matching is old news in terrain-relative navigation."**
Correct — for descent navigation. It has not been applied to cross-sensor orbital co-registration of Chandrayaan-2 against LRO, where the published baseline uses CLAHE and histogram matching. Our claim is the domain transfer plus the evaluation protocol, and we state that explicitly rather than overclaiming.

**"What if the DEM has artifacts?"**
SLDEM has known streak artifacts from LOLA ground tracks. We use the render only for coarse initialization, gate on render-match confidence, and fall back to intensity matching when confidence is low.

**"Which IIRS band?"**
A reflectance-dominated short SWIR band. Avoid beyond ~3 µm where thermal emission dominates and the signal is no longer terrain shading. Register one band, apply the transform to the full cube.

**"Why not just use a homography like everyone else?"**
Because pushbroom sensors over curved, non-planar terrain do not produce projective relationships. We select the transform model by cross-validated residual instead of assuming one.

**"What if there is no DEM for a region?"**
The pipeline degrades gracefully to Stage 1 priming + matcher + LSM, which is still a correct system. We report which mode was used in `metrics.json`.

---

## 12. Data sources

**See `DATASET_STATUS.md` for the full audit of the links given in the problem statement, including one that is broken as published, and the dataset-agnostic adapter layer this requires.** Summary:

| Data | Source | Notes |
|---|---|---|
| OHRC, TMC-2, IIRS | `https://chmapbrowse.issdc.gov.in/` (PS-given link) — requires ISSDC registration/login before download | PDS4 products, XML labels |
| LRO NAC | `https://lroc.im-ldi.com/images/downloads` (**corrected** — the PS text has a typo, see `DATASET_STATUS.md`); browse via `https://quickmap.lroc.im-ldi.com/` (PS-given, verified working) | 0.5–2 m/px |
| LROC WAC global mosaic | LROC curated downloads | ~100 m/px |
| SELENE TC / MI | JAXA Kaguya (SELENE) data archive — **no link given in the PS**; sourced independently | ~10 m/px |
| SLDEM2015 | Barker et al. 2016, via LROC | 512 ppd, ~59 m/px, ±60° lat |
| LOLA polar DEM | NASA PDS Geosciences Node | 5 m/px |
| CH-2 TMC DEM | Suresh et al. 2023, via PRADAN | 10 m/px |
| NAC DTMs | LROC curated downloads | 2–5 m/px, select sites only |

**The PS explicitly states "Specific datasets link will be provided – TBD."** The links above are general public archives, not confirmed to be the exact evaluation set. `src/lucid/io/` is built against the PDS4 label schema, not against any specific file list, so it does not need to change when the official dataset drops.

---

## 13. Key references

1. Makharia, Singla, Amitabh, Dube, Sharma — *Comparative Evaluation of Traditional and Deep Learning Feature Matching Algorithms using Chandrayaan-2 Lunar Data*, arXiv:2509.04775 (2025). **[SAC/ISRO — the baseline to beat]**
2. Makharia et al. — *Image Registration of High Resolution Chandrayaan-2 Data*, InGARSS 2024.
3. Edstedt et al. — *RoMa: Robust Dense Feature Matching*, CVPR 2024, arXiv:2305.15404.
4. Edstedt et al. — *RoMa v2: Harder Better Faster Denser Feature Matching*, arXiv:2511.15706 (2025).
5. Grethen et al. — *MoonAnything: A Vision Benchmark with Large-Scale Lunar Supervised Data*, MMSys '26, arXiv:2604.00682. Repo: `github.com/clementinegrethen/MoonAnything`
6. *MARTIAN: A Rendering Framework for Aerial Mars Imagery from HiRISE Orbital Data*, arXiv:2605.29647.
7. *Geometry-aided Vision-based Localization of Future Mars Helicopters in Challenging Illumination Conditions*, arXiv:2502.09795.
8. *A two-stage offline-online visual navigation framework for high-precision lunar polar landing*, Chinese Journal of Aeronautics (2026).
9. *DEM Refinement and Validation on the Lunar Surface Using Shape-from-Shading with Chandrayaan-2 OHRC Imagery*, arXiv:2604.17436.
10. Kumar, Kaushal, Murthy — *MoonMetaSync: Lunar Image Registration Analysis*, IEEE WNYISPW 2024, arXiv:2410.11118.
11. Gruen, A. — *Adaptive Least Squares Correlation*, 1985. **[the sub-pixel method]**
12. Reddy & Chatterji — *An FFT-Based Technique for Translation, Rotation and Scale-Invariant Image Registration*, 1996. **[what we deliberately do not use]**

---

## 14. The pitch, compressed

> Chandrayaan-2 images and LRO reference images of the same crater look completely different because the Sun was somewhere else. Everyone else builds features that try to *ignore* the Sun. We know the terrain shape from LOLA, and we know exactly where the Sun was from the metadata — so we re-render the reference under the Chandrayaan-2 image's exact solar geometry and make the two images look the same before matching. The same renderer gives us ground truth for free, which lets us report the first performance-versus-solar-angle curve for this data. **We don't teach the matcher to ignore the Sun — we put the Sun in the same place first.**
