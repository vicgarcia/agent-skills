---
name: ffmpeg
description: Convert, transcode, trim, filter, and process video and audio files using the ffmpeg CLI. Handles format conversion, encoding (H.264/H.265/VP9/AV1), scaling, cropping, rotation, speed changes, overlays, watermarks and picture-in-picture, audio manipulation, multi-track mixing and loudness, subtitles, GIFs, concatenation and transitions, multi-source editing, frame extraction and contact sheets, metadata, and long-running background jobs.
compatibility: Requires ffmpeg installed and available in PATH. Frame/contact-sheet recipes optionally use ImageMagick's `magick`.
---

# ffmpeg

ffmpeg is a universal multimedia processor. It reads one or more inputs, optionally decodes, filters, and re-encodes streams, and writes to one or more outputs.

```bash
ffmpeg [global options] [input options] -i INPUT [output options] OUTPUT
```

Options before `-i` apply to that input; options after the last `-i` and before the output path apply to the output. **Order matters.**

Baseline flags for every agent-run command:

```bash
ffmpeg -y -nostdin -hide_banner -loglevel error ...
```

- `-y` — overwrite outputs without prompting (otherwise ffmpeg hangs if the output exists).
- `-nostdin` — never read the keyboard. Prevents hangs, and stops ffmpeg from swallowing stdin inside `while read` loops.
- `-hide_banner -loglevel error` — keep output small. Use `-loglevel warning` while debugging, or add `-stats` for a single updating progress line.

Discover build capabilities before assuming a codec or filter exists: `ffmpeg -encoders`, `-decoders`, `-filters`, `-muxers`. Get help on one filter with `ffmpeg -hide_banner -h filter=NAME`.

---

## Stream Copy vs Transcode

This is the key decision: remux, or re-encode?

**Stream copy** (`-c copy`) moves encoded data into a new container without decoding. It is lossless and near-instant, but only works when the codec is legal in the target container, and it cannot be combined with filters.

```bash
ffmpeg -y -nostdin -i input.mov -c copy output.mp4
```

**Transcoding** decodes and re-encodes. You need it to change codec, bitrate, resolution, or frame rate, or to apply any filter. It is slow and lossy.

```bash
ffmpeg -y -nostdin -i input.mov -c:v libx264 -c:a aac output.mp4
```

Mix them per stream: `-c:v libx264 -c:a copy` re-encodes video and passes audio through.

### Container / codec compatibility

| Container | Video | Audio | Subtitles |
|-----------|-------|-------|-----------|
| MP4 / MOV | h264, hevc, av1, (prores in MOV) | aac, mp3, ac3, alac, opus* | mov_text |
| MKV | almost anything | almost anything | srt, ass, pgs |
| WebM | vp8, vp9, av1 | opus, vorbis | webvtt |

\*Opus in MP4 works in modern players but not everywhere. If `-c copy` fails with a muxer error ("codec not currently supported in container"), transcode the offending stream.

---

## Format Conversion & Remuxing

```bash
ffmpeg -y -nostdin -i input.mov -c copy output.mp4          # MOV → MP4
ffmpeg -y -nostdin -i input.mkv -c copy output.mp4          # MKV → MP4 (if codecs fit; subtitles may need -sn or -c:s mov_text)
ffmpeg -y -nostdin -i input.mp4 -vn -c:a copy output.m4a    # Extract AAC audio losslessly
ffmpeg -y -nostdin -i input.mp4 -vn -c:a libmp3lame -q:a 2 output.mp3  # Audio → MP3
```

`-vn` drops video, `-an` drops audio, `-sn` drops subtitles, `-dn` drops data streams.

Check the source audio codec before choosing the extension when extracting with `-c:a copy`: AAC → `.m4a`, MP3 → `.mp3`, Opus → `.opus`/`.ogg`, AC3 → `.ac3`, FLAC → `.flac`. If unsure, use `.mka`, which accepts any audio codec.

---

## Video Encoding

### H.264 (maximum compatibility)

```bash
ffmpeg -y -nostdin -i input.mov \
  -c:v libx264 -crf 23 -preset medium -pix_fmt yuv420p \
  -c:a aac -b:a 128k -movflags +faststart output.mp4
```

- `-crf` — constant quality. 0 is lossless and 51 is worst; 18–28 is the useful range and 23 is the default. Each ±6 roughly halves or doubles the file size.
- `-preset` — `ultrafast` … `medium` … `veryslow`. Slower presets give smaller files at the same quality. Use `fast`/`veryfast` when speed matters.
- `-pix_fmt yuv420p` — needed for browsers, phones, and QuickTime. Sources that are 10-bit, 4:2:2, or 4:4:4 (screen recordings, ProRes) otherwise produce files many players reject.
- `-movflags +faststart` — moves the index to the front so web playback can start before the download finishes. Use it for any MP4 meant for the web.
- `-tune film|animation|stillimage|grain` — optional content hint.

```bash
# High quality / archival
ffmpeg -y -nostdin -i input.mov -c:v libx264 -crf 18 -preset slow -pix_fmt yuv420p -c:a aac -b:a 192k output.mp4
# Small, for sharing
ffmpeg -y -nostdin -i input.mov -c:v libx264 -crf 28 -preset fast -pix_fmt yuv420p -c:a aac -b:a 96k output.mp4
```

### H.265 / HEVC (≈40% smaller, less compatible)

```bash
ffmpeg -y -nostdin -i input.mov -c:v libx265 -crf 28 -preset medium -tag:v hvc1 -c:a aac -b:a 128k output.mp4
```

At CRF 28, x265 is roughly equivalent to x264 at CRF 23. `-tag:v hvc1` is needed for Apple devices and QuickTime to play HEVC in MP4.

### VP9 / WebM

```bash
ffmpeg -y -nostdin -i input.mp4 -c:v libvpx-vp9 -crf 32 -b:v 0 -row-mt 1 -c:a libopus -b:a 96k output.webm
```

