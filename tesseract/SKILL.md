---
name: tesseract
description: Extract text from images, screenshots, scans, photos of documents, and rasterized PDFs using the tesseract OCR CLI. Covers page segmentation modes, output formats (text, searchable PDF, TSV word boxes with confidence), image preprocessing, character restriction, and languages. Use whenever exact characters matter — serial numbers, code, long passages, form fields.
compatibility: Requires tesseract 5+ installed and available in PATH. Image preprocessing examples use ImageMagick's `magick` (optional but recommended); the video-to-text recipe also uses ffmpeg (optional).
---

# tesseract (OCR)

Tesseract 5 extracts text from images using an LSTM neural-network engine. Accuracy depends far more on **input preparation** than on options: clean, high-contrast, upright text at a sensible size wins over any amount of flag tuning.

```bash
tesseract IMAGE OUTPUTBASE [-l LANGS] [--psm N] [--oem N] [--dpi N] [-c VAR=VALUE] [CONFIG...]
```

- `OUTPUTBASE` gets an extension per output config: `out` → `out.txt`. Use `-` (or `stdout`) to print to stdout.
- Input formats: PNG, JPEG, TIFF (including multi-page), BMP, PNM, GIF, WebP. **There is no PDF input.** Rasterize PDFs with ImageMagick first.
- Progress and warnings ("Estimating resolution as 96") go to stderr. Silence them with `2>/dev/null`, or pass `--dpi`.

```bash
tesseract image.png - 2>/dev/null           # text to stdout
tesseract image.png out                      # writes out.txt
tesseract image.png out -l eng+spa           # multiple languages
tesseract --list-langs                       # installed language data
tesseract --help-psm                         # page segmentation modes
tesseract --version                          # version and build info
```

---

## Page Segmentation Modes (`--psm`)

Choosing the right layout assumption is the biggest lever after preprocessing.

| PSM | Assumes | Use for |
|-----|---------|---------|
| 3 | Fully automatic layout (default) | Documents, pages, articles |
| 4 | A single column of text of variable sizes | Receipts, simple lists |
| 6 | A single uniform block of text | Cropped paragraphs, code blocks, terminal screenshots |
| 7 | A single text line | Cropped titles, captions, form fields |
| 8 | A single word | Labels, buttons |
| 10 | A single character | Single glyphs |
| 11 | Sparse text, no particular order | UI screenshots, signs, diagrams, photos with scattered text |
| 12 | Sparse text with orientation detection | Scattered text that may be rotated |
| 13 | Raw line; bypasses tesseract's heuristics | Single lines that 7 garbles |
| 0 | Orientation and script detection only | Find the rotation (needs `osd.traineddata`) |
| 1 | Auto layout + OSD | Pages that may be rotated |

If a result is empty or garbled, crop to the region of interest and retry with `--psm 6`, `7`, or `11` before tuning anything else.

`--oem` selects the engine: `1` = LSTM only, `3` = default (LSTM). `0` (legacy) works only with the legacy `tessdata` models, which are not shipped in `tessdata_fast` or `tessdata_best`.

---

## Output Formats

Pass config names after the output base; several can be combined in one run.

```bash
tesseract scan.png out txt                   # out.txt (default)
tesseract scan.png out pdf                   # out.pdf: searchable PDF (image + invisible text layer)
tesseract scan.png out txt pdf               # both
tesseract scan.png out tsv                   # out.tsv: word boxes and confidences
tesseract scan.png out hocr                  # out.hocr: HTML with bounding boxes
tesseract scan.png out alto                  # out.xml: ALTO XML
```

### TSV: word positions and confidence

Columns: `level page_num block_num par_num line_num word_num left top width height conf text`. Level 5 rows are words. `conf` runs 0–100, and `-1` marks structural rows.

```bash
# Words with confidence below 60 (likely errors), plus their boxes
tesseract shot.png - --psm 11 tsv 2>/dev/null | awk -F'\t' 'NR>1 && $1==5 && $11<60 {print $11, $7","$8, $12}'

# Mean word confidence (a quick quality score for an OCR pass)
tesseract shot.png - tsv 2>/dev/null | awk -F'\t' 'NR>1 && $1==5 && $12!="" {s+=$11; n++} END {if(n) printf "%.1f over %d words\n", s/n, n}'

# Find where a word appears (to crop or annotate around it)
tesseract shot.png - tsv 2>/dev/null | awk -F'\t' '$12=="Submit" {print "x="$7, "y="$8, "w="$9, "h="$10}'
```

Mean confidence above ~85 usually means clean output; below ~70, preprocess and retry.

---

## Preprocessing

