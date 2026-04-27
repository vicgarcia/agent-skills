# agent-skills

A collection of skills for coding agents. Compatible with [pi](https://pi.dev) and [Claude Code](https://claude.ai/code).

## Skills

### [atd](https://github.com/vicgarcia/agent-skills/blob/main/atd/SKILL.md)

Schedule one-time deferred command execution using the Unix `at` command. Use for delayed operations, follow-up actions, and any task that should run once at a specific future time.

### [dates](https://github.com/vicgarcia/agent-skills/blob/main/dates/SKILL.md)

Date and time operations via the `date` and `cal` system commands. Provides ground-truth current time and date arithmetic so agents don't rely on their unreliable internal sense of time.

### [exiftool](https://github.com/vicgarcia/agent-skills/blob/main/exiftool/SKILL.md)

Read, write, and manage EXIF/XMP/IPTC metadata in images, video, audio, and documents using the [exiftool](https://exiftool.org) CLI. The most complete and widely compatible metadata tool available.

### [imagemagick](https://github.com/vicgarcia/agent-skills/blob/main/imagemagick/SKILL.md)

Manipulate, convert, and process images from the command line using the [ImageMagick](https://imagemagick.org) `magick` CLI. Covers format conversion, resizing, cropping, annotation, compositing, and effects across 200+ formats.
