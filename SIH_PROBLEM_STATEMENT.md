# SIH 2026 — Official Problem Statement (as given on the portal)

**This file reproduces the problem statement verbatim, exactly as provided by the user from the SIH portal, with no interpretation added.** All analysis, reasoning, and proposed solution live in `PROJECT_CONTEXT.md`. This file exists so the raw source text is never lost, edited, or paraphrased away as the project evolves.

---

## Title

**Multi-modal, Sun angle and scale invariant image correspondence using Chandrayaan-2 optical images (OHRC, TMC and IIRS)**

---

## Background

> Image Registration is the process of aligning two or more images of the same scene taken at different times, from different viewpoints, or by different sensors into a common coordinate system. It has two main components:
> - **Source Image (Moving):** The image that is to be geometrically transformed to align with the reference image.
> - **Reference Image (Fixed):** The target image about which source image is to be geometrically transformed.

---

## Description

> The process of lunar images registration involves finding match points between source and reference image and then aligning the source image with the reference image. The key challenges involved in this process are as follows:
>
> - **Illumination variation:** Illumination variation refers to changes in sun azimuth and elevation effect on the surface lighting conditions that affect the appearance of the lunar surface features which is hard to correlate.
> - **Viewpoint variation:** It refers to geometric distortions caused by different camera positions/orientations capturing the same scene. Objects appear shifted, scaled, rotated, or perspective-distorted depending on observing angle.
> - **Scale Variation:** Lunar imaging missions operate at vastly different altitudes and at different spatial resolutions. This creates scale ratios.

---

## Expected Solution

> Generic software solution for finding correspondence between Chandrayaan-2 acquired optical images and Lunar reference images with a sub-pixel accuracy of source image maintaining uniform distribution across the images.
>
> - Software and registered product with corresponding match points.
> - Evaluation metric (e.g. RMSE, inlier match count, inlier ratio, etc.)

---

## Administrative details

| Field | Value |
|---|---|
| Organization | Indian Space Research Organisation (ISRO) |
| Department | Department of Space / Indian Space Research Organisation |
| Category | Software |
| Theme | Space Technology |

---

## Datasets (as given)

> Specific datasets link will be provided — **TBD**
>
> - **Chandrayaan-2 orbiter optical payload:** OHRC, TMC-2, IIRS lunar images. Link: `https://chmapbrowse.issdc.gov.in/`
> - **Reference:** LRO NAC Images (Lunar Reconnaissance Orbiter Narrow Angle Camera). Links: `https://lroc.im.-ldi.com/images/downloads/`, `https://quickmap.lroc.im-ldi.com/`
> - SELENE Images *(no link given in the source text)*

**Note on the line "Specific datasets link will be provided – TBD":** the portal text itself says the exact curated dataset for evaluation has **not** been finalized/released yet. The three sources above are general public archive/browse portals, not necessarily the exact files the judges will test against. See `DATASET_STATUS.md` for how the project handles this.

---

## Verbatim source text (unedited, as pasted by the user)

```
Multi-modal, Sun angle and scale invariant image correspondence using Chandrayaan-2 optical images (OHRC, TMC and IIRS)
this is the problem statement for SIH 2026.

Background
Image Registration is the process of aligning two or more images of the same scene taken at different times, from different viewpoints, or by different sensors into a common coordinate system. It has two main components:
• Source Image (Moving): The image that is to be geometrically transformed to align with the reference image.
• Reference Image (Fixed): The target image about which source image is to be geometrically transformed.

Description
The process of lunar images registration involves finding match points between source and reference image and then aligning the source image with the reference image. The key challenges involved in this process are as follows:
• Illumination variation: Illumination variation refers to changes in sun azimuth and elevation effect on the surface lighting conditions that affect the appearance of the lunar surface features which is hard to correlate.
• Viewpoint variation: It refers to geometric distortions caused by different camera positions/orientations capturing the same scene. Objects appear shifted, scaled, rotated, or perspective-distorted depending on observing angle.
• Scale Variation: Lunar imaging missions operate at vastly different altitudes and at different spatial resolutions. This creates scale ratios.

Expected Solution
Generic software solution for finding correspondence between Chandrayaan-2 acquired optical images and Lunar reference images with a sub-pixel accuracy of source image maintaining uniform distribution across the images.
• Software and registered product with corresponding match points.
• Evaluation metric (eg. RMSE, inlier match count, inlier ratio, etc.)

Organization: Indian Space Research Organisation (ISRO)
Department: Department of Space / Indian Space Research Organisation
Category: Software
Theme: Space Technology

Specific datasets link will be provided - TBD
• Chandrayaan-2 orbiter optical payload: OHRC, TMC-2, IIRS lunar images. (Link: https://chmapbrowse.issdc.gov.in/)
• Reference: LRO NAC Images (Lunar Reconnaissance Orbiter Narrow Angle Camera) (Link: https://lroc.im.-ldi.com/images/downloads/ , https://quickmap.lroc.im-ldi.com/), SELENE Images
```

---

## Traceability — every clause mapped to where it's handled

Use this table to verify nothing in the official text has been dropped or silently reinterpreted.

| Exact clause from PS | Handled in `PROJECT_CONTEXT.md` |
|---|---|
| "aligning two or more images ... into a common coordinate system" | §1, §5 Stage 1 |
| Source/moving vs reference/fixed definitions | §5 Stage 1 (`AlignedPair`) |
| "finding match points between source and reference" | §5 Stage 3 |
| "Illumination variation ... sun azimuth and elevation" | §3, §5 Stage 2 — the core innovation |
| "Viewpoint variation ... shifted, scaled, rotated, or perspective-distorted" | §2, §5 Stage 1 (map projection) + Stage 4 (local model) |
| "Scale Variation ... vastly different altitudes ... spatial resolutions" | §2, §5 Stage 1 (common GSD) |
| "Generic software solution" | §5 fallback/blind mode in Stage 1; regime routing in Stage 3 |
| "sub-pixel accuracy" | §5 Stage 6 (LSM) |
| "uniform distribution across the images" | §5 Stage 5 |
| "registered product with corresponding match points" | §5 Stage 7 outputs |
| "Evaluation metric (RMSE, inlier match count, inlier ratio, etc.)" | §5 Stage 8, §6 metrics.json |
| OHRC, TMC-2, IIRS as source | §5 Stage 3 regime routing table |
| LRO NAC, SELENE as reference | §5 Stage 3 regime routing table, §12 |

Nothing in the official text has been left unaddressed. Where the PS is silent (e.g. it does not name a specific transform model, a specific matcher, or a specific evaluation split), `PROJECT_CONTEXT.md` states the decision made and the reasoning.
