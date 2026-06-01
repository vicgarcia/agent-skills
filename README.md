# agent-skills

A collection of skills for coding agents. Compatible with [pi](https://pi.dev) and [Claude Code](https://claude.ai/code).

## Skills

### [atd](https://github.com/vicgarcia/agent-skills/blob/main/atd/SKILL.md)

Schedule one-time deferred command execution using the Unix `at` command. Use for delayed operations, follow-up actions, and any task that should run once at a specific future time.

### [charts](https://github.com/vicgarcia/agent-skills/blob/main/charts/SKILL.md)

Generate publication-quality charts (bar, line, pie, scatter, radar, funnel, gauge, treemap, boxplot, heatmap, candlestick, sankey) as SVG or PNG files using [charts-cli](https://github.com/Michaelliv/charts-cli) and ECharts JSON configs. Use when the user asks to visualize data, create a chart, or render any statistical or business graphic. Requires Node.js 18+ and `npm install -g charts-cli`.

### [date](https://github.com/vicgarcia/agent-skills/blob/main/date/SKILL.md)

Date and time operations via the `date` system command. Provides ground-truth current time, date arithmetic, and full calendar navigation (month boundaries, weekday lookups, week ranges) so agents don't rely on their unreliable internal sense of time.

### [drawio](https://github.com/vicgarcia/agent-skills/blob/main/drawio/SKILL.md)

Create and edit draw.io diagram files (`.drawio`) by writing XML directly. Use for flowcharts, architecture diagrams, process flows, decision trees, org charts, ER diagrams, UML, network diagrams, and any other visual diagram. No dependencies required — produces files readable by draw.io desktop, diagrams.net, and the VS Code draw.io extension.

### [exiftool](https://github.com/vicgarcia/agent-skills/blob/main/exiftool/SKILL.md)

Read, write, and manage EXIF/XMP/IPTC metadata in images, video, audio, and documents using the [exiftool](https://exiftool.org) CLI. The most complete and widely compatible metadata tool available.

### [ffmpeg](https://github.com/vicgarcia/agent-skills/blob/main/ffmpeg/SKILL.md)

Convert, transcode, trim, filter, and process video and audio files using the [ffmpeg](https://ffmpeg.org) CLI. Handles format conversion, encoding, scaling, audio manipulation, subtitles, GIF creation, concatenation, metadata, and long-running background jobs.

### [ffprobe](https://github.com/vicgarcia/agent-skills/blob/main/ffprobe/SKILL.md)

Inspect and extract metadata from video, audio, and multimedia container files using the [ffprobe](https://ffmpeg.org/ffprobe.html) CLI. Query streams, format info, packets, frames, and chapters in multiple output formats.

### [imagemagick](https://github.com/vicgarcia/agent-skills/blob/main/imagemagick/SKILL.md)

Manipulate, convert, and process images from the command line using the [ImageMagick](https://imagemagick.org) `magick` CLI. Covers format conversion, resizing, cropping, annotation, compositing, and effects across 200+ formats.

### [nmap](https://github.com/vicgarcia/agent-skills/blob/main/nmap/SKILL.md)

Network reconnaissance using nmap and nmap-vulners. Covers the full recon pipeline from host discovery through CVE identification — port scanning, service detection, OS fingerprinting, NSE scripts, UDP services, and timeout-safe progressive scanning patterns for reliable agent execution. Requires Docker setup with `setcap` for non-root raw socket access and the vulners NSE script for CVE lookup.
