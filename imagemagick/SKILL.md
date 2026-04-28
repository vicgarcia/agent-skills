---
name: imagemagick
description: Manipulate, convert, and process images from the command line using the magick CLI. Resize, crop, annotate, filter, composite, and convert between 200+ formats.
compatibility: Requires ImageMagick 7+ with the `magick` command available in PATH.
---

# ImageMagick

ImageMagick is a CLI tool for image manipulation. The `magick` command is the unified entry point for all operations — format conversion, resizing, color adjustments, effects, annotation, compositing, and batch processing across 200+ image formats.

```bash
magick [input] [options...] [output]
```

Options are applied in left-to-right order. The output format is determined by the output file extension.

**Option ordering matters.** Flags must appear before the output filename — anything after the output path is silently ignored:

```bash
# Wrong — -strip has no effect here
magick input.jpg output.jpg -strip

# Correct
magick input.jpg -strip output.jpg
```

Processing options also apply to the image state at that point in the pipeline, so order between options matters too:
```bash
magick input.jpg -resize 800x -sharpen 0x1 output.jpg   # Sharpen after resize (correct for most cases)
magick input.jpg -sharpen 0x1 -resize 800x output.jpg   # Sharpen before resize (different result)
```

Useful inspection and listing commands:

```bash
magick --version           # Check version and build info
magick -list format        # All supported formats
magick -list font          # Available fonts
magick identify image.jpg  # Inspect image metadata (inline subcommand)
```

`identify` is an inline subcommand, not a separate binary:

```bash
magick identify -verbose image.jpg   # Full metadata dump
magick identify -format "%wx%h\n" *.jpg  # Print dimensions for multiple files
```

---

## Format Conversion

The output format is inferred from the extension. No explicit flag needed.

```bash
magick input.jpg output.png
magick input.png output.webp
magick input.tiff output.jpg
magick input.heic output.jpg
```

### Multiple frames / layers

Some formats (GIF, TIFF, PDF) can contain multiple frames. Use bracket notation to select:

```bash
magick input.gif[0] first-frame.png     # First frame only
magick input.pdf[0] cover.png           # First page of PDF
magick input.pdf[0-2] page.png          # Pages 0, 1, 2 → page-0.png, page-1.png, page-2.png
```

### Images to PDF

```bash
magick image1.jpg image2.jpg image3.jpg output.pdf
magick *.jpg combined.pdf
```

### PDF page to image

```bash
magick -density 150 input.pdf[0] -quality 90 page.png
```

`-density` sets the DPI before rasterizing — use 150–300 for readable output.

### SVG to image

SVG is vector and must be rasterized. Set `-density` before the input to control output resolution:

```bash
magick -density 150 input.svg output.png
magick -density 300 input.svg output.png    # Higher DPI for sharp/print output
```

`-density` must come before the input filename — if placed after, ImageMagick has already rasterized at the default 72 DPI and the result will be blurry or undersized.

---

## Resize, Scale, Crop & Extend

### Resize

```bash
magick input.jpg -resize 800x600 output.jpg         # Fit within 800×600, preserve aspect ratio
magick input.jpg -resize 800x output.jpg            # Fix width, height auto
magick input.jpg -resize x600 output.jpg            # Fix height, width auto
magick input.jpg -resize 50% output.jpg             # Scale by percentage
magick input.jpg -resize '800x600!' output.jpg      # Force exact dimensions (distorts)
magick input.jpg -resize '800x600>' output.jpg      # Only shrink, never enlarge
magick input.jpg -resize '800x600<' output.jpg      # Only enlarge, never shrink
```

`>`, `<`, and `!` are shell special characters — always quote geometry strings containing them.

### Thumbnail (faster, strips metadata)

```bash
magick input.jpg -thumbnail 200x200 thumb.jpg
```

Prefer `-thumbnail` over `-resize` for previews — it's optimized for speed and auto-strips EXIF.

### Crop

```bash
magick input.jpg -crop WxH+X+Y output.jpg        # W wide, H tall, starting at (X, Y)
magick input.jpg -crop 400x300+100+50 output.jpg  # 400×300 region at offset (100, 50)
magick input.jpg -crop 2x2@ output.jpg            # Split into 2×2 grid of tiles
```

After cropping, canvas size may retain original dimensions. Use `+repage` to reset:

```bash
magick input.jpg -crop 400x300+100+50 +repage output.jpg
```

### Gravity-based operations

`-gravity` positions the reference point for crop, extent, composite, and annotation. Valid values:

