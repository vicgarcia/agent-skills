# exiftool

Read, write, copy, and strip metadata (EXIF, XMP, IPTC, GPS, MakerNotes, QuickTime, PDF, and more) across hundreds of formats with the [exiftool](https://exiftool.org) CLI — the most complete and accurate metadata tool available.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

## What it covers

- **Reading** — specific tags, groups, `-G1 -a -s`, JSON/CSV/XML output, custom print formats, raw numeric values.
- **Writing & editing** — set/delete tags, list operations, group targeting, backups and restore.
- **Dates & times** — shifting, timezones, and QuickTime/MP4 UTC handling for video.
- **GPS** — reading as DMS or decimal, writing coordinates, geotagging from GPX/NMEA tracks.
- **Copying metadata** — between files, selective copy/exclude, tag redirection, restoring metadata after a magick/ffmpeg conversion.
- **Renaming & organizing** — rename and sort files into folders by capture date (with dry runs).
- **Privacy scrub** — stripping photos and videos, a first-pass check for common identifying tags, embedded-metadata limitations, keeping chosen tags, and PDF/video caveats.
- **Organize by capture date** — mixed photo/video sorting with collision-safe names, `TestName` dry runs, and counting files with no embedded date.
- **Extraction** — embedded thumbnails, RAW previews, video timed metadata (GoPro/DJI), large-file support.
- **Batch** — recursive, conditional (`-if`), persistent `-stay_open` mode, and Windows/Git Bash notes.

## Requirements

`exiftool` installed and on `PATH`.

```bash
winget install OliverBetz.ExifTool          # Windows
brew install exiftool                         # macOS
sudo apt install libimage-exiftool-perl      # Debian/Ubuntu
```

## Canonical documentation

- [ExifTool tag names](https://exiftool.org/TagNames/)
- [ExifTool options reference](https://exiftool.org/exiftool_pod.html)
- [Supported file formats](https://exiftool.org/#supported) · [FAQ](https://exiftool.org/faq.html)
