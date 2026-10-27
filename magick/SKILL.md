---
name: magick
description: Manipulate, convert, and process images from the command line using the ImageMagick 7 `magick` CLI. Covers format conversion across 200+ formats, resizing, cropping, padding, color and effects, annotation, compositing, drawing, montage/contact sheets, image comparison, redaction, and batch processing.
compatibility: Requires ImageMagick 7+ with the `magick` command available in PATH. On Windows, always use `magick`, never `convert`.
---

# magick (ImageMagick)

ImageMagick 7's `magick` command is the single entry point for still-image work: conversion, resizing, cropping, color, effects, annotation, compositing, comparison, and batch processing across 200+ formats.

```bash
magick [input-options] INPUT [processing-options] OUTPUT
```

The output format comes from the output extension (`out.webp`), or force it with a prefix (`png:out.dat`). Use `-` for stdin/stdout (`png:-`).

**Always use `magick`, never `convert`.** `convert` is the deprecated IM6 name, and on Windows it is a system disk utility (it converts FAT to NTFS). Other tools are subcommands: `magick identify`, `magick montage`, `magick compare`, `magick mogrify`, `magick composite`.

### Option order matters

Options are applied left to right, to the images loaded so far. Anything after the output filename is ignored.

```bash
magick input.jpg output.jpg -strip            # wrong: -strip ignored
magick input.jpg -strip output.jpg            # right
magick input.jpg -resize 800x -sharpen 0x1 output.jpg   # sharpen after resize (usually what you want)
```

**Settings** (`-density`, `-background`, `-fill`, `-font`, `-gravity`, `-quality`) persist until changed. **Operators** (`-resize`, `-crop`, `-annotate`) act immediately. Input settings such as `-density` and `-size` must come *before* the input they affect.

### Parentheses

Parentheses process a subset of images independently. Escape them in bash:

```bash
magick photo.jpg \( logo.png -resize 200x \) -gravity SouthEast -geometry +20+20 -composite out.jpg
```

---

## Inspecting Images

```bash
magick identify image.jpg                                    # one line: format, size, depth, file size
magick identify -format '%f %m %wx%h %[colorspace] %z-bit alpha:%A q:%Q %b\n' *.jpg
magick identify -format '%[EXIF:Orientation] %[EXIF:DateTimeOriginal]\n' photo.jpg
magick identify -verbose image.jpg | head -60                # full dump (long, so truncate it)
magick identify -format '%n\n' anim.gif                      # number of frames/pages
```

| Token | Meaning | Token | Meaning |
|-------|---------|-------|---------|
| `%f` | filename | `%m` | format (JPEG, PNG) |
| `%w` `%h` | width, height | `%b` | file size |
| `%z` | bit depth | `%A` | alpha channel (True/False/Blend) |
| `%Q` | JPEG quality estimate | `%x` `%y` | resolution (DPI) |
| `%[colorspace]` | sRGB, Gray, CMYK | `%k` | unique colors (slow) |
| `%n` | frame count | `%[EXIF:tag]` | an EXIF value |

Mean brightness (0 = black, 1 = white): `magick image.png -colorspace Gray -format '%[fx:mean]\n' info:`. For metadata beyond basics, use exiftool.

Capability lists: `magick -list format` (check HEIC/AVIF/JXL support here), `magick -list font`, `magick -list compose`, `magick -list gravity`, `magick -list filter`.

---

## Format Conversion

```bash
magick input.png output.jpg                 # alpha flattened onto white (set -background to change)
magick input.jpg output.webp
magick input.heic output.jpg                # needs a libheif delegate; check magick -list format | grep -i heic
magick input.png -background white -alpha remove -alpha off output.jpg   # explicit flatten
```

### Multi-frame formats (GIF, TIFF, PDF, ICO)

Index into frames with `[N]`, `[N-M]`, or `[N,M]`. Quote the filename so the shell leaves the brackets alone.

```bash
magick 'anim.gif[0]' first-frame.png
magick anim.gif -coalesce frames/frame_%03d.png    # all frames, fully composed
magick 'scan.tiff[2]' page3.png
```

### PDF and SVG (vector → raster)

Set `-density` (DPI) **before** the input. Afterward is too late: the image was already rasterized at 72 DPI.

```bash
magick -density 200 'document.pdf[0]' -quality 90 page1.jpg
magick -density 300 document.pdf -background white -alpha remove pages/page_%03d.png
magick -density 192 icon.svg -resize 512x512 icon.png
```

PDF input requires Ghostscript and permission under ImageMagick's security policy. A missing-delegate error differs from a policy-denied error. Inspect `magick -list policy`; do not broadly disable security restrictions to process an untrusted document. Use page bounds and resource limits for large PDFs.

### Images → PDF, and icons