Tesseract works best on dark text over a light, uniform background, with capital letters roughly **20–40 px tall** (around 300 DPI for print). These fixes use ImageMagick's `magick`; any image editor that can grayscale, upscale, and threshold works too.

| Problem | Fix |
|---------|-----|
| Small text (UI screenshots, low-res scans) | `-resize 200%` or `300%` (use `-filter Lanczos` for screenshots) |
| Color / photo background | `-colorspace Gray` then `-normalize` or `-auto-level` |
| Uneven lighting, shadows | `-lat 25x25-5%` (local adaptive threshold), or `-threshold 55%` on even scans |
| Light text on dark background (dark-mode UIs, terminals) | `-negate`, if the raw result is poor. Clean inverted text often OCRs fine as-is on tesseract 5 (tested), so try without it first and compare |
| Skewed scan | `-deskew 40% +repage` |
| Rotated 90°/180° | `-rotate 90`, or detect with `--psm 0` |
| Speckle noise | `-median 2` or `-despeckle` |
| Text touching the image edge | `-bordercolor white -border 20` |
| Huge image, text in one spot | Crop to the region first (`-crop WxH+X+Y +repage`) |

```bash
# General-purpose cleanup for photos of documents and scans
magick input.jpg -auto-orient -colorspace Gray -normalize -deskew 40% +repage -resize 150% -sharpen 0x1 prep.png
tesseract prep.png - --psm 3 2>/dev/null

# Dark-mode screenshot
magick shot.png -colorspace Gray -negate -resize 200% -bordercolor white -border 20 prep.png
tesseract prep.png - --psm 6 2>/dev/null

# Pipe without a temp file
magick shot.png -colorspace Gray -resize 200% png:- | tesseract stdin - --psm 6 2>/dev/null
```

Binarize (threshold) only when grayscale isn't enough. The LSTM engine binarizes internally and often does better on clean grayscale. Compare confidence with and without preprocessing.

---

## PDFs and Multi-Page Input

```bash
# Scanned PDF → text (rasterize at 300 DPI, flatten on white, grayscale)
mkdir -p pages
magick -density 300 scan.pdf -background white -alpha remove -colorspace Gray pages/p_%03d.png
for p in pages/p_*.png; do tesseract "$p" - --psm 3 2>/dev/null; printf '\f'; done > scan.txt

# Scanned PDF → searchable PDF (one combined output via a file list)
ls pages/p_*.png > pages.txt
tesseract pages.txt searchable -l eng pdf 2>/dev/null       # → searchable.pdf

# Multi-page TIFF works directly
tesseract scan.tiff out pdf
```

When the input is a `.txt` file, tesseract treats it as a list of image paths, one per line, and produces combined output. Born-digital PDFs already have a text layer and don't need OCR. If a text extractor such as `pdftotext` is available, try it first.

---

## Slide Decks and Screen Recordings (video → text)

To pull text out of a talk, demo, or screen recording, extract one frame per visual change, drop near-duplicates, then OCR what's left. This needs ffmpeg for frame extraction and ImageMagick for duplicate detection; the OCR step is plain tesseract.

```bash
mkdir -p slides

# 1. One frame per visual change (plus frame 0). Low threshold on purpose; see note below.
ffmpeg -y -nostdin -i talk.mp4 -vf "select='eq(n,0)+gt(scene,0.02)',scale=1920:-2" -fps_mode vfr slides/s_%03d.png

# 2. Drop near-duplicates: compare each frame to the last one kept
prev=""; for f in slides/s_*.png; do
  if [ -n "$prev" ]; then
    d=$(magick compare -metric RMSE "$prev" "$f" null: 2>&1 | sed -E 's/.*\(([0-9.e-]+)\).*/\1/')
    awk -v d="$d" 'BEGIN{exit !(d < 0.02)}' && { rm "$f"; continue; }
  fi; prev="$f"
done

# 3. OCR each remaining slide
for f in slides/s_*.png; do
  echo "=== $f"; magick "$f" -colorspace Gray -resize 150% png:- | tesseract stdin - --psm 3 2>/dev/null
done > slides.txt
```

**Use a low scene threshold (0.02), not the usual 0.15–0.3.** Tested on a deck of text-only slides on white backgrounds: threshold `0.15` caught 1 of 3 slides, `0.05` caught 2, and `0.03` or lower caught all 3. Text on a plain background barely changes the pixel difference between slides, so the default thresholds used for camera cuts miss them. A low threshold can let in extra near-identical frames (cursor movement, compression noise), which is exactly what the duplicate-removal step in the middle cleans up. On that test deck the full pipeline returned the exact text of every slide.

