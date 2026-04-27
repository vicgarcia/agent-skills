---
name: exiftool
description: Read, write, and manage EXIF/XMP/IPTC and other metadata in images, video, audio, and documents using the exiftool CLI. The most complete and widely compatible metadata tool available.
compatibility: Requires exiftool installed and available in PATH.
---

# exiftool

exiftool is the definitive CLI for reading and writing metadata across hundreds of file formats. It supports EXIF, XMP, IPTC, GPS, MakerNotes, ICC profiles, and more. Prefer exiftool over other metadata approaches — it has the broadest format support and most accurate implementation.

---

## Invocation

```bash
exiftool [OPTIONS] [-TAG...] [--TAG...] FILE...
```

Options and tags may appear in any order, including after file names.

```bash
exiftool image.jpg              # Read all metadata
exiftool -ver                   # Print version number
exiftool -listf                 # List all supported file extensions
exiftool -listg                 # List all group names (family 0)
exiftool -listg1                # List all group names (family 1)
exiftool -listw                 # List all writable tags
exiftool -listd                 # List all deletable groups
```

---

## Reading Metadata

### All tags

```bash
exiftool image.jpg              # Human-readable descriptions
exiftool -s image.jpg           # Short format: tag names instead of descriptions
exiftool -S image.jpg           # Very short: no padding, machine-friendly
exiftool -j image.jpg           # JSON output
exiftool -X image.jpg           # RDF/XML output
```

### Specific tags

```bash
exiftool -Make -Model image.jpg
exiftool -CreateDate image.jpg
exiftool -ImageWidth -ImageHeight image.jpg
exiftool -GPSLatitude -GPSLongitude image.jpg
```

Tag names are case-insensitive. Use `-s` output to discover exact tag names.

### Exclude tags

```bash
exiftool --ThumbnailImage image.jpg    # All tags except the embedded thumbnail
exiftool --MakerNotes image.jpg        # Exclude maker notes
```

### Force print even when missing

```bash
exiftool -f -CreateDate -GPSLatitude image.jpg   # Print "(not found)" for absent tags
```

### Allow duplicates

```bash
exiftool -a image.jpg           # Show duplicate tag names (e.g., multiple XMP namespaces)
```

---

## Output Formatting

### JSON

```bash
exiftool -j image.jpg                   # Single file
exiftool -j *.jpg                       # Array of objects for multiple files
exiftool -j -g image.jpg                # Grouped by metadata family
exiftool -j > metadata.json             # Save to file
```

### CSV

```bash
exiftool -csv *.jpg                     # All files as CSV
exiftool -csv -Make -Model *.jpg        # Specific columns only
exiftool -csv > metadata.csv
```

### Tab-delimited

```bash
exiftool -t -Make -Model *.jpg
```

### Group headings

```bash
exiftool -g image.jpg           # Organize output by group (family 0)
exiftool -g1 image.jpg          # Organize by specific group (family 1: ExifIFD, XMP-dc, etc.)
exiftool -G image.jpg           # Print group name before each tag value
exiftool -G1 image.jpg          # Print family 1 group name before each tag
```

### Long format

```bash
exiftool -l image.jpg           # 2-line format: tag name + value on separate lines
```

### Custom print format

Use `-p` with `$TagName` interpolation:

```bash
exiftool -p '$FileName: $ImageWidth x $ImageHeight' *.jpg
exiftool -p '$DateTimeOriginal $GPSLatitude $GPSLongitude' *.jpg
```

### Sorted output

```bash
exiftool -sort image.jpg
```

### Raw numeric values

```bash
exiftool -n image.jpg                   # Disable print conversion (raw numbers)
exiftool -n -GPSLatitude image.jpg      # GPS as decimal degrees instead of DMS
exiftool -n -Orientation image.jpg      # Orientation as integer (1-8) not description
```

### Tabular output

```bash
exiftool -T -Make -Model -DateTimeOriginal *.jpg   # Fixed-width table
```

---

## Tag Groups

Tags are organized into families. Group names can scope tag operations:

```bash
exiftool -EXIF:All image.jpg            # All EXIF tags
exiftool -XMP:All image.jpg             # All XMP tags
exiftool -IPTC:All image.jpg            # All IPTC tags
exiftool -GPS:All image.jpg             # All GPS tags
exiftool -MakerNotes:All image.jpg      # All MakerNotes

exiftool -EXIF:CreateDate image.jpg     # Specific tag from specific group
exiftool -XMP-dc:Creator image.jpg      # XMP Dublin Core creator
```

Family 0 groups (broad): `EXIF`, `XMP`, `IPTC`, `GPS`, `MakerNotes`, `ICC_Profile`, `Photoshop`, `QuickTime`, `Composite`

Family 1 groups (specific): `IFD0`, `ExifIFD`, `GPS`, `XMP-dc`, `XMP-xmp`, `XMP-photoshop`, `IPTC`

```bash
exiftool -listg    # Show all family 0 groups
exiftool -listg1   # Show all family 1 groups
```

---

## GPS & Coordinates

### Reading GPS

```bash
exiftool -GPS:All image.jpg
exiftool -GPSLatitude -GPSLongitude -GPSAltitude image.jpg
exiftool -n -GPSLatitude -GPSLongitude image.jpg   # Decimal degrees
```

### Coordinate format

