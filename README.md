# agent-skills

A collection of standalone skills for coding agents. Compatible with [pi](https://pi.dev) and [Claude Code](https://claude.ai/code).

Each skill is self-contained: it lives in its own directory, is useful on its own, and requires no other skill from this repo. Every skill's full content is in a single `SKILL.md` — no accessory reference files — and each links out to the canonical documentation for its tool. Each skill directory also has a `README.md` (linked below) with a short overview, requirements, and install commands.

## Skills

### Media

| Skill | What it does |
|-------|--------------|
| [ffmpeg](ffmpeg/README.md) | Convert, transcode, trim, filter, and process video and audio; extract frames and contact sheets so an agent can "see" a video. Encoding (H.264/H.265/VP9/AV1), overlays, audio loudness, subtitles, GIFs, background jobs. Uses [ffmpeg](https://ffmpeg.org). |
| [ffprobe](ffprobe/README.md) | Inspect video/audio containers read-only — streams, codecs, duration, resolution, rotation, keyframes, chapters — in compact, JSON, or CSV output. Uses [ffprobe](https://ffmpeg.org/ffprobe.html). |
| [magick](magick/README.md) | Manipulate and convert images across 200+ formats — resize, crop, color, effects, annotation, compositing, montage, comparison, redaction, batch. Uses [ImageMagick](https://imagemagick.org) 7. |
| [exiftool](exiftool/README.md) | Read, write, copy, and strip metadata (EXIF/XMP/IPTC/GPS/QuickTime) in images, video, and documents; rename and sort files by capture date. Uses [exiftool](https://exiftool.org). |
| [tesseract](tesseract/README.md) | OCR text from screenshots, scans, photos, and rasterized PDFs — page segmentation, searchable PDFs, word-level confidence, preprocessing. Uses [tesseract](https://github.com/tesseract-ocr/tesseract). |

Each of these five works entirely on its own. They also compose well, and the tables below are a quick guide to which one to reach for.

#### Which media skill do I need?

| I want to… | Use |
|------------|-----|
| Know what's inside a video or audio file (codec, duration, streams, rotation) | [ffprobe](ffprobe/README.md) |
| Convert, trim, compress, filter, or extract frames/audio/GIFs from video or audio | [ffmpeg](ffmpeg/README.md) |
| Convert, resize, crop, annotate, composite, or compare still images | [magick](magick/README.md) |
| Read, write, or strip metadata (EXIF, GPS, dates), or rename files by capture date | [exiftool](exiftool/README.md) |
| Get text out of an image, scan, screenshot, or video frame | [tesseract](tesseract/README.md) |

#### Common jobs and where they're documented

Every recipe below lives in one skill's `SKILL.md`, with the commands inline. The last column lists other tools the recipe can use, so you know what to install.

| Job | Documented in | Also uses |
|-----|---------------|-----------|
| Describe an unknown video (summary + contact sheet) | ffmpeg | exiftool (optional) |
| Shrink a video to fit a size limit | ffmpeg | — |
| Pull text out of a slide deck or screen recording | tesseract | ffmpeg, magick |
| OCR a screenshot, photo of a page, or scanned PDF | tesseract | magick (preprocessing, PDFs) |
| Web-optimize a folder of photos | magick | exiftool (restore copyright) |
| Visual diff of two screenshots or images | magick | ffmpeg (to grab video frames) |
| Privacy-scrub photos and videos before sharing | exiftool | magick, ffmpeg |
| Sort photos and videos into folders by capture date | exiftool | — |
| Keep metadata through a format conversion | exiftool | magick, ffmpeg |

### Diagrams & charts

| Skill | What it does |
|-------|--------------|
| [drawio](drawio/README.md) | Create and edit draw.io diagrams (`.drawio`) by writing XML directly — flowcharts, architecture, ER, UML, network diagrams. No dependencies to author. |
| [charts](charts/README.md) | Generate publication-quality charts (bar, line, pie, scatter, radar, funnel, gauge, treemap, heatmap, candlestick, sankey) as SVG/PNG via [charts-cli](https://github.com/Michaelliv/charts-cli) and ECharts configs. |

### System & time

| Skill | What it does |
|-------|--------------|
| [atd](atd/README.md) | Schedule one-time deferred command execution with the Unix `at` command — delayed operations and follow-up actions. |
| [date](date/README.md) | Ground-truth current time, date arithmetic, and calendar navigation via the `date` command, so agents don't rely on unreliable internal time. |

### Security

| Skill | What it does |
|-------|--------------|
| [nmap](nmap/README.md) | Network reconnaissance for authorized security testing with [nmap](https://nmap.org) and nmap-vulners — discovery, port/service scanning, OS fingerprinting, NSE, CVE lookup, agent-safe progressive scanning. |

## Installing a skill

Skills install into a directory named after the skill. Copy a skill's `SKILL.md` into that directory under your agent's skills folder:

```bash
# Claude Code
mkdir -p ~/.claude/skills/ffmpeg
cp ffmpeg/SKILL.md ~/.claude/skills/ffmpeg/

# pi
mkdir -p ~/.pi/agent/skills/ffmpeg
cp ffmpeg/SKILL.md ~/.pi/agent/skills/ffmpeg/
```

Each skill lists its own install/requirements in its `README.md`. Re-copy after updating a skill.
