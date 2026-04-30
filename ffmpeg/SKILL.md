---
name: ffmpeg
description: Convert, transcode, trim, filter, and process video and audio files using the ffmpeg CLI. Handles format conversion, encoding, scaling, audio manipulation, subtitles, GIF creation, concatenation, metadata, and long-running background jobs.
compatibility: Requires ffmpeg installed and available in PATH.
---

# ffmpeg

ffmpeg is a universal multimedia processor. It reads one or more inputs, optionally transcodes or filters streams, and writes to one or more outputs.

```bash
ffmpeg [global options] [input options] -i INPUT [output options] OUTPUT
```

Options placed before `-i` apply to the input. Options placed after `-i` and before the output path apply to the output. Order matters.

Every agent-run command must include these two flags:

```bash
ffmpeg -y -nostdin ...
```

- `-y` — overwrite output files without prompting. Without this, ffmpeg hangs waiting for keyboard input when the output file already exists.
- `-nostdin` — close stdin entirely so ffmpeg never blocks waiting for input.

Suppress the build banner with `-hide_banner`:

```bash
ffmpeg -y -nostdin -hide_banner -i input.mp4 output.mp4
```

---

## Stream Copy vs Transcode

The most important concept in ffmpeg: remux or transcode?

Stream copy (`-c copy`) passes the encoded data through without decoding or re-encoding. It is lossless and near-instant, but only works when the input codec is compatible with the output container.

```bash
# Remux MOV to MP4 — no quality loss, takes seconds
ffmpeg -y -nostdin -i input.mov -c copy output.mp4
```

Transcoding decodes and re-encodes. It is required when changing codec, bitrate, resolution, or applying filters, and is much slower.

```bash
# Transcode to H.264
ffmpeg -y -nostdin -i input.mov -c:v libx264 -c:a aac output.mp4
```

---

## Format Conversion & Remuxing

Change container without re-encoding. Output format is inferred from the file extension.

```bash
# MOV to MP4 (very common — Apple devices record MOV)
ffmpeg -y -nostdin -i input.mov -c copy output.mp4

# MKV to MP4
ffmpeg -y -nostdin -i input.mkv -c copy output.mp4

# MP4 to MKV
ffmpeg -y -nostdin -i input.mp4 -c copy output.mkv

# Extract audio from video container
ffmpeg -y -nostdin -i input.mp4 -vn -c:a copy output.m4a

# Strip video, keep audio only as MP3 (requires transcode)
ffmpeg -y -nostdin -i input.mp4 -vn -c:a libmp3lame -q:a 2 output.mp3
```

`-vn` disables video output. `-an` disables audio output. `-sn` disables subtitles.

Not all codecs are valid in all containers. H.264 works in MP4 and MKV but not WebM; VP9 works in WebM but not MP4. If `-c copy` fails with a muxer error, the codec is incompatible with the container — transcode instead.

---

## Video Encoding

### H.264 (most compatible)

```bash
ffmpeg -y -nostdin -i input.mov -c:v libx264 -crf 23 -preset medium -c:a aac -b:a 128k output.mp4
```

- `-crf` — Constant Rate Factor. 0 = lossless, 51 = worst; 18–28 is the useful range and 23 is the default. Lower values produce better quality at larger file sizes.
- `-preset` — encoding speed vs compression efficiency. Options: `ultrafast`, `superfast`, `veryfast`, `faster`, `fast`, `medium`, `slow`, `slower`, `veryslow`. Slower presets produce smaller files at the same CRF. `medium` is a good default; use `fast` when speed matters.

```bash
# High quality (e.g. archival, post-production)
ffmpeg -y -nostdin -i input.mov -c:v libx264 -crf 18 -preset slow -c:a aac -b:a 192k output.mp4

# Small file for web/sharing
ffmpeg -y -nostdin -i input.mov -c:v libx264 -crf 28 -preset fast -c:a aac -b:a 96k output.mp4
```

### H.265 / HEVC (better compression, less compatible)

```bash
ffmpeg -y -nostdin -i input.mov -c:v libx265 -crf 28 -preset medium -c:a aac -b:a 128k output.mp4
```

H.265 CRF 28 is roughly equivalent quality to H.264 CRF 23 at ~40% smaller file size. It is less universally supported by players and devices.

### Target bitrate (when file size matters)

```bash
# Target 2 Mbps video, 128k audio
ffmpeg -y -nostdin -i input.mp4 -c:v libx264 -b:v 2M -c:a aac -b:a 128k output.mp4
```

### Pixel format