```bash
magick page1.jpg page2.jpg page3.jpg output.pdf
magick *.jpg -resize 1700x2200 -gravity center -extent 1700x2200 -units PixelsPerInch -density 200 letter.pdf
magick logo.png -define icon:auto-resize=256,64,48,32,16 favicon.ico
```

---

## Resize, Crop & Extend

```bash
magick in.jpg -resize 800x600 out.jpg        # fit within 800x600, keep aspect
magick in.jpg -resize 800x out.jpg           # width 800
magick in.jpg -resize x600 out.jpg           # height 600
magick in.jpg -resize 50% out.jpg
magick in.jpg -resize '800x600!' out.jpg     # exact, distorts
magick in.jpg -resize '1600x1600>' out.jpg   # only shrink if larger (recommended default)
magick in.jpg -resize '800x600^' -gravity center -extent 800x600 out.jpg   # fill and center-crop to exact size
magick in.jpg -thumbnail 300x300 thumb.jpg   # fast resize and strip profiles; for previews
```

Quote any geometry containing `>`, `<`, `!`, or `^`: these are shell metacharacters.

### Crop

```bash
magick in.jpg -crop 400x300+100+50 +repage out.jpg               # WxH+X+Y
magick in.jpg -gravity center -crop 1000x1000+0+0 +repage out.jpg   # center crop
magick in.jpg -crop 50%x100% +repage half_%d.jpg                 # split into left/right halves
magick in.jpg -crop 3x3@ +repage tile_%d.jpg                     # 3x3 grid of tiles
magick in.png -trim +repage out.png                              # auto-crop uniform borders
magick in.jpg -fuzz 10% -trim +repage out.jpg                    # trim near-uniform borders (scans, screenshots)
```

**Always add `+repage` after `-crop` or `-trim`.** Otherwise the virtual canvas keeps the old size and offset, which breaks later operations and PNG/GIF output.

### Gravity, pad, border

`-gravity` sets the anchor for crop, extent, composite, and annotate: `NorthWest North NorthEast West Center East SouthWest South SouthEast`.

```bash
magick in.jpg -background white -gravity center -extent 1200x1200 out.jpg   # pad to square canvas
magick in.png -background none -gravity center -extent 1200x800 out.png     # transparent padding
magick in.jpg -bordercolor black -border 10 out.jpg
magick in.jpg -bordercolor white -border 20 -bordercolor '#ccc' -border 1 out.jpg   # mat + hairline
```

---

## Quality & Compression

```bash
magick in.jpg -quality 82 out.jpg                          # 75–85 is the web sweet spot
magick in.jpg -strip -interlace Plane -quality 82 out.jpg  # strip metadata; progressive JPEG
magick in.jpg -define jpeg:extent=300KB out.jpg            # largest quality under 300 KB
magick in.jpg -quality 80 out.webp                         # lossy WebP (smaller than JPEG at the same quality)
magick in.png -define webp:lossless=true out.webp
magick in.jpg -quality 50 out.avif                         # AVIF (if supported)
magick in.png -define png:compression-level=9 out.png      # PNG is lossless; level only affects size and speed
magick in.png -colors 256 png8:out.png                     # palette PNG, often 3–5x smaller for UI and screenshots
```

`-strip` removes EXIF orientation, so run `-auto-orient` **before** `-strip`, or phone photos end up sideways. It also removes ICC profiles. For an embedded wide-gamut ICC profile, convert to an actual sRGB ICC profile with `-profile /path/to/sRGB.icc` **before** stripping. `-colorspace sRGB` alone is not a substitute for ICC conversion of arbitrary profiled RGB images. Preserve the profile when accurate conversion is unavailable.

---

## Color & Adjustments

```bash
magick in.jpg -colorspace Gray out.jpg                     # grayscale
magick in.tif -colorspace sRGB out.jpg                     # CMYK → sRGB for web
magick in.jpg -brightness-contrast 10x20 out.jpg           # +10 brightness, +20 contrast
magick in.jpg -modulate 100,130,100 out.jpg                # brightness,saturation,hue (100 = unchanged)
magick in.jpg -level 10%,90% out.jpg                       # stretch levels
magick in.jpg -gamma 1.3 out.jpg                           # lighten midtones (<1 darkens)
magick in.jpg -auto-level out.jpg                          # auto stretch per channel
magick in.jpg -normalize out.jpg                           # stretch histogram
magick in.jpg -negate out.jpg                              # invert
magick in.jpg -threshold 50% out.png                       # pure black/white
```

### Transparency

```bash
magick in.png -fuzz 8% -transparent white out.png                    # make a near-white background transparent
magick in.png -alpha set -channel A -evaluate multiply 0.5 +channel out.png   # 50% opacity
magick in.png -alpha extract mask.png                                # alpha as a grayscale mask
magick in.jpg -fuzz 10% -fill none -draw 'color 0,0 floodfill' out.png   # flood-remove the background from a corner
```