`-b:v 0` is required for VP9 constant-quality mode.

### AV1 (best compression, slowest to encode)

```bash
ffmpeg -y -nostdin -i input.mp4 -c:v libsvtav1 -crf 35 -preset 8 -c:a libopus -b:a 96k output.mkv
```

SVT-AV1 presets run 0 (slowest) to 13 (fastest); 6–10 is practical. Check availability with `ffmpeg -encoders | grep -i av1`.

### Hardware encoders

Much faster but lower quality per bit. Check what exists with `ffmpeg -hide_banner -encoders | grep -Ei 'nvenc|qsv|amf|videotoolbox'`.

```bash
ffmpeg -y -nostdin -i input.mp4 -c:v h264_nvenc -preset p5 -cq 23 -c:a copy output.mp4   # NVIDIA
```

### Target bitrate and two-pass (hit a file size)

```bash
# Single-pass average bitrate
ffmpeg -y -nostdin -i input.mp4 -c:v libx264 -b:v 2M -maxrate 2.5M -bufsize 4M -c:a aac -b:a 128k output.mp4

# Two-pass (accurate size targeting)
ffmpeg -y -nostdin -i input.mp4 -c:v libx264 -b:v 2M -preset medium -pass 1 -an -f null -
ffmpeg -y -nostdin -i input.mp4 -c:v libx264 -b:v 2M -preset medium -pass 2 -c:a aac -b:a 128k output.mp4
```

Video bitrate for a size target: `video_kbps = target_MB * 8192 / duration_s - audio_kbps`. Two-pass leaves `ffmpeg2pass-0.log*` files in the working directory; delete them afterward.

#### Worked recipe: shrink a video to fit a size limit (e.g. an upload cap)

```bash
in=in.mp4; target_mb=25; audio_kbps=96
dur=$(ffprobe -v error -of csv=p=0 -show_entries format=duration "$in")
vk=$(awk -v t="$target_mb" -v d="$dur" -v a="$audio_kbps" 'BEGIN{printf "%d", (t*8192/d - a)*0.97}')
echo "video bitrate: ${vk}k"    # if under ~500k at 1080p, also downscale (scale=-2:720 or 480)

ffmpeg -y -nostdin -i "$in" -vf "scale='min(1280,iw)':-2" -c:v libx264 -preset medium -b:v ${vk}k -pass 1 -an -f null -
ffmpeg -y -nostdin -i "$in" -vf "scale='min(1280,iw)':-2" -c:v libx264 -preset medium -b:v ${vk}k -pass 2 \
  -pix_fmt yuv420p -c:a aac -b:a ${audio_kbps}k -movflags +faststart out.mp4
rm -f ffmpeg2pass-0.log*
ls -l out.mp4
```

The `0.97` factor leaves headroom for container overhead. Tested on a 20 s 720p clip with a 1 MB target: 12.4 MB in, 1.02 MB out (97% of the limit, under it), duration unchanged. If the result still lands over the limit, lower `target_mb` slightly and rerun, or drop the resolution. For long inputs, run both passes in the background (see Long-Running Jobs).

---

## Audio Encoding

```bash
ffmpeg -y -nostdin -i input.wav -c:a aac -b:a 128k output.m4a          # AAC: 96k speech, 128k good, 192k+ high
ffmpeg -y -nostdin -i input.wav -c:a libmp3lame -q:a 2 output.mp3      # MP3 VBR (0 best … 9 worst)
ffmpeg -y -nostdin -i input.wav -c:a libmp3lame -b:a 192k output.mp3   # MP3 CBR
ffmpeg -y -nostdin -i input.wav -c:a libopus -b:a 96k output.opus      # Opus: best quality per bit
ffmpeg -y -nostdin -i input.wav -c:a flac output.flac                  # Lossless
```

### Sample rate and channels

```bash
ffmpeg -y -nostdin -i input.wav -ar 44100 output.wav              # Resample
ffmpeg -y -nostdin -i input.wav -ac 1 output.wav                  # Downmix to mono
ffmpeg -y -nostdin -i input.mp4 -vn -ar 16000 -ac 1 -c:a pcm_s16le speech.wav   # Speech-recognition input
```

---

## Trimming & Cutting

### Fast trim with stream copy

With `-ss` before `-i`, ffmpeg seeks to the nearest keyframe at or before the requested time. With `-c copy` the cut snaps to keyframes, so the start may be up to a few seconds early. This is instant.

```bash
ffmpeg -y -nostdin -ss 00:00:30 -to 00:01:45 -i input.mp4 -c copy clip.mp4    # 0:30 → 1:45
ffmpeg -y -nostdin -ss 60 -t 30 -i input.mp4 -c copy clip.mp4                 # 30 s from 1:00
ffmpeg -y -nostdin -sseof -60 -i input.mp4 -c copy tail.mp4                   # last 60 s
```

With `-ss` before `-i`, timestamps are reset, so `-to` refers to the original timeline in recent ffmpeg versions. If an output's length looks wrong, use `-t` (duration) instead; it is unambiguous.

### Frame-accurate trim (transcode)

Keep `-ss` before `-i` (fast seek) and re-encode. When transcoding, ffmpeg decodes from the prior keyframe and discards frames up to the exact time, so the cut is both fast and accurate.

```bash
ffmpeg -y -nostdin -ss 00:00:30 -i input.mp4 -t 75 -c:v libx264 -crf 18 -pix_fmt yuv420p -c:a aac clip.mp4
```

### Frame-accurate multi-range cut (keep several ranges)

```bash
# Keep 0–10 s and 20–30 s
ffmpeg -y -nostdin -i in.mp4 -filter_complex "
  [0:v]trim=0:10,setpts=PTS-STARTPTS[v0];[0:a]atrim=0:10,asetpts=PTS-STARTPTS[a0];
  [0:v]trim=20:30,setpts=PTS-STARTPTS[v1];[0:a]atrim=20:30,asetpts=PTS-STARTPTS[a1];
  [v0][a0][v1][a1]concat=n=2:v=1:a=1[v][a]" -map "[v]" -map "[a]" -c:v libx264 -crf 20 -c:a aac out.mp4
```

