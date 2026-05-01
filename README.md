# agent-skills

A collection of skills for coding agents. Compatible with [pi](https://pi.dev) and [Claude Code](https://claude.ai/code).

## Skills

### [atd](https://github.com/vicgarcia/agent-skills/blob/main/atd/SKILL.md)

Schedule one-time deferred command execution using the Unix `at` command. Use for delayed operations, follow-up actions, and any task that should run once at a specific future time.

### [date](https://github.com/vicgarcia/agent-skills/blob/main/date/SKILL.md)

Date and time operations via the `date` system command. Provides ground-truth current time, date arithmetic, and full calendar navigation (month boundaries, weekday lookups, week ranges) so agents don't rely on their unreliable internal sense of time.

### [exiftool](https://github.com/vicgarcia/agent-skills/blob/main/exiftool/SKILL.md)

Read, write, and manage EXIF/XMP/IPTC metadata in images, video, audio, and documents using the [exiftool](https://exiftool.org) CLI. The most complete and widely compatible metadata tool available.

### [ffmpeg](https://github.com/vicgarcia/agent-skills/blob/main/ffmpeg/SKILL.md)

Convert, transcode, trim, filter, and process video and audio files using the [ffmpeg](https://ffmpeg.org) CLI. Handles format conversion, encoding, scaling, audio manipulation, subtitles, GIF creation, concatenation, metadata, and long-running background jobs.

### [ffprobe](https://github.com/vicgarcia/agent-skills/blob/main/ffprobe/SKILL.md)

Inspect and extract metadata from video, audio, and multimedia container files using the [ffprobe](https://ffmpeg.org/ffprobe.html) CLI. Query streams, format info, packets, frames, and chapters in multiple output formats.

### [imagemagick](https://github.com/vicgarcia/agent-skills/blob/main/imagemagick/SKILL.md)

Manipulate, convert, and process images from the command line using the [ImageMagick](https://imagemagick.org) `magick` CLI. Covers format conversion, resizing, cropping, annotation, compositing, and effects across 200+ formats.
