---
name: ffprobe
description: Inspect and extract metadata from video, audio, and multimedia container files using the ffprobe CLI. Query streams, format info, codecs, duration, resolution, rotation, packets, frames, keyframes, and chapters in multiple output formats. Read-only — never modifies the input.
compatibility: Requires ffprobe installed and available in PATH (ships with FFmpeg).
---

# ffprobe

ffprobe is a read-only multimedia stream analyzer: it does not modify the input. It still parses potentially untrusted media and can make network requests, so keep it updated and bound expensive or remote probes.

```bash
ffprobe [OPTIONS] INPUT
```

Probe results go to stdout and diagnostics go to stderr. Always silence stderr with `-v error` (errors only) or `-v quiet`, and drop the banner with `-hide_banner`:

```bash
ffprobe -v error -hide_banner -show_format input.mp4
```

---

## Compact Summary (recommended first look)

A full `-show_streams` dump is long. This gives one line per stream plus the duration, which is usually all you need before deciding what to do:

```bash
ffprobe -v error -hide_banner \
  -show_entries format=format_name,duration,size,bit_rate:stream=index,codec_type,codec_name,width,height,avg_frame_rate,pix_fmt,sample_rate,channels:stream_tags=language \
  -of compact input.mp4
```

The same summary as JSON for parsing:

```bash
ffprobe -v error -of json -show_format -show_streams input.mp4 | jq '{
  duration: .format.duration, size: .format.size,
  streams: [.streams[] | {i: .index, type: .codec_type, codec: .codec_name,
            w: .width, h: .height, fps: .avg_frame_rate, ch: .channels, lang: .tags.language}]
}'
```

---

## Container / Format Info

`-show_format` outputs container-level metadata.

```bash
ffprobe -v error -hide_banner -show_format input.mp4
```

Key fields:
- `format_name` — the detected container (`mov,mp4,m4a,3gp`, `matroska,webm`)
- `duration` — total duration in seconds
- `size` — file size in bytes
- `bit_rate` — overall bitrate in bits/second
- `TAG:title`, `TAG:creation_time`, etc. — embedded container tags

---

## Stream Info

`-show_streams` outputs one section per stream. `codec_type` is `video`, `audio`, `subtitle`, `data`, or `attachment`.

```bash
ffprobe -v error -hide_banner -show_streams input.mp4
```

### Video stream fields

- `codec_name` — `h264`, `hevc`, `vp9`, `av1`, `prores`, `mjpeg` (`mjpeg`/`png` streams with `disposition.attached_pic=1` are cover art, not video)
- `width`, `height` — frame dimensions in pixels
- `r_frame_rate` / `avg_frame_rate` — frame rate as a fraction (`30000/1001` = 29.97 fps). Prefer `avg_frame_rate` for variable-frame-rate (phone) footage.
- `pix_fmt` — `yuv420p` (universal), `yuv420p10le` (10-bit/HDR), `yuv444p`
- `color_space`, `color_transfer`, `color_primaries` — HDR indicators (`smpte2084` = HDR10/PQ, `arib-std-b67` = HLG)
- `nb_frames` — frame count (may be absent for some containers)
- `side_data_list` → `rotation` — phone videos often store rotation here (`-90`, `90`, `180`) rather than in the pixels. Width/height are pre-rotation values, so check this before scaling or cropping.
- `sample_aspect_ratio` / `display_aspect_ratio` — anamorphic content has SAR ≠ 1:1

### Audio stream fields

- `codec_name` — `aac`, `mp3`, `opus`, `flac`, `ac3`, `eac3`, `pcm_s16le`
- `sample_rate`, `channels`, `channel_layout` (`mono`, `stereo`, `5.1(side)`)
- `bit_rate`, `sample_fmt` (`fltp`, `s16`, `s32`)

### Subtitle streams and tags

Subtitle streams have `codec_name` such as `subrip`, `ass`, `webvtt`, `mov_text` (text-based, convertible to SRT), or `hdmv_pgs_subtitle`/`dvd_subtitle` (bitmap, so they cannot be converted to SRT without OCR). Language and title live in tags:

```bash
ffprobe -v error -show_entries stream=index,codec_type,codec_name:stream_tags=language,title -of compact input.mkv
```

---

## Stream Selection

`-select_streams` restricts output. It applies to `-show_streams`, `-show_packets`, and `-show_frames`.

| Specifier | Meaning |
|-----------|---------|
| `v` | All video streams |
| `a` | All audio streams |
| `s` | All subtitle streams |
| `v:0` | First video stream |
| `a:1` | Second audio stream |
| `2` | Stream at global index 2 |

```bash
ffprobe -v error -hide_banner -show_streams -select_streams a input.mkv
```

---

## Output Formats

Set the writer with `-of` (alias `-output_format`):

| Format | Description |
|--------|-------------|
| `default` | Section-based key=value |
| `json` | JSON (best for parsing with `jq`) |
| `csv` | One line per section; `csv=p=0` drops the section name |
| `compact` | Like CSV with `key=value` pairs, one line per stream (good for quick reading) |
| `flat` | Dot-path keys (`streams.stream.0.codec_name="h264"`) |
| `ini`, `xml` | INI sections / XML |

---

## Selective Field Output

`-show_entries` limits output to specific fields. Syntax: `section=field1,field2:section2=field3`.