Most players, web browsers, and phones require `yuv420p`. If playback fails on a device that normally works, this is the first thing to add.

```bash
ffmpeg -y -nostdin -i input.mov -c:v libx264 -crf 23 -pix_fmt yuv420p -c:a aac output.mp4
```

---

## Audio Encoding

### AAC (standard for MP4/M4A)

```bash
ffmpeg -y -nostdin -i input.wav -c:a aac -b:a 128k output.m4a
# 128k = good quality; 192k = high quality; 96k = acceptable for speech
```

### MP3

```bash
ffmpeg -y -nostdin -i input.wav -c:a libmp3lame -q:a 2 output.mp3
# -q:a 0–9 (VBR): 0 = best, 9 = worst. 2 is a common high-quality target.
ffmpeg -y -nostdin -i input.wav -c:a libmp3lame -b:a 192k output.mp3   # CBR
```

### Opus (best quality-per-bitrate, for MKV/WebM/OGG)

```bash
ffmpeg -y -nostdin -i input.wav -c:a libopus -b:a 96k output.opus
# Opus at 96k typically matches AAC at 128k
```

### FLAC (lossless)

```bash
ffmpeg -y -nostdin -i input.wav -c:a flac output.flac
```

### Change sample rate or channels

```bash
# Downsample to 44100 Hz
ffmpeg -y -nostdin -i input.wav -ar 44100 output.wav

# Convert to mono
ffmpeg -y -nostdin -i input.wav -ac 1 output.wav

# Both at once
ffmpeg -y -nostdin -i input.wav -ar 16000 -ac 1 output.wav
```

---

## Trimming & Cutting

### Fast trim with stream copy

Placing `-ss` and `-to`/`-t` before `-i` seeks to the nearest keyframe, which may be slightly before the requested time. The seek is near-instant and the output is keyframe-accurate — correct for clips, previews, and segment extraction. Use `-c copy` to avoid re-encoding.

```bash
# From 0:30 to 1:45
ffmpeg -y -nostdin -ss 00:00:30 -to 00:01:45 -i input.mp4 -c copy output.mp4

# Start at 1 minute, take 30 seconds
ffmpeg -y -nostdin -ss 00:01:00 -t 30 -i input.mp4 -c copy output.mp4

# Last 60 seconds
ffmpeg -y -nostdin -sseof -60 -i input.mp4 -c copy output.mp4
```

### Frame-accurate trim (requires transcode)

Placing `-ss` and `-to`/`-t` after `-i` decodes from the start to the exact timestamp. Use only when exact frame accuracy is required.

```bash
ffmpeg -y -nostdin -i input.mp4 -ss 00:00:30 -to 00:01:45 -c:v libx264 -crf 23 -c:a aac output.mp4
```

---

## Scaling & Video Filters

Video filters are applied with `-vf`. Multiple filters are chained with commas.

### Scale (resize)

```bash
# Scale to 1280 wide, preserve aspect ratio
ffmpeg -y -nostdin -i input.mp4 -vf scale=1280:-2 -c:v libx264 -crf 23 output.mp4

# Scale to 720p height
ffmpeg -y -nostdin -i input.mp4 -vf scale=-2:720 -c:v libx264 -crf 23 output.mp4

# Exact dimensions (may distort)
ffmpeg -y -nostdin -i input.mp4 -vf scale=1920:1080 -c:v libx264 -crf 23 output.mp4
```

`-2` means "calculate this dimension to preserve aspect ratio, and round to a multiple of 2." Use `-2` rather than `-1` — H.264 requires dimensions divisible by 2.

### Pad to a target aspect ratio

```bash
# Letterbox a 4:3 video to 16:9 (1920x1080) with black bars
ffmpeg -y -nostdin -i input.mp4 \
  -vf "scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2" \
  -c:v libx264 -crf 23 output.mp4
```

### Change frame rate

```bash
ffmpeg -y -nostdin -i input.mp4 -vf fps=30 -c:v libx264 -crf 23 output.mp4
ffmpeg -y -nostdin -i input.mp4 -vf fps=24000/1001 -c:v libx264 -crf 23 output.mp4  # 23.976
```

### Watermark / overlay text

```bash
ffmpeg -y -nostdin -i input.mp4 \
  -vf "drawtext=text='© Example':fontsize=32:fontcolor=white@0.7:x=20:y=H-th-20" \
  -c:v libx264 -crf 23 -c:a copy output.mp4
```

`H` and `W` are the frame height/width. `th` and `tw` are the text height/width. `@0.7` sets opacity. On Linux, font lookup by name often fails — specify the font file explicitly: `fontfile=/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf`.