### Split into equal segments

```bash
ffmpeg -y -nostdin -i input.mp4 -c copy -f segment -segment_time 600 -reset_timestamps 1 part_%03d.mp4
```

Segments split on keyframes, so their lengths are approximate.

---

## Scaling, Cropping, Padding

Video filters apply with `-vf`; chain them with commas. Any filter forces a transcode of that stream (no `-c copy` on a filtered stream).

```bash
ffmpeg -y -nostdin -i in.mp4 -vf scale=1280:-2 -c:v libx264 -crf 23 -c:a copy out.mp4     # 1280 wide, keep aspect
ffmpeg -y -nostdin -i in.mp4 -vf scale=-2:720 -c:v libx264 -crf 23 -c:a copy out.mp4      # 720p
ffmpeg -y -nostdin -i in.mp4 -vf "scale='min(1920,iw)':-2" -c:v libx264 -crf 23 out.mp4   # cap width, never upscale
ffmpeg -y -nostdin -i in.mp4 -vf scale=iw/2:-2 -c:v libx264 -crf 23 out.mp4               # half size
```

`-2` computes the dimension from the aspect ratio and rounds it to an even number (H.264/H.265 require this). Use `-2`, never `-1`. Add `:flags=lanczos` for sharper downscales.

### Fit inside a box with letterbox/pillarbox bars

```bash
ffmpeg -y -nostdin -i in.mp4 \
  -vf "scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2:color=black,setsar=1" \
  -c:v libx264 -crf 23 -c:a copy out.mp4
```

### Fill a box by cropping (e.g. 16:9 → 9:16 vertical)

```bash
ffmpeg -y -nostdin -i in.mp4 \
  -vf "scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920" \
  -c:v libx264 -crf 23 -c:a copy vertical.mp4
```

### Crop

`crop=W:H:X:Y` (X and Y default to centered); expressions can use `iw`, `ih`.

```bash
ffmpeg -y -nostdin -i in.mp4 -vf crop=640:480:100:50 -c:v libx264 -crf 23 out.mp4
ffmpeg -y -nostdin -i in.mp4 -vf "crop=iw:ih-120:0:60" -c:v libx264 -crf 23 out.mp4      # trim 60px top and bottom
```

### Detect and remove black bars

```bash
ffmpeg -nostdin -hide_banner -ss 60 -i in.mp4 -t 10 -vf cropdetect -f null - 2>&1 | grep -o 'crop=[0-9:]*' | sort | uniq -c | sort -rn | head -1
# → e.g. "crop=1920:800:0:140" — apply it with -vf crop=1920:800:0:140
```

Sample from the middle of the file (`-ss`), not the start: openings are often black or have titles.

---

## Rotate, Flip, Frame Rate, Speed

```bash
ffmpeg -y -nostdin -i in.mp4 -vf transpose=1 -c:v libx264 -crf 23 -c:a copy out.mp4   # 90° clockwise
ffmpeg -y -nostdin -i in.mp4 -vf transpose=2 -c:v libx264 -crf 23 -c:a copy out.mp4   # 90° counter-clockwise
ffmpeg -y -nostdin -i in.mp4 -vf "transpose=1,transpose=1" -c:v libx264 -crf 23 out.mp4   # 180°
ffmpeg -y -nostdin -i in.mp4 -vf hflip -c:v libx264 -crf 23 out.mp4                   # mirror (vflip = upside down)
ffmpeg -y -nostdin -i in.mp4 -vf fps=30 -c:v libx264 -crf 23 -c:a copy out.mp4        # change frame rate
```

When transcoding, ffmpeg automatically applies the source's rotation metadata to the pixels. Don't also add `transpose` for the same rotation, or it is applied twice. Change only the rotation flag without re-encoding (ffmpeg 6.1+): `ffmpeg -y -nostdin -display_rotation 90 -i in.mp4 -c copy out.mp4`.

### Speed up / slow down

```bash
# 2× speed (video and audio)
ffmpeg -y -nostdin -i in.mp4 -filter_complex "[0:v]setpts=0.5*PTS[v];[0:a]atempo=2.0[a]" -map "[v]" -map "[a]" -c:v libx264 -crf 23 -c:a aac fast.mp4

# 0.5× slow motion
ffmpeg -y -nostdin -i in.mp4 -filter_complex "[0:v]setpts=2.0*PTS[v];[0:a]atempo=0.5[a]" -map "[v]" -map "[a]" -c:v libx264 -crf 23 -c:a aac slow.mp4

# Timelapse: keep every 30th frame, no audio
ffmpeg -y -nostdin -i in.mp4 -vf "setpts=PTS/30" -an -r 30 -c:v libx264 -crf 23 timelapse.mp4
```

`atempo` accepts 0.5–100; chain it for lower values (`atempo=0.5,atempo=0.5` = 0.25×). `setpts=F*PTS` with F = 1/speed.

---

## Text, Logo, and Picture-in-Picture Overlays

```bash
# Text / watermark with a background box
ffmpeg -y -nostdin -i in.mp4 \
  -vf "drawtext=fontfile=arial.ttf:text='© Example':fontsize=32:fontcolor=white@0.8:box=1:boxcolor=black@0.4:boxborderw=8:x=20:y=h-th-20" \
  -c:v libx264 -crf 23 -c:a copy out.mp4
```

