---
name: siril-astro-processor
description: >
  Automate deep sky astrophotography image processing using Siril's CLI scripting engine.
  Use this skill whenever the user wants to process astrophotos, stack light frames,
  calibrate astronomical images, remove light pollution gradients, stretch deep sky images,
  remove stars, reduce noise, or do any post-processing on telescope/camera captures.
  Triggers on: astrophotography processing, deep sky imaging, stacking subs/subframes,
  FITS/.fit files from telescopes, Siril, light frames, dark frames, flat frames, bias frames,
  calibration frames, star removal, background extraction, image stretching, nebula/galaxy
  processing, OSC (one-shot-color) processing, or any mention of processing telescope data.
---

# Siril Astrophoto Processor

This skill generates and executes Siril CLI scripts (`.ssf` files) to process deep sky
astrophotography data. It follows a proven workflow adapted from Paolo's deep sky processing
guide, using Siril's native command-line interface for reliable, reproducible batch processing.

## Environment Preflight

Run these checks *before* generating a script. Each maps to a real failure mode we've hit:

```bash
# 1. Siril CLI present (macOS install path)
/Applications/Siril.app/Contents/MacOS/siril-cli --version || siril-cli --version

# 2. StarNet binary configured in Siril (look in the config file, NOT just $PATH)
grep -E '^starnet_(exe|weights)=' \
  ~/Library/Application\ Support/org.siril.Siril/siril/config.1.4.ini

# 3. Dataset folder writable (zip extracts often arrive read-only)
test -w "$DATASET_DIR/LIGHTS" || echo "needs: chmod -R u+w $DATASET_DIR"

# 4. Confirm OSC vs mono — read BAYERPAT from a sample frame
/Applications/Siril.app/Contents/MacOS/siril-cli -s - <<EOF
requires 1.2.0
cd "$DATASET_DIR/LIGHTS"
load $(ls "$DATASET_DIR/LIGHTS" | head -1)
dumpheader
EOF
# BAYERPAT='RGGB' (or similar) → OSC, must -debayer
# No BAYERPAT → mono, do not -debayer
```

If StarNet is missing, see **macOS Gotchas → Installing StarNet2** below.

## How This Skill Works

1. **Gather info** about the user's dataset (target, number of frames, calibration frames available, camera sensor)
2. **Generate a `.ssf` script** tailored to their data and goals
3. **Execute via `siril-cli -s script.ssf`** (or `siril --cli -s script.ssf` depending on install)
4. **Review results** and iterate on processing parameters

## Understanding the Data

Before generating a script, determine:

- **What data exists?** Look in the user's directory for:
  - `LIGHTS/` folder containing `.fits`, `.fit`, `.cr2`, `.nef`, or other RAW files
  - `DARKS/`, `FLATS/`, `BIASES/` folders (calibration frames — often absent for beginners)
  - Already-stacked `result.fit` or similar

- **Camera type?** Check FITS headers for sensor info:
  - OSC (One-Shot-Color / Bayer matrix) — most common for beginners
  - Mono with filters — advanced setup

- **What processing stage are they at?**
  - Raw subs needing full pipeline → start from conversion
  - Already stacked → skip to post-processing
  - Already stretched → only enhancement steps remain

## The Processing Pipeline

The pipeline has two major phases: **preprocessing** (steps 1-3) and **post-processing** (steps 4-10).

### Phase 1: Preprocessing (Automated via Script)

These steps are fully scriptable and should run as a single `.ssf` script:

#### Step 1: Convert raw files to FITS sequence
```
cd LIGHTS              # convert reads from CWD — must cd into the frames dir
convert light -out=../process -debayer    # -debayer required for OSC data
cd ../process
```
`convert` looks for files in CWD (not in a subfolder), so always `cd LIGHTS` first.
Add `-debayer` for OSC sensors (Bayer-pattern, e.g. ZWO ASI***MC). Omit for mono.

#### Step 2: Calibrate (if calibration frames exist)
```
calibrate light -dark=master_dark.fits -flat=master_flat.fits -bias=master_bias.fits -cc=dark 3 3 -debayer -prefix=pp_
```
Skip this step entirely if the user has no darks/flats/biases. The "OSC Processing without DBF" workflow omits calibration.