| | | |
|---|---|---|
| `NorthWest` | `North` | `NorthEast` |
| `West` | `Center` | `East` |
| `SouthWest` | `South` | `SouthEast` |

```bash
magick input.jpg -gravity Center -crop 400x400+0+0 +repage output.jpg  # Center crop
```

### Extend / Add padding

`-extent` sets the canvas size. Content is placed according to `-gravity`:

```bash
# Add white padding to reach 1000×1000
magick input.jpg -background white -gravity center -extent 1000x1000 output.jpg

# Add transparent padding (PNG only)
magick input.png -background none -gravity center -extent 1200x800 +repage output.png
```

---

## Compositing, Combining & Annotation

### Append images into a strip

Combine multiple images side-by-side or stacked vertically:

```bash
magick image1.jpg image2.jpg image3.jpg +append output.jpg   # Horizontal strip
magick image1.jpg image2.jpg image3.jpg -append output.jpg   # Vertical stack
```

Images are aligned to a common edge (top for horizontal, left for vertical). Mismatched dimensions leave empty space — normalize sizes first if needed.

### Composite (layer one image over another)

```bash
magick base.jpg overlay.png -composite output.jpg
```

Control placement with `-gravity` and `-geometry` (offset):

```bash
# Center the overlay
magick base.jpg overlay.png -gravity Center -composite output.jpg

# Bottom-right corner, 10px inset
magick base.jpg overlay.png -gravity SouthEast -geometry +10+10 -composite output.jpg
```

### Text annotation

`-annotate` draws text at a position. `-gravity` sets the anchor, `-geometry` offsets from it:

```bash
magick input.jpg \
  -font Helvetica -pointsize 36 \
  -fill white \
  -gravity SouthWest -annotate +20+20 "Caption text" \
  output.jpg
```

With a semi-transparent fill for readability over photos:

```bash
magick input.jpg \
  -font Helvetica -pointsize 32 \
  -fill 'rgba(255,255,255,0.75)' \
  -gravity South -annotate +0+20 "© Example" \
  output.jpg
```

### Contact sheet / montage

`montage` is a subcommand of `magick` that arranges multiple images into a labeled grid. It has its own option set — do not mix in standard `magick` processing options (like `-resize` or `-blur`) outside of parenthesized image groups:

```bash
magick montage [inputs] [montage-options] output
```

```bash
# Basic grid, auto layout
magick montage *.jpg output-sheet.jpg

# Control tile layout and cell size
magick montage *.jpg -tile 4x3 -geometry 200x200+5+5 output-sheet.jpg
```

`-tile COLSxROWS` — grid dimensions. `-geometry WxH+HGAP+VGAP` — cell size and spacing.

Add labels under each image:

```bash
magick montage -label '%f' *.jpg -tile 4x -geometry 200x200+4+4 output-sheet.jpg
```

`-label` must come **before** the inputs — it is a per-image setting. Placed after, it applies to nothing. `%f` uses the filename. Other tokens: `%w` width, `%h` height, `%b` file size.

Style the contact sheet:

```bash
magick montage -label '%f' *.jpg \
  -tile 4x \
  -geometry 200x200+6+6 \
  -background '#1a1a1a' \
  -fill white \
  -font Helvetica \
  -pointsize 11 \
  output-sheet.jpg
```

### Create a solid color canvas

Useful as a base for compositing or as a background:

```bash
magick -size 1920x1080 xc:white output.png
magick -size 800x600 xc:'#1a1a2e' output.png
magick -size 100x100 xc:'rgba(255,0,255,0.5)' output.png   # Semi-transparent
```

---

## Quality & Compression

### JPEG quality

```bash
magick input.jpg -quality 85 output.jpg    # 85 = good balance of quality/size
magick input.jpg -quality 60 output.jpg    # Aggressive compression
```

Quality range: 1 (worst) – 100 (best). 75–85 is the practical sweet spot for web images.

### WebP quality

```bash
magick input.jpg -quality 85 output.webp
```

WebP uses the same `-quality` flag as JPEG (1–100), but the encoding is different — WebP at 85 produces smaller files than JPEG at 85 with comparable visual quality. For lossless WebP:

```bash
magick input.png -define webp:lossless=true output.webp
```

### PNG compression

```bash
magick input.png -define png:compression-level=9 output.png   # Max compression (slower)
magick input.png -define png:compression-level=6 output.png   # Default
```

PNG compression (0–9) affects file size and write speed but not visual quality.

### Strip metadata

```bash
magick input.jpg -strip output.jpg         # Remove EXIF, ICC profiles, comments
```