- Variables: `w`/`h` (frame), `tw`/`th` (text), `t` (seconds). Center with `x=(w-tw)/2:y=(h-th)/2`.
- Running timestamp: `text='%{pts\:hms}'`. Frame number: `text='%{frame_num}'`.
- Show only between 5 and 10 s: append `:enable='between(t,5,10)'`.
- Text with `:` `'` `%` or `\` is messy to escape — write it to a file and use `textfile=caption.txt`.
- **Font:** pass `fontfile=` explicitly; fontconfig is often absent on Windows builds. Linux: `/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf`.

```bash
# Logo bottom-right, 20px inset, scaled to 200px at 70% opacity
ffmpeg -y -nostdin -i in.mp4 -i logo.png -filter_complex \
  "[1:v]scale=200:-1,format=rgba,colorchannelmixer=aa=0.7[l];[0:v][l]overlay=W-w-20:H-h-20" \
  -c:v libx264 -crf 23 -c:a copy out.mp4

# Picture-in-picture (cam in top-right quarter-size)
ffmpeg -y -nostdin -i main.mp4 -i cam.mp4 -filter_complex "[1:v]scale=iw/4:-2[pip];[0:v][pip]overlay=W-w-20:20:shortest=1" -c:v libx264 -crf 23 -c:a copy pip.mp4

# Side by side
ffmpeg -y -nostdin -i left.mp4 -i right.mp4 -filter_complex "[0:v][1:v]hstack=inputs=2[v]" -map "[v]" -map 0:a? -c:v libx264 -crf 23 sbs.mp4
```

Uppercase `W`/`H` = main input, lowercase `w`/`h` = overlay (long forms: `main_w`/`overlay_w`). Corners with a 20px margin: `20:20`, `W-w-20:20`, `20:H-h-20`, `W-w-20:H-h-20`. `hstack` needs equal heights, `vstack` equal widths — scale first if they differ (`[0:v]scale=-2:1080[l];[1:v]scale=-2:1080[r];[l][r]hstack`); `xstack` builds custom grids. Only mapped audio survives — the overlay input's audio is dropped unless you map or `amix` it.

```bash
# PIP with a 3px white border
ffmpeg -y -nostdin -i main.mp4 -i cam.mp4 -filter_complex \
  "[1:v]scale=320:-2,pad=iw+6:ih+6:3:3:color=white[pip];[0:v][pip]overlay=W-w-20:H-h-20" \
  -c:v libx264 -crf 23 -c:a copy out.mp4

# Multiple insets: chain overlays, each stage feeding the next
ffmpeg -y -nostdin -i main.mp4 -i a.mp4 -i b.mp4 -filter_complex \
  "[1:v]scale=320:-2[p1];[2:v]scale=320:-2[p2];[0:v][p1]overlay=20:20[t];[t][p2]overlay=W-w-20:20[v]" \
  -map "[v]" -map 0:a -c:v libx264 -crf 23 -c:a copy out.mp4
```

### Fades, crossfades, region blur

```bash
# Fade in 1 s and fade out the last 2 s of a 60 s clip
ffmpeg -y -nostdin -i in.mp4 -vf "fade=t=in:st=0:d=1,fade=t=out:st=58:d=2" -af "afade=t=in:d=1,afade=t=out:st=58:d=2" -c:v libx264 -crf 23 -c:a aac out.mp4

# Crossfade two clips (offset = clip1 duration - transition duration)
ffmpeg -y -nostdin -i a.mp4 -i b.mp4 -filter_complex \
  "[0:v][1:v]xfade=transition=fade:duration=1:offset=9[v];[0:a][1:a]acrossfade=d=1[a]" \
  -map "[v]" -map "[a]" -c:v libx264 -crf 23 -c:a aac out.mp4

# Blur a region (e.g. a face or plate) at x=100,y=200, size 300x150
ffmpeg -y -nostdin -i in.mp4 -filter_complex \
  "[0:v]crop=300:150:100:200,boxblur=20[b];[0:v][b]overlay=100:200" -c:v libx264 -crf 23 -c:a copy out.mp4
```

- Get the `xfade` offset from ffprobe: `ffprobe -v error -of csv=p=0 -show_entries format=duration a.mp4`. Both inputs must already match in size, frame rate, and SAR (see concat filter below).
- Other `transition=` values include `wipeleft`, `slideup`, `circleopen`, `dissolve` — `ffmpeg -h filter=xfade` lists all.
- For 3+ clips, chain `xfade` pairwise; each offset is the cumulative duration so far minus the transitions already applied.

---

## Audio Filters

```bash
ffmpeg -y -nostdin -i in.mp4 -af volume=2.0 -c:v copy out.mp4          # linear gain (0.5 = half)
ffmpeg -y -nostdin -i in.mp4 -af volume=6dB -c:v copy out.mp4          # gain in dB
ffmpeg -y -nostdin -i in.mp4 -af "highpass=f=80,lowpass=f=12000" -c:v copy out.mp4   # remove rumble and hiss band
ffmpeg -y -nostdin -i in.mp4 -af afftdn -c:v copy out.mp4              # FFT noise reduction
ffmpeg -y -nostdin -i in.mp4 -af adelay=250:all=1 -c:v copy out.mp4   # delay all audio 250 ms (shifts audio later)
```

`adelay` can only shift audio later. To make audio *earlier*, delay the video: `-itsoffset 0.25 -i in.mp4 -i in.mp4 -map 0:v -map 1:a -c copy out.mp4`.

### Loudness normalization (EBU R128)

```bash
# Single pass (good enough for most uses)
ffmpeg -y -nostdin -i in.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11 -c:v copy -c:a aac -b:a 192k out.mp4