#### Step 3: Register (align) all frames
```
register light -2pass -maxstars=500 -interp=lanczos3
seqapplyreg light -interp=lanczos3        # REQUIRED after -2pass
```
**Critical**: Siril 1.4's `register -2pass` only computes the per-frame transforms and
writes them to the `.seq` file. It does *not* generate the `r_*.fits` registered frames.
You must follow with `seqapplyreg` to produce the aligned frames that `stack` consumes.
Skipping this gives "Reading sequence failed: r_light.seq" at the stack step.

#### Step 4: Stack frames
```
stack r_light rej winsorized 3 3 -norm=auto -weight=wfwhm -out=../result
```
- `rej winsorized 3 3` rejects outliers (satellites, cosmic rays, planes)
- `-weight=wfwhm` weights by seeing quality
- `-norm=auto` normalizes brightness across frames

### Phase 2: Post-Processing (Step-by-Step, Often Interactive)

These steps benefit from visual inspection between each one. Generate them as individual commands or as a script with saves between steps so the user can review.

#### Step 5: Crop stacking edges
```
crop x y width height
```
Remove black/uneven edges from stacking. The user should specify crop coordinates, or use a generous auto-crop by trimming ~5% from each edge.

#### Step 6: Background extraction (remove gradients)
```
subsky -rbf -samples=30 -tolerance=1.0 -smooth=0.5
```
Removes light pollution gradients and uneven illumination. RBF method is more flexible than polynomial.

#### Step 7: Photometric Color Calibration (optional, requires plate-solve)
```
platesolve -focal=focal_mm -pixelsize=pixel_um -downscale
pcc
```
Requires knowing focal length and pixel size. Corrects color balance using catalog star colors.

#### Step 8: Noise reduction (on linear data)
```
denoise -da3d -mod=0.7
```
Apply BEFORE stretching for best results. `-mod=0.7` is a moderate setting; lower values preserve more detail.

#### Step 9: Star removal
```
starnet -stretch -upscale
```
Separates stars from nebulosity/galaxy. Creates two images:
- Starless image (for stretching and enhancing)
- Starmask (stars only, to be recombined later)

#### Step 10: Stretch (linear → nonlinear)
```
autostretch
```
Or for more control:
```
ght -D=0.08 -B=0 -LP=0.05 -human
```
Or:
```
asinh 100 0.05 -human
```
This is the most subjective step. `autostretch` is safe; GHT/asinh give more control.

#### Step 11: Enhancement (on stretched data)
```
clahe 3 8                    # Local contrast
satu 0.3                     # Boost color saturation
unsharp 2 0.5                # Sharpen
```

#### Step 12: Save results
```
save result_final
savetif result_final -deflate
savejpg result_final 95
```

## Script Generation Strategy

When generating scripts, follow these principles:

1. **Save intermediate results.** After stacking, after background extraction, after stretching — save at each major step so the user can go back.

2. **Use conservative defaults.** It's better to under-process than over-process. The user can always re-run with stronger settings.

3. **Comment everything.** Each command in the script should have a comment explaining what it does and why.

4. **Handle the "no calibration" case.** Most beginners (and the datasets in this project) don't have darks/flats/biases. The script should work without them.

5. **Prefix conventions.** Siril uses prefixes to track processing:
   - `pp_` = preprocessed (calibrated)
   - `r_` = registered (aligned)
   - The stacked result is typically `result.fit`

## Example: Full Pipeline Script for OSC Without Calibration (`natural` recipe)

This is the working template. Substitute `{ABS_DATASET_DIR}` with an absolute path to
the folder *containing* `LIGHTS/`. All paths must be absolute — the CLI ignores the
shell's CWD.

