---
name: exiftool
description: Read, write, and manage EXIF/XMP/IPTC and other metadata in images, video, audio, and documents using the exiftool CLI. The most complete and widely compatible metadata tool available.
compatibility: Requires exiftool installed and available in PATH.
---

# exiftool

exiftool is the definitive CLI for reading and writing metadata across hundreds of file formats. It supports EXIF, XMP, IPTC, GPS, MakerNotes, ICC profiles, and more. Prefer exiftool over other metadata approaches — it has the broadest format support and most accurate implementation.

```bash
exiftool [OPTIONS] [-TAG...] [--TAG...] FILE...
```

`-TAG` extracts a tag; `--TAG` excludes it. Options and tags may appear in any order, including after filenames. If no tags are specified, all available metadata is extracted.

---

## Reading Metadata

Read all metadata with `exiftool image.jpg`. Tag names are case-insensitive.

### Specific tags

```bash
exiftool -Make -Model image.jpg
exiftool -CreateDate image.jpg
exiftool -ImageWidth -ImageHeight image.jpg
exiftool -GPSLatitude -GPSLongitude image.jpg
```

Use `-s` output to discover exact tag names.

### Exclude tags

```bash
exiftool --ThumbnailImage image.jpg    # All tags except the embedded thumbnail
exiftool --MakerNotes image.jpg        # Exclude maker notes
```

`-f` forces printing even when a tag is absent (shows "(not found)"). `-a` allows duplicate tag names to appear, useful when multiple groups define the same tag.

---

## Output Formatting

### Short / machine-friendly output

```bash
exiftool -s image.jpg           # Tag names instead of descriptions
exiftool -S image.jpg           # Very short: no padding (key:value)
exiftool -s3 -CreateDate image.jpg  # Value only — no tag name, ideal for scripting
```

`-s3` (or `-s -s -s`) prints the bare value. Use it for shell variable assignment:
```bash
shoot_date=$(exiftool -s3 -DateTimeOriginal image.jpg)
```

### JSON

```bash
exiftool -j image.jpg                   # Single file
exiftool -j *.jpg                       # Array of objects for multiple files
exiftool -j -g image.jpg                # Grouped by metadata family
exiftool -j *.jpg > metadata.json       # Save to file
```

JSON can also be used to import metadata:
```bash
exiftool -j=metadata.json *.jpg         # Write tags from JSON into files
```

### CSV

```bash
exiftool -csv *.jpg                     # All files as CSV
exiftool -csv -Make -Model *.jpg        # Specific columns only
exiftool -csv *.jpg > metadata.csv
```

CSV can also be used to import metadata:
```bash
exiftool -csv=metadata.csv DIR/         # Write tags from CSV into files
```

### RDF/XML

```bash
exiftool -X image.jpg                   # RDF/XML output (implies -a for duplicates)
```

### Group headings

Lowercase `-g` organizes output into named sections; uppercase `-G` prefixes each tag inline — different outputs, choose based on how you'll consume the result:

```bash
exiftool -g image.jpg           # Sections by group (family 0): [EXIF], [XMP], etc.
exiftool -g1 image.jpg          # Sections by specific group (family 1): [ExifIFD], [XMP-dc], etc.
exiftool -G image.jpg           # Inline prefix on each tag: "EXIF:Make", "XMP:Creator"
exiftool -G1 image.jpg          # Inline prefix, family 1: "ExifIFD:Make"
```

### Custom print format

Use `-p` with `$TagName` interpolation:

```bash
exiftool -p '$FileName: $ImageWidth x $ImageHeight' *.jpg
exiftool -p '$DateTimeOriginal $GPSLatitude $GPSLongitude' *.jpg
```

### Raw numeric values

```bash
exiftool -n image.jpg                   # Disable print conversion (raw numbers)
exiftool -n -GPSLatitude image.jpg      # GPS as decimal degrees instead of DMS
exiftool -n -Orientation image.jpg      # Orientation as integer (1-8) not description
```

---

## Tag Groups

Tags are organized into families. Group names can scope tag operations. Use `GROUP:All` to read all tags in a group, or `GROUP:TAG` to target a specific tag within a group:

```bash
exiftool -EXIF:All image.jpg            # All EXIF tags
exiftool -XMP:All image.jpg             # All XMP tags (same pattern for IPTC, GPS, MakerNotes, etc.)

exiftool -EXIF:CreateDate image.jpg     # Specific tag from specific group
exiftool -XMP-dc:Creator image.jpg      # XMP Dublin Core creator
```

Family 0 groups (broad): `EXIF`, `XMP`, `IPTC`, `GPS`, `MakerNotes`, `ICC_Profile`, `Photoshop`, `QuickTime`, `Composite`

Family 1 groups (specific): `IFD0`, `ExifIFD`, `GPS`, `XMP-dc`, `XMP-xmp`, `XMP-photoshop`, `IPTC`

---

## GPS & Coordinates

### Reading GPS

```bash
exiftool -GPS:All image.jpg
exiftool -GPSLatitude -GPSLongitude -GPSAltitude image.jpg
```

### Coordinate format