Combine with quality for maximum size reduction:

```bash
magick input.jpg -strip -quality 82 output.jpg
```

---

## Color & Adjustments

### Grayscale

```bash
magick input.jpg -colorspace Gray output.jpg
```

### Colorspace conversion

```bash
magick input.tiff -colorspace sRGB output.jpg    # Convert CMYK/other to sRGB for web
magick input.jpg -colorspace CMYK output.tiff    # Convert to CMYK for print workflows
```

### Brightness, contrast, saturation

```bash
magick input.jpg -brightness-contrast 10x20 output.jpg     # +10 brightness, +20 contrast
magick input.jpg -brightness-contrast -15x0 output.jpg     # Darken, no contrast change
magick input.jpg -modulate 100,130,100 output.jpg           # Brightness,Saturation,Hue (100=no change)
```

`-modulate B,S,H` — values are percentages. 100 = unchanged. 150 saturation boosts color intensity.

### Levels and gamma

```bash
magick input.jpg -level 10%,90% output.jpg         # Remap input range to expand contrast
magick input.jpg -gamma 1.5 output.jpg             # Lighten midtones
magick input.jpg -gamma 0.7 output.jpg             # Darken midtones
```

### Auto adjustments

```bash
magick input.jpg -auto-level output.jpg            # Auto-stretch levels per channel
magick input.jpg -auto-gamma output.jpg            # Auto-correct gamma
magick input.jpg -normalize output.jpg             # Stretch histogram to full range
```

### Opacity / transparency

```bash
# Make white background transparent (PNG output required)
magick input.png -fuzz 5% -transparent white output.png

# Set overall opacity
magick input.png -alpha set -channel alpha -evaluate multiply 0.5 output.png
```

`-fuzz` sets color tolerance as a percentage — useful when the background isn't a perfectly uniform color.

---

## Filters & Effects

### Blur

```bash
magick input.jpg -blur 0x3 output.jpg              # Gaussian blur, sigma=3
magick input.jpg -gaussian-blur 0x2 output.jpg     # Explicit Gaussian
magick input.jpg -rotational-blur 10 output.jpg    # Rotational/spin blur
```

`-blur RxS` — radius (0 = auto) and sigma. Larger sigma = more blur.

### Sharpen

```bash
magick input.jpg -sharpen 0x1.5 output.jpg         # Sharpen, sigma=1.5
magick input.jpg -unsharp 0x1+0.5+0 output.jpg     # Unsharp mask (common for photos)
```

### Rotation & flip

```bash
magick input.jpg -rotate 90 output.jpg             # Rotate clockwise 90°
magick input.jpg -rotate -45 output.jpg            # Rotate counter-clockwise 45°
magick input.jpg -flip output.jpg                  # Vertical flip
magick input.jpg -flop output.jpg                  # Horizontal flip
magick input.jpg -auto-orient output.jpg           # Fix rotation from EXIF orientation
```

### Border & frame

```bash
magick input.jpg -border 10x10 -bordercolor black output.jpg     # 10px solid border
magick input.jpg -frame 15x15+3+3 output.jpg                     # Raised-frame effect
```

---

## Agent Rules

1. **Never overwrite the input file** — always write to a new output path unless explicitly instructed otherwise.
2. **Check input exists** before running — `magick identify input.jpg` will error clearly if the file is missing.
3. **Infer format from extension** — no `-format` flag needed; just name the output with the right extension.
4. **Quote geometry strings** with `>`, `<`, or `!` — these are shell special characters (`>` and `<` redirect I/O, `!` triggers history expansion): `-resize '1200x>'`, `-resize '800x600!'`.
5. **Use `+repage` after crop** — omitting it leaves ghost canvas dimensions that break downstream operations.
6. **Prefer `-thumbnail` for previews** — faster than `-resize` and automatically strips metadata.
7. **Use `-density` before vector input** — for PDF and SVG, set `-density` before the input filename, not after: `magick -density 150 in.pdf out.png`. Placing it after causes ImageMagick to rasterize at the default 72 DPI first.
8. **Test with `-verbose`** — add `-verbose` to any command to see what ImageMagick is doing step by step.
9. **Use built-in help for unfamiliar options** — `magick -help` lists all options; `magick -list` subtypes (e.g. `magick -list compose`, `magick -list filter`, `magick -list font`) enumerate valid values for a given parameter.

---

## Further Reading

- [Usage guide](https://usage.imagemagick.org) — practical examples across all major feature areas
- [CLI options reference](https://imagemagick.org/script/command-line-options.php) — every flag, with syntax and defaults