```
#!/bin/siril
requires 1.2.0

# ============================================
# Deep Sky Processing - OSC without calibration, "natural" recipe
# Target: {target_name}     Frames: {num_frames} lights
# ============================================

cd '{ABS_DATASET_DIR}'

# --- PREPROCESSING ---
cd LIGHTS
convert light -out=../process -debayer    # -debayer for OSC; drop for mono
cd ../process

register light -2pass -maxstars=500 -interp=lanczos3
seqapplyreg light -interp=lanczos3        # required after -2pass

stack r_light rej winsorized 3 3 -norm=auto -weight=wfwhm -out=../result

# --- BEFORE PREVIEW (also feeds the natural-recipe recombine later) ---
cd ..
load result
autostretch
save before_stretched
savepng before && savejpg before 95

# --- POST-PROCESSING (linear) ---
load result
subsky -rbf -samples=30 -tolerance=1.0
save result_bg_extracted

denoise -da3d -mod=0.7
save result_denoised

starnet -stretch                          # writes starless_<input>.fit + starmask_<input>.fit
save result_starless

autostretch                               # per-channel; balances the green-dominant Bayer mix
save result_starless_stretched

clahe 3 8
satu 0.25
save result_enhanced                      # this is the "starless" recipe output

# --- NATURAL-RECIPE STAR RECOMBINE ---
# Re-derive starless of the autostretched stack (different from result_starless,
# which is starless of the linear/denoised data) so we can subtract cleanly.
load before_stretched
starnet                                   # NO -stretch flag; data is already stretched
save before_starless

# (before - before_starless) = stars at the brightness/size from "before"
pm "$result_enhanced$+$before_stretched$-$before_starless$" -rescale
save final

savepng after && savejpg after 95 && savetif after -deflate

close
```

**Iterating on aesthetics**: nothing above the "NATURAL-RECIPE STAR RECOMBINE" comment
needs to change between recipes. Swap only that block for `starless` or `punchy` —
they all consume `result_enhanced.fit`.

## Adapting the Script

Read `references/siril-commands.md` for the full command reference when you need to:
- Adjust stacking parameters (rejection method, sigma values, weighting)
- Use different stretching algorithms (GHT, asinh, modasinh)
- Add plate solving for color calibration
- Process mono camera data with separate RGB channels
- Handle drizzle for undersampled data

## Troubleshooting

**"No images found"**: Check that LIGHTS/ folder exists and contains supported files. The folder name is case-sensitive on Linux.

**Green tint after stacking**: The channels may be unlinked. This is a display issue; PCC or manual color balance fixes it.

**Script fails at register**: Not enough stars detected. Try `-maxstars=1000` or check if frames are too out of focus.

**`Reading sequence failed: r_light.seq` at stack step**: You used `register -2pass` without the `seqapplyreg` follow-up. See Step 3.

**Image comes out heavily green**: GHT or other linked stretches preserve the Bayer green dominance. Add `rmgreen` after the stretch, or use `autostretch` (unlinked) instead.

**Stars dominate the image after recombine**: You autostretched the *starmask alone*, which overstretches because the mask is mostly empty space. Use the `natural` recipe (subtraction approach), or scale the starmask down with `pm "...+0.3*$starmask$..."`.

**Script fails at starnet**: StarNet++ not installed or path not configured. Check Siril preferences or install StarNet2CLI.

**Dark image after stacking**: This is normal — the image is in linear state. Stretching (step 10) makes it visible.

## Recipes (aesthetic presets)

Star treatment is the most personal post-processing decision. Pick one of these named
recipes for the final-blend step; the rest of the pipeline is the same. Default is
**natural** unless the user expresses a preference.

### Recipe: `natural` (recommended default)
Processed nebula + stars exactly as they appear in a single autostretch of the raw stack.
Stars come out at their real relative size and brightness — no separate stretch on the
starmask, which tends to overstretch faint stars.
```
# Assume result_enhanced.fit (processed starless) already exists from main pipeline.
load result               # the linear stack
autostretch
save before_stretched
starnet                   # no -stretch — data is already stretched
save before_starless

# (before_stretched - before_starless) isolates the stars at "natural" intensity.
pm "$result_enhanced$+$before_stretched$-$before_starless$" -rescale
save final
savepng after && savejpg after 95
```