`-fuzz` is the color tolerance; it must come before the operator it affects.

---

## Filters & Effects

```bash
magick in.jpg -blur 0x3 out.jpg                  # Gaussian (radius 0 = auto, sigma 3)
magick in.jpg -sharpen 0x1 out.jpg
magick in.jpg -unsharp 0x1+0.7+0.02 out.jpg      # photo sharpening after downscale
magick in.jpg -rotate 90 out.jpg                 # clockwise; non-90° angles expand the canvas (-background sets fill)
magick in.jpg -flip out.jpg                      # vertical mirror (-flop = horizontal)
magick in.jpg -auto-orient out.jpg               # apply EXIF orientation to the pixels
magick in.jpg -deskew 40% +repage out.jpg        # straighten scanned text
magick in.jpg -median 3 out.jpg                  # despeckle
```

### Pixelate / redact a region

```bash
# Pixelate a 300x100 region at +50+400
magick in.jpg \( +clone -crop 300x100+50+400 +repage -scale 5% -scale 2000% \) -geometry +50+400 -composite out.jpg
# Solid redaction box
magick in.jpg -fill black -draw 'rectangle 50,400 350,500' out.jpg
```

For true privacy, prefer solid boxes. Heavy blur and pixelation can sometimes be reversed.

---

## Compositing, Combining & Drawing

```bash
magick base.jpg overlay.png -gravity SouthEast -geometry +20+20 -composite out.jpg     # watermark
magick base.jpg \( logo.png -alpha set -channel A -evaluate multiply 0.6 +channel \) -gravity Center -composite out.jpg
magick a.jpg b.jpg c.jpg +append strip.jpg                                             # side by side
magick a.jpg b.jpg -append stacked.jpg                                                 # vertical
```

`+append` aligns to the top and `-append` to the left. Set `-gravity center` before append to center instead.

### Text

```bash
magick in.jpg -font Arial -pointsize 48 -fill white -stroke black -strokewidth 2 \
  -gravity South -annotate +0+30 'Caption text' out.jpg

# Auto-wrapped text block, sized to fit a box
magick -size 800x200 -background '#0008' -fill white -font Arial caption:'Long text that wraps automatically' \
  in.jpg +swap -gravity South -composite out.jpg
```

`label:` makes single-line text sized to fit, `caption:` wraps within `-size`, and `-annotate` draws onto an existing image. Use `magick -list font` for valid font names, or pass a `.ttf` path.

### Shapes and canvases

```bash
magick -size 1920x1080 xc:'#1a1a2e' bg.png                       # solid canvas
magick -size 1200x630 gradient:'#4f46e5'-'#9333ea' bg.png         # gradient
magick in.png -fill none -stroke red -strokewidth 4 -draw 'rectangle 100,100 400,300' annotated.png   # highlight a region
magick in.png -fill none -stroke red -strokewidth 4 -draw 'circle 250,250 250,300' annotated.png
```

Drawing a box or arrow on a screenshot is a good way to point at something in an image you return to the user.

---

## Montage (contact sheets / grids)

```bash
magick montage *.jpg -tile 4x -geometry 300x300+6+6 sheet.jpg
magick montage -label '%f' *.jpg -tile 5x -geometry 240x240+4+4 -pointsize 12 -background '#111' -fill white sheet.jpg
```

`-label` must come **before** the files it labels. `-geometry WxH+GAPX+GAPY` sets the cell size and spacing. `-tile 4x` fixes the columns, and the rows follow. Keep the total width ≤ ~1600 px if the sheet is meant to be viewed. `montage` has its own option set — don't mix standard processing options (like `-resize`) outside parenthesized image groups.

---

## Comparing Images

```bash
magick compare -metric AE -fuzz 2% a.png b.png diff.png      # differing pixels: "31124 (0.1297)" = count (fraction)
magick compare -metric RMSE a.png b.png diff.png             # error: "19297.6 (0.2945)" = raw (normalized 0–1)
magick compare -metric DSSIM a.png b.png null:               # structural dissimilarity (0 = identical)
```

The metric is printed to **stderr** with no trailing newline (capture it with `2>&1`). Use the normalized value in parentheses for thresholds: 0 means identical. Extract it with `sed -E 's/.*\(([0-9.e-]+)\).*/\1/'`. The exit code is 0 if similar, 1 if they differ, 2 on error — so don't treat 1 as a failure. Images must have identical dimensions; resize or crop first. `SSIM` semantics have changed between IM 7 releases, so prefer `DSSIM` or `RMSE`.

### Visual diff recipes

Use these for screenshot regression checks, before/after comparisons, or confirming an edit changed only what it should.

