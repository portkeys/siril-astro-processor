# Siril CLI Commands Reference

## Running Scripts

```bash
siril-cli -s script.ssf              # Run script
siril-cli -s script.ssf -a /path     # Pass argument (accessible as {0} in script)
siril-cli -s script.ssf -d var=val   # Set global variable
```

## Script Syntax (.ssf files)

```
#!/bin/siril
# Comments start with #
# Use backslash (\) for line continuation
# Single quotes for paths with spaces: 'My File.fits'
# Variables: set with var=value, use with $var
# Arguments from CLI: {0}, {1}, etc.
```

---

## 1. File & Directory Operations

| Command | Syntax | Description |
|---------|--------|-------------|
| cd | `cd directory` | Change working directory |
| load | `load filename` | Load single image (FITS, RAW, BMP, PNG, JPEG, TIFF, SER) |
| load_seq | `load_seq seqname` | Load sequence by name |
| convert | `convert basename [-debayer] [-fitseq] [-out=dir]` | Convert RAW/images to FITS |
| close | `close` | Close current image/sequence |
| set16bits | `set16bits` | Set 16-bit mode |
| set32bits | `set32bits` | Set 32-bit mode |

## 2. Calibration & Preprocessing

```
calibrate seqname [-bias=file] [-dark=file] [-flat=file] \
  [-cc=dark siglo sighi] [-cfa] [-debayer] [-equalize_cfa] \
  [-opt] [-all] [-prefix=pre_] [-fitseq]
```

- `-bias=`, `-dark=`, `-flat=`: Master calibration frames
- `-cc=dark siglo sighi`: Cosmic ray correction using dark
- `-cfa`: Preserve Bayer pattern
- `-debayer`: Demosaic
- `-opt`: Optimize dark/flat exposure matching
- `-all`: Process all (not just selected)

**Cosmetic correction:**
- `cosme file.lst` — Fix hot/cold pixels from list
- `find_hot file cold_sigma hot_sigma` — Detect hot/cold pixels

## 3. Registration (Alignment)

```
register seqname [-2pass] [-selected] [-prefix=reg_] \
  [-layer=] [-maxstars=200] [-interp=lanczos3] \
  [-drizzle [-pixfrac=] [-kernel=] [-flat=]]
```

- `-2pass`: Two-pass (coarse then fine) — recommended
- `-maxstars=`: Max stars for matching
- `-drizzle`: Drizzle for undersampled data
- `-interp=`: lanczos3 (default), cubic, linear, nearest

**Apply existing registration:**
```
seqapplyreg seqname [-prefix=] [-interp=] [-drizzle]
```

## 4. Stacking

```
stack seqname { sum | min | max }
stack seqname { med | median } [-nonorm | -norm=auto] [-rgb_equal] [-out=result]
stack seqname { rej | mean } [rejection_type] [sigma_lo sigma_hi] \
  [-norm=auto] [-weight={noise|wfwhm|nbstars}] [-out=result] [-32b]
```

**Rejection types:** `winsorized`, `linear`, `percentile`, `mad`

**Common usage:**
```
stack pp_light rej winsorized 3 3 -norm=auto -weight=wfwhm -out=result
```

## 5. Background/Sky Subtraction

```
subsky { -rbf | degree } [-samples=20] [-tolerance=1.0] [-smooth=0.5] [-dither]
```

- `-rbf`: Radial Basis Function (advanced, recommended)
- `degree`: Polynomial (0=constant, 1=linear, 2=quadratic, etc.)
- `-samples=`: Number of sample points
- `-tolerance=`: Outlier rejection tolerance (0-2)

**For sequences:** `seqsubsky seqname { -rbf | degree } [-prefix=]`

## 6. Color Calibration

**Photometric Color Calibration (requires plate-solving first):**
```
pcc [-limitmag=] [-catalog=] [-bgtol=lower,upper]
```

**Spectral Photometric CC (more advanced):**
```
spcc [-oscsensor=name] [-oscfilter=name] [-whiteref=] [-bgtol=0.2,1.0]
```

## 7. Noise Reduction

```
denoise [-mod=0.8] [-da3d | -vst | -sos=n] [-indep] [-nocosmetic]
```

- `-da3d`: DCT 3D denoising (best quality)
- `-vst`: Variance Stabilizing Transform (faster)
- `-mod=`: Modulation strength 0-1 (lower = less aggressive)