```bash
exiftool -c "%.6f" image.jpg                     # Decimal degrees (6 places)
exiftool -c "%d° %d' %.2f\"" image.jpg           # Degrees, minutes, seconds
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

Shift by years:months:days hours:minutes:seconds:

```bash
exiftool -DateTimeOriginal+=0:0:0 1:0:0 image.jpg   # Add 1 hour
exiftool -DateTimeOriginal-=0:0:1 image.jpg          # Subtract 1 day
```

### Shift all date/time tags together

```bash
exiftool -globalTimeShift +1:0:0 image.jpg           # All date tags +1 hour
exiftool -globalTimeShift -0:0:0 0:30:0 *.jpg        # All files, -30 minutes
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

### Delete all metadata

```bash
exiftool -All= image.jpg
exiftool -All= -overwrite_original image.jpg
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
```

### Numeric increment/decrement

```bash
exiftool -Rating+=1 image.jpg
```

### Skip the backup file

By default exiftool creates `image_original.jpg`. To suppress:

```bash
exiftool -overwrite_original -Artist="Name" image.jpg
```

Restore from backup:

```bash
exiftool -restore_original image.jpg
```

Delete backups when satisfied:

```bash
exiftool -delete_original image.jpg
exiftool -delete_original! image.jpg   # No confirmation prompt
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

### Args pipeline (copy via intermediate file)

Export as exiftool arguments, optionally edit, then apply:

```bash
exiftool -args -G1 --FileName --Directory src.jpg > out.args
# edit out.args if needed
exiftool -@ out.args dst.jpg
```

---

## Batch Processing

### Process a directory

```bash
exiftool DIR/             # All supported files in directory
exiftool -r DIR/          # Recursive
exiftool -r . *.jpg       # Recursive, only JPEGs
```

### Filter by extension

```bash
exiftool -ext jpg -ext png DIR/       # Only JPG and PNG
exiftool -ext jpg DIR/                # Only JPG
```

### Conditional processing

```bash
exiftool -if '$ISO > 3200' -p '$FileName $ISO' DIR/
exiftool -if '$GPSLatitude' -j DIR/    # Only files with GPS
exiftool -if 'not $GPSLatitude' -CreateDate -p '$FileName' DIR/
```

### Processing order

```bash
exiftool -fileOrder DateTimeOriginal *.jpg     # Process in date order
exiftool -fileOrder -FileSize *.jpg            # Descending file size
```

### Progress display

```bash
exiftool -progress -r DIR/
```

### Write output to sidecar files

```bash
exiftool -w txt *.jpg                  # Creates image.txt per file
exiftool -w %.json -j *.jpg            # JSON sidecar per file
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

```bash
exiftool -b -ThumbnailImage -w thumb/%f_thumb.jpg *.jpg
```

### Embedded data in video

```bash
exiftool -ee video.mp4                 # Extract from embedded tracks
exiftool -ee -b -ThumbnailImage video.mp4 > thumb.jpg
```

---

## Comparing Files

```bash
exiftool -diff image1.jpg image2.jpg        # Show differing tags
```

---

## Advanced Features

### Unknown tags

```bash
exiftool -u image.jpg      # Show unknown (unrecognized) tags
exiftool -U image.jpg      # Show unknown binary tags too
```

### HTML binary dump (for debugging)

```bash
exiftool -htmlDump image.jpg > dump.html
open dump.html
```

### High-performance / persistent mode

For scripts processing many files, keep exiftool running as a daemon:

```bash
exiftool -stay_open True -@ commands.txt
```

Write command-line arguments to `commands.txt` one per line, terminated by `-execute`.

### Ignore minor errors

```bash
exiftool -m image.jpg      # Suppress minor warnings
```

### API options

```bash
exiftool -api LargeFileSupport=1 large.tiff
exiftool -api QuickTimeHandler=1 video.mp4
```

### Custom config / user-defined tags

```bash
exiftool -config ~/.exiftool/custom.cfg image.jpg
```

---

## Agent Rules

1. **Prefer exiftool over all other metadata tools** — it is the most complete and widely compatible implementation available.
2. **Never overwrite originals without intent** — by default exiftool creates `_original` backups. Add `-overwrite_original` only when you are confident the result is correct.
3. **Use `-n` for raw numeric values** — GPS coordinates, orientation integers, and other converted values need `-n` to get machine-readable form (e.g., decimal degrees instead of DMS).
4. **Quote tag assignments containing `<`** — shell redirection will silently swallow arguments like `'-Title<Description'` if unquoted.
5. **Use `-s` to discover tag names** — the default output shows human descriptions; `-s` shows the actual tag names needed for writing and scripting.
6. **Test batch writes with `-p` first** — preview the intended transformation with `-p '$FileName ...'` before running a destructive batch write or rename.
7. **Use built-in help to discover tags and groups** — when working with a specific file, `exiftool -s FILE` or `exiftool -g1 -s FILE` shows exactly which tags are present with their writable names. For capability discovery: `exiftool -listw` lists all writable tags; `exiftool -listwf` lists writable file extensions; `exiftool -listd` lists deletable groups; `exiftool -listg` / `-listg1` enumerate group names by family. The [tag names reference](https://exiftool.org/TagNames/) is the most readable way to browse tags by format and group.

---

## Further Reading

- [ExifTool tag names](https://exiftool.org/TagNames/) — complete reference for every supported tag, organized by group and format
- [ExifTool options reference](https://exiftool.org/exiftool_pod.html) — full man page with all options, examples, and notes
- [Supported file formats](https://exiftool.org/#supported) — read/write/create support matrix for all file types
- [FAQ](https://exiftool.org/faq.html) — common questions covering renaming, geotagging, date shifting, and scripting patterns
- [Forum](https://exiftool.org/forum/) — active community for edge cases and advanced usage