```bash
magick compare -metric AE -fuzz 3% before.png after.png diff.png 2>&1; echo   # differing-pixel count; diff.png marks changes in red
magick before.png after.png -compose difference -composite -auto-level heat.png   # intensity map of the changes
magick before.png after.png diff.png +append side_by_side.png                     # before | after | diff in one image to view
```

Turn the metric into a pass/fail decision with the normalized value (0 = identical):

```bash
metric=$(magick compare -metric RMSE a.png b.png null: 2>&1)
rc=$?
[ "$rc" -le 1 ] || { printf '%s\n' "$metric" >&2; exit "$rc"; }
d=$(printf '%s\n' "$metric" | sed -nE 's/.*\(([0-9.eE+-]+)\).*/\1/p')
[ -n "$d" ] || { echo 'Could not parse comparison metric' >&2; exit 1; }
awk -v d="$d" 'BEGIN{exit !(d < 0.02)}' && echo same || echo different
```

**Match dimensions first, because mismatches do not error.** Tested on IM 7.1.2: comparing a 400×300 image against a 200×150 one returned `44000 (0.3667)` with no warning, a confident-looking number that means nothing. Normalize before comparing with `-resize 'WxH!'` or `-extent`. To diff two video frames, extract the same timestamp from each video first.

---

## Batch Processing

### `mogrify` (in place by default — use `-path`)

`magick mogrify` **overwrites its inputs** unless you give it `-path`. Always use `-path` with an existing directory:

```bash
mkdir -p web thumbs
magick mogrify -path web -auto-orient -resize '1600x1600>' -strip -quality 82 -format jpg *.jpg *.png
magick mogrify -path thumbs -thumbnail 300x300 -format webp *.jpg
```

### Shell loop (full control over names)

```bash
mkdir -p out
for f in *.png; do magick "$f" -resize 50% "out/${f%.*}.webp"; done
```

### Web-optimize a folder of photos

Resize, convert to WebP, and strip metadata (including GPS) in one pass. Order matters: `-auto-orient` must run **before** `-strip`, or phone photos end up sideways.

```bash
mkdir -p web
for f in *.jpg *.jpeg *.png *.heic; do
  [ -e "$f" ] || continue                       # skip patterns that matched nothing
  magick "$f" -auto-orient -resize '2048x2048>' -colorspace sRGB -strip -quality 82 "web/${f%.*}.webp"
done
magick identify -format '%f %wx%h %b\n' web/*   # verify sizes and bytes
```

Tested: a 3000×2000 JPEG with an EXIF rotation tag and GPS came out upright (1365×2048) with the GPS removed. `.heic` input needs a libheif delegate; the `[ -e ]` guard skips it quietly if there are no HEIC files, but a build without HEIC support fails on real ones (check `magick -list format | grep -i heic`).

`-strip` also removes copyright and artist tags. To put ownership back (and nothing else) after stripping, use [exiftool](https://exiftool.org) if it's installed:

```bash
exiftool -overwrite_original -Copyright="© 2026 Name" -Artist="Name" web/*.webp
```

---

## Platform Notes (Windows / Git Bash)

- **Always use `magick`, never `convert`** on Windows — `convert` is the system FAT→NTFS disk utility. Subcommands are `magick identify`, `magick montage`, `magick compare`, `magick mogrify`.
- **Fonts:** `-font Arial` works on Windows; list available fonts with `magick -list font`. On Linux, pass a `.ttf` path if name lookup fails.
- **Delegates:** HEIC/AVIF/JXL/PDF/RAW support depends on the build. Check `magick -list format` before promising them; PDF/EPS needs Ghostscript installed.
- **Git Bash path conversion:** arguments starting with `/` are rewritten to Windows paths; `MSYS_NO_PATHCONV=1` disables it if a non-path argument is mangled.

---

## Agent Rules

1. **Never overwrite the input or an existing user output without approval.** Use a fresh destination, and always pass a distinct output directory with `-path` to `mogrify`. Check for basename collisions when converting multiple formats.
2. **Use `magick`, never `convert`.**
3. **Put settings before the input when they affect reading** (`-density`, `-size`), and put every operation before the output filename.
4. **Quote geometry strings** containing `> < ! ^` and quote `'file[0]'` frame selectors.
5. **Add `+repage` after `-crop` and `-trim`.**
6. **Run `-auto-orient` before `-strip`** on photos.
7. **Check for delegates** (`magick -list format`) before promising HEIC, AVIF, JXL, PDF, or RAW support.
8. **Verify the result** with `magick identify -format` and view a downscaled copy for visual edits.
9. **Use `magick -help` and `magick -list <type>`** rather than guessing at an option's syntax.

---

## Further Reading

- [Usage examples](https://usage.imagemagick.org) — practical examples across every feature area
- [Command-line options](https://imagemagick.org/script/command-line-options.php) — every flag, with syntax and defaults
- [Format and print escapes](https://imagemagick.org/script/escape.php) — the `-format` tokens