### Fade in / fade out

Fade start times are in seconds — get the file duration with ffprobe before writing the command.

```bash
ffmpeg -y -nostdin -i input.mp4 \
  -vf "fade=t=in:st=0:d=1,fade=t=out:st=58:d=2" \
  -c:v libx264 -crf 23 -c:a copy output.mp4
```

### Rotate video

```bash
# Rotate 90° clockwise
ffmpeg -y -nostdin -i input.mp4 -vf transpose=1 -c:v libx264 -crf 23 output.mp4
# transpose=1: 90° clockwise
# transpose=2: 90° counter-clockwise
# transpose=0: 90° counter-clockwise + vertical flip
# transpose=3: 90° clockwise + vertical flip
```

When transcoding (not stream copy), ffmpeg auto-applies rotation metadata to the pixels — phone videos that play sideways in some players are fixed by simply transcoding:

```bash
ffmpeg -y -nostdin -i input.mp4 -c:v libx264 -crf 23 -c:a copy output.mp4
```

---

## Audio Filters

Audio filters are applied with `-af`.

### Volume adjustment

```bash
# Double the volume
ffmpeg -y -nostdin -i input.mp4 -af volume=2.0 -c:v copy output.mp4

# Halve the volume
ffmpeg -y -nostdin -i input.mp4 -af volume=0.5 -c:v copy output.mp4

# Adjust by dB
ffmpeg -y -nostdin -i input.mp4 -af volume=6dB -c:v copy output.mp4
```

### Loudness normalization (broadcast standard)

```bash
# Pass 1: measure (use the same I/TP/LRA targets you'll apply in pass 2)
ffmpeg -y -nostdin -i input.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11:print_format=json -f null /dev/null 2>&1 | tail -12

# Pass 2: apply measured values (substitute actual values from pass 1)
ffmpeg -y -nostdin -i input.mp4 \
  -af "loudnorm=I=-16:TP=-1.5:LRA=11:measured_I=-23:measured_LRA=7:measured_TP=-2:measured_thresh=-33:offset=0" \
  -c:v copy output.mp4
```

Single-pass loudnorm is less accurate but sufficient for most cases:

```bash
ffmpeg -y -nostdin -i input.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11 -c:v copy output.mp4
```

### Audio fade

The fade-out start time (`st=`) is the file duration minus the fade duration. For a 30-second file with a 2-second fade out, `st=28`.

```bash
ffmpeg -y -nostdin -i input.mp3 -af "afade=t=in:d=2,afade=t=out:st=28:d=2" output.mp3
```

### Delay audio (for sync issues)

`adelay` takes milliseconds per channel (`250|250` for stereo, `250` for mono) and can only shift audio later. To shift audio earlier, delay the video instead with `setpts`.

```bash
# Shift audio 250ms later
ffmpeg -y -nostdin -i input.mp4 -af "adelay=250|250" -c:v copy output.mp4

# Shift video 250ms later (effectively makes audio earlier)
ffmpeg -y -nostdin -i input.mp4 -vf setpts=PTS+0.25/TB -c:a copy output.mp4
```

---

## Stream Selection & Mapping

By default ffmpeg picks the "best" stream of each type. Use `-map` for explicit control.

```bash
# Map first video stream and first audio stream
ffmpeg -y -nostdin -i input.mkv -map 0:v:0 -map 0:a:0 -c copy output.mp4

# Keep all streams from input 0
ffmpeg -y -nostdin -i input.mkv -map 0 -c copy output.mkv

# Extract only the second audio track
ffmpeg -y -nostdin -i input.mkv -map 0:a:1 -c:a copy audio_track2.aac

# Merge separate video and audio files
ffmpeg -y -nostdin -i video.mp4 -i audio.m4a -map 0:v:0 -map 1:a:0 -c copy output.mp4
```

Stream specifier syntax: `INPUT_INDEX:TYPE:STREAM_INDEX`
- `0:v:0` — first video stream from input 0
- `0:a:1` — second audio stream from input 0
- `1:a:0` — first audio stream from input 1

---

## Subtitles

### Extract subtitle track to file

```bash
# Extract first subtitle stream as SRT
ffmpeg -y -nostdin -i input.mkv -map 0:s:0 -c:s srt output.srt

# Extract as WebVTT
ffmpeg -y -nostdin -i input.mkv -map 0:s:0 -c:s webvtt output.vtt
```

### Burn subtitles into video (hardcode)

