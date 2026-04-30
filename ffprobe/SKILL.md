---
name: ffprobe
description: Inspect and extract metadata from video, audio, and multimedia container files using the ffprobe CLI. Query streams, format info, packets, frames, and chapters in multiple output formats.
compatibility: Requires ffprobe installed and available in PATH (ships with FFmpeg).
---

# ffprobe

ffprobe is a read-only multimedia stream analyzer. It never modifies the input file.

```bash
ffprobe [OPTIONS] INPUT_FILE
```

Probe results go to stdout; diagnostics go to stderr. Control stderr with `-v`:

```bash
ffprobe -hide_banner input.mp4
ffprobe -v error -hide_banner input.mp4      # Suppress all but errors
ffprobe -v quiet -hide_banner input.mp4      # Suppress everything on stderr
```

---

## Container / Format Info

`-show_format` outputs container-level metadata: format name, duration, file size, bitrate, and embedded tags.

```bash
ffprobe -v quiet -hide_banner -show_format input.mp4
```

Key fields:
- `format_name` — detected container format (e.g., `mov,mp4,m4a,3gp`, `matroska`)
- `duration` — total duration in seconds
- `size` — file size in bytes
- `bit_rate` — overall bitrate in bits/second
- `TAG:title`, `TAG:date`, etc. — embedded container tags

---

## Stream Info

`-show_streams` outputs one section per stream with codec, timing, and format details.

```bash
ffprobe -v quiet -hide_banner -show_streams input.mp4
```

`codec_type` is always `video`, `audio`, `subtitle`, `data`, or `attachment`.

### Video stream fields

- `codec_name` — codec identifier (`h264`, `hevc`, `vp9`, `av1`)
- `width`, `height` — frame dimensions in pixels
- `r_frame_rate` — frame rate as a fraction (`30000/1001` = 29.97 fps)
- `avg_frame_rate` — average frame rate; more reliable for VFR content
- `pix_fmt` — pixel format (`yuv420p`, `yuv420p10le`, etc.)
- `color_space`, `color_transfer`, `color_primaries` — HDR/color metadata
- `nb_frames` — frame count; may be absent for some containers
- `duration` — stream duration in seconds

### Audio stream fields

- `codec_name` — codec identifier (`aac`, `mp3`, `opus`, `flac`, `pcm_s16le`)
- `sample_rate` — samples per second
- `channels` — channel count
- `channel_layout` — layout name (`stereo`, `5.1`, `7.1`)
- `bit_rate` — stream bitrate
- `sample_fmt` — sample format (`fltp`, `s16`, `s32`)

### Subtitle streams and tags

Subtitle streams have `codec_name` (`subrip`, `ass`, `webvtt`, `hdmv_pgs_subtitle`) and typically `TAG:language`. Most streams also carry `TAG:title` and `TAG:handler_name`, targetable with `-show_entries stream_tags=language,title`.

---

## Stream Selection

`-select_streams` restricts output to a specific stream type or index. Applies to `-show_streams`, `-show_packets`, and `-show_frames`.

| Specifier | Meaning |
|-----------|---------|
| `v` | All video streams |
| `a` | All audio streams |
| `s` | All subtitle streams |
| `v:0` | First video stream |
| `a:1` | Second audio stream |
| `2` | Stream at global index 2 |

```bash
ffprobe -v quiet -hide_banner -show_streams -select_streams v input.mp4
ffprobe -v quiet -hide_banner -show_streams -select_streams a input.mp4
ffprobe -v quiet -hide_banner -show_streams -select_streams v:0 input.mp4
```

---

## Output Formats

Use `-of` (alias `-output_format`) to change from the `default` section-based format.

```bash
ffprobe -v quiet -hide_banner -show_streams -of json input.mp4
```

| Format | Description |
|--------|-------------|
| `default` | Section-based key=value |
| `json` | JSON |
| `csv` | One line per section, comma-separated |
| `compact` | Like CSV but configurable delimiter |
| `flat` | Hierarchical dot-path keys (`streams.stream.0.codec_name="h264"`) |
| `ini` | INI-style sections |
| `xml` | XML with optional XSD-compliant fully-qualified names |

---

## Selective Field Output

`-show_entries` limits output to specific fields. Syntax: `section=field1,field2:section2=field3`.

```bash
ffprobe -v quiet -hide_banner -show_entries format=duration,size input.mp4

ffprobe -v quiet -hide_banner -show_entries stream=codec_name,width,height \
  -select_streams v:0 input.mp4

ffprobe -v quiet -hide_banner \
  -show_entries format=duration:stream=codec_name,codec_type input.mp4
```

### Extracting a single bare value

`-of csv=p=0` with a single `-show_entries` field produces a clean scalar with no labels or section wrappers:

```bash
ffprobe -v quiet -hide_banner -of csv=p=0 \
  -show_entries format=duration input.mp4

ffprobe -v quiet -hide_banner -of csv=p=0 \
  -show_entries stream=width -select_streams v:0 input.mp4

duration=$(ffprobe -v quiet -hide_banner -of csv=p=0 \
  -show_entries format=duration input.mp4)
```

---

## Packet Info

`-show_packets` outputs one section per packet: timestamps, duration, size, stream index, and flags.

```bash
ffprobe -v quiet -hide_banner -show_packets input.mp4
ffprobe -v quiet -hide_banner -show_packets -select_streams v:0 input.mp4
```

Key fields: `pts_time`, `dts_time`, `duration_time`, `size`, `stream_index`, `flags`.

`flags` is a 3-character string. The first character is `K` for a keyframe or `_` otherwise; the remaining positions indicate discard (`D`) and corrupt (`C`) status.

`-show_data` appends a hex dump of the raw packet payload.

