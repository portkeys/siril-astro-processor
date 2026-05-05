# Siril Astro Processor — Claude Code plugin

Process deep sky astrophotography from your terminal using natural language.
Hand Claude Code a folder of light frames and get back a finished, ready-to-share
image — no Siril GUI, no `.ssf` scripts, no menus to learn.

Built around the [Siril](https://siril.org/) CLI and StarNet2, with a curated
pipeline that handles convert → register → stack → background-extract → denoise
→ star removal → stretch → recombine. Pick a named **recipe** (`natural`,
`starless`, or `punchy`) for the star treatment, and the plugin generates and
runs the right script for your data.

## Demo

Talk to Claude Code:

> "Process the photos in `~/AstroData/2025-LagoonNebula/`"

The plugin will:

1. Run a preflight check (siril-cli present, StarNet configured, dataset writable).
2. Read a sample FITS header to detect OSC vs mono and confirm the Bayer pattern.
3. Generate a tailored `.ssf` script and run it via `siril-cli`.
4. Save `before.png` (raw stack autostretched) and `after.png` (final processed)
   side-by-side so you can see the result.

If you want a different look, say so:

> "Make it the punchy version with stars at 50%"

The plugin re-runs only the recombine step and produces a new `after_v2.png`.

## Install

```bash
# In Claude Code:
/plugin marketplace add portkeys/siril-astro-processor
/plugin install siril-astro-processor@portkeys-siril-astro-processor
```

## Prerequisites

The plugin handles environment setup automatically on first use, but you'll
need:

- **macOS or Linux** with Claude Code installed
- **Siril 1.4+** ([download](https://siril.org/download/))
  ```bash
  # macOS via Homebrew (cask):
  brew install --cask siril
  ```
- **StarNet2** (only required for star removal)
  - On Apple Silicon Macs the Homebrew cask installer needs `sudo`. Ask Claude
    Code to "install StarNet2" — it will use the workaround documented in the
    skill (extract the binary to `~/Applications/`, patch the rpath, configure
    Siril) automatically.
- **Disk space**: stacking 100 frames produces ~1 GB of intermediate FITS files.
  Plan accordingly.

## How recipes work

Star treatment is the most personal post-processing decision. The plugin offers
three named recipes; the rest of the pipeline is identical.

| Recipe | What it does | When to pick it |
|---|---|---|
| `natural` *(default)* | Processed nebula + stars exactly as they appear in a single autostretch of the raw stack. Stars look real-sized. | Most cases. Closest to "what the photo would look like, but better." |
| `starless` | The processed nebula only — stars removed entirely. | When the nebula structure is the whole point and stars distract from it. |
| `punchy` | Recombined with an autostretched starmask, scale factor adjustable (0.3–1.0). | When you want more dramatic, contrasted stars over the nebula. |

You can iterate just on the recipe — the expensive convert + stack steps run
once and are cached.

## What gets produced

In your dataset directory, alongside `LIGHTS/`:

```
result.fit                      # raw stack (linear)
result_bg_extracted.fit         # background-extracted
result_denoised.fit             # noise-reduced
result_starless.fit             # starnet output
result_enhanced.fit             # processed starless (autostretch + clahe + satu)
final.fit                       # final composite
before.png / before.jpg         # raw stack autostretched (the "before" preview)
after.png / after.jpg / after.tif  # final result, multiple formats
```

Every intermediate is preserved, so you can re-run from any point if you want
to try a different recipe or stretch.

## Troubleshooting

The skill includes a thorough troubleshooting + macOS gotchas section that the
agent reads automatically. Common issues:

- **"No images found"** — your frames must be inside a `LIGHTS/` subfolder.
- **Heavily green image** — happens with OSC data without color calibration.
  Ask: "remove the green cast" — it'll add `rmgreen`.
- **Stars dominate the image** — you ran the `punchy` recipe with too high a
  scale factor. Ask for `natural` or lower the factor.

## Contributing

Issues and PRs welcome. The skill itself is a single
[`SKILL.md`](skills/process-deep-sky/SKILL.md) — start there if you want to
add a recipe or fix a workflow.

## License

MIT
