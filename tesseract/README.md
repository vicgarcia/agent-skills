# tesseract

Extract text from images, screenshots, scans, photos of documents, and rasterized PDFs with the [tesseract](https://github.com/tesseract-ocr/tesseract) OCR CLI.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

## What it covers

- **Page segmentation modes** — picking the right `--psm` for pages, blocks, lines, and scattered UI text.
- **Output formats** — plain text, searchable PDF, and TSV word boxes with per-word confidence scores.
- **Quality scoring** — reading confidence from TSV to flag likely errors instead of presenting garbled text as fact.
- **Preprocessing** — grayscale, upscaling, deskew, inversion for dark mode, thresholding, and cropping (via ImageMagick).
- **PDFs & multi-page** — rasterizing PDFs at 300 DPI and combining pages into one searchable output.
- **Video → text** — extract one frame per slide or screen change, drop near-duplicates, and OCR each one, with timestamps (uses ffmpeg and ImageMagick).
- **Character restriction** — whitelists/blacklists and column-preserving layout hints.
- **Languages** — multi-language OCR, scripts, and installing extra `traineddata` models.
- **Limits & platform notes** — where OCR is unreliable, and Windows/Git Bash specifics.

## Requirements

tesseract **5+** installed and on `PATH`. Image preprocessing examples use ImageMagick's `magick` (optional but recommended).

```bash
winget install UB-Mannheim.TesseractOCR     # Windows (select language packs in the installer)
brew install tesseract                        # macOS
sudo apt install tesseract-ocr               # Debian/Ubuntu
```

Extra language models: [tessdata_fast](https://github.com/tesseract-ocr/tessdata_fast) (default) and [tessdata_best](https://github.com/tesseract-ocr/tessdata_best).

## Canonical documentation

- [tesseract repository](https://github.com/tesseract-ocr/tesseract/blob/main/README.md)
- [Command-line usage](https://tesseract-ocr.github.io/tessdoc/Command-Line-Usage.html)
- [Improving OCR quality](https://tesseract-ocr.github.io/tessdoc/ImproveQuality.html)