```bash
# From an external SRT file
ffmpeg -y -nostdin -i input.mp4 -vf subtitles=subtitles.srt -c:v libx264 -crf 23 -c:a copy output.mp4

# From a subtitle track in the same file (use a copy of the file as subtitle source)
ffmpeg -y -nostdin -i input.mkv -vf subtitles=input.mkv -c:v libx264 -crf 23 -c:a copy output.mp4
```

Hardcoded subtitles cannot be turned off by the viewer. The `subtitles` filter requires ffmpeg to be built with `--enable-libass`.

### Add an external SRT as a soft subtitle stream

```bash
ffmpeg -y -nostdin -i input.mp4 -i subtitles.srt -map 0 -map 1 -c copy -c:s mov_text output.mp4
```

`mov_text` is the subtitle codec for MP4 containers. Use `ass` or `srt` for MKV.

---

## GIF Creation

Creating quality GIFs requires a two-pass approach: generate an optimal palette from the video first, then apply it.

```bash
# Pass 1: generate palette
ffmpeg -y -nostdin -ss 00:00:05 -t 3 -i input.mp4 \
  -vf "fps=15,scale=480:-1:flags=lanczos,palettegen" palette.png

# Pass 2: apply palette
ffmpeg -y -nostdin -ss 00:00:05 -t 3 -i input.mp4 -i palette.png \
  -filter_complex "[0:v]fps=15,scale=480:-1:flags=lanczos[x];[x][1:v]paletteuse" output.gif
```

- `fps=15` — good balance of smoothness vs file size; drop lower for smaller files
- `scale=480:-1` — GIF file size grows fast with resolution; keep width at 480px or below

Without palette generation, a single-pass encode is faster but produces lower quality output:

```bash
ffmpeg -y -nostdin -ss 00:00:05 -t 3 -i input.mp4 -vf "fps=12,scale=320:-1" output.gif
```

---

## Concatenation

### Concat demuxer (fastest — stream copy, same codec required)

Create a text file listing the inputs:

```bash
# concat_list.txt
file 'part1.mp4'
file 'part2.mp4'
file 'part3.mp4'
```

```bash
ffmpeg -y -nostdin -f concat -safe 0 -i concat_list.txt -c copy output.mp4
```

All input files must have the same codec, resolution, frame rate, and sample rate. If they differ, use the concat filter instead.

### Concat filter (transcode — handles mismatched inputs)

```bash
ffmpeg -y -nostdin -i part1.mp4 -i part2.mp4 \
  -filter_complex "[0:v][0:a][1:v][1:a]concat=n=2:v=1:a=1[outv][outa]" \
  -map "[outv]" -map "[outa]" \
  -c:v libx264 -crf 23 -c:a aac output.mp4
```

`concat=n=2:v=1:a=1` — concatenate 2 segments, 1 video stream, 1 audio stream.

---

## Metadata

### Write tags

```bash
ffmpeg -y -nostdin -i input.mp4 \
  -metadata title="My Video" \
  -metadata artist="Name" \
  -metadata date="2024" \
  -c copy output.mp4
```

Standard tag keys for MP4: `title`, `artist`, `album`, `date`, `comment`, `genre`. Use `date` not `year` — `year` is an ID3/MP3 tag name; MP4 players look for `date`.

### Strip all metadata

```bash
ffmpeg -y -nostdin -i input.mp4 -map_metadata -1 -c copy output.mp4
```

### Copy metadata from a different file

```bash
ffmpeg -y -nostdin -i input.mp4 -i metadata_source.mp4 \
  -map 0 -map_metadata 1 -c copy output.mp4
```

---

## Long-Running Jobs & Progress Monitoring

Transcoding jobs routinely exceed the 2-minute default Bash tool timeout. For any job that could take more than a minute, use background execution with progress file output.

### Estimating job length

Run ffprobe to get the file duration before starting a transcode, then use that duration with a rough speed estimate to decide whether the job needs background mode.

```bash
duration=$(ffprobe -v quiet -hide_banner -of csv=p=0 \
  -show_entries format=duration input.mp4)
```

As a rough guide: a 1-hour H.264 encode at `preset medium` takes ~5–20 minutes on a modern CPU. Speed depends heavily on resolution, preset, and hardware.

### Background execution with progress file

```bash
ffmpeg -y -nostdin -hide_banner \
  -progress /tmp/ffmpeg_progress.txt \
  -stats_period 2 \
  -i input.mp4 \
  -c:v libx264 -crf 23 -c:a aac \
  output.mp4
```

Run this with `run_in_background: true`. The `-progress` file is updated every `-stats_period` seconds with key=value lines:

