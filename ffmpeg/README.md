# ffmpeg

Convert, transcode, trim, filter, and process video and audio with the [ffmpeg](https://ffmpeg.org) CLI — plus turn video into images the agent can actually look at.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

## What it covers

- **Remux vs transcode** — the copy/re-encode decision, container/codec compatibility.
- **Encoding** — H.264, H.265/HEVC, VP9, AV1, hardware encoders, two-pass size targeting, `faststart` and `yuv420p` for web.
- **Editing** — trimming (fast and frame-accurate), multi-range cuts, concatenation, segmenting.
- **Filters** — scale, crop, pad, rotate, speed changes, text/logo/PiP overlays (borders, multiple insets), fades, crossfades and chained transitions, region blur.
- **Audio** — format conversion, loudness normalization (EBU R128), silence/volume analysis, multi-track mixing and balance, replacing tracks.
- **Multi-source editing** — normalizing drone/phone/screen footage, then composite → concat → mix as a worked pipeline.
- **Subtitles** — extract, convert, soft-mux, and burn in.
- **GIFs** — palette-based high-quality GIF creation and GIF↔MP4.
- **Seeing video** — single frames, keyframes, scene detection (including the low threshold text-only slides need), N-frames-evenly-spaced, tiled contact sheets, waveforms, and images→video.
- **Describe an unknown video** — technical summary plus a contact sheet in a few commands, without watching it.
- **Size targets** — a worked two-pass recipe that fits a video under an upload limit (tested: 12.4 MB → 1.02 MB for a 1 MB target).
- **Operations** — background execution with progress files, resumable batch loops, and Windows/Git Bash notes.

## Requirements

`ffmpeg` installed and on `PATH`. Frame/contact-sheet recipes optionally use ImageMagick's `magick`.

```bash
winget install Gyan.FFmpeg          # Windows
brew install ffmpeg                  # macOS
sudo apt install ffmpeg             # Debian/Ubuntu
```

## Canonical documentation

- [ffmpeg documentation](https://ffmpeg.org/ffmpeg.html)
- [ffmpeg filters](https://ffmpeg.org/ffmpeg-filters.html)
- Encoding guides: [H.264](https://trac.ffmpeg.org/wiki/Encode/H.264) · [H.265](https://trac.ffmpeg.org/wiki/Encode/H.265) · [VP9](https://trac.ffmpeg.org/wiki/Encode/VP9) · [AV1](https://trac.ffmpeg.org/wiki/Encode/AV1)