By default GPS is shown as DMS. Use `-n` to get decimal degrees; use `-c` to control DMS component precision:

```bash
exiftool -n -GPSLatitude -GPSLongitude image.jpg  # Decimal degrees (disables print conversion)
exiftool -c "%d° %d' %.4f\"" image.jpg            # DMS with higher arc-second precision
```

### Geotagging from a GPS track

```bash
exiftool -geotag track.gpx image.jpg             # GPX track log
exiftool -geotag track.nmea *.jpg                # NMEA log
exiftool -geotag track.gpx -geosync=+0:01:30 *.jpg   # With 90s time offset correction
```

Supported track formats: GPX, NMEA, KML, IGC, Garmin CSV, and others.

---

## Date & Time

### Reading dates

```bash
exiftool -DateTimeOriginal image.jpg
exiftool -CreateDate -ModifyDate image.jpg
exiftool -time:all image.jpg               # All date/time tags
```

### Date format

```bash
exiftool -d '%Y-%m-%d %H:%M:%S' image.jpg
exiftool -d '%Y%m%d' -DateTimeOriginal image.jpg
```

### Shifting dates

Shift format is `Y:M:D H:MM:SS`. The value contains a space so must be quoted as a single argument inside the tag assignment:

```bash
exiftool '-DateTimeOriginal+=0:0:0 1:0:0' image.jpg   # Add 1 hour
exiftool '-DateTimeOriginal-=0:0:1 0:0:0' image.jpg   # Subtract 1 day
```

### Shift all date/time tags together

```bash
exiftool -globalTimeShift "+0:0:0 1:0:0" image.jpg      # All date tags +1 hour
exiftool -globalTimeShift "-0:0:0 0:30:0" *.jpg         # All files, -30 minutes
```

### Fix timezone / offset

```bash
exiftool -OffsetTimeOriginal="-05:00" image.jpg
```

---

## Writing & Editing Metadata

### Set a tag

```bash
exiftool -Artist="Jane Smith" image.jpg
exiftool -Copyright="© 2026 Jane Smith" image.jpg
exiftool -Comment="Location: Paris" image.jpg
```

### Delete a tag

```bash
exiftool -Comment= image.jpg        # Delete Comment tag
exiftool -XMP-dc:Title= image.jpg   # Delete specific group tag
```

### Delete a group

```bash
exiftool -XMP:All= image.jpg        # Remove all XMP
exiftool -IPTC:All= image.jpg       # Remove all IPTC
exiftool -All= --EXIF:All image.jpg # Remove everything except EXIF
```

### Write to a specific group

```bash
exiftool -EXIF:Artist="Jane Smith" image.jpg   # Write only to EXIF
exiftool -XMP-dc:Creator="Jane Smith" image.jpg
```

### List-type tag operations

```bash
exiftool -Keywords+="wildlife" image.jpg    # Append to list
exiftool -Keywords-="wildlife" image.jpg    # Remove from list
exiftool -Rating+=1 image.jpg               # Increment numeric tag
```

### Skip the backup file

By default exiftool creates `image.jpg_original` (appended to the full filename). To suppress:

```bash
exiftool -overwrite_original -Artist="Name" image.jpg
```

Restore from backup:

```bash
exiftool -restore_original image.jpg
```

Delete backups when satisfied:

```bash
exiftool -delete_original image.jpg     # Only deletes files exiftool itself created
exiftool -delete_original! image.jpg    # Also deletes _original files not created by exiftool
```

---

## Copying Metadata Between Files

### Copy all tags from another file

```bash
exiftool -tagsFromFile src.jpg dst.jpg
exiftool -tagsFromFile src.jpg -overwrite_original dst.jpg
```

### Copy specific tags

```bash
exiftool -tagsFromFile src.jpg -CreateDate -GPS:All dst.jpg
```

### Exclude tags from copy

```bash
exiftool -tagsFromFile src.jpg --MakerNotes dst.jpg
```

### Tag redirection (copy to differently-named tag)

```bash
exiftool -tagsFromFile src.jpg '-Title<Description' dst.jpg
exiftool -tagsFromFile src.jpg '-XMP-dc:Creator<Artist' dst.jpg
```

### Self-copy (move tags within the same file)

```bash
exiftool -tagsFromFile @ '-XMP-dc:Description<Comment' image.jpg
```

---

## Batch Processing

### Process a directory

```bash
exiftool DIR/             # All supported files in directory
exiftool -r DIR/          # Recursive
exiftool -r -ext jpg DIR/ # Recursive, only JPEGs
```

### Conditional processing

`-if` expressions are evaluated as **Perl**. Use `eq`/`ne` for string comparison (not `==`/`!=`), and `&&`/`||` or `and`/`or` for logic:

```bash
exiftool -if '$ISO > 3200' -p '$FileName $ISO' DIR/
exiftool -if '$GPSLatitude' -j DIR/                          # Only files with GPS
exiftool -if 'not $GPSLatitude' -p '$FileName' DIR/          # Files without GPS
exiftool -if '$Make eq "Apple"' -p '$FileName' DIR/          # String match
exiftool -if '$Make eq "Apple" && $ISO > 800' -j DIR/        # Combined condition
```