```bash
ffprobe -v error -show_entries format=duration,size input.mp4
ffprobe -v error -show_entries stream=codec_name,width,height -select_streams v:0 input.mp4
ffprobe -v error -show_entries format=duration:stream=index,codec_type,codec_name -of compact input.mp4
```

### Extracting a single bare value

`-of csv=p=0` with a single field yields a clean scalar, ideal for shell variables:

```bash
duration=$(ffprobe -v error -of csv=p=0 -show_entries format=duration input.mp4)
width=$(ffprobe -v error -of csv=p=0 -show_entries stream=width -select_streams v:0 input.mp4)

# WxH in one call
ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0 input.mp4
```

---

## Packets, Frames, and Keyframes

`-show_packets` outputs one section per packet (cheap, no decoding). `-show_frames` outputs per-frame data and requires decoding (slow).

```bash
# Keyframe timestamps (packet flag K)
ffprobe -v error -select_streams v:0 -show_entries packet=pts_time,flags -of csv=p=0 input.mp4 | grep K

# Frame types (I/P/B) for the first 10 seconds
ffprobe -v error -select_streams v:0 -read_intervals %10 -show_entries frame=pts_time,pict_type -of csv=p=0 input.mp4
```

The packet `flags` field is a 3-character string: `K` = keyframe, `D` = discard, `C` = corrupt, `_` = not set.

`-count_packets` adds `nb_read_packets` without a full decode. Packet count is **not a guaranteed frame count**: codec/container packetization can differ. Use `-count_frames` for `nb_read_frames` when a decoded frame count is required; it can be slow on long files.

```bash
ffprobe -v error -select_streams v:0 -count_packets -show_entries stream=nb_read_packets -of csv=p=0 input.mp4
```

---

## Chapters

```bash
ffprobe -v error -show_chapters -of json input.mkv \
  | jq -r '.chapters[] | "\(.start_time | tonumber | floor)s  \(.tags.title)"'
```

Produces no output if the file has no chapters.

---

## Time Ranges

`-read_intervals` limits probing to part of the file. Syntax: `[START]%[END|+DURATION]`.

```bash
ffprobe -v error -show_packets -read_intervals "%30" input.mp4            # first 30 s
ffprobe -v error -show_packets -read_intervals "00:05:00%+30" input.mp4   # 30 s from 5:00
ffprobe -v error -show_packets -read_intervals "00:01:00%00:01:30" input.mp4  # explicit window
```

---

## Remote and Network Sources

ffprobe accepts URLs, including HLS/DASH manifests. `-analyzeduration` (microseconds) and `-probesize` (bytes) limit analysis, not total wall time or total reading with `-show_packets`/`-show_frames`. Use protocol-supported `-rw_timeout` (microseconds) for blocked I/O plus the harness's overall timeout; use `-read_intervals` to bound packet/frame work. Network access can disclose your address or fetch nested manifest resources.

```bash
ffprobe -v error -show_streams -of json "https://example.com/playlist.m3u8"
ffprobe -v error -analyzeduration 5000000 -probesize 5000000 -show_streams rtmp://live.example.com/app/key
```

---

## Human-Readable Output

`-pretty` formats durations as `HH:MM:SS`, adds SI prefixes, and appends units. Use it for displaying to the user, not for parsing.

```bash
ffprobe -v error -pretty -show_format input.mp4
```

---

## Scripting Patterns

```bash
# Does the file have audio?
[ -n "$(ffprobe -v error -select_streams a -show_entries stream=index -of csv=p=0 input.mp4)" ] && echo yes || echo no

# Batch: duration and resolution of every file
for f in *.mp4; do
  printf '%s\t%s\t%s\n' "$f" \
    "$(ffprobe -v error -of csv=p=0 -show_entries format=duration "$f")" \
    "$(ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0 "$f")"
done

# Save full metadata
ffprobe -v error -of json -show_format -show_streams -show_chapters input.mp4 > metadata.json
```

---

## Platform Notes (Windows / Git Bash)

ffprobe runs as a native Windows executable under Git Bash and behaves normally. `2>/dev/null` works under Git Bash (`2>NUL` in cmd/PowerShell). Git Bash rewrites arguments starting with `/` into Windows paths; this helps for file paths, and `MSYS_NO_PATHCONV=1` disables it if a non-path argument is mangled.

---

## Agent Rules

1. **Always pass `-v error` (or `-v quiet`)** — stderr noise otherwise pollutes output and context.
2. **Prefer `-show_entries` plus `-of compact` or `-of csv=p=0`** over full dumps; use `-of json` with `jq` for structured parsing.
3. **Prefer `avg_frame_rate` over `r_frame_rate`** for variable-frame-rate content.
4. **Check for rotation side data** on phone footage before scaling or cropping — width and height are pre-rotation values.
5. **Avoid `-count_frames` and `-show_frames` on long files** unless needed. Use `-count_packets` or `-read_intervals` instead.
6. **ffprobe does not modify its input**, but remote access and untrusted parsing still need normal network, timeout, and sandbox precautions.
7. **Use `ffprobe -sections`** to discover section names; inspect a small JSON sample or the documentation for actual fields.

---

## Further Reading

- [ffprobe documentation](https://ffmpeg.org/ffprobe.html) — full options reference, output writers, timecode handling
- [FFmpeg stream specifiers](https://ffmpeg.org/ffmpeg.html#Stream-specifiers) — specifier grammar shared across all FFmpeg tools
