# magick (ImageMagick)

Manipulate, convert, and process images from the command line with the [ImageMagick](https://imagemagick.org) 7 `magick` CLI, across 200+ formats.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

## What it covers

- **Inspection** — `identify` format tokens, brightness checks, capability/delegate lists.
- **Conversion** — JPEG/PNG/WebP/AVIF/HEIC, multi-frame formats, PDF/SVG rasterization, images→PDF, favicons.
- **Resize, crop, pad** — fit/fill/exact geometry, `+repage`, gravity, borders and mats.
- **Quality & compression** — JPEG/WebP/PNG tuning, size targets, metadata stripping (orientation-safe).
- **Color & effects** — grayscale, levels, gamma, transparency, blur/sharpen, deskew.
- **Compositing & drawing** — watermarks, overlays, append strips, text/captions, shapes, gradients, canvases.
- **Montage** — labeled contact sheets and grids.
- **Comparison** — AE/RMSE/DSSIM metrics for regression and before/after checks.
- **Redaction** — pixelate and solid-box privacy edits.
- **Batch** — `mogrify -path` and shell loops, plus Windows/Git Bash notes (`magick`, never `convert`).

## Requirements

ImageMagick **7+** with the `magick` command on `PATH`.

```bash
winget install ImageMagick.ImageMagick    # Windows
brew install imagemagick                    # macOS
sudo apt install imagemagick               # Debian/Ubuntu (confirm it provides `magick`, i.e. v7+)
```

Optional delegates: Ghostscript (PDF/EPS), libheif (HEIC). Check with `magick -list format`.

## Canonical documentation

- [Usage examples](https://usage.imagemagick.org)
- [Command-line options](https://imagemagick.org/script/command-line-options.php)
- [Format and print escapes](https://imagemagick.org/script/escape.php)