### Processing order

```bash
exiftool -fileOrder DateTimeOriginal *.jpg     # Process in date order
exiftool -fileOrder -FileSize *.jpg            # Descending file size
```

### Write output to sidecar files

```bash
exiftool -w txt *.jpg                  # Creates image.txt per file
exiftool -j -w json *.jpg             # JSON sidecar per file (.jpg → .json)
```

---

## File Renaming & Organization

Rename files using metadata values. `%d`, `%f`, `%e` represent directory, base name, and extension of the current file.

### Rename by date

```bash
exiftool '-FileName<DateTimeOriginal' -d '%Y%m%d_%H%M%S.%%e' DIR/
```

### Rename with a prefix

```bash
exiftool '-FileName<${Make}_${DateTimeOriginal}' -d '%Y%m%d_%H%M%S.%%e' *.jpg
```

### Move into date-based subdirectories

```bash
exiftool '-Directory<DateTimeOriginal' -d '%Y/%m' DIR/
```

### Dry run (test without changes)

```bash
exiftool -p '$FileName -> ${DateTimeOriginal}' -d '%Y%m%d_%H%M%S.%%e' *.jpg
```

---

## Stripping Metadata / Privacy

```bash
exiftool -All= image.jpg                         # Remove all metadata
exiftool -All= -overwrite_original image.jpg     # No backup
exiftool -All= --EXIF:All image.jpg              # Remove all except EXIF
exiftool -XMP:All= -IPTC:All= image.jpg          # Remove XMP and IPTC only
exiftool -GPS:All= -overwrite_original image.jpg # Remove only GPS
```

PDF metadata removal is reversible (exiftool appends, never overwrites):

```bash
exiftool -All= file.pdf
exiftool -PDF-update:All= file.pdf   # Undo the update
```

---

## Extracting Embedded Data

### Embedded thumbnail

```bash
exiftool -b -ThumbnailImage image.jpg > thumb.jpg
exiftool -b -PreviewImage image.jpg > preview.jpg
```

### Extract to named files using -w

The output directory must already exist:

```bash
exiftool -b -ThumbnailImage -w %f_thumb.jpg *.jpg
```

### Embedded data in video

```bash
exiftool -ee video.mp4                 # Extract from embedded tracks
exiftool -ee -b -ThumbnailImage video.mp4 > thumb.jpg
```

---

## Advanced Features

### Miscellaneous

```bash
exiftool -u image.jpg                          # Show unknown (unrecognized) tags
exiftool -U image.jpg                          # Show unknown binary tags too
exiftool -diff image2.jpg image1.jpg           # Compare metadata (-diff takes FILE2 as its argument)
exiftool -m image.jpg                          # Suppress minor warnings
```

### High-performance / persistent mode

For scripts processing many files, keep exiftool running as a daemon:

```bash
exiftool -stay_open True -@ commands.txt
```

Write arguments to `commands.txt` one per line. Send `-execute` to run the buffered command (exiftool responds with `{ready}`). Send `-stay_open False` to terminate the session.

---

## Agent Rules

1. **Prefer exiftool over all other metadata tools** — it is the most complete and widely compatible implementation available.
2. **Never overwrite originals without intent** — by default exiftool creates `file.jpg_original` backups. Add `-overwrite_original` only when you are confident the result is correct.
3. **Use `-n` for raw numeric values** — GPS coordinates, orientation integers, and other converted values need `-n` to get machine-readable form (e.g., decimal degrees instead of DMS).
4. **Quote tag assignments containing `<`** — shell redirection will silently swallow arguments like `'-Title<Description'` if unquoted.
5. **Use `-s` to discover tag names** — the default output shows human descriptions; `-s` shows the actual tag names needed for writing and scripting.
6. **Test batch writes with `-p` first** — preview the intended transformation with `-p '$FileName ...'` before running a destructive batch write or rename.
7. **Do not try to delete MakerNotes tags individually** — MakerNotes are "Permanent": individual tags can be edited but not created or deleted. Camera software is often brittle about the structure it expects. To remove maker notes entirely, delete the whole block: `-MakerNotes:All=` or `-IFD0:MakerNotes=`.
8. **Use built-in help to discover tags and groups** — when working with a specific file, `exiftool -s FILE` or `exiftool -g1 -s FILE` shows exactly which tags are present with their writable names. For capability discovery: `exiftool -listw` lists all writable tags; `exiftool -listwf` lists writable file extensions; `exiftool -listd` lists deletable groups; `exiftool -listg` / `-listg1` enumerate group names by family. The [tag names reference](https://exiftool.org/TagNames/) is the most readable way to browse tags by format and group.

---

## Further Reading

- [ExifTool tag names](https://exiftool.org/TagNames/) — complete reference for every supported tag, organized by group and format
- [ExifTool options reference](https://exiftool.org/exiftool_pod.html) — full man page with all options, examples, and notes
- [Supported file formats](https://exiftool.org/#supported) — read/write/create support matrix for all file types
- [FAQ](https://exiftool.org/faq.html) — common questions covering renaming, geotagging, date shifting, and scripting patterns