# Two-pass: measure, then apply the measured values
ffmpeg -nostdin -hide_banner -i in.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11:print_format=json -f null - 2>&1 | tail -12
ffmpeg -y -nostdin -i in.mp4 -af "loudnorm=I=-16:TP=-1.5:LRA=11:measured_I=-23.1:measured_LRA=7.2:measured_TP=-2.0:measured_thresh=-33.4:offset=0.3:linear=true" -c:v copy -c:a aac -b:a 192k out.mp4
```

Targets: -16 LUFS for podcasts and web, -14 for streaming music, -23 for broadcast (EBU).

### Analysis (prints to stderr, writes nothing)

```bash
ffmpeg -nostdin -hide_banner -i in.mp4 -af volumedetect -vn -f null - 2>&1 | grep -E 'mean_volume|max_volume'
ffmpeg -nostdin -hide_banner -i in.mp4 -af silencedetect=noise=-35dB:d=0.5 -vn -f null - 2>&1 | grep -E 'silence_(start|end)'
```

`silencedetect` output gives cut points for removing pauses or splitting tracks. Remove silence directly: `-af silenceremove=stop_periods=-1:stop_duration=0.7:stop_threshold=-35dB`.

### Mix / replace audio

```bash
# Mix background music under the original audio at 20% volume
ffmpeg -y -nostdin -i video.mp4 -i music.mp3 -filter_complex \
  "[1:a]volume=0.2[m];[0:a][m]amix=inputs=2:duration=first:dropout_transition=2[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac out.mp4

# Replace audio entirely
ffmpeg -y -nostdin -i video.mp4 -i audio.m4a -map 0:v:0 -map 1:a:0 -c copy -shortest out.mp4
```

- `amix` averages its inputs, so each track comes out quieter — set the balance with `volume` on each input before mixing (e.g. drone motor noise at 0.3 under a voiceover).
- `duration=first` matches the first input's length; `duration=longest` keeps the longer one.
- `-shortest` ends the output with the shorter stream, avoiding trailing silence or a frozen last frame when the replacement audio and the video differ in length.

---

## Stream Selection & Mapping

By default ffmpeg picks one "best" video, audio, and subtitle stream. Use `-map` for explicit control; once any `-map` is given, only mapped streams are included.

```bash
ffmpeg -y -nostdin -i input.mkv -map 0:v:0 -map 0:a:0 -c copy output.mp4        # first video + first audio
ffmpeg -y -nostdin -i input.mkv -map 0 -c copy output.mkv                         # everything
ffmpeg -y -nostdin -i input.mkv -map 0 -map -0:s -c copy output.mkv               # everything except subtitles
ffmpeg -y -nostdin -i input.mkv -map 0:v -map 0:a:m:language:eng -c copy out.mkv  # only English audio
```

Specifier syntax: `INPUT:TYPE:INDEX` (`0:a:1` = input 0, second audio stream). A leading `-` negates (`-map -0:s`). `m:key:value` matches stream metadata.

---

## Concatenation

### Concat demuxer (stream copy; inputs must match)

All parts need the same codecs, resolution, frame rate, pixel format, and audio layout. Pieces cut from the same source qualify.

```bash
printf "file '%s'\n" part1.mp4 part2.mp4 part3.mp4 > list.txt
ffmpeg -y -nostdin -f concat -safe 0 -i list.txt -c copy joined.mp4
```

Inside `list.txt`, a single quote in a filename must be written as `'\''`. Paths are relative to the list file's directory.

### Concat filter (re-encode; handles mismatched inputs)

Normalize every input to the same size, frame rate, and SAR first, or the filter errors:

```bash
ffmpeg -y -nostdin -i a.mp4 -i b.mov -filter_complex "
  [0:v]scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2,setsar=1,fps=30[v0];
  [1:v]scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2,setsar=1,fps=30[v1];
  [0:a]aformat=sample_rates=48000:channel_layouts=stereo[a0];
  [1:a]aformat=sample_rates=48000:channel_layouts=stereo[a1];
  [v0][a0][v1][a1]concat=n=2:v=1:a=1[v][a]" \
  -map "[v]" -map "[a]" -c:v libx264 -crf 23 -pix_fmt yuv420p -c:a aac joined.mp4
```

Clips from different devices (drone, phone, screen recording) rarely match. `setsar=1` matters as much as `scale` and `fps`: a mismatched sample aspect ratio is an easy-to-miss cause of `concat` failing even when width and height already match.

---

## Subtitles

Check subtitle codecs with ffprobe first. Text codecs (`subrip`, `ass`, `mov_text`, `webvtt`) convert freely. Bitmap codecs (`hdmv_pgs_subtitle`, `dvd_subtitle`) can only be copied or burned in.

```bash
ffmpeg -y -nostdin -i in.mkv -map 0:s:0 -c:s srt subs.srt                       # extract first subtitle track to SRT
ffmpeg -y -nostdin -i in.mkv -map 0:s:0 subs.vtt                                 # to WebVTT
ffmpeg -y -nostdin -i in.mp4 -i subs.srt -map 0 -map 1 -c copy -c:s mov_text -metadata:s:s:0 language=eng out.mp4   # soft subs in MP4
ffmpeg -y -nostdin -i in.mkv -i subs.srt -map 0 -map 1 -c copy out.mkv           # soft subs in MKV
```

### Burn in (hardcode)

```bash
ffmpeg -y -nostdin -i in.mp4 -vf "subtitles=subs.srt:force_style='FontSize=24,Outline=2'" -c:v libx264 -crf 23 -c:a copy out.mp4
ffmpeg -y -nostdin -i in.mkv -vf "subtitles=in.mkv:si=0" -c:v libx264 -crf 23 -c:a copy out.mp4          # embedded text track
```

The `subtitles` filter needs libass. The filename is parsed as filter syntax, so prefer a simple relative name. On Windows, absolute paths must escape the drive colon: `subtitles='C\:/path/subs.srt'`.

---

## GIF Creation

Quality GIFs need a palette generated from the clip itself. One-command version using `split`:

```bash
ffmpeg -y -nostdin -ss 5 -t 3 -i in.mp4 \
  -filter_complex "fps=12,scale=480:-1:flags=lanczos,split[a][b];[a]palettegen=stats_mode=diff[p];[b][p]paletteuse=dither=bayer:bayer_scale=4" \
  -loop 0 out.gif
