# Fujifilm X-T50 support notes

Working notes from diagnosing "the photo warps and changes colour before settling" on Fujifilm X-T50 RAF files,
and how to build a bundled camera colour profile for the model.

## Why a photo visibly changes on first view

1. The grid/loupe shows the raw's **embedded JPEG** (the camera's own Film Simulation rendering) as a stand-in
   while LightCraft decodes (`Slot::ThumbQuick` in `crates/ui-egui/src/render.rs`).
2. In the background, LightCraft decodes the sensor and, because the X-T50 has no *bundled* colour profile yet
   (only `X-H2S` and `X-T4` are bundled — see `crates/engine/src/camera_profiles.rs`), runs a **per-photo colour
   fit** (`fit_preview` in `crates/engine/src/camera_preview.rs`): it compares a small proxy of the sensor render
   against that same embedded JPEG and solves for a 3×3 chromaticity matrix plus a hue/saturation/value table to
   match it.
3. Once that finishes, LightCraft's own render replaces the stand-in.

That swap — JPEG stand-in → freshly-solved colour transform — is the visible "warp and colour change". It's the
intended behaviour for a camera without a bundled profile, not a decode bug. It only costs time on the *first*
decode of a photo; the render is cached afterwards (content hash + settings hash).

## Decode verification performed

The RAF decoder (`crates/raw/src/vendor/raf.rs`) reads CFA layout, crop and black levels from each file's own
tags rather than hard-coding anything per sensor generation, so X-Trans 5 (X-T50, X-T5, X-H2, X100VI) goes
through the same generic path already verified on X-Trans I/III bodies.

Ran `lightcraft-cli calibrate` directly against 30 local X-T50 RAFs (actual sensor decode, not a shortcut):

```
X-T50: 29 photos, 105839 colour pairs, hue/saturation table true
```

29 of 30 fit cleanly against their own embedded JPEG; the one skip was a file lacking a usable embedded preview
(normal, not an error). This confirms the decode itself is correct for this camera — the only gap is the missing
bundled colour profile.

## Photos available for building a profile

Scanned the library for X-T50 RAF files (camera model string at byte offset 24, 20 bytes, from the RAF header):

| Folder | X-T50 RAF files |
|---|---|
| `japan_2026/raw` | 116 |
| `summer_2026_hiking` (incl. `flume_gorge`) | 483 |
| **Total** | **599** |

That's more volume and broader scene variety (travel/city + outdoor/hiking light) than the bundled `X-T4`
profile was trained on (545 photos from one trip).

## What it takes to build a profile (from `docs/camera-preview-colour.md`)

1. **Minimum:** at least 5 "usable" photos per model (decodable embedded JPEG, ≥5% coloured pixels, aspect ratio
   within 2% of the sensor crop). 599 available is far more than enough.
2. **Per photo:** `lightcraft-cli calibrate` decodes the raw sensor to a 192×192 proxy and the embedded JPEG to
   the same size, pairs up to 4000 pixels between them (excluding near-black/clipped/monochrome samples), and
   pools those pairs across every photo of the model.
3. **The fit:** one ridge-regularised 3×3 chromaticity matrix (bounded coefficients, fit on white-balanced
   camera RGB so no single photo's white point is baked in), plus a 5×72×5 hue/saturation/value table (DNG
   `ProfileHueSatMap`-style) fit to the residual hue errors the matrix alone misses.
4. **Acceptance gates:** a third of pixels are held out from both fits; the combined model must cut linear-display
   squared error by ≥30% with per-channel RMS ≤0.10, else the fit is rejected (that photo keeps the documented
   neutral fallback or its own per-file fit).
5. **Output:** one `X-T50.json` — aggregate matrix + table only, no photos or pixel data retained — written to
   `<DIR>` via `--out`, or by default to the profiles folder LightCraft reads (`$LIGHTCRAFT_CAMERA_PROFILES`, else
   `<config>/camera-profiles`; macOS: `~/Library/Application Support/LightCraft/camera-profiles`). A profile
   dropped there overrides/extends the built-in bundled set automatically — no app restart beyond a cache
   invalidation.
6. **Validation approach used for the bundled profiles** (recommended before shipping a local one too): withhold
   every 5th photo, fit on the rest, and measure mean CIE76 ΔE against the camera JPEG on the withheld set only.
   For X-H2S/X-T4 this showed the pooled profile beating per-file fitting slightly (X-H2S 1.70 → 1.62, X-T4
   2.43 → 2.34 ΔE).

## Command

```sh
lightcraft-cli calibrate --max 0 --out <profiles-dir> \
  "/Users/warrenvu/iCloud Drive (Archive) - 1/japan_2026/raw" \
  "/Users/warrenvu/Documents/photos/summer_2026_hiking"
```

`--max 0` pools every usable file instead of the 300-file default. Run the held-out validation pass (every 5th
file withheld) before installing the result to the live profiles folder, and only install after a positive ΔE
result (done for X-T50: see Result below).

## Result (2026-10-08): profile built, validated and installed locally

- **Inventory corrected:** a second Japan folder (`~/Documents/photos/japan_2026/raw`, 487 files) was missing above,
  and many RAFs exist twice (`raw/` and its parent folder). Deduplicated by content: **738 unique X-T50 photos**
  (Provia 244, Velvia 189, Classic Chrome 127, Astia 118, Nostalgic Neg 35, Reala ACE 16, Classic Negative 9).
- **Held-out validation** (every 5th unique photo withheld, 147 photos; mean CIE76 ΔE vs the camera JPEG at 200 px,
  rendered via `lightcraft-cli render --size 400`; absolute numbers are not comparable to the X-T4 ones above, only
  the differences between columns are):

  | Film simulation | n | per-file fit | one X-T50 profile | one profile per film sim |
  |---|---|---|---|---|
  | All | 146 | 6.73 | 6.79 | **6.45** (100/146 improved) |
  | Provia | 48 | 6.26 | 6.36 | 6.31 |
  | Velvia | 35 | 8.04 | 8.72 | 7.99 |
  | Classic Chrome | 28 | 5.54 | 5.84 | 5.48 |
  | Astia | 23 | 6.55 | 6.68 | 6.23 |
  | Nostalgic Neg | 9 | 8.54 | 5.41 | 5.28 |
  | Reala ACE | 3 | 6.01 | 4.91 | 5.01 |

  One pooled profile is about even with per-file fitting on average: it fixes photos whose own fit fails (the
  negative film sims), but pulls Velvia and Classic Chrome towards an average look. Per-film-sim profiles beat both.
- **Shipped:** the model profile plus Astia, Classic Chrome, Nostalgic Neg, Provia, Reala ACE and Velvia profiles
  (`assets/camera-profiles/X-T50*.json`, 715 / 113 / 127 / 28 / 241 / 15 / 187 photos). Classic Negative has only 4
  usable photos and Eterna/Pro Neg none, so they use the model profile. A photo's film simulation is read from the
  maker note (`FilmMode`, 0x1401) of its embedded JPEG and selects `X-T50 <simulation>` before `X-T50`. End to end on
  the 147 withheld photos: mean ΔE 6.81 (per-file fit) → 6.44, 103 improved; Provia 6.26 → 6.31 and Velvia 8.04 → 7.99
  are unchanged within noise, Astia 6.55 → 6.23, Nostalgic Neg 8.54 → 5.28, Reala ACE 6.01 → 5.01. One Provia photo
  regressed by 8.9 ΔE.
- **Local override:** files in `~/Library/Application Support/LightCraft/camera-profiles/` replace the bundled ones;
  delete them to use the bundled set.