**Deconvolution:**
- `rl [-loadpsf=] [-iters=] [-tv]` — Richardson-Lucy
- `wiener [-loadpsf=] [-alpha=]` — Wiener filter

## 8. Star Removal

```
starnet [-stretch] [-upscale] [-stride=value] [-nostarmask]
```

- `-stretch`: Pre-stretch before processing
- `-upscale`: Better quality (slower)
- `-nostarmask`: Don't create separate star mask
- Outputs: starless image + starmask (stars-only)

## 9. Stretching (Linear → Nonlinear)

**Automatic stretch:**
```
autostretch [-linked] [shadowsclip targetbg]
```

**Arcsinh stretch (good general purpose):**
```
asinh [-human] stretch [offset] [-clipmode=rgbblend]
```
- `stretch`: 1-1000 (higher = more stretch)
- `offset`: Black point 0-1

**Generalized Hyperbolic Stretch (most control):**
```
ght -D=0.08 [-B=0] [-LP=0.05] [-SP=0.5] [-HP=1.0] [-human]
```
- `-D=`: Key parameter — shadows displacement (0.01-0.5 typical)
- `-B=`: Black point
- `-LP=`/`-SP=`/`-HP=`: Low/Symmetry/High points

**Modified Arcsinh (GHT-like parameters):**
```
modasinh -D=0.1 [-LP=] [-SP=] [-HP=] [-human]
```

**Linear stretch:**
```
linstretch -BP=0.001 [-sat]
```

## 10. Histogram & Curves

```
mtf low mid high [channels]    # Midtones Transfer (low=black, mid=gamma, high=white)
clahe cliplimit tileSize       # Local contrast enhancement
```

## 11. Cropping & Geometry

```
crop [x y width height]        # Crop (no args = use selection)
rotate degree [-nocrop]        # Rotate
mirrorx                        # Horizontal mirror
mirrory                        # Vertical mirror
resample factor [-interp=]     # Resize (0.5 = half)
binxy coefficient [-sum]       # Pixel binning
```

## 12. Saving & Exporting

```
save filename                  # Save as FITS
savetif filename [-deflate]    # Save as TIFF (16-bit)
savetif8 filename [-deflate]   # Save as TIFF (8-bit)
savepng filename               # Save as PNG
savejpg filename [quality]     # Save as JPEG (quality 0-100)
```

## 13. Plate Solving

```
platesolve [-force] [-focal=mm] [-pixelsize=um] \
  [-catalog=] [-radius=deg] [-downscale]
```

Requires approximate coordinates or FITS header info. Sets WCS in header.

## 14. Color Saturation

```
satu amount [-hue=]            # Adjust saturation (amount: -1 to 1+)
```

## 15. Pixel Math / Compositing

```
pm "expression" [-rescale [low high]] [-nosum]
```

Variables in the expression are FITS files in the current working directory, referenced
by basename (no `.fit` extension), wrapped in `$`. Max 10 images per expression.

```
pm "$starless$+$starmask$" -rescale          # additive recombine, normalize to [0,1]
pm "$nebula$+0.3*$stars$" -rescale           # blend stars at 30% strength
pm "$processed$+$full$-$full_starless$"      # natural-recipe: stars from full stack
```

Other compositing:
```
rgbcomp -lum=file red green blue             # RGB composition from channels
```

---

## Typical OSC Deep Sky Script (No Calibration Frames)

```
#!/bin/siril
requires 1.4.0

# Set working directory
cd {0}

# Convert to FITS sequence (cd into LIGHTS — convert reads from CWD)
cd LIGHTS
convert light -out=../process -debayer        # -debayer for OSC; drop for mono
cd ../process

# Register (align) frames; seqapplyreg required after -2pass to write r_*.fits
register light -2pass -maxstars=500 -interp=lanczos3
seqapplyreg light -interp=lanczos3

# Stack with rejection
stack r_light rej winsorized 3 3 -norm=auto -weight=wfwhm -out=../result

cd ..
load result

# Crop edges (adjust coordinates to your image)
# crop x y w h

# Background extraction
subsky -rbf -samples=30 -tolerance=1.0

# Noise reduction on linear data
denoise -da3d -mod=0.7

# Star removal (creates starless + starmask)
starnet -stretch

# Stretch the starless image
autostretch

# Save results
save result_processed
savetif result_processed
savejpg result_processed 95
```