```

- `fps` 10–15 and width ≤ 480 keep the size sane; GIF size grows fast with both.
- `-loop 0` loops forever and `-loop -1` plays once.
- For better compression, consider animated WebP (`-c:v libwebp -lossless 0 -q:v 70 -loop 0 out.webp`) or a muted MP4.

GIF → MP4 (much smaller): `ffmpeg -y -nostdin -i in.gif -movflags +faststart -pix_fmt yuv420p -vf "scale=trunc(iw/2)*2:trunc(ih/2)*2" out.mp4`.

---

## Frames, Contact Sheets, and "Seeing" a Video

The agent can view still images but cannot watch video. To understand a video, extract a few representative frames or one tiled contact sheet and view those. **Output directories must exist** (`mkdir -p frames`); ffmpeg will not create them.

### Single frames

```bash
ffmpeg -y -nostdin -ss 00:01:23.5 -i in.mp4 -frames:v 1 frame.png              # lossless PNG at a known time
ffmpeg -y -nostdin -ss 83.5 -i in.mp4 -frames:v 1 -vf scale=1280:-2 frame.jpg   # downscaled for viewing
ffmpeg -y -nostdin -sseof -1 -i in.mp4 -frames:v 1 last.png                     # near-last frame
ffmpeg -y -nostdin -ss 30 -i in.mp4 -vf "thumbnail=n=150,scale=1280:-2" -frames:v 1 poster.jpg  # representative poster
```

### Many frames

```bash
ffmpeg -y -nostdin -i in.mp4 -vf "fps=1/10,scale=640:-2" frames/f_%04d.jpg      # one frame every 10 s
ffmpeg -y -nostdin -skip_frame nokey -i in.mp4 -fps_mode vfr -frame_pts true frames/kf_%d.jpg  # keyframes only (fast)
ffmpeg -y -nostdin -i in.mp4 -vf "select='gt(scene,0.3)',scale=1280:-2" -fps_mode vfr frames/scene_%03d.jpg  # scene changes
```

Use scene threshold 0.3–0.4 for camera cuts and 0.1–0.2 for subtle changes. **Text-only slides on a plain background need far lower: ~0.02–0.03.** Tested on a 3-slide white-background deck: `0.15` caught 1 slide, `0.05` caught 2, `0.03` caught all 3, because text barely moves the pixel difference. Check the frame count after extracting, and expect to dedupe the extras a low threshold lets in. List scene-change timestamps without writing images:

```bash
ffmpeg -nostdin -hide_banner -i in.mp4 -vf "select='gt(scene,0.3)',showinfo" -an -f null - 2>&1 | grep -o 'pts_time:[0-9.]*' | cut -d: -f2
```

### N frames evenly spaced (fast on any length)

Seeking with `-ss` before `-i` is near-instant at any position, so this is cheap even on a feature-length video. The midpoint of each slice avoids black first/last frames, and the timestamp is encoded in the filename.

```bash
in=in.mp4; n=12; mkdir -p frames
dur=$(ffprobe -v error -of csv=p=0 -show_entries format=duration "$in")
for i in $(seq 0 $((n-1))); do
  t=$(awk -v d="$dur" -v i="$i" -v n="$n" 'BEGIN{printf "%.2f", d*(i+0.5)/n}')
  ffmpeg -y -nostdin -hide_banner -loglevel error -ss "$t" -i "$in" -frames:v 1 -vf scale=480:-2 \
    "frames/$(printf '%02d' "$i")_$(awk -v t="$t" 'BEGIN{printf "%02d-%02d-%02d", t/3600, t%3600/60, t%60}').jpg"
done
```

### Contact sheet (one image summarizing the whole video)

```bash
# Single ffmpeg command — 16 frames, 4x4 grid (decodes the whole file; fine under ~10 min)
dur=$(ffprobe -v error -of csv=p=0 -show_entries format=duration in.mp4)
ffmpeg -y -nostdin -i in.mp4 -vf "fps=16/$dur,scale=320:-2,tile=4x4:padding=4:color=black" -frames:v 1 sheet.jpg