### Recipe: `starless` (dramatic nebula, no stars)
Skip the recombine entirely. Use the v1-style processed starless image directly. Best
when the nebula structure is the whole point and stars are visual noise.

### Recipe: `punchy` (stars at adjustable strength)
Recombine an autostretched starmask at a scale factor. 0.3 = subtle accent stars,
0.5 = balanced, 1.0 = star-dominated.
```
load starmask_result_denoised
autostretch
rmgreen                   # OSC without PCC: green cast
save stars_stretched
pm "$result_enhanced$+0.3*$stars_stretched$" -rescale
```

### Why we don't use raw GHT for this
GHT in `-human` mode preserves the green-channel dominance of debayered Bayer data,
producing a green-cast image. Either follow GHT with `rmgreen`, or use `autostretch`
which per-channel-stretches and dodges the issue.

## macOS Gotchas

These all bit us during development. Friends on Mac will hit them too.

### Siril CLI defaults CWD to ~/Pictures
Siril's CLI ignores the shell's working directory and defaults to `~/Pictures`. Always
use **absolute paths** in `cd` commands inside `.ssf` scripts, and pass an absolute path
to `siril-cli -s`. Otherwise: `File [script.ssf] does not exist`.

### Read-only datasets after zip extraction
macOS often extracts archives with permissions like `dr-xr-xr-x`. Siril needs to write
the `process/` sequence and result files into the dataset folder.
```bash
chmod -R u+w "$DATASET_DIR"
```

### Siril config location
Settings live in `~/Library/Application Support/org.siril.Siril/siril/config.1.4.ini`.
The keys we care about are `starnet_exe` and `starnet_weights`. Set them via:
```bash
printf 'requires 1.2.0\nset core.starnet_exe=/path/to/starnet2\n' | siril-cli -s -
```

### Installing StarNet2 on Apple Silicon
The Homebrew cask `starnet2` works but its installer needs sudo. Workaround:
```bash
# 1. Download via brew (so the zip lands in cache, even if install fails)
brew install --cask starnet2 || true

# 2. Extract to user space
mkdir -p ~/Applications/StarNet2
unzip -o ~/Library/Caches/Homebrew/Cask/StarNet2T_MacOS.zip*.zip \
  -d ~/Applications/StarNet2

# 3. Patch the binary's rpath (it hardcodes /usr/local/lib)
cd ~/Applications/StarNet2/StarNet2T_MacOS
install_name_tool -add_rpath "$PWD/lib" ./starnet2
codesign --force --deep --sign - ./starnet2

# 4. Point Siril at it
printf 'requires 1.2.0\nset core.starnet_exe=%s/starnet2\nset core.starnet_weights=%s/StarNet2_weights.pt\n' \
  "$PWD" "$PWD" | /Applications/Siril.app/Contents/MacOS/siril-cli -s -
```

### Pixelmath command name
The CLI command is `pm`, not `pixelmath`. Syntax: `pm "$name1$ + $name2$" -rescale`.
Variables reference FITS files in CWD by basename (no extension). Max 10 images.

### `ght` requires explicit `-SP=`
GHT defaults `-SP=` to 0, but `-LP` must be ≤ `-SP`. Always pass `-SP=0.5` (or whatever
midtone target you want). Without it: `Error: LP must be >= 0.0 and <= SP`.

### Saving non-autostretched data as PNG
`savepng` on a freshly-processed image whose values are still concentrated below 0.1
produces a near-black PNG even though the FITS is fine. The recombine via `pm -rescale`
normalizes to [0,1] and saves correctly. If you need a PNG of an intermediate, run
`autostretch` (or `pm "$image$" -rescale`) before `savepng`.

## Generating the script for a new dataset

1. Run the **Environment Preflight** checks. Capture: dataset path, OSC vs mono, calibration frames present, StarNet ok.
2. Ask the user (or pick a default): which **Recipe**? `natural` is the safe pick.
3. Generate a `.ssf` from the template below, substituting the dataset path and recipe block.
4. Run with `siril-cli -s /absolute/path/to/script.ssf`.
5. Inspect `before.png` and `after.png`. Iterate the recipe (not the whole pipeline) if the user wants different star treatment.