`-count_packets` adds `nb_read_packets` to the stream section. Requires reading the full file.

---

## Frame Info

`-show_frames` outputs per-frame metadata and requires decoding.

```bash
ffprobe -v quiet -hide_banner -show_frames -select_streams v:0 input.mp4
```

Key fields: `media_type`, `pts_time`, `pkt_dts_time`, `pict_type` (I/P/B), `key_frame`, `width`, `height`, `pix_fmt`.

`-count_frames` adds `nb_read_frames` per stream via a full decode pass — slow on long files.

---

## Chapters

`-show_chapters` outputs chapter id, start/end times, and title. Produces no output if the file has no chapters.

```bash
ffprobe -v quiet -hide_banner -show_chapters input.mkv
ffprobe -v quiet -hide_banner -show_chapters -of json input.mkv
```

Key fields: `id`, `start_time`, `end_time`, `TAG:title`.

---

## Time Ranges

`-read_intervals` limits probing to a portion of the file. Syntax: `[START]%END` — START defaults to the beginning when omitted. END can be a timestamp or a `+offset` relative to START.

```bash
# First 30 seconds
ffprobe -v quiet -hide_banner -show_packets -read_intervals "%30" input.mp4

# 30-second window starting at 5 minutes
ffprobe -v quiet -hide_banner -show_packets -read_intervals "00:05:00%+30" input.mp4

# Explicit window: 1:00 to 1:30
ffprobe -v quiet -hide_banner -show_packets -read_intervals "00:01:00%00:01:30" input.mp4
```

---

## Remote and Network Sources

ffprobe accepts URLs anywhere a file path is accepted.

```bash
ffprobe -v quiet -hide_banner -show_streams https://example.com/video.mp4
ffprobe -v quiet -hide_banner -show_streams -of json "http://example.com/playlist.m3u8"
```

For HLS and DASH manifests, ffprobe follows the playlist and reports stream info from the segments. For live streams, use `-analyzeduration` and `-probesize` to cap how much data is read:

```bash
ffprobe -v quiet -hide_banner \
  -analyzeduration 5000000 -probesize 5000000 \
  -show_streams rtmp://live.example.com/stream/key
```

`-analyzeduration` is in microseconds; `-probesize` is in bytes.

---

## Human-Readable Output

Values are raw by default (fractional seconds, bytes, fractional frame rates). `-pretty` applies all four formatting helpers:

```bash
ffprobe -v quiet -hide_banner -pretty -show_format input.mp4
```

- `-sexagesimal` — times as `HH:MM:SS.microseconds`
- `-prefix` — SI prefixes on numeric values (k, M, G)
- `-byte_binary_prefix` — binary prefixes for byte values (Ki, Mi, Gi)
- `-unit` — appends units to values (`bit/s`, `byte`, etc.)

---

## Scripting Patterns

### JSON + jq

```bash
ffprobe -v quiet -hide_banner -of json -show_format input.mp4 \
  | jq '.format.duration'

ffprobe -v quiet -hide_banner -of json -show_streams input.mp4 \
  | jq '[.streams[] | {type: .codec_type, codec: .codec_name}]'

ffprobe -v quiet -hide_banner -of json -show_streams -select_streams v:0 input.mp4 \
  | jq '.streams[0] | "\(.width)x\(.height)"'
```

### Batch inspection

```bash
for f in *.mp4; do
  dur=$(ffprobe -v quiet -hide_banner -of csv=p=0 \
    -show_entries format=duration "$f")
  echo "$f: $dur"
done
```

### Check for an audio stream

```bash
has_audio=$(ffprobe -v quiet -hide_banner -of csv=p=0 \
  -show_entries stream=codec_type -select_streams a input.mp4)
[ -n "$has_audio" ] && echo "has audio" || echo "no audio"
```

### Check codec and resolution

```bash
codec=$(ffprobe -v quiet -hide_banner -of csv=p=0 \
  -show_entries stream=codec_name -select_streams v:0 input.mp4)
width=$(ffprobe -v quiet -hide_banner -of csv=p=0 \
  -show_entries stream=width -select_streams v:0 input.mp4)

if [ "$codec" != "h264" ] || [ "$width" -gt 1920 ]; then
  echo "Unexpected format: $codec ${width}px"
fi
```

### Save full metadata

```bash
ffprobe -v quiet -hide_banner -of json \
  -show_format -show_streams -show_chapters input.mp4 > metadata.json
```

### List chapters

```bash
ffprobe -v quiet -hide_banner -of json -show_chapters input.mkv \
  | jq -r '.chapters[] | "\(.start_time | tonumber | floor)s  \(.tags.title)"'
```

---

## Agent Rules

1. **Always use `-v quiet -hide_banner`** — stderr output contaminates stdout in scripts and pipelines.
2. **Prefer `-of json` for structured data** — parse with `jq` rather than grepping text output.
3. **Use `-select_streams` to limit scope** — avoids unnecessary output and simplifies parsing.
4. **Avoid `-count_frames` unless required** — forces a full decode pass; slow on long files. `nb_frames` from `-show_streams` is usually sufficient.
5. **Use `-show_entries` to minimize output** — faster and easier to parse than full section dumps.
6. **ffprobe never modifies files** — safe to run on any input without backups.
7. **Prefer `avg_frame_rate` over `r_frame_rate`** — `r_frame_rate` can be misleading for VFR content.
8. **Use `ffprobe -sections` to discover field names** — lists all section types and their available fields.

---

## Further Reading

- [ffprobe documentation](https://ffmpeg.org/ffprobe.html) — full options reference, output writers, and timecode handling
- [FFmpeg stream specifier syntax](https://ffmpeg.org/ffmpeg.html#Stream-specifiers) — complete specifier grammar shared across all FFmpeg tools
