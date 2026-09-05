# Dataset status — audit and handling plan

Direct answer to "have we addressed the dataset link": **partially, until this file.** The portal gave three link lines. One resolves fine, one is malformed as published, and one is entirely missing a URL. Below is the exact state of each, verified by fetching them, plus what the project does about the fact that the PS itself says the real dataset is not final yet.

---

## 1. What the PS actually says about the dataset

> "Specific datasets link will be provided — TBD"

Read this literally: **the curated evaluation dataset does not exist yet as far as the public PS text is concerned.** The three links given are pointers to the general public archives the source data comes from, not a confirmed fixed file list. This has one direct engineering consequence, stated in §2 below.

---

## 2. Link-by-link audit

### 2.1 Chandrayaan-2 source imagery (OHRC, TMC-2, IIRS)

**Given:** `https://chmapbrowse.issdc.gov.in/`

**Status: resolves, and it is a browse tool, not a download portal.** Fetched directly — it is the "Chandrayaan Data Explorer," built by SAC-ISRO, and it states outright:

> "Users are required to register and login for Chandrayaan-2 Orbiter imaging payload data download."

Login goes through `https://chmapbrowse.issdc.gov.in/server/`. It lists TMC (5 m, 0.5–0.8 µm), IIRS (80 m, 0.8–5.0 µm, ~20 nm spectral resolution), OHRC (28 cm), and DFSAR (2–75 m) — note the PS text says OHRC is "high resolution," and the portal itself states 28 cm, slightly different from the commonly cited 25–30 cm figure used across the literature we surveyed. Use the portal's own 28 cm figure in the PPT.

**Action item for the team, not something Claude Code can do:** someone needs to register an ISSDC account now. This is a manual, human, identity-verified step — budget a day for approval turnaround, don't leave it for the week before the deadline.

**Companion archive, not named in the PS but the actual bulk-download endpoint:** PRADAN, `https://pradan.issdc.gov.in/ch2/`. Same registration wall. Both point at the same underlying PDS4 archive; chmapbrowse is footprint/map-based discovery, PRADAN is the bulk product browser. Use chmapbrowse to find footprints, PRADAN to pull the files.

### 2.2 LRO NAC reference imagery

**Given:** `https://lroc.im.-ldi.com/images/downloads/` and `https://quickmap.lroc.im-ldi.com/`

**Status: the first one is broken as typed. It is a typo on the SIH portal.**

Verified by search: the real LROC domain is `lroc.im-ldi.com` — a single hyphen, no period before it. The PS text inserts a stray `.` producing `im.-ldi.com`, which is not a resolvable domain. The correct URL is:

```
https://lroc.im-ldi.com/images/downloads
```

This corrected page is the LROC "Curated Downloads" page — WAC global mosaics, illumination maps, DTMs, site maps, all directly downloadable, no login required.

The second link, `https://quickmap.lroc.im-ldi.com/`, was fetched directly and **works exactly as given** — it is LROC QuickMap, the interactive 2D/3D browse tool built by Applied Coherent Technology for the LROC team at ASU. This is where you visually confirm footprint overlap between a Chandrayaan-2 product and available NAC coverage before downloading anything.

**Do not silently "fix" this in front of the judges by just using the working link — mention in your submission or Q&A that you caught and corrected a typo in the official dataset link if asked.** It signals attention to detail, and it protects you if a teammate later hits the dead link and assumes their setup is broken.

### 2.3 SELENE imagery

**Given:** named as "SELENE Images," **no URL provided in the PS text.**

**Status: unaddressed by the portal, addressed by us.** SELENE (Kaguya) Terrain Camera and Multiband Imager data are distributed through JAXA's Kaguya (SELENE) Data Archive. Since the PS gives no link, do not guess at one and hardcode it into the pipeline. Treat SELENE as a same-schema alternate reference source, wired through the same PDS-style metadata adapter as NAC/WAC, and confirm the exact access point once (or if) the official TBD dataset link names it.

---

## 3. What this means for the pipeline — the actual engineering decision

Because the real dataset is TBD, **the pipeline must not be written against specific filenames, a specific footprint, or a specific product ID.** Concretely, in `CLAUDE.md` terms:

- `src/lucid/io/` parses **the PDS4 label schema**, not a fixed manifest. Any product from PRADAN/chmapbrowse that conforms to the standard OHRC/TMC-2/IIRS PDS4 label will parse correctly, whatever the eventual official set turns out to be.
- Reference-side ingestion (`Acquisition` construction) is sensor-agnostic across NAC, WAC, and SELENE TC/MI — same dataclass, different `sensor` field.
- The demo and the ablation dataset (Section 5/6 of `PROJECT_CONTEXT.md`) use **whatever real footprints we can register from the public archives now**, plus the synthetic ground-truth harness for accuracy claims. This means our accuracy numbers do not depend on ISRO releasing the official evaluation set before the internal round.
- The moment the official "TBD" link is published, the only required change is re-running the existing pipeline against the new files — no code change, because the adapter is schema-based, not manifest-based.

This is worth stating explicitly to the judges: *"our software was built to be correct against the PDS4 standard, not against a fixed sample, because the dataset link was marked TBD at submission time — so it runs unmodified against whatever ISRO eventually publishes."* That directly answers the PS's own phrase "generic software solution."

---

## 4. Immediate action items (not code — do these this week)

| Action | Who | Why now |
|---|---|---|
| Register on `chmapbrowse.issdc.gov.in` / PRADAN | A team member, human step | Approval can take days; blocks all real-data work |
| Pick 2–3 real OHRC/TMC-2/IIRS footprints with confirmed NAC/WAC overlap via QuickMap | Whoever owns `src/lucid/io/` | Need real pairs to validate the parser and the relighting module beyond synthetic tests |
| Download SLDEM2015 tiles and/or LOLA polar DEM for the same footprints | Whoever owns `src/lucid/relight/` | The renderer is on the critical path; it needs real DEM tiles, not just synthetic ones, before Batch 1 closes |
| Bookmark the corrected LROC downloads URL for the team | Anyone | Avoid repeated confusion from the broken PS link |
| Watch for the official "TBD" dataset announcement | SPOC / team lead | If it lands before the internal round, swap in immediately — no code change needed per §3 |

---

## 5. Summary answer

- **chmapbrowse link:** works, but is a login-gated browse portal, not a direct download — registration is a human action item, not a code task.
- **LROC NAC download link:** **broken as published** (typo: `im.-ldi.com` → should be `im-ldi.com`). Corrected version verified working and recorded above.
- **QuickMap link:** works exactly as given, verified.
- **SELENE:** no link was given at all; we've noted it as an open item rather than inventing a URL.
- **The deeper point:** the PS says the dataset is TBD, so the architecture is deliberately schema-based rather than manifest-based, which means this isn't a loose end — it's already handled by design.