To align each slide's text with the timeline, list the change timestamps with the same threshold:

```bash
ffmpeg -nostdin -hide_banner -i talk.mp4 -vf "select='gt(scene,0.02)',showinfo" -an -f null - 2>&1 | grep -o 'pts_time:[0-9.]*' | cut -d: -f2
```

Frame 0 is always kept, so the list is one shorter than the slide count. Raise the duplicate threshold (`0.02` in the `awk` test) if near-identical slides survive; lower it if distinct slides with small edits get merged. Slides with images or colored backgrounds usually need a higher scene threshold than text-only ones, so check the frame count before OCRing.

---

## Restricting Characters and Layout Hints

```bash
tesseract code.png - --psm 7 -c tessedit_char_whitelist=0123456789 2>/dev/null           # digits only
tesseract plate.png - --psm 7 -c tessedit_char_whitelist=ABCDEFGHJKLMNPRSTUVWXYZ0123456789 2>/dev/null
tesseract table.png - --psm 6 -c preserve_interword_spaces=1 2>/dev/null                 # keep column alignment
tesseract shot.png - --psm 6 -c tessedit_char_blacklist='|' 2>/dev/null                   # common table-border noise
```

Whitelists help with fields of known format, but they force every glyph into the allowed set, so a misread becomes a confident wrong answer.

---

## Languages

Language data files (`eng.traineddata`, `deu.traineddata`, …) live in the `tessdata` directory. Check what's installed with `tesseract --list-langs`. Combine languages with `+` (`-l eng+fra`); order matters slightly, so put the dominant language first. Scripts: `-l script/Latin`, `script/Cyrillic` (if installed). If a language is missing, tell the user rather than guessing. Models come from [tessdata_fast](https://github.com/tesseract-ocr/tessdata_fast) (faster, the usual default) and [tessdata_best](https://github.com/tesseract-ocr/tessdata_best) (more accurate). Point tesseract at a custom location with `--tessdata-dir DIR` or the `TESSDATA_PREFIX` environment variable.

---

## Limits

- Handwriting: poor accuracy; don't rely on it.
- Stylized or low-contrast text, text on busy photos, curved text: expect errors. Crop and preprocess.
- Tables: text comes out, but structure does not. Use `tsv` coordinates or `preserve_interword_spaces` to reconstruct columns.
- Math, tiny subscripts, and mixed-direction text: unreliable.
- Viewing the image is often better than OCR for gist and layout. OCR is better for exact transcription of long or precise text. Combine them: OCR, then view the image to spot-check low-confidence words.

---

## Platform Notes (Windows / Git Bash)

- **Null device:** `2>/dev/null` works under Git Bash. In native `cmd`/PowerShell use `2>NUL`.
- **ImageMagick on Windows:** always invoke it as `magick`, never `convert` (on Windows `convert` is the system disk utility). ImageMagick 7 is required for the `magick` entry point.
- **tessdata location:** the UB-Mannheim Windows installer sets `TESSDATA_PREFIX` and installs language packs you select during setup. Re-run the installer to add languages, or drop `.traineddata` files into the `tessdata` folder.
- **Git Bash path conversion:** arguments starting with `/` may be rewritten to Windows paths. File paths are fine; if a non-path option is mangled, prefix the command with `MSYS_NO_PATHCONV=1`.

---

## Agent Rules

1. **Send stderr to the null device** (`2>/dev/null`) or capture it separately so warnings don't mix into the text.
2. **Preprocess before tuning options.** Grayscale, upscale small text, invert dark mode, crop to the region.
3. **Pick `--psm` deliberately:** 3 for pages, 6 for a block, 7 for a line, 11 for scattered UI or sign text.
4. **Check confidence via `tsv`** when accuracy matters, and report uncertainty instead of presenting garbled output as fact.
5. **Rasterize PDFs at 300 DPI** (tesseract can't read PDFs); flatten transparency onto white.
6. **Verify critical values** (numbers, names, codes) by viewing the cropped region yourself.
7. **Write preprocessed intermediates to a scratch location** and clean them up afterward.
8. **Never overwrite the input.** Preprocessing writes a new file.

---

## Further Reading

- [tesseract documentation and wiki](https://github.com/tesseract-ocr/tesseract/blob/main/README.md)
- [Command-line usage](https://tesseract-ocr.github.io/tessdoc/Command-Line-Usage.html)
- [Improving OCR quality](https://tesseract-ocr.github.io/tessdoc/ImproveQuality.html)
- [tessdata_fast](https://github.com/tesseract-ocr/tessdata_fast) and [tessdata_best](https://github.com/tesseract-ocr/tessdata_best) language models