```
frame=1240
fps=28.5
bitrate=2400.0kbits/s
total_size=18432000
out_time=00:00:41.330000
speed=2.33x
progress=continue
```

`progress` is `continue` while running and `end` when the job completes.

### Polling progress

```bash
tail -20 /tmp/ffmpeg_progress.txt
```

Key fields:
- `out_time` — how much output has been encoded so far
- `speed` — encoding speed relative to real time (2x = encoding 2 seconds of video per second)
- `progress=end` — job is complete

Estimate time remaining: `remaining = (total_duration - out_time_seconds) / speed`

### Short jobs (under 2 minutes)

Remuxing, stream-copy trimming, and audio extraction finish in seconds and can run synchronously without a progress file.

```bash
ffmpeg -y -nostdin -i input.mov -c copy output.mp4
ffmpeg -y -nostdin -ss 0 -t 30 -i input.mp4 -c copy clip.mp4
```

---

## Batch Processing

### Process all files in a directory

```bash
for f in *.mov; do
  ffmpeg -y -nostdin -hide_banner -i "$f" -c:v libx264 -crf 23 -c:a aac "${f%.mov}.mp4"
done
```

`${f%.mov}` strips the `.mov` extension from the filename.

### Parallel batch (multiple jobs at once)

```bash
for f in *.mov; do
  ffmpeg -y -nostdin -hide_banner \
    -progress "/tmp/progress_${f}.txt" \
    -i "$f" -c:v libx264 -crf 23 -c:a aac "${f%.mov}.mp4" &
done
wait
```

`&` runs each job in the background. `wait` blocks until all finish. Monitor per-file progress by reading the individual `/tmp/progress_*.txt` files.

### Probe before processing

```bash
for f in *.mp4; do
  codec=$(ffprobe -v quiet -hide_banner -of csv=p=0 \
    -show_entries stream=codec_name -select_streams v:0 "$f")
  if [ "$codec" != "h264" ]; then
    echo "$f: codec=$codec — transcoding"
    ffmpeg -y -nostdin -i "$f" -c:v libx264 -crf 23 -c:a copy "${f%.mp4}_h264.mp4"
  else
    echo "$f: already H.264 — copying"
    ffmpeg -y -nostdin -i "$f" -c copy "${f%.mp4}_copy.mp4"
  fi
done
```

---

## Agent Rules

1. Always use `-y -nostdin` — every ffmpeg command must include both. Without them, any process that writes an existing output file will hang indefinitely waiting for keyboard input.

2. Probe duration before starting — run `ffprobe` to get file duration before any transcode. Use the duration plus a rough speed estimate to decide whether the job needs background mode.

3. Use `-progress /tmp/ffmpeg_progress.txt` for jobs over ~1 minute — run with `run_in_background: true` and poll the progress file. Never run a long transcode synchronously and hope it finishes within the timeout.

4. Prefer `-c copy` (stream copy) over transcoding — if the goal is format conversion or container change and the codec is compatible, stream copy is always faster and produces no quality loss. Only transcode when you need to change codec, bitrate, resolution, or apply filters.

5. Use `-pix_fmt yuv420p` when compatibility matters — H.264 output without this flag may use `yuv444p` or other formats that are technically valid but rejected by many players and devices.

6. Use `-2` not `-1` for aspect-ratio-preserving scale — `scale=1280:-1` can produce odd-numbered dimensions that H.264 rejects. Always use `scale=1280:-2` to ensure divisibility by 2.

7. Use `-ss` before `-i` for seek speed, after `-i` for frame accuracy — pre-input seek is fast but keyframe-accurate. Post-input seek is frame-accurate but decodes from the start. For clips and previews, pre-input is almost always correct.

8. Never guess at supported codecs or filters — check with `ffmpeg -codecs`, `ffmpeg -encoders`, or `ffmpeg -filters`. Codec availability depends on how ffmpeg was built; a codec present on one machine may be absent on another.

9. Check `speed=` in the progress file to estimate remaining time — `remaining_seconds = (total_duration - elapsed) / speed`. A speed below `1.0x` means the job will take longer than the source duration.

10. Use `ffprobe` to validate input before processing — if a file has an unexpected codec, resolution, or stream layout, it is better to detect this before starting a long job than to discover a mux error at the end.

---

## Further Reading

- [ffmpeg documentation](https://ffmpeg.org/ffmpeg.html) — full options reference, stream specifiers, filtergraph syntax
- [ffmpeg filters documentation](https://ffmpeg.org/ffmpeg-filters.html) — complete filter reference with all parameters