# Any length, labeled with timestamps — after the "N frames evenly spaced" loop above (needs ImageMagick)
args=(); for f in frames/*.jpg; do b=$(basename "$f" .jpg); args+=(-label "${b#*_}" "$f"); done
magick montage "${args[@]}" -tile 4x -geometry +4+4 -background '#111' -fill white -pointsize 14 sheet.jpg
```

Keep contact sheets ≤ ~1600 px on the long edge so details stay legible. View the sheet, pick an interesting cell, then extract that timestamp at full resolution.

### Visualize audio

```bash
ffmpeg -y -nostdin -i in.mp3 -filter_complex "aformat=channel_layouts=mono,showwavespic=s=1600x300:colors=0x3b82f6" -frames:v 1 waveform.png
ffmpeg -y -nostdin -i in.mp3 -lavfi "showspectrumpic=s=1600x600:legend=1" spectrogram.png
```

### Images → video

```bash
ffmpeg -y -nostdin -framerate 24 -i frames/f_%04d.png -c:v libx264 -crf 18 -pix_fmt yuv420p out.mp4   # numbered sequence
ffmpeg -y -nostdin -loop 1 -i cover.jpg -i audio.mp3 -c:v libx264 -tune stillimage -pix_fmt yuv420p \
  -vf "scale=trunc(iw/2)*2:trunc(ih/2)*2" -c:a aac -b:a 192k -shortest out.mp4                         # still + audio
```

The sequence must be contiguous. `-pattern_type glob` works on Linux/macOS but **not on Windows builds**; renumber the files instead.

### Describe an unknown video without watching it

Combine a technical summary, a timestamped contact sheet, and an audio-structure check. Each step stays cheap on any video length.

```bash
in=input.mp4; mkdir -p frames

# 1. Technical summary: duration, size, and every stream on one line each (ffprobe ships with ffmpeg)
ffprobe -v error -show_entries format=duration,size:stream=index,codec_type,codec_name,width,height,avg_frame_rate,channels:stream_tags=language,title -of compact "$in"

# 2. Optional, if exiftool is installed: capture date, device, GPS
exiftool -api QuickTimeUTC -s -CreationDate -CreateDate -Make -Model -GPSCoordinates -Duration "$in"

# 3. Visual overview, ffmpeg only: 16 evenly spaced frames in one 4x4 sheet
dur=$(ffprobe -v error -of csv=p=0 -show_entries format=duration "$in")
ffmpeg -y -nostdin -hide_banner -loglevel error -i "$in" -vf "fps=16/$dur,scale=320:-2,tile=4x4:padding=4:color=black" -frames:v 1 sheet.jpg
```

View `sheet.jpg`, pick an interesting cell, then extract that timestamp at full resolution with a single-frame command. The `tile` route decodes the whole file, so for videos over ~10 minutes use the fast-seek loop from "N frames evenly spaced" above and label each cell with its timestamp via `magick montage` (tested: 16 frames in a 4-column sheet, 1632×1016, labels readable).

For audio content, find speech versus silence with `silencedetect` (Audio Filters section) or render a waveform (Visualize audio). For a transcript, extract `-ar 16000 -ac 1` WAV for a speech-to-text tool.

---

## Metadata

For rich photo/video metadata (GPS, camera, XMP), use exiftool. ffmpeg handles basic container tags:

```bash
ffmpeg -y -nostdin -i input.mp4 -metadata title="My Video" -metadata artist="Name" -metadata date="2026" -c copy output.mp4
ffmpeg -y -nostdin -i input.mp4 -map_metadata -1 -c copy output.mp4                 # strip global metadata
ffmpeg -y -nostdin -i input.mp4 -i src.mp4 -map 0 -map_metadata 1 -c copy out.mp4   # copy tags from another file
```

Use `date`, not `year`, for MP4. Transcoding does not carry all source metadata: GPS and camera tags in QuickTime boxes are often lost — copy them back with exiftool.

### Cover art and chapters

```bash
ffmpeg -y -nostdin -i audio.mp3 -i cover.jpg -map 0:a -map 1 -c copy -id3v2_version 3 -disposition:v attached_pic output.mp3
ffmpeg -y -nostdin -i input.mp4 -f ffmetadata meta.txt                              # export chapters/metadata to edit
ffmpeg -y -nostdin -i input.mp4 -i meta.txt -map 0 -map_metadata 1 -map_chapters 1 -c copy output.mp4  # re-import
```

---

## Long-Running Jobs & Progress Monitoring

Transcodes regularly outlast a 2-minute tool timeout. Estimate first, then background anything expected to exceed ~60 seconds.

```bash
dur=$(ffprobe -v error -of csv=p=0 -show_entries format=duration input.mp4)
res=$(ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0 input.mp4)
```

Rough speeds (multiple of real time) on a modern CPU: `-c copy` 100–1000×; x264 `veryfast` 1080p 5–15×; x264 `medium` 1080p 2–6×; x265 `medium` 1080p 0.5–2×; SVT-AV1 preset 8 1080p 1–4×. Divide by ~4 for 4K.

### Background execution with a progress file

```bash
ffmpeg -y -nostdin -hide_banner -loglevel error \
  -progress progress.txt -stats_period 2 \
  -i input.mp4 -c:v libx264 -crf 23 -pix_fmt yuv420p -c:a aac output.mp4 \
  2> ffmpeg.log
```

Launch with the Bash tool's `run_in_background: true`. You are notified when it exits, so you don't poll in a sleep loop. `-progress` rewrites key=value blocks every `-stats_period` seconds (`frame=`, `fps=`, `out_time=`, `speed=`, `progress=continue`/`end`). A `speed` below `1x` means the job takes longer than the video's duration.

### Check progress on demand

```bash
dur=$(ffprobe -v error -of csv=p=0 -show_entries format=duration input.mp4)
us=$(grep out_time_us progress.txt | tail -1 | cut -d= -f2)
spd=$(grep speed= progress.txt | tail -1 | cut -d= -f2 | tr -d 'x ')
awk -v d="$dur" -v u="$us" -v s="$spd" 'BEGIN{done=u/1e6; printf "%.1f%% done, ~%.0fs remaining\n", 100*done/d, (s>0)?(d-done)/s:0}'
```

After the job: check the exit status and `ffmpeg.log` (a non-empty log at `-loglevel error` means trouble), confirm the output duration matches the input, and delete `progress.txt`, `ffmpeg.log`, and any `ffmpeg2pass-0.log*`.

---

## Batch Processing

Test on one file, then loop. Always quote `"$f"`, keep `-nostdin` (otherwise ffmpeg consumes the loop's stdin and skips files), and write to a separate output directory.

```bash
mkdir -p out
for f in *.mov; do
  [ -e "out/${f%.*}.mp4" ] && continue        # resumable: skip finished files
  ffmpeg -y -nostdin -hide_banner -loglevel error -i "$f" \
    -c:v libx264 -crf 23 -pix_fmt yuv420p -c:a aac -movflags +faststart "out/${f%.*}.mp4" \
    || echo "FAILED: $f" >> out/failures.txt
done
```

For long batches, run the whole loop with `run_in_background: true` and append a line per finished file to a log so progress is visible with `wc -l`. x264/x265 already use every core, so parallel transcodes rarely help; parallelism does help many small light jobs (thumbnails, remuxes) via `xargs -P`.

---

## Platform Notes (Windows / Git Bash)

ffmpeg runs as a native Windows executable under Git Bash. A few traps:

- **Null output:** use `-f null -`, not `/dev/null`. Redirecting the shell's stderr with `2>/dev/null` is fine under Git Bash (`2>NUL` in cmd/PowerShell).
- **Drive letters inside filters** need the colon escaped and forward slashes, because `:` separates filter options: `subtitles='C\:/subs/movie.srt'`, `fontfile='C\:/Windows/Fonts/arial.ttf'`. Relative paths avoid the problem; prefer them.
- **Fonts:** fontconfig is often not set up on Windows builds, so pass `fontfile=` explicitly to `drawtext`.
- **Glob patterns:** `-pattern_type glob` is unavailable on Windows builds. Use numbered sequences (`frame_%04d.png`) or a concat list.
- **Git Bash path conversion:** arguments starting with `/` are rewritten to Windows paths (helpful for file paths). If a non-path argument is mangled, prefix with `MSYS_NO_PATHCONV=1`.

---

## Example: Editing Together Multiple Sources

A common multi-camera scenario: full-screen drone footage, a phone clip of takeoff/landing, and a screen recording of a controller app, combined into one edited video with the screen recording inset as PIP during the flight, all clips joined together, and audio mixed from multiple tracks.

General workflow:

1. **Trim each source to its usable range first** (fast, stream-copy where possible) — don't carry raw, untrimmed footage through every downstream step.
2. **Normalize before combining** — mismatched resolution, frame rate, or SAR across a phone/drone/screen recording will break `concat` and produce garbage with `overlay`. Scale, set `fps`, and `setsar=1` on every clip before combining.
3. **Composite PIP for the segments where the second angle matters** (e.g. the controller screen during flight), then `concat` that composited segment with plain full-screen segments (e.g. the takeoff shot alone).
4. **Mix audio last**, once the video timeline is locked — layer drone audio, phone audio, and/or a voiceover with `amix`, since drone motor noise typically needs to sit lower than a voiceover or the phone's audio.

```bash
# 1. Trim to usable ranges (stream copy, fast)
ffmpeg -y -nostdin -ss 00:00:05 -to 00:04:30 -i drone_raw.mp4 -c copy drone_trim.mp4
ffmpeg -y -nostdin -ss 00:00:10 -to 00:00:40 -i phone_raw.mp4 -c copy phone_trim.mp4
ffmpeg -y -nostdin -ss 00:00:00 -to 00:04:00 -i screen_raw.mp4 -c copy screen_trim.mp4

# 2 & 3. Normalize the drone clip and overlay the screen recording as PIP for its duration
ffmpeg -y -nostdin -i drone_trim.mp4 -i screen_trim.mp4 -filter_complex \
  "[0:v]scale=1920:1080,fps=30,setsar=1[main]; \
   [1:v]scale=480:-2,fps=30[pip]; \
   [main][pip]overlay=main_w-overlay_w-30:main_h-overlay_h-30[v]" \
  -map "[v]" -map 0:a -c:v libx264 -crf 23 -c:a aac flight_with_pip.mp4

# Normalize the phone takeoff clip separately (no PIP, just matching format)
ffmpeg -y -nostdin -i phone_trim.mp4 -vf "scale=1920:1080,fps=30,setsar=1" \
  -c:v libx264 -crf 23 -c:a aac phone_norm.mp4

# Concat the phone takeoff clip in front of the flight+PIP segment
ffmpeg -y -nostdin -i phone_norm.mp4 -i flight_with_pip.mp4 -filter_complex \
  "[0:v][0:a][1:v][1:a]concat=n=2:v=1:a=1[v][a]" \
  -map "[v]" -map "[a]" -c:v libx264 -crf 23 -c:a aac joined.mp4

# 4. Mix in a separately recorded voiceover, lowering the original audio underneath it
ffmpeg -y -nostdin -i joined.mp4 -i voiceover.wav -filter_complex \
  "[0:a]volume=0.4[bg]; [bg][1:a]amix=inputs=2:duration=first[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac final.mp4
```

Writing an intermediate file at each stage means a mistake in one step doesn't force redoing the whole pipeline — cheap disk space is worth it against re-running a long transcode.

---

## Agent Rules

1. **Always `-y -nostdin`**, and keep logs small with `-hide_banner -loglevel error`. Without `-y -nostdin`, a command that writes an existing output can hang forever waiting on a prompt.
2. **Probe first.** Duration decides whether to background the job; the codec decides whether `-c copy` is possible.
3. **Prefer `-c copy`** for container changes and keyframe-tolerant trims. Transcode only when you must change codec, bitrate, resolution, or apply a filter.
4. **Add `-pix_fmt yuv420p` and `-movflags +faststart`** to H.264/H.265 MP4 meant for sharing or the web.
5. **Use `-2`, never `-1`, for computed dimensions** (`scale=1280:-2`): H.264/H.265 need even sizes.
6. **Don't combine `-c copy` with `-vf`/`-af`/`-filter_complex`** on the same stream. Filters require decoding.
7. **Background anything over ~60 s** with `-progress FILE` and `run_in_background: true`; never block on a long transcode hoping it beats the timeout, and don't poll with sleep loops.
8. **Check build support** (`ffmpeg -encoders`, `-filters`, `-muxers`) rather than assuming `libx265`, `libsvtav1`, or `libass` exist.
9. **Verify every output** with ffprobe (duration, streams) after the job, and view a frame for visual changes.
10. **Create output directories first** and bound frame extraction with `-t`, `fps=1/N`, or `-frames:v N` so an unbounded `%04d.png` doesn't write hundreds of thousands of files.

---

## Further Reading

- [ffmpeg documentation](https://ffmpeg.org/ffmpeg.html) — full options, stream specifiers, filtergraph syntax
- [ffmpeg filters documentation](https://ffmpeg.org/ffmpeg-filters.html) — every filter with parameters
- Encoding guides: [H.264](https://trac.ffmpeg.org/wiki/Encode/H.264), [H.265](https://trac.ffmpeg.org/wiki/Encode/H.265), [VP9](https://trac.ffmpeg.org/wiki/Encode/VP9), [AV1](https://trac.ffmpeg.org/wiki/Encode/AV1)
- [Concatenation](https://trac.ffmpeg.org/wiki/Concatenate) and the [filtering guide](https://trac.ffmpeg.org/wiki/FilteringGuide)
