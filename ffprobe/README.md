# ffprobe

Inspect and extract metadata from video, audio, and multimedia containers with the [ffprobe](https://ffmpeg.org/ffprobe.html) CLI. Read-only — it never modifies the input.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

## What it covers

- **Compact summary** — a single one-liner that reports container, duration, and every stream.
- **Streams & format** — codecs, resolution, frame rate, pixel format, HDR indicators, rotation side-data, channel layout, subtitle codecs.
- **Selective output** — `-show_entries`, `-of json`/`compact`/`csv`, extracting bare scalar values for shell variables.
- **Packets, frames, keyframes** — keyframe timestamps, I/P/B frame types, and packet counts versus decoded frame counts.
- **Chapters, time ranges, and remote/HLS sources.**
- **Scripting patterns** — audio detection, batch duration/resolution tables, full-metadata dumps, and Windows/Git Bash notes.

## Requirements

`ffprobe` installed and on `PATH` (it ships with FFmpeg — see the [ffmpeg](../ffmpeg/README.md) install commands).

## Canonical documentation

- [ffprobe documentation](https://ffmpeg.org/ffprobe.html)
- [FFmpeg stream specifiers](https://ffmpeg.org/ffmpeg.html#Stream-specifiers)
